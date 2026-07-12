# SCR1 执行单元（EXU）技术分析

> 对应源码: `src/core/pipeline/scr1_pipe_exu.sv`
> 模块名: `scr1_pipe_exu`
> 上游模块: IDU（指令译码单元）
> 制定者: Syntacore LLC, 2016-2021

---

## 1. 模块定位

EXU 是 SCR1 处理器五级流水线中实际的执行级，接收 IDU 译码后的 EXU 命令结构体，完成指令执行的全部工作。它是流水线中最复杂的模块，承担了指令提交、算术计算、访存、异常处理、WFI 管理、PC 维护和 CSR 交互等职责。

```
       clk / rst_n
           │
  ┌────────┴────────────────────────────────────────────────────────────────┐
  │                         scr1_pipe_exu                                    │
  │                                                                          │
  │                       ┌──────────────────┐                               │
  │   idu2exu_cmd_i ────▶│  指令执行队列     │                               │
  │   idu2exu_req_i  ───▶│  (EXU Queue)     │                               │
  │                       └────────┬─────────┘                               │
  │                                │ exu_queue (type_scr1_exu_cmd_s)         │
  │           ┌────────────────────┼────────────────────┐                    │
  │           ▼                    ▼                     ▼                    │
  │   ┌─────────────┐   ┌──────────────────┐   ┌──────────────────┐         │
  │   │  IALU       │   │  Exception Logic │   │  WFI Logic       │         │
  │   │  • 主 ALU   │   │  • ecode 优先级  │   │  • halt FSM      │         │
  │   │  • 地址加法 │   │  • mtval 多路器  │   │  • run start     │         │
  │   │  • 乘除法器 │   │  • exc_req 综合   │   │                  │         │
  │   └──────┬──────┘   └────────┬─────────┘   └────────┬─────────┘         │
  │          │                   │                       │                    │
  │          └───────────────────┼───────────────────────┘                    │
  │                              │                                            │
  │   ┌──────────────┐   ┌───────┴──────────┐   ┌──────────────────┐         │
  │   │  PC Logic    │   │  EXU Status      │   │  CSR Interface    │         │
  │   │  • init PC   │   │  • ready / busy  │   │  • read/write     │         │
  │   │  • curr PC   │   │  • instret       │   │  • access FSM     │         │
  │   │  • new PC MUX│   │  • exc_req_o     │   │  • events         │         │
  │   └──────┬───────┘   └──────────────────┘   └──────────────────┘         │
  │          │        ┌──────────────────┐                                    │
  │          └───────▶│  MPRF Interface   │                                    │
  │                   │  • rs1/rs2 read   │                                    │
  │   ┌──────────┐   │  • rd write       │                                    │
  │   │   LSU    │   └──────────────────┘                                    │
  │   │  (instance)                                                           │
  │   └──────────┘                                                            │
  └──────────────────────────────────────────────────────────────────────────┘
```

### 配置宏一览

| 宏名 | 作用 |
|------|------|
| `SCR1_DBG_EN` | 使能调试接口（HDU 交互） |
| `SCR1_TDU_EN` | 使能触发调试单元（TDU 交互） |
| `SCR1_RVM_EXT` | 使能整数乘除法扩展（多周期 IALU） |
| `SCR1_RVC_EXT` | 使能压缩指令扩展（影响 inc_pc 和跳转对齐检查） |
| `SCR1_CLKCTRL_EN` | 使能时钟门控信号 |
| `SCR1_NO_EXE_STAGE` | 无独立执行阶段配置（队列退化为组合直通） |
| `SCR1_MPRF_RAM` | MPRF 使用同步 RAM（需预取地址） |
| `SCR1_CSR_REDUCED_CNT` | 精简 CSR 计数器（移除 instret_no_exc 输出） |
| `SCR1_MTVAL_ILLEGAL_INSTR_EN` | mtval 记录非法指令编码 |
| `SCR1_TRGT_SIMULATION` | 使能 SVA 断言和 Tracelog 信号 |

---

## 2. 指令执行队列

### 2.1 队列结构与控制逻辑

EXU 在执行指令前需要通过队列寄存当前指令的 EXU 命令。队列行为由 `SCR1_NO_EXE_STAGE` 宏控制，分为两种配置：

**配置 A：有执行阶段（`~SCR1_NO_EXE_STAGE`）**

队列是一个完整的寄存器流水级。指令在 `exu_queue_en` 为高时从 IDU 锁存到 `exu_queue` 寄存器；随后，队列有效标志 `exu_queue_vd_ff` 控制后续逻辑的使能。

