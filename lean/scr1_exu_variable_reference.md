# SCR1 EXU 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_exu.sv`
> 模块: `scr1_pipe_exu` — Execution Unit

---

## 1. 局部参数 (localparam)

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_JUMP_MASK` | `32'hFFFF_FFFE` | 跳转目标地址对齐掩码（低 1 位清零，强制半字对齐） |

---

## 2. 局部枚举类型 (typedef enum)

### 2.1 CSR 访问状态 — `scr1_csr_access_e`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_INIT` | 1'b0 | CSR 访问尚未完成（init 状态） |
| `SCR1_CSR_RDY` | 1'b1 | CSR 访问已就绪（rdy 状态，数据已写入） |

- 位宽: 1 bit (`enum logic`)
- `csr_access_init = (csr_access_ff == SCR1_CSR_INIT)`: 当 FSM 处于 INIT 状态时为高，表示 CSR 访问周期正在进行。

---

## 3. 从外部头文件引入的类型

### 3.1 EXU 命令结构体 — `type_scr1_exu_cmd_s` (`scr1_riscv_isa_decoding.svh`)

| 字段 | 类型 | 位宽 | 含义 |
|------|------|:---:|------|
| `instr_rvc` | logic | 1 | RVC 压缩指令标志 |
| `ialu_op` | `type_scr1_ialu_op_sel_e` | 1 | IALU 操作数选择 (REG_IMM / REG_REG) |
| `ialu_cmd` | `type_scr1_ialu_cmd_sel_e` | 5 | IALU 运算命令 |
| `sum2_op` | `type_scr1_ialu_sum2_op_sel_e` | 1 | 地址加法器操作数选择 (PC_IMM / REG_IMM) |
| `lsu_cmd` | `type_scr1_lsu_cmd_sel_e` | 4 | LSU 加载存储命令 |
| `csr_op` | `type_scr1_csr_op_sel_e` | 1 | CSR 操作数来源 (IMM / REG) |
| `csr_cmd` | `type_scr1_csr_cmd_sel_e` | 2 | CSR 操作命令 (WRITE / SET / CLEAR) |
| `rd_wb_sel` | `type_scr1_rd_wb_sel_e` | 3 | 回写数据来源选择 |
| `jump_req` | logic | 1 | 无条件跳转请求 |
| `branch_req` | logic | 1 | 条件分支请求 |
| `mret_req` | logic | 1 | MRET 异常返回请求 |
| `fencei_req` | logic | 1 | FENCE.I 请求 |
| `wfi_req` | logic | 1 | WFI 等待中断请求 |
| `exc_req` | logic | 1 | 异常请求（来自 IDU 译码） |
| `exc_code` | `type_scr1_exc_code_e` | 4 | 异常编码 |
| `rs1_addr` | logic [4:0] | 5 | 源寄存器 1 地址 |
| `rs2_addr` | logic [4:0] | 5 | 源寄存器 2 地址 |
| `rd_addr` | logic [4:0] | 5 | 目标寄存器地址 |
| `imm` | logic [31:0] | 32 | 立即数 |

### 3.2 异常编码 — `type_scr1_exc_code_e` (`scr1_arch_types.svh`)

| 枚举值 | 含义 | EXU 触发场景 |
|--------|------|-------------|
| `SCR1_EXC_CODE_INSTR_MISALIGN` | 指令地址未对齐 | 非 RVC 扩展下的跳转目标地址未对齐 |
| `SCR1_EXC_CODE_INSTR_ACCESS_FAULT` | 指令访问错误 | IMEM 返回错误（由 IDU 转发至 exc_code） |
| `SCR1_EXC_CODE_ILLEGAL_INSTR` | 非法指令 | CSR 读写异常 / IDU 译码非法指令 |
| `SCR1_EXC_CODE_ECALL_M` | M 模式环境调用 | ECALL 指令 |
| `SCR1_EXC_CODE_BREAKPOINT` | 断点异常 | TDU 硬件指令断点或数据断点触发 |
| `SCR1_EXC_CODE_LD_ADDR_MISALIGN` | 加载地址未对齐 | LSU 检测到未对齐的加载地址 |
| `SCR1_EXC_CODE_LD_ACCESS_FAULT` | 加载访问错误 | LSU DMEM 读取错误 |
| `SCR1_EXC_CODE_ST_ADDR_MISALIGN` | 存储地址未对齐 | LSU 检测到未对齐的存储地址 |
| `SCR1_EXC_CODE_ST_ACCESS_FAULT` | 存储访问错误 | LSU DMEM 写入错误 |

