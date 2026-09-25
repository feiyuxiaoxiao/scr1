# SCR1 RISC-V Core 学习笔记

本项目是 Syntacore 开源的 MCU 级 32 位 RISC-V 处理器内核 SCR1，纯 SystemVerilog 编写，支持 RV32I/RV32E + RVM/RVC 扩展，2~4 级流水线。本笔记按学习顺序记录各阶段内容。

## 项目概览

```
src/                        RTL 源码（重点）
├── includes/               宏定义、架构描述、CSR/调试定义
├── core/                   CPU 核心
│   ├── scr1_core_top.sv    核心顶层
│   └── pipeline/           流水线各单元
├── top/                    集群顶层（AHB/AXI 总线、TCM、定时器）
└── tb/                     测试平台
sim/tests/                  软件测试程序（hello、ISR、ISA 测试、CoreMark）
dependencies/               riscv-tests 等官方测试套件
Makefile                    构建仿真流程（Verilator/VCS/ModelSim）
```

### 核心架构（2~4 级流水线）

`src/core/pipeline/scr1_pipe_top.sv` 是理解一切的关键，它实例化了流水线全部模块：

| 模块 | 流水线职责 | 对应文件 |
|---|---|---|
| IFU | 取指 Fetch | `scr1_pipe_ifu.sv` |
| IDU | 译码 Decode | `scr1_pipe_idu.sv` |
| EXU | 执行 Execute | `scr1_pipe_exu.sv` |
| MPRF | 寄存器堆（读写 x0~x31） | `scr1_pipe_mprf.sv` |
| CSR | 控制状态寄存器 + 中断异常 | `scr1_pipe_csr.sv` |

可选模块：IPIC 中断控制器、HDU 调试单元、TDU 硬件断点、TCM 紧耦合内存。

### 学习路径

| 阶段 | 内容 | 目标 |
|---|---|---|
| 1 | `scr1_arch_description.svh` | 理解 4 种配置（MAX/BASE/MIN/CUSTOM）和宏开关 |
| 2 | `scr1_pipe_top.sv` | 建立流水线整体连接关系 |
| 3 | IFU→IDU→EXU→MPRF→CSR | 理解指令从取指到退休的完整通路 |
| 4 | `sim/tests/hello` + testbench | 理解软件如何运行在核心上 |
| 5 | 跑一次仿真（Verilator） | 动手验证理解 |

---

## 第 1 课：RISC-V 是什么 + CPU 如何执行指令

### ISA 与微架构的分工

- **ISA（指令集架构）**：硬件与软件的"合同"，只规定指令、寄存器、内存怎么读写，不规定硬件实现。
- **微架构（Microarchitecture）**：真正的硬件设计（流水线深度、旁路等）。**SCR1 就是一个微架构实现**，实现 RV32I 指令集。

### RV32I 最核心的 3 件事

- **寄存器堆**：32 个通用寄存器 x0~x31，每个 32 位（所以叫 RV**32**）。x0 恒为 0。
- **指令格式**：所有指令都是 32 位（RVC 压缩指令 16 位）。R 型格式：

```
31     25 24   20 19   15 14   12 11     7 6      0
+--------+-------+-------+-------+--------+--------+
|  funct7 |  rs2  |  rs1  |funct3 |  rd    | opcode |
+--------+-------+-------+-------+--------+--------+
```

  - `opcode`：这条指令干什么
  - `rs1/rs2`：源寄存器号
  - `rd`：目标寄存器号
  - `funct3/funct7`：进一步细分操作

- **程序计数器 PC**：指向当前执行的指令地址。取完一条 PC 自动 +4（压缩指令 +2）。

### CPU 执行一条指令的四个动作

```
取指 (Fetch) → 译码 (Decode) → 执行 (Execute) → 写回 (Write Back) → 更新PC → 回到取指
```

### 流水线：让四个动作重叠

流水线把步骤拆开让它们重叠，理想情况每周期完成一条指令（IPC=1）。SCR1 对应 2~4 级流水线。

---

## 第 2 课：把流水线概念落到 SCR1 的真实连线上

`scr1_pipe_top.sv` 自己不干活，纯粹是**接线员**——声明信号，连模块。

### 模块间通信的"握手"协议

SCR1 用**单一 valid 信号 + 上游 ready 应答**：

- `ifu2idu_vd`（取指有效）：IFU 说"我给你一条有效指令"
- `idu2ifu_rdy`（译码就绪）：IDU 说"我现在能接收"

两者同时为 1，数据才算真正交接成功。

### 各站信号流

- **取指 Fetch**（`i_pipe_ifu`）：拿 PC 向 IMEM 发请求，指令经 `ifu2idu_instr_o` 送出。
- **译码 Decode**（`i_pipe_idu`）：纯组合逻辑，拆字段生成 `idu2exu_cmd`，同时输出 `idu2exu_use_rs1_o`、`idu2exu_use_rs2_o`、`idu2exu_req_o`。
- **执行 Execute**（`i_pipe_exu`）：通过 `exu2mprf_rs1_addr_o`/`mprf2exu_rs1_data_i` 读寄存器，ALU 运算后经 `exu2mprf_w_req_o` + `exu2mprf_rd_addr_o` + `exu2mprf_rd_data_o` 写回。
- **PC 前进**（`:468-471`）：`exu2pipe_pc_curr_o`（当前 PC，给 CSR 记录）、`exu2pipe_pc_new_req_o`（换 PC 请求）、`exu2pipe_pc_new_o`（新 PC）。顺序执行时 new_pc = curr_pc + 4。

### 完整数据通路

```
IFU取指 ──ifu2idu_instr──▶ IDU译码 ──idu2exu_cmd──▶ EXU执行 ──exu2mprf_rd_*──▶ MPRF写回
                                                      │
                          new_pc_req/new_pc ◀─────────┘（PC更新送回IFU）
```

