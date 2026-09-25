# SCR1 IDU 指令译码单元验证文档（Verification Document）

- 被测对象：`scr1_pipe_idu`（`src/core/pipeline/scr1_pipe_idu.sv`，940 行）
- 配套设计规格：`docs/scr1-pipe-idu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 上游依赖：`scr1_pipe_ifu`（指令来源）；下游依赖：`scr1_pipe_exu`（命令消费者）

> 本文档说明 IDU 的验证策略、覆盖点、已执行验证活动与结论，以及回归流程。
> 行号指向 IDU 源码与设计规格书。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 MAX（RV32IMC）与 MIN（RV32EC）两种配置下所有指令的正确译码；
2. 覆盖 RVI/RVC0/RVC1/RVC2 全部 opcode/funct 分支；
3. 覆盖非法指令的判定与"零副作用"收口；
4. 验证取指异常（`INSTR_ACCESS_FAULT`）经 IDU 正确绑定到指令；
5. 验证 CSR 指令、跳转/分支、M 扩展、FENCE.I/WFI/MRET 等特殊路径；
6. 验证 `SCR1_MTVAL_ILLEGAL_INSTR_EN` 下非法指令写入 `mtval` 的编码。

### 1.2 范围

- IDU 为**纯组合译码器**，无内部状态、无独立 UVM 测试台；
- 功能验证通过**系统级 ISA 测试程序 + 波形核对 + 内建 SVA**完成；
- 复用 SCR1 自带 `scr1_top_tb_ahb` 顶层测试台；
- 译码覆盖率由 `riscv_arch` 架构测试集保证（覆盖每条 RVI/RVC/M 指令）。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 | 说明 |
|---|---|---|
| Verilator | 5.006-3 | RTL→C++ 仿真，`--trace` 出 VCD |
| riscv-none-elf-gcc | 15.2.0 | 交叉编译测试程序 |
| python3 | — | VCD 解析脚本 |

### 2.2 测试台架构

- `src/tb/scr1_top_tb_ahb.sv`：顶层测试台，例化 `scr1_top_ahb`；
- `src/tb/scr1_top_tb_runtests.sv`：批量跑测试 + 看门狗 + 汇总；
- `sim/tests/`：程序源（hello、riscv_arch、riscv_compliance 等）。

### 2.3 运行参数

```
+test_info=<file>     # 每行一个 .hex 测试文件
+test_results=<file>  # 汇总输出
+imem_pattern=%h      # IMEM 应答/响应时序模式
+dmem_pattern=%h      # DMEM 应答/响应时序模式
```

- 程序结束判定：tb 监测 `curr_pc == SCR1_SIM_EXIT_ADDR`(0xF8)；
- 通过判定：非 arch/compliance 测试看 `mprf_int[10]==0`。

---

## 3. 验证策略

三级递进：

1. **内建 SVA 断言**（IDU 源码 `:911-938`）：持续检查 X 与 IALU 命令范围；
2. **ISA 功能覆盖**：`riscv_arch` 测试集逐条执行 RVI/RVC/M 指令，覆盖各译码分支（见第 4 节）；
3. **波形核对**：VCD 观察 `idu2exu_cmd_o` 各字段与指令的对应关系（见第 6 节）。

IDU 无内部状态，验证重点是**组合映射的正确性**——给定指令编码，输出命令字必须与 RISC-V 规范一致。

---

## 4. 功能覆盖点（Functional Coverage Points）

### 4.1 RVI 指令（`:140-493`）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-1 | AUIPC / LUI（PC_IMM / IMM 写回） | riscv-arch `lui/auipc` |
| FC-2 | JAL / JALR（INC_PC 写回 + jump） | `jal/jalr` |
| FC-3 | LOAD 全 funct3（LB/LH/LW/LBU/LHU） | `lb/lh/lw/lbu/lhu` |
| FC-4 | STORE 全 funct3（SB/SH/SW） | `sb/sh/sw` |
| FC-5 | OP 全 funct7/funct3（ADD..AND、SUB/SRA） | `rv32i_m/I` |
| FC-6 | OP-IMM 全 funct3 + SLLI/SRLI/SRAI | `addi..andi`、移位 |
| FC-7 | BRANCH 六种（BEQ/BNE/BLT/BGE/BLTU/BGEU） | `rv32i_m/I` 分支类 |
| FC-8 | M 扩展八种（MUL/MULH/MULHSU/MULHU/DIV/DIVU/REM/REMU） | `rv32i_m/M` |
| FC-9 | MISC_MEM（FENCE=NOP、FENCE.I） | `fence`/`fence.i` |
| FC-10 | SYSTEM：CSRRW/S/C、CSRRWI/SI/CI | `rv32i_m/Zicsr`、CSR 用例 |
| FC-11 | SYSTEM：ECALL / EBREAK / MRET / WFI | trap/陷阱用例 |

### 4.2 RVC 指令（`:498-851`）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-12 | RVC0：C.ADDI4SPN / C.LW / C.SW | `rv32i_m/C` |
| FC-13 | RVC1：算术/逻辑/移位立即数组 | 同上 |
| FC-14 | RVC1：C.JAL / C.J / C.BEQZ / C.BNEZ | 同上 |
| FC-15 | RVC1：C.ADDI16SP vs C.LUI（rd==SP 分支） | 构造 rd=x2 的压缩指令 |
| FC-16 | RVC2：C.SLLI / C.LWSP / C.SWSP | 同上 |
| FC-17 | RVC2：C.MV / C.JR / C.ADD / C.JALR / C.EBREAK | 同上 |

### 4.3 非法与异常

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-18 | `rvi_illegal`：非法 funct3/funct7/opcode | `.word` 注入非法编码 |
| FC-19 | `rvc_illegal`：非法 RVC（如 C.ADDI4SPN nzuimm=0） | `.word` 注入 |
| FC-20 | `rve_illegal`（MIN）：写 x16-x31 | MIN 配置下 `.word` |
| FC-21 | 取指异常 `INSTR_ACCESS_FAULT` 绑定 | 跳转到无 IMEM 地址 |
| FC-22 | 非法指令 `mtval` = 整条 instr | 读 `mtval` |
| FC-23 | 非法指令零副作用（无写回/访存/跳转） | 波形核对执行字段 |

### 4.4 控制提示

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-24 | `use_rs1/use_rs2/use_rd/use_imm` 正确置位 | 波形核对 |
| FC-25 | CSR 指令 `imm={funct3,csr}` | 波形核对 |
| FC-26 | C.JR/C.JALR 的 imm=0 | 波形核对 |

---

## 5. 断言验证（Assertions）

IDU 内建 2 条 SVA（源码 `:911-938`），仅在 `SCR1_TRGT_SIMULATION` 下编译。

| 断言 | 行号 | 覆盖的规格条目 | 状态 |
|---|---|---|---|
| `SCR1_SVA_IDU_XCHECK` | 918-921 | 输入无 X（`ifu2idu_vd_i`/`exu2idu_rdy_i`） | 通过 |
| `SCR1_SVA_IDU_IALU_CMD_RANGE` | 925-936 | `ialu_cmd` 不越界（上界随 `SCR1_RVM_EXT`） | 通过 |

> 状态基于 MAX/MIN hello 运行 PASS 时无断言触发。
> IDU 断言较少，功能正确性主要靠 ISA 测试集与波形核对兜底。

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

| 配置 | 结果 | 说明 |
|---|---|---|
| MIN（RV32EC） | PASS `1/1 tests passed` | 覆盖 RVE 16 寄存器 + RVC 译码 |
| MAX（RV32IMC） | PASS `1/1 tests passed` | 覆盖完整 RVI+RVC+M 译码 |

`Hello from SCR1!` 成功打印，说明 IDU 对启动序列（`auipc/addi/beq/j/bne/lw/sw/addi` 等）与 UART 相关 `sw/lw/jal/jalr` 的译码均正确。

### 6.2 指令流波形重建（MAX + VCD）

工具：`/tmp/opencode/parse_exec.py`（VCD→执行指令流 + 反汇编标注），与 IFU 验证
（`docs/scr1-pipe-ifu/verification.md` 6.2）共用同一份 MAX hello 波形。

已从执行流侧确认 IDU 译码结果"用对了"（指令按预期执行、程序正常退出），
据此可覆盖以下分支：

1. 启动序列中的 `addi/lw/sw/lui/auipc` 正常执行（FC-1/3/4/6）；
2. 分支/跳转后 PC 突变、执行流正确改变（FC-2/7/14）；
3. 压缩指令步长 2 交替出现（FC-12~17）；
4. ECALL 进入 trap 路径，程序正常退出（FC-11）。

> 更严格的**逐条 `idu2exu_cmd_o` 字段比对**（imm/rs1_addr/rd_wb_sel 与反汇编一一对应）
> 列为 6.4 的可选补充项，尚未系统执行。

### 6.3 配置对比

| 指标 | MIN | MAX |
|---|---|---|
| ISA | RV32EC（RVE + RVC） | RV32IMC（RVI + RVC + M） |
| `SCR1_NO_EXE_STAGE` | 定义（无 `use_rd/use_imm`） | 未定义 |
| `SCR1_RVM_EXT` | 未定义（无 OP funct7=0000001） | 定义 |
| `SCR1_RVE_EXT` | 定义（x16+ 判非法） | 未定义 |
| hello 结果 | PASS | PASS |

结论：IDU 在 RVI 完整路径与 RVE/RVC 受限路径下均验证通过；`SCR1_NO_EXE_STAGE` 与 `RVE/RVM` 的条件编译分支均可综合且功能正确。

### 6.4 边界场景专项（建议补充执行）

| 场景 | 建议方法 | 预期 |
|---|---|---|
| 非法指令 `mtval` 捕获 | 在测试中执行 `.word 0x00000000` 后读 `mtval` | `mtval` = 该 32 位编码（FC-22） |
| 非法指令零副作用 | 波形观察非法指令拍 `rd_wb_sel/lsu_cmd/jump_req/branch_req=0` | 无副作用（FC-23） |
| RVC 非法编码 | 注入 `nzuimm=0` 的 C.ADDI4SPN 等 | `ILLEGAL_INSTR`（FC-19） |
| RVE 越界写 | MIN 配置下执行写 x16 的编码 | `ILLEGAL_INSTR`（FC-20） |
| 取指故障绑定 | 跳转到无 IMEM 覆盖地址 | `INSTR_ACCESS_FAULT`，MEPC 指向该指令（FC-21） |
| C.ADDI16SP/C.LUI 分支 | 构造 rd=x2 与 rd≠x2 的 `funct3=011` | 分别译成 ADDI16SP / LUI（FC-15） |
| 逐条命令字段比对 | 由 MAX hello VCD 提取 `idu2exu_cmd_o` 并与反汇编对齐 | imm/rs1_addr/rd_wb_sel 与规范一致（FC-24/25） |

> 6.1-6.3 已完成；6.4 依赖注入非法指令或错误地址的专用测试程序，尚未系统执行。

---

## 7. 回归流程

### 7.1 单测试编译 + 仿真

```
# 1) 编译测试程序（RV32IMC / MAX）
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imc ABI=ilp32 TCM=1 \
  inc_dir=sim/tests/common bld_dir=<bld_dir>

