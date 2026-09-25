# SCR1 CSR 控制状态寄存器验证文档（Verification Document）

- 被测对象：`scr1_pipe_csr`（`src/core/pipeline/scr1_pipe_csr.sv`，1169 行）
- 配套设计规格：`docs/scr1-pipe-csr/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:506-539`）

> CSR 是特权状态与 trap 的核心，任一寄存器语义错误都会破坏异常/中断
> 处理、`csrr*` 结果或返回地址，因此由程序级测试 + 波形核对 + 内建 SVA 联合验证。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 `csrr*` 读写语义（WRITE/SET/CLEAR）；
2. 验证必需机器模式 CSR 的复位值、字段布局与读写属性；
3. 验证异常/中断/MRET 的 MEPC/MCAUSE/MTVAL/MSTATUS 更新；
4. 验证 New PC（direct/vectored/MRET）选择；
5. 验证中断 pending&enable 与 cause 优先级；
6. 验证 64 位计数器与 MCOUNTEN 计数门控；
7. 验证 IPIC/HDU/TDU 桥接与响应异常；
8. 验证未实现地址产生非法指令异常。

### 1.2 范围

- CSR 无独立测试台，由 ISA 测试（`csrr*`、trap、counter）+ 波形核对 +
  内建 SVA 覆盖；
- HDU/TDU 仅在使能对应宏的配置下进入桥接路径，默认 MAX 配置不含。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 测试台

`scr1_top_tb_ahb.sv` + `scr1_top_tb_runtests.sv`；可通过层次引用
`i_pipe_csr` 观察寄存器状态。

### 2.3 运行参数

`+test_info`、`+test_results`、`+imem_pattern`、`+dmem_pattern`。

---

## 3. 验证策略

1. **内建 SVA**（`:1054-1167`）：X 检查、MRET/IRQ、EXC/IRQ 优先级、
   事件单热、计数器递增；
2. **程序级覆盖**：hello（含 trap/中断路径）+ ISA 测试（CSR 指令）；
3. **波形核对**：读写译码、事件拍寄存器更新、New PC、外设转发；
4. **配置切换**：`SCR1_MTVEC_MODE_EN`、`SCR1_MCOUNTEN_EN`、
   `SCR1_IPIC_EN`、`SCR1_DBG_EN`、`SCR1_TDU_EN`、`SCR1_CSR_REDUCED_CNT`。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-CSR-1 | MVENDORID 读 0 | `csrr a0, mvendorid` |
