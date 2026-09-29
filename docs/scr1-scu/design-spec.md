# SCR1 SCU 系统控制单元 设计规格（Design Specification）

- 模块：`scr1_scu`（`src/core/scr1_scu.sv`，516 行）
- 条件：仅在 `SCR1_DBG_EN` 时编译（`:32`、`:515`）
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:193-231`，例化名 `i_scu`）
- 头文件：`src/includes/scr1_scu.svh`（类型与位域，84 行）
- 职责：生成 System/Core/HDU/DM 复位及其 qualifier，暴露调试器可写复位的 CSR

---

## 1. 概述与职责

按文件头注释（`:6-27`），SCU 负责：

1. **生成四路复位**：System、Core、HDU、DM 及其状态/qualifier（`:9`）；
2. **软件复位**：调试器可经 CSR 触发 System/Core 复位（`:10`）；
3. **行为配置**：MODE 寄存器设置 DM/HDU 复位行为（`:11`）；
4. **状态观测**：STATUS 与 STICKY_STATUS（`:12`）。

### 结构（`:14-26`）

TAPC scan-chain 接口、SCU CSR 读写接口、SCU CSRs（CONTROL/MODE/STATUS/STICKY_STATUS）、
复位逻辑（System/Core/DM/HDU）。

---

## 2. 端口（`:34-72`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `pwrup_rst_n` | in | 36 | Power-Up 复位 |
| `rst_n` / `cpu_rst_n` | in | 37/38 | 常规复位 / CPU 复位 |
| `test_mode` / `test_rst_n` | in | 39/40 | DFT |
| `clk` | in | 41 | 时钟 |
| `tapcsync2scu_ch_sel_i` / `_id_i` | in | 44/45 | chain 选择 / ID |
| `tapcsync2scu_ch_capture/shift/update_i` | in | 46-48 | chain 控制 |
| `tapcsync2scu_ch_tdi_i` | in | 49 | TDI |
| `scu2tapcsync_ch_tdo_o` | out | 50 | TDO |
| `ndm_rst_n_i` | in | 53 | 来自 DM 的非 DM 复位 |
| `hart_rst_n_i` | in | 54 | 来自 DM 的 HART 复位 |
| `sys_rst_n_o` | out | 57 | 系统/簇复位 |
| `core_rst_n_o` | out | 58 | 核心复位 |
| `dm_rst_n_o` | out | 59 | DM 复位 |
| `hdu_rst_n_o` | out | 60 | HDU 复位 |
| `sys_rst_status_o` / `core_rst_status_o` | out | 63/64 | 复位状态（同步到 POR 域） |
| `sys_rdc_qlfy_o` / `core_rdc_qlfy_o` | out | 67/68 | 到外部 SOC 的 RDC qualifier |
| `core2hdu_rdc_qlfy_o` / `core2dm_rdc_qlfy_o` | out | 69/70 | core 到 HDU/DM 的 RDC |
| `hdu2dm_rdc_qlfy_o` | out | 71 | HDU 到 DM 的 RDC |

---

## 3. SCU CSR 结构（`scr1_scu.svh`）

### 3.1 数据寄存器（DR）格式（`:16-18`、`:45-49`）

`op(2) + addr(2) + data(4) = 8 位`（`SCR1_SCU_DR_SYSCTRL_{OP,ADDR,DATA}_WIDTH=2/2/4`）。

### 3.2 op 与 addr 编码（`:23-43`）

| op | 值 | addr | 值 |
|---|---|---|---|
| WRITE | 0 | CONTROL | 0 |
| READ | 1 | MODE | 1 |
| SETBITS | 2 | STATUS | 2 |
| CLRBITS | 3 | STICKY | 3 |

（`SCR1_XPROP_EN` 时额外追加 `'X` 项）

### 3.3 寄存器类型（`:64-81`）

| 类型 | 字段 | 行号 |
|---|---|---|
| `control_reg` | `{rsrv[1:0], core_reset, sys_reset}` | 64-68 |
| `mode_reg` | `{rsrv[1:0], hdu_rst_bhv, dm_rst_bhv}` | 70-74 |
| `status_reg` | `{hdu_reset, dm_reset, core_reset, sys_reset}` | 76-81 |

局部参数：`SCR1_SCU_RST_SYNC_STAGES_NUM = 2`（`:77`）。

---

## 4. TAPC scan-chain 接口（`:158-206`）

- `scu_csr_req = sel & (ch_id == 0)`（`:171`，SCU CSR 链 ID=0）；
- `cap/shft/upd_req = scu_csr_req & ch_*`（`:172-174`）；
- 移位寄存器 `tapc_shift_ff`：`pwrup_rst_n_sync` 复位，`cap|shft` 更新（`:181-187`）；
  `tapc_shift_next = cap ? shadow : shft ? {tdi, ff[7:1]} : ff`（`:189-191`）；
- 影子寄存器 `tapc_shadow_ff`：update 时拷贝 `op`/`addr`，并把 `data` 置为 `scu_csr_wdata`
  （`:196-204`）——即回读时用 CSR 读数据回填；
- `scu2tapcsync_ch_tdo_o = tapc_shift_ff[0]`（`:206`）。

---

## 5. CSR 读写接口（`:208-262`）

### 5.1 写请求译码（`:216-229`）

`upd_req && op != READ` 时按 addr：CONTROL/MODE 写使能；STICKY 仅在 `op==CLRBITS`
且更新时写使能（`:225`）。

### 5.2 写数据构造（`:232-244`）

| op | 写数据 |
|---|---|
| WRITE | `data` |
| READ | `scu_csr_rdata` |
| SETBITS | `scu_csr_rdata \| data` |
| CLRBITS | `scu_csr_rdata & ~data` |

