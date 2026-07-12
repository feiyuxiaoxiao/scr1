# SCR1 IALU 整数算术逻辑单元 技术设计文档

> 对应源码: `src/core/pipeline/scr1_pipe_ialu.sv`
> 模块名: `scr1_pipe_ialu`
> 制定者: Syntacore LLC, 2016-2021

---

## 1. 总体架构

### 1.1 功能定位

IALU 是 SCR1 处理器流水线中 EXU（执行单元）的核心运算组件，承担所有整数算术、逻辑、比较、移位和乘除运算。处理器所有需要计算的指令最终都会路由到 IALU 的某个功能单元完成运算。

核心职责：

1. 执行加减法（ADD/ADDI/SUB）和逻辑操作（AND/OR/XOR 及各自的立即数形式）
2. 执行分支比较（BEQ/BNE/BLT(U)/BGE(U)），通过减法生成标志位并导出分支方向
3. 执行算术比较（SLT(U)/SLTI(U)），同样通过减法和标志位判定
4. 通过独立的地址加法器计算 PC 相对跳转目标、寄存器间接跳转目标、访存地址
5. 执行移位操作（SLL/SRL/SRA 及各自的立即数形式）
6. 支持 M 扩展乘除法（MUL/MULH/MULHSU/MULHU/DIV(U)/REM(U)）

### 1.2 模块划分

```
                         scr1_pipe_ialu
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌──────────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │   Main Adder     │   │  Address     │   │   Shift Logic  │  │
│  │  (33-bit +/-)    │   │  Adder       │   │  (SLL/SRL/SRA) │  │
│  │  + Flags(zsoc)   │   │  (32-bit +)  │   │                │  │
│  └────────┬─────────┘   └──────┬───────┘   └───────┬────────┘  │
│           │                    │                    │           │
│           ▼                    ▼                    ▼           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Output Result MUX                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│           ▲                                                      │
│           │                                                      │
│  ┌────────┴──────────────────────────────────────────────────┐  │
│  │                   MUL/DIV Unit (RVM)                       │  │
│  │  ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌───────────┐  │  │
│  │  │ MDU FSM  │ │ Multiplier │ │ Divider  │ │ MDU Adder │  │  │
│  │  │ Control  │ │ (RADIX-2   │ │(Non-resto│ │(+) / (-)  │  │  │
│  │  │ + Count  │ │  or Fast)  │ │ ring)    │ │           │  │  │
│  │  └──────────┘ └────────────┘ └──────────┘ └───────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 配置宏一览

| 宏名 | 作用 |
|------|------|
| `SCR1_RVM_EXT` | 使能 RISC-V M 扩展（乘除法），无此宏时仅包含基础整数指令 |
| `SCR1_FAST_MUL` | 使能单周期快速乘法器（`*` 运算符直接乘）；否则使用 32 周期 Radix-2 乘法器 |
| `SCR1_TRGT_SIMULATION` | 使能仿真专用信号和 SVA 行为断言 |

---

## 2. 主加法器 (Main Adder)

### 2.1 功能概述

主加法器是 IALU 计算核心，承载几乎所有非移位、非乘除指令的算术运算。所有比较指令（分支、SLT）本质上也是减法——通过扩位 33 位减法得出结果，然后从标志位（z/s/o/c）推导比较结论。

### 2.2 加法/减法实现

```systemverilog
always_comb begin
    main_sum_res = (exu2ialu_cmd_i != SCR1_IALU_CMD_ADD)
                 ? ({1'b0, exu2ialu_main_op1_i} - {1'b0, exu2ialu_main_op2_i})
                 : ({1'b0, exu2ialu_main_op1_i} + {1'b0, exu2ialu_main_op2_i});
    // ...
