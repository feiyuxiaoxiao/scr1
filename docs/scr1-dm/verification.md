# SCR1 Debug Module 调试模块 验证文档（Verification Document）

- 被测对象：`scr1_dm`（`src/core/scr1_dm.sv`，1427 行，仅 `SCR1_DBG_EN`）
- 配套设计规格：`docs/scr1-dm/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:457-495`，例化名 `i_dm`）

> `SCR1_DBG_EN` 随 `SCR1_CFG_RV32IMC_MAX` 启用（`scr1_arch_description.svh:79`），
> MAX 回归会编译 DM，但调试功能需 JTAG/DMI 激励与软件配置才实际激活。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 DMI 寄存器读写译码与读数据 mux；
2. 验证 DMCONTROL/DMSTATUS/HARTINFO/ABSTRACTCS 字段；
3. 验证抽象命令解码与合法性（type/regsize/memsize/reserved）；
4. 验证抽象命令 FSM 各状态迁移与 ERR 路径；
5. 验证 cmderr/busy 生成与清除；
6. 验证 DATA0/1、COMMAND、ABSTRACTAUTO、PROGBUF 寄存器；
7. 验证 DHI FSM（EXEC/HALT/RESUME）与 HART 命令下发；
8. 验证 PBUF 指令 mux 与 EBREAK 结束；
9. 验证复位/ndmreset 输出。

### 1.2 范围

- CSR/DMI 驱动 + 流水线 HDU/PBUF 交互 + 波形；
- 需在 `SCR1_DBG_EN` 配置下构建。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_dm.*`：`abs_fsm_ff`、`dhi_fsm_ff`、
`abstractcs_cmderr_ff`、`abs_err_acc_busy_ff`、`abs_err_exc_ff`、
`dmcontrol_*_ff`、`dmstatus_allany_*_ff`、`abs_data0_ff`/`abs_data1_ff`、
`abs_progbuf0-5_ff`、`hart_cmd_ff`、`dm2pipe_cmd_req_o`、`dm2pipe_pbuf_instr_o`、
`ndm_rst_n_o`/`hart_rst_n_o`。

---

## 3. 验证策略

1. **DMI 驱动**：通过 TAPC→DMI 发起读/写事务；
2. **命令驱动**：写 COMMAND 触发抽象命令 FSM；
3. **HART 交互**：用 HDU 命令响应/事件驱动 DHI 与 cmderr；
4. **波形核对**：状态迁移、错误码、数据寄存器内容；
5. **内建 SVA**：6 条 X 检查。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-DM-1 | DMI 读 DMCONTROL | 读 0x10 |
| FC-DM-2 | DMI 读 DMSTATUS（VERSION=2、AUTHENTICATED=1） | 读 0x11 |
| FC-DM-3 | DMI 读 HARTINFO（DATAADDR=0x7b2） | 读 0x12 |
| FC-DM-4 | DMI 读 ABSTRACTCS（PROGBUFSIZE=6/DATACOUNT=2） | 读 0x16 |
| FC-DM-5 | DMI 读 DATA0/1、PROGBUF0-5 | 读对应地址 |
| FC-DM-6 | 未知地址返回 0 | 读非法地址 |
| FC-DM-7 | 写 DMCONTROL（dmactive/haltreq/resumereq/ndmreset） | 写 0x10 |
| FC-DM-8 | dmactive=0 时命令位清 0 | 清 dmactive |
| FC-DM-9 | `dm2pipe_active_o` 延迟跟随 dmactive | 波形 |
| FC-DM-10 | ndmreset/hart_rst_n 输出 | 写 ndmreset |
| FC-DM-11 | havereset_skip_pwrup 上电保持 / 运行中清除 | 复位序列 |
| FC-DM-12 | DMSTATUS havereset 置/清（ackhavereset） | 写 ack |
| FC-DM-13 | DMSTATUS resumeack 置（resumereq+run） | resume 序列 |
| FC-DM-14 | DMSTATUS halted 置/清 | halt/run |
| FC-DM-15 | 写 COMMAND、读回 | 写 0x17 |
| FC-DM-16 | ABSTRACTAUTO 位 0 与 autoexec 重放 | 写 0x18 + DATA0 写 |
| FC-DM-17 | PROGBUF 写保护（仅 IDLE） | 非 IDLE 写 |
| FC-DM-18 | 抽象命令 type 非法 → ABS_ERR_CMD | 写非法 type |
| FC-DM-19 | hartmem 未 halt → ABS_ERR_NOHALT | run 态发 mem 命令 |
| FC-DM-20 | mprf/csr 未 halt → NOHALT | run 态发 reg 命令 |
| FC-DM-21 | regsize≠2 → CMD | 写非法 size |
| FC-DM-22 | memsize≥3 → CMD | 写非法 memsize |
| FC-DM-23 | 处理中再访问 → BUSY | 命令执行中发 DMI |
| FC-DM-24 | 抽象指令异常 → CMDERR=EXCEPTION | 触发 dhi_resp_exc |
| FC-DM-25 | 清 cmderr 返回 IDLE | 写 ABSTRACTCS |
| FC-DM-26 | hart 非预期复位 → cmderr=EXCEPTION | 运行中 hart reset |
| FC-DM-27 | CSR 只读访问（hart_state_run） | 读 MISA 等 |
| FC-DM-28 | DATA0 CSR_RO 回填 MISA/MVENDORID/…/PC | 波形 |
| FC-DM-29 | DATA1 用于 FORADDR 态 | 内存访问波形 |
| FC-DM-30 | DHI EXEC 序列（DRUN→DHALTED） | 执行抽象指令 |
| FC-DM-31 | DHI HALT_REQ → DHALTED | haltreq |
| FC-DM-32 | DHI RESUME_REQ → RUN | resumereq |
| FC-DM-33 | `dm2pipe_cmd_o`/`cmd_req` 下发 | 波形 |
| FC-DM-34 | PBUF 按 addr 选 PROGBUF0-5 | 执行 progbuf |
| FC-DM-35 | PBUF 遇 EBREAK 结束 | progbuf 含 EBREAK |
| FC-DM-36 | `abs_exec_instr_ff` 在 addr==0 注入 | 抽象指令执行 |
| FC-DM-37 | dreg resp/fail 恒值、rdata 按 use_addr | 波形 |

---

## 5. 断言验证（Assertions）

`SCR1_TRGT_SIMULATION` 下 `:1386-1423`：

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SVA_DM_X_CONTROL` | 1391-1396 | req/dreg_req/cmd_resp/hart_event 无 X | 通过 |
| `SVA_DM_X_DMI` | 1398-1401 | req 时 wr/addr/wdata 无 X | 通过 |
| `SVA_DM_X_HART_PBUF` | 1403-1406 | pbuf addr 无 X | 通过 |
| `SVA_DM_X_HART_DREG` | 1408-1411 | dreg req 时 wr/wdata 无 X | 通过 |
| `SVA_DM_X_HART_CMD` | 1413-1416 | cmd_resp 时 rcode 无 X | 通过 |
| `SVA_DM_X_HART_EVENT` | 1418-1421 | hart_event 时 status 无 X | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归

