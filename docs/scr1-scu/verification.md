# SCR1 SCU 系统控制单元 验证文档（Verification Document）

- 被测对象：`scr1_scu`（`src/core/scr1_scu.sv`，516 行，仅 `SCR1_DBG_EN`）
- 配套设计规格：`docs/scr1-scu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:193-231`，例化名 `i_scu`）

> `SCR1_DBG_EN` 随 `SCR1_CFG_RV32IMC_MAX` 启用，MAX 回归会编译 SCU；
> 复位生成贯穿整个复位序列，是核心功能路径的一部分。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 TAPC scan-chain（ch_id=0）移位/捕获/更新；
2. 验证 SCU CSR 读写：CONTROL/MODE/STATUS/STICKY 及 READ/SETBITS/CLRBITS 语义；
3. 验证四路复位生成与级联关系（sys→core→hdu/dm）；
4. 验证 MODE 位对 HDU/DM 复位行为的影响；
5. 验证复位状态与 qualifier 的时序（qualifier 先于复位变化）；
6. 验证 STICKY_STATUS 置位与清除。

### 1.2 范围

- TAPC 激励 + 复位序列波形 + 内建 SVA；需 `SCR1_DBG_EN` 配置。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_scu.*`：`tapc_shift_ff`、`tapc_shadow_ff`、
`scu_control_ff`、`scu_mode_ff`、`scu_status_ff`、`scu_sticky_sts_ff`、
`sys_rst_n_o`、`core_rst_n_o`、`hdu_rst_n_o`、`dm_rst_n_o` 与各 qualifier。

---

## 3. 验证策略

1. **chain 事务**：capture→shift→update 写/读各 CSR；
2. **复位序列**：观察上电与 ndmreset 下的四路复位级联；
3. **模式配置**：写 MODE 后重新触发复位，观察 HDU/DM 是否被屏蔽；
4. **内建 SVA**：6 条（X 检查 + 5 条 qualifier 时序）。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-SCU-1 | ch_id=0 选中 SCU 链 | 设置 ch_id |
| FC-SCU-2 | capture 载入影子值 | capture |
| FC-SCU-3 | shift 串入 TDI | shift |
| FC-SCU-4 | TDO = shift_ff[0] | 波形 |
| FC-SCU-5 | update 拷贝 op/addr、data=rdata | update |
| FC-SCU-6 | 写 CONTROL（sys/core_reset） | WRITE addr=0 |
| FC-SCU-7 | 读 CONTROL | READ addr=0 |
| FC-SCU-8 | 写 MODE（hdu/dm_rst_bhv） | WRITE addr=1 |
| FC-SCU-9 | SETBITS 语义 | op=2 |
| FC-SCU-10 | CLRBITS 语义 | op=3 |
| FC-SCU-11 | 读 STATUS | READ addr=2 |
| FC-SCU-12 | STICKY 仅在 CLRBITS 时写 | op=3 addr=3 |
| FC-SCU-13 | 读 STICKY | READ addr=3 |
| FC-SCU-14 | sticky 位由复位上升沿置 1 | 触发复位 |
| FC-SCU-15 | sticky 位由 CLRBITS 清除 | op=3 |
| FC-SCU-16 | System 复位生成（sys_reset 位） | 写 CONTROL |
| FC-SCU-17 | Core 复位随 System 级联 | 波形 |
| FC-SCU-18 | ndm_rst_n_i 参与 System 复位 | DM ndmreset |
| FC-SCU-19 | hart_rst_n_i 参与 Core 复位 | DM hart reset |
| FC-SCU-20 | HDU 复位受 MODE 屏蔽 | 写 MODE |
| FC-SCU-21 | DM 复位受 MODE 屏蔽 | 写 MODE |
| FC-SCU-22 | `sys_rst_status_o`/`core_rst_status_o` 极性 | 波形 |
| FC-SCU-23 | core 三个 qualifier 同源 | 波形 |
| FC-SCU-24 | qualifier 先于复位下降（SVA） | 断言 |
| FC-SCU-25 | 未知 CSR 地址读为 X | READ 非法 addr |

---

## 5. 断言验证（Assertions）

`SCR1_TRGT_SIMULATION` 下 `:466-512`：

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SCR1_SVA_SCU_RESETS_XCHECK` | 481-484 | 输入复位无 X | 通过 |
| `SCR1_SVA_SCU_SYS2SOC_QLFY_CHECK` | 487-490 | sys 复位下降沿后 qlfy 已先下降 | 通过 |
| `SCR1_SVA_SCU_CORE2SOC_QLFY_CHECK` | 492-495 | core 同 sys | 通过 |
| `SCR1_SVA_SCU_CORE2HDU_QLFY_CHECK` | 497-500 | core→hdu | 通过 |
| `SCR1_SVA_SCU_CORE2DM_QLFY_CHECK` | 502-505 | core→dm | 通过 |
| `SCR1_SVA_SCU_HDU2DM_QLFY_CHECK` | 507-510 | hdu→dm | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归

`SCR1_CFG_RV32IMC_MAX` 定义 `SCR1_DBG_EN`，`hello` PASS 说明：

- SCU 上电复位正确生成 `core_rst_n`，使流水线正常启动；
- 6 条 SVA 在复位序列中无违例（qualifier 时序正确）；
- 非 Verilator 下的 `$assertoff/$asserton` 屏蔽亦被覆盖（Verilator 分支不启用）。

### 6.2 波形核对

可核对：

1. 上电后 `scu_control_ff=0`、`scu_mode_ff=0`、`scu_sticky_sts_ff=0`；
2. `sys_rst_n_o`/`core_rst_n_o` 在各复位源释放后依次拉高；
3. `core_rdc_qlfy_o = core2hdu_rdc_qlfy_o = core2dm_rdc_qlfy_o`（FC-SCU-23）。

### 6.3 定向专项（建议补充执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| CSR 读写 | TAPC 写/读 CONTROL/MODE/STATUS | 值正确回读 | 6-9,11 |
| sticky | 触发复位后 CLRBITS | 位置 1 后清 0 | 14-15 |
| 软件 sys 复位 | 写 CONTROL.sys_reset | sys/core 复位级联 | 16-17 |
| 行为屏蔽 | 写 MODE=1 后触发 | HDU/DM 复位被屏蔽 | 20-21 |
| ndm/hart 复位 | DM 触发 | System/Core 联动 | 18-19 |

> 6.3 尚未系统执行。

---

## 7. 回归流程

MAX 配置即含 SCU，回归命令同其它模块（`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`），
测试台 `hello`，复位序列即覆盖 qualifier SVA。波形信号见 2.2 节。

---

## 8. 验证结论

1. SCU 在 MAX 配置下编译并通过 hello 回归，6 条内建 SVA 无违例；
2. 设计规格所述的四路复位表达式、MODE 行为、STICKY 逻辑、qualifier 生成均与源码一致；
3. 复位生成属核心路径，上电序列即验证了基本功能；
4. 建议按 6.3 补充 CSR 读写与 MODE 屏蔽的定向测试。

**遗留建议**：建立 SCU CSR（TAPC）读写与复位行为配置的定向测试，覆盖 FC-SCU-1~25。
