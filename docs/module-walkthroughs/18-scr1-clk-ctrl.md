# scr1_clk_ctrl.sv 讲解

## 一句话职责

时钟控制器：复位路径选择 + 门控使能 FSM + ICG（时钟门控单元）驱动。

## 文件信息

- **路径**: `src/core/scr1_clk_ctrl.sv`
- **行数**: 55 行
- **配置依赖**: `SCR1_CLKCTRL_EN`

## 核心架构

### 1. 常开时钟（`:26-27`）

```systemverilog
assign clk_alw_on = clk;   // 常开时钟（调试/复位域用）
assign clk_dbgc   = clk;   // 调试时钟（JTAG/DM 用）
```

**为什么需要？** 系统时钟门控后，调试模块仍需时钟（调试器随时可访问）。

### 2. 门控使能寄存器（`:30-44`）

```systemverilog
always_ff @(negedge rst_n, posedge clk) begin
    if (~rst_n) begin
        clk_en <= 1'b1;     // 复位后默认开启
    end else begin
        if (wake_req) begin
            clk_en <= 1'b1; // 唤醒请求（中断/调试）
        end else if (sleep_req) begin
            clk_en <= 1'b0; // 睡眠请求（WFI）
        end
    end
end
```

**优先级**：`wake` 覆盖 `sleep`（确保中断能唤醒）。

### 3. 复位路径选择（`:28`）

```systemverilog
assign ctrl_rst_n = (test_mode) ? test_rst_n : rst_n;
```

测试模式用 `test_rst_n`（DFT 专用），正常运行用 `rst_n`。

### 4. ICG 实例化（`:46-51`）

```systemverilog
scr1_cg i_cg (
    .clk(clk),
    .clk_en(clk_en),
    .test_mode(test_mode),
    .clk_out(clk_en_out)  // 门控后时钟
);
```

**ICG 行为**（见 `scr1_cg.sv`）：
```systemverilog
always_latch begin
    if (~clk) latch_en <= test_mode | clk_en;
end
assign clk_out = latch_en & clk;
```

锁存器在 `clk=0` 时采样 `clk_en`，避免毛刺。

## 使能来源

| 来源 | 信号 | 作用 |
|---|---|---|
| WFI | `wfi_halted` | `sleep_req`（停时钟省电） |
| 中断 | `csr2exu_ip_ie` | `wake_req`（中断唤醒） |
| 调试 | `dm2pipe_active` | `wake_req`（调试器访问） |

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `rst_n` | in | 系统复位 |
| `clk` | in | 系统时钟 |
| `test_mode` | in | 测试模式 |
| `test_rst_n` | in | 测试复位 |
| `clk_en_i` | in | 外部使能请求（可选） |
| `clk_en_o` | out | 门控使能（去 ICG） |
| `clk_alw_on_o` | out | 常开时钟 |
| `clk_dbgc_o` | out | 调试时钟 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_CLKCTRL_EN` | 模块使能，否则 `clk_en_out=1'b1`（常开） |

## 与其他模块关系

```
CSR/HDU → ClkCtrl: sleep_req / wake_req
ClkCtrl → ICG: clk_en
ICG → 全芯片：clk_en_out（门控时钟）
```

## 配套文档

- 设计规格：`docs/scr1-clk-ctrl/design-spec.md`
- 验证文档：`docs/scr1-clk-ctrl/verification.md`
