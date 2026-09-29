# SCR1 AHB Top 顶层（AHB 总线）设计规格（Design Specification）

- 模块：`scr1_top_ahb`（`src/top/scr1_top_ahb.sv`，513 行）
- 条件：默认编译；TCM 使能时会定义 `SCR1_IMEM_ROUTER_EN`（`:13-15`）
- 职责：把 `scr1_core_top` 与 TCM、内存映射定时器、AHB 桥组装为完整的 AHB 系统顶层

---

## 1. 概述与职责

`scr1_top_ahb` 是面向 AHB 总线的处理器簇顶层，包含：

1. **复位同步**：Power-Up/Regular/CPU 复位同步，以及 TAPC 复位组合（`:170-217`）；
2. **核心例化**：`scr1_core_top`（`:222-282`）；
3. **存储子系统**：TCM（可选，`:289-311`）、内存映射定时器（`:318-337`）；
4. **路由**：指令路由器（`:344-375`）、数据路由器（`:391-454`）；
5. **AHB 桥**：指令 AHB（`:460-479`）、数据 AHB（`:485-509`）。

### 关键点

- 数据路由器端口分配：`port0`=AHB、`port1`=TCM、`port2`=timer（`:391-454`）；
- 指令路由器：有 TCM 时用路由器（port0=AHB、port1=TCM），否则直通 AHB（`:377-386`）；
- 无 TCM 时数据路由器 `port1` 接固定 `RDY_ER` 响应（`:426-435`）；
- `core_rdc_qlfy_o` 在本层悬空（`:231`，仅 `sys_*` 输出到外部）。

---

## 2. 端口（`:17-81`）

| 信号 | 方向 | 行号 | 条件 | 说明 |
|---|---|---|---|---|
| `pwrup_rst_n` / `rst_n` / `cpu_rst_n` | in | 19-21 | — | 三路复位 |
| `test_mode` / `test_rst_n` | in | 22/23 | — | DFT |
| `clk` / `rtc_clk` | in | 24/25 | — | 系统/实时时钟 |
| `sys_rst_n_o` / `sys_rdc_qlfy_o` | out | 27/31 | `SCR1_DBG_EN` | 系统复位输出 |
| `fuse_mhartid` | in | 35 | — | Hart ID |
| `fuse_idcode[31:0]` | in | 37 | `SCR1_DBG_EN` | IDCODE |
| `irq_lines[IRQ_LINES_NUM-1:0]` | in | 42 | `SCR1_IPIC_EN` | IPIC 中断线 |
| `ext_irq` | in | 44 | `!IPIC_EN` | 外部中断 |
| `soft_irq` | in | 46 | — | 软中断 |
| `trst_n/tck/tms/tdi` | in | 50-53 | `SCR1_DBG_EN` | JTAG |
| `tdo` / `tdo_en` | out | 54/55 | `SCR1_DBG_EN` | JTAG 输出 |
| `imem_hprot/hburst/hsize/htrans/hmastlock/haddr` | out | 59-64 | — | 指令 AHB 主机 |
| `imem_hready`/`hrdata`/`hresp` | in | 65-67 | — | 指令 AHB 从机 |
| `dmem_hprot/hburst/hsize/htrans/hmastlock/haddr/hwrite/hwdata` | out | 70-77 | — | 数据 AHB 主机 |
| `dmem_hready`/`hrdata`/`hresp` | in | 78-80 | — | 数据 AHB 从机 |

---

## 3. 复位逻辑（`:170-217`）

| 例化 | 类型 | rst_n_in | 行号 |
|---|---|---|---|
| `i_pwrup_rstn_reset_sync` | `scr1_reset_sync_cell` | `1'b1` | 174-183 |
| `i_rstn_reset_sync` | 同上 | `rst_n` | 186-195 |
| `i_cpu_rstn_reset_sync` | 同上 | `cpu_rst_n` | 198-207 |
| `i_tapc_rstn_and2_cell` | `scr1_reset_and2_cell` | `{trst_n, pwrup_rst_n}` | 211-216 |

