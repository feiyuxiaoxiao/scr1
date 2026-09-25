# SCR1 IALU 整数运算单元设计规格（Design Specification）

- 模块：`scr1_pipe_ialu`（`src/core/pipeline/scr1_pipe_ialu.sv`，720 行）
- 层次：`scr1_pipe_exu` 的子模块，例化于 `scr1_pipe_exu.sv:445-465`
- 特性：主加法器/地址加法器/移位 + 可选乘除（`SCR1_RVM_EXT`）
- 参考：`docs/scr1_um.pdf`、RISC-V `M` 扩展规范

---

## 1. 概述与职责

IALU 是 EXU 的组合运算核心，提供四条执行路径：

1. **主加法器**：加/减、算术/无符号比较、相等比较，产出 `flags{z,s,o,c}`；
2. **地址加法器**：AUIPC/分支/跳转/load/store 的地址或目标计算（无条件加）；
3. **移位逻辑**：SLL/SRL/SRA；
4. **MUL/DIV 单元（MDU）**：M 扩展乘除，迭代实现。

**职责边界**：IALU 只做运算，不做异常、不决定写回来源（由 EXU 的写回 mux 选择）。
操作数 `ialu_main_op1/op2` 与地址操作数 `ialu_addr_op1/op2` 均由 EXU 预选
（EXU `:403-440`）。

**三条结果输出**：

| 输出 | 含义 |
|---|---|
| `ialu2exu_main_res_o` | 主运算结果（算术/逻辑/移位/比较布尔/乘除结果） |
| `ialu2exu_cmp_res_o` | 比较结果（分支/`slt` 条件用） |
| `ialu2exu_addr_res_o` | 地址加法器结果 |

---

## 2. 端口与接口

见源码 `:30-50`。端口按 `SCR1_RVM_EXT` 分为两组。

### 2.1 公共（仅 RVM，`:31-37`）

| 信号 | 方向 | 位宽 | 行号 | 说明 |
|---|---|---|---|---|
| `clk` | in | 1 | 33 | 时钟 |
| `rst_n` | in | 1 | 34 | 复位 |
| `exu2ialu_rvm_cmd_vd_i` | in | 1 | 35 | MUL/DIV 命令有效 |
| `ialu2exu_rvm_res_rdy_o` | out | 1 | 36 | MUL/DIV 结果就绪 |

> 无 RVM 时这些端口不出现，模块纯组合。

### 2.2 主/地址（`:39-49`）

| 信号 | 方向 | 位宽 | 行号 |
|---|---|---|---|
| `exu2ialu_main_op1_i` | in | XLEN | 40 |
| `exu2ialu_main_op2_i` | in | XLEN | 41 |
| `exu2ialu_cmd_i` | in | `type_scr1_ialu_cmd_sel_e` | 42 |
| `ialu2exu_main_res_o` | out | XLEN | 43 |
| `ialu2exu_cmp_res_o` | out | 1 | 44 |
| `exu2ialu_addr_op1_i` | in | XLEN | 47 |
| `exu2ialu_addr_op2_i` | in | XLEN | 48 |
| `ialu2exu_addr_res_o` | out | XLEN | 49 |

---

## 3. 局部参数/类型/信号

### 3.1 局部参数（`:56-69`）

**`SCR1_FAST_MUL`**（单周期乘法）：

```systemverilog
localparam SCR1_MUL_WIDTH     = `SCR1_XLEN;          // :58
localparam SCR1_MUL_RES_WIDTH = 2 * `SCR1_XLEN;      // :59
localparam SCR1_MDU_SUM_WIDTH = `SCR1_XLEN + 1;      // :60
```

**非 FAST_MUL**（Radix-2，32 周期）：

```systemverilog
localparam SCR1_MUL_STG_NUM   = 32;                  // :62
localparam SCR1_MUL_WIDTH     = 32 / SCR1_MUL_STG_NUM; // =1  // :63
localparam SCR1_MUL_CNT_INIT  = 32'b1 << (`SCR1_XLEN/SCR1_MUL_WIDTH - 2); // :64
localparam SCR1_MDU_SUM_WIDTH = `SCR1_XLEN + SCR1_MUL_WIDTH; // :65
```

**除法**（`:67-68`）：

```systemverilog
localparam SCR1_DIV_WIDTH     = 1;
localparam SCR1_DIV_CNT_INIT  = 32'b1 << (`SCR1_XLEN/SCR1_DIV_WIDTH - 2);
```

### 3.2 类型（`:75-93`）

```systemverilog
typedef struct packed { logic z; logic s; logic o; logic c; } type_scr1_ialu_flags_s;

