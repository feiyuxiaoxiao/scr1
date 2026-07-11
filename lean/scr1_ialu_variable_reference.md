# SCR1 IALU 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_ialu.sv`
> 模块: `scr1_pipe_ialu` — Integer Arithmetic Logic Unit

---

## 1. 局部参数 (localparam)

### 1.1 通用参数

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_IALU_CMD_ALL_NUM_E` | 15 (无 RVM) / 23 (有 RVM) | IALU 命令总数 |
| `SCR1_IALU_CMD_WIDTH_E` | `$clog2(CMD_ALL_NUM_E)` | IALU 命令编码位宽 |

### 1.2 快速乘法器参数 (`SCR1_FAST_MUL`)

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_MUL_WIDTH` | `SCR1_XLEN` (32) | 乘法器操作数位宽 |
| `SCR1_MUL_RES_WIDTH` | `2 × SCR1_XLEN` (64) | 乘法结果位宽（双倍宽） |
| `SCR1_MDU_SUM_WIDTH` | `SCR1_XLEN + 1` (33) | MDU 加法器位宽（含进/借位） |

### 1.3 非快速乘法器参数 (Radix-2, ~SCR1_FAST_MUL)

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_MUL_STG_NUM` | 32 | 乘法迭代级数（等于 XLEN） |
| `SCR1_MUL_WIDTH` | `32 / 32 = 1` | 每拍处理 1 位乘法器 |
| `SCR1_MUL_CNT_INIT` | `32'b1 << (32/1 - 2)` = `32'h4000_0000` | 乘法迭代计数器初值（第 31 位为 1） |
| `SCR1_MDU_SUM_WIDTH` | `SCR1_XLEN + SCR1_MUL_WIDTH` = 33 | MDU 加法器位宽 |

### 1.4 除法器参数

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_DIV_WIDTH` | 1 | 每拍处理 1 位的除法迭代宽度 |
| `SCR1_DIV_CNT_INIT` | `32'b1 << (32/1 - 2)` = `32'h4000_0000` | 除法迭代计数器初值（第 31 位为 1） |

---

## 2. 枚举类型 (typedef enum)

### 2.1 IALU 标志位结构体 — `type_scr1_ialu_flags_s` (packed struct)

| 字段 | 类型 | 含义 |
|------|------|------|
| `z` | logic | 零标志：结果为 0 时置 1 |
| `s` | logic | 符号标志：结果的符号位（最高位） |
| `o` | logic | 溢出标志：有符号加减发生溢出 |
| `c` | logic | 进位/借位标志：减法借位 = 不进位 |

### 2.2 MDU FSM 状态 — `type_scr1_ialu_fsm_state`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_IALU_MDU_FSM_IDLE` | 2'b00 | 空闲状态：等待乘除法命令 |
| `SCR1_IALU_MDU_FSM_ITER` | 2'b01 | 迭代状态：执行迭代运算 |
| `SCR1_IALU_MDU_FSM_CORR` | 2'b10 | 修正状态：有符号除法结果修正 |

- 位宽: 2 bit (`enum logic[1:0]`)

### 2.3 MDU 命令类型 — `type_scr1_ialu_mdu_cmd`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_IALU_MDU_NONE` | 2'b00 | 无乘除操作 |
| `SCR1_IALU_MDU_MUL` | 2'b01 | 乘法操作 |
| `SCR1_IALU_MDU_DIV` | 2'b10 | 除法/取余操作 |

- 位宽: 2 bit (`enum logic[1:0]`)

### 2.4 IALU 命令枚举 — `type_scr1_ialu_cmd_sel_e`（外部引入）