各级 2 级同步（`SCR1_CLUSTER_TOP_RST_SYNC_STAGES_NUM=2`，`:86`）。

---

## 4. 核心与存储子系统

### 4.1 `i_core_top`（`:222-282`）

- 复位用同步后的 `pwrup_rst_n_sync`/`rst_n_sync`/`cpu_rst_n_sync`（`:224-226`）；
- `core_rst_n_local` 供 TCM/timer/router/AHB 桥使用（`:230`）；
- IRQ：IPIC 时 `irq_lines`，否则 `ext_irq`；`soft_irq`、定时器中断 `timer_irq`（`:244-250`）；
- `core_mtimer_val_i = timer_val`（`:253`）；
- IMEM/DMEM 接口对接路由器（`:265-281`）。

### 4.2 `i_tcm`（`:289-311`，`SCR1_TCM_EN`）

`SCR1_TCM_SIZE = SCR1_DMEM_AWIDTH'(~SCR1_TCM_ADDR_MASK + 1)`（`:290`）；
提供 imem/dmem 两套接口。

### 4.3 `i_timer`（`:318-337`）

内存映射定时器：`rtc_clk` 驱动计数，输出 `timer_val`/`timer_irq`。

---

## 5. 路由与 AHB 桥

### 5.1 指令路由器（`:340-386`）

- 有 `SCR1_IMEM_ROUTER_EN`：`scr1_imem_router`，参数 `SCR1_TCM_ADDR_MASK/PATTERN`
  （`:344-348`），port0=AHB（`:360-365`）、port1=TCM（`:366-374`）；
- 否则直接连线（`:379-384`）：`ahb_imem_* = core_imem_*`，ack/resp/rdata 反向直通。

### 5.2 数据路由器（`:391-454`）

参数：`SCR1_PORT1_ADDR_MASK/PATTERN`（TCM，无 TCM 时 mask=0/pattern=32'hFFFFFFFF，
`:393-399`）、`SCR1_PORT2`（timer，`:401-402`）。
端口：port0=AHB（`:446-453`）、port1=TCM 或固定 `RDY_ER`（`:416-435`）、port2=timer（`:437-444`）。

### 5.3 AHB 桥（`:457-509`）

`scr1_imem_ahb`（`:460-479`）与 `scr1_dmem_ahb`（`:485-509`）分别把内部存储器接口
转换为 AHB 主机事务。

---

## 6. 配置宏影响

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_TCM_EN` | 例化 TCM；派生 `SCR1_IMEM_ROUTER_EN` | `:13-15`/`:289`/`:340` |
| `SCR1_IMEM_ROUTER_EN` | 指令路由器 vs 直通 | `:340`/`:377` |
| `SCR1_IPIC_EN` | `irq_lines` vs `ext_irq` | `:41-45`/`:244-248` |
| `SCR1_DBG_EN` | `sys_*` 端口、JTAG、TAPC 复位 | `:26-56`/`:209-217` |

---

## 7. 附录

### 7.1 子模块例化索引

| 例化名 | 模块 | 行号 | 条件 |
|---|---|---|---|
| `i_core_top` | `scr1_core_top` | 222-282 | — |
| `i_tcm` | `scr1_tcm` | 289-311 | `SCR1_TCM_EN` |
| `i_timer` | `scr1_timer` | 318-337 | — |
| `i_imem_router` | `scr1_imem_router` | 344-375 | `SCR1_IMEM_ROUTER_EN` |
| `i_dmem_router` | `scr1_dmem_router` | 391-454 | — |
| `i_imem_ahb` | `scr1_imem_ahb` | 460-479 | — |
| `i_dmem_ahb` | `scr1_dmem_ahb` | 485-509 | — |

### 7.2 职责边界

- 本层只做系统级互连，不含流水线逻辑；
- 定时器与 TCM 均复位于 `core_rst_n_local`；
- AHB 主机接口对外，从机响应由外部 AHB 从设备提供。
