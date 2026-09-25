# SCR1 EXU 执行单元设计规格书（Design Specification）

- 模块：`scr1_pipe_exu`
- 源文件：`src/core/pipeline/scr1_pipe_exu.sv`（1086 行）
- 位置：SCR1 RV32 内核流水线顶层 `scr1_pipe_top` 的第三级（IDU 之后，MPRF/CSR/LSU 的中心）
- 版本：基于 SCR1 开源版本（Syntacore），文档对应 2026-09 学习记录

> 本文档描述 EXU 的功能规格、接口、微架构、时序与验证边界，行号均指向源文件。
> 全文以 `SCR1_CFG_RV32IMC_MAX`（定义 `RVM/RVC/DBG/TDU/CLKCTRL`，不定义
> `SCR1_NO_EXE_STAGE`）为主线，配置相关的分支单独标注。

---

## 目录

1. 概述与设计目标
2. 外部接口
3. 内部结构（微架构）
4. 功能规格
5. 配置宏对行为的影响
6. 时序行为示例
7. 断言规格
8. 假设与约束
- 附录 A 端口清单
- 附录 B 局部参数/类型/信号
- 附录 C 异常码与 trap_val 映射
- 附录 D 一次异常/中断/MRET 的完整流程
- 附录 E 关键信号 → 消费者映射
- 附录 F 与 IFU/IDU 的职责边界

---

## 1. 概述与设计目标

EXU（Execution Unit，执行单元）是流水线的第三级，也是内核的"枢纽"。它负责：

1. **指令执行**：取操作数、经 IALU 计算、写回 MPRF；LOAD/STORE 经 LSU 访存；
2. **操作数取数**：向 MPRF 发 rs1/rs2 读地址（含时钟门控）；
3. **写回选择**：按 `rd_wb_sel` 从 IALU/SUM2/IMM/INC_PC/LSU/CSR 中选择写回数据；
4. **异常处理**：产生异常请求、编码异常码、计算 `trap_val`；
5. **中断/MRET**：向 CSR 发出取中断、取异常、MRET 事件；
6. **WFI 控制**：进入/退出 WFI 停机状态；
7. **PC 计算**：复位初始化 PC、维护当前 PC、计算 New PC 并请求 IFU 换 PC；
8. **调试/触发接口**：HDU（Debug halted/单步/程序缓冲）与 TDU（硬件断点）。

EXU 是**唯一产生 flow control 的流水级**：分支、跳转、异常、中断、MRET、FENCE.I、WFI 唤醒都通过 `exu2ifu_pc_new_req_o` / `exu2ifu_pc_new_o` 通知 IFU 换 PC（`:722-735`、`:707-720`）。IFU/IDU 自身从不产生 PC 变化。

### 1.1 设计权衡

- **可选级间寄存器（`SCR1_NO_EXE_STAGE`）**：默认有 IDU→EXU 寄存器，用于时序收敛与寄存器读；MIN 配置摘掉，EXU 命令组合直通，牺牲时序换取极简结构（`:306-383`）。
- **条件操作数字段锁存**：队列寄存器只锁存 `use_rs1/use_rs2/use_rd/use_imm` 为真的字段（`:358-369`），省去未用寄存器的翻转功耗。
- **PC 增量优化**：`pc_curr_next` 只在 bit6 翻转时才用完整加法结果，否则拼接高位（`:700-702`），省去高位加法器。
- **异常码/`trap_val` 分层编码**：异常请求是多个来源的或（`:483-493`），异常码用 `case(1'b1)` 优先级编码（`:514-529`），`trap_val` 再按异常码选择（`:545-575`），层次清晰且面积小。
- **WFI 用两根时钟域寄存器**：`wfi_run_start_ff` / `wfi_halted_ff` 在 `clk_alw_on`（不断门）上翻转（`:613`、`:632`），保证停机期间状态机仍能唤醒。
- **CSR 访问两态握手**：`INIT/RDY` 两态防止连续 CSR 写冲突（`:946-958`）。

### 1.2 术语与命名约定

| 术语 | 含义 |
|---|---|
| 指令队列（exu_queue） | IDU→EXU 的一级命令缓冲，`SCR1_NO_EXE_STAGE` 时被摘除 |
| barrier | 阻塞指令执行的复合条件（WFI/调试），`:311-316` |
| retire（退休） | 指令被 EXU 接受完成，`exu2pipe_instret_o` |
| IALU | 整数算术逻辑单元，含主结果与地址（SUM2）两路 |
| SUM2 | 第二加法器，计算跳转/分支目标、LOAD/STORE 地址、AUIPC |
| trap_val | 写入 `mtval` 的异常附带值 |
| New PC | EXU 请求 IFU 跳转到的目标地址 |
| csr_access_init | CSR 访问 FSM 处于 INIT 态（可发起新 CSR 访问） |

**命名规则**：`<生产者>2<消费者>_<信号名>_<o|i>`，如 `exu2ifu_pc_new_req_o`。

---

## 2. 外部接口（Port List）

共 60+ 个端口（含条件编译）。完整清单见附录 A，本节给出分组与语义。

### 2.1 公共与时钟

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `rst_n` | in | 复位（低有效） | 56 |
| `clk` | in | 门控 EXU 时钟 | 57 |
| `clk_alw_on` | in | 不门控时钟（仅 `SCR1_CLKCTRL_EN`） | 59 |
| `clk_pipe_en` | in | EXU 时钟使能（仅 `SCR1_CLKCTRL_EN`） | 60 |

### 2.2 IDU <-> EXU

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `idu2exu_req_i` | in | IDU 请求 | 64 |
| `exu2idu_rdy_o` | out | EXU 就绪 | 65 |
| `idu2exu_cmd_i` | in | 命令字 | 66 |
| `idu2exu_use_rs1_i/use_rs2_i` | in | 操作数门控 | 67-68 |
| `idu2exu_use_rd_i/use_imm_i` | in | 门控（仅无 `SCR1_NO_EXE_STAGE`） | 70-71 |

