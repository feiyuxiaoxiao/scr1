# SCR1 LSU 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_lsu.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| ack | acknowledge | 确认（握手应答信号，如 `dmem2lsu_req_ack_i`） |
| addr | address | 地址 |
| aw | address width | 地址位宽（如 `SCR1_DMEM_AWIDTH`） |

---

## B

| 缩写 | 全称 | 含义 |
|------|------|------|
| bp | breakpoint | 断点 |
| brkm | breakpoint monitor | 断点监视器（TDU 子模块） |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| clk | clock | 时钟信号 |
| cmd | command | 命令（如 `exu2lsu_cmd_i`、`lsu2dmem_cmd_o`） |
| curr | current | 当前值（如 `lsu_fsm_curr`） |

---

## D

| 缩写 | 全称 | 含义 |
|------|------|------|
| dbrkpt | data breakpoint | 数据断点 |
| dmem | data memory | 数据存储器 |
| dmon | data (address) monitor | 数据（地址）监视（TDU 监视 LSU 数据地址流） |
| dw | data width | 数据位宽（如 `SCR1_DMEM_DWIDTH`） |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| e（后缀） | enum | 枚举类型后缀（如 `type_scr1_lsu_fsm_e`、`type_scr1_mem_cmd_e`） |
| en | enable | 使能（条件编译开关，如 `SCR1_TDU_EN`） |
| er | error | 错误（DMEM 响应类型 RDY_ER） |
| exc | exception | 异常 |
| exu | execution unit | 执行单元 |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| ff | flip-flop | 触发器（寄存器后缀，如 `lsu_cmd_ff`） |
| fsm | finite state machine | 有限状态机 |

---

## H

| 缩写 | 全称 | 含义 |
|------|------|------|
| hwbrk | hardware breakpoint | 硬件断点 |
| hword | half word | 半字（16 位） |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| i（后缀） | input | 输入端口后缀 |
| ibrkpt | instruction breakpoint | 指令断点 |

---

## L

| 缩写 | 全称 | 含义 |
|------|------|------|
| lb / lbu | load byte (unsigned) | 字节加载（无符号扩展） |
| ldata | load data | 加载数据 |
| lh / lhu | load halfword (unsigned) | 半字加载（无符号扩展） |
| lsu | load-store unit | 加载/存储单元 |
| lw | load word | 字加载 |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| mem | memory | 存储器（如 `dmem`、`scr1_memif.svh`） |
| mslgn | misalign | 未对齐（如 `dmem_addr_mslgn`） |
| mon | monitor | 监视器（TDU 对流水线的监视接口） |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n（前缀） | negation | 取反/低有效（如 `rst_n` = reset active-low） |
| next | — | 下一时钟周期的值（组合逻辑信号后缀，如 `lsu_fsm_next`） |
| none | — | 无操作（LSU 命令默认值） |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o（后缀） | output | 输出端口后缀（如 `lsu2exu_rdy_o`） |
| ok | — | DMEM 响应类型：成功（RDY_OK） |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| rdata | read data | 读取数据（如 `dmem2lsu_rdata_i`） |
| rd | read | 读命令（如 `SCR1_MEM_CMD_RD`） |
| rdy | ready | 就绪（如 `lsu2exu_rdy_o`） |
| req | request | 请求（如 `exu2lsu_req_i`） |
| resp | response | 响应（如 `dmem2lsu_resp_i`） |
| rst | reset | 复位 |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| sb / sh / sw | store byte/half/word | 字节/半字/字存储 |
| scr1 | Syntacore RISC-V Core 1 | 处理器核心名称 |
| sdata | store data | 存储数据 |
| sel | select | 选择（枚举类型后缀信号，如 `type_scr1_lsu_cmd_sel_e`） |
| sva | SystemVerilog assertions | SystemVerilog 断言 |
| svh | SystemVerilog header | 头文件后缀 |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| tdu | trigger debug unit | 触发调试单元 |
| trgt | target | 目标（如 `SCR1_TRGT_SIMULATION`） |

---

## U

| 缩写 | 全称 | 含义 |
|------|------|------|
| upd | update | 更新使能（如 `lsu_cmd_upd`） |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| vd | valid | 有效标志 |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| wdata | write data | 写入数据（如 `lsu2dmem_wdata_o`） |
| wdth | width | 数据宽度（如 `dmem_wdth_word`） |
| wr | write | 写命令（如 `SCR1_MEM_CMD_WR`） |
| word | — | 字（32 位） |

---

## X

| 缩写 | 全称 | 含义 |
|------|------|------|
| xcheck | X check | 未知态（X 态）检查 |
| xlen | — | RISC-V 通用寄存器位宽（SCR1 为 32） |

---

## 接口命名规范补充

代码中接口信号命名遵循统一模式：

```
<源模块>2<目标模块>_<信号名>_<方向后缀>
```

| 元素 | 含义 | 示例 |
|------|------|------|
| 源模块 | 信号发出方 | `exu`、`dmem`、`tdu`、`lsu` |
| 2 | "to" | — |
| 目标模块 | 信号接收方 | `lsu`、`exu`、`dmem`、`tdu` |
| 信号名 | 信号功能描述 | `req`、`cmd`、`addr`、`rdata`、`resp` |
| `_i`（后缀） | input | 本模块视角的输入 |
| `_o`（后缀） | output | 本模块视角的输出 |

示例解读:

| 信号全名 | 拆解 |
|----------|------|
| `exu2lsu_req_i` | EXU → LSU 的访存请求，在 LSU 模块是输入 |
| `exu2lsu_cmd_i` | EXU → LSU 的命令（LB/LH/...），在 LSU 模块是输入 |
| `exu2lsu_addr_i` | EXU → LSU 的访存地址，在 LSU 模块是输入 |
| `exu2lsu_sdata_i` | EXU → LSU 的存储数据，在 LSU 模块是输入 |
| `lsu2exu_rdy_o` | LSU → EXU 的就绪信号，在 LSU 模块是输出 |
| `lsu2exu_ldata_o` | LSU → EXU 的加载数据，在 LSU 模块是输出 |
| `lsu2exu_exc_o` | LSU → EXU 的异常请求，在 LSU 模块是输出 |
| `lsu2exu_exc_code_o` | LSU → EXU 的异常代码，在 LSU 模块是输出 |
| `lsu2dmem_req_o` | LSU → DMEM 的存储器请求，在 LSU 模块是输出 |
| `lsu2dmem_cmd_o` | LSU → DMEM 的读写命令，在 LSU 模块是输出 |
| `lsu2dmem_width_o` | LSU → DMEM 的数据宽度，在 LSU 模块是输出 |
| `lsu2dmem_addr_o` | LSU → DMEM 的存储器地址，在 LSU 模块是输出 |
| `lsu2dmem_wdata_o` | LSU → DMEM 的写入数据，在 LSU 模块是输出 |
| `dmem2lsu_req_ack_i` | DMEM → LSU 的请求确认，在 LSU 模块是输入 |
| `dmem2lsu_rdata_i` | DMEM → LSU 的读取数据，在 LSU 模块是输入 |
| `dmem2lsu_resp_i` | DMEM → LSU 的响应类型，在 LSU 模块是输入 |
| `lsu2tdu_dmon_o` | LSU → TDU 的数据地址监视信息，在 LSU 模块是输出 |
| `tdu2lsu_ibrkpt_exc_req_i` | TDU → LSU 的指令断点异常请求，在 LSU 模块是输入 |
| `tdu2lsu_dbrkpt_exc_req_i` | TDU → LSU 的数据断点异常请求，在 LSU 模块是输入 |
