# SCR1 Debug Module 调试模块 设计规格（Design Specification）

- 模块：`scr1_dm`（`src/core/scr1_dm.sv`，1427 行）
- 条件：仅在 `SCR1_DBG_EN` 时编译（`:42`、`:1427`）
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:457-495`，例化名 `i_dm`）
- 头文件：`src/includes/scr1_dm.svh`（寄存器地址与字段偏移，141 行）
- 职责：实现 RISC-V Debug 规范的 Debug Module，作为调试器与 HART 之间的桥梁

---

## 1. 概述与职责

按文件头注释（`:6-37`），DM 提供：

1. **系统复位**：调试器可通过 DMCONTROL.ndmreset 复位系统（`:9`）；
2. **HART 状态控制**：halt/resume 命令经 DHI 接口下发给流水线 HDU（`:10`）；
3. **HART 状态观测**：DMSTATUS 反映 halt/resume/reset 状态（`:11`）；
4. **抽象命令（Abstract Command）**：访问 MPRF、CSR、内存（`:12-15`）；
5. **抽象命令状态**：busy 标志与错误码 cmderr（`:16`）；
6. **程序缓冲区（Program Buffer）**：在被 halt 的 HART 上执行小程序（`:17-18`）。

### 内部结构（`:20-37`）

DM↔DMI 接口、DM 寄存器（DMCONTROL/DMSTATUS）、抽象命令控制逻辑、抽象命令 FSM、
抽象命令状态逻辑、抽象指令逻辑、抽象寄存器（COMMAND/ABSTRACTAUTO/PROGBUF0-5/DATA0-1）、
DHI FSM、HART 命令寄存器、DHI 接口。

### 三个 FSM

| FSM | 类型 | 状态数 | 定义 |
|---|---|---|---|
| Abstract Command FSM | `type_scr1_abs_fsm_e` | 13 | `:90-104` |
| DHI FSM | `type_scr1_dhi_fsm_e` | 7 | `:106-114` |
| 抽象错误码 | `type_scr1_abs_err_e` | 5 | `:116-122` |

---

## 2. 端口（`:46-84`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `rst_n` / `clk` | in | 48/49 | DM 复位/时钟 |
| `dmi2dm_req_i` / `_wr_i` | in | 52/53 | DMI 请求/写 |
| `dmi2dm_addr_i[6:0]` | in | 54 | DMI 地址 |
| `dmi2dm_wdata_i[31:0]` | in | 55 | DMI 写数据 |
| `dm2dmi_resp_o` | out | 56 | DMI 响应（恒 1，`:555`） |
| `dm2dmi_rdata_o[31:0]` | out | 57 | DMI 读数据 |
| `ndm_rst_n_o` / `hart_rst_n_o` | out | 60/61 | 非 DM 复位 / HART 复位 |
| `dm2pipe_active_o` | out | 62 | DM active 标志 |
| `dm2pipe_cmd_req_o` | out | 63 | 命令请求 |
| `dm2pipe_cmd_o`（`type_scr1_hdu_dbgstates_e`） | out | 64 | 命令（下一状态） |
| `pipe2dm_cmd_resp_i` / `_rcode_i` | in | 65/66 | 命令响应 / 返回码（0=Ok,1=Error） |
| `pipe2dm_hart_event_i` | in | 67 | HART 事件 |
| `pipe2dm_hart_status_i`（`type_scr1_hdu_hartstatus_s`） | in | 68 | HART 状态 |
| `soc2dm_fuse_mhartid_i[XLEN-1:0]` | in | 70 | MHARTID |
| `pipe2dm_pc_sample_i[XLEN-1:0]` | in | 71 | PC 采样 |
| `pipe2dm_pbuf_addr_i[PBUF_ADDR_WIDTH-1:0]` | in | 74 | PBUF 地址 |
| `dm2pipe_pbuf_instr_o[INSTR_WIDTH-1:0]` | out | 75 | PBUF 指令 |
| `pipe2dm_dreg_req_i` / `_wr_i` / `_wdata_i` | in | 78-80 | 数据寄存器请求/写/写数据 |
| `dm2pipe_dreg_resp_o` / `_fail_o` / `_rdata_o` | out | 81-83 | 数据寄存器响应/失败/读数据 |

---

## 3. 局部类型与参数

### 3.1 抽象命令 FSM（`:90-104`）

`IDLE, ERR, EXEC, XREG_RW, MEM_SAVE_XREG, MEM_SAVE_XREG_FORADDR, MEM_RW,
MEM_RETURN_XREG, MEM_RETURN_XREG_FORADDR, CSR_RO, CSR_SAVE_XREG, CSR_RW, CSR_RETURN_XREG`。

### 3.2 DHI FSM（`:106-114`）

`IDLE, EXEC, EXEC_RUN, EXEC_HALT, HALT_REQ, RESUME_REQ, RESUME_RUN`。

### 3.3 错误码（`:116-122`）

`ABS_ERR_NONE/BUSY/CMD/EXCEPTION/NOHALT = 0..4`（宽度 `CMDERR_WDTH+1`）。

### 3.4 抽象指令字段常量（`:130-142`）

opcode：`SYSTEM=1110011`、`LOAD=0000011`、`STORE=0100011`；
funct3：CSRRW=001、CSRRS=010、SB/SW/LW 等（`:135-142`）。

### 3.5 寄存器常量（`:144-192`）

- DMCONTROL 固定位（HARTRESET/HASEL/HARTSEL* /RESERVED=0，`:146-151`）；
- DMSTATUS：IMPEBREAK=1、AUTHENTICATED=1、VERSION=2、其余 unavailable/nexist=0（`:155-166`）；
- HARTINFO：NSCRATCH=1、DATASIZE=1、DATAADDR=12'h7b2（`:170-175`）；
- ABSTRACTCS：PROGBUFSIZE=6、DATACOUNT=2（`:179-184`）；
- 命令类型：`HARTREG=0`、`HARTMEM=2`；regfile `INT=0`；`ABS_EXEC_EBREAK`（`:186-192`）。

---

## 4. 寄存器地址映射（`scr1_dm.svh`）

宽度：`DMI_ADDR=7`、`DMI_DATA=32`、`DMI_OP=2`、`CH_ID=2`（`:13-17`）。

| 名称 | 地址 | 行号 |
|---|---|---|
| DATA0 / DATA1 | 0x04 / 0x05 | 24/25 |
| DMCONTROL / DMSTATUS / HARTINFO | 0x10 / 0x11 / 0x12 | 26-28 |
| ABSTRACTCS / COMMAND / ABSTRACTAUTO | 0x16 / 0x17 / 0x18 | 29-31 |
| PROGBUF0..5 | 0x20..0x25 | 32-37 |
| HALTSUM0 | 0x40 | 38 |

字段偏移：DMCONTROL（`:41-54`）、DMSTATUS（`:57-79`）、COMMAND 类型/accessreg/accessmem
（`:82-107`）、ABSTRACTCS（`:110-125`）、HARTINFO（`:128-138`）。

---

## 5. DM ↔ DMI 接口（`:427-572`）

### 5.1 寄存器选择（`:434-448`）

`dmi_req_<name> = dmi2dm_req_i & (dmi2dm_addr_i == SCR1_DBG_<NAME>)`；
`dmi_rpt_command = abs_autoexec_ff & dmi_req_data0`（autoexec 写 DATA0 触发重放，`:441`）。

### 5.2 请求聚合（`:450-453`）

`dmi_req_any` = 所有寄存器请求的 OR（含 command/rpt/progbuf/data）。

### 5.3 读数据 mux（`:459-552`）

按 `dmi2dm_addr_i` 选择；DMSTATUS（`:463-487`）与 DMCONTROL（`:489-504`）逐字段组装，
ABSTRACTCS（`:506-521`）、HARTINFO（`:523-535`）为常量 + 动态位；
ABSTRACTAUTO 读 `abs_autoexec_ff`（`:537`）；DATA0/1、PROGBUF0-5 直读（`:538-545`）；
HALTSUM0 读 halted 标志（`:546`）；default 返回 0（`:548-550`）。

### 5.4 响应与写请求（`:555-572`）

`dm2dmi_resp_o = 1'b1`（`:555`，始终应答）；
写请求 = `dmi_req_<name> & dmi2dm_wr_i`（`:560-572`），另有 `dreg_wr_req = pipe2dm_dreg_req_i & pipe2dm_dreg_wr_i`（`:563`）。