### 2.3 EXU <-> MPRF

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `exu2mprf_rs1_addr_o` / `mprf2exu_rs1_data_i` | out/in | rs1 读地址/数据 | 75-76 |
| `exu2mprf_rs2_addr_o` / `mprf2exu_rs2_data_i` | out/in | rs2 读地址/数据 | 77-78 |
| `exu2mprf_w_req_o` | out | 写请求 | 79 |
| `exu2mprf_rd_addr_o` / `exu2mprf_rd_data_o` | out | 写地址/数据 | 80-81 |

### 2.4 EXU <-> CSR

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `exu2csr_rw_addr_o` | out | CSR 地址 | 84 |
| `exu2csr_r_req_o` / `csr2exu_r_data_i` | out/in | 读请求/数据 | 85-86 |
| `exu2csr_w_req_o` / `exu2csr_w_cmd_o` / `exu2csr_w_data_o` | out | 写请求/命令/数据 | 87-89 |
| `csr2exu_rw_exc_i` | in | CSR 访问异常 | 90 |
| `exu2csr_take_irq_o` / `exu2csr_take_exc_o` | out | 取中断/异常 | 93-94 |
| `exu2csr_mret_update_o` / `exu2csr_mret_instr_o` | out | MRET 更新/指令 | 95-96 |
| `exu2csr_exc_code_o` / `exu2csr_trap_val_o` | out | 异常码/trap 值 | 97-98 |
| `csr2exu_new_pc_i` | in | 异常/中断/MRET 新 PC | 99 |
| `csr2exu_irq_i` / `csr2exu_ip_ie_i` | in | 中断请求 / 有中断使能 | 100-101 |
| `csr2exu_mstatus_mie_up_i` | in | MSTATUS/MIE 本拍更新 | 102 |

### 2.5 EXU <-> DMEM

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `exu2dmem_req_o` / `cmd_o` / `width_o` / `addr_o` / `wdata_o` | out | DMEM 请求/命令/宽度/地址/写数据 | 105-109 |
| `dmem2exu_req_ack_i` / `rdata_i` / `resp_i` | in | 应答/读数据/响应 | 110-112 |

### 2.6 EXU 控制与 PC

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `exu2pipe_exc_req_o` | out | 上条指令异常 | 115 |
| `exu2pipe_brkpt_o` | out | 软件断点（EBREAK） | 116 |
| `exu2pipe_init_pc_o` | out | 复位退出 | 117 |
| `exu2pipe_wfi_run2halt_o` | out | 进入 WFI 停机 | 118 |
| `exu2pipe_instret_o` | out | 指令退休 | 119 |
| `exu2csr_instret_no_exc_o` | out | 无异常退休（仅非 `REDUCED_CNT`） | 121 |
| `exu2pipe_exu_busy_o` | out | EXU 忙 | 123 |
| `exu2pipe_wfi_halted_o` | out | WFI 停机状态（仅 `CLKCTRL_EN`） | 156 |
| `exu2pipe_pc_curr_o` | out | 当前 PC | 158 |
| `exu2csr_pc_next_o` | out | 下一 PC（供 MEPC） | 159 |
| `exu2ifu_pc_new_req_o` / `exu2ifu_pc_new_o` | out | 换 PC 请求/数据 | 160-161 |

### 2.7 HDU 调试（`SCR1_DBG_EN`）与 TDU 触发（`SCR1_TDU_EN`）

- HDU 输入（`:127-136`）：`no_commit`、`irq_dsbl`、`pc_advmt_dsbl`、`dmode_sstep_en`、`pbuf_fetch`、`dbg_halted`、`dbg_run2halt`、`dbg_halt2run`、`dbg_run_start`、`dbg_new_pc`。
- TDU（`:141-151`）：指令监视 `exu2tdu_imon_o`、断点匹配/异常请求、数据监视、断点退休标志。

---

## 3. 内部结构（微架构）

### 3.1 结构框图

```
             idu2exu_cmd_i ──► [指令队列 exu_queue] ──┬─► IALU(主/地址) ──► rd_wb mux ──► MPRF
 idu2exu_req_i ──► [队列控制/valid] ──► exu_rdy ────┤
 mprf2exu_rs*_data_i ───────────────────────────────┤─► LSU ──► DMEM
                                                    ├─► 异常逻辑 ──► CSR(take_exc/trap_val)
                                                    ├─► WFI 逻辑 ──► CSR / pipe
                                                    └─► PC 逻辑 ──► IFU(new_pc_req/new_pc)
                                                                     └─► CSR(pc_next)
```

### 3.2 子块清单

| 子块 | 行号 | 说明 |
|---|---|---|
| 指令执行队列 | 291-383 | 命令缓冲 + valid 标志 + barrier |
| IALU | 385-465 | 操作数取数 + `scr1_pipe_ialu` 例化 |
| 异常逻辑 | 467-575 | 请求、优先级编码、trap_val |
| WFI 逻辑 | 577-649 | 进入/退出停机 |
| PC 逻辑 | 651-746 | 初始化/当前 PC/New PC |
| LSU | 748-791 | `scr1_pipe_lsu` 例化 |
| EXU 状态 | 793-830 | `exu_rdy`、退休、异常/断点输出 |
| MPRF 接口 | 832-890 | 取数阶段 + 写回阶段 |
| CSR 接口 | 892-993 | CSR 读写 + 访问 FSM + 事件 |
| TDU 接口 | 995-1014 | 指令监视/断点退休 |
| Tracelog/断言 | 1017-1084 | 仿真专用 |

### 3.3 流水阶段与打拍

EXU 内部有三处寄存器：

1. **指令队列寄存器**（`:339-371`，仅无 `SCR1_NO_EXE_STAGE`）——IDU→EXU 的级间寄存器；
2. **PC 寄存器 `pc_curr_ff`**（`:686-692`）——当前 PC；
3. **WFI/CSR 状态寄存器**（`:610-639`、`:946-952`）。