| 枚举值 | 编码 | 对应指令 | 含义 |
|--------|:---:|------|------|
| `SCR1_IALU_CMD_NONE` | 5'h00 | — | 禁用 IALU |
| `SCR1_IALU_CMD_AND` | 5'h01 | AND/ANDI | 按位与 |
| `SCR1_IALU_CMD_OR` | 5'h02 | OR/ORI | 按位或 |
| `SCR1_IALU_CMD_XOR` | 5'h03 | XOR/XORI | 按位异或 |
| `SCR1_IALU_CMD_ADD` | 5'h04 | ADD/ADDI | 加法 |
| `SCR1_IALU_CMD_SUB` | 5'h05 | SUB | 减法 |
| `SCR1_IALU_CMD_SUB_LT` | 5'h06 | SLT/SLTI | 有符号小于比较 |
| `SCR1_IALU_CMD_SUB_LTU` | 5'h07 | SLTU/SLTIU | 无符号小于比较 |
| `SCR1_IALU_CMD_SUB_EQ` | 5'h08 | BEQ | 相等比较 |
| `SCR1_IALU_CMD_SUB_NE` | 5'h09 | BNE | 不等比较 |
| `SCR1_IALU_CMD_SUB_GE` | 5'h0A | BGE | 有符号大于等于 |
| `SCR1_IALU_CMD_SUB_GEU` | 5'h0B | BGEU | 无符号大于等于 |
| `SCR1_IALU_CMD_SLL` | 5'h0C | SLL/SLLI | 逻辑左移 |
| `SCR1_IALU_CMD_SRL` | 5'h0D | SRL/SRLI | 逻辑右移 |
| `SCR1_IALU_CMD_SRA` | 5'h0E | SRA/SRAI | 算术右移 |
| `SCR1_IALU_CMD_MUL` | 5'h0F | MUL | 乘法（取低 32 位） |
| `SCR1_IALU_CMD_MULHU` | 5'h10 | MULHU | 无符号乘法（取高 32 位） |
| `SCR1_IALU_CMD_MULHSU` | 5'h11 | MULHSU | 有符号×无符号乘法（取高 32 位） |
| `SCR1_IALU_CMD_MULH` | 5'h12 | MULH | 有符号乘法（取高 32 位） |
| `SCR1_IALU_CMD_DIV` | 5'h13 | DIV | 有符号除法 |
| `SCR1_IALU_CMD_DIVU` | 5'h14 | DIVU | 无符号除法 |
| `SCR1_IALU_CMD_REM` | 5'h15 | REM | 有符号取余 |
| `SCR1_IALU_CMD_REMU` | 5'h16 | REMU | 无符号取余 |

- 位宽: `SCR1_IALU_CMD_WIDTH_E` = `$clog2(15)`=4 或 `$clog2(23)`=5
- 定义于: `scr1_riscv_isa_decoding.svh:59-86`

---

## 3. 模块端口 (I/O)

### 3.1 M 扩展专用端口（`SCR1_RVM_EXT`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `clk` | 1 | IALU 时钟 |
| input | `rst_n` | 1 | 异步复位，低有效 |
| input | `exu2ialu_rvm_cmd_vd_i` | 1 | MUL/DIV 命令有效（来自 EXU） |
| output | `ialu2exu_rvm_res_rdy_o` | 1 | MUL/DIV 结果就绪（去往 EXU） |

### 3.2 主加法器接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `exu2ialu_main_op1_i` | `SCR1_XLEN-1:0` | 主 ALU 第一操作数 |
| input | `exu2ialu_main_op2_i` | `SCR1_XLEN-1:0` | 主 ALU 第二操作数 |
| input | `exu2ialu_cmd_i` | `SCR1_IALU_CMD_WIDTH_E-1:0` | IALU 运算命令 |
| output | `ialu2exu_main_res_o` | `SCR1_XLEN-1:0` | 主 ALU 运算结果 |
| output | `ialu2exu_cmp_res_o` | 1 | IALU 比较结果（用于分支判定） |

### 3.3 地址加法器接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `exu2ialu_addr_op1_i` | `SCR1_XLEN-1:0` | 地址加法器第一操作数 |
| input | `exu2ialu_addr_op2_i` | `SCR1_XLEN-1:0` | 地址加法器第二操作数 |
| output | `ialu2exu_addr_res_o` | `SCR1_XLEN-1:0` | 地址加法器结果 |

