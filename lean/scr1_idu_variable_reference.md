# SCR1 IDU 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_idu.sv`
> 模块: `scr1_pipe_idu` — Instruction Decoder Unit

---

## 1. 局部参数 (localparam)

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_MPRF_ZERO_ADDR` | 5'd0 | 零寄存器 x0 的 MPRF 物理地址 |
| `SCR1_MPRF_RA_ADDR` | 5'd1 | 返回地址寄存器 x1(ra) 的 MPRF 物理地址 |
| `SCR1_MPRF_SP_ADDR` | 5'd2 | 栈指针寄存器 x2(sp) 的 MPRF 物理地址 |

---

## 2. 从外部头文件引入的类型

### 2.1 指令类型 — `type_scr1_instr_type_e`（`scr1_riscv_isa_decoding.svh`）

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_INSTR_RVC0` | 2'b00 | RVC 压缩指令象限 0（寄存器操作数在 abi' 区） |
| `SCR1_INSTR_RVC1` | 2'b01 | RVC 压缩指令象限 1（目标寄存器在 [11:7]） |
| `SCR1_INSTR_RVC2` | 2'b10 | RVC 压缩指令象限 2（栈指针和通用操作） |
| `SCR1_INSTR_RVI` | 2'b11 | 标准 32 位 RV32I 指令 |

### 2.2 RV32I 操作码 — `type_scr1_rvi_opcode_e`（`scr1_riscv_isa_decoding.svh`）

| 枚举值 | 编码 (instr[6:2]) | 含义 |
|--------|:---:|------|
| `SCR1_OPCODE_LOAD` | 5'b00000 | 加载指令 |
| `SCR1_OPCODE_MISC_MEM` | 5'b00011 | 存储器序指令 (FENCE/FENCE.I) |
| `SCR1_OPCODE_OP_IMM` | 5'b00100 | 立即数算术指令 |
| `SCR1_OPCODE_AUIPC` | 5'b00101 | AUIPC |
| `SCR1_OPCODE_STORE` | 5'b01000 | 存储指令 |
| `SCR1_OPCODE_OP` | 5'b01100 | 寄存器算术指令 |
| `SCR1_OPCODE_LUI` | 5'b01101 | LUI |
| `SCR1_OPCODE_BRANCH` | 5'b11000 | 条件分支 |
| `SCR1_OPCODE_JALR` | 5'b11001 | JALR |
| `SCR1_OPCODE_JAL` | 5'b11011 | JAL |
| `SCR1_OPCODE_SYSTEM` | 5'b11100 | 系统指令 (ECALL/EBREAK/MRET/WFI/CSR) |

### 2.3 EXU 命令结构体 — `type_scr1_exu_cmd_s`（`scr1_riscv_isa_decoding.svh`）

| 字段 | 类型 | 位宽 | 含义 |
|------|------|:---:|------|
| `instr_rvc` | logic | 1 | 压缩指令标志（IMEM 错误时复用为 RVI 高半取指错误标志） |
| `ialu_op` | `type_scr1_ialu_op_sel_e` | 1 | 整数 ALU 操作数选择（REG_IMM / REG_REG） |
| `ialu_cmd` | `type_scr1_ialu_cmd_sel_e` | 5 | 整数 ALU 运算命令（ADD/SUB/XOR/AND/OR/SLL/SRL/SRA/MUL/DIV...） |
| `sum2_op` | `type_scr1_ialu_sum2_op_sel_e` | 1 | 第二个加法器操作数选择（PC_IMM / REG_IMM） |
| `lsu_cmd` | `type_scr1_lsu_cmd_sel_e` | 4 | 加载/存储命令（LB/LH/LW/LBU/LHU/SB/SH/SW） |
| `csr_op` | `type_scr1_csr_op_sel_e` | 1 | CSR 操作数来源（IMM / REG） |
| `csr_cmd` | `type_scr1_csr_cmd_sel_e` | 2 | CSR 操作（WRITE / SET / CLEAR） |
| `rd_wb_sel` | `type_scr1_rd_wb_sel_e` | 3 | 回写到 rd 的数据来源（IALU/SUM2/IMM/INC_PC/LSU/CSR） |
| `jump_req` | logic | 1 | 无条件跳转请求（JAL/JALR/C.J/C.JAL/C.JR/C.JALR） |
| `branch_req` | logic | 1 | 条件分支请求 |
| `mret_req` | logic | 1 | MRET 异常返回请求 |
| `fencei_req` | logic | 1 | FENCE.I 指令缓存刷新请求 |
| `wfi_req` | logic | 1 | WFI 等待中断请求 |
| `rs1_addr` | logic [4:0] | 5 | 源寄存器 1 地址（CSRRxI 时复用为 zimm 零扩展值） |
| `rs2_addr` | logic [4:0] | 5 | 源寄存器 2 地址 |
| `rd_addr` | logic [4:0] | 5 | 目标寄存器地址 |
| `imm` | logic [31:0] | 32 | 立即数（CSR 时复用为 `{funct3, csr_addr}`，非法指令时复用为整条指令） |
| `exc_req` | logic | 1 | 异常请求 |
| `exc_code` | `type_scr1_exc_code_e` | 4 | 异常编码 |