### 5.3 读数据 mux（`:250-262`）

按 addr 选 CONTROL/MODE/STATUS/STICKY，`default` 为 `'x`（`:259`）。

---

## 6. SCU CSRs（`:264-335`）

- **CONTROL**（`:279-285`）：`pwrup_rst_n_sync` 清 0，写请求时载入 `scu_csr_wdata`。
  用于调试器生成 System/Core 复位。
- **MODE**（`:291-297`）：清 0，写请求时载入。设置 DM/HDU 复位行为。
- **STATUS**（`:303-306`）：
  - `sys_reset = sys_rst_status_o`、`core_reset = core_rst_status_o`（`:303-304`）；
  - `dm_reset = ~dm_rst_n_status`、`hdu_reset = ~hdu_rst_n_status_sync`（`:305-306`）。
- **STATUS 上升沿检测**（`:309-317`）：`status_ff_dly` 打一拍，
  `posedge = status_ff & ~status_ff_dly`。
- **STICKY_STATUS**（`:323-335`）：每 bit 在 `posedge` 时置 1；否则若 `wr_req`
  （CLRBITS）则载入 `scu_csr_wdata[i]`。即“自上次清除以来是否发生过复位”。

---

## 7. 复位逻辑（`:337-464`）

### 7.1 输入同步（`:348-355`）

`pwrup_rst_n_sync`/`rst_n_sync`/`cpu_rst_n_sync` 直通（假定同步）；
`sys_reset_n = ~control.sys_reset`、`core_reset_n = ~control.core_reset`（`:354-355`）。

### 7.2 System 复位（`:357-385`）

- `sys_rst_n_in = sys_reset_n & ndm_rst_n_i & rst_n_sync`（`:371`）；
- `scr1_reset_qlfy_adapter_cell_sync`（`:360-369`）产生 `sys_rst_n_o`/`sys_rst_n_qlfy`/`sys_rst_n_status`；
- `scr1_data_sync_cell`（2 级，`:373-380`）同步状态；
- `sys_rst_status_o = ~sys_rst_n_status_sync`（`:382`）；
- `sys_rdc_qlfy_o = sys_rst_n_qlfy`（`:385`）。

### 7.3 Core 复位（`:387-420`）

- `core_rst_n_in_sync = sys_rst_n_in & hart_rst_n_i & core_reset_n & cpu_rst_n_sync`（`:401`）；
- 经 adapter（`:390-399`）与 2 级状态同步（`:403-410`）；
- `core_rst_status_o = ~core_rst_n_status_sync`（`:412`）；
- **三个 qualifier 同源**：`core_rdc_qlfy_o = core2hdu_rdc_qlfy_o = core2dm_rdc_qlfy_o = core_rst_n_qlfy`（`:416-420`）。

### 7.4 HDU 复位（`:422-449`）

- `hdu_rst_n_in_sync = mode.hdu_rst_bhv | core_rst_n_in_sync`（`:436`）；
- 经 adapter（`:425-434`）与 2 级状态同步（`:438-445`）；
- `hdu2dm_rdc_qlfy_o = hdu_rst_n_qlfy`（`:449`）。

### 7.5 DM 复位（`:451-464`）

- `dm_rst_n_in = ~mode.dm_rst_bhv | sys_reset_n`（`:464`）；
- 经 `scr1_reset_buf_cell`（`:454-462`，非 qualifier adapter）产生 `dm_rst_n_o`/`dm_rst_n_status`。

### 7.6 复位行为语义

- MODE 位为 0 时，对应复位与系统/核心复位联动；
- MODE 位为 1 时，对应复位的 `reset_n_in` 恒 1（被屏蔽），仅受 `pwrup`/`test` 影响，
  即该域可跨系统/核心复位保持。

---

## 8. 断言（`SCR1_TRGT_SIMULATION`，`:466-512`）

| 断言 | 行号 | 检查 |
|---|---|---|
| `SCR1_SVA_SCU_RESETS_XCHECK` | 481-484 | 输入复位无 X |
| `SCR1_SVA_SCU_SYS2SOC_QLFY_CHECK` | 487-490 | `sys_rst_n_o` 下降沿前 `sys_rdc_qlfy_o` 已下降 |
| `SCR1_SVA_SCU_CORE2SOC_QLFY_CHECK` | 492-495 | core 同 sys |
| `SCR1_SVA_SCU_CORE2HDU_QLFY_CHECK` | 497-500 | core→hdu |
| `SCR1_SVA_SCU_CORE2DM_QLFY_CHECK` | 502-505 | core→dm |
| `SCR1_SVA_SCU_HDU2DM_QLFY_CHECK` | 507-510 | hdu→dm |

非 Verilator 时用 `$assertoff`/`$asserton` 屏蔽前两拍（`:471-478`）。

---

## 9. 附录

### 9.1 复位输入来源汇总

| 复位 | 输入表达式 | 行号 |
|---|---|---|
| sys | `~control.sys_reset & ndm_rst_n & rst_n` | 371 |
| core | `sys_in & hart_rst_n & ~control.core_reset & cpu_rst_n` | 401 |
| hdu | `mode.hdu_rst_bhv \| core_in_sync` | 436 |
| dm | `~mode.dm_rst_bhv \| ~control.sys_reset` | 464 |

### 9.2 职责边界

- SCU 使用 `pwrup_rst_n_sync` 作为自身寄存器的复位源（而非 core/sys 复位）；
- 所有 qualifier 由 adapter cell 在同域生成，保证“qualifier 先于复位变化”；
- DM 复位用 buffer cell（无 qualifier），HDU/Core/System 用 qualifier adapter cell。
