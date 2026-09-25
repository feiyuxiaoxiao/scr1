# SCR1 MPRF 多端口寄存器堆设计规格（Design Specification）

- 模块：`scr1_pipe_mprf`（`src/core/pipeline/scr1_pipe_mprf.sv`，167 行）
- 层次：`scr1_pipe_top` 的子模块，例化于 `scr1_pipe_top.sv:477`
- 职责：保存 31/15 个通用寄存器（x0 硬连接 0），支持 2 读 1 写
- 两种实现：分布式逻辑（默认）/ RAM（Intel FPGA）

---

## 1. 概述与职责

MPRF 是整数寄存器堆：

1. **两读端口**：`rs1`/`rs2`，EXU 组合读取操作数；
2. **一写端口**：EXU 写回 `rd`（load/ALU/CSR 结果）；
3. **x0 恒 0**：不存 x0、不写 x0、读 x0 得 0；
4. **写优先（write-first）语义**：同拍写读同地址时，读口返回新写数据
   （RV 单发射下 EXU 需回读当拍写回值）。

寄存器数量随 RVE：32（RV32I）或 16（RV32E）。

---

## 2. 端口与接口

见源码 `:9-24`。

| 信号 | 方向 | 位宽 | 行号 | 说明 |
|---|---|---|---|---|
| `rst_n` | in | 1 | 12 | 复位（仅 `SCR1_MPRF_RST_EN`） |
| `clk` | in | 1 | 14 | 时钟 |
| `exu2mprf_rs1_addr_i` | in | `SCR1_MPRF_AWIDTH` | 17 | rs1 读地址 |
| `mprf2exu_rs1_data_o` | out | XLEN | 18 | rs1 读数据 |
| `exu2mprf_rs2_addr_i` | in | `SCR1_MPRF_AWIDTH` | 19 | rs2 读地址 |
| `mprf2exu_rs2_data_o` | out | XLEN | 20 | rs2 读数据 |
| `exu2mprf_w_req_i` | in | 1 | 21 | 写请求 |
| `exu2mprf_rd_addr_i` | in | `SCR1_MPRF_AWIDTH` | 22 | rd 写地址 |
| `exu2mprf_rd_data_i` | in | XLEN | 23 | rd 写数据 |

> `rst_n` 端口本身也受 `SCR1_MPRF_RST_EN` 条件编译（`:11-13`）。

### 2.1 存储器声明（`:50-64`）

- **分布式逻辑**（默认）：`type_scr1_mprf_v [1:SCR1_MPRF_SIZE-1] mprf_int`；
- **RAM**（Intel FPGA）：两块存储 `mprf_int`/`mprf_int2`，各带厂商 `ramstyle` 属性
  （Max10→M9K、ArriaV→M10K），范围均为 `[1:SCR1_MPRF_SIZE-1]`。

索引从 1 开始（排除 x0），地址 0 由外部逻辑映射为 0。

---

## 3. 控制逻辑（`:66-88`）

```systemverilog
assign rs1_addr_vd = |exu2mprf_rs1_addr_i;      // :71  x0 → 读 0
assign rs2_addr_vd = |exu2mprf_rs2_addr_i;      // :72
assign wr_req_vd   = exu2mprf_w_req_i & |exu2mprf_rd_addr_i;  // :74  x0 → 不写
```

- `rs*_addr_vd`：地址非零才真正读，否则读口返回 0（实现 x0）；
- `wr_req_vd`：写请求且 rd≠x0（x0 不可写）。

### 3.1 RAM 专用的冲突检测（`:77-88`）

```systemverilog
rs1_new_data_req  = wr_req_vd & (rs1_addr == rd_addr);   // :78
rs2_new_data_req  = wr_req_vd & (rs2_addr == rd_addr);   // :79
read_new_data_req = rs1_new_data_req | rs2_new_data_req; // :80

always_ff @(posedge clk) begin                            // :82-87
    rs1_addr_vd_ff      <= rs1_addr_vd;
    rs2_addr_vd_ff      <= rs2_addr_vd;
    rs1_new_data_req_ff <= rs1_new_data_req;
    rs2_new_data_req_ff <= rs2_new_data_req;
end
```