### 5.5 HART 状态解码（`:577-580`）

`hart_state_reset/run/dhalt/drun` 由 `pipe2dm_hart_status_i.dbg_state` 比较得到。

---

## 6. DM 寄存器（`:582-694`）

### 6.1 时钟使能（`:595-605`）

`clk_en_dm = dmcontrol_wr_req | dmcontrol_dmactive_ff | clk_en_dm_ff`（`:595`）；
`clk_en_dm_ff` 采样 `dmactive_ff`；`dm2pipe_active_o = clk_en_dm_ff`（`:605`）。
即 DM active 后仍保持一拍使能以便复位清场。

### 6.2 DMCONTROL（`:610-650`）

- 复位清 0 dmactive/ndmreset/ackhavereset/haltreq/resumereq（`:611-616`）；
- `dmactive_next` 仅由写决定（`:626-628`）；
- 其余位：dmactive=0 时清 0（`:635-639`），否则写时更新（`:640-645`）；
- **复位输出**：`hart_rst_n_o = ndm_rst_n_o = ~dmcontrol_ndmreset_ff`（`:649-650`，两者同源）。

### 6.3 `havereset_skip_pwrup`（`:655-665`）

复位为 1，dmactive 后首次观察到 hart 处于 reset 且两个复位释放时清 0（`:663-665`）；
用于区分“上电复位”与“运行中复位”。

