# SCR1 流水线顶层 设计规格（Design Specification）

- 模块：`scr1_pipe_top`（`src/core/pipeline/scr1_pipe_top.sv`，825 行）
- 层次：CPU 核心流水线顶层，向上由 `scr1_core_top` 例化
- 职责：例化并互连 IFU/IDU/EXU/MPRF/CSR/IPIC/TDU/HDU，派生流水线控制与时钟请求

> `scr1_pipe_top` 是一个**结构化集成层**：本身几乎没有时序逻辑，主要工作是
> （1）把各子模块例化并接线，（2）做少量跨模块控制信号派生，
> （3）处理配置宏与时钟/复位域的条件化。

---

## 1. 概述与职责

1. **模块例化**：IFU、IDU、EXU、MPRF、CSR、IPIC、TDU、HDU、tracelog；
2. **接口暴露**：IMEM/DMEM、中断、外部定时器、调试、时钟控制、MHARTID 熔丝；
3. **控制派生**：
   - `stop_fetch`（WFI 停机 + 调试取指停止）；
   - `pipe2clkctl_sleep_req_o` / `wake_req_o`（时钟门控请求）；
   - `pipe2dm_pc_sample_o`（调试 PC 采样）；
   - HDU/TDU 输入在复位域交叉上的限定（`*_qlfy`）。
4. **配置条件化**：IPIC/DBG/TDU/CLKCTRL/MPRF_RST/CSR_REDUCED_CNT 分支。

**注意**：IALU 与 LSU **不在**顶层例化，而在 `scr1_pipe_exu` 内部（`scr1_pipe_top.sv` 全文无
`scr1_pipe_ialu`/`scr1_pipe_lsu` 例化）。

---

## 2. 头文件（`:6-21`）

| 文件 | 条件 |
|---|---|
| `scr1_arch_description.svh` | 总是 |
| `scr1_memif.svh` | 总是 |
| `scr1_riscv_isa_decoding.svh` | 总是 |
| `scr1_csr.svh` | 总是 |
| `scr1_ipic.svh` | `SCR1_IPIC_EN` |
| `scr1_hdu.svh` | `SCR1_DBG_EN` |
| `scr1_tdu.svh` | `SCR1_TDU_EN` |

---

## 3. 端口（`:23-102`）

### 3.1 公共（`:24-30`）

| 信号 | 方向 | 条件 | 行号 |
|---|---|---|---|
| `pipe_rst_n` | in | 总是 | 25 |
| `pipe2hdu_rdc_qlfy_i` | in | DBG | 27 |
| `dbg_rst_n` | in | DBG | 28 |
| `clk` | in | 总是 | 30 |

### 3.2 指令内存接口（`:32-38`）

`pipe2imem_req_o`/`cmd_o`/`addr_o`、`imem2pipe_req_ack_i`/`rdata_i`/`resp_i`。

### 3.3 数据内存接口（`:40-48`）

`pipe2dmem_req_o`/`cmd_o`/`width_o`/`addr_o`/`wdata_o`、
`dmem2pipe_req_ack_i`/`rdata_i`/`resp_i`。

### 3.4 调试接口（`:50-77`，`SCR1_DBG_EN`）

- Run Control：`dbg_en`、`dm2pipe_active_i`、`dm2pipe_cmd_req_i`/`cmd_i`、
  `pipe2dm_cmd_resp_o`/`cmd_rcode_o`/`hart_event_o`/`hart_status_o`；
- Program Buffer：`pipe2dm_pbuf_addr_o`、`dm2pipe_pbuf_instr_i`；
- Abstract Data：`pipe2dm_dreg_*`、`dm2pipe_dreg_*`；
- PC：`pipe2dm_pc_sample_o`。

### 3.5 中断与定时器（`:79-89`）

```systemverilog
`ifdef SCR1_IPIC_EN
    input [SCR1_IRQ_LINES_NUM-1:0] soc2pipe_irq_lines_i;
`else
    input                          soc2pipe_irq_ext_i;
`endif
    input soc2pipe_irq_soft_i;
    input soc2pipe_irq_mtimer_i;
    input [63:0] soc2pipe_mtimer_val_i;