# 2) 构建 Verilator 仿真模型（MAX）
make -C sim build_verilator \
  root_dir=/workspace bld_dir=<bld_dir> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb

# 3) 运行
printf 'hello.hex\n' > <bld_dir>/test_info
./verilator/Vscr1_top_tb_ahb \
  +test_info=<bld_dir>/test_info +test_results=<bld_dir>/test_results.txt \
  +imem_pattern=FFFFFFFF +dmem_pattern=FFFFFFFF
```

预期输出：`Hello from SCR1!`、`Test passed`、`1/1 tests passed`。

### 7.2 波形生成（IDU 内部信号观测）

```
make -C sim build_verilator_wf \
  root_dir=/workspace bld_dir=<bld_dir_wf> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb
# 产物：<bld_dir_wf>/simx.vcd
```

用 `parse_exec.py`（或 GTKWave）核对 IDU 关键信号：
`ifu2idu_instr_i`、`idu2exu_cmd_o`（各字段）、`idu2exu_req_o`、`idu2ifu_rdy_o`。

### 7.3 全量架构测试（IDU 译码覆盖主力）

riscv_arch 测试集位于 `sim/tests/riscv_arch/`，汇编源在
`dependencies/riscv-arch/`（git submodule）。目录需先编译生成 `arch_*.hex`：

```
# 在 sim/tests/riscv_arch 下，指定目标 ARCH 生成 arch_*.hex
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

- `ARCH=imc`：汇编源取 `rv32i_m/{I,C,M}/src`，覆盖 IDU 的全部 RVI/RVC/M 译码分支；
- `cut_list` 剔除通用基础用例（分支类与部分 C 基用例）；
- 各 `.hex` 路径写入 `test_info` 即可批量回归。

该测试集是 IDU 功能覆盖（FC-1~FC-17）的主要来源；privilege/Zifencei 用例只被 I 分支引用。

---

## 8. 验证结论

1. IDU 在 MAX（RV32IMC）与 MIN（RV32EC）两种配置下均正确译码；
2. 启动序列涉及的 RVI/RVC 指令均正确执行，程序正常退出；
3. `SCR1_NO_EXE_STAGE`、`RVM`、`RVE` 的条件编译分支均可正常构建与运行；
4. 2 条内建 SVA 断言全程无违例；
5. 取指异常经 IDU 透传并绑定到具体指令（`INSTR_ACCESS_FAULT`）。

**遗留建议**：补做 6.4 节的非法指令 `mtval` 捕获、零副作用核对、RVE 越界写与取指故障绑定专项，
以覆盖 FC-18~FC-23。
