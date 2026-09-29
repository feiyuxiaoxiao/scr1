# SCR1 复位处理原语设计规格（Design Specification）

- 文件：`src/core/primitives/scr1_reset_cells.sv`（231 行，无 `ifdef` 保护）
- 职责：提供复位缓冲、复位/数据同步、复位域限定适配与组合复位单元
- 本文件包含 7 个模块

---

## 1. 模块清单

| 模块 | 行号 | 功能 |
|---|---|---|
| `scr1_reset_buf_cell` | 9-44 | 复位缓冲（含状态输出） |
| `scr1_reset_sync_cell` | 49-96 | 复位 CDC 同步（参数化级数） |
| `scr1_data_sync_cell` | 101-144 | 数据 CDC/RDC 同步（参数化级数） |
| `scr1_reset_qlfy_adapter_cell_sync` | 154-195 | 复位/RDC 限定适配 |
| `scr1_reset_and2_cell` | 197-206 | 两输入复位与 |
| `scr1_reset_and3_cell` | 209-218 | 三输入复位与 |
| `scr1_reset_mux2_cell` | 221-231 | 两输入复位选择 |

---

## 2. `scr1_reset_buf_cell`（`:9-44`）

- `rst_n_mux = test_mode ? test_rst_n : rst_n`（`:23`）；
- `reset_n_ff` 在 `negedge rst_n_mux` 异步清零，否则采样 `reset_n_in`（`:25-31`）；
- 输出 `reset_n_out = test_mode ? test_rst_n : reset_n_ff`（`:33`）；
- `reset_n_status` 由同源寄存器输出（`:35-42`）。

---

## 3. `scr1_reset_sync_cell`（`:49-96`）

- 参数 `STAGES_AMOUNT=2`；
- `local_rst_n_in = test_mode ? test_rst_n : rst_n`（`:63`）；
- `STAGES_AMOUNT==1`：单级 `always_ff`（`:67-77`）；
- `STAGES_AMOUNT>1`：移位链 `rst_n_dff <= {dff[STAGES-2:0], rst_n_in}`（`:79-90`）；
- 输出末级或 `test_rst_n`（`:94`）。

---

## 4. `scr1_data_sync_cell`（`:101-144`）

- 参数 `STAGES_AMOUNT=1`；
- 单级/多级两种生成（`:112-140`）；
- `data_out = data_dff[STAGES-1]`（`:142`）。

---

## 5. `scr1_reset_qlfy_adapter_cell_sync`（`:154-195`）

- 前级同步：`reset_n_front_ff` 采样 `reset_n_in_sync`（`:171-177`）；
- `reset_n_out_qlfy = reset_n_front_ff`（`:181`）；
- 后接 `scr1_reset_buf_cell` 产生 `reset_n_out`/`reset_n_status`（`:184-193`）；
- 注释说明总级数 = 1 前级同步 + 1 输出缓冲（`:150-152`）。

---

## 6. 组合复位单元

`scr1_reset_and2_cell`（`:197-206`）、`scr1_reset_and3_cell`（`:209-218`）：
`rst_n_out = test_mode ? test_rst_n : (&rst_n_in)`；
`scr1_reset_mux2_cell`（`:221-231`）：`rst_n_out = test_mode ? test_rst_n : rst_n_in[select]`。

---

## 7. 附录

### 7.1 职责边界

- 全部为无状态/同步原语，供 cluster 各处例化（如 `scr1_core_top`、`scr1_top_*`）；
- 所有输出在 `test_mode=1` 时旁路为 `test_rst_n`；
- 无 SVA。
