# SCR1 TDU 触发调试单元 设计规格（Design Specification）

- 模块：`scr1_pipe_tdu`（`src/core/pipeline/scr1_pipe_tdu.sv`，610 行）
- 条件：仅在 `SCR1_TDU_EN` 时编译（`:33`、`:610`）
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:602-661`）
- 职责：提供触发/观察点 CSR，监控指令地址流与数据地址流，产生断点异常或调试模式重定向请求

---

## 1. 概述与职责

TDU（Trigger Debug Unit）实现 RISC-V Debug 规范的硬件触发器功能：

1. **CSR 接口**：TSELECT、TDATA1(=MCONTROL/ICOUNT)、TDATA2、TINFO；
2. **指令断点**：指令地址与 TDATA2 匹配（exec 类型）；
3. **数据观察点**：load/store 地址与 TDATA2 匹配；
4. **指令计数触发**：ICOUNT，计数到值触发（单步）；
5. **动作**：触发时产生断点异常（exception）或进入调试模式（dmode）；
6. **模式限制**：触发器可限定仅在 M 模式或仅 Debug 模式生效。

---

## 2. 端口（`:37-69`）

| 信号 | 方向 | 行号 |
|---|---|---|
| `rst_n` | in | 39 |
| `clk` | in | 40 |
| `clk_en` | in | 41 |
| `tdu_dsbl_i` | in | 42 |
| `csr2tdu_req_i` | in | 45 |
| `csr2tdu_cmd_i`（`type_scr1_csr_cmd_sel_e`） | in | 46 |
| `csr2tdu_addr_i[SCR1_CSR_ADDR_TDU_OFFS_W-1:0]` | in | 47 |
| `csr2tdu_wdata_i[SCR1_TDU_DATA_W-1:0]` | in | 48 |
| `tdu2csr_rdata_o[SCR1_TDU_DATA_W-1:0]` | out | 49 |
| `tdu2csr_resp_o`（`type_scr1_csr_resp_e`） | out | 50 |
| `exu2tdu_imon_i`（`type_scr1_brkm_instr_mon_s`） | in | 53 |
| `tdu2exu_ibrkpt_match_o[SCR1_TDU_ALLTRIG_NUM-1:0]` | out | 54 |
| `tdu2exu_ibrkpt_exc_req_o` | out | 55 |
| `exu2tdu_bp_retire_i[SCR1_TDU_ALLTRIG_NUM-1:0]` | in | 56 |
| `tdu2lsu_ibrkpt_exc_req_o` | out | 62 |
| `lsu2tdu_dmon_i`（`type_scr1_brkm_lsu_mon_s`） | in | 63 |
| `tdu2lsu_dbrkpt_match_o[SCR1_TDU_MTRIG_NUM-1:0]` | out | 64 |
| `tdu2lsu_dbrkpt_exc_req_o` | out | 65 |
| `tdu2hdu_dmode_req_o` | out | 68 |

### 死代码提示

`:59-61` 的 `tdu2lsu_brk_en_o` 位于 `ifndef SCR1_TDU_EN` 内，而整个文件已在
`ifdef SCR1_TDU_EN` 中（`:33`），该分支**永不生效**；`:525-527` 同理。属遗留代码。

---

## 3. 参数与 CSR 映射（`scr1_tdu.svh`）

| 参数 | 值 | 行号 |
|---|---|---|
| `SCR1_TDU_MTRIG_NUM` | `SCR1_TDU_TRIG_NUM` | :18 |
| `SCR1_TDU_ALLTRIG_NUM` | MTRIG(+1 若 ICOUNT) | :19-23 |
| `SCR1_TDU_DATA_W` | `XLEN` | :26 |
| `SCR1_CSR_ADDR_TDU_OFFS_W` | 3 | :29 |
| OFFS_TSELECT/TDATA1/TDATA2/TINFO | 0/1/2/4 | :30-33 |

- `SCR1_TDU_TRIG_NUM`：MAX 配置 = 4（`scr1_arch_description.svh:81`）、BASE 配置 = 2（`:96`）、自定义默认 2（`:145`）；
- `SCR1_TDU_ICOUNT_EN`：MAX/BASE/自定义均开启（`:82`/`:97`/`:146`）。

TDATA1 字段（`:42-68`）：TYPE[XLEN-1:XLEN-4]、DMODE[XLEN-5]、MASKMAX、
HIT=20、ACTION[17:12]、M=6、EXECUTE=2、STORE=1、LOAD=0；
类型常量 MCONTROL=2（`:72`）、ICOUNT=3（`:98`）。

监控结构体（`:106-117`）：`type_scr1_brkm_instr_mon_s{vd,req,addr}`、
`type_scr1_brkm_lsu_mon_s{vd,load,store,addr}`。

---

## 4. 局部参数（`:71-77`）

```systemverilog
localparam MTRIG_NUM   = SCR1_TDU_MTRIG_NUM;
localparam ALLTRIG_NUM = SCR1_TDU_ALLTRIG_NUM;
localparam ALLTRIG_W   = $clog2(ALLTRIG_NUM+1);
```

---

## 5. CSR 读写接口（`:159-305`）

### 5.1 响应（`:166`）

```systemverilog
assign tdu2csr_resp_o = csr2tdu_req_i ? SCR1_CSR_RESP_OK : SCR1_CSR_RESP_ER;
```

### 5.2 读多路器（`:168-239`）

`case(csr2tdu_addr_i)`：

| 地址 | 内容 |
|---|---|
| TSELECT | `csr_tselect_ff` |
| TDATA2 | 选中触发器的 `csr_tdata2_ff` |
| TDATA1 | 若选中 MCONTROL：拼装 TYPE/DMODE/MASKMAX/HIT/ACTION/M/EXECUTE/STORE/LOAD；若选中 ICOUNT：拼装 ICOUNT 字段 |
| TINFO | 对应 type 位置 1 |

固定常量位（`:186-201`）：SELECT=0、TIMING=0、CHAIN=0、MATCH=0、RESERVEDA=0、
S=0、U=0。

### 5.3 写数据（`:244-264`）

```systemverilog
SCR1_CSR_CMD_WRITE: csr_wr_req=1;  csr_wr_data=wdata;
SCR1_CSR_CMD_SET  : csr_wr_req=|wdata; csr_wr_data=rdata | wdata;
SCR1_CSR_CMD_CLEAR: csr_wr_req=|wdata; csr_wr_data=rdata & ~wdata;
```

### 5.4 寄存器选择（`:269-305`）

`case(addr)` 置位 `csr_addr_tselect`/`csr_addr_mcontrol[i]`/`csr_addr_tdata2[i]`/
`csr_addr_icount`（后两者按 TSELECT 索引）。

---

## 6. 寄存器实现

### 6.1 TSELECT（`:317-330`）

```systemverilog
csr_tselect_upd = clk_en & csr_addr_tselect & csr_wr_req
                & (csr_wr_data[ALLTRIG_W-1:0] < ALLTRIG_W'(ALLTRIG_NUM));
```

越界索引被忽略（不更新）。

### 6.2 ICOUNT（`:332-392`，`SCR1_TDU_ICOUNT_EN`）

- `csr_icount_clk_en = clk_en & (wr_req | m_ff)`（`:339`）；
- `csr_icount_upd`：dmode=0 时可直接写，dmode=1 时需 `tdu_dsbl_i`（`:340-342`）；
- 递减使能（`:362-364`）：M 模式且未禁用，且 `monitor.vd` 且计数非 0；
- **递减**（`:365`）：`imon.req & decr_en & ~skip_ff`；
- **skip 禁用**（`:366`）：`imon.req & decr_en & skip_ff` → 清 skip；
- next（`:368-387`）：
  - 写：字段来自 `csr_wr_data`（ACTION 转为 `==1` 的单位）；
  - 否则：`hit` 由 `bp_retire[last]` 置 1；count 若 `count_decr` 减 1；
- skip next（`:389-391`）：写时取 M 位，skip_dsbl 清 0，否则保持。

### 6.3 MCONTROL（`:394-467`，generate 每触发器一份）

- `wr_req = addr_mcontrol & wr_req`（`:403`）；
- `clk_en = clk_en & (wr_req | m_ff)`（`:404-405`）；
- `upd`：dmode=0 可直接写，dmode=1 需 `tdu_dsbl_i`（`:406-408`）；
- 字段：dmode/m/exec/load/store/action(`==1`)/hit（`:430-451`）；
- hit：非写时由 `bp_retire[trig]` 置 1（`:447-449`）。

### 6.4 TDATA2（`:453-464`）

```systemverilog
csr_tdata2_upd[trig] = ~dmode_ff ? clk_en & addr_tdata2 & wr_req
                                 : clk_en & addr_tdata2 & wr_req & tdu_dsbl_i;
always_ff @(posedge clk) if (upd) csr_tdata2_ff[trig] <= csr_wr_data;  // 无复位
```

---

## 7. 触发/观察点判定（`:469-523`）

### 7.1 指令计数命中（`:473-475`）

```systemverilog
csr_icount_hit = ~tdu_dsbl_i & csr_icount_m_ff
               ? exu2tdu_imon_i.vd & (csr_icount_count_ff == 14'b1) & ~csr_icount_skip_ff
               : 1'b0;
```

### 7.2 EXU 输出（`:477-483`）

```systemverilog
`ifndef SCR1_TDU_ICOUNT_EN
tdu2exu_ibrkpt_match_o   = csr_mcontrol_exec_hit;
tdu2exu_ibrkpt_exc_req_o = |csr_mcontrol_exec_hit;
`else
tdu2exu_ibrkpt_match_o   = {csr_icount_hit, csr_mcontrol_exec_hit};
tdu2exu_ibrkpt_exc_req_o = |csr_mcontrol_exec_hit | csr_icount_hit;
`endif
```

### 7.3 执行断点命中（`:492-500`）

```systemverilog
csr_mcontrol_exec_hit[trig] = ~tdu_dsbl_i
                            & csr_mcontrol_m_ff[trig]
                            & csr_mcontrol_exec_ff[trig]
                            & exu2tdu_imon_i.vd
                            & exu2tdu_imon_i.addr == csr_tdata2_ff[trig];
```

> `==` 优先级高于 `&`，故为 `... & (addr == tdata2)`。

### 7.4 LSU 异常请求（`:502-506`）

同 EXU 聚合，含 icount 时 OR 之。

### 7.5 数据观察点命中（`:511-520`）

```systemverilog
csr_mcontrol_ldst_hit[trig] = ~tdu_dsbl_i
                            & csr_mcontrol_m_ff[trig]
                            & lsu2tdu_dmon_i.vd
                            & ((load_ff & dmon.load) | (store_ff & dmon.store))
                            & lsu2tdu_dmon_i.addr == csr_tdata2_ff[trig];
```

匹配输出：`tdu2lsu_dbrkpt_match_o = csr_mcontrol_ldst_hit`、
`tdu2lsu_dbrkpt_exc_req_o = |csr_mcontrol_ldst_hit`（`:522-523`）。

---

## 8. TDU ↔ HDU（`:529-542`）

```systemverilog
tdu2hdu_dmode_req_o = |{for all trig: action_ff[i] & bp_retire[i]}
`ifdef ICOUNT
                    | (csr_icount_action_ff & bp_retire[last])
`endif
                    ;
```

任一“动作为进入调试模式”的触发器退休时，请求 HDU 进入调试模式。

---

## 9. 假设与约束

1. `tdu_dsbl_i` 为高时所有触发/观察点失效（`exec_hit`/`ldst_hit`/`icount_hit` 均门控）；
2. dmode 触发器的 CSR 写入在未禁用 TDU 时被屏蔽（写保护）；
3. `csr_tdata2_ff` 无复位（TDATA2 视为普通数据）；
4. ICOUNT 常与单步配合（count=1）；
5. 触发匹配为精确地址相等（无 mask，MASKMAX 硬连线 0）。

---

## 10. 内建断言（`:544-606`，`SCR1_TRGT_SIMULATION`）

| 断言 | 行号 | 检查 |
|---|---|---|
| `SVA_TDU_X_CONTROL` | 549-554 | 控制信号无 X |
| `SVA_DM_X_CLK_EN` | 556-559 | `clk_en` 无 X |
| `SVA_DM_X_DSBL` | 561-564 | `tdu_dsbl_i` 无 X |
| `SVA_DM_X_CSR2TDU_REQ` | 566-569 | `csr2tdu_req_i` 无 X |
| `SVA_DM_X_I_MON_VD` | 571-574 | `imon.vd` 无 X |
| `SVA_DM_X_D_MON_VD` | 576-579 | `dmon.vd` 无 X |
| `SVA_DM_X_BP_RETIRE` | 581-584 | `bp_retire` 无 X |
| `SVA_TDU_X_CSR` | 586-589 | req 时 cmd/addr 无 X |
| `SVA_TDU_XW_CSR` | 591-594 | 写时 wdata 无 X |
| `SVA_TDU_X_IMON` | 596-599 | `imon.vd` 时 req/addr 无 X |
| `SVA_TDU_X_DMON` | 601-604 | `dmon.vd` 时无 X |

---

## 附录 A：寄存器速查

| 寄存器 | 写效果 | 读效果 |
|---|---|---|
| TSELECT | 选触发器（越界忽略） | 当前选择 |
| TDATA1/MCONTROL | 配 dmode/m/exec/load/store/action | 状态与命中 |
| TDATA1/ICOUNT | 配 count/action/m | 计数与命中 |
| TDATA2 | 写匹配地址 | 读匹配地址 |
| TINFO | 只读 | 类型位 |

## 附录 B：信号→消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `tdu2exu_ibrkpt_exc_req_o` | EXU | 指令断点异常 |
| `tdu2lsu_ibrkpt_exc_req_o` | LSU | 指令断点异常（load/store 前） |
| `tdu2lsu_dbrkpt_exc_req_o` | LSU | 数据观察点异常 |
| `tdu2hdu_dmode_req_o` | HDU | 进入调试模式 |
| `tdu2csr_rdata_o`/`resp_o` | CSR | TDU CSR 读写 |
