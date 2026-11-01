# scr1_tapc_synchronizer.sv 讲解

## 一句话职责

TCK→SysCLK CDC 同步器：TCK 二分频降频 + 四拍同步 + 边沿检测出单拍脉冲。

## 文件信息

- **路径**: `src/core/scr1_tapc_synchronizer.sv`
- **行数**: 183 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心难点

**为什么普通两级同步器不行？**

TCK 是 JTAG 时钟，频率未知（可能 1MHz~50MHz），与 SysCLK（系统时钟，可能 100MHz+）完全异步。

- 如果 SysCLK >> TCK：两级同步器可能采样到多次（同一电平被连采几拍）
- 如果 SysCLK << TCK：可能完全漏掉（TCK 脉冲太窄）

## 核心架构

### 1. TCK 二分频（`:68-82`）

```systemverilog
// 上升沿翻转
always_ff @(posedge tapc_tck, negedge tapc_trst_n) begin
    if (~tapc_trst_n) tck_divpos <= 1'b0;
    else              tck_divpos <= ~tck_divpos;
end

// 下降沿翻转
always_ff @(negedge tapc_tck, negedge tapc_trst_n) begin
    if (~tapc_trst_n) tck_divneg <= 1'b0;
    else              tck_divneg <= ~tck_divneg;
end
```

**效果**：
```
TCK:       ─┐┌──┐┌──┐┌──
tck_divpos: ─┘  └──┘  └──  （周期=2*TCK）
tck_divneg:   ┌──┐┌──┐     （与 divpos 反相）
```

**为什么有效？** 把一个可能很快的 TCK 变成**至少 2 个 TCK 周期宽的电平**，慢到 SysCLK 能可靠采到。

### 2. 四拍同步（`:84-89`）

```systemverilog
always_ff @(posedge clk, negedge pwrup_rst_n) begin
    tck_divpos_sync <= {tck_divpos_sync[2:0], tck_divpos};  // 移位寄存器
    tck_divneg_sync <= {tck_divneg_sync[2:0], tck_divneg};
end
```

**效果**：
```
tck_divpos:    ────┐┌────────
divpos_sync[0]:  ────┘┌───────  （延迟 1 拍）
divpos_sync[1]:      ────┘┌────  （延迟 2 拍）
divpos_sync[2]:          ────┘┌  （延迟 3 拍）
divpos_sync[3]:              ────┘┌（延迟 4 拍）
```

### 3. 边沿检测出单拍脉冲（`:94-97`）

```systemverilog
assign tck_rise_load  = tck_divpos_sync[2] ^ tck_divpos_sync[1];  // :94
assign tck_rise_reset = tck_divpos_sync[3] ^ tck_divpos_sync[2];  // :95
assign tck_fall_load  = tck_divneg_sync[2] ^ tck_divneg_sync[1];  // :96
assign tck_fall_reset = tck_divneg_sync[3] ^ tck_divneg_sync[2];  // :97
```

**原理**：相邻两拍异或 = 边沿检测。

```
divpos_sync[1]: ──┐┌──────
divpos_sync[2]: ────┘┌────
tck_rise_load:  ────┐─────  （单拍脉冲）
```

### 4. 脉冲锁存（`:99-155`）

#### Update 信号（`:99-109`）
```systemverilog
always_ff @(posedge clk, negedge pwrup_rst_n) begin
    if (tck_fall_load) begin
        tapcsync2core_ch_update_o <= tapc2tapcsync_ch_update_i;
    end else if (tck_fall_reset) begin
        tapcsync2core_ch_update_o <= '0;
    end
end
```

#### Capture/Shift/TDI 信号（`:111-152`）
```systemverilog
// TCK 下降沿采进 stage0
always_ff @(negedge tapc_tck, negedge tapc_trst_n) begin
    dmi_ch_capture_sync[0] <= tapc2tapcsync_ch_capture_i;
    dmi_ch_shift_sync[0]   <= tapc2tapcsync_ch_shift_i;
end

// SysCLK 同步 stage1/2
always_ff @(posedge clk, negedge pwrup_rst_n) begin
    dmi_ch_capture_sync[2:1] <= {dmi_ch_capture_sync[1], dmi_ch_capture_sync[0]};
end

// tck_rise_load 时装入
if (tck_rise_load) begin
    tapcsync2core_ch_capture_o <= dmi_ch_capture_sync[2];
    tapcsync2core_ch_shift_o   <= dmi_ch_capture_sync[2];
    tapcsync2core_ch_tdi_o     <= dmi_ch_tdi_sync[2];
end
```

#### ChID/ChSel 信号（`:157-177`）
```systemverilog
// 准静态信号，直接锁存
always_ff @(posedge clk, negedge dm_rst_n) begin
    if (tck_rise_load) begin
        tapcsync2dmi_ch_sel_o <= tapc2tapcsync_dmi_ch_sel_i;
        tapcsync2core_ch_id_o <= tapc2tapcsync_ch_id_i;
    end
end
```

### 5. TDO 直通（`:179`）

```systemverilog
assign tapc2tapcsync_ch_tdo_i = tapcsync2core_ch_tdo_o;
```

**为什么不同步？** 回程方向是 SysCLK → TCK，而 `scr1_tapc` 本身就在 **TCK 下降沿**驱动 TDO（`scr1_tapc.sv:326`/`:341`）。TCK 域自己采样自己的下降沿，天然无竞争。

## 关键信号

| 信号 | 方向 | 域 | 含义 |
|---|---|---|---|
| `tapc_tck` | in | TCK | JTAG 时钟 |
| `tapc_trst_n` | in | TCK | JTAG 复位 |
| `clk` | in | SysCLK | 系统时钟 |
| `pwrup_rst_n` | in | SysCLK | 上电复位 |
| `tapc2tapcsync_ch_capture_i` | in | TCK | capture 输入 |
| `tapcsync2core_ch_capture_o` | out | SysCLK | capture 输出（同步后） |
| `tapcsync2core_ch_tdo_o` | out | SysCLK | TDO 输出 |
| `tapc2tapcsync_ch_tdo_i` | in | TCK | TDO 输入（直通） |

## 代价

`tck_divpos` 翻转一次要 2 个 TCK 周期，加上 3~4 拍 SysCLK 同步，一次 capture 传到核里要**好几个 TCK 周期**。

**为什么可接受？** JTAG 本来就慢（MHz 量级），这点延迟无所谓。

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | 模块使能 |

## 配套文档

- 设计规格：`docs/scr1-tapc-synchronizer/design-spec.md`
- 验证文档：`docs/scr1-tapc-synchronizer/verification.md`
