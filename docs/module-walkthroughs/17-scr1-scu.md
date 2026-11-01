# scr1_scu.sv 讲解

## 一句话职责

系统控制单元：四路复位表达式生成 + RDC qualifier + STICKY 标志 + TAPC 链转发。

## 文件信息

- **路径**: `src/core/scr1_scu.sv`
- **行数**: 516 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心架构

### 1. 复位层次（`:371-464`）

```
sys_rst_n_in (系统级)
    └─ core_rst_n_in_sync (核心级，:401)
        ├─ hdu_rst_n_in_sync (HDU 级，:436)
        └─ dm_rst_n_in (DM 级，:464)
```

**sys_rst_n_in**（`:371`）：
```systemverilog
assign sys_rst_n_in = rst_n & cpu_rst_n & pwrup_rst_n;
```

**core_rst_n_in_sync**（`:401`）：
```systemverilog
scr1_reset_qlfy_adapter_cell_sync i_core_rst (
    .rst_n(sys_rst_n_in),
    .rst_n_in(core_rst_n_in_sync),
    .rst_n_out(core_rst_n_out),
    .rst_n_out_qlfy(core_rst_n_qlfy)  // 限定信号（早一拍）
);
```

**hdu_rst_n_in_sync**（`:436`）：
```systemverilog
assign hdu_rst_n_in_sync = core_rst_n_out & ~scu_halt_req;
```

**dm_rst_n_in**（`:464`）：
```systemverilog
scr1_reset_buf_cell i_dm_rst (
    .reset_n_in(dm_rst_n_in_sync),
    .reset_n_out(dm_rst_n_out)
);
```

### 2. RDC Qualifier（`:385`/`:416-420`/`:449`）

**为什么需要 Qualifier？**

跨复位域（RDC）时，目的域在复位撤销前需要先清掉跨域通路（避免亚稳态传播）。Qualifier 信号比复位早一拍拉低，给同步器留时间。

**SVA 验证**（`:487-510`）：
```systemverilog
property p_rdc_qlfy;
    @(posedge clk) disable iff (~rst_n)
    (~reset_n_qlfy) |=> (~reset_n_out);  // qualifier 先落，复位后落
endproperty
```

### 3. CONTROL 寄存器（`:279-285`/`:354-355`）

| 位 | 名称 | 作用 |
|---|---|---|
| `[0]` | `sys_rst` | 系统复位请求 |
| `[1]` | `core_rst` | 核心复位请求 |

写 1 触发对应复位（自清除）。

### 4. MODE 寄存器（`:291-297`）

| 位 | 名称 | 作用 |
|---|---|---|
| `[0]` | `debug_en` | 调试模式使能 |
| `[1]` | `halt_on_reset` | 复位后停机 |

### 5. STATUS 寄存器（`:303-306`）

| 位 | 名称 | 含义 |
|---|---|---|
| `[0]` | `halted` | 流水线已停机 |
| `[1]` | `running` | 流水线运行中 |

### 6. STICKY 寄存器（`:323-335`）

| 位 | 名称 | 含义 | 清除方式 |
|---|---|---|---|
| `[0]` | `halted_sticky` | 曾停机 | 写 1 清 |
| `[1]` | `reset_sticky` | 曾复位 | 写 1 清 |

**用途**：调试器查询"上次为什么停机"（即使当前已恢复）。

### 7. TAPC 链转发（`:158-206`）

```systemverilog
// SCU_ACCESS 链
assign scu_dr_select = (ir_reg == SCR1_TAP_IR_SCU);
assign scu_dr_shift  = tapc_dr_shift & scu_dr_select;

// 移位寄存器
always_ff @(posedge tapc_tck) begin
    if (scu_dr_shift) begin
        scu_shift_reg <= {scu_shift_reg[SCR1_SCU_DR_WIDTH-2:0], tapc_tdi};
    end
end
```

**SCU DR 链内容**：
- `CONTROL` 写
- `STATUS` 读
- `MODE` 写
- `STICKY` 读/清

### 8. Wdata RMW 处理（`:232-244`）

```systemverilog
// 读 - 改 - 写（只改被写位）
case (addr)
    SCU_CONTROL: wdata = (wmask & write_data) | (~wmask & control_reg);
    SCU_MODE:    wdata = (wmask & write_data) | (~wmask & mode_reg);
    ...
endcase
```

**为什么需要？** JTAG 移位是"全宽写"，但寄存器可能只有低位有效。RMW 保留未写位。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `rst_n` | in | 系统复位 |
| `pwrup_rst_n` | in | 上电复位 |
| `cpu_rst_n` | in | CPU 复位 |
| `core_rst_n_out` | out | 核心复位（去 core_top） |
| `hdu_rst_n_out` | out | HDU 复位（去 pipe_top） |
| `dm_rst_n_out` | out | DM 复位 |
| `scu_halt_req` | in | HDU 停机请求 |
| `tapc_scu_access_i` | in | TAPC 选中 SCU 链 |

## 复位完整流程

```
外部拉高 rst_n → SCU 同步（sys→core→hdu→dm）
→ Qualifier 先落（清跨域通路）→ 复位后落
→ 各域触发器清零 → 状态机回 IDLE
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | SCU 模块使能（TAPC 链/复位生成） |

## 配套文档

- 设计规格：`docs/scr1-scu/design-spec.md`
- 验证文档：`docs/scr1-scu/verification.md`
