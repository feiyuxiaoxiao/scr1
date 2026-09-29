# SCR1 时钟门控原语设计规格（Design Specification）

- 模块：`scr1_cg`（`src/core/primitives/scr1_cg.sv`，32 行；`SCR1_CLKCTRL_EN` 保护 `:8`/`:32`）
- 职责：带测试旁路的锁存器型时钟门控模型

---

## 1. 概述与职责

仿真用时钟门控：在 `clk` 低电平期间锁存使能，避免毛刺：

```systemverilog
always_latch begin
    if (~clk) latch_en <= test_mode | clk_en;
end
assign clk_out = latch_en & clk;
```

- `test_mode=1` 时强制导通（便于 DFT）；
- 综合时需替换为实现工艺专用的时钟门控单元（`:16-18` 注释说明）。

---

## 2. 端口（`:9-14`）

| 端口 | 方向 | 说明 |
|---|---|---|
| `clk` | in | 源时钟 |
| `clk_en` | in | 门控使能 |
| `test_mode` | in | 测试模式（旁路） |
| `clk_out` | out | 门控时钟 |

---

## 3. 附录

### 3.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_CLKCTRL_EN` | 模块是否存在 | `:8`/`:32` |

### 3.2 职责边界

- 仅实现仿真模型，无 SVA；
- 实际综合依赖工艺库替换；
- 被 `scr1_clk_ctrl` 用于产生 core/dm 时钟。