`SCR1_CFG_RV32IMC_MAX` 定义 `SCR1_DBG_EN`，`hello` PASS 说明：

- DM 例化与 DMI 接口不破坏正常执行；
- dmactive 复位为 0，未激活调试时抽象命令 FSM 保持 IDLE；
- 内建 X 检查无违例。

### 6.2 波形核对

可核对：

1. 上电后 `dmcontrol_dmactive_ff=0`、`abs_fsm_ff=IDLE`、`dhi_fsm_ff=IDLE`；
2. `dm2pipe_cmd_req_o=0`、`dm2pipe_active_o=0`；
3. `ndm_rst_n_o`/`hart_rst_n_o` 跟随 ndmreset 复位值。
4. 6 条 SVA 无违例（FC 中 1-14 的复位态）。

### 6.3 定向专项（建议补充执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| DMI 寄存器读写 | 经 TAPC 访问各地址 | 读回字段正确 | 1-6 |
| dmactive 控制 | 写 dmactive=1/0 | 命令位联动 | 7-9 |
| halt/resume | 写 haltreq/resumereq | DMSTATUS 联动、命令下发 | 30-33 |
| 抽象 CSR 访问 | halted 后发 CSR 命令 | 经 PBUF 执行并回填 DATA0 | 27-28,36 |
| 抽象内存访问 | halted 后发 mem 命令 | DATA1 参与、正确读写 | 29 |
| 非法命令 | 写非法 type/size | cmderr=CMD | 18,21-22 |
| 未 halt 访问 | run 态发命令 | cmderr=NOHALT | 19-20 |
| busy | 处理中再访问 | cmderr=BUSY | 23 |
| 异常 | 触发 hart 异常 | cmderr=EXCEPTION | 24 |
| 清错误 | 写 ABSTRACTCS | 回 IDLE | 25 |
| ndmreset | 写 ndmreset | 复位输出有效 | 10 |
| PBUF EBREAK | progbuf 末放 EBREAK | 结束执行 | 34-35 |

> 6.3 尚未系统执行。

---

## 7. 回归流程

MAX 配置即含 DM，回归命令同其它模块的 `build_verilator`
（`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`），测试台 `hello`。
DM 的完整功能验证需要 JTAG 激励（TAPC→DMI）与软件交互，波形信号见 2.2 节。

---

## 8. 验证结论

1. DM 在 MAX 配置下编译并通过 hello 回归，6 条内建 SVA 无违例；
2. 设计规格所述的寄存器映射、抽象命令 FSM、cmderr/busy、DHI FSM、PBUF 逻辑与源码一致；
3. 文档中标注的源码注意点（`dm2dmi_resp_o` 恒 1、`memvalid` 单 bit 检查、
   `ndm_rst_n_o` 与 `hart_rst_n_o` 同源）均经源码核对；
4. 调试功能需 JTAG/DMI 激励才激活，建议按 6.3 建立定向测试。

**遗留建议**：建立 JTAG→DMI→DM→HART 全链路的定向测试，覆盖 FC-DM-1~37。