```systemverilog
// 队列装载条件
assign exu_queue_en = exu2idu_rdy_o & idu2exu_req_i;

// 队列寄存器（时钟门控选择写入不同字段）
always_ff @(posedge clk) begin
    if (exu_queue_en) begin
        exu_queue.instr_rvc      <= idu2exu_cmd_i.instr_rvc;
        // ... 控制字段无条件写入
        if (idu2exu_use_rs1_i)   exu_queue.rs1_addr <= idu2exu_cmd_i.rs1_addr;
        if (idu2exu_use_rs2_i)   exu_queue.rs2_addr <= idu2exu_cmd_i.rs2_addr;
        if (idu2exu_use_rd_i)    exu_queue.rd_addr  <= idu2exu_cmd_i.rd_addr;
        if (idu2exu_use_imm_i)   exu_queue.imm      <= idu2exu_cmd_i.imm;
    end
end
```

**配置 B：无执行阶段（`SCR1_NO_EXE_STAGE`）**

队列退化为组合直通，无任何寄存器：

```systemverilog
assign exu_queue_vd = idu2exu_req_i & ~exu_queue_barrier;
assign exu_queue    = idu2exu_cmd_i;
```

### 2.2 队列有效标志寄存器（仅配置 A）

```systemverilog
assign exu_queue_vd_upd = exu_queue_barrier | exu_rdy;

always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)
        exu_queue_vd_ff <= 1'b0;
    else if (exu_queue_vd_upd)
        exu_queue_vd_ff <= exu_queue_vd_next;
end

// 队列仅在无障碍、有 IDU 请求且不发生 PC 跳转时变为有效
assign exu_queue_vd_next = ~exu_queue_barrier & idu2exu_req_i & ~exu2ifu_pc_new_req_o;
assign exu_queue_vd      = exu_queue_vd_ff;
```

关键：当发生 PC 跳转（`exu2ifu_pc_new_req_o`）时，新入队的指令会被立即清除——因为跳转意味着流水线刷新，当前指令不应被执行。

### 2.3 队列屏障（Barrier）——跳过执行条件

队列屏障阻止新指令被装入或执行。满足以下任一条件时屏障拉高：

```systemverilog
assign exu_queue_barrier = wfi_halted_ff          // WFI 已暂停状态
                         | wfi_halt_req            // 该周期 WFI 进入暂停
                         | wfi_run_start_ff        // WFI 刚退出的第一周期
`ifdef SCR1_DBG_EN
                         | hdu2exu_dbg_halted_i    // 调试暂停状态
                         | hdu2exu_dbg_run2halt_i  // 进入调试暂停
                         | dbg_run_start_npbuf     // 调试恢复的第一周期（非 PBUF）
`endif
;
```

屏障与 EXU 就绪信号联合控制反压：

```systemverilog
assign exu2idu_rdy_o = exu_rdy & ~exu_queue_barrier;
```

---

## 3. 整数算术逻辑单元（IALU）

### 3.1 操作数获取

IALU 有两套独立的操作数通路：

**主 ALU 操作数**（用于算术/逻辑/分支比较）:

| `ialu_op` 值 | `ialu_main_op1` | `ialu_main_op2` |
|:---:|---|---|
| `REG_REG` | `mprf2exu_rs1_data_i` | `mprf2exu_rs2_data_i` |
| `REG_IMM` | `mprf2exu_rs1_data_i` | `exu_queue.imm` |

当 `SCR1_RVM_EXT` 使能且 IALU 操作无效（`~ialu_vd`）时，操作数清零以节省功耗。

**地址加法器操作数**（用于跳转目标/访存地址/AUIPC）:

| `sum2_op` 值 | `ialu_addr_op1` | `ialu_addr_op2` |
|:---:|---|---|
| `PC_IMM` | `pc_curr_ff` | `exu_queue.imm` |
| `REG_IMM` | `mprf2exu_rs1_data_i` | `exu_queue.imm` |

### 3.2 IALU 子模块

`scr1_pipe_ialu` 实例化，负责：

- **主 ALU**：加减、逻辑（AND/OR/XOR）、移位（SLL/SRL/SRA）、比较（SLT/SLTU/SUB_EQ/SUB_NE）
- **地址加法器**：独立的无符号加法器，用于地址计算
- **乘除法器**（`SCR1_RVM_EXT`）：多周期 MUL/DIV/REM 操作
  - `ialu_rdy` 拉高前，`exu_rdy` 保持为低，阻塞流水线

### 3.3 分支判定

```systemverilog
assign branch_taken = exu_queue.branch_req & ialu_cmp;
assign jb_taken     = exu_queue.jump_req | branch_taken;
assign jb_new_pc    = ialu_addr_res & SCR1_JUMP_MASK; // 低 1 位清零
```

`ialu_cmp` 由 IALU 在 `ialu_cmd` 为比较类型命令时产生（从主 ALU 结果的符号位/零标志导出）。

---

## 4. 异常检测与上报

### 4.1 异常请求综合

