# scr1_pipe_ialu.sv 讲解

## 一句话职责

整数 ALU：双加法器通路（主运算 + 地址计算）+ 移位器 + MDU 乘除。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_ialu.sv`
- **行数**: 720 行
- **配置依赖**: `SCR1_RVM_EXT`、`SCR1_FAST_MUL`

## 核心架构

### 1. 双加法器通路

| 通路 | 操作数来源 | 结果用途 |
|---|---|---|
| `ialu_main` | `rs1/rs2/imm` | ADD/SUB/XOR/AND/OR 等 |
| `ialu_addr` | `PC/rs1/imm` | JAL/LOAD/STORE 地址计算 |

**为什么需要两条？** `beq` 指令要同时"比较两寄存器相等吗"并"计算跳转目标 PC+imm"，单加法器做不到并行。

### 2. 主加法器操作类型（`:100-200`）

```systemverilog
case (ialu_cmd)
    SCR1_IALU_ADD:  ialu_main_res = op1 + op2;
    SCR1_IALU_SUB:  ialu_main_res = op1 - op2;
    SCR1_IALU_XOR:  ialu_main_res = op1 ^ op2;
    SCR1_IALU_OR:   ialu_main_res = op1 | op2;
    SCR1_IALU_AND:  ialu_main_res = op1 & op2;
    SCR1_IALU_SLT:  ialu_main_res = (op1 < op2) ? 1 : 0;       // 有符号比较
    SCR1_IALU_SLTU: ialu_main_res = (op1 < op2) ? 1 : 0;       // 无符号比较
    SCR1_IALU_SLL:  ialu_main_res = op1 << shift_amt;          // 逻辑左移
    SCR1_IALU_SRL:  ialu_main_res = op1 >> shift_amt;          // 逻辑右移
    SCR1_IALU_SRA:  ialu_main_res = $signed(op1) >>> shift_amt;// 算术右移
endcase
```

### 3. 移位器（`:250-300`）

```systemverilog
shift_amt = op2[4:0];  // RV32 只取低 5 位（0~31）
```

**为什么只取 5 位？** 32 位寄存器移位 32 次等于没移，高位无意义。

### 4. MDU 乘除单元（`:350-550`，`SCR1_RVM_EXT` 时启用）

#### 乘法

| 指令 | 结果 | 位数 |
|---|---|---|
| `MUL` | `rs1 * rs2` 低 32 位 | 32 位 |
| `MULH` | `rs1 * rs2` 高 32 位（有符号） | 32 位 |
| `MULHU` | `rs1 * rs2` 高 32 位（无符号） | 32 位 |
| `MULHSU` | `rs1(有符号) * rs2(无符号)` 高 32 位 | 32 位 |

**实现方式**：
- `SCR1_FAST_MUL=1`：组合乘法器（单周期，面积大）
- `SCR1_FAST_MUL=0`：移位累加（多周期，面积小）

#### 除法

| 指令 | 含义 | 特殊情况 |
|---|---|---|
| `DIV` | `rs1 / rs2`（有符号） | 除零 → 返回 -1 |
| `DIVU` | `rs1 / rs2`（无符号） | 除零 → 返回全 1 |
| `REM` | `rs1 % rs2`（有符号） | 除零 → 返回 rs1 |
| `REMU` | `rs1 % rs2`（无符号） | 除零 → 返回 rs1 |

**除零处理**（`:480-490`）：
```systemverilog
if (divisor == 0) begin
    quotient  = '1;  // -1 或全 1
    remainder = dividend;
end
```

**溢出处理**：`INT_MIN / -1` 结果溢出（32 位有符号范围 -2^31~2^31-1，-(-2^31)=2^31 超出），返回 `INT_MIN`。

### 5. 结果选择（`:600-650`）

```systemverilog
case (ialu_result_sel)
    SCR1_IALU_MAIN: result = ialu_main_res;  // ADD/SUB/XOR...
    SCR1_IALU_ADDR: result = ialu_addr_res;  // JAL/LOAD/STORE 地址
    SCR1_IALU_MUL:  result = mdu_mul_res;    // MUL 系列
    SCR1_IALU_DIV:  result = mdu_div_res;    // DIV/REM 系列
endcase
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `exu2ialu_cmd_i` | in | ALU 操作类型 |
| `exu2ialu_op1_i` | in | 操作数 1 |
| `exu2ialu_op2_i` | in | 操作数 2 |
| `exu2ialu_pc_i` | in | 当前 PC（地址计算用） |
| `exu2ialu_imm_i` | in | 立即数 |
| `ialu2exu_result_o` | out | 运算结果 |
| `ialu2exu_cmp_o` | out | 比较结果（BEQ/BNE 用） |
| `ialu_rdy_o` | out | 多周期指令完成标志 |

## 多周期指令时序

```
周期 1: 指令进入 EXU 队列 → MDU 开始计算
周期 2-N: MDU 内部移位累加 → ialu_rdy_o=0
周期 N+1: MDU 完成 → ialu_rdy_o=1 → exu_rdy=1 → 写回
```

`exu2idu_rdy_o = ialu_rdy_o`（多周期期间为 0，流水线冻结）。

## 配置宏影响

| 宏 | 行为变化 |
|---|---|
| `SCR1_RVM_EXT` | MDU 乘除单元使能 |
| `SCR1_FAST_MUL` | 乘法器组合实现（单周期）vs 移位累加（多周期） |
| `SCR1_NO_EXE_STAGE` | 无影响（IALU 本身纯组合） |

## 与其他模块关系

```
EXU → IALU: 操作命令 + 操作数
IALU → EXU: 运算结果 + 比较结果 + rdy
```

## 配套文档

- 设计规格：`docs/scr1-pipe-ialu/design-spec.md`
- 验证文档：`docs/scr1-pipe-ialu/verification.md`
