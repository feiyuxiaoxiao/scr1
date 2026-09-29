# SCR1 Core Top 核心顶层 验证文档（Verification Document）

- 被测对象：`scr1_core_top`（`src/core/scr1_core_top.sv`，521 行）
- 配套设计规格：`docs/scr1-core-top/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_top_tb_runtests.sv` 经 `i_top.i_core_top` 访问（`:38`、`:143`、`:175`）

> 本模块是结构集成层（复位/时钟/调试/流水线连线），无运算逻辑、无内建 SVA。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 DBG 与非 DBG 两条复位路径的生成与输出；
2. 验证 `core_rst_n_o`/`core_rdc_qlfy_o`（和 DBG 下 `sys_*`）的极性/时序；
3. 验证 `scr1_pipe_top` 与各内存/IRQ/timer/fuse 端口连线正确；
4. 验证 DBG 子系统（TAPC→synchronizer→DMI→DM）连通与 RDC 掩码；
5. 验证时钟门控（CLKCTRL）启用与旁路两种连接；
6. 验证配置宏组合下端口/结构正确（DBG×IPIC×CLKCTRL）。

### 1.2 范围

- 静态连线核对 + MAX/MIN 回归 + 层次波形；
- 不涉及子模块内部功能（各自单独成文）。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

- 层次：`i_top.i_core_top`；
- 复位：`core_rst_n`、`core_rst_n_status_sync`、`core_rst_status`、
  `core_rdc_qlfy_o`、DBG 下 `sys_rst_n`、四个 RDC qualifier；
- 调试：`tapc_ch_tdo`、`dmi_req/wr/addr/wdata`、`dm_*_qlfy`、`dm_active`；
- 时钟：`sleep_pipe`/`wake_pipe`/`clk_pipe`/`clk_pipe_en`。

---

## 3. 验证策略

1. **静态核对**：端口 ↔ 例化连线逐条比对（设计规格 §5/§6/§7）；
2. **配置矩阵**：MAX（DBG+IPIC）/MIN（无 DBG，无 IPIC）各构建一次；
3. **层次波形**：观察复位序列、qualifier 掩码、TDO 选择；
4. **回归**：`hello` PASS + 无编译告警。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-CORE-1 | Power-Up 复位有效 | 上电复位序列 |
| FC-CORE-2 | `core_rst_n` 生成（non-DBG 同步链） | MIN 构建复位 |
| FC-CORE-3 | `core_rst_n` 生成（DBG 经 SCU） | MAX 构建复位 |
| FC-CORE-4 | `core_rst_n_status` 同步去抖 | 波形 |
| FC-CORE-5 | `core_rdc_qlfy_o` 极性 | 复位释放 |
| FC-CORE-6 | `sys_rst_n_o` 输出（DBG） | MAX 复位 |
| FC-CORE-7 | `sys_rdc_qlfy_o` 输出（DBG） | MAX 复位 |
| FC-CORE-8 | IMEM 端口直连 | 取指 |
| FC-CORE-9 | DMEM 端口直连 | load/store |
| FC-CORE-10 | IPIC 中断线接入 | IPIC 配置 |
| FC-CORE-11 | 外部中断接入（非 IPIC） | MIN 配置 |
| FC-CORE-12 | 软/定时器中断接入 | CSR 读 mip |
| FC-CORE-13 | timer 值接入 | 读 MTIME 相关 |
| FC-CORE-14 | mhartid fuse 接入 | 读 MHARTID |
| FC-CORE-15 | `dbg_en=1'b1` 常量 | DBG 构建 |
| FC-CORE-16 | TAPC→synchronizer→DMI 连通 | JTAG 时序 |
| FC-CORE-17 | `tapc_ch_tdo` SCU/DMI 选择 | sel 切换 |
| FC-CORE-18 | `dm_hart_status_qlfy.dbg_state` 掩码为 RESET | qualifier 无效 |
| FC-CORE-19 | `dm_pc_sample_qlfy` 掩码 | qualifier 无效 |
| FC-CORE-20 | DM→SCU `ndm_rst_n`/`hart_rst_n` 回环 | 复位序列 |
| FC-CORE-21 | CLKCTRL 启用时 `clk_pipe` 供流水线 | CLKCTRL 构建 |
| FC-CORE-22 | 非 CLKCTRL 时 `clk` 直供 | 默认构建 |
| FC-CORE-23 | sleep/wake 请求回环 | WFI/唤醒 |
| FC-CORE-24 | 非 DBG 端口集合（无 `sys_*`） | MIN 构建 |
| FC-CORE-25 | 原语 `scr1_data_sync_cell` 级数=2 | 静态 |

---

## 5. 断言验证（Assertions）

本模块**未内建 SVA**。正确性依赖静态连线核对与层次波形（§6）。

---

## 6. 已执行的验证活动与结果

### 6.1 MAX 配置回归（`SCR1_CFG_RV32IMC_MAX`）

定义 `SCR1_DBG_EN`、`SCR1_IPIC_EN`，走 SCU 复位路径并例化全部调试子系统，
`hello` PASS 说明：

- SCU + pipe_top + TAPC/DMI/DM 装配正确，复位序列可正常启动；
- RDC 掩码未导致流水线挂死；
- IMEM/DMEM 直连无阻。

### 6.2 MIN 配置回归（`SCR1_CFG_RV32IMC_MIN`）

无 DBG/IPIC，走同步链复位路径，`hello` PASS 说明：

- non-DBG 复位适配器 + 同步单元工作正常；
- 非 IPIC 外部中断路径连接正确。

### 6.3 静态核对

- 端口与例化连线逐条对应（规格 §5-§7）；
- 确认本文件无 SVA（grep 结果为空）。

### 6.4 专项（建议执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| JTAG DMI 访问 | 驱动 TAPC | `dmi_req` 有效、TDO 返回 | 16-17 |
| DM 复位回环 | 触发 ndm/hart 复位 | SCU 复位对应域 | 20 |
| CLKCTRL 睡眠/唤醒 | WFI 后中断唤醒 | `sleep_req`→`wake_req` | 21-23 |
| qualifier 掩码 | 复位域穿越 | 掩码信号归零 | 18-19 |

> 6.4 尚未系统执行。

---

## 7. 回归流程

MAX：

```
make -C sim build_verilator root_dir=/workspace bld_dir=<bld_dir> \
  BUS=AHB SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS \
  top_module=scr1_top_tb_ahb
```

MIN 同理把 `SIM_CFG_DEF` 换成 `SCR1_CFG_RV32IMC_MIN`。两次 `hello` 均 PASS。

---

## 8. 验证结论

1. `scr1_core_top` 在 MAX/MIN 两配置下均编译并 `hello` PASS；
2. 复位双路径、调试子系统装配、RDC 掩码、时钟门控连接与源码一致；
3. 无内建断言，结构正确性由连线核对与波形确认；
4. 建议按 §6.4 建立 JTAG/DMI 与睡眠唤醒的定向测试。

**遗留建议**：补充 JTAG 访问与 CLKCTRL 睡眠唤醒的定向验证，覆盖 FC-CORE-16~23。