```systemverilog
assign exu_exc_req = exu_queue_vd & (
    exu_queue.exc_req          // IDU 译码异常（ECALL/EBREAK/ILLEGAL/IMEM Fault）
  | lsu_exc_req                // LSU 异常（地址未对齐/访问错误）
  | csr2exu_rw_exc_i           // CSR 读写异常（非法 CSR 地址）
  | jb_misalign                // 跳转目标未对齐（仅 ~RVC）
  | exu2hdu_ibrkpt_hw_o        // TDU 硬件断点（仅 TDU + DBG）
);
```

每类异常的来源：
- `exu_queue.exc_req` / `exu_queue.exc_code`：由 IDU 在译码阶段填充（ECALL_M / EBREAK / ILLEGAL_INSTR / INSTR_ACCESS_FAULT）
- `lsu_exc_req` / `lsu_exc_code`：由 LSU 子模块产生（LD/ST 未对齐/访问错误）
- `csr2exu_rw_exc_i`：CSR 模块判定为非法 CSR 地址
- `jb_misalign`：仅 `~SCR1_RVC_EXT` 时生效，检测跳转目标 `jb_new_pc[1:0] != 0`
- `exu2hdu_ibrkpt_hw_o`：TDU 硬件指令断点或数据断点

### 4.2 异常编码优先级

```systemverilog
always_comb begin
    case (1'b1)  // 优先级从高到低
        exu2hdu_ibrkpt_hw_o : exc_code = SCR1_EXC_CODE_BREAKPOINT;     // 最高：硬件断点
        exu_queue.exc_req   : exc_code = exu_queue.exc_code;           // IDU 译码异常
        lsu_exc_req         : exc_code = lsu_exc_code;                 // LSU 异常
        csr2exu_rw_exc_i    : exc_code = SCR1_EXC_CODE_ILLEGAL_INSTR; // CSR 非法访问
        jb_misalign         : exc_code = SCR1_EXC_CODE_INSTR_MISALIGN; // 跳转未对齐
        default             : exc_code = SCR1_EXC_CODE_ECALL_M;        // 不应到达
    endcase
end
```

使用 `case (1'b1)`（即优先编码器）代替 `if-else` 链，第一个为真的 case 项胜出。

### 4.3 异常陷阱值（mtval）

| 异常类型 | `exc_trap_val` | 说明 |
|------|------|------|
| `INSTR_MISALIGN` | `jb_new_pc` | 未对齐的跳转目标地址 |
| `INSTR_ACCESS_FAULT` | `instr_fault_rvi_hi ? inc_pc : pc_curr_ff` | 若 RVI 高半取指错误指向高 16 位地址 |
| `ILLEGAL_INSTR` (MTVAL_EN) | `exu_queue.exc_req ? exu_queue.imm : exu_illegal_instr` | IDU 填充的非法指令编码，或 EXU 合成的 CSR 非法编码 |
| `ILLEGAL_INSTR` (~MTVAL_EN) | 0 | 不记录指令值 |
| `BREAKPOINT` | `pc_curr_ff`（I 断点）或 `ialu_addr_res`（D 断点） | 断点触发的 PC 或数据地址 |
| LD/ST 异常 | `ialu_addr_res` | LSU 访存地址 |
| 其他 | 0 | |

`instr_fault_rvi_hi` 信号复用 `exu_queue.instr_rvc` 字段：当 IMEM 在 32 位指令的高 16 位发生访问错误时，RVC 标志被 IDU/IFU 用来标记该含义。

---

## 5. WFI 状态机

### 5.1 状态定义

```
                   ~ip_ie & (wfi_req | run_start)
    ┌─────────┐  ──────────────────────────────▶  ┌──────────┐
    │ RUNNING │                                    │  HALTED  │
    │ (halted=0)│  ◀─────────────────────────    │(halted=1) │
    └─────────┘        ip_ie | dbg_halt2run       └──────────┘
         │                                                 │
         │    halt_req 上升沿                               │  run_req 上升沿
         │    ───────────▶ exu2pipe_wfi_run2halt_o          │  ───────────▶ PC 保持，queue barrier
         │                                                 │
         │  进入 HALTED: exu_queue_barrier 生效             │  退出 HALTED: wfi_run_start_ff 单周期脉冲
         │  阻塞流水线                                     │  保持 queue barrier 一周期
         └─────────────────────────────────────────────────┘
```

### 5.2 暂停条件（`wfi_halt_cond`）

```systemverilog
assign wfi_halt_cond = ~csr2exu_ip_ie_i                     // 无待处理中断
                     & ((exu_queue_vd & exu_queue.wfi_req)   // WFI 指令
                        | wfi_run_start_ff)                   // 或 WFI 刚启动
`ifdef SCR1_DBG_EN
                     & ~hdu2exu_no_commit_i                  // HDU 未禁止提交
                     & ~hdu2exu_dmode_sstep_en_i             // 未在单步模式
                     & ~hdu2exu_dbg_run2halt_i               // 未在进入调试暂停
`endif
                     ;
```