### 6.4 DMSTATUS（`:670-694`）

- `allany_havereset`：dmactive=0→0；未跳过上电且 hart reset→1；ackhavereset→0；否则保持（`:682-685`）；
- `allany_resumeack`：resumereq 有效且 hart 运行→1，否则按条件清（`:686-689`）；
- `allany_halted`：hart dhalt→1，hart run→0，否则保持（`:691-694`）。

---

## 7. 抽象命令控制逻辑（`:696-802`）

### 7.1 解码（`:710-748`）

`abs_cmd = dmi_req_command ? dmi2dm_wdata_i : abs_command_ff`（`:710`，写优先，否则用已存 COMMAND）。
字段：`regno[11:0]`（`:713`）、`type[31:24]`（`:722`）、`regacs`=transfer（`:723`）、
`regtype[15:12]`（`:724`）、`regfile[11:5]`（`:725`）、`regsize[22:20]`（`:726-727`）、
`regwr`=write（`:728`）、`execprogbuf`=postexec（`:729`）；
`regvalid` = reservedB/reservedA 均为 0（`:731-732`）；
`memsize[22:20]`、`memwr`、`memvalid`（`:734-743`）。

- `abs_cmd_csr_ro`：regno ∈ {MISA, MVENDORID, MARCHID, MIMPID, MHARTID, DPC}（`:715-720`）；
- `abs_reg_access_csr = (regtype==HARTREG_CSR)`；`abs_reg_access_mprf = (regtype==INTFPU)&(regfile==INT)`（`:746-748`）。

### 7.2 有效标志与请求（`:753-771`）

`regsize_vd = (regsize==2)`（32 位）；`memsize_vd = (memsize<3)`；
`hartreg_vd`/`hartmem_vd` 由 type + 有效位（`:756-757`）；
`abs_cmd_csr_ro_access_vd` 额外要求 `hart_state_run`（`:766-767`，只读 CSR 可在运行态访问）；
`mprf_access_vd`/`mem_access_vd`（`:770-771`）。

### 7.3 控制寄存器（`:776-802`）

`abs_cmd_postexec_ff`/`abs_cmd_wr_ff`/`abs_cmd_regno_ff`/`abs_cmd_size_ff` 在
`clk_en_abs & abs_fsm_idle` 时更新（`:776-783`）；
`abs_cmd_*_next` 仅在 `(command_wr_req|dmi_rpt_command) & hart_state_dhalt & abs_fsm_idle`
且命令有效时置位（`:785-802`）。

---

## 8. 抽象命令 FSM（`:804-903`）

- 状态寄存器：`clk_en_dm` 且 dmactive 时更新，否则强制 IDLE（`:808-816`）；
- 状态转移（`:818-891`）：
  - **IDLE**：按命令类型跳 CSR_RO / CSR_SAVE_XREG / XREG_RW / EXEC / MEM_SAVE_XREG，
    非法或未 halt 时 ERR（`:822-833`）；
  - **EXEC**：`dhi_resp` 后，异常或 busy 错误→ERR，否则 IDLE（`:835-843`）；
  - **XREG_RW**：busy→ERR，postexec→EXEC，否则 IDLE（`:845-853`）；
  - **CSR_RO**：busy→ERR 否则 IDLE（`:855`）；
  - **CSR_SAVE_XREG / CSR_RW**：靠 `dhi_resp` 推进（`:856-857`）；
  - **CSR_RETURN_XREG**：异常/busy→ERR，postexec→EXEC，否则 IDLE（`:859-868`）；
  - **MEM_SAVE_XREG → MEM_SAVE_XREG_FORADDR → MEM_RW → MEM_RETURN_XREG →
    MEM_RETURN_XREG_FORADDR**：逐拍 `dhi_resp` 推进（`:870-873`）；
  - **MEM_RETURN_XREG_FORADDR**：异常/busy→ERR，postexec→EXEC，否则 IDLE（`:875-884`）；
  - **ERR**：写 ABSTRACTCS 且 cmderr 被清 → IDLE（`:886-890`）；