end
```

**关键设计决策**：所有非 ADD 命令（包括 SUB、SLT、分支比较等）统一走减法通路。操作数扩展为 33 位（高位补 0）以正确捕获进位/借位。

### 2.3 溢出检测

```systemverilog
main_sum_pos_ovflw = ~exu2ialu_main_op1_i[31] &  exu2ialu_main_op2_i[31] &  main_sum_res[31];
main_sum_neg_ovflw =  exu2ialu_main_op1_i[31] & ~exu2ialu_main_op2_i[31] & ~main_sum_res[31];
```

正溢出：两个**非负数**相加后结果符号位变为 1（即超过 `+2^31-1`）。
负溢出：两个**负数**相加后结果符号位变为 0（即低于 `-2^31`）。

溢出仅用于有符号比较（SLT/BLT/BGE），无符号运算不使用该标志。

### 2.4 标志位 (Flags) 与比较派生

四个标志位从 33 位减法结果中提取：

| 标志 | 表达式 | 含义 |
|:---:|------|------|
| `c` (进位) | `main_sum_res[32]` | 无符号减法借位。`c=0` 表示发生借位（`op1 < op2` 无符号），`c=1` 表示无需借位（`op1 >= op2`） |
| `z` (零) | `~\|main_sum_res[31:0]` | 结果零：`op1 == op2` |
| `s` (符号) | `main_sum_res[31]` | 结果符号位，直接表示符号 |
| `o` (溢出) | `pos_ovflw \| neg_ovflw` | 有符号溢出 |

### 2.5 各命令的判定逻辑

| 命令 | 对应指令 | 结果公式 | 判定逻辑 |
|------|------|------|------|
| `CMD_ADD` | ADD, ADDI | `op1 + op2` | 直接使用加法结果 |
| `CMD_SUB` | SUB | `op1 - op2` | 直接使用减法结果 |
| `CMD_SUB_LT` | SLT, SLTI | `op1 < op2` (signed) | `s ^ o`：符号异或溢出 = 真小于 |
| `CMD_SUB_LTU` | SLTU, SLTIU | `op1 < op2` (unsigned) | `c == 0`：发生借位 = 无符号小于 |
| `CMD_SUB_EQ` | BEQ | `op1 == op2` | `z == 1`：结果为零 |
| `CMD_SUB_NE` | BNE | `op1 != op2` | `~z`：结果非零 |
| `CMD_SUB_GE` | BGE | `op1 >= op2` (signed) | `~(s ^ o)`：不小于即大于等于 |
| `CMD_SUB_GEU` | BGEU | `op1 >= op2` (unsigned) | `c == 1`：无需借位 |

**有符号小于判定的推导** (`s ^ o`)：

- 无溢出时 (`o=0`)：结果符号 s 直接反映大小关系（`s=1` 表示结果为负，即 `op1 < op2`）
- 有溢出时 (`o=1`)：结果符号与实际大小关系相反（正溢出时结果为负，但实际 `op1 > op2`），因此取反

`o=1` 时 `s ^ 1 = ~s`，正好修正了溢出导致的符号反转。

---

## 3. 地址加法器 (Address Adder)

### 3.1 功能概述

地址加法器是一个完全独立于主加法器的 32 位纯加法通路，用于计算各种指令跳转和访存目标地址。

```systemverilog
assign ialu2exu_addr_res_o = exu2ialu_addr_op1_i + exu2ialu_addr_op2_i;
```

### 3.2 适用指令

| 指令类型 | 操作数 1 | 操作数 2 | 计算目标 |
|------|------|------|------|
| AUIPC | 当前 PC | 20 位立即数 << 12 | PC 相对地址（AUIPC 目标） |
| BEQ/BNE/BLT(U)/BGE(U) | 当前 PC | 分支偏移量 | 分支目标地址 |
| JAL | 当前 PC | J 型立即数 | 跳转目标地址 |
| JALR | 源寄存器值 | I 型立即数 | 寄存器间接跳转目标 |
| LB(U)/LH(U)/LW | 基址寄存器值 | I 型立即数 | 数据存储器加载地址 |
| SB/SH/SW | 基址寄存器值 | S 型立即数 | 数据存储器存储地址 |

### 3.3 与主加法器的独立性

地址加法器是完全独立的组合逻辑通路，不经过主加法器和输出 MUX。这意味着：

1. 在同一周期内，主加法器可以做运算/比较，地址加法器可以**同时**计算跳转/访存地址
2. 没有任何信号需要在两者之间传递或共享

---

## 4. 移位器 (Shift Logic)

### 4.1 功能概述

移位器支持三种 RISC-V 移位操作：逻辑左移、逻辑右移、算术右移。

### 4.2 命令编码

```systemverilog
assign ialu_cmd_shft = (exu2ialu_cmd_i == SCR1_IALU_CMD_SLL)
                     | (exu2ialu_cmd_i == SCR1_IALU_CMD_SRL)
                     | (exu2ialu_cmd_i == SCR1_IALU_CMD_SRA);