暂停条件 `wfi_halt_cond` 为真时触发 `wfi_halt_req`（仅当尚未暂停时）：
```systemverilog
assign wfi_halt_req = ~wfi_halted_ff & wfi_halt_cond;
```

### 5.3 退出暂停条件（`wfi_run_req`）

```systemverilog
assign wfi_run_req = wfi_halted_ff
                   & (csr2exu_ip_ie_i               // 中断挂起且使能
`ifdef SCR1_DBG_EN
                      | hdu2exu_dbg_halt2run_i       // 调试退出暂停
`endif
                     );
```

### 5.4 暂停标志寄存器

```systemverilog
assign wfi_halted_upd = wfi_halt_req | wfi_run_req;
assign wfi_halted_next = wfi_halt_req | ~wfi_run_req;
```

`wfi_halted_next` 的逻辑：
- `wfi_halt_req` 为高 → 下一值为 1（进入暂停）
- `wfi_run_req` 为高且 `wfi_halt_req` 为低 → 下一值为 0（退出暂停）
- 两者同时为高 → 保持 1（暂停优先）

### 5.5 运行起始标志（`wfi_run_start_ff`）

```systemverilog
assign wfi_run_start_next = wfi_halted_ff & csr2exu_ip_ie_i & ~exu2csr_take_irq_o;

always_ff @(negedge rst_n, posedge clk_alw_on) begin
    if (~rst_n)
        wfi_run_start_ff <= 1'b0;
    else
        wfi_run_start_ff <= wfi_run_start_next;
end
```

当 WFI 被中断唤醒：
1. `csr2exu_ip_ie_i` 拉高 → `wfi_run_req` 拉高 → `wfi_halted_ff` 清零 → 下一周期 `wfi_run_start_ff` 拉高
2. `wfi_run_start_ff` 单周期脉冲：
   - 触发新 PC 请求（指向 `pc_curr_ff`，不跳转）
   - 拉高 `exu_queue_barrier`（防止新指令立即入队）
   - 若中断未被接收（`~exu2csr_take_irq_o`），`wfi_run_start_next` 维持 1 直到中断被处理

注意：WFI 暂停标志寄存器使用 `clk_alw_on`（非门控时钟）采样，确保即使流水线时钟停止也能正确响应中断事件。

---

## 6. 程序计数器（PC）逻辑

### 6.1 PC 初始化

```systemverilog
always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)
        init_pc_v <= '0;
    else if (~&init_pc_v)
        init_pc_v <= {init_pc_v[2:0], 1'b1}; // 移位 1 填充
end

assign init_pc = ~init_pc_v[3] & init_pc_v[2];
```

`init_pc_v` 是一个 4 位移位寄存器，复位后依次产生值 `0000 → 0001 → 0011 → 0111 → 1111`。`init_pc` 在 `init_pc_v == 4'b0011` 时拉高一周期，向 IFU 发出新 PC 请求，将 PC 初始化为 `SCR1_RST_VECTOR`。

这一机制确保在 PC 初始化完成前，不会有任何其他事件触发 PC 更新或新 PC 请求。SVA 断言 `SCR1_SVA_EXU_CURR_PC_UPD_BEFORE_INIT` 和 `SCR1_SVA_EXU_NEW_PC_REQ_BEFORE_INIT` 对此进行验证。

### 6.2 当前 PC 寄存器

```systemverilog
assign pc_curr_upd = (exu2pipe_instret_o | exu2csr_take_irq_o
`ifdef SCR1_DBG_EN
                    | dbg_run_start_npbuf
`endif
                    ) & (~hdu2exu_pc_advmt_dsbl_i & ~hdu2exu_no_commit_i);

always_ff @(negedge rst_n, posedge clk) begin
    if (~rst_n)
        pc_curr_ff <= SCR1_RST_VECTOR;
    else if (pc_curr_upd)
        pc_curr_ff <= pc_curr_next;
end
```

PC 更新条件（`pc_curr_upd`）：
- 指令退休（`exu2pipe_instret_o`）
- 中断被接收（`exu2csr_take_irq_o`）
- 调试模式下恢复运行的第一周期（`dbg_run_start_npbuf`）

同时受 HDU 控制：
- `hdu2exu_pc_advmt_dsbl_i`：禁止 PC 推进（调试模式）
- `hdu2exu_no_commit_i`：禁止指令提交

### 6.3 递增 PC（`inc_pc`）

```systemverilog
`ifdef SCR1_RVC_EXT
assign inc_pc = pc_curr_ff + (exu_queue.instr_rvc ? 32'd2 : 32'd4);
`else
assign inc_pc = pc_curr_ff + 32'd4;
`endif
```

当 RVC 扩展使能时，压缩指令跳过 2 字节，非压缩指令跳过 4 字节。

### 6.4 当前 PC 下一值（`pc_curr_next`）

```systemverilog
assign pc_curr_next = exu2ifu_pc_new_req_o ? exu2ifu_pc_new_o                     // 跳转/异常
                    : (inc_pc[6] ^ pc_curr_ff[6]) ? inc_pc                        // 跨 64B 边界
                                                  : {pc_curr_ff[31:6], inc_pc[5:0]}; // 同 64B 区间
