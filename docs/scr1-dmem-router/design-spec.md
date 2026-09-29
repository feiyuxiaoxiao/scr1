# SCR1 数据存储路由器设计规格（Design Specification）

- 模块：`scr1_dmem_router`（`src/top/scr1_dmem_router.sv`，278 行）
- 职责：按地址区间把核心数据访问路由到 port0（外部总线桥）、port1（TCM）、port2（定时器）

---

## 1. 概述与职责

数据路由器是三条端口的选择器（含写数据与宽度）：

1. **地址判定**（`:88-95`）：优先级 port1 → port2 → port0（默认）；
2. **FSM**：`ADDR`/`DATA` 两态，锁存 `port_sel_r`（`:97-130`）；
3. **响应回选**：`port_sel_r` 选择 `rdata`/`resp`（`:145-164`）；
4. **请求分发**：三条端口各按 `port_sel` 与状态生成 `req`、`cmd`、`width`、`addr`、`wdata`（`:176-264`）。

---

## 2. 参数与端口

### 2.1 参数（`:9-14`）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `SCR1_PORT1_ADDR_MASK` | `` `SCR1_DMEM_AWIDTH'hFFFF0000 `` | port1 掩码 |
| `SCR1_PORT1_ADDR_PATTERN` | `` `SCR1_DMEM_AWIDTH'h00010000 `` | port1 模式 |
| `SCR1_PORT2_ADDR_MASK` | `` `SCR1_DMEM_AWIDTH'hFFFF0000 `` | port2 掩码 |
| `SCR1_PORT2_ADDR_PATTERN` | `` `SCR1_DMEM_AWIDTH'h00020000 `` | port2 模式 |

### 2.2 端口（`:15-59`）

核心接口（`:20-28`）；port0/1/2 均含 `req_ack/req/cmd/width/addr/wdata/rdata/resp`
（`:30-58`）。

---

## 3. 端口选择（`:88-95`）

```systemverilog
port_sel = SCR1_SEL_PORT0;
if ((dmem_addr & PORT1_MASK) == PORT1_PATTERN)      port_sel = SCR1_SEL_PORT1;
else if ((dmem_addr & PORT2_MASK) == PORT2_PATTERN) port_sel = SCR1_SEL_PORT2;
```

注意：port1 判定优先于 port2；两者都不命中则落 port0。

---

## 4. FSM（`:97-164`）

- `ADDR`：`dmem_req & sel_req_ack` → `DATA`，锁存 `port_sel_r`（`:103-108`）；
- `DATA`：
  - `RDY_OK`：仍有 `req & ack` 则留 `DATA`（更新 `port_sel_r`），否则回 `ADDR`（`:111-118`）；
  - `RDY_ER`：回 `ADDR`（`:119-121`）；
- `sel_req_ack`：仅在 `ADDR`（或 `DATA & RDY_OK`）时按 `port_sel` 选 ack（`:132-143`）。

---

## 5. 端口请求与输出（`:166-264`）

| 组 | 逻辑 | 行号 |
|---|---|---|
| `dmem_req_ack/rdata/resp` | 直连 `sel_*` | 169-171 |
| portN_req | `dmem_req & (port_sel==SEL_PORTN)`（ADDR 或 DATA&OK） | 176-190/207-221/238-252 |
| portN_cmd/width/addr/wdata | `XPROP` 时未选中端口置 `CMD_ERROR`/`WIDTH_ERROR`/`'x`，否则直通 | 192-264 |

---

## 6. 断言（`:266-276`，`SCR1_TRGT_SIMULATION`）

`SCR1_SVA_DMEM_RT_XCHECK`：`dmem_req` 时 `{port_sel, dmem_cmd, dmem_width}` 无 X
（`:271-274`）。

---

## 7. 附录

### 7.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_XPROP_EN` | 未选中端口 cmd/width/addr/wdata 置错误/X | `:192`/`:223`/`:254` |
| `SCR1_TRGT_SIMULATION` | 使能 SVA | `:266-276` |

### 7.2 职责边界

- 三选一路由，不缓存；
- 单 outstanding：地址阶段决定端口，数据阶段用锁存选择回读；
- 默认参数下 port1=0x0001_0000、port2=0x0002_0000。
