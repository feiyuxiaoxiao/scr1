# SCR1 IPIC 验证文档（Verification Document）

- 被测对象：`scr1_ipic`（`src/core/pipeline/scr1_ipic.sv`，605 行，仅 `SCR1_IPIC_EN`）
- 配套设计规格：`docs/scr1-ipic/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:581-595`）

> IPIC 无内建 SVA，验证依赖程序级 CSR 访问序列 + 波形核对。
> 默认 MAX 配置未启用 `SCR1_IPIC_EN`，需专门配置或自建激励。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 IRQ 线同步（可选）、电平/边沿检测、反相；
2. 验证 IPR 置位/清除（等级与边沿两类清除条件）；
3. 验证 CISV/ISVR 服务状态机与 EOI/SOI 语义；
4. 验证 IER/IMR/IINVR/IDX/ICSR/CICSR 读写；
5. 验证固定优先级仲裁与嵌套升级；
6. 验证 `ipic2csr_irq_m_req_o` 产生条件。

### 1.2 范围

- IPIC 无独立测试台；通过 CSR 驱动 + 波形；
- 默认配置不含 IPIC，需定义 `SCR1_IPIC_EN` 构建。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 测试台与参数

`scr1_top_tb_ahb.sv`；`+test_info`、`+test_results`、`+imem_pattern`、`+dmem_pattern`。

### 2.3 观察点

层次引用 `i_top.i_core_top.i_pipe_top.i_pipe_ipic.*`（`ipic_ipr_ff`、`ipic_isvr_ff`、
`ipic_cisv_ff`、`irq_req_idx`、`ipic2csr_irq_m_req_o`）。

---

## 3. 验证策略

1. **CSR 驱动**：软件通过 CSR 地址访问 IPIC 寄存器；
2. **激励注入**：驱动 `soc2ipic_irq_lines_i` 观察 pending/service；
3. **波形核对**：同步延迟、边沿锁存、优先级、EOI 升级；
4. **配置切换**：`SCR1_IPIC_SYNC_EN`、`SCR1_IPIC_EN`。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-IPIC-1 | `SCR1_IPIC_SYNC_EN` 下 2 拍同步延迟 | 波形 |
| FC-IPIC-2 | 未使能同步时线直通 | 波形 |
| FC-IPIC-3 | 电平模式 pending 跟随线 | 拉高/拉低线 |
| FC-IPIC-4 | 边沿模式上升沿置 pending | 脉冲线 |
| FC-IPIC-5 | IINVR 反相后检测 | 写 ICSR.INV |
| FC-IPIC-6 | IER=0 时不请求 | 清 enable |
| FC-IPIC-7 | IPR 写 1 清除 | 写 IPR |
| FC-IPIC-8 | 电平模式线仍高不可清 | 写 IPR 后读 |
| FC-IPIC-9 | CICSR 写 IP 清当前 pending | 写 CICSR.IP |
| FC-IPIC-10 | ICSR 写 IP 清索引 pending | 写 ICSR.IP |
| FC-IPIC-11 | SOI 启动中断（更新 CISV/ISVR） | 写 SOI |
| FC-IPIC-12 | 无待服务时 SOI 不启动 | 写 SOI |
| FC-IPIC-13 | CISV 记录服务索引 | 读 CISV |
| FC-IPIC-14 | 无服务时 CISV=void(0x10) | 读 CISV |
| FC-IPIC-15 | EOI 结束服务 | 写 EOI |
| FC-IPIC-16 | ISVR 置位/清除 | 读 ISVR |
| FC-IPIC-17 | 固定优先级：索引小者优先 | 多线同时拉高 |
| FC-IPIC-18 | 服务中低优先级不产生请求 | 波形 |
| FC-IPIC-19 | 高优先级抢占请求（`hi_prior_pnd`） | 波形 |
| FC-IPIC-20 | EOI 后立即升级到下一 pending | EOI+波形 |
| FC-IPIC-21 | CICSR 写 IE | 写 CICSR.IE |
| FC-IPIC-22 | ICSR 写 IE | 写 ICSR.IE |
| FC-IPIC-23 | ICSR 写 IM 切边沿/电平 | 写 ICSR.IM |
| FC-IPIC-24 | ICSR 写 INV | 写 ICSR.INV |
| FC-IPIC-25 | IDX 读写 | 写/读 IDX |
| FC-IPIC-26 | ICSR 读回 line/PRV/IS | 读 ICSR |
| FC-IPIC-27 | 16 线同时挂起选 index 0 | 全 1 激励 |
| FC-IPIC-28 | MEIP 路径集成到 CSR | 中断流程 |
| FC-IPIC-29 | IPR 更新仅在 next≠ff 时写 | 波形 |

---

## 5. 断言验证（Assertions）

`scr1_ipic` **不含内建 SVA**，也无 `SCR1_TRGT_SIMULATION` 断言块。验证以
激励/波形/CSR 读回为准；上表各项通过 `ipic2csr_rdata_o` 读回与内部层次信号核对。

---

## 6. 已执行的验证活动与结果

### 6.1 默认配置

默认 MIN/MAX 配置**未定义** `SCR1_IPIC_EN`，IPIC 不例化，`soc2pipe_irq_ext_i`
直连 CSR。因此 hello 回归不覆盖 IPIC 内部逻辑。

### 6.2 配置构建（建议补充执行）

| 配置 | 验证点 |
|---|---|
| `SCR1_IPIC_EN` | FC-IPIC-1~29 |
| `SCR1_IPIC_EN + SCR1_IPIC_SYNC_EN` | FC-IPIC-1 |

### 6.3 专项激励建议

| 场景 | 方法 | 预期 |
|---|---|---|
| 电平 pending | 持续拉高 line0，读 IPR | bit0=1；拉低后写 IPR 清除 |
| 边沿 pending | line0 单拍脉冲，读 IPR | bit0 锁存；写 IPR 清除 |
| 优先级 | 同时拉高 line1/line3 | `irq_req_idx=1` |
| 嵌套 | 服务 line5 时拉低 line2 | `hi_prior_pnd=1`，产生新请求 |
| EOI 升级 | 服务 line1，line2 pending，写 EOI | CISV→2，ISVR 更新 |
| 反相 | 写 ICSR.INV[0]，拉低 line0 | pending 置位 |

> 上述专项未在本次回归中系统执行。

---

## 7. 回归流程

```
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imc ABI=ilp32 TCM=1 \
  inc_dir=sim/tests/common bld_dir=<bld_dir>
```

IPIC 需在配置中定义 `SCR1_IPIC_EN` 后构建 Verilator 仿真（参数同其它模块的
`build_verilator` 流程）。波形核对信号：`soc2ipic_irq_lines_i`、`irq_lvl`、
`irq_edge_detected`、`ipic_ipr_ff`、`ipic_isvr_ff`、`ipic_cisv_ff`、
`irq_req_idx`、`irq_serv_idx`、`ipic2csr_irq_m_req_o`、`irq_start_vd`。

---

## 8. 验证结论

1. IPIC 为条件编译模块，默认配置不激活，需专门配置验证；
2. 设计规格中的优先级、EOI/SOI、IPR 清除条件均与源码一致；
3. 因无内建 SVA，建议按 6.3 建立定向激励以覆盖全部 FC-IPIC 点。

**遗留建议**：建立 `SCR1_IPIC_EN` 配置的定向测试，覆盖 FC-IPIC-1~29。