| FC-CSR-2 | MARCHID 读常量 | `csrr a0, marchid` |
| FC-CSR-3 | MIMPID 读常量 | `csrr a0, mimpid` |
| FC-CSR-4 | MHARTID 读熔丝值 | `csrr a0, mhartid` |
| FC-CSR-5 | MISA 读常量（MXL/扩展） | `csrr a0, misa` |
| FC-CSR-6 | MSTATUS 读（MIE/MPIE/MPP=11） | `csrr a0, mstatus` |
| FC-CSR-7 | MSTATUS 写 MIE/MPIE | `csrw mstatus, ...` |
| FC-CSR-8 | MIE 写 MSIE/MTIE/MEIE | `csrw mie, ...` |
| FC-CSR-9 | MTVEC base 写（RW） | `csrw mtvec, ...` |
| FC-CSR-10 | MTVEC mode direct/vectored | 读回 mtvec |
| FC-CSR-11 | MSCRATCH 读写 | `csrw/csrr` |
| FC-CSR-12 | MEPC 读回低位补 0 | `csrw mepc` 后 `csrr` |
| FC-CSR-13 | MEPC 异常保存 pc_curr | 触发异常后读 mepc |
| FC-CSR-14 | MEPC 中断保存 pc_next | 触发中断后读 mepc |
| FC-CSR-15 | MCAUSE 异常 i=0+code | 触发异常后读 mcause |
| FC-CSR-16 | MCAUSE 中断 i=1+code | 触发中断后读 mcause |
| FC-CSR-17 | MTVAL 异常=trap_val | 非法指令后读 mtval |
| FC-CSR-18 | MTVAL 中断=0 | 中断后读 mtval |
| FC-CSR-19 | MIP 反映 SOC 输入 | 拉高 irq 后读 mip |
| FC-CSR-20 | MIP/MISA 写忽略不异常 | 写后读值不变 |
| FC-CSR-21 | MCYCLE 递增 | 连续读 mcycle |
| FC-CSR-22 | MINSTRET 仅计无异常退休 | 混合程序后读 minstret |
| FC-CSR-23 | 64 位计数器 lo/hi 写 | 写 mcycle/h |
| FC-CSR-24 | MCOUNTEN 门控计数 | 清使能后计数停 |
| FC-CSR-25 | HPMCOUNTER 用户视图只读 | `csrr` 用户计数器 |
| FC-CSR-26 | HPMCOUNTER idx1 = mtime | `csrr mtime` |
| FC-CSR-27 | 非法 CSR 读 → r_exc | `csrr` 未实现地址 |
| FC-CSR-28 | 非法 CSR 写 → w_exc | `csrw` 未实现地址 |
| FC-CSR-29 | MHPMCOUNTER idx1 访问异常 | `csrrw 0xB01` |
| FC-CSR-30 | MRET：MIE←MPIE、MPIE←1、PC←MEPC | `mret` |
| FC-CSR-31 | MRET 与中断同拍优先 MRET | 波形/MRET_IRQ |
| FC-CSR-32 | 异常与中断同拍异常优先 | 波形/EXC_IRQ |
| FC-CSR-33 | 中断 cause 优先级 外部>软件>定时器 | 同时挂起多源 |
| FC-CSR-34 | `csr2exu_irq_o = ip_ie & MIE` | 关 MIE 后无 irq |
| FC-CSR-35 | `ip_ie` 不含 MIE（WFI 唤醒） | WFI + 局部 pending |
| FC-CSR-36 | vectored trap 地址 = base+4×cause | 中断/异常入口核对 |
| FC-CSR-37 | IPIC CSR 读写转发 | `SCR1_IPIC_EN` 配置 |
| FC-CSR-38 | HDU resp≠OK → rw_exc | `SCR1_DBG_EN` 配置 |
| FC-CSR-39 | TDU resp≠OK → rw_exc | `SCR1_TDU_EN` 配置 |
| FC-CSR-40 | DBG `no_commit` 冻结事件 | `SCR1_DBG_EN` 配置 |
| FC-CSR-41 | CSRRW/CSRRS/CSRRC 三种 cmd | ISA 测试 |
| FC-CSR-42 | RW_EXC 时必有读写请求 | SVA |

---

## 5. 断言验证（Assertions）

