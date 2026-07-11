# SCR1 MPRF 代码缩写、缩略词对照表

> 对应源码: `src/core/pipeline/scr1_pipe_mprf.sv`

---

## A

| 缩写 | 全称 | 含义 |
|------|------|------|
| addr | address | 地址（如 `rs1_addr`、`rd_addr`） |
| arch | architecture | 架构（头文件名前缀 `scr1_arch_*`） |
| awidth | address width | 地址位宽（如 `SCR1_MPRF_AWIDTH`） |

---

## C

| 缩写 | 全称 | 含义 |
|------|------|------|
| clk | clock | 时钟信号 |

---

## E

| 缩写 | 全称 | 含义 |
|------|------|------|
| en | enable | 使能（条件编译后缀，如 `SCR1_MPRF_RST_EN`） |
| exu | execution unit | 执行单元 |
| ext | extension | RISC-V 扩展（如 `SCR1_RVE_EXT`、`SCR1_RVC_EXT`） |

---

## F

| 缩写 | 全称 | 含义 |
|------|------|------|
| ff | flip-flop | 触发器/寄存器（延迟一拍信号的后缀，如 `rs1_data_ff`、`rd_data_ff`） |
| fpga | field-programmable gate array | 现场可编程门阵列 |

---

## I

| 缩写 | 全称 | 含义 |
|------|------|------|
| i（后缀） | input | 输入端口后缀（如 `exu2mprf_rs1_addr_i`） |
| ifdef | if defined | 条件编译指令 |
| int | internal | 内部（存储器阵列内部名称后缀，如 `mprf_int`） |

---

## M

| 缩写 | 全称 | 含义 |
|------|------|------|
| mprf | multi-port register file | 多端口寄存器文件（本模块核心缩写） |
| m9k / m10k | — | Intel FPGA 嵌入式存储器块型号（9 Kbits / 10 Kbits） |

---

## N

| 缩写 | 全称 | 含义 |
|------|------|------|
| n（后缀） | negative / active-low | 低有效信号后缀（如 `rst_n`） |

---

## O

| 缩写 | 全称 | 含义 |
|------|------|------|
| o（后缀） | output | 输出端口后缀（如 `mprf2exu_rs1_data_o`） |

---

## R

| 缩写 | 全称 | 含义 |
|------|------|------|
| ram | random-access memory | 随机存取存储器（RAM 块实现方式） |
| rd | register destination | 目标寄存器（RISC-V 指令中 `[11:7]` 字段，回写目标） |
| req | request | 请求（如 `w_req`、`new_data_req`） |
| rs1 / rs2 | register source 1 / 2 | 源寄存器 1 / 2（RISC-V 指令中 `[19:15]` / `[24:20]` 字段） |
| rst | reset | 复位信号 |
| rve | RV32E | RISC-V 嵌入式基础整数指令集（16 寄存器） |
| rvi | RV32I | RISC-V 基础整数指令集（32 寄存器） |

---

## S

| 缩写 | 全称 | 含义 |
|------|------|------|
| scr1 | Syntacore RISC-V Core 1 | 处理器核心名称 |
| simul | simulation | 仿真（如 `SCR1_TRGT_SIMULATION`） |
| sva | SystemVerilog Assertions | SystemVerilog 断言 |
| svh | SystemVerilog Header | SystemVerilog 头文件后缀 |

---

## T

| 缩写 | 全称 | 含义 |
|------|------|------|
| trgt | target | 目标平台（如 `SCR1_TRGT_FPGA_INTEL`） |

---

## V

| 缩写 | 全称 | 含义 |
|------|------|------|
| v（后缀） | vector / value | 向量/值（数据类型后缀，如 `type_scr1_mprf_v`） |
| vd | valid | 有效标志（如 `rs1_addr_vd`、`wr_req_vd`） |

---

## W

| 缩写 | 全称 | 含义 |
|------|------|------|
| w（前缀） | write | 写操作（如 `w_req`） |
| wr | write | 写（如 `wr_req_vd`） |

---

## X

| 缩写 | 全称 | 含义 |
|------|------|------|
| x0-x31 | — | RISC-V 通用寄存器命名（x0=零寄存器, x1=ra, x2=sp 等） |
| xlen | — | RISC-V 通用寄存器位宽（SCR1 中为 32） |

---

## 接口命名规范补充

MPRF 模块的接口信号命名遵循与 IFU/IDU 一致的格式：

```
<源模块>2<目标模块>_<信号名>_<方向后缀>
```

| 信号全名 | 拆解 |
|----------|------|
| `exu2mprf_rs1_addr_i` | EXU → MPRF 的 rs1 地址，在 MPRF 是输入 |
| `mprf2exu_rs1_data_o` | MPRF → EXU 的 rs1 数据，在 MPRF 是输出 |
| `exu2mprf_w_req_i` | EXU → MPRF 的写请求，在 MPRF 是输入 |
| `exu2mprf_rd_addr_i` | EXU → MPRF 的 rd（目标寄存器）写地址，在 MPRF 是输入 |
| `exu2mprf_rd_data_i` | EXU → MPRF 的写数据，在 MPRF 是输入 |

### 内部信号命名约定

| 后缀/前缀 | 含义 | 示例 |
|-----------|------|------|
| `_ff` | 触发器输出（延迟一拍） | `rs1_data_ff`、`rd_data_ff`、`rs1_addr_vd_ff` |
| `_vd` | 有效标志 | `rs1_addr_vd`、`wr_req_vd` |
| `_req` | 请求/条件标志 | `rs1_new_data_req`、`read_new_data_req` |
| `int` | 内部阵列 | `mprf_int`、`mprf_int2` |
| `2` (数字) | "to" / 第二份副本 | `mprf2exu`（MPRF 到 EXU）、`mprf_int2`（RAM 副本 2） |
