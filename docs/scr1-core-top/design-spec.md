# SCR1 Core Top 核心顶层 设计规格（Design Specification）

- 模块：`scr1_core_top`（`src/core/scr1_core_top.sv`，521 行）
- 层次：`scr1_top`/`scr1_top_ahb`/`scr1_top_axi` 例化本模块；下辖 `scr1_pipe_top`（`:276`）、
  `scr1_scu`（`:193`）、`scr1_tapc`（`:362`）、`scr1_tapc_synchronizer`（`:385`）、
  `scr1_dmi`（`:417`）、`scr1_dm`（`:457`）、`scr1_clk_ctrl`（`:503`）
- 职责：把“复位/时钟/调试子系统”与“CPU 流水线”装配成完整 core，向上暴露统一的
  复位、内存接口、IRQ 与（可选的）JTAG 调试接口

---

## 1. 概述与职责

`scr1_core_top` 是 one-hart 的集成层，做四件事：

1. **复位管理**：DBG 使能时由 `scr1_scu` 生成 `core_rst_n`/`sys_rst_n`/`dm_rst_n`/`hdu_rst_n`
   与各 RDC qualifier；非 DBG 时用简化的同步链与 `scr1_reset_qlfy_adapter_cell_sync`；
2. **流水线装配**：例化 `scr1_pipe_top`，对接 IMEM/DMEM、IRQ、timer、fuse；
3. **调试子系统装配**（`SCR1_DBG_EN`）：TAPC → TAPC synchronizer → DMI → DM → HDU/PBUF；
4. **时钟门控**（`SCR1_CLKCTRL_EN`）：由 `scr1_clk_ctrl` 生成流水线睡眠/唤醒时钟。

### 设计要点

- **两条复位路径**：DBG 与 non-DBG 结构不同，端口集合也不同
  （`sys_rst_n_o`/`sys_rdc_qlfy_o` 仅 DBG 存在，`:30-33`）；
- **RDC qualifier 掩码**：DM 与 pipeline 之间所有响应/事件/状态信号在
  `hdu2dm_rdc_qlfy` 无效时被清零（`:447-455`），避免复位域穿越期间的伪值；
- **TDO 选择**：`tapc_ch_tdo = (scu_tdo & scu_sel) | (dmi_tdo & dmi_sel)`（`:414-415`）。

---

## 2. 端口（`:20-80`）

