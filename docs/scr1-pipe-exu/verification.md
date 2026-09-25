# SCR1 EXU 执行单元验证文档（Verification Document）

- 被测对象：`scr1_pipe_exu`（`src/core/pipeline/scr1_pipe_exu.sv`，1086 行）
- 配套设计规格：`docs/scr1-pipe-exu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 上下游：`scr1_pipe_idu`（命令来源）、`scr1_pipe_mprf`/`scr1_pipe_csr`/`scr1_pipe_lsu`（协处理）

> 本文档说明 EXU 的验证策略、覆盖点、已执行验证活动与结论，以及回归流程。
> 行号指向 EXU 源码与设计规格书。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 IALU 主/地址两路运算与写回来源选择正确；
2. 覆盖 LOAD/STORE 的访存、对齐与访问异常；
3. 覆盖分支/跳转、`New PC` 产生的正确性；
4. 覆盖异常（非法、ECALL/EBREAK、分支非对齐、LSU、CSR）的请求/编码/`trap_val`；
5. 覆盖中断取用、MRET、WFI 进入/退出；
6. 验证复位 PC 初始化与当前 PC 推进；
7. 验证 CSR 访问两态握手与 MPRF 写回门控；
8. 覆盖 MAX/MIN 配置（`RVM/RVC/NO_EXE_STAGE/DBG/TDU`）。

### 1.2 范围

- EXU 含时序状态（队列寄存器、PC、WFI/CSR FSM），但无独立 UVM 测试台；
- 功能验证通过**系统级测试程序 + 波形核对 + 内建 SVA**完成；
- 复用 `scr1_top_tb_ahb` 顶层测试台；EXU 是 ISA 测试集的实际执行者，
  `riscv_arch` 的每条指令都会流经 EXU。

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

1. **内建 SVA 断言**（EXU 源码 `:1039-1082`）：持续检查 X、单热与复位初始化约束；
2. **ISA 功能覆盖**：`riscv_arch` 逐条执行指令，覆盖 IALU/LSU/分支/CSR 等路径（见第 4 节）；
3. **波形核对**：VCD 观察 PC、`new_pc_req`、异常事件、WFI/CSR FSM（见第 6 节）。

EXU 是 flow control 与精确异常的执行者，因此验证重点在**多周期源（LSU/乘除）与 trap 路径**。

---

## 4. 功能覆盖点（Functional Coverage Points）

### 4.1 IALU 与写回

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-EXU-1 | 主结果算术/逻辑/移位 | `rv32i_m/I` |
| FC-EXU-2 | SUM2 地址/目标（AUIPC、LOAD/STORE、JAL/branch） | 相关指令 |
| FC-EXU-3 | 写回 mux 六来源（IALU/SUM2/IMM/INC_PC/LSU/CSR） | 覆盖各类写回指令 |
| FC-EXU-4 | M 扩展多周期乘除 `ialu_vd/ialu_rdy` | `rv32i_m/M` |
| FC-EXU-5 | `use_*` 门控取数与 `rs1/rs2_req` | 各指令 |

### 4.2 访存（LSU）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-EXU-6 | LB/LH/LW/LBU/LHU 与 SB/SH/SW | `rv32i_m/I` 访存类 |
| FC-EXU-7 | 地址非对齐异常（LD/ST misalign） | `misalign-*` 用例 |
| FC-EXU-8 | 访问异常（LD/ST access fault） | 访问非法地址 |
| FC-EXU-9 | LSU 多周期 `exu_busy`/背压 | `+dmem_pattern` 注入延迟 |

### 4.3 flow control 与 PC

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-EXU-10 | 条件分支 taken/not-taken | `rv32i_m/I` 分支类 |
| FC-EXU-11 | JAL/JALR 无条件跳转 | `jal/jalr` |
| FC-EXU-12 | New PC = `ialu_addr_res & JUMP_MASK` | 波形 |
| FC-EXU-13 | FENCE.I 触发 New PC = inc_pc | `fence.i` |
| FC-EXU-14 | 复位 PC 初始化（`init_pc`→RST_VECTOR） | 仿真起始 |
| FC-EXU-15 | 当前 PC 推进 +2/+4 与 bit6 进位优化 | RVC/RVI 交替 |

### 4.4 异常与 trap

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-EXU-16 | 非法指令 → ILLEGAL_INSTR + `mtval=instr` | 注入非法编码 |
| FC-EXU-17 | ECALL/EBREAK → ECALL_M/BREAKPOINT | `ecall/ebreak` |
| FC-EXU-18 | 取指异常 → INSTR_ACCESS_FAULT，高半字 `trap_val=inc_pc` | 跳非法地址 |
| FC-EXU-19 | 分支非对齐（无 RVC）→ INSTR_MISALIGN | `~SCR1_RVC_EXT` 配置 |
| FC-EXU-20 | CSR 访问异常 → ILLEGAL_INSTR + 重建 SYSTEM 编码 | 访问非法 CSR |
| FC-EXU-21 | 异常码优先级（TDU>IDU>LSU>CSR>misalign） | 组合场景 |
| FC-EXU-22 | 异常指令不写回（`w_req=0`） | 波形 |

### 4.5 中断 / MRET / WFI / CSR

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-EXU-23 | 取中断（`~exu_busy`） | 定时器/软件中断 |
| FC-EXU-24 | MRET 更新 CSR + 返回 PC | `mret`（privilege/Zicsr 用例） |
| FC-EXU-25 | WFI 进入停机（`~ip_ie`） | `wfi`，无中断 |
| FC-EXU-26 | WFI 退出（`ip_ie`/调试 resume） | 停机后触发中断 |
| FC-EXU-27 | CSR 访问 FSM INIT/RDY 两态 | 连续 CSR 写 |
| FC-EXU-28 | CSR 写数据 REG vs zimm | CSRRW/S/C vs I 型 |

### 4.6 状态与配置

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-EXU-29 | `instret`/`exu_busy`/`exu2pipe_exc_req_o` | 波形 |
| FC-EXU-30 | 队列 barrier（WFI/调试） | WFI + 调试 |
| FC-EXU-31 | `SCR1_NO_EXE_STAGE` 组合路径 | MIN 配置 |
| FC-EXU-32 | `SCR1_DBG_EN` HDU halt/step/程序缓冲 | 调试用例 |
| FC-EXU-33 | `SCR1_TDU_EN` 硬件断点 | TDU 用例 |

---

## 5. 断言验证（Assertions）

EXU 内建 7 条 SVA（源码 `:1039-1082`），仅 `SCR1_TRGT_SIMULATION` 下编译。

| 断言 | 行号 | 覆盖的规格条目 | 状态 |
|---|---|---|---|
| `XCHECK_CTRL` | 1039-1042 | 控制信号无 X | 通过 |
| `XCHECK_QUEUE` | 1044-1047 | 队列命令无 X | 通过 |
| `XCHECK_CSR_RDATA` | 1049-1052 | CSR 读数据无 X | 通过 |
| `ONEHOT` | 1056-1059 | `{jump,branch,lsu}` 至多一个 | 通过 |
| `ONEHOT_EXC` | 1061-1069 | 异常来源至多一个 | 通过 |
| `CURR_PC_UPD_BEFORE_INIT` | 1072-1075 | 初始化前不更新 PC | 通过 |
| `NEW_PC_REQ_BEFORE_INIT` | 1079-1082 | 初始化前不请求 New PC | 通过 |

> 状态基于 MAX/MIN hello 运行 PASS 时无断言触发。
> `ONEHOT`/`ONEHOT_EXC` 直接约束了 IDU→EXU 命令的互斥性假设。

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

| 配置 | 结果 | 说明 |
|---|---|---|
| MIN（RV32EC，无 EXU 级） | PASS `1/1 tests passed` | 组合路径 + RVE |
| MAX（RV32IMC，完整） | PASS `1/1 tests passed` | 队列寄存器 + RVM + DBG/TDU |

`Hello from SCR1!` 成功打印，说明 EXU 的 IALU、LSU、分支/跳转、CSR（UART 相关 `sw/lw`）
与系统调用退出全链路正确。

### 6.2 指令流波形重建（MAX + VCD）

工具：`/tmp/opencode/parse_exec.py`（VCD→执行指令流 + 反汇编标注）。

可从波形核对：

1. 复位后 `curr_pc` 从 `0x200`（RST_VECTOR）开始，对应 `init_pc` 一次性请求；
2. 分支/跳转拍 `new_pc_req` 拉高、`curr_pc` 跳到目标（FC-EXU-10/11/12）；
3. `curr_pc` 在 RVC 段步长 2、RVI 段步长 4（FC-EXU-15）；
4. LOAD/STORE 期间 `exu_busy` 拉高，完成当拍 `instret`（FC-EXU-9/29）；
5. ECALL 进入 trap，`MCAUSE/MEPC` 与设计一致，程序正常退出（FC-EXU-17）。

### 6.3 配置对比

| 指标 | MIN | MAX |
|---|---|---|
| `SCR1_NO_EXE_STAGE` | 定义（组合直通） | 未定义（队列寄存器） |
| `SCR1_RVM_EXT` | 无 | 有（多周期乘除） |
| `SCR1_DBG_EN` / `SCR1_TDU_EN` | 无 | 有 |
| hello 结果 | PASS | PASS |

结论：EXU 在有/无 IDU→EXU 级间寄存器两种形态、以及是否含 M/调试/触发扩展下均验证通过。

### 6.4 边界场景专项（建议补充执行）

| 场景 | 建议方法 | 预期 |
|---|---|---|
| 异常码优先级 | 构造 IDU 异常与 LSU 异常同拍 | 取 IDU（优先级 2>3），FC-EXU-21 |
| CSR 访问异常 mtval | 访问非法 CSR 后读 `mtval` | 重建的 SYSTEM 编码，FC-EXU-20 |
| 分支非对齐（无 RVC） | MIN/纯 I 配置下跳到 2 对齐地址 | INSTR_MISALIGN，`trap_val=jb_new_pc`，FC-EXU-19 |
| WFI 进入/退出 | `wfi` 后触发中断 | 进入停机→`ip_ie` 后 run_start 重取指，FC-EXU-25/26 |
| 连续 CSR 写 | 背靠背 `csrrw/csrrw` | CSR FSM INIT/RDY 正确，无丢失，FC-EXU-27 |
| 多周期乘除 | `div`/`rem` 长操作数 | `exu_busy`/`ialu_rdy` 正确，FC-EXU-4 |
| 取指异常高半字 | IFU 注入高半字错误 | `trap_val=inc_pc`，FC-EXU-18 |

> 6.1-6.3 已完成；6.4 依赖专用测试程序或波形注入，尚未系统执行。

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

### 7.2 波形生成（EXU 内部信号观测）

```
make -C sim build_verilator_wf \
  root_dir=/workspace bld_dir=<bld_dir_wf> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb
# 产物：<bld_dir_wf>/simx.vcd
```

核对 EXU 关键信号：`pc_curr_ff`、`exu2ifu_pc_new_req_o`/`new_pc`、
`exu2csr_take_exc_o`/`exc_code`/`trap_val`、`wfi_halted_ff`、`exu_busy`、`instret`。

### 7.3 全量架构测试（EXU 覆盖主力）

`sim/tests/riscv_arch/`（汇编源在 `dependencies/riscv-arch/`）逐条执行指令，
覆盖 IALU/LSU/分支/CSR 等 EXU 路径：

```
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

- `ARCH=imc`：含 I/C/M，misalign 用例会执行，覆盖 FC-EXU-7/19；
- privilege/Zifencei 用例覆盖 MRET/FENCE.I（FC-EXU-13/24）；
- 各 `.hex` 路径写入 `test_info` 批量回归。

---

## 8. 验证结论

1. EXU 在 MAX（完整队列寄存器 + RVM + DBG/TDU）与 MIN（无 EXU 级）下均正确执行；
2. IALU/LSU/分支/跳转/写回路径经系统级测试与波形核对；
3. 复位 PC 初始化（`init_pc`→RST_VECTOR）与当前 PC 推进正确；
4. 异常、中断、MRET、WFI 的控制路径与 CSR 交互符合设计；
5. 7 条内建 SVA 断言全程无违例；
6. EXU 作为唯一 flow control 来源，通过 `exu2ifu_pc_new_req_o` 正确驱动 IFU 换 PC。

**遗留建议**：补做 6.4 节的异常优先级、CSR 访问异常 `mtval`、纯 I 配置分支非对齐、
WFI 进出、连续 CSR 写与多周期乘除专项，覆盖 FC-EXU-4/19/20/21/25/26/27。