其余为组合逻辑。默认（MAX）下指令需经队列寄存器一拍才进入执行，因此 `exu_queue_vd` 表示"队首指令有效"。

---

## 4. 功能规格

### 4.1 指令执行队列（`:291-383`）

队列负责把 IDU 的命令锁存一拍并在 EXU 内消费。分两种实现：

**A. 有 EXU 级（`SCR1_NO_EXE_STAGE` 未定义，MAX）**

- `exu_queue_barrier`（`:311-316`）：
  `wfi_halted_ff | wfi_halt_req | wfi_run_start_ff | hdu2exu_dbg_halted_i | hdu2exu_dbg_run2halt_i | dbg_run_start_npbuf`。
  barrier 期间禁止新指令进入与当前指令退休。
- `exu_queue_en = exu2idu_rdy_o & idu2exu_req_i`（`:318`）——级间握手成功。
- valid 寄存器（`:325-331`）：更新条件 `exu_queue_vd_upd = exu_queue_barrier | exu_rdy`（`:323`），
  `exu_queue_vd_next = ~exu_queue_barrier & idu2exu_req_i & ~exu2ifu_pc_new_req_o`（`:333`）。
  **`~exu2ifu_pc_new_req_o` 很关键**：正在请求换 PC（flush）时不允许锁存新指令。
- 队列寄存器（`:339-371`）：`exu_queue_en` 时锁存全部命令字段；`use_rs1/2/rd/imm` 为真的字段才锁存（`:358-369`），否则保持旧值以省功耗。

**B. 无 EXU 级（`SCR1_NO_EXE_STAGE` 定义，MIN）**

- `exu_queue_barrier` 少了 `wfi_halt_req`（`:375-379`，因为没有队列寄存器，halt 请求不需阻塞队列）。
- 组合直通：`exu_queue_vd = idu2exu_req_i & ~exu_queue_barrier`（`:380`），
  `exu_queue = idu2exu_cmd_i`（`:381`）。
- 无 `exu_queue_vd_ff` / `idu2exu_use_*_ff`。

> 调试信号 `dbg_run_start_npbuf = hdu2exu_dbg_run_start_i & ~hdu2exu_pbuf_fetch_i`（`:303`）：
> 从程序缓冲取指时的 run_start 不算"非缓冲 run_start"。

### 4.2 IALU 与操作数取数（`:385-465`）

**主操作数（`:410-427`）**

| `ialu_op` | op1 | op2 |
|---|---|---|
| `REG_REG` | `mprf2exu_rs1_data_i` | `mprf2exu_rs2_data_i` |
| 其他（`REG_IMM`） | `mprf2exu_rs1_data_i` | `exu_queue.imm` |

含 `SCR1_RVM_EXT` 时，若 `~ialu_vd` 则强制 op1/op2='0（`:412-415`），避免无谓翻转。

`ialu_vd = exu_queue_vd & (ialu_cmd != NONE) & ~tdu2exu_ibrkpt_exc_req_i`（`:403-407`）——仅 M 扩展需要，用于多周期乘除。

**地址操作数（第二加法器 SUM2，`:432-440`）**

| `sum2_op` | op1 | op2 |
|---|---|---|
| `REG_IMM` | `mprf2exu_rs1_data_i` | `exu_queue.imm` |
| 其他（`PC_IMM`） | `pc_curr_ff` | `exu_queue.imm` |

**例化**（`:445-465`）：`scr1_pipe_ialu` 输出 `ialu_main_res`（主结果）、`ialu_cmp`（比较结果，供分支）、`ialu_addr_res`（地址/目标）。

### 4.3 异常逻辑（`:467-575`）

#### 4.3.1 异常请求（`:482-493`）

```systemverilog
exu_exc_req = exu_queue_vd & (exu_queue.exc_req | lsu_exc_req | csr2exu_rw_exc_i
                             | jb_misalign | exu2hdu_ibrkpt_hw_o);
```

- `jb_misalign = exu_queue_vd & jb_taken & |jb_new_pc[1:0]`（`:478-480`，仅无 `SCR1_RVC_EXT`）——纯 RVI 配置下，跳转/分支目标必须 4 字节对齐。
- `exu2hdu_ibrkpt_hw_o = tdu2exu_ibrkpt_exc_req_i | tdu2lsu_dbrkpt_exc_req_i`（`:828`，调试+触发）。

调试下另有 `exu_exc_req_ff` 寄存器（`:500-508`）：`exu_exc_req_next = hdu2exu_dbg_halt2run_i ? 1'b0 : exu_exc_req`，供 `~exu_queue_vd` 期间保持异常（`:819`）。

#### 4.3.2 异常码编码器（`:514-529`）

`case (1'b1)` 按**优先级**选择，靠前者优先：

| 优先级 | 条件 | exc_code | 行号 |
|---|---|---|---|
| 1 | `exu2hdu_ibrkpt_hw_o`（调试+触发） | BREAKPOINT | 518 |
| 2 | `exu_queue.exc_req` | `exu_queue.exc_code` | 521 |
| 3 | `lsu_exc_req` | `lsu_exc_code` | 522 |
| 4 | `csr2exu_rw_exc_i` | ILLEGAL_INSTR | 523 |
| 5 | `jb_misalign`（无 RVC） | INSTR_MISALIGN | 525 |
| default | — | ECALL_M | 527 |

> IDU 产生的 `ILLEGAL_INSTR`/`ECALL_M`/`BREAKPOINT` 走优先级 2（`exu_queue.exc_req`）；
> CSR 访问异常走优先级 4，且**统一编码为 ILLEGAL_INSTR**。

#### 4.3.3 trap_val 复用与重建（`:531-575`）