核心信号流一共 5 类：取指数据、译码命令、寄存器读写、CSR 读写、PC 更新。

---

## 第 3 课：取指单元 `scr1_pipe_ifu.sv`

IFU 职责：维护 PC、向指令内存发请求、把指令喂给 IDU。复杂度来自 RVC 压缩指令的对齐问题和 IMEM 延迟。

### 指令队列

```
SCR1_IFU_Q_SIZE_WORD = 2    // 队列能存 2 个 32 位字
SCR1_IFU_Q_SIZE_HALF = 4    // 等价于 4 个半字（16 位）
```

以**半字（16 位）**为单位存储，因为 RVC 只有 16 位。核心是环形缓冲：写指针 `q_wptr`、读指针 `q_rptr`（`:386-411`），两者相等即空（`:453`）。

### 指令类型译码

`instr_type`（`:282-302`）根据 `[1:0]` 判断一个字里的指令组合：

```
2'b00 : RVC + RVC      // 两个都是压缩指令
2'b10 : RVI低 + RVC    // 上半是 RVI 低半，下半是 RVC
default: RVI高 + RVI低  // 一个完整 RVI 指令
```

`instr_hi_rvi_lo_ff`（`:290`）记忆"上一个半字是否是 RVI 指令低半"，用于拼接跨取指边界的指令。

### 有限状态机

`type_scr1_ifu_fsm_e`（`:91-94`）只有两个状态：

- `IDLE`：等待 `ifu_fetch_req`（来自 EXU 的 new_pc 请求）
- `FETCH`：持续向 IMEM 发请求，直到 `ifu_stop_req`（异常或停取）

### 与 IMEM 的握手

- 请求侧（`:513`）：`ifu2imem_req_o` 与 `imem2ifu_req_ack_i` 都拉高 = 握手完成。
- 响应侧（`:507-510`）：`imem2ifu_resp_i` 返回 OK 或 ER。IMEM 可能延迟多周期，用"未完成事务计数器" `imem_pnd_txns_cnt`（`:543-553`）跟踪。

### 错误处理

- IMEM 返回 ER → `imem_resp_er_discard_pnd` 触发，丢弃所有未消费指令（`:558-591`）。
- 错误标志 `q_err[]` 与指令一起入队（`:429`），经 `ifu2idu_imem_err_o` 传给 IDU，指令以"取指异常"身份退休。

### 输出给 IDU

```
ifu2idu_instr_o = q_head_is_rvc ? q_data_head                 // 压缩指令，只取低半
                                : {q_data_next, q_data_head}; // RVI，拼两个半字
```

### 难点总结

队列 + 对齐处理是支持 RVC 付出的代价（纯 RV32I 的 IFU 会简单得多）。文件末尾 `:769-822` 有 SVA 断言，用于验证。

---

## 第 4 课：译码单元 `scr1_pipe_idu.sv`

IDU 是纯组合逻辑，**没有状态、没有寄存器**，只有一大块 `always_comb` 译码。

### 输出：`idu2exu_cmd_o`

`type_scr1_exu_cmd_s` 是打包结构体（定义在 `scr1_riscv_isa_decoding.svh`），是"给 EXU 的完整指令单"：

| 字段 | 含义 |
|---|---|
| `ialu_cmd` | ALU 操作：ADD/SUB/MUL/XOR/SLL… |
| `ialu_op` | 运算数来源：寄存器-寄存器 或 寄存器-立即数 |
| `sum2_op` | 第二个加法器的源：PC+立即数（跳转/取地址）或 寄存器+立即数（访存） |
| `lsu_cmd` | 访存操作：LB/LH/LW/LBU/LHU/SB/SH/SW |
| `csr_cmd/csr_op` | CSR 读写操作 |
| `rd_wb_sel` | 结果写回来源：IALU / 访存 / CSR / 立即数 / PC+4 |
| `jump_req` / `branch_req` | 跳转/分支请求 |
| `exc_req` / `exc_code` | 异常请求与异常码 |
| `rs1_addr/rs2_addr/rd_addr` | 三个寄存器号 |
| `imm` | 符号扩展后的立即数 |

### 设计要点

- **默认值先行**（`:96-123`）：`always_comb` 开头把 `idu2exu_cmd_o` 全部赋安全默认值，然后逐个 opcode 覆盖，保证不产生锁存器。
- **字段切分**（`:85-92`）：`instr_type = instr[1:0]`、`rvi_opcode = instr[6:2]`、`funct3`/`funct7`。RVC 字段位置和 RVI 不同，`funct3` 用三目选择（`:89`）。
- **立即数展开**：RISC-V 立即数位散落在指令里，译码器重排。如 JAL（`:178`）：`imm = {{12{instr[31]}}, instr[19:12], instr[20], instr[30:21], 1'b0}`，`{12{instr[31]}}` 是符号扩展。

### 非法指令检查

每个分支检查编码合法性，非法则置 `rvi_illegal`。末尾（`:871-907`）把命令清零并改为：

```
idu2exu_cmd_o.exc_req = 1'b1;
idu2exu_cmd_o.exc_code = SCR1_EXC_CODE_ILLEGAL_INSTR;
```

### 定位

IDU 只是一张**查表**：opcode+funct3+funct7 → 控制信号。译码器负责"翻译"，执行单元负责"干活"。

---

## 第 5 课：执行单元 `scr1_pipe_exu.sv`

EXU 是 CPU 核心，1086 行，分 6 块读。

### 1. 指令队列（IDU 与 EXU 的缓冲）

默认配置下 IDU 和 EXU 之间**有一级寄存器**（`:339-370`），在 `exu_queue_en` 时锁存 `idu2exu_cmd_i`：