因为 RAM 读是同步的（一拍延迟），需把地址有效与冲突标志打一拍，
在数据返回拍选择正确的来源。

---

## 4. 分布式逻辑实现（`:127-152`）

```systemverilog
assign mprf2exu_rs1_data_o = rs1_addr_vd ? mprf_int[exu2mprf_rs1_addr_i] : '0;  // :133
assign mprf2exu_rs2_data_o = rs2_addr_vd ? mprf_int[exu2mprf_rs2_addr_i] : '0;  // :134
```

- **异步读**：组合读，无延迟；
- **写**（`:137-151`）：
  - `SCR1_MPRF_RST_EN`：同步复位到 `'{default:'0}`，否则仅 `wr_req_vd` 写入；
  - 无复位版本（`~SCR1_MPRF_RST_EN`）只有 `if (wr_req_vd)` 分支。

### 4.1 写读时序

分布式实现中，组合读直接取当前存储器内容；写入为非阻塞赋值（`<=`），
同拍该地址读到的是旧值。模块读地址恒为**当前 EXU 队列指令**的 `rs1/rs2`
（EXU `:862-863`），而下一指令仅在当前指令退休时才被锁存进队列
（EXU `:318`），其读地址在下一拍才出现，故写/读天然不在同拍冲突。

> RAM 实现不同：为匹配同步读延迟，EXU 会在 `exu_queue_en` 时**提前呈现下一指令
> 的读地址**（EXU `:859-860`），于是当前指令的写与下一指令的读可能同拍，
> 需要模块内显式写优先旁路（`rd_data_ff`，§5.1）。

---

## 5. RAM 实现（`:90-126`）

```systemverilog
assign mprf2exu_rs1_data_o = rs1_new_data_req_ff ? rd_data_ff
                           : rs1_addr_vd_ff       ? rs1_data_ff : '0;  // :100-102
assign mprf2exu_rs2_data_o = rs2_new_data_req_ff ? rd_data_ff
                           : rs2_addr_vd_ff       ? rs2_data_ff : '0;  // :104-106

always_ff @(posedge clk) if (read_new_data_req) rd_data_ff <= exu2mprf_rd_data_i; // :108-112
always_ff @(posedge clk) begin                                     // :114-118
    rs1_data_ff <= mprf_int [exu2mprf_rs1_addr_i];
    rs2_data_ff <= mprf_int2[exu2mprf_rs2_addr_i];
end
always_ff @(posedge clk) if (wr_req_vd) begin                      // :121-126
    mprf_int [exu2mprf_rd_addr_i] <= exu2mprf_rd_data_i;
    mprf_int2[exu2mprf_rd_addr_i] <= exu2mprf_rd_data_i;
end
```

### 5.1 两存储器 + 外部写优先

- **为何两块**：同一拍可能同时发生 2 次读 + 1 次写，单口 RAM 无法满足，
  故用两块简单双口存储器，写时同时写两块；
- **同步读**：`rs*_data_ff` 一拍后有效；
- **写优先旁路**：若上一拍发生写读冲突（`rs*_new_data_req_ff`），
  输出改用 `rd_data_ff`（上一拍的写数据），实现 `WRITE_FIRST` 行为。

### 5.2 约束

`SCR1_MPRF_RAM` 下必须关闭 `SCR1_NO_EXE_STAGE` 与 `SCR1_MPRF_RST_EN`
（`scr1_arch_description.svh:194-198`）：
- RAM 读有一拍延迟，需要 EXU 队列级提供稳定的读地址/写数据；
- RAM 无复位端口（`rst_n` 不出现），故不能启用复位。

---

## 6. 与 EXU 的接口（边界）

EXU 侧（`scr1_pipe_exu.sv`）：

```systemverilog
exu2mprf_rs1_addr_o = mprf_rs1_req ? mprf_rs1_addr : '0;   // :866
exu2mprf_rs2_addr_o = mprf_rs2_req ? mprf_rs2_addr : '0;   // :867

exu2mprf_w_req_o = (rd_wb_sel != NONE) & exu_queue_vd & ~exu_exc_req
                 & (DBG ? ~no_commit : 1)
                 & ((rd_wb_sel == CSR) ? csr_access_init : exu_rdy);  // :872-876
```

