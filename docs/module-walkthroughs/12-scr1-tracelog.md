# scr1_tracelog.sv 讲解

## 一句话职责

轨迹日志：采样 PC/异常/中断事件，输出格式化日志（用于仿真调试）。

## 文件信息

- **路径**: `src/core/scr1_tracelog.sv`
- **行数**: 453 行
- **配置依赖**: `SCR1_TRACE_LOG_EN`、`SCR1_TRGT_SIMULATION`

## 核心功能

### 1. 事件类型（`:80-150`）

| 事件码 | 含义 | 触发条件 |
|---|---|---|
| `TRACE_NOP` | 无事件 | 默认填充 |
| `TRACE_INSTRET` | 指令退休 | `exu2pipe_instret_i=1` |
| `TRACE_EXCEPTION` | 异常 | `exu2csr_exc_i=1` |
| `TRACE_INTERRUPT` | 中断 | `exu2csr_irq_i=1` |
| `TRACE_WFI` | WFI 停机 | `wfi_halted=1` |
| `TRACE_DEBUG` | 调试停机 | `hdu2pipe_dmode_i=1` |

### 2. 采样时序（`:200-280`）

```systemverilog
always_ff @(posedge clk) begin
    if (trace_en && sample_condition) begin
        trace_event <= event_type;
        trace_pc    <= curr_pc;
        trace_tval  <= trap_value;
        trace_valid <= 1'b1;
    end
end
```

**采样条件**：
- `trace_en`：trace 使能（DCSR 控制）
- `sample_condition`：事件发生（退休/异常/中断）

### 3. 输出格式（`:300-380`）

```systemverilog
if (trace_valid) begin
    $format("TRACE: %t - PC=0x%08h EVENT=%0d TVAL=0x%08h",
             $time, trace_pc, trace_event, trace_tval);
end
```

**仿真专用**：综合时 `SCR1_TRGT_SIMULATION` 未定义，整个模块被 `ifdef` 移除。

### 4. 配置选项

| 配置 | 含义 |
|---|---|
| `SCR1_TRACE_LOG_EN` | 模块使能 |
| `SCR1_TRACE_DEPTH` | FIFO 深度（缓存事件） |
| `SCR1_TRACE_TVAL_EN` | 是否输出 trap_value |

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `trace_en_i` | in | Trace 使能 |
| `exu2pipe_instret_i` | in | 指令退休 |
| `exu2csr_exc_i` | in | 异常发生 |
| `exu2csr_irq_i` | in | 中断发生 |
| `curr_pc_i` | in | 当前 PC |
| `trap_value_i` | in | MTVAL 值 |
| `trace_valid_o` | out | 输出有效 |
| `trace_event_o` | out | 事件类型 |
| `trace_pc_o` | out | 事件 PC |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TRACE_LOG_EN` | 模块使能，否则输出恒 0 |
| `SCR1_TRGT_SIMULATION` | 仿真目标：包含 `$format`；综合：移除整个模块 |

## 与其他模块关系

```
EXU → Tracelog: instret/exc/irq
CSR → Tracelog: trap_value (MTVAL)
IFU → Tracelog: curr_pc
Tracelog → 仿真器：$format 输出
```

## 配套文档

- 设计规格：`docs/scr1-tracelog/design-spec.md`
- 验证文档：`docs/scr1-tracelog/verification.md`