| 信号 | 方向 | 行号 | 条件 | 说明 |
|---|---|---|---|---|
| `pwrup_rst_n` | in | 22 | — | Power-Up 复位 |
| `rst_n` | in | 23 | — | 常规复位 |
| `cpu_rst_n` | in | 24 | — | CPU 复位 |
| `test_mode` | in | 25 | — | DFT 测试模式 |
| `test_rst_n` | in | 26 | — | DFT 测试复位 |
| `clk` | in | 27 | — | core 时钟 |
| `core_rst_n_o` | out | 28 | — | core 复位输出 |
| `core_rdc_qlfy_o` | out | 29 | — | core RDC qualifier |
| `sys_rst_n_o` | out | 31 | `SCR1_DBG_EN` | 系统复位 |
| `sys_rdc_qlfy_o` | out | 32 | `SCR1_DBG_EN` | 系统 RDC qualifier |
| `core_fuse_mhartid_i[XLEN-1:0]` | in | 36 | — | MHARTID fuse |
| `tapc_fuse_idcode_i[31:0]` | in | 38 | `SCR1_DBG_EN` | IDCODE fuse |
| `core_irq_lines_i[IRQ_LINES_NUM-1:0]` | in | 43 | `SCR1_IPIC_EN` | IPIC 中断线 |
| `core_irq_ext_i` | in | 45 | `!IPIC_EN` | 外部中断 |
| `core_irq_soft_i` | in | 47 | — | 软中断 |
| `core_irq_mtimer_i` | in | 48 | — | 定时器中断 |
| `core_mtimer_val_i[63:0]` | in | 51 | — | 外部定时器值 |
| `tapc_trst_n/tck/tms/tdi` | in | 55-58 | `SCR1_DBG_EN` | JTAG |
| `tapc_tdo` / `tapc_tdo_en` | out | 59/60 | `SCR1_DBG_EN` | JTAG 输出 |
| `imem2core_req_ack_i` | in | 64 | — | IMEM ack |
| `core2imem_req_o` | out | 65 | — | IMEM 请求 |
| `core2imem_cmd_o`（`type_scr1_mem_cmd_e`） | out | 66 | — | IMEM 命令 |
| `core2imem_addr_o[IMEM_AWIDTH-1:0]` | out | 67 | — | IMEM 地址 |
| `imem2core_rdata_i[IMEM_DWIDTH-1:0]` | in | 68 | — | IMEM 读数据 |
| `imem2core_resp_i`（`type_scr1_mem_resp_e`） | in | 69 | — | IMEM 响应 |
| `dmem2core_req_ack_i` | in | 72 | — | DMEM ack |
| `core2dmem_req_o` | out | 73 | — | DMEM 请求 |
| `core2dmem_cmd_o/width_o` | out | 74/75 | — | DMEM 命令/宽度 |
| `core2dmem_addr_o[DMEM_AWIDTH-1:0]` | out | 76 | — | DMEM 地址 |
| `core2dmem_wdata_o[DMEM_DWIDTH-1:0]` | out | 77 | — | DMEM 写数据 |
| `dmem2core_rdata_i[DMEM_DWIDTH-1:0]` | in | 78 | — | DMEM 读数据 |
| `dmem2core_resp_i`（`type_scr1_mem_resp_e`） | in | 79 | — | DMEM 响应 |

---

## 3. 局部参数与信号（`:85-186`）

- `SCR1_CORE_TOP_RST_SYNC_STAGES_NUM = 2`（`:85`）——复位状态同步级数。

信号分组：

| 组 | 行号 | 说明 |
|---|---|---|
| Reset（non-DBG 专用） | 94-96 | `core_rst_n_in_sync/qlfy/status` |
| Reset 公共 | 98-105 | `core_rst_n`、`core_rst_n_status_sync`、`core_rst_status`、`core2hdu_rdc_qlfy`、`core2dm_rdc_qlfy`、各 sync |
| TAPC-DMI 接口 | 109-130 | channel 选择/捕获/移位/更新/TDI/TDO、`dmi_*` |
| TAPC-SCU 接口 | 132-138 | `tapc_scu_*`、`tapc_ch_tdo`、`sys_rst_n/status`、`hdu_rst_n` |
| SCU 其余 | 139-143 | `hdu2dm_rdc_qlfy`、`ndm_rst_n`、`dm_rst_n`、`hart_rst_n` |
| DM-Pipeline 接口 | 149-175 | run control、pbuf、dreg、pc_sample |
| 时钟门控 | 180-185 | `sleep_pipe/wake_pipe/clk_pipe/clk_pipe_en/clk_dbgc/clk_alw_on` |

---

## 4. 复位逻辑

### 4.1 DBG 路径（`:192-236`）

例化 `scr1_scu`（`:193-231`）：

- 输入：`pwrup_rst_n`/`rst_n`/`cpu_rst_n`/`test_mode`/`test_rst_n`/`clk`、
  SCU scan-chain（由 TAPC 经 synchronizer 驱动，`:203-209`）、
  `ndm_rst_n_i`/`hart_rst_n_i`（由 DM 产生，`:212-213`）；
- 输出：`sys_rst_n_o`、`core_rst_n_o`、`dm_rst_n_o`、`hdu_rst_n_o`（`:216-219`）、
  `sys_rst_status_o`/`core_rst_status_o`（`:222-223`）、
  四个 RDC qualifier（`:226-230`）。
- `assign sys_rst_n_o = sys_rst_n`（`:233`）；
- `assign pwrup_rst_n_sync = pwrup_rst_n`（`:236`，复位输入假定同步）。

