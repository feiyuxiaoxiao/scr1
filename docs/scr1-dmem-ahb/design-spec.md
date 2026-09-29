# SCR1 数据存储 AHB 桥设计规格（Design Specification）

- 模块：`scr1_dmem_ahb`（`src/top/scr1_dmem_ahb.sv`，480 行）
- 职责：把核心的数据存储器接口（读/写、字节/半字/字宽度）转换为 AHB 单次事务

---

## 1. 概述与职责

数据桥相比指令桥增加了写通路与宽度/字节偏移处理：

1. **宽度转换** `scr1_conv_mem2ahb_width`（`:82-103`）：BYTE→`HSIZE_8B`、HWORD→`HSIZE_16B`、WORD→`HSIZE_32B`；
2. **写数据对齐** `scr1_conv_mem2ahb_wdata`（`:105-156`）：按宽度与 `addr[1:0]` 把核心 wdata 放到 AHB 32 位正确字节；
3. **读数据回对齐** `scr1_conv_ahb2mem_rdata`（`:158-197`）：按 `hwidth`/`haddr` 从 `hrdata` 抽取到 `dmem_rdata`；
4. **请求 FIFO**：缓存 `hwrite/hwidth/haddr/hwdata`，支持 bypass 或深度 2（`:240-334`）；
5. **data FIFO**：锁存本拍地址宽度与写数据，供响应回对齐使用（`:385-411`）；
6. **FSM** 与响应处理同指令桥（`:338-383`/`:416-439`）。

---

## 2. 核心接口（`:223-235`）

| 信号 | 逻辑 | 行号 |
|---|---|---|
| `dmem_req_ack` | `~req_fifo_full` | 226 |
| `req_fifo_wr` | `~req_fifo_full & dmem_req` | 227 |
| `dmem_rdata` | `scr1_conv_ahb2mem_rdata(resp_fifo.hwidth, resp_fifo.haddr, resp_fifo.hrdata)` | 229 |
| `dmem_resp` | `resp_fifo_hready ? (hresp==OKAY ? RDY_OK : RDY_ER) : NOTRDY` | 231-235 |

---

## 3. 请求 FIFO（`:237-334`）

- 结构体含 `hwrite/hwidth/haddr/hwdata`（`:59-64`）；
- bypass 模式（`:240-267`）：单条 `req_fifo_r`，组合生成 `req_fifo_new`；
- 缓冲模式（`:269-333`）：深度 2，`2'b11` 仅当 `cnt=1` 时覆盖（`:299-306`）；
- `full=(cnt==2)`、`empty=~|cnt`（`:326-327`）。

---

## 4. FSM（`:335-383`）

与 `scr1_imem_ahb` 相同：`ADDR`/`DATA` 两态，`hresp!=OKAY` 时回 `ADDR`；
`SCR1_XPROP_EN` 下增加 `SCR1_FSM_ERR`（`:53-56`/`:357-361`）。

---

## 5. data FIFO 与响应（`:385-439`）

- `data_fifo` 锁存 `hwidth`、`haddr[1:0]`、`hwdata`（`:388-411`）；
- **IN_BP**：响应组合，`hrdata` 直通（`:416-421`）；
- **否则**：`resp_fifo` 寄存一拍（`:423-438`）。

---

## 6. AHB 接口（`:441-478`）

| 信号 | 值/逻辑 | 行号 |
|---|---|---|
| `hprot[DATA]` | `1'b1`（数据访问） | 444 |
| `hprot[PRV/BUF/CACHE]` | 全 0 | 445-447 |
| `hburst` | `SINGLE` | 449 |
| `hsize` | `req_fifo[0].hwidth` | 450 |
| `hmastlock` | `1'b0` | 451 |
| `htrans` | 同指令桥模式 | 453-474 |
| `haddr` | `req_fifo[0].haddr` | 476 |
| `hwrite` | `req_fifo[0].hwrite` | 477 |
| `hwdata` | `data_fifo.hwdata` | 478 |

---

## 7. 附录

### 7.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_DMEM_AHB_OUT_BP` | 请求 FIFO 单条 vs 深度 2 | `:42-45`/`:240`/`:269` |
| `SCR1_DMEM_AHB_IN_BP` | 响应组合 vs 寄存 | `:416`/`:422` |
| `SCR1_XPROP_EN` | 未用位/错误状态置 X | `:53-56`/`:112`/`:293-296`/`:358-361` |

### 7.2 职责边界

- 单 outstanding 数据事务，读写共用 FIFO；
- 宽度/偏移转换在桥内完成，核心侧仍是 `type_scr1_mem_width_e`；
- 本模块无 SVA。
