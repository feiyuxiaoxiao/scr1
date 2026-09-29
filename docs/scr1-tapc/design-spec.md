# SCR1 TAPC TAP 控制器 设计规格（Design Specification）

- 模块：`scr1_tapc`（`src/core/scr1_tapc.sv`，457 行）
- 条件：仅在 `SCR1_DBG_EN` 时编译（`:28`、`:457`）
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:362-383`，例化名 `i_tapc`）
- 头文件：`src/includes/scr1_tapc.svh`（状态/指令/宽度，66 行）
- 职责：控制 JTAG TAP，按指令寄存器选择数据寄存器与 DMI/SCU scan-chain（`:9-11`）

---

## 1. 概述与职责

按文件头注释（`:8-23`），TAPC 包含：

1. 同步复位生成；
2. TAPC FSM（IEEE 1149.1 16 态）；
3. 指令寄存器（IR）；
4. DR/DMI/SCU scan-chain；
5. TDO 使能与输出寄存器；
6. 数据寄存器：BYPASS、IDCODE、BUILD ID。

### 关键点

- 全部时序逻辑工作在 **TCK** 域；
- 指令宽度 5 位（`:17`）；DR 宽度 IDCODE=32、BLD_ID=32、BYPASS=1（`:18-20`）；
- `SCR1_TAP_BLD_ID_VALUE = `SCR1_MIMPID`（`:22`，BUILD ID 内容）。

---

## 2. 端口（`:32-53`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `tapc_trst_n` | in | 34 | TRSTn |
| `tapc_tck` | in | 35 | TCK |
| `tapc_tms` | in | 36 | TMS |
| `tapc_tdi` | in | 37 | TDI |
| `tapc_tdo` | out | 38 | TDO |
| `tapc_tdo_en` | out | 39 | TDO 缓冲控制 |
| `soc2tapc_fuse_idcode_i[31:0]` | in | 42 | IDCODE fuse |
| `tapc2tapcsync_scu_ch_sel_o` | out | 45 | SCU 链选择 |
| `tapc2tapcsync_dmi_ch_sel_o` | out | 46 | DMI 链选择 |
| `tapc2tapcsync_ch_id_o[1:0]` | out | 47 | 链 ID |
| `tapc2tapcsync_ch_capture_o` | out | 48 | capture |
| `tapc2tapcsync_ch_shift_o` | out | 49 | shift |
| `tapc2tapcsync_ch_update_o` | out | 50 | update |
| `tapc2tapcsync_ch_tdi_o` | out | 51 | TDI 转发 |
| `tapcsync2tapc_ch_tdo_i` | in | 52 | 链 TDO 回读 |

---

## 3. 类型与常量（`scr1_tapc.svh`）

### 3.1 宽度参数（`:16-20`）

状态宽 4、指令宽 5、IDCODE/BLD_ID 宽 32、BYPASS 宽 1。

### 3.2 TAP 状态（`:27-48`）

16 态：`RESET, IDLE, DR_SEL_SCAN, DR_CAPTURE, DR_SHIFT, DR_EXIT1, DR_PAUSE,
DR_EXIT2, DR_UPDATE, IR_SEL_SCAN, IR_CAPTURE, IR_SHIFT, IR_EXIT1, IR_PAUSE,
IR_EXIT2, IR_UPDATE`（`SCR1_XPROP_EN` 时额外 `XXX`）。

### 3.3 指令编码（`:50-63`）

| 指令 | 编码 | 功能 |
|---|---|---|
| IDCODE | 0x01 | 读 IDCODE |
| BLD_ID | 0x04 | 读 BUILD ID |
| SCU_ACCESS | 0x09 | 访问 SCU |
| DTMCS | 0x10 | 访问 DTMCS（DMI 链 ID=1） |
| DMI_ACCESS | 0x11 | 访问 DMI（DMI 链 ID=2） |
| BYPASS | 0x1F | 旁路 |

---

## 4. 同步复位（`:119-129`）

`trst_n_int` 在 `negedge tck` 更新：`~trst_n → 0`，否则 `~tap_fsm_reset`
（`:123-129`）。即 TRST 后保持“软复位”直到 FSM 进入 RESET 态。

---

## 5. TAP FSM（`:131-172`）

- 状态寄存器：`posedge tck`，`~trst_n` 复位到 RESET（`:135-141`）；
- 状态转移（`:143-167`）：标准 1149.1 状态图，按 TMS 在
  RESET/IDLE/DR-SEL-SCAN/DR-CAPTURE/DR-SHIFT/DR-EXIT1/DR-PAUSE/DR-EXIT2/DR-UPDATE
  与 IR 对应状态间迁移；`SCR1_XPROP_EN` 时 default 为 `XXX`，否则保持；