- `instr_fault_rvi_hi = exu_queue.instr_rvc`（`:534`）——IDU 复用 `instr_rvc` 表达"取指故障落在 RVI 高半字"（见 IDU 规格书 4.2）。
- `exu_illegal_instr`（`:535-541`）：把 CSR 访问异常涉及的状态**重建成 SYSTEM 指令编码**：
  `{csr_addr, rs1/zimm, imm[14:12](funct3), rd, OPCODE_SYSTEM, INSTR_RVI}`，作为 `mtval`。
- `exc_trap_val` 按 `exc_code` 选择（`:545-575`）：

| exc_code | trap_val | 行号 |
|---|---|---|
| INSTR_MISALIGN（无 RVC） | `jb_new_pc` | 548 |
| INSTR_ACCESS_FAULT | `instr_fault_rvi_hi ? inc_pc : pc_curr_ff` | 550-552 |
| ILLEGAL_INSTR（MTVAL 使能） | `exu_queue.exc_req ? exu_queue.imm : exu_illegal_instr` | 554-556 |
| ILLEGAL_INSTR（未使能） | `'0` | 558 |
| BREAKPOINT（TDU） | 指令断点→`pc_curr_ff`；数据断点→`ialu_addr_res` | 561-567 |
| LD/ST misalign/fault | `ialu_addr_res` | 569-572 |
| default | `'0` | 573 |

**取指故障高半字**：若故障在高半字，`trap_val = inc_pc`（= PC+2，指向高半字）；否则 `pc_curr_ff`。

### 4.4 WFI 逻辑（`:577-649`）

| 信号 | 表达式 | 行号 |
|---|---|---|
| `wfi_halt_cond` | `~csr2exu_ip_ie_i & ((exu_queue_vd & wfi_req) \| wfi_run_start_ff) & dbg guards` | 591-596 |
| `wfi_halt_req` | `~wfi_halted_ff & wfi_halt_cond` | 597 |
| `wfi_run_req` | `wfi_halted_ff & (csr2exu_ip_ie_i \| dbg_halt2run)` | 601-605 |
| `wfi_run_start_next` | `wfi_halted_ff & csr2exu_ip_ie_i & ~exu2csr_take_irq_o` | 622 |
| `wfi_halted_upd` | `wfi_halt_req \| wfi_run_req` | 627 |
| `wfi_halted_next` | `wfi_halt_req \| ~wfi_run_req` | 641 |

- 执行到 WFI（`wfi_req`）且**无"待处理且使能"的中断**（`~ip_ie`）时进入停机（`wfi_halt_req`）。
- 停机后一旦 `ip_ie` 有效或有调试 resume，`wfi_run_req` 退出停机。
- `wfi_run_start_ff` 是"退出停机后重新起动取指"的一拍标志，用于产生 New PC（见 4.5）并阻塞队列。
- `wfi_run_start_ff` / `wfi_halted_ff` 在 `SCR1_CLKCTRL_EN` 时用 `clk_alw_on`（`:613`、`:632`），保证被门控的 `clk` 停振时仍能靠不门控时钟更新状态。
- 输出：`exu2pipe_wfi_run2halt_o = wfi_halt_req`（`:646`）、`exu2pipe_wfi_halted_o = wfi_halted_ff`（`:648`）。

### 4.5 PC 逻辑（`:651-746`）

#### 4.5.1 初始化（`:660-674`）

- `init_pc_v` 是 4 位移位寄存器，复位为 0，之后每拍 `{init_pc_v[2:0], 1'b1}` 直到全 1 停止（`:668-670`）。
- `init_pc = ~init_pc_v[3] & init_pc_v[2]`（`:674`）——在 `init_pc_v == 4'b0111` 的那一拍拉高一次，
  产生 "New PC = 复位向量" 的一次性请求，驱动 IFU 从 `SCR1_RST_VECTOR` 起取指。

#### 4.5.2 当前 PC（`:676-702`）

| 信号 | 表达式 | 行号 |
|---|---|---|
| `pc_curr_upd` | `(exu2pipe_instret_o \| exu2csr_take_irq_o \| dbg_run_start_npbuf) & (dbg guards)` | 679-684 |
| `inc_pc` | `pc_curr_ff + (instr_rvc ? 2 : 4)`（无 RVC 恒 +4） | 694-698 |
| `pc_curr_next` | `new_pc_req ? new_pc : (inc_pc[6]^pc_curr_ff[6]) ? inc_pc : {pc_curr_ff[XLEN-1:6], inc_pc[5:0]}` | 700-702 |

- PC 只在**指令退休或有中断**时推进（`:679`）。
- bit6 进位优化：+2/+4 只会改变低 6 位；仅当 bit6 翻转时采用完整 `inc_pc`，否则保持高位不变、拼接低位，省去高位加法器。
- 复位值 `SCR1_RST_VECTOR`（`:688`，定义于 `scr1_csr.svh:98` = `SCR1_ARCH_RST_VECTOR`）。

#### 4.5.3 New PC 多路选择与请求（`:704-746`）

`exu2ifu_pc_new_o`（`:707-720`）：

| 优先级 | 条件 | New PC |
|---|---|---|
| 1 | `init_pc` | `SCR1_RST_VECTOR` |
| 2 | `exu2csr_take_exc_o` / `take_irq_o` / `mret_instr_o` | `csr2exu_new_pc_i` |
| 3 | `dbg_run_start_npbuf`（DBG） | `hdu2exu_dbg_new_pc_i` |
| 4 | `wfi_run_start_ff` | `pc_curr_ff` |
| 5 | `exu_queue.fencei_req` | `inc_pc` |
| default | 跳转/分支 | `ialu_addr_res & SCR1_JUMP_MASK` |

`exu2ifu_pc_new_req_o`（`:722-735`）= `init_pc | take_irq | take_exc | (mret_instr & ~mstatus_mie_up) | (vd & fencei_req) | (wfi_run_start_ff & clk_pipe_en) | dbg_run_start_npbuf | (vd & jb_taken)`。

