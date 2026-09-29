# SCR1 指令存储 AHB 桥设计规格（Design Specification）

- 模块：`scr1_imem_ahb`（`src/top/scr1_imem_ahb.sv`，319 行）
- 职责：把核心的指令存储器接口（`type_scr1_mem_*`）转换为 AHB 单次读事务

---

## 1. 概述与职责

指令桥把核心发出的取指请求转换为 AHB NONSEQ 读传输，并缓存响应：

1. **请求 FIFO**：吸收主机地址，支持 1 深度 bypass（`SCR1_IMEM_AHB_OUT_BP`）或 2 深度 FIFO（`:37-40`/`:99-174`）；
2. **FSM**：`ADDR → DATA` 两态，错误时回 `ADDR`（`:179-222`）；
3. **响应**：`SCR1_IMEM_AHB_IN_BP` 直接组合转发，否则寄存一拍（`:227-246`）；
4. **AHB 常量**：`hprot=0`、`hburst=SINGLE`、`hsize=32B`、`hmastlock=0`（`:251-258`）。

---

## 2. 核心接口（`:85-94`）

| 信号 | 逻辑 | 行号 |
|---|---|---|
| `imem_req_ack` | `~req_fifo_full` | 85 |
| `req_fifo_wr` | `~req_fifo_full & imem_req` | 86 |
| `imem_rdata` | `resp_fifo.hrdata` | 88 |
| `imem_resp` | `resp_fifo_hready ? (hresp==OKAY ? RDY_OK : RDY_ER) : NOTRDY` | 90-94 |

---

## 3. 请求 FIFO（`:96-174`）

### 3.1 bypass 模式（`:99-120`）

单条 `req_fifo_r` 寄存器；`req_fifo_full` 表示已缓存且未被读走；
`req_fifo[0] = full ? req_fifo_r : imem_addr`（`:120`）。

### 3.2 缓冲模式（`:122-174`）

深度 2（`SCR1_FIFO_WIDTH=2`），`req_fifo_cnt` 记录占用：

- 组合更新（`:123-155`）：按 `{rd, wr}` 执行读写/换位；
- `2'b11` 仅在 `cnt=1` 时出现，直接用新地址覆盖（`:144-148`）；
- `full = (cnt == 2)`、`empty = ~|cnt`（`:166-167`）。

---

## 4. FSM（`:179-222`）

| 状态 | 行为 | 行号 |
|---|---|---|
| `ADDR` | `hready` 后：FIFO 空留 `ADDR`，否则转 `DATA` | 184-188 |
| `DATA` | `hready & OKAY`：FIFO 空回 `ADDR`，否则留 `DATA`；非 OKAY 回 `ADDR` | 189-197 |
| default | 转 `ERR`（`1'bx`） | 198-200 |

`req_fifo_rd`：`ADDR` 且 `hready` 读，`DATA` 且 `hready&OKAY` 读（`:205-222`）。

---

## 5. 响应处理（`:224-246`）

- **IN_BP**：`resp_fifo_hready = (fsm==DATA) ? hready : 0`，`hresp`/`hrdata` 直通（`:227-230`）；
- **否则**：`resp_fifo_hready` 与响应数据均寄存一拍（`:232-245`）。

---

## 6. AHB 接口（`:248-283`）

| 信号 | 值/逻辑 | 行号 |
|---|---|---|
| `hprot[3:0]` | 全 0（数据/特权/缓冲/缓存位） | 251-254 |
| `hburst` | `SCR1_HBURST_SINGLE` | 256 |
| `hsize` | `SCR1_HSIZE_32B` | 257 |
| `hmastlock` | `1'b0` | 258 |
| `htrans` | `ADDR` 有请求→NONSEQ；`DATA` 且 OKAY 有请求→NONSEQ；否则 IDLE/ERR | 260-281 |
| `haddr` | `req_fifo[0].haddr` | 283 |

---

## 7. 断言（`:285-317`，`SCR1_TRGT_SIMULATION`）

5 条 SVA：`imem_req` 无 X（`:291`）、`imem_req` 时 `imem_addr` 无 X（`:296`）、地址 4 字节对齐（`:301`）、
`hready` 无 X（`:307`）、`hresp` 无 X（`:312`）。

---

## 8. 附录

### 8.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_IMEM_AHB_OUT_BP` | 请求 FIFO 单条 vs 深度 2 | `:37-40`/`:99`/`:122` |
| `SCR1_IMEM_AHB_IN_BP` | 响应组合 vs 寄存 | `:227`/`:231` |
| `SCR1_XPROP_EN` | （本模块未使用） | — |
| `SCR1_TRGT_SIMULATION` | 使能 SVA | `:285-317` |

### 8.2 职责边界

- 仅实现单 outstanding 取指事务；
- 不含 cache/burst 逻辑，固定 SINGLE；
- AHB 从机侧握手完全由 FSM 与 FIFO 状态驱动。