---

## 3. 模块端口 (I/O)

### 3.1 仿真专用端口（仅 `SCR1_TRGT_SIMULATION`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `rst_n` | 1 | 异步复位，低有效（仅断言中 disable iff 使用） |
| input | `clk` | 1 | 时钟（仅断言采样使用） |

### 3.2 IFU ↔ IDU 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `idu2ifu_rdy_o` | 1 | IDU 准备好接收新指令（直通 `exu2idu_rdy_i`） |
| input | `ifu2idu_instr_i` | `SCR1_IMEM_DWIDTH` (32) | 来自 IFU 的 32 位指令原始编码 |
| input | `ifu2idu_imem_err_i` | 1 | 指令访问异常（IMEM 返回错误） |
| input | `ifu2idu_err_rvi_hi_i` | 1 | RVI 指令高半取指错误（IFU 跨边界场景） |
| input | `ifu2idu_vd_i` | 1 | IFU 输出指令有效标志 |

### 3.3 IDU ↔ EXU 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `idu2exu_req_o` | 1 | 译码结果有效（直通 `ifu2idu_vd_i`） |
| output | `idu2exu_cmd_o` | `type_scr1_exu_cmd_s` | 完整的 EXU 命令结构体 |
| output | `idu2exu_use_rs1_o` | 1 | 指令使用了 rs1（时钟门控用） |
| output | `idu2exu_use_rs2_o` | 1 | 指令使用了 rs2（时钟门控用） |
| output | `idu2exu_use_rd_o` | 1 | 指令使用了 rd（仅 `~SCR1_NO_EXE_STAGE`） |
| output | `idu2exu_use_imm_o` | 1 | 指令使用了立即数（仅 `~SCR1_NO_EXE_STAGE`） |
| input | `exu2idu_rdy_i` | 1 | EXU 准备好接收新命令（反压信号） |

---

## 4. 内部信号

### 4.1 指令译码核心信号

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `instr` | wire | 32 | 当前正在译码的指令（`ifu2idu_instr_i` 的别名） |
| `instr_type` | wire | `type_scr1_instr_type_e` | 指令类型（RVI / RVC0/1/2，由 `instr[1:0]` 译码） |
| `rvi_opcode` | wire | `type_scr1_rvi_opcode_e` | RVI 指令操作码（`instr[6:2]`，仅 RVI 有效） |
| `funct3` | wire | 3 | funct3 字段：RVI 取 `[14:12]`，RVC 取 `[15:13]` |
| `funct7` | wire | 7 | funct7 字段：`instr[31:25]`（仅 RVI 使用） |
| `funct12` | wire | 12 | funct12 字段：`instr[31:20]`（仅 RVI SYSTEM 指令使用） |
| `shamt` | wire | 5 | 移位量：`instr[24:20]`（SLLI/SRLI/SRAI 指令用） |

### 4.2 非法指令检测

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `rvi_illegal` | logic | 1 | RVI 指令非法（未定义操作码/funct3/funct7 组合） |
| `rvc_illegal` | logic | 1 | RVC 指令非法（仅 `SCR1_RVC_EXT` 定义时存在） |
| `rve_illegal` | logic | 1 | RVE 指令非法——寄存器地址超出 x0-x15 范围（仅 `SCR1_RVE_EXT` 定义时存在） |

---

## 5. 外部依赖头文件

| 文件名 | 提供的关键内容 |
|--------|------|
| `scr1_memif.svh` | 存储器接口类型（本模块不直接使用，但通过 IFU 间接关联） |
| `scr1_arch_types.svh` | `type_scr1_exc_code_e`（异常编码枚举）、`SCR1_MTVAL_ILLEGAL_INSTR_EN` 宏 |
| `scr1_riscv_isa_decoding.svh` | `type_scr1_instr_type_e`、`type_scr1_rvi_opcode_e`、`type_scr1_exu_cmd_s` 及所有子枚举类型 |
| `scr1_arch_description.svh` | `SCR1_XLEN`、`SCR1_IMEM_DWIDTH`、ISA 扩展使能宏（`SCR1_RVC_EXT`、`SCR1_RVM_EXT`、`SCR1_RVE_EXT`） |
