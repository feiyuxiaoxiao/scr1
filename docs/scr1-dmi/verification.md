# SCR1 DMI 调试模块接口 验证文档（Verification Document）

- 被测对象：`scr1_dmi`（`src/core/scr1_dmi.sv`，182 行，仅 `SCR1_DBG_EN`）
- 配套设计规格：`docs/scr1-dmi/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_core_top` 例化（`scr1_core_top.sv:417-437`，例化名 `i_dmi`）

> `SCR1_DBG_EN` 随 `SCR1_CFG_RV32IMC_MAX` 启用；DMI 无内建 SVA，功能需 JTAG 激励。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 DTMCS（ch_id=1）与 DMI access（ch_id=2）chain 选择；
2. 验证 capture/shift/update 数据寄存器行为与 TDO 输出；
3. 验证 DTMCS 常量字段（ABITS=7、VERSION=1）；
4. 验证 op 解码（req/wr）；
5. 验证 addr/data 透传与 DM 读数据缓存。

### 1.2 范围

- JTAG/TAP 激励驱动 + 波形核对；需 `SCR1_DBG_EN` 配置构建。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

层次引用 `i_top.i_core_top.i_dmi.*`：`tap_dr_ff`、`tap_dr_rdata`、`tap_dr_shift`、
`dm_rdata_ff`、`tapc_dmi_access_req`、`tapc_dtmcs_sel`、
`dmi2dm_req_o`/`wr_o`/`addr_o`/`wdata_o`。

---

## 3. 验证策略

1. **chain 选择**：切换 `ch_id` 观察 `tapc_dtmcs_sel`/`tapc_dmi_access_req`；
2. **移位**：capture + shift 序列观察 `tap_dr_ff` 与 TDO；
3. **op 解码**：构造 op=00/01/10/11 观察 req/wr；
4. **读回**：DM 响应后检查 `dm_rdata_ff` 与后续 DMI 读数据。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-DMI-1 | ch_id=1 选 DTMCS | 设置 ch_id |
| FC-DMI-2 | ch_id=2 选 DMI access | 设置 ch_id |
| FC-DMI-3 | DTMCS 读：ABITS=7 | capture |
| FC-DMI-4 | DTMCS 读：VERSION[0]=1，其余 0 | capture |
| FC-DMI-5 | DMI 读：DATA=dm_rdata_ff，ADDR/OP=0 | capture |
| FC-DMI-6 | capture 载入 tap_dr_rdata | 波形 |
| FC-DMI-7 | shift 串入 TDI | 波形 |
| FC-DMI-8 | TDO = tap_dr_ff[0] | 波形 |
| FC-DMI-9 | update+sel+ch2 → access_req | 波形 |
| FC-DMI-10 | op=00 → req=0 | 移位值 |
| FC-DMI-11 | op=01/11 → req=1, wr=0（读） | 移位值 |
| FC-DMI-12 | op=10 → req=1, wr=1（写） | 移位值 |
| FC-DMI-13 | addr/data 透传到 DM | 波形 |
| FC-DMI-14 | 读响应后 dm_rdata_ff 更新 | resp 拉起 |
| FC-DMI-15 | 写响应不更新 dm_rdata_ff | wr 时 |
| FC-DMI-16 | 无访问时 dmi2dm_* 全 0 | 波形 |

---

## 5. 断言验证（Assertions）

本模块**未内建 SVA**。正确性通过 TAP 激励与波形核对（§6）。

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归

`SCR1_CFG_RV32IMC_MAX` 定义 `SCR1_DBG_EN`，`hello` PASS 说明：

- DMI 例化与 TAPC/DM 连线不破坏正常执行；
- 复位后 `tap_dr_ff=0`、`dm_rdata_ff=0`、`dmi2dm_req_o=0`。

### 6.2 静态核对

- chain ID 判据（`:101`/`:151`）、字段位域（`:69-74`）、op 解码（`:160-161`）
  与 `scr1_dm.svh` 参数一致；
- 无 SVA（grep 为空）。

### 6.3 定向专项（建议执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| DTMCS 读 | ch_id=1 capture | ABITS=7、VERSION=1 | 1,3-4 |
| DMI 读 | ch_id=2，op=01 | req=1,wr=0 | 2,11 |
| DMI 写 | ch_id=2，op=10 | req=1,wr=1 | 12 |
| 读回缓存 | 读后 DM resp | dm_rdata_ff 更新 | 14 |
| 空操作 | op=00 | req=0，输出全 0 | 10,16 |

> 6.3 尚未系统执行。

---

## 7. 回归流程

MAX 配置即含 DMI，回归命令同其它模块（`SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX`），
测试台 `hello`。完整功能验证需 JTAG 激励，波形信号见 2.2 节。

---

## 8. 验证结论

1. DMI 在 MAX 配置下编译并通过 hello 回归；
2. 设计规格所述的 chain 选择、移位、op 解码、读数据缓存均与源码一致；
3. 无内建断言，功能正确性需 JTAG 激励确认；
4. 建议按 6.3 建立 TAP 级定向测试。

**遗留建议**：建立 TAPC↔DMI↔DM 串行访问的定向测试，覆盖 FC-DMI-1~16。
