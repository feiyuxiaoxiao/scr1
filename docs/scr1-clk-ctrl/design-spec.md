# SCR1 Clock Control 时钟控制 设计规格（Design Specification）

- 模块：`scr1_clk_ctrl`（`src/core/scr1_clk_ctrl.sv`，55 行）
- 条件：仅在 `SCR1_CLKCTRL_EN` 时编译（`:8`、`:55`）
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:503-518`，例化名 `i_clk_ctrl`）
- 职责：根据流水线的 sleep/wake 请求对流水线时钟进行门控（`:1-3`）

---

## 1. 概述与职责

时钟控制模块在流水线进入睡眠（WFI）时关断流水线时钟，在唤醒请求到来时恢复：

1. **请求握手**：`pipe2clkctl_sleep_req_i` / `pipe2clkctl_wake_req_i`；
2. **使能状态机**：维护 `clkctl2pipe_clk_en_o`（单 bit 状态）；
3. **时钟门控**：用 `scr1_cg` 门控出 `clkctl2pipe_clk_o`；
4. **旁路时钟**：`clk_alw_on` 与 `clk_dbgc` 恒等于 `clk`（不门控）。

---

## 2. 端口（`:9-22`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `clk` | in | 10 | 模块时钟 |
| `rst_n` | in | 11 | 复位 |
| `test_mode` / `test_rst_n` | in | 12/13 | DFT |
| `pipe2clkctl_sleep_req_i` | in | 15 | 关时钟请求 |
| `pipe2clkctl_wake_req_i` | in | 16 | 开时钟请求 |
| `clkctl2pipe_clk_alw_on_o` | out | 18 | 常开时钟 |
| `clkctl2pipe_clk_o` | out | 19 | 门控后时钟 |
| `clkctl2pipe_clk_en_o` | out | 20 | 使能标志 |
| `clkctl2pipe_clk_dbgc_o` | out | 21 | 调试子系统时钟 |

---

## 3. 逻辑（`:24-51`）

- `clkctl2pipe_clk_alw_on_o = clk`、`clkctl2pipe_clk_dbgc_o = clk`（`:26-27`，不门控）；
- `ctrl_rst_n = test_mode ? test_rst_n : rst_n`（`:28`）；
- **使能状态机**（`:30-44`，`posedge clk`/`negedge ctrl_rst_n`）：
  - 复位时 `clk_en = 1'b1`（`:32`）；
  - 使能中：`sleep_req & ~wake_req` → 关（`:35-37`）；
  - 关闭中：`wake_req` → 开（`:39-41`）；
- **门控单元** `scr1_cg i_scr1_cg_pipe`（`:46-51`）：`clk_en` + `test_mode` → `clk_out`
  （`src/core/primitives/scr1_cg.sv`）。

---

## 4. 附录

### 4.1 配置宏

仅 `SCR1_CLKCTRL_EN`（`:8`）。`scr1_arch_description.svh` 默认关闭；
未启用时 `scr1_core_top` 直接用 `clk` 驱动流水线（`scr1_core_top.sv:283-284`）。

### 4.2 职责边界

- 仅门控“流水线时钟”，不门控 debug 子系统与常开时钟；
- 睡眠请求优先于同时到来的唤醒（`sleep_req & ~wake_req`）；
- 使能状态跨复位由 `ctrl_rst_n` 控制，测试模式下用 `test_rst_n`。
