# scr1_tapc_shift_reg.sv 讲解

## 一句话职责

JTAG 移位寄存器：参数化宽度 + 双复位（异步 + 同步）+ LSB 先出。

## 文件信息

- **路径**: `src/core/scr1_tapc_shift_reg.sv`
- **行数**: 112 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心架构

### 1. 参数化（`:9-11`）

```systemverilog
parameter int unsigned SCR1_WIDTH = 8;       // 寄存器宽度
parameter logic [SCR1_WIDTH-1:0] SCR1_RESET_VALUE = '0;  // 复位值
```

**用途**：复用同一个模块实现不同宽度的移位寄存器（IR/DR/SCU 链）。

### 2. 双复位优先级（`:40-53`/`:59-73`）

```systemverilog
always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n) begin                    // 异步复位（最高优先级）
        shift_reg <= SCR1_RESET_VALUE;
    end else if (~rst_n_sync) begin      // 同步复位（次高）
        shift_reg <= SCR1_RESET_VALUE;
    end else if (fsm_dr_select & fsm_dr_capture) begin
        shift_reg <= din_parallel;       // 并行捕获
    end else if (fsm_dr_select & fsm_dr_shift) begin
        shift_reg <= {din_serial, shift_reg[SCR1_WIDTH-1:1]};  // 串行移位
    end
end
```

**优先级顺序**：异步复位 > 同步复位 > capture > shift。

### 3. 移位方向（`:52`/`:70`）

```systemverilog
shift_reg <= {din_serial, shift_reg[SCR1_WIDTH-1:1]};
```

**LSB 先出**：新数据进最高位，整体右移，`dout_serial = shift_reg[0]`（`:86`）。

这正是 IEEE 1149.1 规范——JTAG 移位总是 LSB 先出。

### 4. 宽度 1 特例（`:57-75`）

```systemverilog
if (SCR1_WIDTH > 1) begin
    // 正常移位：{din, reg[W-1:1]}
end else begin
    // 宽度 1：直接赋值
    shift_reg <= din_serial;
end
```

**为什么需要？** `{din, shift_reg[0:1]}` 对宽度 1 不合法（索引越界），必须单独分支。

### 5. 并行输出（`:81`）

```systemverilog
assign dout_parallel = shift_reg;
```

**用途**：TAP FSM 读移位寄存器内容（如 IR 译码）。

### 6. SVA 断言（`:88-108`，仅仿真）

```systemverilog
`ifdef SCR1_TRGT_SIMULATION
SCR1_SVA_TAPC_SHIFTREG_XCHECK: assert property (
    @$negedge clk) disable iff (~rst_n)
    !$isunknown({rst_n_sync, fsm_dr_select, ...})
) else $error("TAPC Shift Reg unknown");
`endif
```

**用途**：仿真时检查控制信号无 `x` 态。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `clk` | in | TCK 或 SysCLK |
| `rst_n` | in | 异步复位 |
| `rst_n_sync` | in | 同步复位 |
| `fsm_dr_select` | in | DR 链选中 |
| `fsm_dr_capture` | in | 并行捕获 |
| `fsm_dr_shift` | in | 移位使能 |
| `din_serial` | in | 串行输入（TDO 前级） |
| `din_parallel` | in | 并行输入 |
| `dout_serial` | out | 串行输出（→ TDO） |
| `dout_parallel` | out | 并行输出（→ TAP FSM） |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | 模块使能 |
| `SCR1_TRGT_SIMULATION` | SVA 断言使能（综合忽略） |

## 配套文档

- 设计规格：`docs/scr1-tapc-shift-reg/design-spec.md`
- 验证文档：`docs/scr1-tapc-shift-reg/verification.md`
