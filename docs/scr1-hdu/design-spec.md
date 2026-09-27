# SCR1 HDU HART 调试单元 设计规格（Design Specification）

- 模块：`scr1_pipe_hdu`（`src/core/pipeline/scr1_pipe_hdu.sv`，904 行）
- 参数：`HART_PBUF_INSTR_REGOUT_EN = 1'b1`（`:38`）
- 条件：仅在 `SCR1_DBG_EN` 时编译（`:33`、`:904`）
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:684-762`）
- 职责：控制 HART 运行状态、Debug 模式执行、Program Buffer、Debug CSR、与 DM/EXU/IFU/CSR/TDU 交互

---

## 1. 概述与职责

HDU（HART Debug Unit）实现 RISC-V Debug 规范中 HART 侧的调试功能：

1. **调试状态机**：RESET / RUN / DHALTED / DRUN 四态；
2. **HART 运行控制**：停机请求、恢复、runctrl 配置、停机超时；
3. **停机原因**：单步、异常、EBREAK、DM 请求、触发器请求；
4. **Program Buffer**：在 Debug 模式下执行调试器提供的指令；
5. **Debug CSR**：DCSR、DPC、DSCRATCH0；
6. **带限定的跨复位域接口**：所有来自 `pipe_rst_n` 域的信号经 `pipe2hdu_rdc_qlfy_i` 限定。

---

## 2. 端口（`:38-117`）

### 2.1 公共（`:39-46`）

`rst_n`、`clk`、`clk_en`、`clk_pipe_en`（CLKCTRL）、`pipe2hdu_rdc_qlfy_i`。

### 2.2 CSR（`:48-54`）

`csr2hdu_req_i`、`csr2hdu_cmd_i`、`csr2hdu_addr_i`、`csr2hdu_wdata_i`、
`hdu2csr_resp_o`、`hdu2csr_rdata_o`。

### 2.3 DM（`:56-75`）

- Run Control：`dm2hdu_cmd_req_i`、`dm2hdu_cmd_i`、`hdu2dm_cmd_resp_o`、
  `hdu2dm_cmd_rcode_o`、`hdu2dm_hart_event_o`、`hdu2dm_hart_status_o`；
- Program Buffer：`hdu2dm_pbuf_addr_o`、`dm2hdu_pbuf_instr_i`；
- Abstract Data：`hdu2dm_dreg_req_o`/`wr_o`/`wdata_o`、`dm2hdu_dreg_resp_i`/`fail_i`/`rdata_i`。

### 2.4 TDU（`:77-82`，`SCR1_TDU_EN`）

`hdu2tdu_hwbrk_dsbl_o`、`tdu2hdu_dmode_req_i`、`exu2hdu_ibrkpt_hw_i`。

### 2.5 HART 状态（`:84-91`）

`pipe2hdu_exu_busy_i`、`pipe2hdu_instret_i`、`pipe2hdu_init_pc_i`、
`pipe2hdu_exu_exc_req_i`、`pipe2hdu_brkpt_i`。

### 2.6 EXU（`:93-109`）

`hdu2exu_pbuf_fetch_o`、`no_commit_o`、`irq_dsbl_o`、`pc_advmt_dsbl_o`、
`dmode_sstep_en_o`、`dbg_halted_o`、`dbg_run2halt_o`、`dbg_halt2run_o`、
`dbg_run_start_o`、`hdu2exu_dbg_new_pc_o`；输入 `pipe2hdu_pc_curr_i`。

### 2.7 IFU（`:111-116`）

`ifu2hdu_pbuf_instr_rdy_i`、`hdu2ifu_pbuf_instr_vd_o`/`err_o`/`instr_o`。

---

## 3. 参数与类型（`:119-124`、`scr1_hdu.svh`）

| 项 | 值 | 行号 |
|---|---|---|
| `SCR1_HDU_TIMEOUT` | 64（2 的幂） | :123 |
| `SCR1_HDU_TIMEOUT_WIDTH` | `$clog2(64)`=6 | :124 |
| `SCR1_HDU_DEBUGCSR_ADDR_WIDTH` | `$clog2(4)`=2 | hdu.svh:22 |
| `SCR1_HDU_PBUF_ADDR_SPAN` | 8 | :24 |
| `SCR1_HDU_CORE_INSTR_WIDTH` | 32 | :27 |

类型：

- `type_scr1_hdu_dbgstates_e`：RESET(00)/RUN(01)/DHALTED(10)/DRUN(11)（hdu.svh:36-44）；
- `type_scr1_hdu_pbufstates_e`：IDLE/FETCH/EXCINJECT/WAIT4END（:47-55）；
- `type_scr1_hdu_fetch_src_e`：NORMAL/PBUF（:67-71）；
- `type_scr1_hdu_haltcause_e`：NONE/EBREAK/TMREQ/DMREQ/SSTEP/RSTEXIT（:98-108）；
- `type_scr1_hdu_runctrl_s`（:94）、`type_scr1_hdu_haltstatus_s`（:113）、
  `type_scr1_hdu_hartstatus_s`（:80）、`type_scr1_hdu_dcsr_s`（:161）。

Debug CSR 偏移（:117-120）：DCSR=0、DPC=1、DSCRATCH0=2、DSCRATCH1=3（未实现）。

DCSR 位（:129-145）：PRV[1:0]、STEP=2、CAUSE[8:6]、STEPIE=11、EBREAKM=15、XDEBUGVER[31:28]=4。

---

## 4. 调试状态机（`:251-374`）

### 4.1 FSM 控制（`:261-269`）

```systemverilog
dm_cmd_dhalted = (dm2hdu_cmd_i == DHALTED);
dm_cmd_run     = (dm2hdu_cmd_i == RUN);
dm_cmd_drun    = (dm2hdu_cmd_i == DRUN);
dm_dhalt_req   = dm2hdu_cmd_req_i & dm_cmd_dhalted;
dm_run_req     = dm2hdu_cmd_req_i & (dm_cmd_run | dm_cmd_drun);
```

### 4.2 状态寄存器（`:274-280`）

时钟域 `clk`，复位为 RESET。

### 4.3 状态转移（`:282-314`）

`~pipe2hdu_rdc_qlfy_i` 时强制回 RESET。否则：

| 当前态 | 次态条件 | 次态 |
|---|---|---|
| RESET | `~init_pc_i` | RESET |
| RESET | `dm_dhalt_req` | DHALTED |
| RESET | 否则 | RUN |
| RUN | `dfsm_update` | DHALTED |
| RUN | 否则 | RUN |
| DHALTED | `~dfsm_update` | DHALTED |
| DHALTED | `dm_cmd_drun` | DRUN |
| DHALTED | 否则（update 且非 drun） | RUN |
| DRUN | `dfsm_update` | DHALTED |
| DRUN | 否则 | DRUN |

状态解码（`:316-319`）：`dbg_state_*`。

### 4.4 转移/更新/事件寄存器（`:321-374`）

```systemverilog
// RESET: update=0, event=init_pc & ~cmd_req
// RUN/DRUN: trans = ~update ? hart_halt_pnd : trans;
//            update = ~update & hart_halt_ack;
//            event = update;
// DHALTED : trans = ~update ? ~trans & dm_run_req : trans;
//            update = ~update & trans;
//            event = update;
```

两拍握手：先置 `trans`，再置 `update`，`event` 随之脉冲。

---

## 5. HART 控制（`:376-467`）

### 5.1 命令请求（`:386-404`）

`hart_cmd_req` 按状态产生；`hart_halt_req = dm_cmd_dhalted & cmd_req`、
`hart_resume_req = (run|drun) & cmd_req`。

### 5.2 Run Control 寄存器（`:406-445`）

- `hart_runctrl_clr`：RUN|DRUN 且下一态 DHALTED；
- `hart_runctrl_upd`：DHALTED 且 `dfsm_trans_next`；
- 恢复 RUN（`~dm_cmd_drun`）：
  - `irq_dsbl = step ? ~stepie : 0`；fetch=NORMAL；pc_advmt_dsbl=0；hwbrkpt_dsbl=0；
    `redirect.sstep=step`；`redirect.ebreak=ebreakm`；
- 恢复 DRUN（`dm_cmd_drun`）：
  - `irq_dsbl=1`；fetch=PBUF；pc_advmt_dsbl=1；hwbrkpt_dsbl=1；
    `redirect.sstep=0`；`redirect.ebreak=1`。

### 5.3 停机超时计数器（`:447-467`）

```systemverilog
halt_req_timeout_cnt_en = halt2run | (halt_req & ~run2halt);
halt_req_timeout_cnt_next = halt2run ? '1
                          : (halt_req & ~run2halt) ? cnt-1 : cnt;