- **强制**：`~abs_fsm_idle & hart_state_reset` → ERR（`:893-895`）。
- 状态判定：`idle/exec/csr_ro/err/use_addr`（`:898-903`，`use_addr` 覆盖两个 FORADDR 态）。

---

## 9. 抽象命令状态逻辑（`:905-1149`）

### 9.1 busy / exception 错误寄存器（`:912-929`）

- `abs_err_acc_busy_upd = clk_en_abs & (abs_fsm_idle | dmi_req_any)`（`:912`）；
  `abs_err_acc_busy_next = ~abs_fsm_idle & dmi_req_any`（`:918`，处理中再访问）；
- `abs_err_exc_upd = clk_en_abs & (abs_fsm_idle | (dhi_resp & dhi_resp_exc))`（`:923`）；
  `abs_err_exc_next = ~abs_fsm_idle & dhi_resp & dhi_resp_exc`（`:929`）。

### 9.2 ABSTRACTCS.cmderr（`:1061-1149`）

- 状态寄存器：dmactive=0 时清 NONE（`:1061-1069`）；
- 组合更新（`:1071-1142`）：IDLE 时按命令合法性给 NOHALT/CMD；EXEC 时 EXCEPTION/BUSY；
  XREG_RW/CSR_RO busy→BUSY；CSR_RETURN/MEM_RETURN_FORADDR 按异常/busy；ERR 时按写数据清位；
- **强制**：`~abs_fsm_idle & hart_state_reset` → EXCEPTION（`:1144-1146`）。
- `abstractcs_busy = ~abs_fsm_idle & ~abs_fsm_err`（`:1149`）。

---

## 10. 抽象指令逻辑（`:931-1045`）

- **执行请求**：`abs_exec_req_next = ~(idle|csr_ro|err) & ~dhi_resp`（`:945`），dmactive=0 清 0（`:947-955`）；
- **mem funct3 mux**（`:960-967`）：按 size 选 SB/LBU、SH/LHU、SW/LW；
- **RS1 mux**（`:972-986`）：XREG_RW 时写用 0/读用 regno；CSR/MEM 保存/返回用 x5；
  FORADDR/MEM_RW 用 x6；
- **RD mux**（`:993-1007`）：按状态给出目标寄存器（x5/x6 或 regno）；
- **指令组装**（`:1012-1045`）：
  - 通用状态用 `CSRRW DSCRATCH0, x5/x6`（`:1021-1029`）；
  - CSR_RW：写用 CSRRW、读用 CSRRS，regno 来自 `abs_cmd_regno_ff`（`:1031-1035`）；
  - MEM_RW：写用 STORE、读用 LOAD，地址来自 x6、数据/目标来自 x5（`:1037-1041`）。

---

## 11. 抽象寄存器（`:1047-1242`）

| 寄存器 | 更新条件 | 行号 |
|---|---|---|
| ABSTRACTCS.cmderr | `clk_en_dm`，dmactive=0 清 0 | 1061-1069 |
| COMMAND | `clk_en_dm`；dmactive=0 清 0，写且 IDLE 时载入 | 1154-1160 |
| ABSTRACTAUTO | `clk_en_dm`；dmactive=0 清 0，写且 IDLE 时载入位 0 | 1165-1171 |
| PROGBUF0-5 | `clk_en_abs & abs_fsm_idle`，各写请求载入 | 1176-1185 |
| DATA0 | `clk_en_abs`；各状态分别从 DMI 或 dreg 载入；CSR_RO 时按 regno 回填常量/PC（`:1210-1219`） | 1190-1223 |
| DATA1 | `clk_en_abs`；IDLE 从 DMI，两个 FORADDR 态从 dreg | 1228-1242 |

`data0_xreg_save = dreg_wr_req & ~abs_cmd_wr_ff`（`:1196`）。
CSR_RO 回填：MISA/MVENDORID/MARCHID/MIMPID/MHARTID 常量、default 为 `pipe2dm_pc_sample_i`（DPC，`:1210-1218`）。

---

## 12. DHI 控制与 FSM（`:1244-1342`）

