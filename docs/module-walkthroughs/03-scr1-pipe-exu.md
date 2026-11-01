# scr1_pipe_exu.sv 讲解

## 一句话职责

执行单元：CPU 核心，IALU 运算 + 异常汇集 + PC 控制（分支/跳转/中断/异常）。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_exu.sv`
- **行数**: 1086 行（MAX 配置）
- **配置依赖**: `SCR1_RVM_EXT`、`SCR1_FAST_MUL`、`SCR1_NO_EXE_STAGE`

## 六大功能块

### 1. 指令队列（`:339-370`）

默认配置下 IDU 和 EXU 之间**有一级寄存器**（`:339-370`），在 `exu_queue_en` 时锁存 `idu2exu_cmd_i`：

```systemverilog
exu_queue_en = exu2idu_rdy_o & idu2exu_req_i;   // :318
```

之后 EXU 所有逻辑读的都是打拍后的 `exu_queue`。

**作用**：
- 组合逻辑隔离：IDU 是纯组合，EXU 也是大组合，中间打一拍避免时序收敛困难
- 气泡插入：多周期指令期间 `exu2idu_rdy_o=0`，队列holding 住指令

### 2. IALU：双加法器通路（`:445`）

实例化 `scr1_pipe_ialu`，内部有**两条独立加法器通路**：

| 通路 | 操作数 | 结果 | 用途 |
|---|---|---|---|
| `ialu_main` | `op1/op2` | `ialu_main_res` | 算术/逻辑/比较 |
| `ialu_addr` | `op1/op2` | `ialu_addr_res` | 地址计算（PC+imm 或 寄存器+imm） |

**为什么需要两条？** `beq` 指令要同时"比较两寄存器"并"算目标地址 PC+imm"，单加法器做不到。

### 3. 异常：三类来源汇集

```systemverilog
exu_exc_req = exu_queue_vd & (
    exu_queue.exc_req       // IDU 判定的异常（非法指令、ECALL、EBREAK）
  | lsu_exc_req             // 访存异常（地址错位、访问错误）
  | csr2exu_rw_exc_i        // CSR 读写异常
  | ...
);
```

异常码由 `exc_code` 编码器（`:514-528`）选择，`exc_trap_val`（`:545-575`）是附加信息——对非法指令会**重新拼出指令编码**（`:535-541`）放进 MTVAL。

### 4. PC 逻辑：CPU 的"方向舵"（重点，`:640-720`）

```systemverilog
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

```systemverilog
branch_taken = exu_queue.branch_req & ialu_cmp;   // :738
jb_taken     = exu_queue.jump_req | branch_taken; // :739
```

顺序指令不触发 `new_pc_req`，IFU 自己按 `inc_pc`（PC+4 或 +2）继续取。

### 5. 写回（`:881-890`）

`rd_wb_sel` 多路选择器：

| rd_wb_sel | 写回值 | 典型指令 |
|---|---|---|
| SUM2 | 地址加法器结果 | AUIPC、JAL 目标 |
| IMM | 立即数 | LUI、LI |
| INC_PC | PC+4/PC+2 | JAL/JALR 返回地址 |
| LSU | 访存读出的数据 | LW/LH/LB |
| CSR | CSR 读出的值 | CSRRW/CSRRS |
| 默认 | IALU 主运算结果 | ADD/SUB/XOR |

`exu2mprf_w_req_o`（`:872-876`）把关：**有异常就不写回**（`~exu_exc_req`）。

### 6. 三个关键"退休"信号

```systemverilog
exu2pipe_instret_o  = exu_queue_vd & exu_rdy;              // 指令退休（无论有无异常）
exu2pipe_exu_busy_o = exu_queue_vd & ~exu_rdy;             // EXU 忙（多周期指令）
exu2idu_rdy_o       = exu_rdy & ~exu_queue_barrier;        // 是否可收下一条
```

`exu_rdy`（`:798-807`）在 LSU 请求时取决于 `lsu_rdy`，RVM 操作时取决于 `ialu_rdy`——多周期指令期间 `exu2idu_rdy_o=0`，流水线被冻结（stall）。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `idu2exu_req_i` | in | IDU 有有效指令 |
| `exu2idu_rdy_o` | out | EXU 准备好收指令 |
| `exu2mprf_rs1_addr_o` | out | 读 rs1 寄存器号 |
| `mprf2exu_rs1_data_i` | in | rs1 寄存器值 |
| `exu2mprf_w_req_o` | out | 写回请求 |
| `exu2mprf_rd_addr_o` | out | 写回寄存器号 |
| `exu2mprf_rd_data_o` | out | 写回数据 |
| `exu2ifu_pc_new_req_o` | out | 换 PC 请求 |
| `exu2ifu_pc_new_o` | out | 新 PC 值 |
| `exu2csr_take_exc_i` | out | 触发异常 |
| `exu2csr_exc_code_o` | out | 异常码 |

## WFI 流程（`:600-639`）

WFI（Wait for Interrupt）是"停取等中断"指令：

1. EXU 收到 WFI → `wfi_halted_ff=1`（`:610`）
2. `pipe2ifu_stop_fetch_o=1`（`:615`），IFU 停取
3. `exu2idu_rdy_o=0`（`:616`），流水线冻结
4. CSR 检测到中断 → `csr2exu_ip_ie_i=1`（`:627`）
5. `wfi_run2halt_ff=1`（`:630`），唤醒
6. `pipe2ifu_stop_fetch_o=0`，恢复取指

## 配置宏影响

| 宏 | 行为变化 |
|---|---|
| `SCR1_NO_EXE_STAGE` | 无指令队列，IDU 直通 IALU |
| `SCR1_RVM_EXT` | MDU 乘除单元使能，`ialu_rdy` 变多周期 |
| `SCR1_FAST_MUL` | 乘法器组合实现（单周期）vs 移位累加（多周期） |

## 与其他模块关系

```
IDU → EXU: idu2exu_cmd_o（执行命令）
EXU ↔ MPRF: rs1/rs2 读，rd 写回
EXU ↔ IALU: 操作数输入，结果输出
EXU ↔ LSU: 访存请求/响应（lsu_rdy 决定 exu_rdy）
EXU ↔ CSR: 异常/中断/CSR 读写
EXU → IFU: new_pc_req + new_pc（换 PC）
```

## 配套文档

- 设计规格：`docs/scr1-pipe-exu/design-spec.md`
- 验证文档：`docs/scr1-pipe-exu/verification.md`