### 3.3 CSR 命令选择 — `type_scr1_csr_cmd_sel_e` (`scr1_csr.svh`)

| 枚举值 | 含义 |
|--------|------|
| `SCR1_CSR_CMD_NONE` | 无效 / 非 CSR 指令 |
| `SCR1_CSR_CMD_WRITE` | CSR 写操作 (CSRRW/CSRRWI) |
| `SCR1_CSR_CMD_SET` | CSR 置位操作 (CSRRS/CSRRSI) |
| `SCR1_CSR_CMD_CLEAR` | CSR 清零操作 (CSRRC/CSRRCI) |

### 3.4 LSU 命令选择 — `type_scr1_lsu_cmd_sel_e` (`scr1_riscv_isa_decoding.svh`)

| 枚举值 | 含义 |
|--------|------|
| `SCR1_LSU_CMD_NONE` | 无 LSU 操作 |

### 3.5 回写数据来源 — `type_scr1_rd_wb_sel_e` (`scr1_riscv_isa_decoding.svh`)

| 枚举值 | 含义 |
|--------|------|
| `SCR1_RD_WB_NONE` | 不写回（无 rd 目标寄存器） |
| `SCR1_RD_WB_IALU` | 写回来自主 ALU 的结果 |
| `SCR1_RD_WB_SUM2` | 写回来自地址加法器的结果 (AUIPC) |
| `SCR1_RD_WB_IMM` | 写回立即数 (LUI) |
| `SCR1_RD_WB_INC_PC` | 写回递增后的 PC (JAL/JALR) |
| `SCR1_RD_WB_LSU` | 写回 LSU 加载数据 |
| `SCR1_RD_WB_CSR` | 写回 CSR 读取数据 |

### 3.6 TDU 相关类型 (`scr1_tdu.svh`)

| 类型 | 用途 |
|------|------|
| `type_scr1_brkm_instr_mon_s` | 指令监视器结构体：包含 `vd`（有效）、`req`（退休）、`addr`（地址）字段 |
| `type_scr1_brkm_lsu_mon_s` | LSU 数据监视器结构体 |
| `SCR1_TDU_ALLTRIG_NUM` | 全部触发器总数 |
| `SCR1_TDU_MTRIG_NUM` | 匹配触发器数 |

---

## 4. 模块端口 (I/O)

### 4.1 通用控制信号

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `rst_n` | 1 | 异步复位，低有效 |
| input | `clk` | 1 | EXU 门控时钟 |
| input | `clk_alw_on` | 1 | 非门控始终开启时钟（仅 `SCR1_CLKCTRL_EN`） |
| input | `clk_pipe_en` | 1 | 流水线时钟使能（仅 `SCR1_CLKCTRL_EN`） |

### 4.2 EXU ↔ IDU 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `idu2exu_req_i` | 1 | IDU 向 EXU 发出的指令请求 |
| output | `exu2idu_rdy_o` | 1 | EXU 就绪，可接收新指令（反压信号：`exu_rdy & ~exu_queue_barrier`） |
| input | `idu2exu_cmd_i` | `type_scr1_exu_cmd_s` | IDU 译码后的指令命令结构体 |
| input | `idu2exu_use_rs1_i` | 1 | 指令使用了 rs1（时钟门控标志） |
| input | `idu2exu_use_rs2_i` | 1 | 指令使用了 rs2（时钟门控标志） |
| input | `idu2exu_use_rd_i` | 1 | 指令使用了 rd（仅 `~SCR1_NO_EXE_STAGE`） |
| input | `idu2exu_use_imm_i` | 1 | 指令使用了立即数（仅 `~SCR1_NO_EXE_STAGE`） |

### 4.3 EXU ↔ MPRF 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2mprf_rs1_addr_o` | `SCR1_MPRF_AWIDTH` | MPRF rs1 读地址 |
| input | `mprf2exu_rs1_data_i` | `SCR1_XLEN` | MPRF rs1 读数据 |
| output | `exu2mprf_rs2_addr_o` | `SCR1_MPRF_AWIDTH` | MPRF rs2 读地址 |
| input | `mprf2exu_rs2_data_i` | `SCR1_XLEN` | MPRF rs2 读数据 |
| output | `exu2mprf_w_req_o` | 1 | MPRF 写请求 |
| output | `exu2mprf_rd_addr_o` | `SCR1_MPRF_AWIDTH` | MPRF rd 写地址 |
| output | `exu2mprf_rd_data_o` | `SCR1_XLEN` | MPRF rd 写数据 |