### 4.2 non-DBG 路径（`:238-270`）

1. `pwrup_rst_n_sync`/`rst_n_sync`/`cpu_rst_n_sync` 直通（`:241-243`）；
2. `core_rst_n_in_sync = rst_n_sync & cpu_rst_n_sync`（`:244`）；
3. `scr1_reset_qlfy_adapter_cell_sync`（`:247-256`）产生 `core_rst_n_qlfy`/
   `core_rst_n`/`core_rst_n_status`；
4. `scr1_data_sync_cell`（2 级，`:258-265`）同步 `core_rst_n_status`；
5. `core_rst_status = ~core_rst_n_status_sync`（`:267`）；
   `core_rdc_qlfy_o = core_rst_n_qlfy`（`:268`）。

两路径汇合：`assign core_rst_n_o = core_rst_n`（`:271`）。

> 原语所在：`scr1_data_sync_cell`（`primitives/scr1_reset_cells.sv:101`）、
> `scr1_reset_qlfy_adapter_cell_sync`（同文件 `:154`）。

---

## 5. 流水线装配（`:276-355`）

`scr1_pipe_top i_pipe_top`：

- **复位/时钟**：`pipe_rst_n=core_rst_n`（`:278`）；非 CLKCTRL 用 `clk`（`:284`），
  CLKCTRL 用 `clk_pipe` 并回传 `sleep_req`/`wake_req`（`:286-292`）；
- **内存**：IMEM（`:295-300`）与 DMEM（`:303-310`）直连 core 顶层端口；
- **调试**（`:312-339`）：`dbg_en=1'b1`（`:314`），run control（`:317-323`）、
  pbuf（`:326-327`）、dreg（`:330-335`）、`pc_sample`（`:338`）；
- **IRQ**（`:342-348`）：IPIC 时接 `core_irq_lines_i`，否则 `core_irq_ext_i`；
- **timer/fuse**：`core_mtimer_val_i`（`:351`）、`core_fuse_mhartid_i`（`:354`）。

---

## 6. 调试子系统（`SCR1_DBG_EN`，`:358-439`）

### 6.1 TAPC（`:362-383`）与 synchronizer（`:385-413`）

- `scr1_tapc` 处理 JTAG 状态机与 scan-chain 输出（合成 `tapc_tdo`/`tapc_tdo_en`），
  输出经 `_tapout` 后缀信号；
- `scr1_tapc_synchronizer` 把 TAPC 时钟域（TCK）信号同步到 core 时钟域，
  输出无后缀的 `tapc_dmi_ch_*`/`tapc_scu_ch_sel`；
- `tapc_ch_tdo` 由 SCU/DMI 两路按 sel 合并（`:414-415`）。

### 6.2 DMI（`:417-437`）

`scr1_dmi` 桥接 TAP scan-chain 与 DM 内部寄存器接口：
`dmi2dm_req/wr/addr/wdata`（`:433-436`）→ DM，`dm2dmi_resp/rdata`（`:431-432`）← DM。

---

## 7. Debug Module（`:442-496`）

### 7.1 RDC qualifier 掩码（`:447-455`）

| 信号 | 规则 | 行号 |
|---|---|---|
| `dm_cmd_resp_qlfy` | `dm_cmd_resp & hdu2dm_rdc_qlfy` | 447 |
| `dm_hart_event_qlfy` | `dm_hart_event & hdu2dm_rdc_qlfy` | 448 |
| `dm_hart_status_qlfy.dbg_state` | qualifier 无效时强制 `SCR1_HDU_DBGSTATE_RESET` | 449-450 |
| `dm_hart_status_qlfy.except/ebreak` | 直通 | 451-452 |
| `dm_pbuf_addr_qlfy` | `dm_pbuf_addr & hdu2dm_rdc_qlfy` | 453 |
| `dm_dreg_req_qlfy` | `dm_dreg_req & hdu2dm_rdc_qlfy` | 454 |
| `dm_pc_sample_qlfy` | `dm_pc_sample & core2dm_rdc_qlfy` | 455 |

