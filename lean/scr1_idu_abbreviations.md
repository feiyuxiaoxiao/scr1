# SCR1 IDU 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_idu.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| addi | add immediate | 立即数加法 |
| addi4spn | add immediate 4 × stack pointer n | 栈指针偏移加法（4 字节对齐） |
| addi16sp | add immediate 16 × stack pointer | 栈指针 16 字节对齐偏移加法 |
| addr | address | 地址 |
| alu / ialu | (integer) arithmetic logic unit | （整数）算术逻辑单元 |
| auipc | add upper immediate to PC | PC 加高位立即数 |

---

## B

| 缩写 | 全称 | 含义 |
|------|------|------|
| beq / bne | branch equal / not equal | 相等/不等分支 |
| blt(u) / bge(u) | branch less than / greater or equal (unsigned) | 有/无符号比较分支 |
| branch | — | 条件分支 |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| clk | clock | 时钟 |
| cmd | command | 命令（如 `ialu_cmd`、`lsu_cmd`） |
| csr | control and status register | 控制和状态寄存器 |
| csrrw(i) | CSR read and write (immediate) | CSR 读后写（立即数形式） |
| csrrs(i) | CSR read and set bits (immediate) | CSR 读后置位（立即数形式） |
| csrrc(i) | CSR read and clear bits (immediate) | CSR 读后清零（立即数形式） |

---

## D

| 缩写 | 全称 | 含义 |
|------|------|------|
| div(u) | divide (unsigned) | 有/无符号除法 |
| dmem | data memory | 数据存储器 |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| ebreak | environment break | 环境断点（进入调试模式） |
| ecall | environment call | 环境调用（M-mode 下为 ECALL_M） |
| exc | exception | 异常 |
| exu | execution unit | 执行单元 |
| e（后缀） | enum | 枚举类型后缀（如 `type_scr1_instr_type_e`） |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| fence(i) | — (instruction fence) | 存储器序同步/指令缓存刷新 |
| funct3/7/12 | function 3/7/12 | RISC-V 指令格式中的功能码字段 |

---

## G

| 缩写 | 全称 | 含义 |
|------|------|------|
| ge(u) | greater or equal (unsigned) | 有/无符号大于等于比较 |
| gpr | general purpose register | 通用寄存器 |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| i（后缀） | input | 输入端口后缀 |
| idu | instruction decode unit | 指令译码单元 |
| ifu | instruction fetch unit | 指令取指单元 |
| imm | immediate | 立即数 |
| inc | increment | 递增（如 `INC_PC` = PC+4） |
| instr | instruction | 指令 |
| isa | instruction set architecture | 指令集架构 |

---

## J

| 缩写 | 全称 | 含义 |
|------|------|------|
| jal(r) | jump and link (register) | 跳转并链接（寄存器间接） |

---

## L

| 缩写 | 全称 | 含义 |
|------|------|------|
| lb(u) / lh(u) / lw | load byte/half/word (unsigned) | 字节/半字/字加载（无符号扩展） |
| lsu | load-store unit | 加载存储单元 |
| lui | load upper immediate | 加载高位立即数 |
| lt(u) | less than (unsigned) | 有/无符号小于比较 |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| mem | memory | 存储器 |
| misc | miscellaneous | 杂项 |
| mprf | multi-port register file | 多端口寄存器文件 |
| mret | machine-mode return | 机器模式异常返回 |
| mtval | machine trap value | 机器模式异常值寄存器 |
| mul(h(u/su)) | multiply (high; unsigned/signed-unsigned) | 乘法（取高位；无符号/有符号-无符号混合） |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n（前缀） | negation | 取反/低有效（如 `rst_n`） |
| ne | not equal | 不等于 |
| nop | no operation | 空操作 |
| none | — | 无效/无操作（枚举默认值） |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o（后缀） | output | 输出端口后缀 |
| op | operation | 操作（RV32I 操作码前缀） |
| opcode | operation code | 操作码（指令 `[6:2]` 字段） |

---

## P

| 缩写 | 全称 | 含义 |
|------|------|------|
| pc | program counter | 程序计数器 |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| ra | return address | 返回地址（x1 寄存器 ABI 名称） |
| rd | register destination | 目标寄存器（指令 `[11:7]` 字段） |
| rdy | ready | 就绪 |
| reg | register | 寄存器 |
| rem(u) | remainder (unsigned) | 有/无符号取余 |
| req | request | 请求 |
| rst | reset | 复位 |
| rs1 / rs2 | register source 1 / 2 | 源寄存器 1 / 2（指令 `[19:15]` / `[24:20]` 字段） |
| rve | RISC-V embedded (RV32E) | RISC-V 嵌入式基础整数指令集（16 寄存器） |
| rvi | RISC-V integer (RV32I) | RISC-V 基础整数指令集（32 寄存器） |
| rvc | RISC-V compressed | RISC-V 16 位压缩指令集扩展 |
| rvm | RISC-V multiply/divide | RISC-V 整数乘除法扩展 |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| sb / sh / sw | store byte/half/word | 字节/半字/字存储 |
| scr1 | Syntacore RISC-V Core 1 | 处理器核心名称 |
| sel | select | 选择（枚举类型后缀信号，如 `rd_wb_sel`） |
| shamt | shift amount | 移位量（`instr[24:20]`） |
| sll(i) | shift left logical (immediate) | 逻辑左移（立即数） |
| sp | stack pointer | 栈指针（x2 寄存器 ABI 名称） |
| sra(i) | shift right arithmetic (immediate) | 算术右移（立即数） |
| srl(i) | shift right logical (immediate) | 逻辑右移（立即数） |
| sub | subtract | 减法 |
| sum2 | — | 第二个加法器（计算跳转/访存地址的独立加法通路） |
| svh | SystemVerilog header | 头文件后缀 |
| system | — | 系统指令操作码 |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| trgt | target | 目标（如 `SCR1_TRGT_SIMULATION`） |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| vd | valid | 有效标志 |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| wb | writeback | 回写（寄存器文件写回阶段） |
| wfi | wait for interrupt | 等待中断 |

---

## X

| 缩写 | 全称 | 含义 |
|------|------|------|
| x0-x31 | — | RISC-V 通用寄存器命名（x0=零寄存器, x1=ra, x2=sp 等） |
| xlen | — | RISC-V 通用寄存器位宽（SCR1 为 32） |
| xprop | X propagation | X 态传播检测（仿真用，枚举类型 fallback 值设为 `'x`） |

---

## Z

| 缩写 | 全称 | 含义 |
|------|------|------|
| zimm | zero-extended immediate | 零扩展立即数（CSR 立即数形式中 `rs1` 字段的零扩展值） |