```
exu_queue_en = exu2idu_rdy_o & idu2exu_req_i   // :318
```

之后 EXU 所有逻辑读的都是打拍后的 `exu_queue`。

### 2. IALU：双操作数结构

实例化 `scr1_pipe_ialu`（`:445`），内部有**两条独立加法器通路**：

- `ialu_main_op1/op2` → `ialu_main_res`：主运算（算术/逻辑/比较）
- `ialu_addr_op1/op2` → `ialu_addr_res`：**地址计算**（PC+imm 或 寄存器+imm）

两条通路并行，`beq` 可同时"比较两寄存器"并"算目标地址 PC+imm"。

### 3. 异常：三类来源汇集

```
exu_exc_req = exu_queue_vd & (exu_queue.exc_req    // IDU 判定的异常（非法指令、ECALL、EBREAK）
                            | lsu_exc_req          // 访存异常（地址错位、访问错误）
                            | csr2exu_rw_exc_i     // CSR 读写异常
                            | ...);
```

异常码由 `exc_code` 编码器（`:514-528`）选择，`exc_trap_val`（`:545-575`）是附加信息——对非法指令会**重新拼出指令编码**（`:535-541`）放进 MTVAL。

### 4. PC 逻辑：CPU 的"方向舵"（重点）

```
always_comb begin
    case (1'b1)
        init_pc               : new_pc = 复位向量;
        take_exc/irq/mret     : new_pc = csr2exu_new_pc_i;  // 中断/异常/MRET
        wfi_run_start_ff      : new_pc = pc_curr_ff;    // WFI 醒来继续
        fencei_req            : new_pc = inc_pc;
        default               : new_pc = ialu_addr_res & JUMP_MASK;  // 分支/跳转
    endcase
end
```

`exu2ifu_pc_new_req_o`（`:722-735`）决定"要不要换 PC"。分支判定：

```
branch_taken = exu_queue.branch_req & ialu_cmp;  // :738
jb_taken     = exu_queue.jump_req | branch_taken; // :739
```

顺序指令不触发 `new_pc_req`，IFU 自己按 `inc_pc`（PC+4 或 +2）继续取。

### 5. 写回

`:881-890` 的 `rd_wb_sel` 多路选择器：

| rd_wb_sel | 写回值 |
|---|---|
| SUM2 | 地址加法器结果（AUIPC、JAL 目标） |
| IMM | 立即数（LUI、LI） |
| INC_PC | PC+4/PC+2（JAL/JALR 返回地址） |
| LSU | 访存读出的数据 |
| CSR | CSR 读出的值 |
| 默认 | IALU 主运算结果 |

`exu2mprf_w_req_o`（`:872-876`）把关：**有异常就不写回**（`~exu_exc_req`）。

### 6. 三个关键"退休"信号

```
exu2pipe_instret_o = exu_queue_vd & exu_rdy;   // 指令退休（无论有无异常）
exu2pipe_exu_busy_o = exu_queue_vd & ~exu_rdy; // EXU 忙（多周期指令）
exu2idu_rdy_o = exu_rdy & ~exu_queue_barrier;  // 是否可收下一条
```

`exu_rdy`（`:798-807`）在 LSU 请求时取决于 `lsu_rdy`，RVM 操作时取决于 `ialu_rdy`——多周期指令期间 `exu2idu_rdy_o=0`，流水线被冻结（stall）。

---

## 第 6 课：控制状态寄存器 `scr1_pipe_csr.sv` 与 Trap 机制

CSR 模块是 CPU 的"软件视图"。主线：**trap（异常+中断）的进出流程**。

### 寄存器分类总览

| 类别 | 寄存器 | 作用 |
|---|---|---|
| 信息（只读） | MVENDORID/MARCHID/MIMPID/MHARTID | 厂商、实现 ID、hart 号 |
| Trap Setup | MSTATUS、MISA、MIE、MTVEC | 中断使能、trap 入口地址 |
| Trap Handling | MSCRATCH、MEPC、MCAUSE、MTVAL、MIP | 保存 trap 现场 |
| 计数器 | MCYCLE、MINSTRET（64 位） | 周期数、已退休指令数 |
| 非标准 | MCOUNTEN | 计数器使能控制 |

### 事件：trap 的三种来源（`:282-297`）

```
e_exc  = exu2csr_take_exc_i    // 异常（指令自身导致的错误）
e_irq  = exu2csr_take_irq_i    // 中断（外部异步请求）
e_mret = exu2csr_mret_update_i // MRET 指令（从 trap 返回）
```

异常优先于中断（`e_irq = take_irq & ~take_exc`）。SVA 断言 `$onehot0({e_irq, e_exc, e_mret})` 确保三者互斥。

### 中断使能的"两级闸门"（重点）

```
csr2exu_ip_ie_o = csr_eirq_pnd_en | csr_sirq_pnd_en | csr_tirq_pnd_en;  // 有pending且本地使能
csr2exu_irq_o   = csr2exu_ip_ie_o & csr_mstatus_mie_ff;                 // 全局使能
```

- 每一路：`MIP.pending & MIE.enable`（如 `csr_eirq_pnd_en = csr_mip_meip & csr_mie_meie_ff`）
- 总开关：`MSTATUS.MIE`（全局中断使能位）

进入中断时自动关总闸，防止嵌套。

### Trap 发生时 CSR 自动做的事（`:627-646` 关键）

```
case (1'b1)
    e_exc, e_irq:   // 进入 trap
        MIE  = 0;         // 关闭全局中断（防止嵌套）
        MPIE = 旧MIE;     // 保存旧值
    e_mret:         // 从 trap 返回
        MIE  = MPIE;      // 恢复中断
        MPIE = 1;
    ...
```

