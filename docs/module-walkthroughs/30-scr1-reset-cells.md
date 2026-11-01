# scr1_reset_cells.sv 讲解

## 一句话职责

复位原语集合：7 个单元（buf/sync/qlfy/and/mux）用于复位同步、 CDC、 RDC 限定。

## 文件信息

- **路径**: `src/core/primitives/scr1_reset_cells.sv`
- **行数**: 231 行
- **配置依赖**: 无

## 7 个原语单元

### 1. `scr1_reset_buf_cell`（`:9-44`）—— 复位缓冲

**异步复位、同步释放**：
```systemverilog
always_ff @(negedge rst_n_mux, posedge clk) begin
    if (~rst_n_mux) reset_n_ff <= 1'b0;  // 异步清零
    else            reset_n_ff <= reset_n_in;  // 同步释放
end
```

**双输出**（`:33`/`:42`）：
```systemverilog
assign reset_n_out = ...;           // 功能逻辑用
assign reset_n_status = ...;        // 状态指示用（独立触发器）
```

**为什么两个独立触发器？** 给后端时序分析分成两条路径——避免负载耦合。

**测试 mux**（`:23`）：
```systemverilog
assign rst_n_mux = (test_mode) ? test_rst_n : rst_n;
```

### 2. `scr1_reset_sync_cell`（`:49-96`）—— 复位同步器

**多级同步**：
```systemverilog
rst_n_dff <= {rst_n_dff[STAGES_AMOUNT-2:0], rst_n_in};  // :87
assign rst_n_out = rst_n_dff[STAGES_AMOUNT-1];          // :94
```

**单级分支**（`:69-77`）：
```systemverilog
if (STAGES_AMOUNT == 1) begin
    // 直接同步
end else begin
    // 移位同步
end
```

**用途**：跨时钟域复位同步，消除 recovery/removal 违例。

### 3. `scr1_data_sync_cell`（`:101-144`）—— 单比特 CDC

**标准多级触发器**：
```systemverilog
data_dff <= {data_dff[STAGES_AMOUNT-2:0], data_in};
assign data_out = data_dff[STAGES_AMOUNT-1];
```

**用途**：跨时钟域传 1 位数据（非复位信号）。

### 4. `scr1_reset_qlfy_adapter_cell_sync`（`:154-195`）—— RDC 限定适配器

**核心技巧**：同一个复位源出两个信号：

```systemverilog
// 前级同步
always_ff @(negedge rst_n_mux, posedge clk) begin
    reset_n_front_ff <= reset_n_in_sync;  // :175
end

// Qualifier 输出（早一拍）
assign reset_n_out_qlfy = reset_n_front_ff;  // :181

// Buffer 输出（晚一拍）
scr1_reset_buf_cell i_buf (
    .reset_n_in(reset_n_front_ff),  // :190
    .reset_n_out(reset_n_out)
);
```

**为什么需要 Qualifier？**

跨复位域（RDC）时，目的域在复位撤销前需要先清掉跨域通路（避免亚稳态传播）。Qualifier 比复位早一拍拉低，给同步器留时间。

**SVA 验证**（见 `scr1_scu.sv:487-510`）：
```systemverilog
(~reset_n_qlfy) |=> (~reset_n_out);  // qualifier 先落，复位后落
```

**总级数**（`:150-153`）：1 前级同步 + 1 输出缓冲 = 2 级。

### 5. `scr1_reset_and2_cell`（`:197-206`）

```systemverilog
assign rst_n_out = (test_mode) ? test_rst_n : (&rst_n_in);
```

**用途**：多路复位相与（如 `tapc_trst_n = trst_n & pwrup_rst_n`）。

### 6. `scr1_reset_and3_cell`（`:209-218`）

同上，三输入。

### 7. `scr1_reset_mux2_cell`（`:221-230`）

```systemverilog
assign rst_n_out = (test_mode) ? test_rst_n : rst_n_in[select];
```

**用途**：二选一复位（如选择系统复位或调试复位）。

## 关键信号（buf_cell 为例）

| 信号 | 方向 | 含义 |
|---|---|---|
| `rst_n` | in | 系统复位 |
| `clk` | in | 时钟 |
| `test_mode` | in | 测试模式 |
| `test_rst_n` | in | 测试复位 |
| `reset_n_in` | in | 输入复位 |
| `reset_n_out` | out | 缓冲后复位 |
| `reset_n_status` | out | 状态指示 |

## 配套文档

- 设计规格：`docs/scr1-reset-cells/design-spec.md`
- 验证文档：`docs/scr1-reset-cells/verification.md`
