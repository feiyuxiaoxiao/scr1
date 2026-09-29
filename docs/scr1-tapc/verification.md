# SCR1 TAPC TAP 控制器 验证文档（Verification Document）

- 被测对象：`scr1_tapc`（`src/core/scr1_tapc.sv`，457 行，仅 `SCR1_DBG_EN`）
- 配套设计规格：`docs/scr1-tapc/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:362-383`，例化名 `i_tapc`）

> `SCR1_DBG_EN` 随 `SCR1_CFG_RV32IMC_MAX` 启用；TAPC 工作在 TCK 域，
> 功能验证需 JTAG（TMS/TDI/TCK）激励。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 TRST 同步复位与 `trst_n_int` 行为；
2. 验证 TAP FSM 16 态迁移（含 DR/IR 全路径）；
3. 验证 IR 移位/更新与复位为 IDCODE；
4. 验证指令译码（DR 选择与链 ID）；
5. 验证 BYPASS/IDCODE/BUILD ID 数据寄存器；
6. 验证 TDO 使能与输出时序；
7. 验证 DMI/SCU 链控制信号转发。

### 1.2 范围

- JTAG 激励驱动 + 波形 + 2 条内建 SVA；需 `SCR1_DBG_EN` 配置。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_tapc.*`：`tap_fsm_ff`、`trst_n_int`、`tap_ir_ff`、
`tap_ir_shift_ff`、`tap_fsm_dr_capture_ff`/`_shift_ff`/`_update_ff`、
`tdo_en_ff`/`tdo_out_ff`、`dr_bypass_sel`/`dr_idcode_sel`/`dr_bld_id_sel`、
`tapc2tapcsync_ch_id_o`。

---

## 3. 验证策略

1. **复位**：拉低 TRST 观察 FSM 回 RESET、IR 回 IDCODE；
2. **状态遍历**：按 TMS 序列走 DR/IR 的 capture→shift→update/pause/exit1/exit2；
3. **指令扫描**：移入各指令码，核对 DR 选择与链 ID；
4. **TDO**：在 shift 态核对输出与使能；
5. **内建 SVA**：2 条 X 检查。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-TAPC-1 | TRST 异步复位 | 拉低 TRST |
| FC-TAPC-2 | `trst_n_int` 随 RESET 态释放 | 波形 |
| FC-TAPC-3 | FSM RESET→IDLE | TMS=0 |
| FC-TAPC-4 | DR-SEL-SCAN→DR-CAPTURE→DR-SHIFT | TMS 序列 |
| FC-TAPC-5 | DR-EXIT1→DR-PAUSE→DR-EXIT2 | TMS 序列 |
| FC-TAPC-6 | DR-UPDATE | TMS 序列 |
| FC-TAPC-7 | IR-SEL-SCAN→IR-CAPTURE→IR-SHIFT | TMS 序列 |
| FC-TAPC-8 | IR-UPDATE 载入指令 | TMS 序列 |
| FC-TAPC-9 | IR 复位为 IDCODE(0x01) | 复位 |
| FC-TAPC-10 | IR capture 载 1 到 LSB | capture |
| FC-TAPC-11 | IR shift 串入 TDI | shift |
| FC-TAPC-12 | 指令 IDCODE 选 DR | 移入 0x01 |
| FC-TAPC-13 | 指令 BLD_ID(0x04) | 移入 0x04 |
| FC-TAPC-14 | 指令 SCU_ACCESS(0x09) | 移入 0x09 |
| FC-TAPC-15 | 指令 DTMCS(0x10) | 移入 0x10 |
| FC-TAPC-16 | 指令 DMI_ACCESS(0x11) | 移入 0x11 |
| FC-TAPC-17 | 指令 BYPASS(0x1F) | 移入 0x1F |
| FC-TAPC-18 | 未知指令回退 BYPASS | 移入非法码 |
| FC-TAPC-19 | 链 ID：DTMCS→1、DMI→2 | 移入对应指令 |
| FC-TAPC-20 | BYPASS DR 读写 1 位 | shift |
| FC-TAPC-21 | IDCODE DR 载入 fuse | capture |
| FC-TAPC-22 | BUILD ID DR 载入 MIMPID | capture |
| FC-TAPC-23 | TDO 使能 = dr_shift\|ir_shift | shift 态 |
| FC-TAPC-24 | TDO 输出 DR 数据 / IR LSB | shift 态 |
| FC-TAPC-25 | DMI/SCU capture/shift/update 转发 | 波形 |
| FC-TAPC-26 | DR 与 IR 的 PAUSE 保持 | TMS=0 |

---

## 5. 断言验证（Assertions）

`SCR1_TRGT_SIMULATION` 下 `:437-453`：

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SCR1_SVA_TAPC_XCHECK` | 443-446 | posedge tck 时 tms/tdi 无 X | 通过 |
| `SCR1_SVA_TAPC_XCHECK_NEGCLK` | 448-451 | DR_SHIFT 时链 TDO 无 X | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归

`SCR1_CFG_RV32IMC_MAX` 定义 `SCR1_DBG_EN`，`hello` PASS 说明：

- TAPC 例化与同步器/DMI/SCU 连线不破坏正常执行；
- 无 JTAG 活动时 TAP FSM 停在 RESET 态，`tapc_tdo_en=0`；
- 2 条 SVA 无违例（需在 DR_SHIFT 时才会激活第二条）。

### 6.2 静态核对

- 状态枚举（16 态）、指令编码、宽度参数与 `scr1_tapc.svh` 一致；
- 控制信号基于 `tap_fsm_next`、TDO 基于 `ff`，时序关系与源码一致；
- `SCR1_TAP_DR_IDCODE_WIDTH`/`BLD_ID`/`BYPASS`（32/32/1）。

### 6.3 定向专项（建议执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| 复位 | TRST 低 | FSM=RESET，IR=IDCODE | 1-2,9 |
| IDCODE 读 | 移入 0x01 后 capture/shift | 输出 fuse IDCODE | 12,21 |
| BUILD ID 读 | 移入 0x04 | 输出 MIMPID | 13,22 |
| DTMCS 访问 | 移入 0x10 | ch_id=1、dmi_sel=1 | 15,19 |
| DMI 访问 | 移入 0x11 | ch_id=2、dmi_sel=1 | 16,19 |
| SCU 访问 | 移入 0x09 | scu_sel=1 | 14,19 |
| BYPASS | 移入 0x1F | 1 位旁路 | 17,20 |
| 非法指令 | 移入其它 | 回退 BYPASS | 18 |

> 6.3 尚未系统执行。

---

## 7. 回归流程

MAX 配置即含 TAPC，回归命令同其它模块（`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`），
测试台 `hello`。完整功能验证需 JTAG 激励，波形信号见 2.2 节。

---

## 8. 验证结论

1. TAPC 在 MAX 配置下编译并通过 hello 回归，2 条内建 SVA 无违例；
2. 设计规格所述的 FSM 状态图、IR/DR 行为、指令译码、TDO 时序、
   链转发均与源码一致；
3. 无 JTAG 活动时 TAPC 处于复位态、不影响核心执行；
4. 建议按 6.3 建立 JTAG 时序定向测试。

**遗留建议**：建立 JTAG 扫描（状态图遍历 + 指令/DR 读写）定向测试，覆盖 FC-TAPC-1~26。
