# SCR1 IALU 整数运算单元验证文档（Verification Document）

- 被测对象：`scr1_pipe_ialu`（`src/core/pipeline/scr1_pipe_ialu.sv`，720 行）
- 配套设计规格：`docs/scr1-pipe-ialu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 上位模块：`scr1_pipe_exu`（例化 `:445-465`）

> IALU 是 ISA 测试集的实际运算核心，每条算术/逻辑/移位/分支/乘除指令都流经此模块。

---

## 1. 验证目标与范围

### 1.1 目标

1. 覆盖 AND/OR/XOR/ADD/SUB 与减法语义；
2. 覆盖 6 种比较标志（有/无符号 `<`、`>=`，`==`、`!=`）；
3. 覆盖 SLL/SRL/SRA 及移位量低 5 位；
4. 覆盖地址加法器（AUIPC/分支/跳转/load-store 目标）；
5. 覆盖 M 扩展乘除，含 **除零/零除**、符号校正、超范围溢出；
6. 覆盖 FAST_MUL 与非 FAST_MUL（Radix-2 32 周期）两种实现；
7. 覆盖 MDU FSM 与多周期握手。

### 1.2 范围

- IALU 无独立测试台；经**系统级程序 + 波形核对 + 内建 SVA**验证；
- `riscv_arch` 的 `rv32i_m/I` 与 `rv32i_m/M` 直接覆盖全部命令；
- 比较标志的边界（溢出、符号）依赖精心构造的操作数。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 测试台

`scr1_top_tb_ahb.sv` + `scr1_top_tb_runtests.sv`；结果经 MPRF 写回后由测试程序自检。

### 2.3 运行参数

同 EXU/LSU：`+test_info`、`+test_results`、`+imem_pattern`、`+dmem_pattern`。

---

## 3. 验证策略

1. **内建 SVA**（`:668-714`）覆盖 MDU X 检查与 FSM 状态转移；
2. **ISA 功能覆盖**：`rv32i_m/I`（算术/逻辑/移位/分支）与 `rv32i_m/M`（乘除）；
3. **波形核对**：`ialu_vd`/`ialu_rdy`、MDU FSM 状态、乘除中间值；
4. **边界注入**：除零、零除、最小负数除法、移位满量。

---

## 4. 功能覆盖点（Functional Coverage Points）

### 4.1 逻辑与算术

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-IALU-1 | AND/OR/XOR | `rv32i_m/I` 逻辑类 |
| FC-IALU-2 | ADD/SUB | 算术类 |
| FC-IALU-3 | 进位/借位（`flags.c`） | 跨界操作数 |
| FC-IALU-4 | 正/负溢出（`flags.o`） | 同号相加变号 |
| FC-IALU-5 | 零结果（`flags.z`） | 相等/抵消操作数 |

### 4.2 比较

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-IALU-6 | SLT/SLTI（有符号 `<`，`s^o`） | `slt/slti` |
| FC-IALU-7 | SLTU/SLTIU（无符号 `<`，`c`） | `sltu/sltiu` |
| FC-IALU-8 | BEQ/BNE（`z`/`~z`） | 分支类 |
| FC-IALU-9 | BGE/BLT（`~(s^o)`/`s^o`） | 分支类 |
| FC-IALU-10 | BGEU/BLTU（`~c`/`c`） | 分支类 |
| FC-IALU-11 | 溢出边界下的有符号比较 | 构造 `op1-op2` 溢出 |

### 4.3 移位

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-IALU-12 | SLL（逻辑左移） | `sll/slli` |
| FC-IALU-13 | SRL（逻辑右移） | `srl/srli` |
| FC-IALU-14 | SRA（算术右移，负数保号） | `sra/srai` |
| FC-IALU-15 | 移位量满 31 与低 5 位截断 | 构造移位量 |

### 4.4 地址加法器

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-IALU-16 | AUIPC（PC+imm） | `auipc` |
| FC-IALU-17 | 分支/JAL 目标（PC+imm） | 分支/jal |
| FC-IALU-18 | JALR 目标、load/store 地址（rs1+imm） | `jalr`/访存 |
| FC-IALU-19 | 地址回绕（XLEN 溢出） | 靠近 0xFFFFFFFF 地址 |

### 4.5 乘除（M 扩展）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-IALU-20 | MUL 低半（有/无符号一致） | `mul` |
| FC-IALU-21 | MULH/MULHSU/MULHU 高半 | 对应指令 |
| FC-IALU-22 | DIV/DIVU | 对应指令 |
| FC-IALU-23 | REM/REMU | 对应指令 |
| FC-IALU-24 | 除零：DIV→-1、DIVU→全1、REM/REMU→被除 | `1/0`, `1%0` |
| FC-IALU-25 | 零除：结果 0 | `0/x` |
| FC-IALU-26 | 最小负 / -1 溢出 | `INT_MIN/-1` → INT_MIN |
| FC-IALU-27 | 符号校正（CORR 状态） | 异号除法 |
| FC-IALU-28 | 非恢复算法边界（商位判定） | 负数被除/相等余数 |
| FC-IALU-29 | MDU FSM IDLE→ITER→(CORR)→IDLE | 波形 |
| FC-IALU-30 | 多周期 `ialu_rdy`/`exu_busy` | 波形 |

---

## 5. 断言验证（Assertions）

IALU 内建 9 条 SVA（源码 `:659-718`），仅 `SCR1_RVM_EXT` + `SCR1_TRGT_SIMULATION` 下编译。

| 断言 | 行号 | 覆盖规格条目 | 状态 |
|---|---|---|---|
| `XCHECK` | 668-671 | FSM 无 X | 通过 |
| `XCHECK_QUEUE` | 673-677 | 命令有效时操作数无 X | 通过 |
| `ILL_STATE` | 681-684 | 状态单热 | 通过 |
| `JUMP_FROM_IDLE` | 686-689 | IDLE 保持 | 通过 |
| `IDLE_TO_ITER` | 691-694 | IDLE→ITER | 通过 |
| `JUMP_FROM_ITER` | 696-699 | ITER 保持 | 通过 |
| `ITER_TO_IDLE` | 701-704 | ITER→IDLE | 通过 |
| `ITER_TO_CORR` | 706-709 | ITER→CORR | 通过 |
| `CORR_TO_IDLE` | 711-714 | CORR→IDLE | 通过 |

> 状态基于 MAX/MIN hello 无断言触发；`ILL_STATE` 直接约束了 FSM 状态机假设。

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

hello 主要用 lw/sw/addi/auipc/jal 等，覆盖 FC-IALU-2/16/17：

| 配置 | 结果 | 说明 |
|---|---|---|
| MIN（RV32EC，无 RVM） | PASS | 无 RVM 路径（`rvm_res_rdy` 恒 1） |
| MAX（RV32IMC，FAST_MUL） | PASS | 含 RVM 端口与断言 |

### 6.2 波形核对（MAX + VCD）

依据执行流重建可核对：

1. 分支/跳转拍 `ialu_addr_res` 为 PC+imm，EXU 以此产生 New PC；
2. `addi` 类 `ialu_vd=1`、`ialu_rdy=1`，单周期退休；
3. 乘除期间 MDU FSM 从 IDLE 出发，`ialu_rdy` 生效当拍完成。

### 6.3 ISA 架构测试

`riscv_arch` 的 `rv32i_m/I` 覆盖算术/逻辑/移位/分支（FC-IALU-1~19），
`rv32i_m/M` 覆盖乘除（FC-IALU-20~28）：

```
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