- `SCR1_JUMP_MASK = XLEN'hFFFF_FFFE`（`:168`）清零 bit0，保证 2 字节对齐。
- `branch_taken = branch_req & ialu_cmp`（`:738`）；`jb_taken = jump_req | branch_taken`（`:739`）。
- MRET 请求 New PC 时要求 **MSTATUS/MIE 不在本拍更新**（`~mstatus_mie_up`，`:725`），避免与 CSR 更新冲突。
- 供 MEPC 的 `exu2csr_pc_next_o`（`:743-745`）：无指令→`pc_curr_ff`；跳转/分支→`jb_new_pc`；否则→`inc_pc`。
- `exu2pipe_pc_curr_o = pc_curr_ff`（`:746`）。

### 4.6 LSU（`:748-791`）

- `lsu_req = (lsu_cmd != NONE) & exu_queue_vd`（`:759`）。
- 例化 `scr1_pipe_lsu`（`:761-791`）：地址 `ialu_addr_res`、写数据 `mprf2exu_rs2_data_i`、命令 `exu_queue.lsu_cmd`；
  输出 `lsu_rdy`、`lsu_l_data`（读数据）、`lsu_exc_req`/`lsu_exc_code`（异常）。
- LSU 的访存宽度/移位/对齐、DMEM 握手、异常码（`LD/ST_ADDR_MISALIGN`、`LD/ST_ACCESS_FAULT`）在 LSU 模块内实现，EXU 只做接口与异常汇聚。

### 4.7 EXU 状态与退休（`:793-830`）

`exu_rdy` 多路选择（`:798-807`）：

| 条件 | `exu_rdy` | 行号 |
|---|---|---|
| `lsu_req` | `lsu_rdy \| lsu_exc_req`（访存完成或异常） | 800 |
| `ialu_vd`（M 扩展） | `ialu_rdy`（多周期乘除完成） | 802 |
| `csr2exu_mstatus_mie_up_i` | `1'b0`（CSR 更新中，暂停） | 804 |
| default | `1'b1` | 805 |

派生信号：

| 信号 | 表达式 | 行号 |
|---|---|---|
| `exu2pipe_init_pc_o` | `init_pc` | 809 |
| `exu2idu_rdy_o` | `exu_rdy & ~exu_queue_barrier` | 810 |
| `exu2pipe_exu_busy_o` | `exu_queue_vd & ~exu_rdy` | 811 |
| `exu2pipe_instret_o` | `exu_queue_vd & exu_rdy` | 812 |
| `exu2csr_instret_no_exc_o` | `instret & ~exu_exc_req` | 814 |
| `exu2pipe_exc_req_o` | `vd ? exu_exc_req : exu_exc_req_ff`（DBG） | 819-821 |
| `exu2pipe_brkpt_o` | `vd & (exc_code == BREAKPOINT)` | 825 |

> `exu2idu_rdy_o` 同时喂给 IDU（作为 `idu2ifu_rdy_o`）和本模块队列控制，形成"EXU 能接收"的背压。

### 4.8 MPRF 接口（`:832-890`）

#### 4.8.1 取数阶段（`:836-867`）

- `SCR1_NO_EXE_STAGE`：直接用 `idu2exu_use_rs1/rs2_i`（`:840-841`）。
- 有 EXU 级：用寄存后的 `idu2exu_use_rs1_ff/rs2_ff`（`:851-852`）。
- `SCR1_MPRF_RAM`：因 RAM 同步读，`exu_queue_en` 时用当前 IDU 地址/标志，否则用上一拍寄存值（`:843-860`）。
- `exu2mprf_rs1_addr_o = mprf_rs1_req ? addr : '0`（`:866-867`）——不用时置 0，省翻转。

#### 4.8.2 写回阶段（`:869-890`）

```systemverilog
exu2mprf_w_req_o = (rd_wb_sel != NONE) & exu_queue_vd & ~exu_exc_req
                 & ~hdu2exu_no_commit_i
                 & ((rd_wb_sel == CSR) ? csr_access_init : exu_rdy);
```

- 异常指令不写回（`~exu_exc_req`）；调试 no_commit 不写回。
- **CSR 写回**要等 `csr_access_init`（CSR 新访问可发起），其余等 `exu_rdy`。
- `exu2mprf_rd_addr_o = MPRF_AWIDTH'(exu_queue.rd_addr)`（`:878`）。
- 写回数据 mux（`:881-890`）：`SUM2→ialu_addr_res`、`IMM→imm`、`INC_PC→inc_pc`、`LSU→lsu_l_data`、`CSR→csr2exu_r_data_i`、default→`ialu_main_res`。

### 4.9 CSR 接口与访问 FSM（`:892-993`）

#### 4.9.1 CSR 读写请求（`:909-934`）

`~vd` 或 TDU 指令断点异常时，读写请求均 0。否则按 `csr_cmd`：

| csr_cmd | `exu2csr_r_req_o` | `exu2csr_w_req_o` | 行号 |
|---|---|---|---|
| WRITE | `\|exu_queue.rd_addr`（有 rd 才读） | `csr_access_init` | 919-922 |
| SET / CLEAR | `1'b1` | `\|exu_queue.rs1_addr & csr_access_init` | 923-927 |
| default | `1'b0` | `1'b0` | 928-931 |

- `exu2csr_w_cmd_o = exu_queue.csr_cmd`（`:936`）。
- `exu2csr_rw_addr_o = exu_queue.imm[CSR_ADDR_WIDTH-1:0]`（`:937`，imm 低 12 位即 CSR 地址）。
- `exu2csr_w_data_o = (csr_op==REG) ? rs1_data : {0, rs1_addr}`（`:938-940`，zimm）。

#### 4.9.2 CSR 访问 FSM（`:943-958`）

| 信号 | 表达式 | 行号 |
|---|---|---|
| 状态寄存器 | `csr_access_ff`，复位 `INIT` | 946-952 |
| `csr_access_next` | `(csr_access_init & csr2exu_mstatus_mie_up_i) ? RDY : INIT` | 954-956 |
| `csr_access_init` | `(csr_access_ff == INIT)` | 958 |

