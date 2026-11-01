# scr1_cg.sv 讲解

## 一句话职责

时钟门控单元：ICG（Integrated Clock Gating Cell）仿真模型，锁存器抗毛刺。

## 文件信息

- **路径**: `src/core/primitives/scr1_cg.sv`
- **行数**: 32 行
- **配置依赖**: `SCR1_CLKCTRL_EN`

## 核心架构

### 1. 锁存器采样（`:22-26`）

```systemverilog
always_latch begin
    if (~clk) latch_en <= test_mode | clk_en;
end
```

**为什么用锁存器而不是触发器？**

门控时钟最怕毛刺。如果 `clk_en` 在 `clk=1` 期间变化，`clk_out = clk_en & clk` 就会在 `clk_out` 上切出一个窄脉冲，下游触发器直接崩。

用低电平透明的锁存器，`clk_en` **只能在 `clk=0` 期间被采进去**，`clk=1` 期间锁存值保持不变。于是 `clk_out` 的高电平永远是完整的一拍。

### 2. 门控输出（`:28`）

```systemverilog
assign clk_out = latch_en & clk;
```

**时序**：
```
clk:    ─┐┌──┐┌──┐┌──
        ││  ││  ││
latch_en: ──┘  └──┘  （clk=0 时采样 clk_en）
        ┌──────┐
clk_out:│      │────  （无毛刺）
```

### 3. 测试模式（`:24`）

```systemverilog
latch_en <= test_mode | clk_en;
```

**为什么需要？** DFT（Design for Test）测试时强制打开时钟，让扫描链能工作。

### 4. 注释警告（`:16-18`）

```systemverilog
// The code below is a clock gate model for simulation.
// For synthesis, replace with implementation-specific clock gate cell.
```

**含义**：这是仿真模型（行为级），综合时要替换成工艺库 ICG 单元（如 TSMC 的 `ICGCPx`）。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `clk` | in | 系统时钟 |
| `clk_en` | in | 时钟使能请求 |
| `test_mode` | in | 测试模式 |
| `clk_out` | out | 门控后时钟 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_CLKCTRL_EN` | 模块使能，否则 `clk_out=clk`（常开） |

## 与其他模块关系

```
ClkCtrl → CG: clk_en
CG → 全芯片：clk_out（门控时钟）
```

## 配套文档

- 设计规格：`docs/scr1-cg/design-spec.md`
- 验证文档：`docs/scr1-cg/verification.md`