assign shft_cmd      = ialu_cmd_shft
                     ? {(exu2ialu_cmd_i != SCR1_IALU_CMD_SLL),
                        (exu2ialu_cmd_i == SCR1_IALU_CMD_SRA)}
                     : 2'b00;
```

| `shft_cmd[1:0]` | 移位类型 | 对应 SystemVerilog 操作 | 含义 |
|:---:|------|------|------|
| `00` | 逻辑左移 | `shft_op1 << shft_op2` | SLL / SLLI |
| `10` | 逻辑右移 | `shft_op1 >> shft_op2` | SRL / SRLI |
| `11` | 算术右移 | `shft_op1 >>> shft_op2` | SRA / SRAI |

编码推导：

- `shft_cmd[1]` = `cmd != SLL`，因此 SLL → 0, SRL → 1, SRA → 1
- `shft_cmd[0]` = `cmd == SRA`，因此 SLL → 0, SRL → 0, SRA → 1

### 4.3 数据通路

```systemverilog
shft_op1 = exu2ialu_main_op1_i;       // 被移位数（signed）
shft_op2 = exu2ialu_main_op2_i[4:0];  // 移位量取低 5 位（0-31）
case (shft_cmd)
    2'b10   : shft_res = shft_op1  >> shft_op2;   // 逻辑右移
    2'b11   : shft_res = shft_op1 >>> shft_op2;   // 算术右移
    default : shft_res = shft_op1  << shft_op2;   // 逻辑左移 (包含 2'b00, 2'b01)
endcase
```

注意 `shft_op1` 声明为 `signed`，确保 `>>>` 算术右移时做符号扩展。

---

## 5. 乘法器逻辑 (Multiplier Logic)

### 5.1 命令编码与操作数符号

乘法子命令 `mul_cmd[1:0]` 编码：

| `mul_cmd[1:0]` | 对应指令 | 操作数 1 符号 | 操作数 2 符号 | 结果取值 |
|:---:|------|:---:|:---:|------|
| `00` | MUL | 无符号 | 无符号 | 低 32 位 |
| `01` | MULH | 有符号 | 有符号 | 高 32 位 |
| `10` | MULHSU | 有符号 | 无符号 | 高 32 位 |
| `11` | MULHU | 无符号 | 无符号 | 高 32 位 |

编码公式：
```systemverilog
assign mul_cmd = { (cmd == MULHU | cmd == MULHSU),
                   (cmd == MULHU | cmd == MULH) };
```

操作数符号判定：
```systemverilog
assign mul_op1_is_sgn = ~&mul_cmd;           // MULHU 以外均为有符号
assign mul_op2_is_sgn = ~mul_cmd[1];          // MULHU 和 MULHSU 以外均为有符号
assign mul_cmd_hi     = |mul_cmd;             // MUL 以外均取高 32 位
```

### 5.2 快速乘法器 (`SCR1_FAST_MUL`)

使用 SystemVerilog 内置 `*` 运算符单周期完成 32×32=64 位乘法。操作数在符号处理后乘即可：

```systemverilog
assign mul_op1 = $signed({mul_op1_sgn, exu2ialu_main_op1_i});  // 33 位: {符号扩展, 原值}
assign mul_op2 = $signed({mul_op2_sgn, exu2ialu_main_op2_i});
assign mul_res = mul_op1 * mul_op2;                              // 64 位结果
```

**符号处理**：RISC-V 语义中 MUL 操作数视为无符号，MULHSU 的 op2 视为无符号。设计中用 `mul_op1_sgn`/`mul_op2_sgn` 信号将负数的符号位扩展为 1 bit 补在操作数最高位之上（相当于符号扩展后取绝对值前对符号的记录），然后用一个 33×33=64 的带符号乘法一次得出正确结果。

输出时根据 `mul_cmd_hi` 选择高位或低位 32 位：
```systemverilog
ialu2exu_main_res_o = mul_cmd_hi
                    ? mul_res[63:32]
                    : mul_res[31:0];