语义：CSR 访问在 `INIT` 态发起；当 CSR 模块反馈 `mstatus/mie` 已更新后进入 `RDY` 一拍，再回 `INIT`。
`csr_access_init` 用于门控连续 CSR 写与 CSR 写回，避免背靠背 CSR 访问冲突。

#### 4.9.3 CSR 事件（`:960-993`）

| 信号 | 表达式 | 行号 |
|---|---|---|
| `exu2csr_take_exc_o` | `exu_exc_req & ~hdu2exu_dbg_halted_i` | 964-968 |
| `exu2csr_exc_code_o` / `trap_val_o` | `exc_code` / `exc_trap_val` | 969-970 |
| `exu2csr_take_irq_o` | `csr2exu_irq_i & ~exu2pipe_exu_busy_o & dbg & clk_pipe_en` | 973-981 |
| `exu2csr_mret_instr_o` | `vd & mret_req & ~tdu_ibrkpt & ~dbg_halted` | 985-992 |
| `exu2csr_mret_update_o` | `mret_instr & csr_access_init` | 993 |

> 中断只在 EXU 不忙时取（`~exu_busy`），保证指令边界精确。

### 4.10 TDU 接口（`:995-1014`，`SCR1_TDU_EN`）

- 指令监视 `exu2tdu_imon_o`：`vd=exu_queue_vd`、`req=instret`、`addr=pc_curr_ff`（`:1001-1003`）。
- `exu2tdu_ibrkpt_ret_o`（`:1005-1013`）：有效指令退休时，指令断点命中置位；若同时访存，合并数据断点命中。

---

## 5. 配置宏对行为的影响

| 宏 | 定义位置 | 对 EXU 的影响 |
|---|---|---|
| `SCR1_RVM_EXT` | MAX `arch_description.svh:72` | 产生 `ialu_vd`/多周期乘除握手；`exu_rdy` 需等 `ialu_rdy`（`:402-408`、`:802`） |
| `SCR1_RVC_EXT` | MAX `:73` | 无 RVC 时才产生 `jb_misalign` 与相关异常码（`:478-480`、`:525`） |
| `SCR1_NO_EXE_STAGE` | MIN `:106`、custom `:135` | 摘除 IDU→EXU 队列寄存器；命令组合直通；`use_rd/imm` 端口消失（`:306-383`） |
| `SCR1_MPRF_RAM` | FPGA Intel `:191` | 因同步读，取数地址/标志需按 `exu_queue_en` 从 IDU 或队列寄存器选择（`:843-860`） |
| `SCR1_DBG_EN` | MAX `:79` | 启用 HDU 接口、`exu_exc_req_ff`、debug barrier、`exu2hdu_ibrkpt_hw_o`（`:127-136`、`:302-304`、`:498-509`、`:818-828`） |
| `SCR1_TDU_EN` | MAX `:80` | 启用 TDU 接口、指令断点优先编码、`trap_val` 分支（`:139-152`、`:516-520`、`:560-568`、`:995-1014`） |
| `SCR1_CLKCTRL_EN` | custom `:138` | WFI/CSR 寄存器改用 `clk_alw_on`；中断取用受 `clk_pipe_en` 约束（`:610-639`、`:978-980`） |
| `SCR1_CSR_REDUCED_CNT` | CSR 选项 | 裁掉 `exu2csr_instret_no_exc_o`（`:120-122`、`:813-815`） |
| `SCR1_TRGT_SIMULATION` | 仿真选项 | 启用 tracelog 与 7 条 SVA（`:1017-1084`） |

**MAX 参考路径**：定义 `RVM/RVC/DBG/TDU/CLKCTRL`，不定义 `NO_EXE_STAGE/MPRF_RAM` → 完整队列寄存器 + 多周期乘除 + 调试/触发。

> 注意：`SCR1_CLKCTRL_EN` 不在 MAX 推荐配置里，MAX 的 `exu2pipe_wfi_halted_o` 端口也不存在；`SCR1_MPRF_RAM` 仅 Intel FPGA 目标启用（`:190-198`）。

---

## 6. 时序行为示例

### 6.1 普通 IALU 指令（有 EXU 级）

1. 拍 N：`idu2exu_req_i=1`、`exu2idu_rdy_o=1`（`exu_rdy=1`）→ `exu_queue_en=1`，命令锁入 `exu_queue`；
2. 拍 N+1：`exu_queue_vd=1`，IALU 计算，`exu_rdy=1` → `instret=1`、`w_req=1` 写回；
3. PC 在 `instret` 拍推进到 `inc_pc`。

### 6.2 LOAD/STORE

`lsu_req=1` → `exu_rdy = lsu_rdy | lsu_exc_req`（`:800`）。访存期间 `exu_busy=1`，EXU 不接收新指令；完成或异常当拍退休。

### 6.3 跳转/分支

`jb_taken=1` → `exu2ifu_pc_new_req_o=1`（`:735`），`new_pc = ialu_addr_res & JUMP_MASK`；`pc_curr_next` 取 `new_pc`；IFU flush 队列并从目标取指。

### 6.4 取中断

`csr2exu_irq_i=1` 且 `~exu_busy` → `take_irq=1`（`:973`）→ New PC = `csr2exu_new_pc_i`（trap 向量），`pc_curr_upd` 推进 PC。CSR 更新 `MCAUSE/MEPC`。

### 6.5 异常

`exu_exc_req=1` → `take_exc=1`、`exc_code`、`trap_val` 送 CSR；`w_req=0`（不写回）；New PC = `csr2exu_new_pc_i`。

### 6.6 WFI

执行 WFI 且 `~ip_ie`：`wfi_halt_req=1` → `wfi_halted_ff=1`（进入停机）。此后 `ip_ie` 有效：`wfi_run_req=1` 退出停机，`wfi_run_start_ff` 拉高一拍 → `new_pc_req=1`、`new_pc=pc_curr_ff`（重新取指）。

### 6.7 MRET