halt_req_timeout_flag = ~|cnt;
```

停机请求需持续 `SCR1_HDU_TIMEOUT` 周期才强制，或 EXU 空闲且已有 cause 时立即。

---

## 6. HART 状态与停机原因（`:469-530`）

### 6.1 原因解码（`:479-499`）

```systemverilog
dmode_cause_sstep  = redirect.sstep & instret_i;
dmode_cause_except = dbg_state_drun & exc_req_i & ~brkpt_i & ~hwbrkpt_hw;
dmode_cause_ebreak = redirect.ebreak & brkpt_i;
dmode_cause_tmreq  = tdu2hdu_dmode_req_i & exu2hdu_ibrkpt_hw_i;
dmode_cause_any    = sstep | ebreak | except | hart_halt_req | tmreq;
```

### 6.2 停机原因编码（`:504-514`）

优先级（`case(1'b1)`）：TMREQ > EBREAK > DMREQ(`hart_halt_req`) > SSTEP > NONE。

### 6.3 停机状态寄存器（`:519-526`）

`hart_halt_ack` 时锁存 `{except, cause}`；复位 0。

### 6.4 pending/ack（`:528-530`）

```systemverilog
hart_halt_pnd = (dfsm_trans | dm_dhalt_req) & ~hart_halt_ack;
hart_halt_ack = ~dbg_halted_o & (timeout_flag | (~exu_busy_i & cause_any));
```

---

## 7. Program Buffer（`:532-621`）

### 7.1 FSM（`:549-588`）

- `ifu_handshake_done = pbuf_instr_vd & ifu_rdy`；
- `pbuf_start_fetch = DHALTED & (next==DRUN)`；
- `pbuf_exc_inj_req = handshake_done & addr_end`；
- `pbuf_exc_inj_end = exu_exc_req | handshake_done`；
- 状态转移：

| 当前态 | 条件 | 次态 |
|---|---|---|
| IDLE | `pbuf_start_fetch` | FETCH |
| FETCH | `exu_exc_req` | WAIT4END |
| FETCH | `exc_inj_req` | EXCINJECT |
| FETCH | 否则 | FETCH |
| EXCINJECT | `exc_inj_end` | WAIT4END |
| WAIT4END | `dbg_halted_o` | IDLE |

### 7.2 地址（`:590-606`）

`pbuf_addr_next_vd = fetch & handshake_done & ~exc_req & ~addr_end`，
IDLE 清零，否则递增。

### 7.3 指令输出寄存器（`:608-621`）

`HART_PBUF_INSTR_REGOUT_EN=1` 时经寄存器输出（频率优化），
`pbuf_instr_wait_latching` 在握手后延迟一拍，避免重复发送。

---

## 8. Debug CSR（`:623-745`）

### 8.1 停机时更新（`:627-628`）

`csr_upd_on_halt = (reset|run) & (next==DHALTED)`：进入停机时锁存 DPC 与 cause。

### 8.2 选择（`:633-651`）

按 `csr2hdu_addr_i` 选 DCSR/DPC/DSCRATCH0；未知地址置 X。

### 8.3 读写数据（`:653-674`）

读：`csr_rd_data = dcsr | dpc | dscratch0`；写数据按 WRITE/SET/CLEAR 语义。

### 8.4 DCSR（`:676-717`）

- 读模型：XDEBUGVER/prv=11/ebreakm/stepie/step/cause；
- 可写：ebreakm/stepie/step；
- `cause` 在 `csr_upd_on_halt` 时取 `hart_haltstatus.cause`。

### 8.5 DPC（`:719-737`）

```systemverilog
csr_dpc_next = csr_upd_on_halt ? pc_curr_i
             : csr_dpc_wr      ? csr_wr_data
                               : csr_dpc_ff;
```

### 8.6 DSCRATCH0（`:739-745`）

映射到 DM Abstract Data 接口：`dscratch0_resp` 由 `dm2hdu_dreg_resp_i`/
`fail_i` 决定；读数据来自 `dm2hdu_dreg_rdata_i`。

---

## 9. 对外接口

### 9.1 HDU ↔ DM（`:747-794`）

- `hdu2dm_hart_event_o = dfsm_event`；
- `hdu2dm_hart_status_o`：dbg_state、except、ebreak；
- `hdu2dm_cmd_rcode_o`：按状态与限定信号判定；
- `hdu2dm_cmd_resp_o`：按状态握手；
- Program Buffer 地址、Abstract Data 请求/写使能/写数据。

### 9.2 HDU → EXU（`:796-820`）

```systemverilog
dbg_halted_o    = (next==DHALTED) | (~qlfy & ~dbg_state_run);
dbg_run_start_o = dbg_state_dhalted & qlfy & dfsm_update;
dbg_halt2run_o  = dbg_halted_o & hart_resume_req [& clk_pipe_en];
dbg_run2halt_o  = hart_halt_ack;
pbuf_fetch_o    = hart_runctrl.fetch_src;
irq_dsbl_o      = hart_runctrl.irq_dsbl;
pc_advmt_dsbl_o = hart_runctrl.pc_advmt_dsbl;
no_commit_o     = dmode_cause_ebreak | dmode_cause_tmreq;
dmode_sstep_en_o= hart_runctrl.redirect.sstep;
dbg_new_pc_o    = csr_dpc_ff;
```

### 9.3 HDU → IFU（`:822-836`）

```systemverilog
pbuf_instr_vd_o  = (fetch | excinj) & ~pbuf_instr_wait_latching;
pbuf_instr_err_o = pbuf_fsm_excinj;
```

指令可经寄存器输出（generate）。

### 9.4 HDU ↔ CSR（`:838-848`）

```systemverilog
hdu2csr_resp_o = ~dbg_state_drun ? RESP_ER
               : dscratch0        ? csr_dscratch0_resp
               : req              ? RESP_OK : RESP_ER;
hdu2csr_rdata_o = csr_rd_data;
```

> Debug CSR 仅在 DRUN 状态下允许访问。

### 9.5 HDU ↔ TDU（`:850-856`）

`hdu2tdu_hwbrk_dsbl_o = hart_runctrl.hwbrkpt_dsbl`。

---

## 10. 假设与约束

1. 所有 `pipe_rst_n` 域输入经 `pipe2hdu_rdc_qlfy_i` 限定，未限定视为复位态；
2. 停机需超时或 EXU 空闲且有 cause；
3. Debug 模式（DRUN）才可访问 Debug CSR；
4. 停机时 DPC 保存 `pc_curr`，恢复时经 `dbg_new_pc` 返回；
5. DSCRATCH1 定义但未实现。

---

## 11. 内建断言（`:858-900`，`SCR1_TRGT_SIMULATION`）

| 断言 | 行号 | 检查 |
|---|---|---|
| `SVA_HDU_XCHECK_COMMON` | 863-868 | 公共信号无 X |
| `SVA_HDU_XCHECK_CSR_INTF` | 870-875 | CSR 接口无 X |
| `SVA_HDU_XCHECK_DM_INTF` | 877-883 | DM 接口无 X |
| `SVA_HDU_XCHECK_TDU_INTF` | 885-890 | TDU 接口无 X |
| `SVA_HDU_XCHECK_HART_INTF` | 892-898 | HART 接口无 X |

---

## 附录 A：状态与原因速查

| 状态 | 含义 |
|---|---|
| RESET | 复位，等待 init_pc |
| RUN | 正常执行 |
| DHALTED | 调试停机 |
| DRUN | 调试模式运行（Program Buffer） |

| 停机原因 | 含义 |
|---|---|
| EBREAK | 软件断点 |
| TMREQ | 触发器请求调试模式 |
| DMREQ | 调试模块请求 |
| SSTEP | 单步 |
| RSTEXIT | 复位退出（定义） |

## 附录 B：信号→消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `hdu2exu_no_commit_o` | EXU | EBREAK/TMREQ 时不提交 |
| `hdu2exu_pbuf_fetch_o` | EXU/IFU | 取 Program Buffer 指令 |
| `hdu2exu_dbg_new_pc_o` | EXU | 恢复地址 |
| `hdu2ifu_pbuf_instr_*` | IFU | 注入调试指令 |
| `hdu2csr_*` | CSR | Debug CSR 访问 |
| `hdu2tdu_hwbrk_dsbl_o` | TDU | 禁硬件断点 |
