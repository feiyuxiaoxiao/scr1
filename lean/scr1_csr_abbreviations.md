# SCR1 CSR 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_csr.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| addr | address | 地址 |
| archid | architecture ID | 架构标识符 |

---

## B

| 缩写 | 全称 | 含义 |
|------|------|------|
| bp | bypass | 旁路 |
| brkm | breakpoint module | 硬件断点模块（TDU 一部分） |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| cisv | current interrupt source vector | 当前中断源向量（IPIC） |
| cicsr | current interrupt context CSR | 当前中断上下文（IPIC） |
| clk | clock | 时钟 |
| clkctrl | clock control | 时钟门控 |
| cmd | command | 命令（CSR 读写命令：WRITE/SET/CLEAR） |
| cnt | counter | 计数器 |
| csr | control and status register | 控制和状态寄存器 |
| cy | cycle | 周期计数器使能位（MCOUNTEN.CY） |
| cycle(h) | cycle (high) | 周期计数器（高 32 位） |

---

## D

| 缩写 | 全称 | 含义 |
|------|------|------|
| dbg | debug | 调试子系统 |
| dcsr | debug control and status register | 调试控制状态寄存器 |
| dscratch | debug scratch | 调试暂存寄存器 |
| dwidth | data width | 数据位宽 |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| e（前缀） | event | 事件标志（e_exc / e_irq / e_mret） |
| e（后缀） | enum | 枚举类型后缀 |
| ec | exception code | 异常/中断编码 |
| ecall | environment call | 环境调用 |
| eirq | external interrupt request | 外部中断请求 |
| eoi | end of interrupt | 中断结束（IPIC） |
| exc | exception | 异常 |
| exu | execution unit | 执行单元 |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| ff（后缀） | flip-flop | 触发器/寄存器后的信号（当前状态） |
| fuse | — | 熔丝/硬连线配置值 |

---

## H

| 缩写 | 全称 | 含义 |
|------|------|------|
| hdu | hardware debug unit | 硬件调试单元 |
| hpm | hardware performance monitor | 硬件性能监视器 |
| hpmcounter | hardware performance monitor counter | 硬件性能监视计数器 |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| i（后缀） | input | 输入端口 |
| icsr | interrupt context status register | 中断上下文状态寄存器（IPIC） |
| idx | index | 索引（IPIC） |
| ie | interrupt enable | 中断使能 |
| impid | implementation ID | 实现标识符 |
| inc | increment | 递增 |
| instret(h) | instruction retired (high) | 已退休指令计数器（高 32 位） |
| ip | interrupt pending | 中断挂起 |
| ipic | integrated programmable interrupt controller | 集成可编程中断控制器 |
| ipr | interrupt pending register | 中断挂起寄存器（IPIC） |
| ir | instruction retired | 指令退休计数器使能位（MCOUNTEN.IR） |
| irq | interrupt request | 中断请求 |
| isa | instruction set architecture | 指令集架构 |
| isvr | interrupt service vector register | 中断服务向量寄存器（IPIC） |

---

## L