typedef enum logic [1:0] { SCR1_IALU_MDU_FSM_IDLE, SCR1_IALU_MDU_FSM_ITER,
                           SCR1_IALU_MDU_FSM_CORR } type_scr1_ialu_fsm_state;
typedef enum logic [1:0] { SCR1_IALU_MDU_NONE, SCR1_IALU_MDU_MUL,
                           SCR1_IALU_MDU_DIV } type_scr1_ialu_mdu_cmd;
```

### 3.3 主要信号（`:100-190`）

| 分组 | 信号 | 行号 |
|---|---|---|
| 主加法器 | `main_sum_res[XLEN:0]`、`main_sum_flags`、`main_sum_pos/neg_ovflw` | 101-104 |
| 移位 | `ialu_cmd_shft`、`shft_op1/op2`、`shft_cmd`、`shft_res` | 109-113 |
| MDU 控制 | `mdu_cmd_is_iter`、`mdu_iter_req/rdy`、`mdu_corr_req`、`div/rem_corr_req` | 118-123 |
| MDU FSM | `mdu_fsm_ff/next/idle/iter/corr` | 126-132 |
| MDU 命令 | `mdu_cmd`、`mdu_cmd_mul/div`、`mul_cmd`、`div_cmd` | 135-142 |
| 乘法器 | `mul_op1/2`、`mul_res`（FAST）或 `mul_part_prod`/`mul_res_hi/lo` | 145-157 |
| 除法器 | `div_*` 系列 | 160-169 |
| MDU 加法器 | `mdu_sum_sub/op1/op2/res` | 172-175 |
| 迭代计数 | `mdu_iter_cnt*` | 178-180 |
| 中间结果 | `mdu_res_c/hi/lo_ff/next` | 183-189 |

> `scr1_search_ms1.svh` 在 `:27` 被 include，但本文件未调用其函数（前导零搜索为遗留引用）。

---

## 4. 主加法器（`:192-221`）

```systemverilog
always_comb begin
    main_sum_res = (exu2ialu_cmd_i != SCR1_IALU_CMD_ADD)
                 ? ({1'b0, op1} - {1'b0, op2})      // 减法/比较
                 : ({1'b0, op1} + {1'b0, op2});     // 加法
    main_sum_pos_ovflw = ~op1[XLEN-1] &  op2[XLEN-1] &  main_sum_res[XLEN-1];
    main_sum_neg_ovflw =  op1[XLEN-1] & ~op2[XLEN-1] & ~main_sum_res[XLEN-1];
    main_sum_flags.c = main_sum_res[XLEN];         // 借位/进位
    main_sum_flags.z = ~|main_sum_res[XLEN-1:0];
    main_sum_flags.s = main_sum_res[XLEN-1];
    main_sum_flags.o = main_sum_pos_ovflw | main_sum_neg_ovflw;
end
```

- 只有 `CMD_ADD` 走加法，**其余命令（含 SUB 与全部比较）走减法**；
- 结果位宽 XLEN+1，故最高位即进位/借位，比较语义天然正确；
- 溢出按“同号相加变号”判定。

---

## 5. 地址加法器（`:223-235`）

```systemverilog
assign ialu2exu_addr_res_o = exu2ialu_addr_op1_i + exu2ialu_addr_op2_i;  // :235
```

- 单周期组合加，无标志；
- EXU 侧按 `sum2_op` 选择操作数（EXU `:432-440`）：
  - `PC_IMM`：op1=PC，op2=imm（AUIPC、JAL/分支目标）；
  - `REG_IMM`：op1=rs1，op2=imm（JALR 目标、load/store 地址）。

---

## 6. 移位逻辑（`:237-263`）

```systemverilog
assign ialu_cmd_shft = (cmd==SLL)|(cmd==SRL)|(cmd==SRA);
assign shft_cmd = ialu_cmd_shft ? {(cmd != SLL), (cmd == SRA)} : 2'b00;
// SLL=2'b00, SRL=2'b10, SRA=2'b11
case (shft_cmd)
    2'b10   : shft_res = shft_op1 >>  shft_op2;   // 逻辑右移
    2'b11   : shft_res = shft_op1 >>> shft_op2;   // 算术右移
    default : shft_res = shft_op1 <<  shft_op2;   // 逻辑左移
