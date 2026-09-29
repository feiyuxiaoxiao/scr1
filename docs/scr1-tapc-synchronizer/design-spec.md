# SCR1 TAPC 同步器设计规格（Design Specification）

- 模块：`scr1_tapc_synchronizer`（`src/core/scr1_tapc_synchronizer.sv`，183 行；`SCR1_DBG_EN` 保护 `:8`/`:183`）
- 职责：在 TCK 域与系统时钟域之间同步 DMI/SCU 扫描链控制信号

---

## 1. 概述与职责

JTAG 的 TAP 在 `tck` 域产生链选择与捕获/移位/更新信号，核心在 `clk` 域消费。
本模块用分频沿检测产生 `load`/`reset` 脉冲，将信号安全跨域：

1. **TCK 沿分频**：`tck_divpos` 在 `posedge tck` 翻转、`tck_divneg` 在 `negedge tck` 翻转（`:68-82`）；
2. **系统域 4 级同步**：`tck_divpos_sync/tck_divneg_sync`（`:84-92`）；
3. **沿脉冲**：`tck_rise_load/reset`、`tck_fall_load/reset`（`:94-97`）；
4. **TCK→sys 传递**：capture/shift 在 TCK 下降沿采样后 2 级同步，tdi 3 级同步（`:111-137`）；
5. **sys 输出锁存**：在 `rise_load` 装载、`rise_reset` 清零（`:139-177`）。

依赖头文件 `scr1_tapc.svh`、`scr1_dm.svh`（`:9-10`）。

---

## 2. 端口（`:12-46`）

- 系统侧：`pwrup_rst_n`、`dm_rst_n`、`clk`；
- JTAG 侧：`tapc_trst_n`、`tapc_tck`；
- 链选择：`scu_ch_sel`、`dmi_ch_sel`（TCK→sys）；
- 链标识：`ch_id[SCR1_DBG_DMI_CH_ID_WIDTH-1:0]`；
- 链控制：`ch_capture`、`ch_shift`、`ch_update`；
- 数据：`ch_tdi`（TCK→sys）、`ch_tdo`（sys→TCK）。

---

## 3. 脉冲产生（`:64-97`）

| 信号 | 定义 | 行号 |
|---|---|---|
| `tck_rise_load` | `pos_sync[2] ^ pos_sync[1]` | 94 |
| `tck_rise_reset` | `pos_sync[3] ^ pos_sync[2]` | 95 |
| `tck_fall_load` | `neg_sync[2] ^ neg_sync[1]` | 96 |
| `tck_fall_reset` | `neg_sync[3] ^ neg_sync[2]` | 97 |

---

## 4. 各输出锁存（`:99-177`）

| 信号 | 采样域 | 装载/清零 | 行号 | 复位 |
|---|---|---|---|---|
| `ch_update_o` | sys | `fall_load`/`fall_reset` | 99-109 | `pwrup_rst_n` |
| `ch_capture_o`/`ch_shift_o`/`ch_tdi_o` | sys | `rise_load`/`rise_reset` | 139-155 | `pwrup_rst_n` |
| `dmi_ch_sel_o`/`ch_id_o` | sys | `rise_load` | 157-167 | `dm_rst_n` |
| `scu_ch_sel_o` | sys | `rise_load` | 169-177 | `pwrup_rst_n` |

中间同步：capture/shift 先在 `negedge tck` 采到 `[0]`（`:111-119`），再 2 级 sys 同步（`:121-129`）；
`tdi` 3 级 sys 同步（`:131-137`）。

`tapc2tapcsync_ch_tdo_i = tapcsync2core_ch_tdo_o`（`:179`）。

---

## 5. 附录

### 5.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_DBG_EN` | 模块是否存在 | `:8`/`:183` |

### 5.2 职责边界

- 仅跨时钟域同步，无协议解码；
- 复位域区分：pwrup（多数）、dm（链选择/ID）；
- 无 SVA。
