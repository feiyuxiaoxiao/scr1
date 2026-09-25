# SCR1 流水线顶层 验证文档（Verification Document）

- 被测对象：`scr1_pipe_top`（`src/core/pipeline/scr1_pipe_top.sv`，825 行）
- 配套设计规格：`docs/scr1-pipe-top/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0

> `scr1_pipe_top` 是纯结构集成层，验证重点是**互连正确性、配置条件化、
> 时钟/复位域、跨模块控制路径**，而非内部算法。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证各子模块例化与端口连接正确；
2. 验证跨模块数据/控制路径（New PC、中断、断点、调试）；
3. 验证配置宏条件编译分支（IPIC/DBG/TDU/CLKCTRL/MPRF_RST/CSR_REDUCED_CNT/NO_EXE_STAGE）；
4. 验证时钟与复位域选择；
5. 验证顶层控制派生（stop_fetch、sleep/wake、PC 采样、限定逻辑）。

### 1.2 范围

- 顶层无内建 SVA（断言均在各子模块内）；
- 集成正确性由程序级回归 + 波形 + 层次探针联合验证；
- 未启用宏的分支（DBG/TDU/IPIC/CLKCTRL）需专门配置构建验证。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 测试台

`scr1_top_tb_ahb.sv` + `scr1_top_tb_runtests.sv`；顶层例化路径
`i_core.i_pipe_top`（层次引用见 tracelog，`:788-818`）。

### 2.3 运行参数

`+test_info`、`+test_results`、`+imem_pattern`、`+dmem_pattern`。

---

## 3. 验证策略

1. **程序级回归**：hello + ISA 测试遍历跨模块路径；
2. **构建矩阵**：MIN/MAX 及带/不带 IPIC、DBG、TDU、CLKCTRL 的配置编译；
3. **波形核对**：握手、New PC 重定向、trap 入口、DMEM 访问；
4. **层次探针**：直接观测 `i_pipe_*` 内部信号。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-TOP-1 | IFU→IDU 取指有效握手 | 正常执行 |
| FC-TOP-2 | IDU→EXU 命令握手（`exu2idu_rdy`） | 正常执行 |
| FC-TOP-3 | EXU→IFU New PC 重定向（分支/跳转） | `beq`/`jal` |
| FC-TOP-4 | EXU↔MPRF 2 读 1 写互连 | 任意寄存器指令 |
| FC-TOP-5 | EXU↔CSR 读写互连 | `csrr*` |
| FC-TOP-6 | CSR→EXU trap New PC | 异常/中断 |
| FC-TOP-7 | IMEM 端口透传 | 取指 |
| FC-TOP-8 | DMEM 端口透传 | `lw`/`sw` |
| FC-TOP-9 | `stop_fetch` 在 WFI 生效 | WFI 后波形 |
| FC-TOP-10 | `clkctl_sleep_req` 在停机且无 IMEM 事务时拉高 | CLKCTRL 配置 |
| FC-TOP-11 | `clkctl_wake_req` 由 `ip_ie` 唤醒 | 局部中断挂起 |
| FC-TOP-12 | `clkctl_wake_req` 由 `dm2pipe_active_i` 唤醒 | DBG+CLKCTRL |
| FC-TOP-13 | IPIC 模式下 `ipic2csr_irq` 送 CSR | IPIC 配置 |
| FC-TOP-14 | 非 IPIC 模式外部中断直连 CSR | 默认配置 |
| FC-TOP-15 | `soc2pipe_mtimer_val_i` 透传到 CSR | 读 `mtime` |
| FC-TOP-16 | `soc2pipe_fuse_mhartid_i` 透传 | 读 `mhartid` |
| FC-TOP-17 | TDU 断点接口互连 | TDU 配置 |
| FC-TOP-18 | HDU 调试接口互连 | DBG 配置 |
| FC-TOP-19 | 复位域交叉限定 `*_qlfy` 生效 | DBG 波形 |
| FC-TOP-20 | `hwbrk_dsbl = ~dbg_en \| hdu_hwbrk_dsbl` | DBG 波形 |
| FC-TOP-21 | `SCR1_MPRF_RST_EN` 连接复位 | 配置构建 |
| FC-TOP-22 | `SCR1_CSR_REDUCED_CNT` 去掉 instret 输入 | 配置构建 |
| FC-TOP-23 | `SCR1_NO_EXE_STAGE` 去掉 use_rd/use_imm | 配置构建 |
| FC-TOP-24 | tracelog 层次探针可读 | 仿真 |
| FC-TOP-25 | 顶层无 LSU/IALU 例化（结构确认） | 代码检查 |
| FC-TOP-26 | IDU 时钟/复位仅在仿真连接 | 代码检查 |
| FC-TOP-27 | CLKCTRL 下 IPIC 用 `clk_alw_on` | 配置检查 |
| FC-TOP-28 | CLKCTRL 下 HDU 用 `clk_dbgc` | 配置检查 |
| FC-TOP-29 | DBG 下 TDU 用 `dbg_rst_n` | 配置检查 |
| FC-TOP-30 | `pipe2dm_pc_sample_o = curr_pc` | DBG 波形 |

---

## 5. 断言验证（Assertions）

`scr1_pipe_top` **不含内建 SVA**。集成阶段的断言由被例化子模块自带的
`SCR1_TRGT_SIMULATION` 断言承担（IFU/IDU/EXU/LSU/IALU/MPRF/CSR），
顶层仅做结构连接，因此以静态连接检查 + 动态回归为主。

| 检查项 | 方法 | 状态 |
|---|---|---|
| 端口连接完整性 | 编译（未连接端口告警） | 通过 |
| 条件编译分支 | 多配置编译 | 部分通过 |
| 层次探针可达 | 仿真编译通过 | 通过 |

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

| 配置 | 顶层差异 | 结果 |
|---|---|---|
| MIN（RV32EC） | 无 IPIC/DBG/TDU，RVC 开启 | PASS |
| MAX（RV32IMC） | 完整 | PASS |

覆盖 FC-TOP-1~8、13/14 的默认分支；两配置 PASS 证明主数据通路互连正确。

### 6.2 波形核对

可核对：

1. `new_pc_req`/`new_pc` 在分支/跳转拍更新（FC-TOP-3）；
2. `exu2csr_take_exc_i` 拍 `csr2exu_new_pc` 变为 trap 入口（FC-TOP-6）；
3. IMEM/DMEM 端口请求-应答与子模块波形一致（FC-TOP-7/8）；
4. `stop_fetch` 在 WFI 后拉高（CLKCTRL 配置，FC-TOP-9）。

### 6.3 配置矩阵（建议补充执行）

| 配置 | 验证点 |
|---|---|
| `SCR1_IPIC_EN` | FC-TOP-13/27 |
| `SCR1_DBG_EN` | FC-TOP-12/18/19/20/28/29/30 |
| `SCR1_TDU_EN` | FC-TOP-17 |
| `SCR1_CLKCTRL_EN` | FC-TOP-10/11/27/28 |
| `SCR1_MPRF_RST_EN` | FC-TOP-21 |
| `SCR1_CSR_REDUCED_CNT` | FC-TOP-22 |
| `SCR1_NO_EXE_STAGE` | FC-TOP-23 |
| MAX + `SCR1_TRGT_SIMULATION` | FC-TOP-24 |

> 6.1-6.2 已完成；6.3 的多数非默认宏组合尚未系统构建验证。

### 6.4 结构检查

- 全文搜索确认无 `scr1_pipe_ialu`/`scr1_pipe_lsu` 例化（FC-TOP-25），
  二者在 EXU 内；
- IDU 的 `.rst_n`/`.clk` 位于 `SCR1_TRGT_SIMULATION` 条件内（FC-TOP-26）。

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

波形核对信号：`new_pc_req`、`new_pc`、`stop_fetch`、`curr_pc`、
`exu2csr_take_exc`、`csr2exu_new_pc`、`pipe2imem_req_o`、`pipe2dmem_req_o`、
`ipic2csr_irq`、`csr2exu_ip_ie`。

---

## 8. 验证结论

1. 八个子模块（+IPIC/TDU/HDU/tracelog）例化与互连正确，主数据通路两端 PASS；
2. New PC（分支/跳转/trap）、中断、DMEM/IMEM 端口透传路径正确；
3. 默认 MIN/MAX 配置回归通过；
4. 顶层为结构层，无内建 SVA，集成正确性依赖子模块断言与程序级测试。

**遗留建议**：补做 6.3 节的 IPIC/DBG/TDU/CLKCTRL/REDUCED_CNT/NO_EXE_STAGE
多配置构建与波形验证。
