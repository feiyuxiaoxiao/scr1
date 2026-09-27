# SCR1 TDU 验证文档（Verification Document）

- 被测对象：`scr1_pipe_tdu`（`src/core/pipeline/scr1_pipe_tdu.sv`，610 行，仅 `SCR1_TDU_EN`）
- 配套设计规格：`docs/scr1-tdu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:602-661`）

> TDU 默认随 `SCR1_CFG_RV32IMC_MAX` 启用（`SCR1_TDU_EN`），因此 MAX 回归会
> 编译并覆盖其内部逻辑；但触发功能需软件配置 CSR 才会实际命中。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 TSELECT/TDATA1/TDATA2/TINFO 读写；
2. 验证 MCONTROL 字段写入与固定常量位；
3. 验证 exec 断点、load/store 观察点匹配；
4. 验证 ICOUNT 计数递减与命中；
5. 验证 `tdu_dsbl_i` 禁用、dmode 写保护；
6. 验证动作（异常 vs 调试模式）输出；
7. 验证 CSR SET/CLEAR 语义与响应。

### 1.2 范围

- 通过 CSR 驱动 + EXU/LSU 监视激励 + 波形；
- 需要在 `SCR1_TDU_EN` 配置下构建。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_pipe_top.i_pipe_tdu.*`（`csr_tselect_ff`、
`csr_mcontrol_*_ff`、`csr_tdata2_ff`、`csr_icount_*`、`csr_mcontrol_exec_hit`、
`csr_mcontrol_ldst_hit`、`csr_icount_hit`、`tdu2hdu_dmode_req_o`）。

---

## 3. 验证策略

1. **CSR 驱动**：软件写 TSELECT/TDATA2/MCONTROL/ICOUNT；
2. **激励注入**：经 EXU/LSU 监视接口产生地址事件；
3. **波形核对**：命中条件、hit 位、异常/调试请求；
4. **内建 SVA**：X 检查（`SCR1_TRGT_SIMULATION`）。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-TDU-1 | TSELECT 写选中触发器 | 写 TSELECT |
| FC-TDU-2 | TSELECT 越界索引忽略 | 写非法索引 |
| FC-TDU-3 | TDATA1 读 MCONTROL 类型 | 选 MTRIG 后读 |
| FC-TDU-4 | TDATA1 读 ICOUNT 类型 | 选末位索引后读 |
| FC-TDU-5 | TDATA1 固定常量位（SELECT/TIMING/CHAIN=0） | 读回 |
| FC-TDU-6 | TDATA2 索引读写 | 选索引后写/读 |
| FC-TDU-7 | TINFO 类型位置 1 | 读 TINFO |
| FC-TDU-8 | MCONTROL 写 exec | 写 EXECUTE 位 |
| FC-TDU-9 | MCONTROL 写 load/store | 写 LOAD/STORE 位 |
| FC-TDU-10 | ACTION 字段转单位（==1） | 写 ACTION=1 |
| FC-TDU-11 | hit 位由退休置 1 | 波形 |
| FC-TDU-12 | exec 断点命中 | imon.vd & addr 匹配 |
| FC-TDU-13 | load 观察点命中 | dmon.load |
| FC-TDU-14 | store 观察点命中 | dmon.store |
| FC-TDU-15 | M 位限制（m_ff=0 不命中） | 清 M |
| FC-TDU-16 | `tdu_dsbl_i` 禁用所有命中 | 拉高禁用 |
| FC-TDU-17 | ICOUNT 计数递减 | 指令退休 |
| FC-TDU-18 | ICOUNT 命中（count==1） | 计数到 1 |
| FC-TDU-19 | ICOUNT skip 机制 | 波形 |
| FC-TDU-20 | dmode 触发器写保护 | dmode=1 时写被屏蔽 |
| FC-TDU-21 | dmode 动作 → `tdu2hdu_dmode_req_o` | 退休+action |
| FC-TDU-22 | `tdu2exu_ibrkpt_exc_req_o` 聚合 | 多触发器 OR |
| FC-TDU-23 | `tdu2lsu_dbrkpt_exc_req_o` 聚合 | OR |
| FC-TDU-24 | CSR 响应始终 OK（req 时） | 读响应 |
| FC-TDU-25 | CSR SET 语义 | 写 SET |
| FC-TDU-26 | CSR CLEAR 语义 | 写 CLEAR |
| FC-TDU-27 | 无 ICOUNT 编译分支（`SCR1_TDU_ICOUNT_EN` 关闭） | 配置构建 |
| FC-TDU-28 | `SCR1_CSR_ADDR_TDU_OFFS` 地址映射 | 读各偏移 |

---

## 5. 断言验证（Assertions）

`SCR1_TRGT_SIMULATION` 下 `:544-606`：

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SVA_TDU_X_CONTROL` | 549-554 | 控制信号无 X | 通过 |
| `SVA_DM_X_CLK_EN` | 556-559 | `clk_en` 无 X | 通过 |
| `SVA_DM_X_DSBL` | 561-564 | `tdu_dsbl_i` 无 X | 通过 |
| `SVA_DM_X_CSR2TDU_REQ` | 566-569 | `csr2tdu_req_i` 无 X | 通过 |
| `SVA_DM_X_I_MON_VD` | 571-574 | `imon.vd` 无 X | 通过 |
| `SVA_DM_X_D_MON_VD` | 576-579 | `dmon.vd` 无 X | 通过 |
| `SVA_DM_X_BP_RETIRE` | 581-584 | `bp_retire` 无 X | 通过 |
| `SVA_TDU_X_CSR` | 586-589 | req 时 cmd/addr 无 X | 通过 |
| `SVA_TDU_XW_CSR` | 591-594 | 写时 wdata 无 X | 通过 |
| `SVA_TDU_X_IMON` | 596-599 | imon.vd 时 req/addr 无 X | 通过 |
| `SVA_TDU_X_DMON` | 601-604 | dmon.vd 时无 X | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归

