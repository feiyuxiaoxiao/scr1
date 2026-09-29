# SCR1 DMI 调试模块接口 设计规格（Design Specification）

- 模块：`scr1_dmi`（`src/core/scr1_dmi.sv`，182 行）
- 条件：仅在 `SCR1_DBG_EN` 时编译（`:19`、`:182`）
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:417-437`，例化名 `i_dmi`）
- 头文件：`src/includes/scr1_dm.svh`
- 职责：为 TAPC 提供访问 Debug Module（DM）与 DTMCS 的通道（`:9`）

---

## 1. 概述与职责

DMI 是 JTAG TAP 与 DM 之间的串行寄存器桥：

1. **DMI↔TAP 接口**（`:11`/`:94-144`）：由 `tapcsync2dmi_ch_*` 控制，维护 41 位
   DMI access 数据寄存器与 32 位 DTMCS 数据寄存器，串行移位并输出 TDO；
2. **DMI↔DM 接口**（`:13`/`:146-178`）：在 update 时把 op/addr/data 解码为
   `dmi2dm_req/wr/addr/wdata`，并缓存 DM 读回数据。

### 结构要点

- **两个 chain**：`ch_id==1` 为 DTMCS（`:101`），`ch_id==2` 为 DMI access（`:150-151`）；
- **op 解码**：`req = op!=2'b00`；`wr = op==2'b10`（`:160-161`）；
- **DTMCS 常量**：ABITS=`DMI_ADDR_WIDTH`(7)、VERSION 低位=1，其余 0（`:107-115`）；
- **TDO**：始终取数据寄存器最低位（`:144`）。

---

## 2. 端口（`:22-43`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `rst_n` / `clk` | in | 24/25 | 复位/时钟 |
| `tapcsync2dmi_ch_sel_i` | in | 28 | chain 选择 |
| `tapcsync2dmi_ch_id_i[1:0]` | in | 29 | chain ID |
| `tapcsync2dmi_ch_capture_i` | in | 30 | capture |
| `tapcsync2dmi_ch_shift_i` | in | 31 | shift |
| `tapcsync2dmi_ch_update_i` | in | 32 | update |
| `tapcsync2dmi_ch_tdi_i` | in | 33 | TDI |
| `dmi2tapcsync_ch_tdo_o` | out | 34 | TDO |
| `dm2dmi_resp_i` | in | 37 | DM 响应 |
| `dm2dmi_rdata_i[31:0]` | in | 38 | DM 读数据 |
| `dmi2dm_req_o` | out | 39 | DMI 请求 |
| `dmi2dm_wr_o` | out | 40 | DMI 写 |
| `dmi2dm_addr_o[6:0]` | out | 41 | DMI 地址 |
| `dmi2dm_wdata_o[31:0]` | out | 42 | DMI 写数据 |

---

## 3. 局部参数与信号

### 3.1 DTMCS 字段偏移（`:52-64`）

`RESERVEDB[31:18]`、`DMIHARDRESET[17]`、`DMIRESET[16]`、`RESERVEDA[15]`、
`IDLE[14:12]`、`DMISTAT[11:10]`、`ABITS[9:4]`、`VERSION[3:0]`。

### 3.2 DMI access 字段位置（`:69-74`）

按 `SCR1_DBG_DMI_OP_WIDTH=2`、`DATA_WIDTH=32`、`ADDR_WIDTH=7` 计算：

| 字段 | 位域 | 值 |
|---|---|---|
| OP | `[1:0]` | 69-70 |
| DATA | `[33:2]` | 71-72 |
| ADDR | `[40:34]` | 73-74 |

总宽 `SCR1_DBG_DMI_DR_DMI_ACCESS_WIDTH = 41`（`scr1_dm.svh:19-21`）。

### 3.3 局部信号（`:80-92`）

`tap_dr_upd`、`tap_dr_ff/shift/rdata/next`（41 位）、`dm_rdata_upd`、`dm_rdata_ff`（32 位）、
`tapc_dmi_access_req`、`tapc_dtmcs_sel`。

---

## 4. DMI ↔ TAP 接口（`:94-144`）

### 4.1 chain 选择与读数据 mux（`:101-121`）

- `tapc_dtmcs_sel = (ch_id==1)`（`:101`）；
- **DTMCS 分支**：RESERVED/DMIHARDRESET/DMIRESET/RESERVEDA/IDLE/DMISTAT=0，
  `ABITS = SCR1_DBG_DMI_ADDR_WIDTH`，`VERSION[0]=1`（`:107-115`）；
- **DMI 分支**：ADDR/OP 置 0，`DATA = dm_rdata_ff`（`:116-120`）。

### 4.2 移位（`:123-125`）

```
dtmcs: {9'b0, tdi, tap_dr_ff[31:1]}
dmi  : {tdi, tap_dr_ff[40:1]}
```

### 4.3 数据寄存器更新（`:127-142`）

`tap_dr_upd = capture | shift`（`:130`）；
`tap_dr_next`：capture→`tap_dr_rdata`，shift→`tap_dr_shift`，否则保持（`:140-142`）。
`dmi2tapcsync_ch_tdo_o = tap_dr_ff[0]`（`:144`）。

---

## 5. DMI ↔ DM 接口（`:146-178`）

### 5.1 访问请求（`:150-151`）

`tapc_dmi_access_req = update & sel & (ch_id==2)`。

### 5.2 请求解码（`:153-165`）

默认全部为 0；当访问请求有效时：

- `dmi2dm_req_o = (op != 2'b00)`（`:160`）；
- `dmi2dm_wr_o = (op == 2'b10)`（`:161`）；
- `dmi2dm_addr_o = tap_dr_ff[ADDR]`、`dmi2dm_wdata_o = tap_dr_ff[DATA]`（`:162-163`）。

### 5.3 读数据缓存（`:167-178`）

`dm_rdata_upd = dmi2dm_req_o & dm2dmi_resp_i & ~dmi2dm_wr_o`（`:170`），
满足时 `dm_rdata_ff <= dm2dmi_rdata_i`（`:175-177`）。

---

## 6. 附录

### 6.1 配置宏

仅 `SCR1_DBG_EN`（`:19`）。MAX 配置定义之（`scr1_arch_description.svh:79`）。

### 6.2 职责边界

- DMI 不解析 DM 寄存器语义，只搬运 op/addr/data；
- 无内建断言；
- 读时序：OCD 先写 DMI access（op=读），DM 响应后数据存入 `dm_rdata_ff`，
  下一次读 DMI 时经 `tap_dr_rdata` 移位返回。