```

### 3.6 时钟控制（`:91-98`，`SCR1_CLKCTRL_EN`）

`pipe2clkctl_sleep_req_o`、`pipe2clkctl_wake_req_o`、
`clkctl2pipe_clk_alw_on_i`、`clkctl2pipe_clk_dbgc_i`、`clkctl2pipe_clk_en_i`。

### 3.7 熔丝（`:100-101`）

`soc2pipe_fuse_mhartid_i[XLEN-1:0]`。

---

## 4. 局部信号（`:104-273`）

| 分组 | 行号 | 说明 |
|---|---|---|
| Pipeline control | 108-134 | `curr_pc`/`next_pc`/`new_pc_req`/`new_pc`、`stop_fetch`、`exu_exc_req`、`brkpt`、`exu_init_pc`、`wfi_run2halt`、`instret`、`instret_nexc`、`ipic2csr_irq`、`brkpt_hw`、`imem_txns_pending`、`wfi_halted` |
| IFU↔IDU | 136-141 | `ifu2idu_*`、`idu2ifu_rdy` |
| IDU↔EXU | 143-152 | `idu2exu_*`、`exu2idu_rdy` |
| EXU↔MPRF | 154-161 | `exu2mprf_*`、`mprf2exu_*` |
| EXU↔CSR | 163-170 | `exu2csr_*`、`csr2exu_r_data`/`rw_exc` |
| EXU↔CSR 事件 | 172-182 | `exu2csr_take_irq/exc/mret_*`、`exu2csr_exc_code`、`exu2csr_trap_val`、`csr2exu_new_pc/irq/ip_ie/mstatus_mie_up` |
| CSR↔IPIC | 184-191 | `csr2ipic_*`、`ipic2csr_rdata` |
| CSR↔TDU | 193-223 | `csr2tdu_*`、`tdu2csr_*`、`exu2tdu_i_mon`、`lsu2tdu_d_mon`、`tdu2exu_i_match`、`tdu2lsu_d_match`、`tdu2*_x_req`、`exu2tdu_bp_retire` |
| 调试 | 225-266 | HDU 命令/状态、`fetch_pbuf`、`exu_no_commit`、`dbg_*`、pbuf 指令 |
| 其它 | 268-273 | `exu_busy`；非 CLKCTRL 时内部 `pipe2clkctl_wake_req_o` |

---

## 5. 流水线控制派生（`:275-295`）

### 5.1 停止取指（`:278-282`）

```systemverilog
assign stop_fetch = wfi_run2halt
`ifdef SCR1_DBG_EN
                  | fetch_pbuf
`endif
                  ;
```

WFI 进入停机或调试取 Program Buffer 时停止正常取指。

### 5.2 时钟请求（`:284-291`，`SCR1_CLKCTRL_EN`）

```systemverilog
assign pipe2clkctl_sleep_req_o = wfi_halted & ~imem_txns_pending;
assign pipe2clkctl_wake_req_o  = csr2exu_ip_ie
`ifdef SCR1_DBG_EN
                               | dm2pipe_active_i
`endif
                               ;