同时并行写入：
- **MEPC** ← 出错的指令地址（异常）或被中断的指令地址（中断）
- **MCAUSE** ← 原因码：中断时最高位置 1 + 中断源编号（外部/软件/定时器）
- **MTVAL** ← 附加信息（异常时是 trap 值，中断时清零）

一个异常四个动作（关中断、存 PC、存原因、存细节），硬件一个周期内完成。

### New PC：trap 跳到哪（`:1003-1019`）

MRET 跳回 `csr_mepc`；否则看 MTVEC 的 mode：

- **vectored（向量模式）**：异常跳 `mtvec_base`，外部中断跳 `base+11*4`，软件中断 `base+3*4`，定时器 `base+7*4`
- **direct（直接模式）**：全部跳 `mtvec_base`，handler 自己查 cause

### CSR 读写接口

`:474-481` 实现 CSRRW/CSRRS/CSRRC 三种写语义：

```
WRITE: w_data = 写入值           // CSRRW 直接覆盖
SET  : w_data = 写入值 | 旧值    // CSRRS 置位
CLEAR: w_data = ~写入值 & 旧值   // CSRRC 清位
```

读写未实现的地址会置 `csr_r_exc`/`csr_w_exc`，最终变成非法指令异常（`:984-991`）。

### 计数器：MCYCLE / MINSTRET

`csr_mcycle` 每周期 +1，`csr_minstret` 每退休一条**无异常**指令 +1（`instret_no_exc`）。

### 中断完整生命周期

```
外部拉高 → MIP 置位 → IP&IE 两级门 → EXU 察觉中断 → CSR 保存现场
→ PC 跳到 MTVEC → handler 运行 → MRET 恢复现场
```

---

## 第 7 课：访存通路（LSU + 存储系统）

### 内存接口定义（`scr1_memif.svh`）

LSU 与 DMEM 之间用统一的"命令/宽度/响应"三元组通信：

| 类型 | 取值 |
|---|---|
| `type_scr1_mem_cmd_e` | RD=0, WR=1 |
| `type_scr1_mem_width_e` | BYTE/HWORD/WORD |
| `type_scr1_mem_resp_e` | NOTRDY / RDY_OK / RDY_ER |

这套接口贯穿 core、router、TCM、AHB bridge 全部层级。

### LSU（`scr1_pipe_lsu.sv`）

**FSM**（`:67-191`）：只有 IDLE / BUSY 两态。

```
IDLE: 收到请求（dmem_req_vd）→ BUSY
BUSY: 收到响应（dmem_resp_received）→ IDLE
```

**地址对齐检查**（`:206-209`）：halfword 必须 2 字节对齐，word 必须 4 字节对齐：

```
dmem_addr_mslgn = (dmem_wdth_hword & addr[0]) | (dmem_wdth_word & |addr[1:0]);
```

**异常码**（`:212-224`）：分 load/store 分别报 `LD/ST_ADDR_MISALIGN` 或 `LD/ST_ACCESS_FAULT`。

**读数据符号扩展**（`:240-248`）：`LB`/`LH` 带符号扩展，`LBU`/`LHU` 零扩展。

**关键信号**：`lsu2exu_rdy_o = dmem_resp_received`（`:236`），即 EXU 的 `exu_rdy` 在访存时以 LSU 收到响应为准——这就是第 5 课"LOAD 是多周期指令"的物理来源。

### 集群存储层次（`scr1_top_ahb.sv`）

访存请求从 core 出发经过的路径：

```
core (IFU/LSU) → Router → 按地址译码分发到:
                       ├─ TCM（紧耦合内存，快速）
                       ├─ Memory-mapped Timer
                       └─ AHB bridge → 外部 AHB 总线（IMEM/DMEM 各一个主端口）
```

### DMEM Router（`scr1_dmem_router.sv`）

核心是**地址译码**（`:88-95`）：

```
if ((addr & PORT1_MASK) == PORT1_PATTERN) → PORT1 (TCM)
else if ((addr & PORT2_MASK) == PORT2_PATTERN) → PORT2 (Timer)
else → PORT0 (AHB bridge)
```

地址掩码/模式参数在 `scr1_arch_description.svh` 定义（TCM 在 `0x00480000`，Timer 在 `0x00490000`）。

Router 还有**两态 FSM**（`ADDR`/`DATA`，`:97-130`）：ADDR 态选端口并锁存 `port_sel_r`，DATA 态等所选端口响应。注意 ADDR 与 DATA 之间有一个寄存器的延迟——桥接到慢速总线（AHB）时靠这个流水化。

### TCM（`scr1_tcm.sv`）

TCM 是核心本地的高速内存，单周期访问，`i_tcm` 在 `scr1_top_ahb.sv:289` 实例化，尺寸由 `SCR1_TCM_ADDR_MASK` 取反决定。

### 两种总线变体

SCR1 提供 AHB（`scr1_top_ahb.sv`）和 AXI（`scr1_top_axi.sv`）两种顶层，通过 Makefile 的 `BUS=AHB/AXI` 选择，但 core 内部接口完全一致——差异只在 bridge 层。

### 访存完整路径总结

```
LSU 发出请求 → Router 按地址选端口 → TCM 单周期响应 或 Timer/AHB 慢速响应
→ 响应沿原路返回 → LSU 做符号扩展 → EXU 写回寄存器
```

---

## 第 8 课：多端口寄存器堆 `scr1_pipe_mprf.sv`

MPRF 是那 32 个 32 位寄存器（RV32E 时 16 个），167 行，核心模块中最简单。

### 接口：2 读 1 写

指令最多同时读 rs1、rs2，写 rd，所以是 2 读 1 写三端口结构。

### x0 恒为 0 的硬件实现（`:71-74`）

