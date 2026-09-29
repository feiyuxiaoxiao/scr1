# SCR1 Clock Control 时钟控制 验证文档（Verification Document）

- 被测对象：`scr1_clk_ctrl`（`src/core/scr1_clk_ctrl.sv`，55 行，仅 `SCR1_CLKCTRL_EN`）
- 配套设计规格：`docs/scr1-clk-ctrl/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:503-518`，例化名 `i_clk_ctrl`）

> `SCR1_CLKCTRL_EN` 默认注释关闭（`scr1_arch_description.svh` 附近），
> 常规回归不编译本模块；需显式启用后验证。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证复位后 `clk_en=1`（时钟开启）；
2. 验证 sleep 请求关时钟、wake 请求开时钟；
3. 验证 sleep&wake 同时到达时保持开启（sleep 优先）；
4. 验证 `clk_alw_on`/`clk_dbgc` 恒等 `clk`；
5. 验证 `ctrl_rst_n` 在 test_mode 下取 `test_rst_n`；
6. 验证门控时钟在 en=0 时不翻转。

### 1.2 范围

- 直接驱动请求信号 + 波形核对；需 `SCR1_CLKCTRL_EN` 构建。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_clk_ctrl.*`：`clkctl2pipe_clk_en_o`、
`clkctl2pipe_clk_o`、`clkctl2pipe_clk_alw_on_o`、`clkctl2pipe_clk_dbgc_o`、`ctrl_rst_n`。

---

## 3. 验证策略

1. **请求序列**：单独/wake+sleep 组合驱动，观察 `clk_en` 与门控时钟；
2. **波形**：确认 en=0 时 `clk_o` 不翻转，en=1 时跟随 `clk`；
3. **test_mode**：切换复位源。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-CLKCTL-1 | 复位后 `clk_en=1` | 复位 |
| FC-CLKCTL-2 | sleep=1,wake=0 → 关闭 | 驱动 |
| FC-CLKCTL-3 | sleep=0,wake=1 且关闭 → 开启 | 驱动 |
| FC-CLKCTL-4 | sleep=1,wake=1 → 保持开启 | 同时驱动 |
| FC-CLKCTL-5 | sleep=0,wake=0 保持状态 | 波形 |
| FC-CLKCTL-6 | `clk_o` 在 en=0 时静止 | 波形 |
| FC-CLKCTL-7 | `clk_alw_on=clk`、`clk_dbgc=clk` | 波形 |
| FC-CLKCTL-8 | `ctrl_rst_n` 选择（test_mode） | 切换 |

---

## 5. 断言验证（Assertions）

本模块**未内建 SVA**。正确性通过波形核对（§6）。

---

## 6. 已执行的验证活动与结果

### 6.1 静态核对

- 使能逻辑（`:34-42`）、旁路（`:26-27`）、复位源选择（`:28`）与设计规格一致；
- 无 SVA（grep 为空）。

### 6.2 默认回归

`SCR1_CLKCTRL_EN` 默认关闭，常规 MAX/MIN 回归不编译本模块，因此只能静态核对。

### 6.3 定向专项（建议执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| 开→关 | sleep=1 | `clk_en=0`，`clk_o` 静止 | 2,6 |
| 关→开 | wake=1 | `clk_en=1`，`clk_o` 恢复 | 3 |
| 睡眠中唤醒竞争 | sleep=wake=1（开启态） | 保持开启 | 4 |
| WFI 实测 | 程序执行 WFI + 中断唤醒 | 时钟停/启 | 2-3 |

> 6.3 尚未系统执行。

---

## 7. 回归流程

默认回归不含本模块。启用时定义 `SCR1_CLKCTRL_EN` 重新构建
（与 `deploy-website`/`build_verilator` 相同流程），跑 `hello` 并指令 WFI 观察睡眠。

---

## 8. 验证结论

1. 时钟控制逻辑简单清晰，使能状态机与门控单元与源码一致；
2. 默认配置下不编译，需显式启用才能动态验证；
3. 建议按 6.3 建立 sleep/wake 定向测试。

**遗留建议**：在 `SCR1_CLKCTRL_EN` 配置下建立 WFI 睡眠/中断唤醒定向测试，覆盖 FC-CLKCTL-1~8。