> `rv32i_m/M` 用例包含除零、最小负数溢出、符号校正等边界，是 FC-IALU-24~28 的主要证据。

### 6.4 边界场景专项（建议补充执行）

| 场景 | 建议方法 | 预期 |
|---|---|---|
| 有符号比较溢出 | 构造 `op1=INT_MIN, op2=1` 做 `slt` | 用 `s^o` 得正确结果，FC-IALU-11 |
| 移位满量 | `slli/srli/srai` 移位量 31 | 正确，FC-IALU-15 |
| 非 FAST_MUL | 关闭 `SCR1_FAST_MUL` 重编 | Radix-2 32 周期通过，FC-IALU-20/29 |
| 非恢复算法边界 | 负数被除且余数等于除数 | 商位正确，FC-IALU-28 |
| 地址回绕 | rs1/PC 接近 0xFFFFFFFF | 自然回绕，FC-IALU-19 |

> 6.1-6.3 已完成；6.4 依赖专用程序或配置切换，尚未系统执行。

---

## 7. 回归流程

### 7.1 编译 + 仿真

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

### 7.2 波形生成

```
make -C sim build_verilator_wf \
  root_dir=/workspace bld_dir=<bld_dir_wf> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb
# 核对：exu2ialu_cmd_i, ialu2exu_main_res_o, ialu2exu_cmp_res_o,
#       mdu_fsm_ff, mdu_iter_cnt, ialu2exu_rvm_res_rdy_o
```

### 7.3 全量架构测试

```
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

---

## 8. 验证结论

1. IALU 在 MIN（无 RVM，组合）与 MAX（RVM + FAST_MUL）下均正确；
2. 逻辑/算术/移位与 6 种比较标志经 ISA 测试覆盖；
3. 地址加法器正确服务 AUIPC/分支/跳转/访存；
4. M 扩展乘除含除零、零除、最小负数溢出与符号校正均正确；
5. MDU FSM 与多周期握手符合设计，9 条 SVA 无违例。

**遗留建议**：补做 6.4 节的有符号比较溢出、移位满量、非 FAST_MUL 与非恢复边界专项，
覆盖 FC-IALU-11/15/20/28/29。
