# SCR1 HDU 验证文档（Verification Document）

- 被测对象：`scr1_pipe_hdu`（`src/core/pipeline/scr1_pipe_hdu.sv`，904 行，仅 `SCR1_DBG_EN`）
- 配套设计规格：`docs/scr1-hdu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:684-762`）

> HDU 在 MAX 配置下编译（`SCR1_DBG_EN`），hello 回归覆盖其接口与复位态；
> 完整调试流程需 DM 侧激励（`scr1_dm` + DMI），属系统级调试场景。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证调试状态机 RESET/RUN/DHALTED/DRUN 转移与握手（trans/update/event）；
2. 验证停机请求、超时计数器、停机原因优先级；
3. 验证 Run Control 寄存器在 RUN/DRUN 恢复时的配置；
4. 验证 Program Buffer FSM 与地址递增/异常注入；
5. 验证 DCSR/DPC/DSCRATCH0 读写；
6. 验证 DM/EXU/IFU/CSR/TDU 接口时序；
7. 验证跨复位域限定（`pipe2hdu_rdc_qlfy_i`）。

### 1.2 范围

- HDU 无独立测试台；由系统调试测试（DM 驱动）+ 波形 + 内建 SVA 覆盖；
- 默认 hello 仅覆盖复位态与接口无 X。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_pipe_top.i_pipe_hdu.*`（`dbg_state`、`dbg_state_next`、
`dfsm_trans/update/event`、`hart_runctrl`、`pbuf_fsm_curr`、`pbuf_addr_ff`、
`hart_haltcause`、`csr_dcsr_*`、`csr_dpc_ff`、`hdu2exu_*`）。

---

## 3. 验证策略

1. **DM 激励**：经 `dm2hdu_cmd_i` 请求停机/运行；
2. **CSR 访问**：读写 DCSR/DPC/DSCRATCH0；
3. **波形核对**：状态转移、握手、超时、Program Buffer；
4. **内建 SVA**：接口 X 检查。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-HDU-1 | RESET 等待 init_pc | 复位后波形 |
| FC-HDU-2 | RESET→RUN | init_pc 且无停机请求 |
| FC-HDU-3 | RESET→DHALTED | init_pc + dm_dhalt_req |
| FC-HDU-4 | RUN→DHALTED | 停机确认 |
| FC-HDU-5 | DHALTED→RUN | 非 drun 运行请求 |
| FC-HDU-6 | DHALTED→DRUN | drun 运行请求 |
| FC-HDU-7 | DRUN→DHALTED | 停机确认 |
| FC-HDU-8 | dfsm trans/update 两拍握手 | 波形 |
| FC-HDU-9 | `hart_event` 脉冲 | DM 观察 |
| FC-HDU-10 | 停机超时计数器归零 | 持续停机请求 |
| FC-HDU-11 | 超时后强制停机 | 波形 |
| FC-HDU-12 | EXU 空闲+有 cause 立即停机 | 波形 |
| FC-HDU-13 | 停机原因 NONE/EBREAK/DMREQ/SSTEP | 各场景 |
| FC-HDU-14 | 停机原因 TMREQ（TDU） | TDU 触发 |
| FC-HDU-15 | 原因优先级 TMREQ>EBREAK>DMREQ>SSTEP | 多原因 |
| FC-HDU-16 | RUN 恢复：irq_dsbl 依 step/stepie | 波形 |
| FC-HDU-17 | RUN 恢复：redirect.ebreak=ebreakm | 波形 |
| FC-HDU-18 | DRUN 恢复：irq_dsbl=1、fetch=PBUF | 波形 |
| FC-HDU-19 | DRUN 恢复：pc_advmt_dsbl/hwbrkpt_dsbl=1 | 波形 |
| FC-HDU-20 | PBUF FSM IDLE→FETCH→WAIT4END→IDLE | 调试运行 |
| FC-HDU-21 | PBUF 地址递增与末尾判定 | 波形 |
| FC-HDU-22 | PBUF 异常注入（EXCINJECT） | 末尾无异常 |
| FC-HDU-23 | PBUF 指令经寄存器/直连两种配置 | 参数切换 |
| FC-HDU-24 | DCSR 读 XDEBUGVER/prv=11 | 读 DCSR |
| FC-HDU-25 | DCSR 写 ebreakm/stepie/step | 写 DCSR |
| FC-HDU-26 | DCSR.cause 入停机时更新 | 读 DCSR |
| FC-HDU-27 | DPC 入停机保存 pc_curr | 读 DPC |
| FC-HDU-28 | DPC 软件写 | 写 DPC |
| FC-HDU-29 | DSCRATCH0 经 DM 数据接口 | 写/读 |
| FC-HDU-30 | 仅 DRUN 下 Debug CSR 可访问 | 非 DRUN 拒绝 |
| FC-HDU-31 | `no_commit` 在 ebreak/tmreq 拉高 | 波形 |
| FC-HDU-32 | `dbg_new_pc = csr_dpc_ff` | 恢复 |
| FC-HDU-33 | `hdu2ifu_pbuf_instr_err` 在 EXCINJECT | 波形 |
| FC-HDU-34 | 复位域限定：`~qlfy` 强制 RESET | 波形 |
| FC-HDU-35 | `hdu2tdu_hwbrk_dsbl` 来源 runctrl | 波形 |

---

## 5. 断言验证（Assertions）

`SCR1_TRGT_SIMULATION` 下 `:858-900`：

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SVA_HDU_XCHECK_COMMON` | 863-868 | 公共信号无 X | 通过 |
| `SVA_HDU_XCHECK_CSR_INTF` | 870-875 | CSR 接口无 X | 通过 |
| `SVA_HDU_XCHECK_DM_INTF` | 877-883 | DM 接口无 X | 通过 |
| `SVA_HDU_XCHECK_TDU_INTF` | 885-890 | TDU 接口无 X | 通过 |
| `SVA_HDU_XCHECK_HART_INTF` | 892-898 | HART 接口无 X | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归