### 4.4 EXU ↔ CSR 读写接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2csr_rw_addr_o` | `SCR1_CSR_ADDR_WIDTH` (12) | CSR 读写地址 |
| output | `exu2csr_r_req_o` | 1 | CSR 读请求 |
| input | `csr2exu_r_data_i` | `SCR1_XLEN` | CSR 读返回数据 |
| output | `exu2csr_w_req_o` | 1 | CSR 写请求 |
| output | `exu2csr_w_cmd_o` | `type_scr1_csr_cmd_sel_e` | CSR 写命令 (WRITE/SET/CLEAR) |
| output | `exu2csr_w_data_o` | `SCR1_XLEN` | CSR 写数据 |
| input | `csr2exu_rw_exc_i` | 1 | CSR 读写访问异常（非法 CSR 地址） |

### 4.5 EXU ↔ CSR 事件接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2csr_take_irq_o` | 1 | 接收中断请求 |
| output | `exu2csr_take_exc_o` | 1 | 接收异常请求 |
| output | `exu2csr_mret_update_o` | 1 | MRET 更新 CSR |
| output | `exu2csr_mret_instr_o` | 1 | MRET 指令执行标志 |
| output | `exu2csr_exc_code_o` | `type_scr1_exc_code_e` | 异常编码 |
| output | `exu2csr_trap_val_o` | `SCR1_XLEN` | 异常陷阱值 (mtval) |
| input | `csr2exu_new_pc_i` | `SCR1_XLEN` | 异常/IRQ/MRET 目标 PC (mtvec/mepc) |
| input | `csr2exu_irq_i` | 1 | 中断请求 |
| input | `csr2exu_ip_ie_i` | 1 | 有中断挂起且本地中断使能 |
| input | `csr2exu_mstatus_mie_up_i` | 1 | 当前周期 MSTATUS 或 MIE 被更新 |

### 4.6 EXU ↔ DMEM 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2dmem_req_o` | 1 | DMEM 请求 |
| output | `exu2dmem_cmd_o` | `type_scr1_mem_cmd_e` | DMEM 命令（读/写） |
| output | `exu2dmem_width_o` | `type_scr1_mem_width_e` | DMEM 数据宽度 (B/H/W) |
| output | `exu2dmem_addr_o` | `SCR1_DMEM_AWIDTH` | DMEM 地址 |
| output | `exu2dmem_wdata_o` | `SCR1_DMEM_DWIDTH` | DMEM 写数据 |
| input | `dmem2exu_req_ack_i` | 1 | DMEM 请求应答 |
| input | `dmem2exu_rdata_i` | `SCR1_DMEM_DWIDTH` | DMEM 读数据 |
| input | `dmem2exu_resp_i` | `type_scr1_mem_resp_e` | DMEM 响应状态 |

### 4.7 EXU 控制输出

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2pipe_exc_req_o` | 1 | 最后一条指令发生异常 |
| output | `exu2pipe_brkpt_o` | 1 | 软件断点 (EBREAK) |
| output | `exu2pipe_init_pc_o` | 1 | 复位序列完成，PC 已初始化 |
| output | `exu2pipe_wfi_run2halt_o` | 1 | 进入 WFI 暂停状态 |
| output | `exu2pipe_instret_o` | 1 | 指令退休（含异常） |
| output | `exu2csr_instret_no_exc_o` | 1 | 指令退休（不含异常，仅 `~SCR1_CSR_REDUCED_CNT`） |
| output | `exu2pipe_exu_busy_o` | 1 | EXU 忙（多周期操作进行中） |

### 4.8 EXU ↔ HDU 接口（仅 `SCR1_DBG_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `hdu2exu_no_commit_i` | 1 | 禁止指令提交 |
| input | `hdu2exu_irq_dsbl_i` | 1 | 禁止中断 |
| input | `hdu2exu_pc_advmt_dsbl_i` | 1 | 禁止 PC 推进 |
| input | `hdu2exu_dmode_sstep_en_i` | 1 | 启用单步调试 (dmode step) |
| input | `hdu2exu_pbuf_fetch_i` | 1 | 从 Program Buffer 取指 |
| input | `hdu2exu_dbg_halted_i` | 1 | 调试暂停状态 |
| input | `hdu2exu_dbg_run2halt_i` | 1 | 进入调试暂停状态 |
| input | `hdu2exu_dbg_halt2run_i` | 1 | 退出调试暂停状态 |
| input | `hdu2exu_dbg_run_start_i` | 1 | 运行状态的第一周期 |
| input | `hdu2exu_dbg_new_pc_i` | `SCR1_XLEN` | 恢复运行后的起始 PC |