```
rs1_addr_vd = |exu2mprf_rs1_addr_i;   // 地址非零才算有效读
wr_req_vd   = exu2mprf_w_req_i & |exu2mprf_rd_addr_i;  // 写 x0 直接忽略
```

地址 0（x0）读返回 0、写被丢弃。寄存器堆数组从下标 1 开始（`:63`）。

### 两套实现

- **分布式逻辑实现**（`:127-152`，默认）：触发器构成，**异步读**，代码最直白。
- **RAM 实现**（`:90-126`，Intel FPGA）：片上 RAM 块（M9K/M10K），**同步读**（地址打一拍出数据），EXU 侧配合打拍。

### RAM 实现的"写读冲突"旁路（重点）

同步读意味着同周期写 rd、读 rs1=rd 读到旧值。SCR1 检测"读地址==写地址"（`rs1_new_data_req`，`:78`）后前向旁路：

```
mprf2exu_rs1_data_o = (rs1_new_data_req_ff) ? rd_data_ff   // 冲突时旁路写数据
                    : (rs1_addr_vd_ff)      ? rs1_data_ff   // 正常同步读
                                            : '0;
```

### 为什么 RAM 实现要两块内存

一个周期内可能 2 读 + 1 写 = 3 个同时独立操作，单端口 RAM 做不到，用两块简单双口 RAM 各服务一个读端口，写端口同时写两块。

---

## 核心模块全景图

```
取指(IFU) → 译码(IDU) → 执行(EXU) → 写回(MPRF)
   ↑                            ↓
   └──── PC 更新 ◀──── CSR(中断/异常) ◀──── LSU(访存)
```

六模块职责速查：

| 模块 | 文件 | 一句话职责 |
|---|---|---|
| IFU | `scr1_pipe_ifu.sv` | 取指 + 指令队列 + RVC 对齐 |
| IDU | `scr1_pipe_idu.sv` | 指令译码 → 生成执行命令 |
| EXU | `scr1_pipe_exu.sv` | ALU 运算 + 异常 + PC 控制 |
| MPRF | `scr1_pipe_mprf.sv` | 2 读 1 写寄存器堆 |
| CSR | `scr1_pipe_csr.sv` | 特权状态 + trap 进出 |
| LSU | `scr1_pipe_lsu.sv` | 访存 + 地址对齐检查 |

---

## 第 9 课：动手跑通仿真（hello 程序）

### 工具链与环境

| 组件 | 来源 | 版本 |
|---|---|---|
| Verilator | `apt install verilator` | 5.006 |
| RISC-V GCC | xPack 预编译版（`riscv-none-elf-gcc`） | 15.2.0 |

xPack 工具链下载：`github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases`，解压后 `bin/` 加入 PATH。multilib 覆盖 rv32e/ec/im/imc 等，`--print-multi-lib` 可查。

### 编译 hello（RV32EC / MIN 配置）

```
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  EXT_CFLAGS=-D__RVE_EXT \
  ARCH=ec ABI=ilp32e TCM=1 \
  inc_dir=sim/tests/common bld_dir=<bld_dir> \
  ADD_LDFLAGS="-lc -lnosys -Wl,--defsym,end=_end"
```

三个关键坑（新版工具链特有）：

1. **RV32E 需 `-D__RVE_EXT`**：crt_tcm.S 据此只保存/恢复 x1-x15（`:28-31`），否则 `sw x31,31*4(sp)` 非法指令。
2. **链接顺序**：`-lc -lgcc` 中 libgcc(emutls) 引用的 malloc/memcpy 在 libc 之后，需在末尾追加 `-lc`。
3. **`end` 符号缺失**：libnosys 的 `_sbrk` 引用 `end`，而 link_tcm.ld 只定义 `_end`（`:100`），用 `-Wl,--defsym,end=_end` 别名解决。

### Verilator 构建与仿真

```
make -C sim build_verilator \
  root_dir=/workspace bld_dir=<bld_dir> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32EC_MIN \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb

printf 'hello.hex\n' > <bld_dir>/test_info   # tb 逐行加载
./verilator/Vscr1_top_tb_ahb +test_info=./test_info +test_results=./test_results.txt
```

输出：

```
---Test: hello.hex
Hello from SCR1!
Test passed
Summary: 1/1 tests passed
```

### 程序执行路径（印证前八课）

`hello.elf` 符号布局（readelf 确认）：

| 符号 | 地址 | 说明 |
|---|---|---|
| `_start` | 0x200 | 复位后入口，链接脚本 `.text.init` |
| `main` | 0x4812EC | 落在 TCM（0x480000 区域） |
| `sc_exit` | 0x482260 | 收尾退出 |
| `SIM_EXIT` | 0xF4 | 链接脚本 `SIM_EXIT = 0x100-12` |
| `SCR1_SIM_EXIT_ADDR` | 0xF8 | tb 判定程序结束的地址 |

- **链接脚本双区**：`.text.init`（含 crt）在 RAM(0x0)，`.text/.data/.bss` 在 TCM(0x480000) 且 `AT>RAM`（加载到 RAM、运行在 TCM，crt 负责拷贝）。这正是第 7 课 imem_router 地址译码的体现。
- **测试判定**：tb 监测 `i_pipe_top.curr_pc == SCR1_SIM_EXIT_ADDR`；对非 compliance/arch 测试，通过条件为 `mprf_int[10]==0`（即 main 返回 0，a0==0）。
- **sc_printf 输出**：通过 UART 外设映射写出，tb 捕获打印，验证了 LSU→外设通路。

---

## 第 9 课补充：MIN vs MAX 配置实测对比

同一 hello 程序在两种配置下均跑通（`Hello from SCR1!` / `1/1 passed`）。

### RTL 配置差异（宏层面，scr1_arch_description.svh）