- EXU 用 `use_rs1/use_rs2` 门控读地址，未用则给 0（不读、不耗电）；
- 写请求在**指令退休且非异常**时发出，写数据由 EXU 写回 mux 选择
  （`ialu_main_res`/SUM2/IMM/INC_PC/LSU/CSR，EXU `:881-890`）。

在 RAM 模式下，EXU 还额外寄存 `mprf_rs1_addr`/`mprf_rs2_addr` 与 `use_*_ff`，
且在 `exu_queue_en` 时改为呈现下一指令地址，以匹配同步读延迟
（EXU `:839-864`）。

---

## 7. 配置宏影响

| 宏 | 位置 | 影响 |
|---|---|---|
| `SCR1_RVE_EXT` | `scr1_arch_types.svh:15-21` | 决定 `SCR1_MPRF_AWIDTH`（4/5）与 `SCR1_MPRF_SIZE`（16/32） |
| `SCR1_MPRF_RST_EN` | `:11-13`、`:137-144`、`:158-163` | 是否带复位端口与同步清零 |
| `SCR1_MPRF_RAM` | `:35-64`、`:90-126` | 选用 RAM 实现（默认分布式） |
| `SCR1_TRGT_FPGA_INTEL` | `scr1_arch_description.svh:190-192` | 定义 `SCR1_MPRF_RAM`（间接） |
| `SCR1_TRGT_FPGA_INTEL_MAX10/ARRIAV` | `:52-57` | 指定 `ramstyle` 原语 |
| `SCR1_TRGT_SIMULATION` | `:154-165` | 编译断言 |

---

## 8. 假设与边界

1. x0 由“数组不含索引 0 + 地址有效逻辑”共同实现，读得 0、写被忽略；
2. 分布式实现无同拍写读冲突（下一指令读地址晚一拍出现）；
   RAM 实现自带写优先旁路以覆盖提前读地址的情形；
3. RAM 实现无复位，寄存器初值为 X，但所有读地址在使用前必被写覆盖
   （除 x0，而 x0 由逻辑返回 0）；
4. 读写均为同步写；分布式读为异步，RAM 读为同步（一拍延迟）。

---

## 9. 内建断言（`:154-165`）

| 断言 | 行号 | 检查 | 条件 |
|---|---|---|---|
| `SCR1_SVA_MPRF_WRITEX` | 159-162 | 写请求时地址与（非 x0 的）写数据无 X | 仅 `SCR1_MPRF_RST_EN` |

> 断言用 `|rd_addr ? rd_data : 0` 屏蔽 x0 的无关数据，避免误报。

---

## 附录 A：参数汇总

| 参数 | 值 | 来源 |
|---|---|---|
| `SCR1_MPRF_AWIDTH` | 4（RVE）/ 5 | `scr1_arch_types.svh:16/19` |
| `SCR1_MPRF_SIZE` | 16（RVE）/ 32 | `scr1_arch_types.svh:17/20` |
| `type_scr1_mprf_v` | `logic [XLEN-1:0]` | `:23` |

## 附录 B：读路径对比

| 特性 | 分布式逻辑 | RAM |
|---|---|---|
| 读延迟 | 0（组合） | 1 拍（同步） |
| 存储器数 | 1 | 2 |
| 写优先 | 无同拍冲突 | 模块内 `rd_data_ff` |
| 复位 | 可选 | 无 |
| `NO_EXE_STAGE` | 可有可无 | 必须关闭 |

## 附录 C：信号 → 消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `mprf2exu_rs1/rs2_data_o` | EXU IALU/LSU/CSR 操作数 | 读操作数 |
| `exu2mprf_rs*_addr_i` | MPRF 读地址 | EXU 提供 |
| `exu2mprf_w_req_i`/`rd_*` | MPRF 写口 | EXU 写回 |
| `mprf_int`（层次引用） | tracelog | 打印寄存器（`scr1_pipe_top.sv:788`） |
