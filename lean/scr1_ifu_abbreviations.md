# SCR1 IFU 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_ifu.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| ack | acknowledge | 确认（握手应答信号） |
| addr | address | 地址 |
| adr | address | 地址（缩写变体，如 `QUEUE_ADR_W`） |
| aw | address width | 地址位宽（如 `IMEM_AWIDTH`） |

---

## B

| 缩写 | 全称 | 含义 |
|------|------|------|
| bypass | — | 旁路，跳过队列直接将数据送到下游 |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| clk | clock | 时钟信号 |
| clkctrl | clock control | 时钟控制（时钟门控） |
| cnt | counter | 计数器 |
| cmd | command | 命令（如存储器读写命令） |
| curr | current | 当前的（如 `ifu_fsm_curr`） |

---

## D

| 缩写 | 全称 | 含义 |
|------|------|------|
| dbg | debug | 调试 |
| dec | decode | 译码（如 `SCR1_NO_DEC_STAGE`） |
| discard | — | 丢弃 |
| drc | discard response counter | 丢弃响应计数器 |
| dw | data width | 数据位宽（如 `IMEM_DWIDTH`） |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| en | enable | 使能（条件编译开关） |
| er | error | 错误（IMEM 响应类型） |
| exu | execution unit | 执行单元 |
| e（后缀） | enum | 枚举类型命名后缀（如 `type_scr1_ifu_fsm_e`） |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| ff | flip-flop | 触发器（寄存器后缀，如 `imem_addr_ff`） |
| fsm | finite state machine | 有限状态机 |
| fetch | — | 取指，从存储器读取指令 |

---

## H

| 缩写 | 全称 | 含义 |
|------|------|------|
| h（前缀） | half | 半字（16 位）如 `q_free_h_next` |
| hdu | hardware debug unit | 硬件调试单元 |
| hword | half word | 半字（16 位） |
| hi | high | 高位部分（如 `imem_rdata_hi` = bit[31:16]） |
| handshake | — | 握手（请求-应答协议） |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| idu | instruction decode unit | 指令译码单元 |
| ifu | instruction fetch unit | 指令取指单元 |
| imem | instruction memory | 指令存储器 |
| instr | instruction | 指令 |
| i/f | interface | 接口（代码注释中常见，如 "IMEM i/f"） |

---

## L

| 缩写 | 全称 | 含义 |
|------|------|------|
| lo | low | 低位部分（如 `imem_rdata_lo` = bit[15:0]） |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| mem | memory | 存储器 |
| mux | multiplexer | 多路选择器 |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n（前缀） | negation / negative | 取反/低有效（如 `rst_n` = reset active-low） |
| next | — | 下一时钟周期的值（组合逻辑信号后缀） |
| nv | not valid | 无效（如 `RVC_NV` 中的 NV = 该半字无效） |
| none | — | 无操作 |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o（后缀） | output | 输出端口命名后缀（如 `ifu2imem_req_o`） |
| ocpd | occupied | 已占用的（如 `q_ocpd_h` = 队列已占用半字数） |
| ok | — | IMEM 响应类型：成功 |
| ovf | overflow | 上溢/溢出 |

---

## P

| 缩写 | 全称 | 含义 |
|------|------|------|
| pbuf | program buffer | 程序缓冲区（调试接口用） |
| pc | program counter | 程序计数器 |
| pipe | pipeline | 流水线 |
| pnd | pending | 待处理的、未完成的（如 `imem_pnd_txns_cnt`） |
| ptr | pointer | 指针（如 `q_rptr`、`q_wptr`） |

---

## Q

| 缩写 | 全称 | 含义 |
|------|------|------|
| q | queue | 队列（指令队列相关信号前缀） |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| r（前缀） | read | 读操作（如 `q_rptr` = 读指针） |
| rdata | read data | 读取数据（如 `imem2ifu_rdata_i`） |
| rdy | ready | 就绪（如 `idu2ifu_rdy_i` = IDU 已就绪可接收） |
| rd | read | 读（命令或操作，如 `SCR1_IFU_QUEUE_RD_HWORD`） |
| req | request | 请求 |
| resp | response | 响应 |
| rst | reset | 复位信号 |
| rvi | RV32I (RISC-V 32-bit Integer) | RISC-V 32 位整数基本指令集 |
| rvc | RISC-V Compressed | RISC-V 16 位压缩指令集扩展 |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| scr1 | Syntacore RISC-V Core 1 | 处理器核心名称 |
| simul | simulation | 仿真 |
| stop | — | 停止 |
| sva | systemverilog assertions | SystemVerilog 断言 |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| txn | transaction | 事务（一次请求-响应交互） |
| trgt | target | 目标（如 `SCR1_TRGT_SIMULATION` = 以仿真为目标） |

---

## U

| 缩写 | 全称 | 含义 |
|------|------|------|
| upd | update | 更新使能（如 `q_rptr_upd`） |
| unaligned | — | 未对齐（如 `new_pc_unaligned_ff`） |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| vd | valid | 有效的（信号有效标志） |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| w（前缀） | write / word | 写操作（如 `q_wptr`）或字（32位，如 `q_free_w_next`） |
| wr | write | 写操作（如 `q_wr_size`） |
| word | — | 字（32 位） |

---

## X

| 缩写 | 全称 | 含义 |
|------|------|------|
| xlen | (RISC-V) XLEN | RISC-V 架构中通用寄存器的位宽（32 或 64） |
| xcheck | (unknown) X check | 未知态（X 态）检查 |

---

## 接口命名规范补充

代码中接口信号命名遵循统一模式：

```
<源模块>2<目标模块>_<信号名>_<方向后缀>
```

| 元素 | 含义 | 示例 |
|------|------|------|
| 源模块 | 信号发出方 | `exu`、`idu`、`hdu`、`imem`、`ifu`、`pipe` |
| 2 | "to" | — |
| 目标模块 | 信号接收方 | `ifu`、`idu`、`imem`、`hdu` |
| 信号名 | 信号功能描述 | `pc_new`、`rdy`、`req`、`instr` |
| `_i`（后缀） | input | 本模块视角的输入（对方视角的输出） |
| `_o`（后缀） | output | 本模块视角的输出（对方视角的输入） |

示例解读:

| 信号全名 | 拆解 |
|----------|------|
| `exu2ifu_pc_new_req_i` | EXU → IFU 的新 PC 请求，在本模块是输入 |
| `ifu2idu_instr_o` | IFU → IDU 的指令数据，在本模块是输出 |
| `imem2ifu_rdata_i` | IMEM → IFU 的读取数据，在本模块是输入 |
| `pipe2ifu_stop_fetch_i` | 流水线控制 → IFU 的停止取指，在本模块是输入 |
| `hdu2ifu_pbuf_fetch_i` | HDU → IFU 的 Program Buffer 取指请求，在本模块是输入 |
| `ifu2hdu_pbuf_rdy_o` | IFU → HDU 的 Program Buffer 就绪，在本模块是输出 |
