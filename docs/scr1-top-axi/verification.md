# SCR1 AXI Top 顶层（AXI 总线）验证文档（Verification）

- 被测模块：`scr1_top_axi`（`src/top/scr1_top_axi.sv`，701 行）
- 验证目标：复位同步、AXI 复位选择、核心/存储/路由互连、AXI 桥与 reinit
- 本模块结构顶层 + 单条 reinit 时序逻辑，未包含 SVA

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-AXITOP-1 | `SCR1_TCM_EN` 定义 `SCR1_IMEM_ROUTER_EN` | `:12-14` | 编译宏检查 |
| FC-AXITOP-2 | 三级复位同步（`STAGES=2`） | `:244-278` | 波形观察 |
| FC-AXITOP-3 | TAPC 复位 `and2({trst_n, pwrup_rst_n})` | `:282-287` | 波形 `tapc_trst_n` |
| FC-AXITOP-4 | `axi_rst_n` 随 `SCR1_DBG_EN` 切换源 | `:290-294` | 两种配置检查 |
| FC-AXITOP-5 | `i_tcm` 尺寸派生 `~ADDR_MASK+1` | `:366-367` | 参数检查 |
| FC-AXITOP-6 | IMEM 直通连线（无路由器） | `:453-462` | 配置检查 |
| FC-AXITOP-7 | DMEM 路由器 port0/1/2 映射 | `:468-535` | 端口映射检查 |
| FC-AXITOP-8 | 无 TCM 时 port1 固定 `RDY_ER` | `:505-514` | 配置响应检查 |
| FC-AXITOP-9 | 指令桥 `core_width=WORD`、`core_wdata='0` | `:562-564` | 端口检查 |
| FC-AXITOP-10 | 桥 bypass 参数映射 | `:542-551`/`:620-629` | 参数检查 |
| FC-AXITOP-11 | `axi_reinit` 置位/清位条件 | `:696-699` | 波形观察 |
| FC-AXITOP-12 | `core_rdc_qlfy_o` 悬空 | `:308` | 端口检查 |

---

## 2. reinit 时序验证

`axi_reinit` 逻辑（`:696-699`）：

- 复位（`core_rst_n_local=0`）→ `axi_reinit=1`；
- 桥空闲（`axi_imem_idle & axi_dmem_idle`）→ `axi_reinit=0`；
- 使用 `negedge core_rst_n_local` 异步复位、`posedge clk` 同步更新。

验证：复位释放后 `axi_reinit` 保持 1，直至两个桥都上报 idle 才清 0。

---

## 3. 结构与接线检查

- `i_core_top` imem/dmem 接 `core_*`（`:342-358`）；
- 路由输出接 `axi_*`/`tcm_*`/`timer_*`（`:417-535`）；
- `timer_val` 回连 `core_mtimer_val_i`（`:330`/`:412`）；
- AXI 桥的 5 通道全部对接 `io_axi_*`（`:569-612`/`:647-690`）。

---

## 4. 覆盖建议

| 场景 | 说明 |
|---|---|
| 复位序列 | 各级复位同步与释放；`axi_rst_n` 选择 |
| AXI reinit | 从复位到 idle 的清 0 过程 |
| IMEM 取指 | 经路由/AXI 桥的读事务 |
| DMEM 读写 | 经路由/AXI 桥的读写事务 |
| 定时器/TCM | port2/port1 命中 |
| 中断输入 | IPIC 16 线或直连模式 |

---

## 5. 断言

本模块未包含 SVA（结构性顶层，唯一时序逻辑 reinit 由波形/形式检查覆盖）。

---

## 6. 验证结论

`scr1_top_axi` 通过子模块验证与 AXI 集成回归保障（`BUS=AXI`，`hello`）。