```

注意：当 `inc_pc` 与 `pc_curr_ff` 的 bit[6] 不同（即跨越 64B 地址边界）时，整个 32 位 PC 由 `inc_pc` 完全替换。否则仅替换低 6 位。这是针对 SRAM-based IMEM 地址预取的优化——当页内低 6 位翻转时无需全量更新高位地址线。

### 6.5 新 PC 多路选择器

```systemverilog
always_comb begin
    case (1'b1)
        init_pc              : exu2ifu_pc_new_o = SCR1_RST_VECTOR;
        exu2csr_take_exc_o,
        exu2csr_take_irq_o,
        exu2csr_mret_instr_o : exu2ifu_pc_new_o = csr2exu_new_pc_i;   // CSR 提供目标
        dbg_run_start_npbuf  : exu2ifu_pc_new_o = hdu2exu_dbg_new_pc_i; // 调试恢复 PC
        wfi_run_start_ff     : exu2ifu_pc_new_o = pc_curr_ff;           // WFI 退出不跳转
        exu_queue.fencei_req : exu2ifu_pc_new_o = inc_pc;               // FENCE.I 顺序执行
        default              : exu2ifu_pc_new_o = ialu_addr_res & SCR1_JUMP_MASK; // 跳转目标
    endcase
end
```

优先级从高到低：

| 优先级 | 触发条件 | 新 PC 来源 | 说明 |
|:---:|------|------|------|
| 1 | `init_pc` | `SCR1_RST_VECTOR` | 复位初始化 |
| 2 | `take_exc_o` / `take_irq_o` / `mret_instr_o` | `csr2exu_new_pc_i` | CSR 提供 mtvec / mepc |
| 3 | `dbg_run_start_npbuf` | `hdu2exu_dbg_new_pc_i` | 调试恢复 |
| 4 | `wfi_run_start_ff` | `pc_curr_ff` | WFI 退出，PC 不跳转 |
| 5 | `exu_queue.fencei_req` | `inc_pc` | FENCE.I 顺序执行下一指令 |
| 6 | 默认（跳转/分支） | `ialu_addr_res & JUMP_MASK` | JAL/JALR/BRANCH 目标 |

### 6.6 新 PC 请求条件

```systemverilog
assign exu2ifu_pc_new_req_o = init_pc
                            | exu2csr_take_irq_o
                            | exu2csr_take_exc_o
                            | (exu2csr_mret_instr_o & ~csr2exu_mstatus_mie_up_i)
                            | (exu_queue_vd & exu_queue.fencei_req)
                            | (wfi_run_start_ff & clk_pipe_en)
                            | dbg_run_start_npbuf
                            | (exu_queue_vd & jb_taken);
```

特殊条件 `exu2csr_mret_instr_o & ~csr2exu_mstatus_mie_up_i`：

当 MRET 指令执行且 MSTATUS/MIE 在同一周期被更新的情况下，不生成新 PC 请求。因为 mstatus 更新需要额外一周期生效——MRET 的 PC 跳转会延迟到下一周期。

在 `SCR1_CLKCTRL_EN` 配置下，WFI 运行起始请求还需要 `clk_pipe_en` 为高才发出新 PC 请求，确保流水线时钟已打开。

---

## 7. 加载存储单元（LSU）接口

### 7.1 LSU 请求生成

```systemverilog
assign lsu_req = (exu_queue.lsu_cmd != SCR1_LSU_CMD_NONE) & exu_queue_vd;
```

当队列中指令的 `lsu_cmd` 不是 NONE（即为 LOAD 或 STORE 指令）且队列有效时，发出 LSU 请求。

### 7.2 LSU ↔ EXU 信号映射

| EXU → LSU | 信号 | 来源 |
|------|------|------|
| 请求 | `exu2lsu_req_i` | `lsu_req` |
| 命令 | `exu2lsu_cmd_i` | `exu_queue.lsu_cmd` |
| 地址 | `exu2lsu_addr_i` | `ialu_addr_res`（地址加法器结果） |
| 存储数据 | `exu2lsu_sdata_i` | `mprf2exu_rs2_data_i` |

| LSU → EXU | 信号 | 流向 |
|------|------|------|
| 就绪 | `lsu2exu_rdy_o` | → `lsu_rdy` → `exu_rdy` 计算 |
| 加载数据 | `lsu2exu_ldata_o` | → `lsu_l_data` → MPRF 写回通路 |
| 异常请求 | `lsu2exu_exc_o` | → `lsu_exc_req` → `exu_exc_req` 综合 |
| 异常编码 | `lsu2exu_exc_code_o` | → `lsu_exc_code` → 异常编码优先级 → `exc_code` |

### 7.3 访存地址来源

所有访存指令的地址均来自地址加法器（`ialu_addr_res`）：
- LOAD/STORE: `REG_IMM` 模式，`rs1 + imm`
- 地址加法器在 EXU 中是独立于主 ALU 的专用通路，保证地址计算和算术运算可并行进行

### 7.4 LSU → DMEM 直接映射

EXU 将 LSU 的 DMEM 接口信号直接透传到模块端口：

```
EXU.lsu_req  → i_lsu.exu2lsu_req_i
i_lsu.lsu2dmem_req_o → EXU.exu2dmem_req_o
i_lsu.lsu2dmem_cmd_o → EXU.exu2dmem_cmd_o
...
```

---

## 8. EXU 状态逻辑

### 8.1 就绪（Ready）标志

```systemverilog
always_comb begin
    case (1'b1)
        lsu_req                  : exu_rdy = lsu_rdy | lsu_exc_req;
        ialu_vd                  : exu_rdy = ialu_rdy;           // RVM 扩展
        csr2exu_mstatus_mie_up_i : exu_rdy = 1'b0;               // mstatus 更新周期
        default                  : exu_rdy = 1'b1;
    endcase
end
```

EXU 就绪标志控制流水线停顿。只有在当前指令完全执行完毕后，才允许装入下一条指令：

| 条件 | exu_rdy | 含义 |
|------|:---:|------|
| LSU 操作 | `lsu_rdy | lsu_exc_req` | 等待 LSU 完成或异常即可 |
| IALU RVM 多周期 | `ialu_rdy` | 等待乘除法器完成 |
| mstatus 更新 | `0` | 该周期不允许提交任何指令 |
| 默认 | `1` | 单周期指令立即就绪 |

### 8.2 关键状态输出

```systemverilog
assign exu2pipe_instret_o       = exu_queue_vd & exu_rdy;     // 指令退休
assign exu2pipe_exu_busy_o      = exu_queue_vd & ~exu_rdy;    // EXU 忙
assign exu2pipe_init_pc_o       = init_pc;                     // PC 初始化完成
assign exu2pipe_wfi_run2halt_o  = wfi_halt_req;               // 进入 WFI
assign exu2pipe_brkpt_o         = exu_queue_vd
                                & (exu_queue.exc_code == SCR1_EXC_CODE_BREAKPOINT);
```

退休（`instret`）含义：队列有效且 EXU 就绪。只要指令完成了执行（即使在执行过程中触发了异常），即视为退休。

异常上报（`exu2pipe_exc_req_o`）：

```systemverilog
`ifdef SCR1_DBG_EN
assign exu2pipe_exc_req_o = exu_queue_vd ? exu_exc_req : exu_exc_req_ff;
`else
assign exu2pipe_exc_req_o = exu_exc_req;
`endif
```

调试模式下，若队列无效（`~exu_queue_vd`）但之前有被保持的异常请求（`exu_exc_req_ff`），仍需上报。这涉及调试暂停期间异常处理的时序。

---

## 9. MPRF 接口

### 9.1 读地址控制（操作数获取）

MPRF rs1/rs2 读地址的生成需要处理两种场景：

**场景 1**：队列可接收新指令（`exu_queue_en = 1`）——从 IDU 直通的新指令读取操作数。

**场景 2**：队列保持旧指令（`exu_queue_en = 0`）——当前指令可能需要多周期，仍需保持 rs1/rs2 地址。

```systemverilog
`ifdef SCR1_MPRF_RAM
assign mprf_rs1_req = exu_queue_en
                    ? (exu_queue_vd_next & idu2exu_use_rs1_i)
                    : (exu_queue_vd      & idu2exu_use_rs1_ff);

assign mprf_rs1_addr = exu_queue_en ? idu2exu_cmd_i.rs1_addr : exu_queue.rs1_addr;
`else
assign mprf_rs1_req = exu_queue_vd & idu2exu_use_rs1_ff;
assign mprf_rs1_addr = exu_queue.rs1_addr;
`endif
```

在 `SCR1_MPRF_RAM` 配置下（同步 RAM），地址需要提前一周期给出：
- 当队列可装载新指令时，地址来自 IDU 端口
- 当队列保持旧指令时，地址来自队列寄存器

非 RAM 配置下（寄存器实现），地址直接来自队列寄存器，因为基于触发器的寄存器堆是组合读。

### 9.2 写回控制

```systemverilog
assign exu2mprf_w_req_o = (exu_queue.rd_wb_sel != SCR1_RD_WB_NONE)   // 有写的目标
                        & exu_queue_vd                                 // 队列有效
                        & ~exu_exc_req                                 // 无异常
`ifdef SCR1_DBG_EN
                        & ~hdu2exu_no_commit_i                         // HDU 允许提交
`endif
                        & ((exu_queue.rd_wb_sel == SCR1_RD_WB_CSR)
                           ? csr_access_init                           // CSR: 等待 CSR 读完成
                           : exu_rdy);                                 // 其他: EXU 就绪
```

关键约束：
- 异常发生时不写回 MPRF（`~exu_exc_req`）：异常导致流水线刷新，不应提交架构状态
- HDU `no_commit` 时不写回：调试模式下禁止指令提交
- CSR 写回特殊处理：需要 `csr_access_init`（即 CSR 读尚未完成时）才写回——因为 CSR 写回的时序在 CSR 读返回数据之后的第一周期

### 9.3 写回数据多路选择器

```systemverilog
case (exu_queue.rd_wb_sel)
    SCR1_RD_WB_SUM2   : exu2mprf_rd_data_o = ialu_addr_res;   // AUIPC: pc + imm
    SCR1_RD_WB_IMM    : exu2mprf_rd_data_o = exu_queue.imm;   // LUI: imm
    SCR1_RD_WB_INC_PC : exu2mprf_rd_data_o = inc_pc;           // JAL/JALR: pc+4
    SCR1_RD_WB_LSU    : exu2mprf_rd_data_o = lsu_l_data;       // LOAD: DMEM data
    SCR1_RD_WB_CSR    : exu2mprf_rd_data_o = csr2exu_r_data_i; // CSR: CSR read data
    default           : exu2mprf_rd_data_o = ialu_main_res;    // IALU 结果
endcase
```

default 分支（`SCR1_RD_WB_IALU`）覆盖所有算术/逻辑指令。

---

## 10. CSR 接口

### 10.1 CSR 读写请求生成

```systemverilog
always_comb begin
    if (~exu_queue_vd | tdu2exu_ibrkpt_exc_req_i) begin
        exu2csr_r_req_o = 1'b0;
        exu2csr_w_req_o = 1'b0;
    end else begin
        case (exu_queue.csr_cmd)
            SCR1_CSR_CMD_WRITE:
                exu2csr_r_req_o = |exu_queue.rd_addr;         // rd != x0 则需要读
                exu2csr_w_req_o = csr_access_init;
            SCR1_CSR_CMD_SET, SCR1_CSR_CMD_CLEAR:
                exu2csr_r_req_o = 1'b1;                        // SET/CLEAR 必须先读
                exu2csr_w_req_o = |exu_queue.rs1_addr & csr_access_init; // rs1 != x0
            default: ...
        endcase
    end
end
```

CSR 读写规则：

| CSR 命令 | 读请求条件 | 写请求条件 |
|------|------|------|
| WRITE (CSRRW) | `rd_addr != 0`（需要读旧值保存到 rd） | `csr_access_init`（CSR 访问周期中） |
| SET (CSRRS) | 始终读 | `rs1 != 0 & csr_access_init`（源操作数非零时才写） |
| CLEAR (CSRRC) | 始终读 | `rs1 != 0 & csr_access_init`（源操作数非零时才写） |

SET/CLEAR 指令必须先读取当前 CSR 值，再根据位掩码修改。若源操作数（rs1 或 zimm）为零，则写请求可被抑制以节省功耗。

### 10.2 CSR 访问 FSM

```systemverilog
typedef enum logic {
    SCR1_CSR_INIT,  // FSM 起始状态 / 访问周期进行中
    SCR1_CSR_RDY    // 访问已就绪
} scr1_csr_access_e;

always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)
        csr_access_ff <= SCR1_CSR_INIT;
    else
        csr_access_ff <= csr_access_next;
end

assign csr_access_next = (csr_access_init & csr2exu_mstatus_mie_up_i)
                       ? SCR1_CSR_RDY
                       : SCR1_CSR_INIT;

assign csr_access_init = (csr_access_ff == SCR1_CSR_INIT);
```

状态转移：

```
                        csr_access_init & mstatus_mie_up
        ┌────────────────────────────────────────────────┐
        │                                                │
        ▼                                                │
  ┌──────────────┐                             ┌──────────────────┐
  │ CSR_INIT     │────────────────────────────▶│  CSR_RDY         │
  │ (访问周期)   │ ◀────────────────────────  │  (数据已就绪)    │
  └──────────────┘     default (默认)          └──────────────────┘
```

FSM 默认停在 INIT 状态。仅在执行 CSR 指令且 `mstatus_mie_up` 为高的周期，FSM 跳转到 RDY 状态一周期，下一周期回到 INIT。

- `csr_access_init = 1` 期间：CSR 读写请求已发出，数据尚未稳定
- `csr_access_init = 0`（RDY）期间：CSR 读数据已返回，MPRF 写回在此周期进行

在 mstatus 更新周期（`csr2exu_mstatus_mie_up_i`），CSR 写操作可能需要额外等待以使 mstatus 变化生效。此时 FSM 多等一周期。

### 10.3 CSR 事件接口

**异常信号**：

```systemverilog
assign exu2csr_take_exc_o = exu_exc_req
                          & ~hdu2exu_dbg_halted_i;   // DBG: 调试暂停时不提交异常
assign exu2csr_exc_code_o = exc_code;
assign exu2csr_trap_val_o = exc_trap_val;
```

**中断接收**：

```systemverilog
assign exu2csr_take_irq_o = csr2exu_irq_i
                          & ~exu2pipe_exu_busy_o      // EXU 不忙
                          & ~hdu2exu_irq_dsbl_i       // 中断未禁用
                          & ~hdu2exu_dbg_halted_i     // 未在调试暂停
                          & clk_pipe_en;               // 时钟使能
```

中断仅在没有多周期操作进行中（`~exu2pipe_exu_busy_o`）时才能被接收，确保中断边界干净。

**MRET 信号**：

```systemverilog
assign exu2csr_mret_instr_o  = exu_queue_vd & exu_queue.mret_req
                             & ~tdu2exu_ibrkpt_exc_req_i     // 无 TDU 断点
                             & ~hdu2exu_dbg_halted_i;
assign exu2csr_mret_update_o = exu2csr_mret_instr_o & csr_access_init;
```

`exu2csr_mret_update_o`：在 CSR 访问 FSM 的 INIT 状态发出，CSR 侧用此信号更新 `mstatus` 和返回状态。

### 10.4 PC 下一值（给 CSR）

```systemverilog
assign exu2csr_pc_next_o = ~exu_queue_vd ? pc_curr_ff  // 无有效指令
                         : jb_taken      ? jb_new_pc    // 跳转目标
                                         : inc_pc;       // 顺序递增
```

CSR 侧在异常/中断发生时使用此 PC 值写入 `mepc`，用于后续 MRET 返回。

---

## 11. TDU 接口

### 11.1 指令监视器

```systemverilog
assign exu2tdu_imon_o.vd   = exu_queue_vd;
assign exu2tdu_imon_o.req  = exu2pipe_instret_o;
assign exu2tdu_imon_o.addr = pc_curr_ff;
```

指令监视器在队列有效时报告指令信息，在退休时上报。TDU 用此信息匹配断点触发条件。

### 11.2 断点退休标志

```systemverilog
always_comb begin
    exu2tdu_ibrkpt_ret_o = '0;
    if (exu_queue_vd) begin
        exu2tdu_ibrkpt_ret_o = tdu2exu_ibrkpt_match_i;
        if (lsu_req)
            exu2tdu_ibrkpt_ret_o[SCR1_TDU_MTRIG_NUM-1:0] |= tdu2lsu_dbrkpt_match_i;
    end
end
```

当有效指令退休时，将其命中的指令断点标志位广播回 TDU。若当前指令同时包含 LSU 操作，额外并入数据断点标志位（低 `MTRIG_NUM` 位）。

### 11.3 硬件断点综合

```systemverilog
assign exu2hdu_ibrkpt_hw_o = tdu2exu_ibrkpt_exc_req_i | tdu2lsu_dbrkpt_exc_req_i;
```

硬件断点信号同时进入异常请求综合（`exu_exc_req`）和异常编码优先级（最高优先级）。

---

## 12. SVA 断言

### 12.1 X 值检查

| 断言名 | 检查内容 |
|------|------|
| `SCR1_SVA_EXU_XCHECK_CTRL` | 控制信号（`idu2exu_req_i`, `csr2exu_irq_i`, `csr2exu_ip_ie_i`, `lsu_req`, `lsu_rdy`, `exu_exc_req`）无 X |
| `SCR1_SVA_EXU_XCHECK_QUEUE` | IDU 请求且队列有效时，`idu2exu_cmd_i` 无 X |
| `SCR1_SVA_EXU_XCHECK_CSR_RDATA` | CSR 读请求时，`csr2exu_r_data_i` 和 `csr2exu_rw_exc_i` 无 X |

### 12.2 行为检查

| 断言名 | 检查内容 |
|------|------|
| `SCR1_SVA_EXU_ONEHOT` | `jump_req`, `branch_req`, `lsu_req` 三者为 one-hot（一条指令不能同时是跳转+分支+LSU） |
| `SCR1_SVA_EXU_ONEHOT_EXC` | 多路异常源为 one-hot（同周期只能有一类异常） |
| `SCR1_SVA_EXU_CURR_PC_UPD_BEFORE_INIT` | `init_pc_v` 未满前，不得有 `pc_curr_upd & ~init_pc` |
| `SCR1_SVA_EXU_NEW_PC_REQ_BEFORE_INIT` | `init_pc_v` 未满前，不得有 `exu2ifu_pc_new_req_o & ~init_pc` |

所有断言在 `posedge clk` 前采样（`negedge clk`），使用 `disable iff (~rst_n)` 避免复位期间的误报。