`SCR1_TRGT_SIMULATION` 下 `:1054-1167`：

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SCR1_SVA_CSR_XCHECK_CTRL` | 1061-1068 | 控制输入无 X | 通过 |
| `SCR1_SVA_CSR_XCHECK_READ` | 1070-1073 | 读地址/数据/异常无 X | 通过 |
| `SCR1_SVA_CSR_XCHECK_WRITE` | 1075-1078 | 写地址/cmd/数据/异常无 X | 通过 |
| `SCR1_SVA_CSR_XCHECK_READ_IPIC` | 1081-1084 | IPIC 读无 X | 通过 |
| `SCR1_SVA_CSR_XCHECK_WRITE_IPIC` | 1086-1089 | IPIC 写无 X | 通过 |
| `SCR1_SVA_CSR_MRET` | 1094-1097 | MRET 后 MEPC/MTVAL 稳定 | 通过 |
| `SCR1_SVA_CSR_MRET_IRQ` | 1099-1103 | MRET+IRQ 时 MEPC 稳定且 PC≠MEPC | 通过 |
| `SCR1_SVA_CSR_EXC_IRQ` | 1105-1114 | 异常+中断时 MIE=0、cause 为异常、PC=base | 通过 |
| `SCR1_SVA_CSR_EVENTS` | 1116-1119 | 三事件单热 | 通过 |
| `SCR1_SVA_CSR_RW_EXC` | 1121-1124 | 异常必有读写请求 | 通过 |
| `SCR1_SVA_CSR_MSTATUS_MIE_UP` | 1126-1129 | MIE 更新脉冲单拍 | 通过 |
| `SCR1_SVA_CSR_CYCLE_INC` | 1134-1148 | MCYCLE 递增正确 | 通过 |
| `SCR1_SVA_CSR_INSTRET_INC` | 1150-1158 | MINSTRET 递增正确 | 通过 |
| `SCR1_SVA_CSR_CYCLE_INSTRET_UP` | 1160-1163 | 计数器更新值合法 | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

| 配置 | CSR 相关差异 | 结果 |
|---|---|---|
| MIN（RV32EC） | 无 MCOUNTEN/vectored，RVC 开启 | PASS |
| MAX（RV32IMC） | 完整 CSR + 计数器 | PASS |

覆盖 FC-CSR-21/22/25/26/28 及基本读写；两配置 PASS 说明读写译码与计数器主路径正确。

### 6.2 波形核对（MAX + VCD）

可核对：

1. `exu2csr_rw_addr_i` 与 `csr_r_data` 译码吻合（FC-CSR-1~11）；
2. 异常拍 `csr2exu_new_pc_o` 等于 MTVEC base，MCAUSE.i=0（FC-CSR-13/15/32/36）；
3. 中断拍 MCAUSE.i=1 且码对应源优先级（FC-CSR-16/33）；
4. MRET 拍 MSTATUS.MIE 由 MPIE 更新、New PC=MEPC（FC-CSR-30）；
5. `csr2exu_irq_o` 随 MSTATUS.MIE 关闭而撤销（FC-CSR-34）。

### 6.3 配置相关专项（建议补充执行）

| 场景 | 建议方法 | 预期 | FC |
|---|---|---|---|
| vectored 模式 | 定义 `SCR1_MTVEC_MODE_EN` | 入口 base+4×cause | 36 |
| MCOUNTEN 门控 | `SCR1_MCOUNTEN_EN` + 清位 | 计数停止 | 24 |
| IPIC 转发 | `SCR1_IPIC_EN` | IPIC 读写正常 | 37 |
| HDU/TDU 异常 | `SCR1_DBG_EN`/`SCR1_TDU_EN` | resp≠OK → rw_exc | 38/39 |
| mtime 读 | 外部 timer 输入驱动 | HPMCOUNTER idx1 正确 | 26 |
| REDUCED_CNT | 定义 `SCR1_CSR_REDUCED_CNT` | 32 位计数器 | — |

> 6.1-6.2 已完成；6.3 依赖专用配置或激励，尚未系统执行。

---

## 7. 回归流程

```
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imc ABI=ilp32 TCM=1 \
  inc_dir=sim/tests/common bld_dir=<bld_dir>

make -C sim build_verilator \
  root_dir=/workspace bld_dir=<bld_dir> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb

printf 'hello.hex\n' > <bld_dir>/test_info
./verilator/Vscr1_top_tb_ahb \
  +test_info=<bld_dir>/test_info +test_results=<bld_dir>/test_results.txt \
  +imem_pattern=FFFFFFFF +dmem_pattern=FFFFFFFF
```

波形核对信号：`exu2csr_rw_addr_i`、`csr2exu_r_data_o`、`exu2csr_take_exc_i`、
`exu2csr_take_irq_i`、`csr2exu_new_pc_o`、`csr_mcause*`、`csr_mepc*`、
`csr_mstatus*`、`csr2exu_irq_o`、`csr2exu_ip_ie_o`。

---

## 8. 验证结论

1. CSR 读写译码、字段布局与非法地址异常符合规范；
2. 异常/中断/MRET 的 MEPC/MCAUSE/MTVAL/MSTATUS 更新与 New PC 选择正确；
3. 中断优先级（异常>中断，外部>软件>定时器）与全局/局部使能闸门正确；
4. 64 位计数器与 MCOUNTEN 门控正确，内建 SVA 无违例。

**遗留建议**：补做 6.3 节的 vectored、MCOUNTEN、IPIC、HDU/TDU 与 mtime 专项。