- 状态判定：`reset`/`ir_upd`/`ir_cap`/`ir_shft`（`:169-172`）。

---

## 6. 指令寄存器（`:174-208`）

- **IR shift reg**（`:181-193`）：`~trst_n` 或 `~trst_n_int` 清 0；
  capture 载入 `{0…0, 1'b1}`（最低位 1）；shift 载入 `{tdi, ff[4:1]}`；
- **IR**（`:198-208`）：复位为 IDCODE（`:200`/`:202`）；`IR_UPDATE` 时载入 shift 值
  （`:208`）。

---

## 7. 寄存器化控制信号（`:210-260`）

在 `posedge tck` 采样，且在 `~trst_n_int` 时清 0：

| 信号 | 条件（基于 `tap_fsm_next`） | 行号 |
|---|---|---|
| `tap_fsm_ir_shift_ff` | `next==IR_SHIFT` | 214-224 |
| `tap_fsm_dr_capture_ff` | `next==DR_CAPTURE` | 226-236 |
| `tap_fsm_dr_shift_ff` | `next==DR_SHIFT` | 238-248 |
| `tap_fsm_dr_update_ff` | `next==DR_UPDATE` | 250-260 |

---

## 8. DR/DMI/SCU scan-chain 与解码（`:262-317`）

### 8.1 选择译码（`:274-289`）

按 `tap_ir_ff`：DTMCS/DMI_ACCESS→`dmi_ch_sel`；IDCODE→`dr_idcode_sel`；
BYPASS→`dr_bypass_sel`；BLD_ID→`dr_bld_id_sel`；SCU_ACCESS→`scu_ch_sel`；
default→bypass。

### 8.2 链 ID 译码（`:294-301`）

DTMCS→1、DMI_ACCESS→2、其余→0（供 DMI/SCU 识别）。

### 8.3 读数据 mux（`:306-317`）

DTMCS/DMI_ACCESS/SCU_ACCESS→`tapcsync2tapc_ch_tdo_i`；IDCODE/BYPASS/BLD_ID→各 DR TDO；
default→bypass。

---

## 9. TDO 使能与输出（`:319-357`）

- `tdo_en_next = dr_shift_ff | ir_shift_ff`（`:336`），`negedge tck` 更新（`:326-334`）；
- `tdo_out_next = dr_shift_ff ? dr_out : ir_shift_ff ? ir_shift_ff[0] : 1'b0`（`:351-353`），
  `negedge tck` 更新（`:341-349`）；
- `tapc_tdo_en = tdo_en_ff`、`tapc_tdo = tdo_out_ff`（`:356-357`）。

---

## 10. 数据寄存器（`:359-426`）

均例化 `scr1_tapc_shift_reg`（`src/core/scr1_tapc_shift_reg.sv`）：

| DR | 宽度 | din_parallel | 行号 |
|---|---|---|---|
| BYPASS | 1 | 0 | 372-386 |
| IDCODE | 32 | `soc2tapc_fuse_idcode_i` | 392-406 |
| BUILD ID | 32 | `SCR1_TAP_BLD_ID_VALUE`(`SCR1_MIMPID`) | 412-426 |

### DMI/SCU 链信号转发（`:428-435`）

`ch_tdi = tapc_tdi`；`ch_capture/shift/update = tap_fsm_dr_*_ff`。

---

## 11. 断言（`SCR1_TRGT_SIMULATION`，`:437-453`）

| 断言 | 行号 | 检查 |
|---|---|---|
| `SCR1_SVA_TAPC_XCHECK` | 443-446 | posedge tck 时 tms/tdi 无 X |
| `SCR1_SVA_TAPC_XCHECK_NEGCLK` | 448-451 | DR_SHIFT 时链 TDO 无 X |

---

## 12. 附录

### 12.1 配置宏

- `SCR1_DBG_EN`（`:28`）；
- `SCR1_XPROP_EN`：影响 FSM/指令枚举的 `XXX` 项与 default（`:161-165`、`tapc.svh:44-62`）。

### 12.2 职责边界

- TAPC 只做 JTAG 协议状态机与链选择，不解析 DMI/SCU 语义；
- 全部寄存器在 TCK 域，与 core 时钟域经 `scr1_tapc_synchronizer` 隔断
  （`scr1_core_top.sv:385-413`）；
- TDO 为寄存输出，`tapc_tdo_en` 控制外部三态缓冲。