`SCR1_CFG_RV32IMC_MAX` 定义 `SCR1_DBG_EN`，hello PASS 表明：

- HDU 例化、`dbg_state=RESET`、接口无 X（5 条 SVA 通过）；
- `hdu2exu_no_commit_o=0`、`hdu2ifu_pbuf_instr_vd_o=0`，不干扰正常执行。

### 6.2 波形核对

可核对：

1. 复位后 `dbg_state=RESET`，`init_pc` 后转 RUN（FC-HDU-1/2）；
2. `hdu2dm_hart_status_o.dbg_state` 正确反映（FC-HDU-2）；
3. 无 DM 命令时 `hdu2dm_cmd_resp_o` 按状态判定（FC-HDU-8）。

### 6.3 系统级调试专项（建议补充执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| 停机/恢复 | DM 发 HALTED/RUN 命令 | 状态往返 | 3-8 |
| 单步 | DCSR.step=1 运行 | 一次退休后停机 | 13/16 |
| EBREAK 停机 | ebreakm=1 + `ebreak` | 停机 cause=EBREAK | 13/17 |
| Program Buffer | DRUN 执行 PBUF | 指令注入并结束 | 20-23 |
| Debug CSR | DRUN 读 DCSR/DPC | 正确 | 24-30 |
| TDU 触发 | 硬件断点 | cause=TMREQ | 14/15 |
| 超时 | 持续停机请求 | 超时后停机 | 10/11 |

> 6.3 依赖 `scr1_dm`/DMI 系统激励，本次未执行。

---

## 7. 回归流程

MAX 配置即含 HDU，回归命令同其它模块的 `build_verilator`
（`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`），测试台 `hello`。波形核对信号见 2.2 节。

---

## 8. 验证结论

1. HDU 在 MAX 配置下编译并通过 hello 回归，5 条内建 SVA 无违例；
2. 状态机、停机原因优先级、Run Control、Program Buffer、Debug CSR 语义与源码一致；
3. 完整调试流程需 DM 侧激励，建议按 6.3 建立系统级调试测试。

**遗留建议**：基于 `scr1_dm` 建立停机/恢复/单步/Program Buffer/Debug CSR
系统级测试，覆盖 FC-HDU-1~35。