---

## 4. 局部信号 (Local Signals)

### 4.1 主加法器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `main_sum_res` | `SCR1_XLEN:0` (33) | 主加法器结果（含进位/借位位） |
| `main_sum_flags` | `type_scr1_ialu_flags_s` | 主加法器标志位（z/s/o/c） |
| `main_sum_pos_ovflw` | 1 | 正溢出：两操作数非负，结果变负 |
| `main_sum_neg_ovflw` | 1 | 负溢出：两操作数非正，结果变正 |
| `main_ops_diff_sgn` | 1 | 主加法器两操作数异号 |
| `main_ops_non_zero` | 1 | 主加法器两操作数均非零 |

### 4.2 移位器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `ialu_cmd_shft` | 1 | 当前 IALU 命令为移位操作 |
| `shft_op1` | `SCR1_XLEN-1:0` (signed) | 移位操作数 1（被移位数） |
| `shft_op2` | 4:0 | 移位操作数 2（移位量，低 5 位） |
| `shft_cmd` | 1:0 | 移位命令编码：00=左移, 10=逻辑右移, 11=算术右移 |
| `shft_res` | `SCR1_XLEN-1:0` | 移位结果 |

### 4.3 MDU FSM 控制信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mdu_cmd_is_iter` | 1 | MDU 命令需要迭代执行 |
| `mdu_iter_req` | 1 | 请求进入迭代阶段 |
| `mdu_iter_rdy` | 1 | 迭代完成（迭代计数器 LSB = 1） |
| `mdu_corr_req` | 1 | 除法/取余需要结果修正 |
| `div_corr_req` | 1 | DIV 需要修正（被除数与除数异号） |
| `rem_corr_req` | 1 | REM(U) 需要修正（余数非零且符号与预期不符） |

### 4.4 MDU FSM 信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mdu_fsm_ff` | `type_scr1_ialu_fsm_state` | 当前 FSM 状态（寄存器） |
| `mdu_fsm_next` | `type_scr1_ialu_fsm_state` | 下一 FSM 状态（组合逻辑） |
| `mdu_fsm_idle` | 1 | FSM 处于 IDLE 状态 |
| `mdu_fsm_iter` | 1 | FSM 处于 ITER 状态（仅仿真） |
| `mdu_fsm_corr` | 1 | FSM 处于 CORR 状态 |

### 4.5 MDU 命令译码信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mdu_cmd` | `type_scr1_ialu_mdu_cmd` | MDU 命令：NONE/MUL/DIV |
| `mdu_cmd_mul` | 1 | MDU 命令为乘法（MUL/MULH/MULHSU/MULHU） |
| `mdu_cmd_div` | 1 | MDU 命令为除法/取余（DIV/DIVU/REM/REMU） |
| `mul_cmd` | 1:0 | 乘法子命令：00=MUL, 01=MULH, 10=MULHSU, 11=MULHU |
| `mul_cmd_hi` | 1 | 需要乘法结果高位部分 |
| `div_cmd` | 1:0 | 除法子命令：00=DIV, 01=DIVU, 10=REM, 11=REMU |
| `div_cmd_div` | 1 | 除法命令为 DIV（非 REM） |
| `div_cmd_rem` | 1 | 除法命令为 REM(U) |

