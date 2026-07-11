# SCR1 EXU 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_exu.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| addr | address | 地址 |
| alw | always | 始终开启（如 `clk_alw_on` — 不受门控的时钟） |
| auipc | add upper immediate to PC | 高 20 位立即数加 PC |
| awidth | address width | 地址位宽 |

---

## B

| 缩写 | 全称 | 含义 |
|------|------|------|
| brkpt | breakpoint | 断点 |
| brkm | breakpoint monitor | 断点监视器 |
| busy | — | 忙标志 |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| clk | clock | 时钟 |
| clkctrl | clock control | 时钟门控控制 |
| cmp(cmp) | compare | 比较结果 |
| cmd | command | 命令 |
| cnt | counter | 计数器（如 `SCR1_CSR_REDUCED_CNT`） |
| csr | control and status register | 控制与状态寄存器 |
| curr | current | 当前的 |
| ctrl | control | 控制信号 |

---

## D

| 缩写 | 全称 | 含义 |
|------|------|------|
| dbg | debug | 调试 |
| dbrkpt | data breakpoint | 数据断点 |
| dmem | data memory | 数据存储器 |
| dmode | debug mode | 调试模式 |
| dmon | data monitor | 数据监视器 |
| dsbl | disable | 禁用 |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| e（后缀） | enum | 枚举类型命名后缀（如 `type_scr1_exc_code_e`） |
| ebreak | environment break | 环境断点 |
| ecall | environment call | 环境调用 |
| en | enable | 使能 |
| exc | exception | 异常 |
| exu | execution unit | 执行单元 |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| fencei | FENCE.I (instruction fence) | 指令缓存刷新屏障 |
| ff | flip-flop | 触发器（寄存器后缀，如 `pc_curr_ff`） |
| fsm | finite state machine | 有限状态机 |

---

## G

| 缩写 | 全称 | 含义 |
|------|------|------|
| ge(u) | greater or equal (unsigned) | 大于等于比较 |

---

## H

| 缩写 | 全称 | 含义 |
|------|------|------|
| halt | — | 暂停／挂起（WFI halted / debug halted） |
| hdu | hardware debug unit | 硬件调试单元 |
| hi | high | 高半部分（如 `instr_fault_rvi_hi` — RVI 高半取指错误） |
| hw | hardware | 硬件（如 `exu2hdu_ibrkpt_hw_o`） |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| i（后缀） | input | 输入端口后缀 |
| ialu | integer arithmetic logic unit | 整数算术逻辑单元（SCR1 中合并了主 ALU、地址加法器和乘法器） |
| ibrkpt | instruction breakpoint | 指令断点 |
| idu | instruction decode unit | 指令译码单元 |
| ifu | instruction fetch unit | 指令取指单元 |
| imm | immediate | 立即数 |
| imon | instruction monitor | 指令监视器 |
| inc | increment | 递增 |
| init | initialize | 初始化 |
| instret | instruction retired | 指令退休（retired）计数 |
| instr | instruction | 指令 |
| ip | interrupt pending | 中断挂起 |
| ie | interrupt enable | 中断使能 |
| irq | interrupt request | 中断请求 |

---

## J

| 缩写 | 全称 | 含义 |
|------|------|------|
| jb | jump / branch | 跳转或分支的统称 |
| jump | — | 无条件跳转 |

---

## L

| 缩写 | 全称 | 含义 |
|------|------|------|
| ldata | load data | 加载数据 |
| lsu | load-store unit | 加载存储单元 |
| lt(u) | less than (unsigned) | 小于比较 |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| misalign | — | 地址未对齐 |
| mie | machine interrupt enable | 机器模式中断使能 |
| mprf | multi-port register file | 多端口寄存器文件 |
| mret | machine-mode return | 机器模式异常返回 |
| mstatus | machine status | 机器模式状态寄存器 |
| mtval | machine trap value | 机器模式异常值寄存器 |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n（后缀） | negation | 取反／低有效（如 `rst_n`） |
| ne | not equal | 不等于 |
| no | — | 否定前缀（如 `no_commit` — 禁止提交） |
| npbuf | not program buffer | 非程序缓冲区（如 `dbg_run_start_npbuf`） |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o（后缀） | output | 输出端口后缀 |
| op | operand / operation | 操作数／操作 |

---

## P

| 缩写 | 全称 | 含义 |
|------|------|------|
| pbuf | program buffer | 程序缓冲区（调试模式下的指令源） |
| pc | program counter | 程序计数器 |
| pnd | pending | 待处理 |
| priv | privileged | 特权指令 |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| rd | register destination | 目标寄存器 |
| rdy | ready | 就绪 |
| reg | register | 寄存器 |
| req | request | 请求 |
| ret | retire | 退休／提交（如 `exu2tdu_ibrkpt_ret_o`） |
| rst | reset | 复位 |
| rs1 / rs2 | register source 1 / 2 | 源寄存器 1 / 2 |
| run | — | 运行状态 |
| rvi | RV32I | RISC-V 基础整数指令集 |
| rvc | RISC-V compressed | RISC-V 16 位压缩指令集扩展 |
| rvm | RISC-V multiply/divide | RISC-V 整数乘除法扩展 |
| rw | read/write | 读写 |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| s（后缀） | struct | 结构体类型后缀（如 `type_scr1_exu_cmd_s`） |
| scr1 | Syntacore RISC-V Core 1 | 处理器核心名称 |
| sdata | store data | 存储数据 |
| sel | select | 选择 |
| slt(u) | set less than (unsigned) | 小于则置位 |
| sra | shift right arithmetic | 算术右移 |
| srl | shift right logical | 逻辑右移 |
| sstep | single step | 单步调试 |
| step | — | 步进（同 sstep） |
| sub | subtract | 减法 |
| sum2 | second sum adder | 第二个加法器（地址计算专用通路） |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| tdu | trigger debug unit | 触发调试单元 |
| trgt | target | 目标平台（如 `SCR1_TRGT_SIMULATION`） |
| trap | — | 陷阱（异常／中断的统称） |
| trig | trigger | 触发器（调试触发条件） |

---

## U

| 缩写 | 全称 | 含义 |
|------|------|------|
| upd | update | 更新 |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| vd | valid | 有效标志 |
| v（后缀） | vector / version | 版本（如 `init_pc_v` — PC 初始化计数器） |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| w（后缀） | width | 位宽（如 `awidth`） |
| wb | write back | 写回 |
| wdata | write data | 写数据 |
| wfi | wait for interrupt | 等待中断指令 |