`mret_req` 且 `csr_access_init`：`mret_update=1` 更新 CSR；`mret_instr=1` 且 `~mstatus_mie_up` 时 `new_pc_req=1`、New PC = `csr2exu_new_pc_i`。

---

## 7. 断言规格（`:1017-1084`，仅 `SCR1_TRGT_SIMULATION`）

| 断言 | 行号 | 检查内容 |
|---|---|---|
| `SCR1_SVA_EXU_XCHECK_CTRL` | 1039-1042 | 控制信号无 X |
| `SCR1_SVA_EXU_XCHECK_QUEUE` | 1044-1047 | 队列命令无 X |
| `SCR1_SVA_EXU_XCHECK_CSR_RDATA` | 1049-1052 | CSR 读数据/异常无 X |
| `SCR1_SVA_EXU_ONEHOT` | 1056-1059 | `{jump_req, branch_req, lsu_req}` 至多一个 |
| `SCR1_SVA_EXU_ONEHOT_EXC` | 1061-1069 | 有效指令的异常来源至多一个 |
| `SCR1_SVA_EXU_CURR_PC_UPD_BEFORE_INIT` | 1072-1075 | 初始化完成前不更新当前 PC |
| `SCR1_SVA_EXU_NEW_PC_REQ_BEFORE_INIT` | 1079-1082 | 初始化完成前不产生 New PC 请求 |

`ONEHOT` 体现设计约束：一条指令不可能同时是跳转、分支和访存。`*_BEFORE_INIT` 保证复位序列先于一切 PC 操作。

---

## 8. 假设与约束

1. IDU 保证命令字字段与指令一致，且 `use_*` 门控有效；
2. MPRF 两个读口在 `exu2mprf_rs*_addr_o` 给出的地址上返回数据；
3. CSR 模块按 `csr_access_init` 握手完成读写并反馈 `mstatus_mie_up`；
4. 分支/跳转目标只可能来自 `ialu_addr_res`，经 `JUMP_MASK` 清 bit0；
5. 异常/中断/MRET 的 New PC 由 CSR 给出（`csr2exu_new_pc_i`），EXU 不自行计算 trap 向量；
6. 退休条件 `exu_queue_vd & exu_rdy` 覆盖 IALU 单周期与 LSU/乘除多周期两类；
7. 调试（HDU）与触发（TDU）通过 barrier/优先级参与执行控制。

---

## 附录 A：端口清单

### A.1 公共与时钟（`:55-61`）

| 端口 | 方向 | 行号 | 条件 |
|---|---|---|---|
| `rst_n` | in | 56 | |
| `clk` | in | 57 | |
| `clk_alw_on` | in | 59 | `SCR1_CLKCTRL_EN` |
| `clk_pipe_en` | in | 60 | `SCR1_CLKCTRL_EN` |

### A.2 IDU ↔ EXU（`:63-72`）

| 端口 | 方向 | 行号 | 条件 |
|---|---|---|---|
| `idu2exu_req_i` | in | 64 | |
| `exu2idu_rdy_o` | out | 65 | |
| `idu2exu_cmd_i` | in | 66 | |
| `idu2exu_use_rs1_i` | in | 67 | |
| `idu2exu_use_rs2_i` | in | 68 | |
| `idu2exu_use_rd_i` | in | 70 | 无 `SCR1_NO_EXE_STAGE` |
| `idu2exu_use_imm_i` | in | 71 | 无 `SCR1_NO_EXE_STAGE` |

### A.3 MPRF（`:74-81`）

| 端口 | 方向 | 行号 |
|---|---|---|
| `exu2mprf_rs1_addr_o` / `mprf2exu_rs1_data_i` | out/in | 75-76 |
| `exu2mprf_rs2_addr_o` / `mprf2exu_rs2_data_i` | out/in | 77-78 |
| `exu2mprf_w_req_o` | out | 79 |
| `exu2mprf_rd_addr_o` / `exu2mprf_rd_data_o` | out | 80-81 |

### A.4 CSR（`:83-102`）

见 2.4 节表格（`:84-102`）。

### A.5 DMEM（`:104-112`）

见 2.5 节表格。

### A.6 EXU 控制/PC（`:114-123`、`:154-161`）

见 2.6 节表格。

### A.7 HDU（`:125-137`，`SCR1_DBG_EN`）

| 端口 | 方向 | 行号 |
|---|---|---|
| `hdu2exu_no_commit_i` | in | 127 |
| `hdu2exu_irq_dsbl_i` | in | 128 |
| `hdu2exu_pc_advmt_dsbl_i` | in | 129 |
| `hdu2exu_dmode_sstep_en_i` | in | 130 |
| `hdu2exu_pbuf_fetch_i` | in | 131 |
| `hdu2exu_dbg_halted_i` | in | 132 |
| `hdu2exu_dbg_run2halt_i` | in | 133 |
| `hdu2exu_dbg_halt2run_i` | in | 134 |
| `hdu2exu_dbg_run_start_i` | in | 135 |
| `hdu2exu_dbg_new_pc_i` | in | 136 |

### A.8 TDU（`:139-152`，`SCR1_TDU_EN`）

见 2.7 节。含 `exu2tdu_imon_o`、`lsu2tdu_dmon_o`、断点匹配/异常请求、`exu2tdu_ibrkpt_ret_o`、`exu2hdu_ibrkpt_hw_o`。

---

## 附录 B：局部参数/类型/信号

### B.1 localparam（`:168`）

| 名称 | 值 | 用途 |
|---|---|---|
| `SCR1_JUMP_MASK` | `XLEN'hFFFF_FFFE` | 跳转/分支目标清 bit0 |

### B.2 typedef（`:174-177`）

| 名称 | 取值 | 用途 |
|---|---|---|
| `scr1_csr_access_e` | `SCR1_CSR_INIT`, `SCR1_CSR_RDY` | CSR 访问 FSM |

### B.3 内部信号（`:183-289`）