| 特性 | MIN（RV32EC） | MAX（RV32IMC） |
|---|---|---|
| 整数寄存器 | `SCR1_RVE_EXT`（16 个） | `SCR1_RVI_EXT`（32 个） |
| M 扩展 | 无 | `SCR1_RVM_EXT` + `SCR1_FAST_MUL` |
| 流水线级间寄存器 | `SCR1_NO_DEC_STAGE` + `SCR1_NO_EXE_STAGE`（约 2 级） | 完整 IFU/IDU/EXU 级（2~4 级） |
| MTVEC | 只读 0 位，仅 direct 模式 | 可写 26 位 + `SCR1_MTVEC_MODE_EN`（vectored） |
| IPIC | 无 | `SCR1_IPIC_EN` + `SCR1_IPIC_SYNC_EN` |
| 调试子系统 | 无 | `SCR1_DBG_EN` + `SCR1_TDU_EN`(4 触发器) + `SCR1_MCOUNTEN_EN` |
| 共有 | RVC、TCM、MPRF_RST_EN | 同左 |

**关键认识**：SCR1"2~4 级流水线"由配置决定——MIN 用 `NO_DEC/NO_EXE_STAGE` 摘掉级间寄存器退化为约 2 级；MAX 完整流水线寄存器实现 4 级。这也是 IDU/EXU 代码里大量"组合直通 + 条件打拍"写法的原因。

### 实测数据

| 指标 | MIN | MAX | 差异 |
|---|---|---|---|
| hello.hex | 32493 B | 31023 B | MAX 反而更小（RVE 16 寄存器需更多溢出指令） |
| hello.elf | 23360 B | 22500 B | 同上 |
| Verilator 模型二进制 | 198 KB | 347 KB | +75% |
| 生成 C++ 目标文件 | 29 个 | 52 个 | +80% |
| Verilator 编译时间 | ~30 s | ~30 s | 2 核机器上相当 |

---

## 第 9 课补充：VCD 波形观察指令流

用 `build_verilator_wf`（带 `--trace`）生成 `simx.vcd`（41MB），脚本解析后完整重建执行序列。这是第一次"看见"流水线实际行为。

### 关键信号（Verilator 扁平命名，VCD id 需从 `$var` 声明提取）

| 信号 | 含义 |
|---|---|
| `curr_pc` | EXU 当前执行 PC |
| `pipe2imem_addr_o` | IFU 取指地址（领先执行约 3 拍） |
| `inc_pc` | 顺序下一 PC（+4/+2） |
| `jb_new_pc` | 分支/跳转目标 |
| `new_pc` | 最终写入 IFU 的 PC |

### 观测结论

1. **复位向量**：复位后 `curr_pc = 0x200`（`SCR1_ARCH_RST_VECTOR`）。
2. **流水线预热**：启动时 `imem_addr` 先动（0x200→0x204→0x208），`curr_pc` 滞后 3 拍才开始逐条推进——IFU 提前取指填充指令队列。
3. **顺序执行 +4**，RVC 压缩指令 **+2**（如 `0x29c` 的 `j`、`0x2a2` 的 `addi` 都是 2 字节）。
4. **分支可见**：`jb_new_pc` 突变即跳转；脚本按 `curr_pc` 增量非 2/4 自动标记 `branch/jump`。
5. **启动序列实证**（对应 crt_tcm.S + link_tcm.ld）：
   - `0x200-0x278`：`zero_int_regs` 把 x1-x31 全部 `li 0`（每条 1 周期，无 stall）
   - `0x27c-0x290`：`auipc/addi` 组装 `__global_pointer$`、`printf_putch(0x480000)`
   - `0x298-0x2a4`：`beq/j/bne` + `lw/sw/addi` 数据搬运循环——把 `.data` 从 RAM 加载区搬到 TCM 运行区（`AT>RAM` 机制）
6. **程序退出**：`...→0x482008→0xf4(SIM_EXIT)→0xf8(SIM_STOP)`，tb 捕获 `curr_pc==0xf8` 判停。

### 实测数据（MAX 配置，hello）

| 指标 | 值 |
|---|---|
| 执行指令数 | 12643（含循环重复） |
| 总周期 | ~22770 |
| 平均 CPI | ~1.8（含复位/预热/退出） |
| VCD 体积 | 41MB |

---

## 第 3 课重讲（深入版）：IFU 逐段拆解

> 对原第 3 课速览版的补充，全部对照 `src/core/pipeline/scr1_pipe_ifu.sv`（826 行）源码行号。

### 配置前提（为什么走"完整队列"路径）

IFU 源码里有大量 `ifdef`，分支走向由配置宏决定：

- `SCR1_NO_DEC_STAGE`：只在 `SCR1_CFG_RV32IC_BASE`/`SCR1_CFG_RV32EC_MIN` 中定义。**MAX（RV32IMC）未定义** → 走 `:715` 的 `else` 分支（完整指令队列，无 bypass）。
- `SCR1_NEW_PC_REG`：只在 custom 段定义（`:136`）。**MAX 未定义** → 走 `ifndef` 分支（`:528-531`/`:596-601`），new_pc 请求直接驱动请求线并绕过 FSM。

### 为什么队列按"半字"存

纯 RV32I 不需要队列按半字存：指令全 32 位且 4 字节对齐，一次取指正好一条。加 RVC 后指令长度可变，**指令边界不再与取指块边界对齐**，消费粒度变成半字：

- 半字是 RVC（1 槽）、RVI（2 槽）、取指块（2 槽）三者的公共最小单位；
- 入队写宽度 `WR_HI`/`WR_FULL`（`:340-373`）、出队读宽度 `RD_HWORD`/`RD_WORD`（`:328-337`）、读指针步进 `+1/+2`（`:409-411`）都以半字计数；
- **跨块 RVI 是半字存储的必然性根源**：32 位指令低半落在上一块高半槽、高半落在下一块低半槽，两个半字隔一次取指延迟分别到达，队列按半字存才能在第二个半字到齐后拼回完整指令（`:746-747`）。