```

快速乘法器下乘除法 FSM 仅用于除法（`mdu_cmd_is_iter = mdu_cmd_div`）。

### 5.3 Radix-2 逐位迭代乘法器 (`~SCR1_FAST_MUL`)

非快速乘法器使用经典 Radix-2 算法，将 32 位乘法拆分为 32 个周期，每周期处理 1 位乘数。

#### 算法流程

```
初始化: {mul_res_hi, mul_res_lo} = {0, oper2}
FOR i = 0 TO 31:
    部分积 = 被乘数 * 乘数的最低 1 位  (即乘数 LSB=1 时 = 被乘数，否则 = 0)
    {mul_res_hi, mul_res_lo} = {mul_res_hi + 部分积, mul_res_lo >> 1}
```

```systemverilog
// 操作数 2：每次取 1 位作为部分积系数
assign mul_op2 = ~mdu_cmd_mul ? '0
               : mdu_fsm_idle                                       // 初始：取 op2 的 LSB 位
                   ? $signed({1'b0, exu2ialu_main_op2_i[0]})
               : $signed({(mdu_iter_cnt[0] & mul_op2_is_sgn         // 迭代：取结果低位寄存器的 LSB 位
                             & mdu_res_lo_ff[0]),
                           mdu_res_lo_ff[0]});

// 部分积 = 被乘数(33-bit) × 当前乘数位(又 1'b1 or 1'b0)
assign mul_part_prod = mul_op1 * mul_op2;

// 累加
assign {mul_res_hi, mul_res_lo} = ~mdu_cmd_mul ? '0
                                : mdu_fsm_idle
                                    ? {mdu_sum_res, exu2ialu_main_op2_i[31:1]}
                                : {mdu_sum_res, mdu_res_lo_ff[31:1]};
```

每次迭代中：
1. `mdu_res_lo_ff` 右移 1 位（丢弃已处理的 LSB，暴露出下一个 LSB）
2. MDU 加法器将 `mul_res_hi` 与 `mul_part_prod` 相加
3. 累加结果写回 `mul_res_hi`，右移后的低半部分写回 `mul_res_lo`

**操作数符号处理**：有符号乘法在两个操作数的最高位之上前补一位符号标志，使用带符号的 33 位乘 1 位 = 33 位乘法得出部分积。符号标志由 `mul_op1_sgn` 和 `mul_op2_sgn` 提供。

**迭代计数器**：从 `SCR1_MUL_CNT_INIT = 32'h4000_0000` 开始右移，当 LSB = 1 时（第 32 拍）`mdu_iter_rdy = 1`，乘法完成。

### 5.4 两种乘法器的区别总结

| 特性 | `SCR1_FAST_MUL` | `~SCR1_FAST_MUL` (Radix-2) |
|------|:---:|:---:|
| 完成延迟 | 1 周期 | 32 周期 |
| 乘法实现 | `*` 运算符直接 33×33 乘 | 逐位累加，每拍 1 位 |
| 操作数 2 位宽 | `SCR1_XLEN:0` = 33 | `SCR1_MUL_WIDTH:0` = 2 |
| FSM 参与 | 否（仅除法用 FSM） | 是（乘法也用 FSM） |
| 中间寄存器 | 无 | `mul_res_hi`, `mul_res_lo` |
| MDU 加法器 | 不参与乘法 | 参与乘法部分积累加 |
| 硬件开销 | 较大（组合乘法器） | 较小（迭代复用） |
| `mdu_cmd_is_iter` | `= mdu_cmd_div` | `= mdu_cmd_mul \| mdu_cmd_div` |

---

## 6. 除法器逻辑 (Divider Logic)

### 6.1 算法：非恢复式除法 (Non-Restoring Division)

SCR1 采用 32 拍的非恢复式（non-restoring）除法算法，每拍产生 1 位商。

### 6.2 命令编码

```systemverilog
assign div_cmd = { (cmd == REM | cmd == REMU),
                   (cmd == REMU | cmd == DIVU) };
```

| `div_cmd[1:0]` | 对应指令 | 操作数符号 |
|:---:|------|:---:|
| `00` | DIV | 有符号 |
| `01` | DIVU | 无符号 |
| `10` | REM | 有符号 |
| `11` | REMU | 无符号 |

其中 `div_cmd[0]` 控制符号（0=有符号, 1=无符号），`div_cmd[1]` 区分除法和取余。

### 6.3 算法流程

```
初始化:
    余数 = 被除数符号扩展
    除数 = 除数符号扩展  (符号由 div_ops_are_sgn 控制)
    商 = 0
    被除数低位 = 被除数 << 1

FOR i = 0 TO 31:
    将余数和被除数低位左移 1 位
    根据前一次商位和操作数符号确定做加还是减:
        本次做减法 ↔ (~inv ^ sgn)
        其中 inv = div_ops_are_sgn & main_ops_diff_sgn  (有符号且异号时取反)
              sgn = 上一次商位取反

    比较: 余数 >= 除数 ?
    商位 = ~(被除数符号 ^ 进位标志)
            | (被除数为负 且 余数和被除数低位全为零)   ← 处理边界

    商 = (商 << 1) | 商位

IF 有符号 且 被除数与除数异号:
    商 = -商   (CORR 阶段修正)

IF 有符号取余 且 余数符号与预期不符:
    余数 = 除数 - 余数  (CORR 阶段修正)
```

### 6.4 MDU 加法器在除法中的复用

```systemverilog
case (mdu_cmd)
    SCR1_IALU_MDU_DIV : begin
        // sgn: 下一拍做减法(1)还是加法(0)，由上一拍的商位控制
        sgn = mdu_fsm_corr ? div_op1_is_neg ^ mdu_res_c_ff
            : mdu_fsm_idle ? 1'b0
            : ~mdu_res_lo_ff[0];      // 上一拍的商位取反

        inv = div_ops_are_sgn & main_ops_diff_sgn;
        mdu_sum_sub = ~inv ^ sgn;      // 最终加/减控制

        mdu_sum_op1 = mdu_fsm_corr                        // CORR 阶段
                    ? $signed({1'b0, mdu_res_hi_ff})       // 余数
                    : mdu_fsm_idle                         // 初始化
                        ? $signed({div_op1_is_neg, exu2ialu_main_op1_i[31]})
                    : $signed({mdu_res_hi_ff, div_dvdnd_lo_ff[31]});  // 迭代: 余数左移

        mdu_sum_op2 = $signed({div_op2_is_neg, exu2ialu_main_op2_i});  // 除数
    end
    // ...
endcase
```

### 6.5 商位计算与边界条件

```systemverilog
div_quo_bit = ~(div_op1_is_neg ^ div_res_rem_c)
            | (div_op1_is_neg & ({mdu_sum_res, div_dvdnd_lo_next} == '0));
```

第一项 `~(div_op1_is_neg ^ div_res_rem_c)`：无符号比较，进位标志 `div_res_rem_c` 指示余数是否小于除数。
第二项：边界条件——当被除数为负时，需要额外检查余数和被除数低位是否**全为零**，以处理"余数精确等于除数"的情况。这是文献 [1](https://en.wikipedia.org/wiki/Division_algorithm) 中提到的 corner case。

### 6.6 CORR 阶段修正

CORR 阶段处理有符号除法的结果修正：

```systemverilog
assign div_corr_req = div_cmd_div & main_ops_diff_sgn;
assign rem_corr_req = div_cmd_rem & |div_res_rem & (div_op1_is_neg ^ div_res_rem_c);
assign mdu_corr_req = mdu_cmd_div & (div_corr_req | rem_corr_req);
```

| 修正请求 | 条件 | 修正操作 |
|------|------|------|
| `div_corr_req` | DIV 且被除数与除数异号 | `商 = -商`（对商取相反数） |
| `rem_corr_req` | REM(U) 且余数非零且符号与预期不符 | `余数 = 除数 - 余数`（恢复余数） |

---

## 7. MDU FSM（乘除状态机）

### 7.1 状态定义

```
                  ┌─────────────────────────────┐
                  │                             │
                  ▼                             │
           ┌───────────┐ iter_req   ┌───────────────┐  iter_rdy     ┌───────────────┐
  reset ──▶│   IDLE    │──────────▶│     ITER      │──────────────▶│     CORR      │
           │  (2'b00)  │           │   (2'b01)     │  & corr_req   │   (2'b10)     │
           └───────────┘           └───────────────┘               └───────────────┘
                  ▲                        │                              │
                  │      iter_rdy          │                              │
                  │    & ~corr_req         │  下一拍                      │
                  └────────────────────────┘◀─────────────────────────────┘
```

### 7.2 状态转移逻辑

```systemverilog
always_comb begin
    mdu_fsm_next = SCR1_IALU_MDU_FSM_IDLE;
    if (exu2ialu_rvm_cmd_vd_i) begin
        case (mdu_fsm_ff)
            SCR1_IALU_MDU_FSM_IDLE : begin
                mdu_fsm_next = mdu_iter_req  ? SCR1_IALU_MDU_FSM_ITER
                                             : SCR1_IALU_MDU_FSM_IDLE;
            end
            SCR1_IALU_MDU_FSM_ITER : begin
                mdu_fsm_next = ~mdu_iter_rdy ? SCR1_IALU_MDU_FSM_ITER
                             : mdu_corr_req  ? SCR1_IALU_MDU_FSM_CORR
                                             : SCR1_IALU_MDU_FSM_IDLE;
            end
            SCR1_IALU_MDU_FSM_CORR : begin
                mdu_fsm_next = SCR1_IALU_MDU_FSM_IDLE;   // 只停留 1 拍
            end
        endcase
    end
end
```

| 转移 | 条件 |
|------|------|
| IDLE → ITER | `exu2ialu_rvm_cmd_vd_i & mdu_iter_req` |
| IDLE → IDLE | 无迭代请求（操作数为零 或 除法除数为零 = 直接旁路） |
| ITER → ITER | `~mdu_iter_rdy`：计数器未递减到位 |
| ITER → IDLE | `mdu_iter_rdy & ~mdu_corr_req`：计算完成且无需修正 |
| ITER → CORR | `mdu_iter_rdy & mdu_corr_req`：计算完成但需符号修正 |
| CORR → IDLE | 无条件（1 拍自动返回） |

**迭代请求条件**：
```systemverilog
assign mdu_iter_req = mdu_cmd_is_iter ? (main_ops_non_zero & mdu_fsm_idle) : 1'b0;
```

- 快速乘法器：仅除法请求迭代 (`mdu_cmd_is_iter = mdu_cmd_div`)
- 非快速乘法器：乘法和除法均请求迭代 (`mdu_cmd_is_iter = mdu_cmd_mul | mdu_cmd_div`)
- 跳过条件：`main_ops_non_zero = 0`（任一操作数为 0），直接输出默认值

### 7.3 迭代计数器

```systemverilog
assign mdu_iter_cnt_next = ~mdu_fsm_idle ? mdu_iter_cnt >> 1
                         : mdu_cmd_div   ? SCR1_DIV_CNT_INIT  // 32'h4000_0000
                         : mdu_cmd_mul   ? SCR1_MUL_CNT_INIT  // 32'h4000_0000（非 FAST_MUL）
                                         : mdu_iter_cnt;

assign mdu_iter_rdy = mdu_iter_cnt[0];   // LSB = 1 时迭代完成
```

计数器从 `32'h4000_0000`（仅第 30 位为 1）开始右移，每拍移 1 位。32 拍后 LSB 变为 1，触发 `mdu_iter_rdy`。

---

## 8. 输出结果多路复用器 (Output MUX)

IALU 的最终输出由 `always_comb` 块根据 `exu2ialu_cmd_i` 做 case 选择：

### 8.1 逻辑运算（AND/OR/XOR）

直接输出按位逻辑结果：
```systemverilog
SCR1_IALU_CMD_AND : ialu2exu_main_res_o = op1 & op2;
SCR1_IALU_CMD_OR  : ialu2exu_main_res_o = op1 | op2;
SCR1_IALU_CMD_XOR : ialu2exu_main_res_o = op1 ^ op2;
```

### 8.2 加减法（ADD/SUB）

直接输出主加法器的低 32 位：
```systemverilog
SCR1_IALU_CMD_ADD, SCR1_IALU_CMD_SUB :
    ialu2exu_main_res_o = main_sum_res[31:0];
```

### 8.3 比较运算（SLT/Branch）

从标志位派生 1-bit 结果并零扩展到 32 位：
```systemverilog
SCR1_IALU_CMD_SUB_LT  : ialu2exu_main_res_o = 32'(s ^ o);    ialu2exu_cmp_res_o = s ^ o;
SCR1_IALU_CMD_SUB_LTU : ialu2exu_main_res_o = 32'(c);        // c=0 即借位 = 小于
SCR1_IALU_CMD_SUB_EQ  : ialu2exu_main_res_o = 32'(z);
SCR1_IALU_CMD_SUB_NE  : ialu2exu_main_res_o = 32'(~z);
SCR1_IALU_CMD_SUB_GE  : ialu2exu_main_res_o = 32'(~(s ^ o));
SCR1_IALU_CMD_SUB_GEU : ialu2exu_main_res_o = 32'(c);        // c=1 即无借位 = 大于等于
```

### 8.4 移位

直接输出移位结果：
```systemverilog
SCR1_IALU_CMD_SLL, SCR1_IALU_CMD_SRL, SCR1_IALU_CMD_SRA :
    ialu2exu_main_res_o = shft_res;
```

### 8.5 乘法

**快速乘法器**：单周期输出，根据 `mul_cmd_hi` 选择高或低 32 位。

**非快速乘法器**：按 FSM 状态输出

| 状态 | 输出 | `rvm_res_rdy_o` |
|------|------|:---:|
| IDLE | 0 | `~mdu_iter_req`（操作数为 0 时立即就绪） |
| ITER | `mul_res_hi` 或 `mul_res_lo` | `mdu_iter_rdy`（最后一拍就绪） |

### 8.6 除法/取余

| 状态 | 输出 | `rvm_res_rdy_o` |
|------|------|:---:|
| IDLE | 除数为 0: `-1`；REM 或无迭代: `op1` | `~mdu_iter_req` |
| ITER | `div_res_rem`（取余）或 `div_res_quo`（除） | `mdu_iter_rdy & ~mdu_corr_req` |
| CORR | `mdu_sum_res[31:0]`（取余修正）或 `-mdu_res_lo_ff`（商修正） | 1 |

**除数为 0 的处理**（IDLE 状态中）：
```systemverilog
ialu2exu_main_res_o = (|exu2ialu_main_op2_i | div_cmd_rem)
                     ? exu2ialu_main_op1_i      // 除数 ≠ 0 或 REM 指令：输出被除数
                     : '1;                        // DIV 且除数为 0: 输出全 1
```
RISC-V 规范规定 DIV(U) 除 0 时商 = `-1`（全 1），REM(U) 除 0 时余数 = 被除数。

---

## 9. 断言 (Assertions)

`SCR1_TRGT_SIMULATION` 使能下，共有 7 条 SVA 断言：

| 断言名 | 类型 | 验证内容 |
|------|:---:|------|
| `SCR1_SVA_IALU_XCHECK` | X 态检查 | `exu2ialu_rvm_cmd_vd_i` 和 `mdu_fsm_ff` 不得为 X |
| `SCR1_SVA_IALU_XCHECK_QUEUE` | X 态检查 | 命令有效时操作数和命令不得为 X |
| `SCR1_SVA_IALU_ILL_STATE` | 行为 | 不得同时出现 "非有效命令、ITER、CORR" 中的多项 |
| `SCR1_SVA_IALU_JUMP_FROM_IDLE` | 行为 | IDLE 中无迭代请求时下一拍必须保持 IDLE |
| `SCR1_SVA_IALU_IDLE_TO_ITER` | 行为 | IDLE + 有效命令 + 迭代请求 → 下一拍 ITER |
| `SCR1_SVA_IALU_JUMP_FROM_ITER` | 行为 | ITER 中未就绪时下一拍必须保持 ITER |
| `SCR1_SVA_IALU_ITER_TO_IDLE` | 行为 | ITER + 就绪 + 无需修正 → IDLE |
| `SCR1_SVA_IALU_ITER_TO_CORR` | 行为 | ITER + 就绪 + 需修正 → CORR |
| `SCR1_SVA_IALU_CORR_TO_IDLE` | 行为 | CORR 下一拍必须回到 IDLE |

这些断言覆盖了 FSM 的全部合法状态转移路径，同时也验证了非法转移不会发生。
