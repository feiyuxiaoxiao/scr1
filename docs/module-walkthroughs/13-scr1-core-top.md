# scr1_core_top.sv 讲解

## 一句话职责

核心顶层：复位双路径（系统 vs 调试）+ 流水线装配 + 调试子系统（TAPC/DMI/DM/SCU）集成。

## 文件信息

- **路径**: `src/core/scr1_core_top.sv`
- **行数**: 521 行
- **配置依赖**: `SCR1_DBG_EN`、`SCR1_CLKCTRL_EN`

## 核心架构

### 1. 复位双路径（`:193-269`）

**DBG 复位域**（通过 SCU）：
```systemverilog
// SCU 输出
hdu_rst_n    // HDU 复位
dm_rst_n     // DM 复位
core_rst_n   // 核心复位

// 分配
i_pipe_top.dbg_rst_n  = hdu_rst_n;   // 流水线调试复位
i_tapc.rst_n          = hdu_rst_n;   // TAPC 复位
i_dm.rst_n            = dm_rst_n;    // DM 复位
```

**非 DBG 复位域**：
```systemverilog
// 系统复位同步
core_rst_n_in_sync = rst_n & cpu_rst_n;  // :244

// 2 级同步器（:258-265）
scr1_reset_qlfy_adapter_cell_sync i_sync (
    .rst_n_in(core_rst_n_in_sync),
    .rst_n_out(core_rst_n_out),
    .rst_n_out_qlfy(core_rst_n_qlfy)  // 限定信号（早一拍）
);
```

**为什么需要双路径？** 调试子系统（JTAG/DM/HDU）需要独立于系统复位——系统复位时调试器不能失联。

### 2. 流水线装配（`:276-355`）

```systemverilog
i_pipe_top (
    .clk(clk),
    .rst_n(core_rst_n_out),      // 系统复位
    .dbg_rst_n(hdu_rst_n),       // 调试复位
    .dbg_en(1'b1),               // 硬连线使能（SCR1_DBG_EN 时）
    .curr_pc(curr_pc),           // 输出到 HDU/TDU
    .exu_busy(exu_busy),         // 忙标志
    ...
);
```

### 3. 调试子系统实例化

| 实例 | 模块 | 复位域 |
|---|---|---|
| `i_tapc` | `scr1_tapc` | `hdu_rst_n`（DBG） |
| `i_dmi` | `scr1_dmi` | `pwrup_rst_n`（上电复位） |
| `i_dm` | `scr1_dm` | `dm_rst_n`（DM 复位） |
| `i_scu` | `scr1_scu` | `sys_rst_n`（系统复位） |

### 4. TAPC 同步器（`:385-413`）

```systemverilog
i_tapc_synchronizer (
    .pwrup_rst_n(pwrup_rst_n),
    .dm_rst_n(dm_rst_n),
    .clk(clk),
    .tapc_trst_n(tapc_trst_n),
    .tapc_tck(tapc_tck),          // JTAG TCK（异步）
    .tapc2tapcsync_ch_capture_i(...),
    .tapcsync2core_ch_capture_o(...),  // 同步到 SysCLK
    ...
);
```

**作用**：TCK（JTAG 时钟，频率未知）→ SysCLK（系统时钟）CDC。

### 5. DMI 桥接（`:417-437`）

```systemverilog
i_dmi (
    .tap_dmi_access_i(tap_dr_dmi),      // TAPC 选中 DMI 链
    .dmi2dm_req_o(dmi2dm_req),          // 请求 DM
    .dmi2dm_wr_o(dmi2dm_wr),            // 写操作
    .dmi2dm_addr_o(dmi2dm_addr),        // DM 寄存器地址
    .dmi2dm_data_o(dmi2dm_data),        // 写数据
    .dm2dmi_resp_i(dm2dmi_resp),        // DM 响应
    ...
);
```

### 6. DM 实例化（`:457-495`）

```systemverilog
i_dm (
    .rst_n(dm_rst_n),
    .clk(clk),
    .dmi2dm_req_i(dmi2dm_req),
    .dm2dmi_resp_o(dm2dmi_resp),
    .dm2hdu_active_o(dm2hdu_active),
    .dm2hdu_progbuf_o(dm2hdu_progbuf),
    ...
);
```

RDC 限定（`:447-455`）：
```systemverilog
assign dm_rdc_qlfy = ~dm_rst_n_qlfy;  // 复位前清跨域通路
```

### 7. 时钟控制器（`:503-518`）

```systemverilog
i_clk_ctrl (
    .rst_n(core_rst_n),
    .clk(clk),
    .clk_en_i(clk_en),           // 门控使能
    .clk_en_o(clk_en_out),       // 去 ICG
    .test_mode(test_mode),
);
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `rst_n` | in | 系统复位 |
| `pwrup_rst_n` | in | 上电复位 |
| `tapc_tck` | in | JTAG 时钟 |
| `tapc_trst_n` | in | JTAG 复位 |
| `clk` | in | 系统时钟 |
| `curr_pc` | out | 当前 PC（去 HDU/TDU） |
| `hdu_rst_n` | out | HDU 复位（去 pipe_top） |

## 复位层次

```
系统级：pwrup_rst_n ⊃ sys_rst_n ⊃ core_rst_n ⊃ hdu_rst_n
                     └─ dm_rst_n
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | 调试子系统实例化（TAPC/DMI/DM/SCU/HDU） |
| `SCR1_CLKCTRL_EN` | 时钟控制器实例化（门控使能 FSM） |

## 配套文档

- 设计规格：`docs/scr1-core-top/design-spec.md`
- 验证文档：`docs/scr1-core-top/verification.md`