### 4.6 乘法器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mul_op1_is_sgn` | 1 | 乘法操作数 1 为有符号 |
| `mul_op2_is_sgn` | 1 | 乘法操作数 2 为有符号 |
| `mul_op1_sgn` | 1 | 乘法操作数 1 为负数 |
| `mul_op2_sgn` | 1 | 乘法操作数 2 为负数 |
| `mul_op1` | `SCR1_XLEN:0` (signed) | 乘法操作数 1（含符号扩展位） |
| `mul_op2` | `SCR1_MUL_WIDTH:0` (signed) | 乘法操作数 2（位宽取决于配置） |
| `mul_res` | `SCR1_MUL_RES_WIDTH-1:0` (signed) | 乘法结果（仅 FAST_MUL） |
| `mul_part_prod` | `SCR1_MDU_SUM_WIDTH:0` (signed) | 部分积（仅非 FAST_MUL） |
| `mul_res_hi` | `SCR1_XLEN-1:0` | 乘法结果高位 (Radix-2) |
| `mul_res_lo` | `SCR1_XLEN-1:0` | 乘法结果低位 (Radix-2) |

### 4.7 除法器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `div_ops_are_sgn` | 1 | 除数为有符号运算 |
| `div_op1_is_neg` | 1 | 被除数为负 |
| `div_op2_is_neg` | 1 | 除数为负 |
| `div_res_rem_c` | 1 | 除法余数比较结果的借位/进位标志 |
| `div_res_rem` | `SCR1_XLEN-1:0` | 余数寄存器 |
| `div_res_quo` | `SCR1_XLEN-1:0` | 商寄存器 |
| `div_quo_bit` | 1 | 当前迭代产生的商位 |
| `div_dvdnd_lo_upd` | 1 | 被除数低位寄存器更新使能 |
| `div_dvdnd_lo_ff` | `SCR1_XLEN-1:0` | 被除数低位寄存器 |
| `div_dvdnd_lo_next` | `SCR1_XLEN-1:0` | 被除数低位寄存器下一值 |

### 4.8 MDU 加法器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mdu_sum_sub` | 1 | MDU 加法器做减法（1）还是加法（0） |
| `mdu_sum_op1` | `SCR1_MDU_SUM_WIDTH-1:0` (signed) | MDU 加法器操作数 1 |
| `mdu_sum_op2` | `SCR1_MDU_SUM_WIDTH-1:0` (signed) | MDU 加法器操作数 2 |
| `mdu_sum_res` | `SCR1_MDU_SUM_WIDTH-1:0` (signed) | MDU 加法器结果 |

### 4.9 MDU 迭代计数器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mdu_iter_cnt_en` | 1 | 迭代计数器更新使能 |
| `mdu_iter_cnt` | `SCR1_XLEN-1:0` | 迭代计数器（寄存器） |
| `mdu_iter_cnt_next` | `SCR1_XLEN-1:0` | 迭代计数器下一值 |

### 4.10 中间结果寄存器信号

| 信号名 | 位宽 | 含义 |
|--------|------|------|
| `mdu_res_upd` | 1 | 中间结果寄存器更新使能 |
| `mdu_res_c_ff` | 1 | 中间结果进位/借位寄存器 |
| `mdu_res_c_next` | 1 | 进位/借位寄存器下一值 |
| `mdu_res_hi_ff` | `SCR1_XLEN-1:0` | 中间结果高位寄存器（除法存余数，乘法存部分积高位） |
| `mdu_res_hi_next` | `SCR1_XLEN-1:0` | 高位寄存器下一值 |
| `mdu_res_lo_ff` | `SCR1_XLEN-1:0` | 中间结果低位寄存器（除法存商，乘法存部分积低位） |
| `mdu_res_lo_next` | `SCR1_XLEN-1:0` | 低位寄存器下一值 |

---

## 5. 条件编译宏一览

| 宏名 | 含义 | 影响模块范围 |
|------|------|------|
| `SCR1_RVM_EXT` | 使能 RVM 乘除法扩展 | 整个 MUL/DIV 逻辑、FSM、MDU 加法器、中间寄存器 |
| `SCR1_FAST_MUL` | 使能单周期快速乘法器 | 乘法器数据通路（参数、操作数位宽、结果寄存器） |
| `SCR1_TRGT_SIMULATION` | 使能仿真专用信号和 SVA 断言 | `mdu_fsm_iter` 信号、7 条断言 |
