# SCR1 AHB Top 顶层（AHB 总线）验证文档（Verification）

- 被测模块：`scr1_top_ahb`（`src/top/scr1_top_ahb.sv`，513 行）
- 验证目标：复位同步、核心与存储子系统互连、路由选择、AHB 桥接线
- 本模块为纯结构性顶层，无自有 FSM/寄存器逻辑，未包含 SVA

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-AHBTOP-1 | `SCR1_TCM_EN` 定义 `SCR1_IMEM_ROUTER_EN` | `:13-15` | 编译宏检查 |
| FC-AHBTOP-2 | Power-Up 复位同步（`rst_n_in=1'b1`） | `:174-183` | 波形观察 `pwrup_rst_n_sync` |
| FC-AHBTOP-3 | Regular 复位同步 | `:186-195` | 波形 `rst_n_sync` |
| FC-AHBTOP-4 | CPU 复位同步 | `:198-207` | 波形 `cpu_rst_n_sync` |
| FC-AHBTOP-5 | TAPC 复位 `and2({trst_n, pwrup_rst_n})` | `:211-216` | 波形 `tapc_trst_n` |
| FC-AHBTOP-6 | `i_tcm` 尺寸由 `~ADDR_MASK+1` 派生 | `:289-290` | 层次检查参数 |
| FC-AHBTOP-7 | IMEM 直通连线（无路由器） | `:379-384` | 无 TCM 配置接线检查 |
| FC-AHBTOP-8 | DMEM 路由器 port0/1/2 映射 | `:391-454` | 端口映射检查 |
| FC-AHBTOP-9 | 无 TCM 时 port1 固定 `RDY_ER` | `:426-435` | 无 TCM 配置响应检查 |
| FC-AHBTOP-10 | IMEM/DMEM AHB 桥接线 | `:460-509` | 端口连接检查 |
| FC-AHBTOP-11 | IRQ 源随 `SCR1_IPIC_EN` 切换 | `:41-45`/`:244-248` | 两种配置检查 |
| FC-AHBTOP-12 | `core_rdc_qlfy_o` 悬空 | `:231` | 端口检查 |

---

## 2. 结构与接线检查

### 2.1 复位同步链

验证 `i_pwrup_rstn_reset_sync`、`i_rstn_reset_sync`、`i_cpu_rstn_reset_sync` 三级
`STAGES_AMOUNT=2`，且 TAPC 复位与 `trst_n`、`pwrup_rst_n` 的与关系（`:170-217`）。

### 2.2 存储器互连

- `i_core_top` 的 imem/dmem 端口接 `core_*_req_ack/req/cmd/...`（`:265-281`）；
- 路由输出接 `ahb_*`、`tcm_*`、`timer_*`（`:340-454`）；
- `timer_val` 回连 `core_mtimer_val_i`（`:253`/`:335`）。

### 2.3 配置组合

至少验证两种编译配置：

1. **TCM 关闭**：指令直通 AHB（`:377-386`），数据 port1 固定错误响应（`:426-435`）；
2. **TCM 开启**：指令/数据路由器均含 TCM 端口。

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| 复位序列 | 各级复位的同步与释放顺序 |
| IMEM 取指路径 | 经路由/AHB 桥取指响应 |
| DMEM 写路径 | 经路由/AHB 桥写数据 |
| 定时器访问 | port2 命中与 `timer_irq` 产生 |
| TCM 访问 | port1 命中（TCM 配置） |
| 中断输入 | IPIC 16 线与直连模式 |

---

## 4. 断言

本模块未包含 SVA（结构性顶层，行为由子模块覆盖）。

---

## 5. 验证结论

`scr1_top_ahb` 为纯互连顶层，正确性依赖子模块验证与集成回归
（`BUS=AHB`、`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`，`hello` PASS）。