| 缩写 | 全称 | 含义 |
|------|------|------|
| lo / hi | low / high | 低/高 8/32 位（计数器拆分） |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| marchid | machine architecture ID | 机器架构标识符 |
| mask | — | 位掩码 |
| mcause | machine cause | 机器异常原因寄存器 |
| mcounten | machine counter enable | 机器计数器使能寄存器（非标准 CSR） |
| mcountinhibit | machine counter inhibit | 机器计数器抑制寄存器 |
| mcycle(h) | machine cycle (high) | 机器周期计数器（高 32 位） |
| meie | machine external interrupt enable | 机器外部中断使能位 |
| meip | machine external interrupt pending | 机器外部中断挂起位 |
| mepc | machine exception program counter | 机器异常返回地址寄存器 |
| mhartid | machine hart ID | 机器硬件线程标识符 |
| mie | machine interrupt enable | 机器中断使能寄存器 |
| mimp | machine implementation ID | 机器实现标识 |
| minstret(h) | machine instructions retired (high) | 机器已退休指令计数器（高 32 位） |
| mip | machine interrupt pending | 机器中断挂起寄存器 |
| misa | machine ISA register | 机器 ISA 与控制寄存器 |
| mode | — | 模式（MTVEC.mode：direct / vectored） |
| mpie | machine previous interrupt enable | 机器先前中断使能位（MSTATUS.MPIE） |
| mpp | machine previous privilege | 机器先前特权级（MSTATUS.MPP） |
| mprf | multi-port register file | 多端口寄存器文件 |
| mret | machine-mode return | 机器模式异常返回指令 |
| mscratch | machine scratch | 机器暂存寄存器 |
| msie | machine software interrupt enable | 机器软件中断使能位 |
| msip | machine software interrupt pending | 机器软件中断挂起位 |
| mstatus | machine status | 机器状态寄存器 |
| mtie | machine timer interrupt enable | 机器定时器中断使能位 |
| mtimer | machine timer | 机器定时器 |
| mtip | machine timer interrupt pending | 机器定时器中断挂起位 |
| mtval | machine trap value | 机器异常值寄存器 |
| mtvec | machine trap vector | 机器陷阱向量基址寄存器 |
| mvendorid | machine vendor ID | 机器厂商标识符 |
| mxl | machine XLEN | 机器模式下的 XLEN 编码 |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n（前缀） | negation / active-low | 取反 / 低有效（如 rst_n） |
| next（后缀） | — | 下一周期值（组合逻辑信号） |
| new（后缀） | — | 新计算值（中间变量） |
| nmret | not MRET | 非 MRET 指令（e_irq_nmret） |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o（后缀） | output | 输出端口 |
| offs | offset | 偏移量 |

---

## P

| 缩写 | 全称 | 含义 |
|------|------|------|
| pc | program counter | 程序计数器 |
| pnd | pending | 挂起（中断挂起且本地使能） |
| priv | privilege | 特权级 |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| r_req / w_req | read / write request | 读/写请求 |
| r_data / w_data | read / write data | 读/写数据 |
| r_exc / w_exc | read / write exception | 读/写访问异常 |
| rd | register destination | 目标寄存器 |
| reg | register | 寄存器 |
| req | request | 请求 |
| resp | response | 响应 |
| ro | read-only | 只读 |
| rst | reset | 复位 |
| rv32e | RISC-V 32-bit embedded | RISC-V 32 位嵌入式指令集（RVE） |
| rvc | RISC-V compressed | RISC-V 16 位压缩指令集扩展 |
| rvi | RISC-V integer | RISC-V 基础整数指令集（RV32I） |
| rvm | RISC-V multiply/divide | RISC-V 整数乘除法扩展 |
| rw | read / write | 读/写 |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| scr1 | Syntacore RISC-V Core 1 | 处理器核心名称 |
| sirq | software interrupt request | 软件中断请求 |
| soc | system-on-chip | 片上系统 |
| soi | start of interrupt | 中断开始（IPIC） |
| sva | SystemVerilog assertions | SystemVerilog 断言 |
| svh（后缀） | SystemVerilog header | 头文件扩展名 |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| tcm | tightly-coupled memory | 紧耦合存储器 |
| tdata | trigger data | 触发数据寄存器 |
| tdu | trigger debug unit | 触发调试单元 |
| tirq | timer interrupt request | 定时器中断请求 |
| trgt | target | 目标平台 |
| tselect | trigger select | 触发选择寄存器 |

---

## U

| 缩写 | 全称 | 含义 |
|------|------|------|
| upd | update | 更新使能信号 |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| v（后缀） | vector / value | 向量/值类型（如 pc_v、exu_cmd_s） |
| vect | vectored | 向量模式（MTVEC.mode=1） |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| wfi | wait for interrupt | 等待中断指令 |
| wr | write | 写使能（MTVEC.base 写位） |

---

## X

| 缩写 | 全称 | 含义 |
|------|------|------|
| xlen | — | 通用寄存器位宽（SCR1 为 32） |
| xprop | X propagation | X 态传播（仿真用） |