endcase
```

- `shft_op1` 为 `signed`，故 `>>>` 做算术右移；
- 移位量取 `op2[4:0]`（低 5 位），符合 RV32 规范。

---

## 7. MUL/DIV 单元（MDU，`:265-550`）

仅 `SCR1_RVM_EXT` 编译。

### 7.1 命令解码（`:284-295`）

- `mdu_cmd_div`：DIV/DIVU/REM/REMU；
- `mdu_cmd_mul`：MUL/MULH/MULHU/MULHSU；
- `mdu_cmd`：DIV 优先，其次 MUL，否则 NONE。

辅助量（`:297-299`）：

```systemverilog
main_ops_non_zero = |op1 & |op2;               // 任一为 0 则 0
main_ops_diff_sgn = op1[XLEN-1] ^ op2[XLEN-1]; // 符号不同
```

### 7.2 迭代控制（`:301-334`）

```systemverilog
SCR1_FAST_MUL: mdu_cmd_is_iter = mdu_cmd_div;              // 仅除法迭代
else:          mdu_cmd_is_iter = mdu_cmd_mul | mdu_cmd_div; // 乘除都迭代

mdu_iter_req = mdu_cmd_is_iter ? (main_ops_non_zero & mdu_fsm_idle) : 1'b0; // :307
mdu_iter_rdy = mdu_iter_cnt[0];                                             // :308

mdu_iter_cnt_en   = exu2ialu_rvm_cmd_vd_i & ~ialu2exu_rvm_res_rdy_o;        // :321
mdu_iter_cnt_next = ~mdu_fsm_idle ? (mdu_iter_cnt >> 1)                     // :329
                  : mdu_cmd_div   ? SCR1_DIV_CNT_INIT
`ifndef SCR1_FAST_MUL
                  : mdu_cmd_mul   ? SCR1_MUL_CNT_INIT
`endif
                                  : mdu_iter_cnt;
