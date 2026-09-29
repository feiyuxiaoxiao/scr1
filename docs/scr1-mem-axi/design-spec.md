# SCR1 存储器 AXI 桥设计规格（Design Specification）

- 模块：`scr1_mem_axi`（`src/top/scr1_mem_axi.sv`，363 行）
- 职责：把核心存储器接口（读/写、三宽度）转换为 AXI4 单拍事务，支持请求/响应 bypass

---

## 1. 概述与职责

AXI 桥用请求状态队列跟踪多笔在途事务：

1. **请求缓冲** `req_fifo[SCR1_REQ_BUF_SIZE]` 保存 `width/addr/wdata`（`:97-101`/`:146-152`）；
2. **状态队列** `req_status[]`：每项 `req_write/req_addr/req_data/req_resp` 四位状态（`:103-108`）；
3. **三指针**：`req_aval_ptr`（接收）、`req_proc_ptr`（处理）、`req_done_ptr`（完成）（`:115-117`）；
4. **bypass 通路**：`force_read`/`force_write` 在 `SCR1_AXI_REQ_BP` 且队列空时旁路缓冲（`:135-136`）；
5. **响应 bypass**：`SCR1_AXI_RESP_BP` 决定 `core_resp` 组合或寄存（`:300-313`）。

---

## 2. 参数与端口

### 2.1 参数（`:10-16`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `SCR1_REQ_BUF_SIZE` | 2 | 请求缓冲深度（2 的幂） |
| `SCR1_AXI_IDWIDTH` | 4 | AXI ID 宽度 |
| `SCR1_ADDR_WIDTH` | 32 | 地址宽度 |
| `SCR1_AXI_REQ_BP` | 1 | 请求 bypass |
| `SCR1_AXI_RESP_BP` | 1 | 响应 bypass |

### 2.2 端口（`:17-78`）

核心接口（`:22-31`）、AXI AW/W/B/AR/R 五通道（`:33-77`）。

---

## 3. 核心握手与空闲（`:126-144`）

| 信号 | 逻辑 | 行号 |
|---|---|---|
| `core_req_ack` | `~axi_reinit & ~req_status[aval].req_resp & core_resp!=RDY_ER` | 126-128 |
| `rready` | `~req_status[done].req_write` | 131 |
| `bready` | `req_status[done].req_write` | 132 |
| `core_idle` | 所有 `req_status[i].req_resp==0` | 139-144 |

`force_read`/`force_write`：bypass 且队列空时等于当前命令（`:135-136`）。

---

## 4. 请求状态队列（`:154-237`）

组合更新（`:158-194`）：

1. **新请求**：置 `req_resp=1`、`req_write`，按 bypass/ready 决定 `req_addr`/`req_data` 何时清零；
2. **地址阶段**（`awvalid&awready | arvalid&arready`）：清 `req_proc_ptr` 的 `req_addr`；
3. **数据阶段**（`wvalid&wready&wlast`）：清 `req_proc_ptr` 的 `req_data`；
4. **完成**（`bvalid&bready | rvalid&rready&rlast`）：清 `req_done_ptr` 的 `req_resp`。

寄存器在 `posedge clk` 按 `req_status_en` 更新（`:197-207`），三指针各自递增（`:209-237`）。

---

## 5. AXI 输出与数据适配（`:241-341`）

| 组 | 逻辑 | 行号 |
|---|---|---|
| `arvalid/awvalid/wvalid` | 由 `req_status` 位生成，或 bypass | 241-243 |
| `araddr/awaddr` | bypass 用 `core_addr`，否则 `req_fifo[proc].axi_addr` | 245-246 |
| `rcvd_resp` | B/R 响应→`RDY_OK`/`RDY_ER`，否则 `NOTRDY` | 248-260 |
| `wstrb` | 按宽度与地址产生字节选通 | 266-281 |
| `wdata` | `wdata << (8*addr[1:0])` 对齐 | 285-286 |
| `rcvd_rdata` | `rdata >> (8*addr[1:0])` 回对齐 | 290-297 |

AXI 常量：`awid=1`、`arid=0`、`awlen=arlen=0`、`awburst=arburst=1`（INCR）、
`awcache=arcache=2`、`wlast=1`，其余 `awlock/awprot/...` 为 0（`:317-341`）。

---

## 6. 响应 bypass（`:300-313`）

- `SCR1_AXI_RESP_BP==1`：`core_rdata` 与 `core_resp` 组合输出（`:302-303`）；
- 否则：`core_resp` 寄存、`core_rdata` 在 R 完成时寄存（`:305-311`）；
- `axi_reinit` 时 `core_resp=NOTRDY`（`:303`/`:307`）。

---

## 7. 断言（`:344-361`，`SCR1_TRGT_SIMULATION`）

4 条 SVA：输入握手信号无 X（`:350`）、`core_req` 时命令/宽度/地址无 X（`:352`）、
B 通道有效时 `{bid,bresp}` 无 X（`:355`）、R 通道有效时 `{rid,rresp}` 无 X（`:358`）。

---

## 8. 附录

### 8.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_AXI_REQ_BP` | 请求 bypass 通路 | `:135-136` |
| `SCR1_AXI_RESP_BP` | 响应组合 vs 寄存 | `:300-313` |
| `SCR1_TRGT_SIMULATION` | 使能 SVA | `:344-361` |

### 8.2 职责边界

- 仅产生单拍（`len=0`）AXI 事务；
- 多笔在途由状态队列与三指针管理；
- 输入侧宽度转换（`width2axsize`）与字节选通、数据移位在本模块内。