```

- **睡眠**：WFI 已停机且无未决 IMEM 事务；
- **唤醒**：存在“pending 且本地使能”的中断（`ip_ie`），或调试模块激活。

> `ip_ie` 不含全局 MIE，因此关中断时的局部挂起也能唤醒（WFI 语义）。

### 5.3 调试 PC 采样（`:293-295`）

```systemverilog
assign pipe2dm_pc_sample_o = curr_pc;
```

---

## 6. 子模块例化

### 6.1 IFU（`:300-335`）

- 无条件连接 `.rst_n(pipe_rst_n)`、`.clk(clk)`（`:301-302`）；
- IMEM 接口直连顶层端口；
- New PC：`.exu2ifu_pc_new_req_i(new_pc_req)`、`.exu2ifu_pc_new_i(new_pc)`；
- `.pipe2ifu_stop_fetch_i(stop_fetch)`；
- DBG：Program Buffer 接口（`:317-324`）；
- CLKCTRL：`.ifu2pipe_imem_txns_pnd_o(imem_txns_pending)`（`:326`）；
- IFU↔IDU：`.idu2ifu_rdy_i`、`.ifu2idu_*`。

### 6.2 IDU（`:340-360`）

- `.rst_n`/`.clk` **仅在 `SCR1_TRGT_SIMULATION` 下连接**（`:341-344`）——
  综合时 IDU 是纯组合逻辑；
- 输出 `idu2exu_req/cmd/use_rs1/use_rs2`，`use_rd`/`use_imm` 受
  `SCR1_NO_EXE_STAGE` 控制（`:355-358`）。

### 6.3 EXU（`:365-472`）

EXU 是最大子模块，端口分组：

| 分组 | 行号 |
|---|---|
| IDU↔EXU | 373-382 |
| EXU↔MPRF | 384-391 |
| EXU↔CSR 读写 | 393-400 |
| EXU↔CSR 事件 | 402-412 |
| EXU↔DMEM | 414-422 |
| EXU↔HDU（DBG） | 424-436 |
| EXU↔TDU | 438-451 |
| EXU 控制 | 453-462 |
| PC/CLKCTRL | 464-471 |

关键派生：`.csr2exu_ip_ie_i(csr2exu_ip_ie)`（`:411`）、
`.exu2ifu_pc_new_req_o(new_pc_req)`（`:470`）、`.exu2ifu_pc_new_o(new_pc)`（`:471`）、
`.exu2pipe_pc_curr_o(curr_pc)`（`:468`）、`.exu2csr_pc_next_o(next_pc)`（`:469`）。

### 6.4 MPRF（`:477-491`）

- `.rst_n` 仅在 `SCR1_MPRF_RST_EN` 下连接（`:478-480`）；
- `.clk` 总是；
- 2 读 1 写接口与 EXU 互连。

### 6.5 CSR（`:496-575`）

- `.clk_alw_on` 仅在 `~SCR1_CSR_REDUCED_CNT && SCR1_CLKCTRL_EN` 下连接（`:499-503`）；
- EXU↔CSR 读写与事件（`:505-524`）；
- IPIC（`:526-533`）；
- PC 接口：`.exu2csr_pc_curr_i(curr_pc)`、`.exu2csr_pc_next_i(next_pc)`（`:536-537`）；
- `instret_no_exc` 受 `~SCR1_CSR_REDUCED_CNT`（`:538-540`）；
- IRQ：IPIC 模式下 `.soc2csr_irq_ext_i(ipic2csr_irq)`，否则直连外部（`:543-547`）；
- HDU（`:554-563`）、TDU（`:565-573`）、MHARTID 熔丝（`:574`）。

### 6.6 IPIC（`:580-596`，`SCR1_IPIC_EN`）

- `clk` 在 CLKCTRL 下用 `clk_alw_on`，否则用 `clk`（`:583-587`）；
- 输入 IRQ lines、CSR 读写；输出 `ipic2csr_rdata`、`ipic2csr_irq_m_req_o`。

### 6.7 TDU（`:602-678`，`SCR1_TDU_EN`）

- `rst_n`：DBG 下用 `dbg_rst_n`，否则 `pipe_rst_n`（`:604-608`）；`clk_en=1`；
- `tdu_dsbl_i = hwbrk_dsbl`（DBG）或 0（`:611-615`）；
- CSR 侧输入在 DBG 下使用 `*_qlfy` 版本（`:618-628`）；
- EXU/LSU 监视与断点匹配（`:632-654`）；
- DBG 下 EPU 请求 `tdu2hdu_dmode_req`，否则悬空（`:655-660`）。

**限定逻辑**（`:663-676`，DBG）：

```systemverilog
assign hwbrk_dsbl        = (~dbg_en) | hdu_hwbrk_dsbl;            // :664
assign csr2tdu_req_qlfy  = dbg_en & csr2tdu_req & pipe2hdu_rdc_qlfy_i; // :666
assign exu2tdu_i_mon_qlfy.vd   = exu2tdu_i_mon.vd & pipe2hdu_rdc_qlfy_i; // :668
assign lsu2tdu_d_mon_qlfy.vd   = lsu2tdu_d_mon.vd & pipe2hdu_rdc_qlfy_i; // :671
assign exu2tdu_bp_retire_qlfy  = exu2tdu_bp_retire & {N{pipe2hdu_rdc_qlfy_i}}; // :675
```

### 6.8 HDU（`:684-762`，`SCR1_DBG_EN`）

- `rst_n=dbg_rst_n`、`clk_en=dm2pipe_active_i`（`:686-687`）；
- `clk`：CLKCTRL 下 `clkctl2pipe_clk_dbgc_i`，否则 `clk`（`:688-693`）；
- CSR 侧 `csr2hdu_req_qlfy`（`:696`）；
- Run Control / Program Buffer / Abstract Data / PC 接口；
- HART 运行状态输入用 `*_qlfy` 版本（`:732-738`）；
- HDU→EXU 控制输出：`fetch_pbuf`、`exu_no_commit`、`exu_irq_dsbl`、
  `exu_pc_advmt_dsbl`、`exu_dmode_sstep_en`（`:741-745`）。

**限定逻辑**（`:764-771`）：

```systemverilog
assign csr2hdu_req_qlfy      = csr2hdu_req & dbg_en & pipe2hdu_rdc_qlfy_i; // :764
assign exu_busy_qlfy         = exu_busy & {N{pipe2hdu_rdc_qlfy_i}};        // :766
assign instret_qlfy          = instret & ...;                               // :767
assign exu_init_pc_qlfy      = exu_init_pc & ...;                           // :768
assign exu_exc_req_qlfy      = exu_exc_req & ...;                           // :769
assign brkpt_qlfy            = brkpt & ...;                                 // :770
assign ifu2hdu_pbuf_rdy_qlfy = ifu2hdu_pbuf_rdy & ...;                      // :771
```

### 6.9 追踪日志（`:780-821`，`SCR1_TRGT_SIMULATION`）

`scr1_tracelog` 通过**层次引用**采集内部信号：

- MPRF：`i_pipe_mprf.mprf_int`、写使能/地址/数据（`:788-791`）；
- EXU：`i_pipe_exu.update_pc_en`/`update_pc`（`:794-795`）；
- IFU：`i_pipe_ifu.ifu2idu_instr_o`（`:798`）；
- CSR：MSTATUS/MIE/MTVEC/MIP/MEPC/MCAUSE/MTVAL 及事件（`:801-818`）。

---

## 7. 时钟与复位域

| 模块 | 复位 | 时钟 | 备注 |
|---|---|---|---|
| IFU | `pipe_rst_n` | `clk` | 无条件 |
| IDU | `pipe_rst_n`（仿真） | `clk`（仿真） | 综合为纯组合 |
| EXU | `pipe_rst_n` | `clk`/`clk_alw_on`/`clk_pipe_en` | CLKCTRL |
| MPRF | `pipe_rst_n`（可选） | `clk` | `SCR1_MPRF_RST_EN` |
| CSR | `pipe_rst_n` | `clk`（+`clk_alw_on`） | CLKCTRL |
| IPIC | `pipe_rst_n` | `clk`/`clk_alw_on` | CLKCTRL |
| TDU | `dbg_rst_n`/`pipe_rst_n` | `clk` | DBG 决定 |
| HDU | `dbg_rst_n` | `clk`/`clk_dbgc` | CLKCTRL |

**复位域交叉限定**：HDU/TDU 位于 `dbg_rst_n` 域，来自 `pipe_rst_n` 域的输入
必须经 `pipe2hdu_rdc_qlfy_i` 限定（`*_qlfy`），避免跨域亚稳态。

---

## 8. 与其它模块的职责边界

| 关注点 | 归属 |
|---|---|
| 取指/PC 更新 | IFU（PC 由 EXU 提供 `new_pc`） |
| 译码 | IDU（纯组合） |
| 执行/异常生成 | EXU（内含 IALU、LSU） |
| 操作数/写回 | MPRF |
| 特权状态/trap/counter | CSR |
| 外部中断控制 | IPIC |
| 硬件触发 | TDU |
| 调试 | HDU |
| 顶层互连/控制派生 | `scr1_pipe_top` |

---

## 9. 配置宏影响

| 宏 | 影响范围 |
|---|---|
| `SCR1_IPIC_EN` | IRQ 源、IPIC 例化、CSR 外部中断选择 |
| `SCR1_DBG_EN` | HDU、调试端口、限定逻辑、stop_fetch、wake_req |
| `SCR1_TDU_EN` | TDU、断点接口、EPU 请求 |
| `SCR1_CLKCTRL_EN` | sleep/wake 请求、各模块时钟选择 |
| `SCR1_MPRF_RST_EN` | MPRF 复位连接 |
| `SCR1_CSR_REDUCED_CNT` | 去掉 instret_no_exc、clk_alw_on |
| `SCR1_NO_EXE_STAGE` | 去掉 use_rd/use_imm |
| `SCR1_TRGT_SIMULATION` | IDU 时钟/复位、tracelog |

---

## 附录 A：子模块例化索引

| 例化名 | 模块 | 行号 |
|---|---|---|
| `i_pipe_ifu` | `scr1_pipe_ifu` | 300-335 |
| `i_pipe_idu` | `scr1_pipe_idu` | 340-360 |
| `i_pipe_exu` | `scr1_pipe_exu` | 365-472 |
| `i_pipe_mprf` | `scr1_pipe_mprf` | 477-491 |
| `i_pipe_csr` | `scr1_pipe_csr` | 496-575 |
| `i_pipe_ipic` | `scr1_ipic` | 581-595 |
| `i_pipe_tdu` | `scr1_pipe_tdu` | 602-661 |
| `i_pipe_hdu` | `scr1_pipe_hdu` | 684-762 |
| `i_tracelog` | `scr1_tracelog` | 780-821 |

## 附录 B：顶层派生信号

| 信号 | 定义 | 行号 |
|---|---|---|
| `stop_fetch` | `wfi_run2halt \| fetch_pbuf` | 278-282 |
| `pipe2clkctl_sleep_req_o` | `wfi_halted & ~imem_txns_pending` | 285 |
| `pipe2clkctl_wake_req_o` | `csr2exu_ip_ie \| dm2pipe_active_i` | 286-290 |
| `pipe2dm_pc_sample_o` | `curr_pc` | 294 |
| `hwbrk_dsbl` | `~dbg_en \| hdu_hwbrk_dsbl` | 664 |
| `csr2tdu_req_qlfy` | `dbg_en & csr2tdu_req & qlfy` | 666 |
| `csr2hdu_req_qlfy` | `csr2hdu_req & dbg_en & qlfy` | 764 |
| `exu_busy_qlfy` | `exu_busy & qlfy` | 766 |
