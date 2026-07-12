# SCR1 IALU 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_ialu.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| add | addition | 加法 |
| addi | add immediate | 立即数加法 |
| addr | address | 地址 |
| alu / ialu | (integer) arithmetic logic unit | （整数）算术逻辑单元 |
| and(i) | AND (immediate) | 按位与（立即数形式） |
| auipc | add upper immediate to PC | PC 高 20 位立即数加法 |

---

## B

| 缩写 | 全称 | 含义 |
|------|------|------|
| beq | branch if equal | 相等则分支 |
| bge(u) | branch if greater or equal (unsigned) | 有/无符号大于等于则分支 |
| blt(u) | branch if less than (unsigned) | 有/无符号小于则分支 |
| bne | branch if not equal | 不等则分支 |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| c (flag) | carry | 进位/借位标志 |
| clk | clock | 时钟信号 |
| cmd | command | 命令 |
| cmp | compare | 比较 |
| corr | correction | 修正（有符号除法结果修正阶段） |
| cnt | counter | 计数器 |

---

## D

| 缩写 | 全称 | 含义 |
|------|------|------|
| diff | different | 不同、差异 |
| div | divide | 除法 |
| divu | divide unsigned | 无符号除法 |
| dvdnd | dividend | 被除数 |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| en | enable | 使能 |
| eq | equal | 等于 |
| exu | execution unit | 执行单元 |
| e (后缀) | enum | 枚举类型命名后缀 |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| ff | flip-flop | 触发器/寄存器（后缀命名习惯） |
| fsm | finite state machine | 有限状态机 |
| flags | — | 标志位集合 |

---

## G

| 缩写 | 全称 | 含义 |
|------|------|------|
| ge(u) | greater or equal (unsigned) | 有/无符号大于等于 |

---

## H

| 缩写 | 全称 | 含义 |
|------|------|------|
| hi | high | 高位部分 |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| i (后缀) | input | 输入端口后缀 |
| ialu | integer arithmetic logic unit | 整数算术逻辑单元 |
| idle | — | 空闲状态 |
| ill | illegal | 非法 |
| iter | iteration | 迭代 |

---

## L

| 缩写 | 全称 | 含义 |
|------|------|------|
| lo | low | 低位部分 |
| lt(u) | less than (unsigned) | 有/无符号小于 |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| mdu | multiply-divide unit | 乘除法单元 |
| mul | multiply | 乘法 |
| mulh | multiply high | 有符号乘法取高 32 位 |
| mulhsu | multiply high signed-unsigned | 有符号×无符号乘法取高 32 位 |
| mulhu | multiply high unsigned | 无符号乘法取高 32 位 |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n (后缀) | negation (active low) | 取反/低有效（如 rst_n） |
| ne | not equal | 不等于 |
| neg | negative | 负 |
| none | — | 无操作 |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o (flag, 后缀) | overflow / output | 溢出标志 / 输出端口后缀 |
| op | operand / operation | 操作数 / 操作 |
| ops | operands | 操作数（复数） |
| or(i) | OR (immediate) | 按位或（立即数形式） |
| ovflw | overflow | 溢出 |

---

## P

| 缩写 | 全称 | 含义 |
|------|------|------|
| pc | program counter | 程序计数器 |
| pos | positive | 正 |
| prod | product | 积 |

---

## Q

| 缩写 | 全称 | 含义 |
|------|------|------|
| quo | quotient | 商 |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| rdy | ready | 就绪 |
| rem | remainder | 余数 |
| remu | remainder unsigned | 无符号取余 |
| req | request | 请求 |
| res | result | 结果 |
| rst | reset | 复位 |
| rvm | RISC-V M extension | RISC-V 整数乘除法扩展 |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| s (flag) | sign | 符号标志 |
| sgn | sign | 符号 |
| shft | shift | 移位 |
| sll(i) | shift left logical (immediate) | 逻辑左移（立即数形式） |
| slt(i(u)) | set if less than (immediate, unsigned) | 有/无符号小于置位 |
| sra(i) | shift right arithmetic (immediate) | 算术右移（立即数形式） |
| srl(i) | shift right logical (immediate) | 逻辑右移（立即数形式） |
| sub | subtract | 减法 |
| sum | summation | 和、加法结果 |
| sva | SystemVerilog assertion | SystemVerilog 断言 |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| trgt | target | 目标（如 SCR1_TRGT_SIMULATION） |

---

## U

| 缩写 | 全称 | 含义 |
|------|------|------|
| upd | update | 更新 |
| u (后缀) | unsigned | 无符号（如 bltu、remu） |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| vd | valid | 有效 |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| w (后缀) | width | 位宽（如 SCR1_MUL_WIDTH） |

---

## X

| 缩写 | 全称 | 含义 |
|------|------|------|
| xcheck | X-state check | X 态检查断言 |
| xlen | — | RISC-V 通用寄存器位宽（32） |
| xor(i) | XOR (immediate) | 按位异或（立即数形式） |

---

## Z

| 缩写 | 全称 | 含义 |
|------|------|------|
| z (flag) | zero | 零标志 |