### 核心判据：`[1:0]==11 ⇔ RVI 起点`

判定"某半字是否为 32 位指令的低 16 位"用最低两位（`:279-280`）：

```systemverilog
assign instr_lo_is_rvi = &imem2ifu_rdata_i[1:0];     // 低半字 [1:0]==11
assign instr_hi_is_rvi = &imem2ifu_rdata_i[17:16];   // 高半字 [1:0]==11
```

规范出处：RISC-V **Unprivileged ISA Manual**，Introduction 章 "Base Instruction-Length Encoding"（`riscv-isa-manual/src/unpriv/intro.adoc`）。原话要点：32 位指令低两位恒 `11`；16 位压缩指令低两位只取 `00/01/10`。全 0/全 1 编码保留为非法（捕获错误跳转/未编程内存）。

**关键澄清（易混点）**：

- "合法压缩指令 ⇔ 低 2 位 ∈ {00,01,10}"是干净双射，硬件可信任；
- 但"低 2 位 == 11"只能推出"不是合法压缩指令"，**不能推出"一定是 32 位指令"**——可能是 32 位指令低半，也可能是数据/非法区（如 `0xffff` 填充）。SCR1 把 `[1:0]==11` 的半字按 RVI 处理，依赖"取指流里都是有效指令"这一层（链接脚本+合法程序保证），而非编码规则保证。
- 因此准确表述：`[1:0]==11 ⇔ RVI` 是**规范主动预留**（编码分配决策），不是"11 恰好没被 RVC 占用"的巧合。

**实测佐证**（MAX hello.elf 反汇编，1635 条真压缩指令低 2 位分布）：

```
00=371  01=620  10=644  11=0（真指令）
```

低 2 位为 11 的 5 个"16 位编码"全部是指令流外的数据：`fc/fe` 的 `.insn 2,0xffff`（向量区填充）、`20/2e/3a` 的 `.insn 6`（字符串常量）。

### 指令类型译码（instr_type，`:282-302`）

取回字 `{rdata[31:16]=高半字H, rdata[15:0]=低半字L}`，枚举命名 `SCR1_IFU_INSTR_<高半内容>_<低半内容>`：

| `{hi_is_rvi,lo_is_rvi}` | 含义 | instr_type |
|---|---|---|
| `00` | L、H 都不是 RVI 开头 | `RVC_RVC`（两个 16 位指令） |
| `10` | H 是某 RVI 的低半 | `RVI_LO_RVC`（L 完整 RVC + H 是跨块 RVI 低半） |
| `01`/`11` | L 是 RVI 低半 | `RVI_HI_RVI_LO`（完整 32 位 RVI，忽略 H 的 `[1:0]`） |

加两个 NV 态（`RVC_NV`/`RVI_LO_NV`，`:125-126`）覆盖非对齐 new_pc 后只认高半字的情形。

### 跨块拼接状态：`instr_hi_rvi_lo_ff`（`:148`, `:308-322`）

寄存器存的是"上一块欠没欠账"：置位 = 上一块高半字槽放了一条 RVI 的低 16 位，还没拼完。置位条件看 `next`（`:320-322`），本质是**命名以 `RVI_LO` 开头的三种 instr_type 都置位**：

```systemverilog
instr_hi_rvi_lo_next = (instr_type==RVI_LO_NV) | (instr_type==RVI_LO_RVI_HI) | (instr_type==RVI_LO_RVC);
```

含义：本块把 RVI 低半放进了高半字槽 → 该指令从块中部（2 对齐）开始、跨块边界，块内拼不完，置位让下一块用低半字槽来还账。下一块 decode 看 `ff`（`:290-293`）把本块低半字当 `RVI_HI` 而非新指令（产出 `RVC_RVI_HI`/`RVI_LO_RVI_HI`）。

反例：`RVI_HI_RVI_LO`（4 对齐完整 RVI）高半槽是 `RVI_HI`，当场拼完，不置位。

### 非对齐 new_pc：`new_pc_unaligned_ff`（`:262-274`）

分支/跳转目标可 2 字节对齐（bit1=1、bit0=0），而 IMEM 取指总是 4 字节块。new_pc 落在块中间时块低半字是废数据须丢弃：

```systemverilog
assign new_pc_unaligned_next = exu2ifu_pc_new_req_i ? exu2ifu_pc_new_i[1] : ...;  // :272-274
```

### 输出多路选择器（MAX 分支，`:745-753`）

```systemverilog
ifu2idu_instr_o = q_head_is_rvc ? q_data_head : {q_data_next, q_data_head};
```

RVI 在队里低半先入（地址小）、高半相邻，读两口拼回 `{next, head}` 即完整指令（little-endian 指令字）。

### IFU FSM：真·两态（`:91-94`, `:477-488`）

`IDLE` 等 `exu2ifu_pc_new_req_i`；`FETCH` 持续发读请求直到 `pipe2ifu_stop_fetch_i` 或错误待丢弃。MAX（无 `SCR1_NEW_PC_REG`）下 new_pc 请求绕过 FSM 直接驱动请求线（`:597`），地址由新 PC/取指地址二选一（`:599-601`）。

### IMEM 接口：两个计数器撑起"乱序响应"

- `imem_pnd_txns_cnt`（未完成事务，`:553`）：握手完成 `+1`、收响应 `-1`，全满（`:554`）停发请求。
- `imem_resp_discard_cnt`（待丢弃响应，`:569-591`）：两种丢弃来源——(1) new_pc 到来作废所有在飞指令；(2) 收到 IMEM 错误（无法保证后续字有效）。计数 >0 → `imem_resp_discard_req`（`:591`）拉高，返回指令只丢弃不入队（`:342`）。
- 有效在飞 = `pnd - discard`（`:590`），决定队列还能收几个字。

