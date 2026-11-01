# scr1_tapc.sv 讲解

## 一句话职责

JTAG TAP 控制器：16 态 TAP FSM + IR（指令寄存器）+ DR（数据寄存器）+ 链转发。

## 文件信息

- **路径**: `src/core/scr1_tapc.sv`
- **行数**: 457 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心架构

### 1. 16 态 TAP FSM（`:135-167`）

IEEE 1149.1 标准状态机：

```
                    ┌─ Capture-IR ─ Shift-IR ─ Exit1-IR ─┐
                    │         ↑              ↓           │
Reset → Test-Logic-Reset ── Run-Test/Idle   Pause-IR     │
                    │         ↓              ↑           │
                    └─ Capture-DR ─ Shift-DR ─ Exit1-DR ─┘
                              ↑              ↓
                        Exit2-DR/IR ─ Update-DR/IR
```

**关键状态**：
- `Test-Logic-Reset`：上电/`trst_n` 后进入，IR=IDCODE
- `Run-Test/Idle`：空闲，可启动 BIST
- `Shift-IR/DR`：移位操作（TDI→reg→TDO）
- `Update-IR/DR`：锁存移位结果

### 2. IR（指令寄存器，`:181-208`）

```systemverilog
localparam SCR1_IR_WIDTH = 4;  // 4 位 IR
logic [SCR1_IR_WIDTH-1:0] ir_reg;
```

**SCR1 IR 编码**：
| IR 值 | 含义 |
|---|---|
| `0000` | 保留 |
| `0001` | IDCODE |
| `0010` | DMI_ACCESS（调试访问） |
| `0011` | SCU_ACCESS（系统控制） |
| `1110` | BYPASS |
| `1111` | 保留 |

**Capture-IR**（`:195-200`）：
```systemverilog
if (tap_fsm_next == SCR1_TAP_FSM_CAPTURE_IR) begin
    ir_reg <= {SCR1_IR_WIDTH{1'b0}};  // 捕获 0（最低位 1 是规范）
    ir_reg[0] <= 1'b1;                 // IDCODE 标记
end
```

**Reset**（`:200`）：
```systemverilog
ir_reg <= SCR1_TAP_IR_IDCODE;  // 复位到 IDCODE
```

### 3. DR 链选择（`:274-317`）

```systemverilog
case (ir_reg)
    SCR1_TAP_IR_IDCODE:   dr_chain = idcode_chain;
    SCR1_TAP_IR_BYPASS:   dr_chain = bypass_chain;
    SCR1_TAP_IR_DMI:      dr_chain = dmi_chain;
    SCR1_TAP_IR_SCU:      dr_chain = scu_chain;
endcase
```

**链转发**：
- `tap_drCapture`：并行加载
- `tap_drShift`：串行移位
- `tap_drUpdate`：锁存结果

### 4. TDO 驱动（`:326-341`）

```systemverilog
// TCK 下降沿驱动 TDO
always_ff @(negedge tapc_tck, negedge trst_n) begin
    if (~trst_n) begin
        tdo <= 1'b0;
    end else if (tap_fsm_next == SCR1_TAP_FSM_SHIFT_DR) begin
        tdo <= dr_chain.tdo;  // LSB 先出
    end
end
```

**tdo_en**（`:336`）：只在 `Shift-DR` 态使能 TDO 输出（三态控制）。

### 5. 内置 DR 链（`:368-426`）

#### IDCODE 链
```systemverilog
localparam [31:0] IDCODE = 32'h20000001;  // Syntacore SCR1
```

#### BYPASS 链
```systemverilog
logic bypass_ff;  // 1 位，直通
```

#### DMI 链
```systemverilog
// 转发到 DMI 模块
assign dmi_tck = tapc_tck;
assign dmi_ch_sel = (ir_reg == SCR1_TAP_IR_DMI);
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `tapc_tck` | in | JTAG 时钟 |
| `tapc_trst_n` | in | JTAG 复位（异步） |
| `tapc_tms` | in | 模式选择（状态转移） |
| `tapc_tdi` | in | 串行数据输入 |
| `tapc_tdo` | out | 串行数据输出 |
| `ir_reg` | out | 当前指令 |
| `dr_select` | out | DR 链使能 |
| `dr_capture` | out | DR 并行加载 |
| `dr_shift` | out | DR 移位使能 |
| `dr_update` | out | DR 锁存 |

## JTAG 完整流程

```
外部 → TMS/TDI/TCK → TAP FSM → IR 译码 → 选择 DR 链
→ Shift-DR 移位 → Update-DR 锁存 → DM/SCU 执行
→ DR 链移回 → TDO 输出
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | TAPC 模块使能 |

## 配套文档

- 设计规格：`docs/scr1-tapc/design-spec.md`
- 验证文档：`docs/scr1-tapc/verification.md`