| 分组 | 代表信号 | 行号 |
|---|---|---|
| 队列 | `exu_queue_vd`、`exu_queue`、`exu_queue_barrier`、`exu_queue_en` | 183-203 |
| IALU | `ialu_main_op1/2`、`ialu_main_res`、`ialu_addr_op1/2`、`ialu_addr_res`、`ialu_cmp` | 205-217 |
| 异常 | `exu_exc_req`、`exc_code`、`exc_trap_val`、`instr_fault_rvi_hi` | 219-228 |
| WFI | `wfi_halt_cond/req`、`wfi_run_req`、`wfi_run_start_ff`、`wfi_halted_ff` | 230-244 |
| PC | `init_pc_v/init_pc`、`inc_pc`、`branch_taken`、`jb_taken`、`jb_new_pc`、`pc_curr_ff` | 246-262 |
| LSU | `lsu_req/rdy/l_data/exc_req/exc_code` | 264-270 |
| 状态 | `exu_rdy` | 272-274 |
| MPRF | `mprf_rs1/2_req`、`mprf_rs1/2_addr` | 276-282 |
| CSR | `csr_access_ff/next/init` | 284-289 |

---

## 附录 C：异常码与 trap_val 映射

| exc_code | 来源 | trap_val | 备注 |
|---|---|---|---|
| INSTR_MISALIGN(0) | EXU（无 RVC 的 jump/branch 非对齐） | `jb_new_pc` | 仅 `~SCR1_RVC_EXT` |
| INSTR_ACCESS_FAULT(1) | IFU（IDU 透传） | 高半字故障→`inc_pc`，否则 `pc_curr_ff` | 经 `instr_fault_rvi_hi` 区分 |
| ILLEGAL_INSTR(2) | IDU / CSR | IDU：`exu_queue.imm`；CSR：重建的 SYSTEM 编码 | `SCR1_MTVAL_ILLEGAL_INSTR_EN` |
| BREAKPOINT(3) | IDU（EBREAK）/ BRKM | 指令断点→`pc_curr_ff`；数据断点→`ialu_addr_res` | TDU 优先 |
| LD/ST misalign/fault(4-7) | LSU | `ialu_addr_res` | |
| ECALL_M(11) | IDU | `'0`（default 分支） | |

---

## 附录 D：一次异常/中断/MRET 的完整流程

**异常（以非法指令为例）**

1. IDU 译出非法 → `idu2exu_cmd_i.exc_req=1`、`exc_code=ILLEGAL_INSTR`、`imm=instr`；
2. EXU 队列接收；`exu_queue_vd=1`；
3. `exu_exc_req=1`（`:483-493`）；异常码编码器取 `exu_queue.exc_code`（`:521`）；
4. `trap_val = exu_queue.imm`（`MTVAL` 使能，`:554`）；
5. `exu2csr_take_exc_o=1`、`exu2csr_exc_code_o`、`exu2csr_trap_val_o` 送 CSR；
6. CSR 更新 `MCAUSE/MEPC/MTVAL`，给出 `csr2exu_new_pc_i`（trap 向量）；
7. EXU `new_pc_req=1`、`new_pc=csr2exu_new_pc_i`（`:710-712`）；`w_req=0` 不写回；
8. IFU flush 队列，从 trap 向量取指。

**中断**

`csr2exu_irq_i & ~exu_busy` → `take_irq=1`（`:973`）→ 同异常路径，New PC 来自 CSR。

**MRET**

`mret_req & csr_access_init` → `mret_update=1`（CSR 恢复 `mstatus/mepc`）→ New PC = `csr2exu_new_pc_i`（返回地址）。

---

## 附录 E：关键信号 → 消费者映射

| EXU 信号 | 生产者 | 消费者 | 作用 |
|---|---|---|---|
| `idu2exu_cmd_i` | IDU | EXU 队列 | 命令字 |
| `exu2idu_rdy_o` | EXU `exu_rdy & ~barrier` | IDU `idu2ifu_rdy_o` | 背压 |
| `exu2mprf_w_req_o/rd_addr/rd_data` | EXU 写回 | MPRF | 寄存器写 |
| `exu2csr_take_exc_o/take_irq_o/mret_*` | EXU 事件 | CSR | trap/MRET 控制 |
| `exu2csr_exc_code_o/trap_val_o` | 异常逻辑 | CSR `MCAUSE/MTVAL` | 异常信息 |
| `exu2csr_rw_addr_o` | `imm[11:0]` | CSR | CSR 地址 |
| `exu2dmem_*` | LSU | DMEM | 访存 |
| `exu2ifu_pc_new_req_o/new_o` | PC 逻辑 | IFU | 换 PC（flow control） |
| `exu2csr_pc_next_o` | PC 逻辑 | CSR `MEPC` | 下一 PC |
| `exu2pipe_instret_o` | 状态逻辑 | pipe/CSR/TDU | 退休 |
| `exu2pipe_exu_busy_o` | 状态逻辑 | CSR（中断门控） | 忙 |
| `exu2tdu_imon_o` | 状态/TDU | TDU | 指令监视 |

顶层例化见 `src/core/pipeline/scr1_pipe_top.sv:365-471`（EXU）、`:506-539`（CSR）。

---

## 附录 F：与 IFU/IDU 的职责边界

| 职责 | IFU | IDU | EXU |
|---|---|---|---|
| 取指、指令切割 | ✓ | — | — |
| 命令译码 | — | ✓ | — |
| flow control 产生 | — | — | ✓ |
| flow control 响应（flush） | ✓ | — | — |
| 当前 PC 维护 | 内部取指 PC | — | ✓（`pc_curr_ff`） |
| 异常产生 | 取指异常 | 非法/ECALL/EBREAK | 分支非对齐、LSU、CSR |
| 异常码/trap_val | — | 传递 | ✓ 汇聚与编码 |
| trap 向量计算 | — | — | 由 CSR 提供 |
| 寄存器读写 | — | — | ✓（MPRF） |

