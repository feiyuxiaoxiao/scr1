# SCR1 复位处理原语验证文档（Verification）

- 被测文件：`src/core/primitives/scr1_reset_cells.sv`（231 行，7 个模块）
- 验证目标：复位缓冲/同步/适配与组合复位单元的功能与测试旁路

---

## 1. 功能检查项（Feature Checklist）

| ID | 模块/功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-RST-1 | `reset_buf_cell` 异步清与采样 | `:25-31` | 复位/释放波形 |
| FC-RST-2 | `reset_buf_cell` 状态输出 | `:35-42` | `reset_n_status` 观察 |
| FC-RST-3 | `reset_sync_cell` 单级分支 | `:67-77` | `STAGES=1` |
| FC-RST-4 | `reset_sync_cell` 多级移位 | `:79-90` | `STAGES>1` |
| FC-RST-5 | `data_sync_cell` 单/多级 | `:112-140` | 参数化检查 |
| FC-RST-6 | `qlfy_adapter` 前级+缓冲 | `:171-193` | `out_qlfy`/`out` 波形 |
| FC-RST-7 | `and2` 两输入与 | `:204` | 四种输入组合 |
| FC-RST-8 | `and3` 三输入与 | `:216` | 八种输入组合 |
| FC-RST-9 | `mux2` 选择 | `:229` | `select` 0/1 |

---

## 2. 测试旁路检查

所有单元在 `test_mode=1` 时输出应为 `test_rst_n`：
`reset_buf_cell`（`:23`/`:33`）、`reset_sync_cell`（`:63`/`:94`）、`and2`（`:204`）、
`and3`（`:216`）、`mux2`（`:229`）。

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| 复位序列 | 异步置位、同步释放 |
| 级数参数 | `STAGES_AMOUNT=1/2/N` |
| 测试模式 | 各单元旁路 |
| 多域同步 | 不同时钟域下同步稳定性 |

---

## 4. 断言

本文件内模块均无 SVA。

---

## 5. 验证结论

复位原语覆盖同步/缓冲/组合逻辑与测试旁路；正确性通过子模块与集成复位序列回归保障。