`SCR1_CFG_RV32IMC_MAX` 默认定义 `SCR1_TDU_EN`，`hello` PASS 表明：

- TDU 例化与 CSR 地址译码不破坏正常执行；
- ICOUNT/触发器复位为 0，未配置时不产生伪命中（FC-TDU-16 的 tdu_dsbl 初始态由 `hwbrk_dsbl` 控制）；
- 内建 X 检查断言无违例。

### 6.2 波形核对

可核对：

1. 上电后 `csr_tselect_ff=0`、`csr_mcontrol_*_ff=0`，无命中（FC-TDU-12/16）；
2. `tdu2exu_ibrkpt_exc_req_o=0`（FC-TDU-22）；
3. `tdu2hdu_dmode_req_o=0`（FC-TDU-21）。

### 6.3 定向专项（建议补充执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| exec 断点 | 写 TDATA2=目标地址、MCONTROL.exec/M=1 | 命中并请求异常 | 12 |
| load 观察点 | 写 LOAD=1 | 命中 | 13 |
| 计数器单步 | ICOUNT.count=1，M=1 | 一次退休后命中 | 18 |
| dmode 写保护 | 写 dmode=1 后再写 | 被屏蔽 | 20 |
| 调试重定向 | ACTION=1 | `dmode_req` 拉高 | 21 |
| 禁用 | 拉高 `tdu_dsbl_i` | 无命中 | 16 |
| 无 ICOUNT | 关闭 `SCR1_TDU_ICOUNT_EN` | 单触发器位宽 | 27 |

> 6.3 尚未系统执行。

---

## 7. 回归流程

MAX 配置即含 TDU，回归命令同其它模块的 `build_verilator`（`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`），
测试台 `hello`。波形核对信号见 2.2 节。

---

## 8. 验证结论

1. TDU 在 MAX 配置下编译并通过 hello 回归，内建 SVA 无违例；
2. 设计规格所述的匹配条件、dmode 写保护、ICOUNT 递减与聚合输出均与源码一致；
3. 触发功能需软件配置 CSR 才激活，建议按 6.3 建立定向测试。

**遗留建议**：建立触发/观察点/ICOUNT 定向测试，覆盖 FC-TDU-1~28。