### 7.2 `scr1_dm`（`:457-495`）

对接 DMI、run control、fuse、pc_sample、pbuf、dreg；输出
`ndm_rst_n_o`/`hart_rst_n_o`（回 SCU，`:471-472`）与 `dm2pipe_*` 控制。

---

## 8. 时钟门控（`SCR1_CLKCTRL_EN`，`:499-519`）

`scr1_clk_ctrl`：输入 `clk`/`core_rst_n`/`test_mode`/`test_rst_n` 与
`pipe2clkctl_sleep_req_i`/`wake_req_i`（`:510-511`），
输出 `clk_alw_on`/`clk_pipe`/`clk_pipe_en`/`clk_dbgc`（`:514-517`）。
未启用 CLKCTRL 时流水线直接用 `clk`（`:283-284`）。

---

## 9. 配置宏影响

| 宏 | 内容 | 位置 |
|---|---|---|
| `SCR1_DBG_EN` | SCU/TAPC/DMI/DM 路径、`sys_rst_n_o` 端口、RDC 掩码 | `:10`/`:30-33`/`:192`/`:358`/`:442` |
| `SCR1_IPIC_EN` | `core_irq_lines_i` vs `core_irq_ext_i` | `:16`/`:42-46`/`:342-346` |
| `SCR1_CLKCTRL_EN` | 时钟门控例化与 `clk_pipe` | `:178`/`:283-292`/`:499-519` |

MAX 配置同时定义 `SCR1_DBG_EN` 与 `SCR1_IPIC_EN`（`scr1_arch_description.svh:79`、`:83`）；
`SCR1_CLKCTRL_EN` 默认注释关闭（同文件 `:202` 附近）。

---

## 10. 附录

### 10.1 子模块例化索引

| 例化名 | 模块 | 行号 | 条件 |
|---|---|---|---|
| `i_scu` | `scr1_scu` | 193-231 | `SCR1_DBG_EN` |
| `i_core_rstn_qlfy_adapter_cell_sync` | `scr1_reset_qlfy_adapter_cell_sync` | 247-256 | `!SCR1_DBG_EN` |
| `i_core_rstn_status_sync` | `scr1_data_sync_cell` | 258-265 | `!SCR1_DBG_EN` |
| `i_pipe_top` | `scr1_pipe_top` | 276-355 | — |
| `i_tapc` | `scr1_tapc` | 362-383 | `SCR1_DBG_EN` |
| `i_tapc_synchronizer` | `scr1_tapc_synchronizer` | 385-413 | `SCR1_DBG_EN` |
| `i_dmi` | `scr1_dmi` | 417-437 | `SCR1_DBG_EN` |
| `i_dm` | `scr1_dm` | 457-495 | `SCR1_DBG_EN` |
| `i_clk_ctrl` | `scr1_clk_ctrl` | 503-518 | `SCR1_CLKCTRL_EN` |

### 10.2 信号映射

| Core 端口 | 内部/子模块 | 行号 |
|---|---|---|
| `imem2core_req_ack_i` | `.imem2pipe_req_ack_i` | 298 |
| `core2imem_*` | `.pipe2imem_*` | 295-297 |
| `dmem2core_*` | `.dmem2pipe_*` | 308-310 |
| `core2dmem_*` | `.pipe2dmem_*` | 303-307 |
| `core_irq_lines_i` | `.soc2pipe_irq_lines_i` | 343 |
| `core_mtimer_val_i` | `.soc2pipe_mtimer_val_i` | 351 |
| `core_fuse_mhartid_i` | `.soc2pipe_fuse_mhartid_i` + DM | 354/481 |

### 10.3 职责边界

- 仅做“复位+时钟+调试+流水线”的连线与复位域处理，不含运算逻辑；
- RDC 掩码保证跨复位域信号在 qualifier 无效时归零；
- 非 DBG 与 DBG 的端口/结构不同，集成方需按配置宏适配。