- `cmd_resp_ok = pipe2dm_cmd_resp_i & ~pipe2dm_cmd_rcode_i`（`:1248`）；
- `hart_rst_unexp = ~dhi_fsm_idle & ~dhi_fsm_halt_req & hart_state_reset`（`:1249`，非预期复位）；
- `halt_req_vd = haltreq_ff & ~dhalt`；`resume_req_vd = resumereq_ff & ~resumeack & dhalt`（`:1251-1253`）。

**DHI FSM**（`:1258-1284`）：`clk_en_dm` 推进；异常复位/dmactive=0 时回 IDLE（`:1280-1283`）；
状态流：IDLE→**EXEC**→EXEC_RUN→EXEC_HALT→IDLE（执行调试指令）；
IDLE→**HALT_REQ**→EXEC_HALT→IDLE；IDLE→**RESUME_REQ**→RESUME_RUN→IDLE。

- `dhi_req` 优先级（`:1292-1299`）：`abs_exec_req_ff` > `halt_req_vd` > `resume_req_vd` > IDLE；
- `dhi_resp = dhi_fsm_exec_halt & hart_state_dhalt`（`:1301`）；
- `dhi_resp_exc = hart_event & hart_status.except & ~hart_status.ebreak`（`:1302-1303`）。

### HART 命令寄存器（`:1305-1342`）

- `hart_cmd_req_next = (exec|halt_req|resume_req) & ~cmd_resp_ok & dmactive`（`:1317-1318`）；
- `hart_cmd_next`：EXEC→DRUN、HALT_REQ→DHALTED、RESUME_REQ→RUN、其它保持（`:1329-1339`）；
- `dm2pipe_cmd_req_o = hart_cmd_req_ff`；`dm2pipe_cmd_o = hart_cmd_ff`（`:1341-1342`）。

---

## 13. 程序缓冲区与抽象数据（`:1344-1384`）

### 13.1 PBUF（`:1348-1376`）

- `hart_pbuf_ebreak_next = abs_fsm_exec & (dm2pipe_pbuf_instr_o == ABS_EXEC_EBREAK)`（`:1355`）；
- 指令 mux：默认 EBREAK；`abs_fsm_exec & ~hart_pbuf_ebreak_ff` 时按 `pipe2dm_pbuf_addr_i`
  选 PROGBUF0-5；否则若 addr==0 用 `abs_exec_instr_ff`（`:1360-1376`）。

### 13.2 抽象数据（`:1382-1384`）

`dm2pipe_dreg_resp_o = 1`、`dm2pipe_dreg_fail_o = 0`（恒）；
`dm2pipe_dreg_rdata_o = abs_fsm_use_addr ? abs_data1_ff : abs_data0_ff`（`:1384`）。

---

## 14. 断言（`SCR1_TRGT_SIMULATION`，`:1386-1423`）

| 断言 | 行号 | 检查 |
|---|---|---|
| `SVA_DM_X_CONTROL` | 1391-1396 | 控制信号无 X |
| `SVA_DM_X_DMI` | 1398-1401 | req 时 wr/addr/wdata 无 X |
| `SVA_DM_X_HART_PBUF` | 1403-1406 | pbuf addr 无 X |
| `SVA_DM_X_HART_DREG` | 1408-1411 | dreg req 时 wr/wdata 无 X |
| `SVA_DM_X_HART_CMD` | 1413-1416 | cmd_resp 时 rcode 无 X |
| `SVA_DM_X_HART_EVENT` | 1418-1421 | hart_event 时 status 无 X |

均为 `@(negedge clk) disable iff (~rst_n)` 的 X 检查。

---

## 15. 附录

### 15.1 源码级注意点

- `dm2dmi_resp_o` 恒 1（`:555`），DM 无法对 DMI 请求背压；
- `abs_cmd_memvalid` 只检查 `RESERVEDB_HI`/`RESERVEDA_HI` 两个单 bit（`:740-743`），
  而非整个保留域；
- `ndm_rst_n_o` 与 `hart_rst_n_o` 同源（`:649-650`），均由 ndmreset 控制；
- `hart_cmd_ff` 复位值为 `SCR1_HDU_DBGSTATE_RUN`（`:1323`）。

### 15.2 职责边界

- DM 不直接访问内存/寄存器，一切经 DHI 抽象命令转为调试指令由 HDU/PBUF 执行；
- 命令合法性、cmderr、busy 均由 DM 判定；
- 复位域：DM 使用 `dm_rst_n`，HART 使用 `hart_rst_n`，两者由 SCU 生成。