### 4.9 EXU ↔ TDU 接口（仅 `SCR1_TDU_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2tdu_imon_o` | `type_scr1_brkm_instr_mon_s` | 指令监视器 |
| input | `tdu2exu_ibrkpt_match_i` | `SCR1_TDU_ALLTRIG_NUM` | 指令断点匹配 |
| input | `tdu2exu_ibrkpt_exc_req_i` | 1 | 指令断点异常请求 |
| output | `lsu2tdu_dmon_o` | `type_scr1_brkm_lsu_mon_s` | LSU 数据监视器（从 LSU 直连） |
| input | `tdu2lsu_ibrkpt_exc_req_i` | 1 | 指令断点异常（给 LSU） |
| input | `tdu2lsu_dbrkpt_match_i` | `SCR1_TDU_MTRIG_NUM` | 数据断点匹配 |
| input | `tdu2lsu_dbrkpt_exc_req_i` | 1 | 数据断点异常请求 |
| output | `exu2tdu_ibrkpt_ret_o` | `SCR1_TDU_ALLTRIG_NUM` | 带断点标志的指令退休 |
| output | `exu2hdu_ibrkpt_hw_o` | 1 | 当前指令有硬件断点（仅 `SCR1_DBG_EN`） |

### 4.10 PC 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `exu2pipe_wfi_halted_o` | 1 | WFI 暂停状态（仅 `SCR1_CLKCTRL_EN`） |
| output | `exu2pipe_pc_curr_o` | `SCR1_XLEN` | 当前 PC |
| output | `exu2csr_pc_next_o` | `SCR1_XLEN` | 下一条指令 PC（给 CSR 用于 mepc） |
| output | `exu2ifu_pc_new_req_o` | 1 | 新 PC 请求（通知 IFU 跳转） |
| output | `exu2ifu_pc_new_o` | `SCR1_XLEN` | 新 PC 值 |

---

## 5. 局部信号 (local signals)

### 5.1 指令队列信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `exu_queue_vd` | 1 | 指令队列中的指令有效 |
| `exu_queue` | `type_scr1_exu_cmd_s` | 指令队列寄存器（当前执行指令的 EXU 命令） |
| `exu_queue_barrier` | 1 | 队列屏障（暂停接收新指令） |
| `dbg_run_start_npbuf` | 1 | 调试运行开始且不是从 Program Buffer 取指（仅 `SCR1_DBG_EN`） |
| `exu_queue_en` | 1 | 队列使能/装载（`exu2idu_rdy_o & idu2exu_req_i`） |
| `exu_illegal_instr` | 32 | 用于 mtval 的非法指令编码（合成 SYSTEM 操作码格式） |
| `idu2exu_use_rs1_ff` | 1 | 上一周期 use_rs1 标志寄存器（仅 `~SCR1_NO_EXE_STAGE`） |
| `idu2exu_use_rs2_ff` | 1 | 上一周期 use_rs2 标志寄存器（仅 `~SCR1_NO_EXE_STAGE`） |
| `exu_queue_vd_upd` | 1 | 队列有效标志更新条件（仅 `~SCR1_NO_EXE_STAGE`） |
| `exu_queue_vd_ff` | 1 | 队列有效标志寄存器（仅 `~SCR1_NO_EXE_STAGE`） |
| `exu_queue_vd_next` | 1 | 队列有效标志下一值（仅 `~SCR1_NO_EXE_STAGE`） |

### 5.2 IALU 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `ialu_rdy` | 1 | IALU 就绪（仅 `SCR1_RVM_EXT`，乘除法多周期） |
| `ialu_vd` | 1 | IALU 操作有效（仅 `SCR1_RVM_EXT`） |
| `ialu_main_op1` | `SCR1_XLEN` | IALU 主操作数 1（来自 MPRF rs1） |
| `ialu_main_op2` | `SCR1_XLEN` | IALU 主操作数 2（来自 MPRF rs2 或 imm） |
| `ialu_main_res` | `SCR1_XLEN` | IALU 主结果 |
| `ialu_addr_op1` | `SCR1_XLEN` | 地址加法器操作数 1（来自 MPRF rs1 或 PC） |
| `ialu_addr_op2` | `SCR1_XLEN` | 地址加法器操作数 2（来自 imm） |
| `ialu_addr_res` | `SCR1_XLEN` | 地址加法器结果 |
| `ialu_cmp` | 1 | IALU 比较结果（分支判断用） |