```

- 计数器初始只有最高位为 1，每次迭代右移；`cnt[0]==1` 表示最后一轮；
- **操作数含 0 时 `mdu_iter_req=0`**，MDU 不进入 ITER，直接在 IDLE 给出结果
  （乘 0 → 0；除/余 0 → 见 §7.7）；
- `always_ff @(posedge clk)`（`:323-327`）**无复位**，依赖使能写入。

### 7.3 MDU FSM（`:336-373`）

| 当前 | 条件 | 次态 |
|---|---|---|
| IDLE | `mdu_iter_req` | ITER |
| IDLE | 否则 | IDLE |
| ITER | `~mdu_iter_rdy` | ITER |
| ITER | `mdu_iter_rdy & mdu_corr_req` | CORR |
| ITER | `mdu_iter_rdy & ~mdu_corr_req` | IDLE |
| CORR | — | IDLE |

- 整段以 `exu2ialu_rvm_cmd_vd_i` 为门槛；命令撤销时次态回 IDLE；
- `mdu_corr_req = mdu_cmd_div & (div_corr_req | rem_corr_req)`（`:316`），
  仅除法可能需要校正（非恢复算法的符号修正）；
  - `div_corr_req = div_cmd_div & ops_diff_sgn`（`:314`）；
  - `rem_corr_req = div_cmd_rem & |div_res_rem & (op1_neg ^ rem_c)`（`:315`）。

### 7.4 乘法器（`:375-421`）

**FAST_MUL**（`:406-409`）：组合乘，结果 2·XLEN 位。

**Radix-2**（`:410-421`）：32 周期迭代；`mul_op2` 在 idle 取被乘数低段，
迭代中取 `mdu_res_lo_ff` 高位移入低位；`mul_part_prod` 为部分积，
`{mul_res_hi, mul_res_lo}` 更新高低段。

符号控制（`:397-404`）：

```systemverilog
mul_cmd = { (MULHU|MULHSU), (MULHU|MULH) };   // MUL=00,MULH=01,MULHSU=10,MULHU=11
mul_cmd_hi     = |mul_cmd;                    // MULH* 要高位
mul_op1_is_sgn = ~&mul_cmd;                   // 除 MULHU 外有符号
mul_op2_is_sgn = ~mul_cmd[1];                 // MULHU/MULHSU 的第二操作数无符号
```

### 7.5 除法器（非恢复算法，`:423-485`）

```systemverilog
div_cmd = { (REM|REMU), (REMU|DIVU) };  // DIV=00,DIVU=01,REM=10,REMU=11 // :449
div_ops_are_sgn = ~div_cmd[0];          // DIV/REM 有符号
div_op1_is_neg  = div_ops_are_sgn & op1[XLEN-1];
div_op2_is_neg  = div_ops_are_sgn & op2[XLEN-1];
```

组合计算（`:456-470`）：`div_res_rem_c`（余数符号）、`div_res_rem`（余数）、
`div_quo_bit`（商位）、`div_res_quo`（商）。商位含**边界情况**处理：
被除数为负时需同时判断“余数 > 除数”与“低半部分为 0”（注释 `:442-446`）：

```systemverilog
div_quo_bit = ~(div_op1_is_neg ^ div_res_rem_c)
            | (div_op1_is_neg & ({mdu_sum_res, div_dvdnd_lo_next} == '0));  // :464-465
```

被除数低位寄存器（`:472-485`）跟踪用于边界判断，`<<1` 推进。

### 7.6 MDU 加法器（`:487-523`）

按 `mdu_cmd` 复用同一加减器：

- **DIV**：`sgn`/`inv` 决定加减，操作数来自 `mdu_res_hi_ff`/`div_dvdnd_lo_ff`/被除数；
- **MUL**（非 FAST）：累加部分积 `mul_part_prod` 到 `mdu_res_hi_ff`；
- 输出 `mdu_sum_res = sub ? (op1-op2) : (op1+op2)`。

### 7.7 中间结果寄存器（`:525-549`）

```systemverilog
mdu_res_upd = exu2ialu_rvm_cmd_vd_i & ~ialu2exu_rvm_res_rdy_o;  // :529
always_ff @(posedge clk) if (mdu_res_upd) begin ... end          // :531-537
mdu_res_c_next  = mdu_cmd_div ? div_res_rem_c : mdu_res_c_ff;
mdu_res_hi_next = mdu_cmd_div ? div_res_rem
                : (mdu_cmd_mul ? mul_res_hi : mdu_res_hi_ff);
mdu_res_lo_next = mdu_cmd_div ? div_res_quo
                : (mdu_cmd_mul ? mul_res_lo : mdu_res_lo_ff);
```

同上，**无复位**，仅在 `mdu_res_upd` 时更新。

---

## 8. 结果形成（`:552-656`）

输出 mux 默认值（`:557-561`）：`main_res=0`、`cmp_res=0`、`rvm_res_rdy=1`（非 RVM 恒就绪）。

| 命令 | `main_res` | `cmp_res` | 行号 |
|---|---|---|---|
| AND/OR/XOR | 位运算 | 0 | 564-572 |
| ADD / SUB | `main_sum_res[XLEN-1:0]` | 0 | 573-578 |
| SUB_LT | `s ^ o` | `s ^ o` | 579-582 |
| SUB_LTU | `c` | `c` | 583-586 |
| SUB_EQ | `z` | `z` | 587-590 |
| SUB_NE | `~z` | `~z` | 591-594 |
| SUB_GE | `~(s^o)` | `~(s^o)` | 595-598 |
| SUB_GEU | `~c` | `~c` | 599-602 |
| SLL/SRL/SRA | `shft_res` | 0 | 603-607 |
| MUL* (FAST) | `mul_cmd_hi ? mul_res[2·XLEN-1:XLEN] : mul_res[XLEN-1:0]` | 0 | 613-616 |
| MUL* (非 FAST) | IDLE→0、ITER→`mul_cmd_hi?mul_res_hi:mul_res_lo`；`rdy` 随状态 | 0 | 617-628 |
| DIV* | IDLE/ITER/CORR 三段给出不同结果与 `rdy` | 0 | 630-652 |

### 8.1 除零/除余特殊语义（`:634-639`）

```systemverilog
IDLE: main_res = (|op2 | div_cmd_rem) ? op1 : '1;
      rvm_res_rdy = ~mdu_iter_req;
```

- 除数为 0：DIV/DIVU 结果 = 全 1（`'1`）；REM/REMU 结果 = 被除数；
- 或被除数 = 0：结果 = 0（`op1` 为 0），直接 IDLE 完成；
- 这正是 RISC-V M 扩展的除零/零除语义。

### 8.2 CORR 结果（`:645-650`）

- REM：`mdu_sum_res[XLEN-1:0]`；
- DIV：`-mdu_res_lo_ff[XLEN-1:0]`（取负校正）；
- `rvm_res_rdy=1`。

---

## 9. 配置宏影响

| 宏 | 位置 | 影响 |
|---|---|---|
| `SCR1_RVM_EXT` | `:31-37`、`:56-69`、`:82-190`、`:265-653`、`:664-716` | 增加乘除端口/逻辑/断言；无则不例化 MDU |
| `SCR1_FAST_MUL` | `:57-66`、`:151-157`、`:301-305`、`:331-333`、`:406-421`、`:510-517`、`:541-548`、`:613-628` | 乘法 1 周期；无则 Radix-2 32 周期，MUL 也走 ITER |
| `SCR1_TRGT_SIMULATION` | `:129-131`、`:370-372`、`:659-718` | 编译 `mdu_fsm_iter` 与 SVA |

`SCR1_XLEN` 影响所有位宽与计数器初值。

---

## 10. 时序

### 10.1 非 M 指令

全组合，单周期出结果。

### 10.2 FAST_MUL 乘法

组合乘，`rvm_res_rdy=1`，单周期完成（EXU 当拍写回）。

### 10.3 除法 / 非 FAST_MUL 乘法

1. **IDLE**：`exu2ialu_rvm_cmd_vd_i=1` 且操作数非零 → `mdu_iter_req=1`，次态 ITER；
2. **ITER**：每拍迭代，`mdu_iter_cnt` 右移；`cnt[0]=1` 时 `iter_rdy=1`；
3. `iter_rdy & corr_req` → **CORR** 一拍做符号校正，`rdy=1`；
4. 否则回 **IDLE**，`rdy=1`。
- `ialu_rdy` 在 EXU 中作为 `exu_rdy` 来源（EXU `:802`），期间 `exu_busy=1`。

---

## 11. 假设与边界

1. `exu2ialu_cmd_i` 在 `exu2ialu_rvm_cmd_vd_i` 有效期间保持稳定（EXU 队列寄存）；
2. MDU 内部计数器/寄存器无复位，仅靠 `exu2ialu_rvm_cmd_vd_i & ~rdy` 使能写入；
   首次使用前状态可能为 X，但首次写覆盖，功能不受影响；
3. 无 RVM 时 `rvm_res_rdy` 恒 1；
4. 除零/零除语义由 IDLE 特判给出，不进入迭代。

---

## 12. 内建断言（`:659-718`，仅 RVM）

| 断言 | 行号 | 检查 |
|---|---|---|
| `XCHECK` | 668-671 | `rvm_cmd_vd`/FSM 无 X |
| `XCHECK_QUEUE` | 673-677 | 命令有效时操作数/命令无 X |
| `ILL_STATE` | 681-684 | `{vld,iter,corr}` 至多一热 |
| `JUMP_FROM_IDLE` | 686-689 | IDLE 无请求保持 IDLE |
| `IDLE_TO_ITER` | 691-694 | IDLE+请求→ITER |
| `JUMP_FROM_ITER` | 696-699 | ITER 未就绪保持 ITER |
| `ITER_TO_IDLE` | 701-704 | ITER 就绪无校正→IDLE |
| `ITER_TO_CORR` | 706-709 | ITER 就绪需校正→CORR |
| `CORR_TO_IDLE` | 711-714 | CORR 必回 IDLE |

---

## 附录 A：端口速查

见第 2 节。RVM 时 12 个端口，非 RVM 时 8 个。

## 附录 B：命令枚举

`type_scr1_ialu_cmd_sel_e`（`scr1_riscv_isa_decoding.svh:59-86`）：

| 类 | 命令 |
|---|---|
| 逻辑 | AND, OR, XOR |
| 算术 | ADD, SUB |
| 比较 | SUB_LT, SUB_LTU, SUB_EQ, SUB_NE, SUB_GE, SUB_GEU |
| 移位 | SLL, SRL, SRA |
| M 扩展 | MUL, MULHU, MULHSU, MULH, DIV, DIVU, REM, REMU |
| 其它 | NONE |

宽度：`SCR1_IALU_CMD_ALL_NUM_E` = 23（RVM）/15（非 RVM）。

## 附录 C：比较标志映射

| 条件 | 表达式 |
|---|---|
| `==` | `z` |
| `!=` | `~z` |
| `<`（有符号） | `s ^ o` |
| `>=`（有符号） | `~(s^o)` |
| `<`（无符号） | `c` |
| `>=`（无符号） | `~c` |

## 附录 D：信号 → 消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `ialu2exu_main_res_o` | EXU 写回 mux | 算术/逻辑/移位/乘除结果、slt 布尔 |
| `ialu2exu_cmp_res_o` | EXU `branch_taken` | 分支条件 |
| `ialu2exu_addr_res_o` | EXU New PC / LSU 地址 | 目标地址 |
| `ialu2exu_rvm_res_rdy_o` | EXU `exu_rdy` | 多周期完成 |

## 附录 E：与 EXU 边界

- EXU 预选主操作数与地址操作数，并门控 `ialu_vd`（EXU `:403-440`）；
- EXU 根据 `ialu_vd` 把 `exu_rdy` 切到 `ialu_rdy`（EXU `:802`）；
- IALU 不参与异常；比较结果仅用于分支/`slt`，写回由 EXU 选择。