### 错误注入队列（精确异常的关键，`:419-444`）

**IMEM 出错时错误被"伪装成一条指令"注入队列**，而非 IFU 当场停住报错：

- ER 响应照样写满 2 槽（`:369-370`），数据是垃圾，但 `q_err[]` 与槽位绑定（`:429/:433`）一起流动；
- 出队时错误转为 `ifu2idu_imem_err_o`（MAX 分支 `:724-733`），注意 `q_err_next` 也参与——RVI 高半槽错也算整条指令错，另有 `ifu2idu_err_rvi_hi_o`（`:731`）单独标记；
- IDU 收 `imem_err_i` 不译码，直接生成 `exc_code=INSTR_ACCESS_FAULT(=1)` 命令（`scr1_pipe_idu.sv:134-137`），经 EXU 触发 CSR trap，`MCAUSE=1`、`MEPC=出错指令地址`，以异常身份退休。

**为什么必须入队**：RISC-V 要求异常精确且绑定具体指令。但 IMEM 出错时队列里可能已囤多条正常指令、出错响应也可能多条在飞，IFU 无法预知哪条指令"该背"这个错。真正能按序判定的是 IDU/EXU，所以错误作为一条指令的属性排进队列、逐槽流动，到 IDU/EXU 才翻牌成异常——异常码、出错 PC、顺序天然精确。

两条腿走路：取指侧立即踩刹车（`imem_resp_er_discard_pnd` 触发 `ifu_stop_req`，FSM 回 `IDLE`，丢弃计数器清掉错误之后的指令）；错误本身这条留在队列走向 IDU 成为唯一异常指令。这也是队列容量要够、错误必须占槽的原因——它占住"顺序里那个出错的位置"。

### SVA 断言（`:769-822`，免费的设计意图文档）

`IFU_DRC_UNDERFLOW/RANGE`（丢弃计数不越界）、`IFU_QUEUE_OVF`（写满保护）、`IFU_IMEM_ERR_BEH`（ER 后进 IDLE 且丢弃计数收敛）、`IFU_NEW_PC_REQ_BEH`（换 PC 后队列必空）、`IFU_IMEM_ADDR_ALIGNED`（发地址 `[1:0]==0`）。

### 与 VCD 观测互证

`pipe2imem_addr_o` 领先 `curr_pc` 约 3 拍 = IFU 在 `FETCH` 态持续预取、指令囤进 4 槽队列；crt 中 `j`/`addi` 步长 2 与 4 交替 = `RD_HWORD`/`RD_WORD` 交替消费的波形表现。

### 配套交付物（docs/scr1-pipe-ifu/）

IFU 单独成文的两份文档，与本节互为印证：

- 设计规格：`docs/scr1-pipe-ifu/design-spec.md`（接口/微架构/时序/断言/配置宏影响）
- 验证文档：`docs/scr1-pipe-ifu/verification.md`（覆盖点/断言核对/回归流程/结论）
- 图形化文档：`docs/scr1-ifu-graphics/index.html`（9 张 Mermaid 图：模块视图、FSM、类型译码树、环形队列、跨块拼接、IMEM 时序等）

### 配套交付物（docs/scr1-pipe-idu/）

IDU 单独成文的两份文档：

- 设计规格：`docs/scr1-pipe-idu/design-spec.md`（接口/4 层译码/指令总表/立即数重组/配置宏影响）
- 验证文档：`docs/scr1-pipe-idu/verification.md`（FC-1~26 覆盖点/断言核对/回归流程/结论）

### 配套交付物（docs/scr1-pipe-exu/）

EXU 单独成文的两份文档：

- 设计规格：`docs/scr1-pipe-exu/design-spec.md`（接口/队列/IALU/异常/WFI/PC/LSU/CSR/配置宏）
- 验证文档：`docs/scr1-pipe-exu/verification.md`（FC-EXU-1~33 覆盖点/断言核对/回归流程/结论）

### 配套交付物（docs/scr1-pipe-lsu/）

LSU 单独成文的两份文档：

- 设计规格：`docs/scr1-pipe-lsu/design-spec.md`（接口/FSM/命令寄存器/异常/数据扩展/TDU）
- 验证文档：`docs/scr1-pipe-lsu/verification.md`（FC-LSU-1~27 覆盖点/断言核对/回归流程/结论）

### 配套交付物（docs/scr1-pipe-ialu/）

IALU 单独成文的两份文档：

- 设计规格：`docs/scr1-pipe-ialu/design-spec.md`（主加法器/地址加法器/移位/MDU 乘除/结果 mux）
- 验证文档：`docs/scr1-pipe-ialu/verification.md`（FC-IALU-1~30 覆盖点/断言核对/回归流程/结论）

### 配套交付物（docs/scr1-pipe-mprf/）

MPRF 单独成文的两份文档：

- 设计规格：`docs/scr1-pipe-mprf/design-spec.md`（2 读 1 写/x0/分布式 vs RAM/写优先）
- 验证文档：`docs/scr1-pipe-mprf/verification.md`（FC-MPRF-1~15 覆盖点/断言核对/回归流程/结论）

### 配套交付物（docs/scr1-pipe-csr/）

CSR 单独成文的两份文档：

- 设计规格：`docs/scr1-pipe-csr/design-spec.md`（CSR 读写译码/trap 事件/MSTATUS~MIP/计数器/IPIC·HDU·TDU 桥接/New PC）
- 验证文档：`docs/scr1-pipe-csr/verification.md`（FC-CSR-1~42 覆盖点/14 断言核对/回归流程/结论）

---

*笔记持续更新中。*