### 5.3 异常信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `exu_exc_req` | 1 | 综合异常请求（含 IDU 译码异常、LSU 异常、CSR 异常、未对齐异常、TDU 断点） |
| `exu_exc_req_ff` | 1 | 异常请求寄存器（仅 `SCR1_DBG_EN`，用于无有效队列时的异常保持） |
| `exu_exc_req_next` | 1 | 异常请求下一值（仅 `SCR1_DBG_EN`） |
| `exc_code` | `type_scr1_exc_code_e` | 最终异常编码（优先级多路选择器输出） |
| `exc_trap_val` | `SCR1_XLEN` | 异常陷阱值（对应 mtval CSR） |
| `instr_fault_rvi_hi` | 1 | RVI 指令高半取指错误标志（复用 `instr_rvc` 字段） |

### 5.4 WFI 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `wfi_halt_cond` | 1 | WFI 暂停条件满足 |
| `wfi_run_req` | 1 | WFI 运行请求（退出暂停） |
| `wfi_halt_req` | 1 | WFI 暂停请求（进入暂停） |
| `wfi_run_start_ff` | 1 | WFI 运行启动标志寄存器 |
| `wfi_run_start_next` | 1 | WFI 运行启动标志下一值 |
| `wfi_halted_upd` | 1 | WFI 暂停标志更新条件 |
| `wfi_halted_ff` | 1 | WFI 暂停状态标志寄存器 |
| `wfi_halted_next` | 1 | WFI 暂停状态标志下一值 |

### 5.5 PC 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `init_pc_v` | 4 | PC 初始化移位寄存器（4 位移位链） |
| `init_pc` | 1 | PC 初始化正在进行（复位序列中） |
| `inc_pc` | `SCR1_XLEN` | 顺序递增 PC（RVC: +2/+4, 非 RVC: +4） |
| `branch_taken` | 1 | 条件分支成立（branch_req & ialu_cmp） |
| `jb_taken` | 1 | 跳转或分支成立（jump_req | branch_taken） |
| `jb_new_pc` | `SCR1_XLEN` | 跳转/分支目标地址（地址加法器结果 & JUMP_MASK） |
| `jb_misalign` | 1 | 跳转目标地址未对齐（仅 `~SCR1_RVC_EXT`） |
| `pc_curr_upd` | 1 | 当前 PC 更新使能 |
| `pc_curr_ff` | `SCR1_XLEN` | 当前 PC 寄存器 |
| `pc_curr_next` | `SCR1_XLEN` | 当前 PC 下一值 |

### 5.6 LSU 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `lsu_req` | 1 | LSU 操作请求（lsu_cmd != NONE 且队列有效） |
| `lsu_rdy` | 1 | LSU 就绪（操作完成） |
| `lsu_l_data` | `SCR1_XLEN` | LSU 加载数据（给 MPRF 写回通路） |
| `lsu_exc_req` | 1 | LSU 异常请求 |
| `lsu_exc_code` | `type_scr1_exc_code_e` | LSU 异常编码 |

### 5.7 EXU 状态信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `exu_rdy` | 1 | EXU 就绪（当前指令执行完成） |

### 5.8 MPRF 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mprf_rs1_req` | 1 | MPRF rs1 读请求 |
| `mprf_rs2_req` | 1 | MPRF rs2 读请求 |
| `mprf_rs1_addr` | `SCR1_MPRF_AWIDTH` | MPRF rs1 读地址（内部） |
| `mprf_rs2_addr` | `SCR1_MPRF_AWIDTH` | MPRF rs2 读地址（内部） |

### 5.9 CSR 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `csr_access_ff` | `scr1_csr_access_e` | CSR 访问状态寄存器 |
| `csr_access_next` | `scr1_csr_access_e` | CSR 访问状态下一值 |
| `csr_access_init` | 1 | CSR 访问处于 INIT 状态（访问周期进行中） |

### 5.10 仿真专用信号（仅 `SCR1_TRGT_SIMULATION`）

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `update_pc` | `SCR1_XLEN` | Tracelog 用：更新的 PC 值 |
| `update_pc_en` | 1 | Tracelog 用：PC 更新使能 |

---

## 6. 子模块实例化

| 实例名 | 模块 | 用途 |
|--------|------|------|
| `i_ialu` | `scr1_pipe_ialu` | 整数算术逻辑单元（主 ALU + 地址加法器 + 乘除法器） |
| `i_lsu` | `scr1_pipe_lsu` | 加载存储单元 |
