# SCR1 指令存储路由器设计规格（Design Specification）

- 模块：`scr1_imem_router`（`src/top/scr1_imem_router.sv`，185 行）
- 职责：按地址区间把核心指令访问路由到 port0（外部总线桥）或 port1（TCM）

---

## 1. 概述与职责

指令路由器根据地址匹配结果在两条端口间选择：

1. **地址判定**：`port_sel = ((imem_addr & MASK) == PATTERN)`（`:64`）；
2. **FSM**：`ADDR`/`DATA` 两态，请求期间锁存 `port_sel_r`（`:66-99`）；
3. **响应回选**：用锁存的 `port_sel_r` 选择 `rdata`/`resp`（`:109-110`）；
4. **端口请求**：按 `port_sel` 分发给 port0/port1（`:122-136`/`:149-163`）。

---

## 2. 参数与端口

### 2.1 参数（`:9-12`）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `SCR1_ADDR_MASK` | `` `SCR1_IMEM_AWIDTH'hFFFF0000 `` | 地址掩码 |
| `SCR1_ADDR_PATTERN` | `` `SCR1_IMEM_AWIDTH'h00010000 `` | 匹配模式 |

### 2.2 端口（`:13-41`）

核心接口（`:18-24`）、PORT0（`:26-32`）、PORT1（`:34-40`），均为
`req_ack/req/cmd/addr/rdata/resp` 结构。

---

## 3. FSM（`:61-107`）

- `port_sel` 组合由当前地址判定（`:64`）；
- `ADDR`：`imem_req & sel_req_ack` → `DATA`，同时锁存 `port_sel_r <= port_sel`（`:72-77`）；
- `DATA`：
  - `sel_resp==RDY_OK`：仍有 `req & ack` 则留 `DATA`（更新 `port_sel_r`），否则回 `ADDR`（`:80-87`）；
  - `sel_resp==RDY_ER`：回 `ADDR`（`:88-90`）；
- `sel_req_ack`：仅在 `ADDR`（或 `DATA & RDY_OK`）时使能，按 `port_sel` 选端口 ack（`:101-107`）。

---

## 4. 端口选择与请求（`:109-171`）

| 组 | 逻辑 | 行号 |
|---|---|---|
| `sel_rdata` | `port_sel_r ? port1_rdata : port0_rdata` | 109 |
| `sel_resp` | `port_sel_r ? port1_resp : port0_resp` | 110 |
| `imem_req_ack/rdata/resp` | 直连 `sel_*` | 115-117 |
| port0_req | `imem_req & ~port_sel`（ADDR 或 DATA&OK） | 122-136 |
| port1_req | `imem_req & port_sel`（ADDR 或 DATA&OK） | 149-163 |

`SCR1_XPROP_EN` 时，未选中端口的 `cmd` 置 `SCR1_MEM_CMD_ERROR`、`addr` 置 `'x`
（`:138-144`/`:165-171`）；否则 `cmd`/`addr` 直通。

---

## 5. 断言（`:173-183`，`SCR1_TRGT_SIMULATION`）

`SCR1_SVA_IMEM_RT_XCHECK`：`imem_req` 时 `{port_sel, imem_cmd}` 无 X（`:178-181`）。

---

## 6. 附录

### 6.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_XPROP_EN` | 未选中端口 cmd/addr 置错误/X | `:138`/`:165` |
| `SCR1_TRGT_SIMULATION` | 使能 SVA | `:173-183` |

### 6.2 职责边界

- 仅做静态地址区间选择，不缓存；
- 单 outstanding：`ADDR→DATA` 完成一次事务；
- port0 通常接 AHB/AXI 桥，port1 接 TCM。
