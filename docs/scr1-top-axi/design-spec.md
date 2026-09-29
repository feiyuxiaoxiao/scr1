# SCR1 AXI Top 顶层（AXI 总线）设计规格（Design Specification）

- 模块：`scr1_top_axi`（`src/top/scr1_top_axi.sv`，701 行）
- 条件：默认编译；TCM 使能时定义 `SCR1_IMEM_ROUTER_EN`（`:12-14`）
- 职责：把 `scr1_core_top` 与 TCM、内存映射定时器、AXI 桥组装为完整 AXI 系统顶层

---

## 1. 概述与职责

结构与 `scr1_top_ahb` 对应，差异在于外部总线为 AXI 且增加了 AXI 桥的 reinit 逻辑：

1. **复位同步**：Power-Up/Regular/CPU 复位同步，TAPC 复位组合（`:244-288`）；
2. **AXI 复位选择**：`axi_rst_n` 由 `SCR1_DBG_EN` 选择 `sys_rst_n_o` 或 `rst_n_sync`（`:290-294`）；
3. **核心例化**：`scr1_core_top`（`:299-359`）；
4. **存储子系统**：TCM（可选，`:366-388`）、定时器（`:395-414`）；
5. **路由**：指令路由器（`:421-451`）、数据路由器（`:468-535`）；
6. **AXI 桥**：指令 `scr1_mem_axi`（`:541-613`）、数据 `scr1_mem_axi`（`:619-691`）；
7. **AXI reinit**：`:696-699`。

### 关键点

- 指令 AXI 桥 `core_width` 固定 `SCR1_MEM_WIDTH_WORD`，`core_wdata` 接 `'0`（`:562-564`）；
- 指令/数据桥的 `SCR1_AXI_REQ_BP`/`SCR1_AXI_RESP_BP` 由 `SCR1_*MEM_AXI_REQ_BP`/`RESP_BP` 决定（`:541-551`/`:619-629`）；
- `axi_reinit` 在 `core_rst_n_local` 下降沿置 1，两个桥都 idle 时清 0（`:696-699`）；
- AXI 桥复位用 `axi_rst_n`，其余存储/路由用 `core_rst_n_local`。

---

## 2. 端口（`:16-148`）

### 2.1 控制/复位/熔丝/中断/JTAG（`:17-55`）

与 `scr1_top_ahb` 完全一致：`pwrup_rst_n`/`rst_n`/`cpu_rst_n`（`:18-20`）、`test_mode`/`test_rst_n`
（`:21-22`）、`clk`/`rtc_clk`（`:23-24`）、`sys_rst_n_o`/`sys_rdc_qlfy_o`（`:26-30`，`SCR1_DBG_EN`）、
`fuse_mhartid`/`fuse_idcode`（`:34-37`）、`irq_lines` 或 `ext_irq`（`:40-44`）、`soft_irq`（`:45`）、
JTAG（`:49-55`，`SCR1_DBG_EN`）。

### 2.2 指令 AXI（`:57-101`）

完整 5 通道：AW（`:58-70`）、W（`:71-76`）、B（`:77-81`）、AR（`:82-94`）、R（`:95-101`）。

### 2.3 数据 AXI（`:103-147`）

同样完整 5 通道：AW（`:104-116`）、W（`:117-122`）、B（`:123-127`）、AR（`:128-140`）、R（`:141-147`）。

---

## 3. AXI 复位与 reinit

### 3.1 `axi_rst_n`（`:290-294`）

```systemverilog
`ifdef SCR1_DBG_EN
assign axi_rst_n = sys_rst_n_o;
`else
assign axi_rst_n = rst_n_sync;
`endif
```

### 3.2 reinit FSM（`:696-699`）

异步复位（`negedge core_rst_n_local`）置 `axi_reinit=1`；当 `axi_imem_idle & axi_dmem_idle`
时清 0，通知 AXI 桥可重新初始化。

---

## 4. 存储与路由（与 AHB 顶层一致）

| 组件 | 行号 | 复位 |
|---|---|---|
| `i_tcm`（`SCR1_TCM_EN`） | 366-388 | `core_rst_n_local` |
| `i_timer` | 395-414 | `core_rst_n_local` |
| `i_imem_router`（`SCR1_IMEM_ROUTER_EN`） | 421-451 | `core_rst_n_local` |
| 直通 assign | 453-462 | — |
| `i_dmem_router` | 468-535 | `core_rst_n_local` |

数据路由器端口：port0=AXI（`:526-534`）、port1=TCM 或固定 `RDY_ER`（`:495-514`）、port2=timer（`:516-524`）。

---

## 5. AXI 桥（`:538-691`）

| 例化 | 参数 | 行号 |
|---|---|---|
| `i_imem_axi` | `SCR1_AXI_REQ_BP`←`SCR1_IMEM_AXI_REQ_BP`；`SCR1_AXI_RESP_BP`←`SCR1_IMEM_AXI_RESP_BP` | 541-613 |
| `i_dmem_axi` | `SCR1_AXI_REQ_BP`←`SCR1_DMEM_AXI_REQ_BP`；`SCR1_AXI_RESP_BP`←`SCR1_DMEM_AXI_RESP_BP` | 619-691 |

两桥均接 `axi_rst_n` 与 `axi_reinit`，并输出 `core_idle`（`:558`/`:636`）供 reinit FSM 使用。

---

## 6. 配置宏影响

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_TCM_EN` | 例化 TCM；派生 `SCR1_IMEM_ROUTER_EN` | `:12-14`/`:366`/`:417` |
| `SCR1_IMEM_ROUTER_EN` | 指令路由器 vs 直通 | `:417`/`:453` |
| `SCR1_IPIC_EN` | `irq_lines` vs `ext_irq` | `:40-44`/`:321-325` |
| `SCR1_DBG_EN` | `sys_*`、JTAG、TAPC 复位、`axi_rst_n` 源 | `:25-55`/`:290-294` |
| `SCR1_IMEM_AXI_REQ_BP`/`RESP_BP` | 指令桥 bypass | `:542-551` |
| `SCR1_DMEM_AXI_REQ_BP`/`RESP_BP` | 数据桥 bypass | `:620-629` |

---

## 7. 附录

### 7.1 子模块例化索引

| 例化名 | 模块 | 行号 | 条件 |
|---|---|---|---|
| `i_core_top` | `scr1_core_top` | 299-359 | — |
| `i_tcm` | `scr1_tcm` | 366-388 | `SCR1_TCM_EN` |
| `i_timer` | `scr1_timer` | 395-414 | — |
| `i_imem_router` | `scr1_imem_router` | 421-451 | `SCR1_IMEM_ROUTER_EN` |
| `i_dmem_router` | `scr1_dmem_router` | 468-535 | — |
| `i_imem_axi` | `scr1_mem_axi` | 541-613 | — |
| `i_dmem_axi` | `scr1_mem_axi` | 619-691 | — |

### 7.2 职责边界

- 本层仅系统互连与 AXI reinit 序列，无流水线逻辑；
- AXI 桥对外，其余子系统走内部存储器接口；
- 无 SVA。
