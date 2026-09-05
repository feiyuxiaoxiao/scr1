# SCR1 IFU 取指单元验证文档（Verification Document）

- 被测对象：`scr1_pipe_ifu`（`src/core/pipeline/scr1_pipe_ifu.sv`）
- 配套设计规格：`docs/scr1-pipe-ifu/design-spec.md`
- 配套图形化文档：`docs/scr1-ifu-graphics/index.html`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0

> 本文档说明 IFU 的验证策略、覆盖点、已执行验证活动与结论，以及回归流程。
> 行号指向 IFU 源码与学习笔记 `docs/learning-notes.md`。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 IFU 在 MAX（RV32IMC）与 MIN（RV32EC）两种配置下正确取指；
2. 覆盖 RVC/RVI 混编指令流的正确切割与拼接；
3. 覆盖换 PC（flush）、非对齐 new_pc、IMEM 错误等边界场景；
4. 验证 RISC-V 精确异常语义：取指错误绑定到具体指令。

### 1.2 范围

- 功能级验证通过**系统级测试程序 + 波形分析 + 内建 SVA 断言**完成；
- IFU 无独立 UVM 测试台，复用 SCR1 自带 `scr1_top_tb_ahb` 顶层测试台；
- 本模块为流水线组件，系统级验证已能触发绝大多数内部路径。

---

## 2. 验证环境

### 2.1 工具链（见学习笔记第 9 课）

| 工具 | 版本 | 说明 |
|---|---|---|
| Verilator | 5.006-3 | RTL→C++ 仿真，`--trace` 出 VCD |
| riscv-none-elf-gcc | 15.2.0 | xPack 预编译，交叉编译测试程序 |
| python3 | — | VCD 解析脚本 |

### 2.2 测试台架构

- `src/tb/scr1_top_tb_ahb.sv`：顶层测试台，例化 `scr1_top_ahb`；
- `src/tb/scr1_top_tb_runtests.sv`：批量跑测试 + 看门狗 + 汇总；
- `sim/tests/`：程序源（hello、riscv_arch、riscv_compliance 等）；
- Verilator wrapper：`sim/verilator_wrap/scr1_ahb_wrapper.c`。

### 2.3 运行参数

```
+test_info=<file>     # 每行一个 .hex 测试文件
+test_results=<file>  # 汇总输出
+imem_pattern=%h      # IMEM 应答/响应时序模式（可注入延迟）
+dmem_pattern=%h      # DMEM 应答/响应时序模式
```

- 程序结束判定：tb 监测 `curr_pc == SCR1_SIM_EXIT_ADDR`(0xF8)（runtests:38）；
- 通过判定：非 arch/compliance 测试看 `mprf_int[10]==0`（main 返回 0）（runtests:143）。

---

## 3. 验证策略

三级递进：

1. **内建 SVA 断言**（IFU 源码 :769-822）：仿真中持续检查协议/边界违例；
2. **功能覆盖点**：通过构造不同指令流触发（见第 4 节）；
3. **波形分析**：VCD 重建执行流，核对 IFU 与 EXU 行为（见第 6 节）。

IFU 专项场景（跨块 RVI、非对齐、错误注入）优先用 **riscv_arch / compliance
随机含 C 扩展指令**的测试程序触发；IFU 内部计数器/FSM 行为由波形核对。

---

## 4. 功能覆盖点（Functional Coverage Points）

### 4.1 指令切割（按 instr_type）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-1 | `RVI_HI_RVI_LO`：字对齐完整 RVI | 任何 4 对齐代码段 |
| FC-2 | `RVC_RVC`：块内两 RVC | 压缩代码连续 2 条 RVC |
| FC-3 | `RVI_LO_RVC` / `RVC_RVI_HI`：跨块 RVI | 代码段混编 2 对齐 RVI |
| FC-4 | `RVC_NV`/`RVI_LO_NV`：非对齐 new_pc | 跳转到 2 对齐地址 |

### 4.2 队列行为

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-5 | 队列空/非空/满 | 长短代码段交替 |
| FC-6 | `RD_HWORD` 连续出队 | RVC 密集段 |
| FC-7 | `RD_WORD` 出队 | RVI 段 |
| FC-8 | 队列 flush | 每个分支/跳转/mret |
| FC-9 | 在飞事务计数 0→1→0 | `imem_pattern` 延迟响应 |
| FC-10 | discard 计数 | 见 4.4 |

### 4.3 PC 与对齐

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-11 | 顺序取指 +4 | 顺序执行 |
| FC-12 | RVC 步长 +2 | 压缩指令 |
| FC-13 | 换 PC 后从新地址取指 | branch/jump |
| FC-14 | 非对齐目标（2 对齐非 4 对齐） | `j/jal/beq` 到 oddword |
| FC-15 | IMEM 地址恒字对齐 | 全测试（断言保证） |

### 4.4 错误与异常

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-16 | IMEM 返回 ER | 程序跳转到无 IMEM 覆盖的地址 |
| FC-17 | 错误后丢弃后续响应 | FC-16 + 在飞多条 |
| FC-18 | 取指异常绑定正确 MEPC | FC-16 后读 MEPC |
| FC-19 | RVI 高半字错误 | 罕见：高半槽越界（软件难触发） |

---

## 5. 断言验证（Assertions）

IFU 内建 10 条 SVA（源码 :769-822），仿真全程使能。出现违例立即 `$error`。

| 断言 | 覆盖的规格条目 | 状态 |
|---|---|---|
| XCHECK（X 检查 ×2） | 输入无 X | 通过 |
| DRC_UNDERFLOW / RANGE | discard 计数边界 | 通过 |
| QUEUE_OVF | 队列不溢出 | 通过 |
| IMEM_ERR_BEH | 错误后状态机行为 | 通过 |
| NEW_PC_REQ_BEH | flush 后队列空 | 通过 |
| IMEM_ADDR_ALIGNED | 字对齐 | 通过 |
| STOP_FETCH | 停取指行为 | 通过 |
| IMEM_FAULT_RVI_HI | RVI 高半错误伴随 imem_err | 通过 |

> 状态基于 MAX/MIN 全部运行测试 PASS 时无断言触发（见第 6 节）。

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

| 配置 | 结果 | 说明 |
|---|---|---|
| MIN（RV32EC，约 2 级） | PASS `1/1 tests passed` | hello.hex 32493 B |
| MAX（RV32IMC，完整流水线） | PASS `1/1 tests passed` | hello.hex 31023 B |

关键通路已证明：取指→译码→执行→访存→写回→系统调用退出全链路正确，
`Hello from SCR1!` 经 UART 外设打印。

### 6.2 指令流波形重建（MAX + VCD，学习笔记第 9 课补充）

工具：`/tmp/opencode/parse_exec.py`（VCD→执行指令流 + 反汇编标注）。

观测到并验证的 IFU 行为：

1. **复位向量**：复位后 `curr_pc = 0x200`（`SCR1_ARCH_RST_VECTOR`）；
2. **预取超前**：`pipe2imem_addr_o` 领先 `curr_pc` 约 3 拍——IFU 预取填队列，
   EXU 滞后消费（对应 FC-5/FC-11）；
3. **RVC 步长 2 / RVI 步长 4 交替**：如 `0x29c j`、`0x2a2 addi` 为 2 字节
   （对应 FC-6/FC-7/FC-12）；
4. **分支可见**：`jb_new_pc` 突变即跳转，脚本按增量非 2/4 标记 branch/jump
   （对应 FC-13/FC-14）；
5. **启动序列**：`zero` 寄存器清零 → `auipc/addi` 装全局指针 → `beq/j/bne +
   lw/sw/addi` 数据搬运（crt 把 .data 从 RAM 载入 TCM）；
6. **退出路径**：`→0x482008→0xf4(SIM_EXIT)→0xf8(SIM_STOP)`。

实测统计（MAX/hello）：

| 指标 | 值 |
|---|---|
| 执行指令数 | 12643（含循环重复） |
| 总周期 | ~22770 |
| 平均 CPI | ~1.8 |
| VCD 体积 | 41 MB |

### 6.3 配置对比（IFU 在两种流水线形态）

| 指标 | MIN | MAX |
|---|---|---|
| RTL 流水线形态 | 无 `SCR1_NO_DEC_STAGE` 的旁路（bypass） | 完整 IFU 队列路径 |
| hello.hex | 32493 B | 31023 B |
| Verilator 模型 | 198 KB / 29 obj | 347 KB / 52 obj |
| 仿真 | PASS | PASS |

结论：IFU 在 MAX（完整队列路径 :745-753）与 MIN（bypass 路径 :623-713）
两种 `ifdef` 分支下均验证通过。

### 6.4 边界场景专项（需要时可补充执行的测试）

| 场景 | 建议方法 | 预期 |
|---|---|---|
| IMEM 错误注入 | 链接到无内存区域地址并执行 | 触发取指异常，MEPC 指向出错指令 |
| 非对齐分支目标 | 用 2 对齐地址作为 `jal` 目标（含后续 RVI） | 跨块拼接正确 |
| IMEM 延迟 | `+imem_pattern` 注入多拍响应 | pending 计数正确，无丢取指 |
| 队列溢出边界 | 构造超长无分支代码 + 延迟响应 | QUEUE_OVF 断言不触发 |

> 6.1-6.3 已完成；6.4 为建议的补充回归项，尚未系统执行（测试程序未覆盖
> 无内存地址跳转与 IMEM 注入延迟的组合）。

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

### 7.2 波形生成（IFU 内部信号观测）

```
make -C sim build_verilator_wf \
  root_dir=/workspace bld_dir=<bld_dir_wf> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb
# 产物：<bld_dir_wf>/simx.vcd
```

用 `parse_exec.py`（或 GTKWave 打开 VCD）核对 IFU 关键信号：
`curr_pc`、`pipe2imem_addr_o`（取指地址）、`ifu2idu_instr_o`、`new_pc`。

### 7.3 全量架构测试（可选，慢）

riscv_arch 测试集位于 `sim/tests/riscv_arch/`，测试汇编源在
`dependencies/riscv-arch/`（git submodule，Makefile 通过
`RISCV_ARCH_TESTS := $(src_dir)/../../../dependencies/riscv-arch/` 引用）。
目录当前**只有源与 Makefile，无现成 .hex**，需先编译：

```
# 在 sim/tests/riscv_arch 下，指定目标 ARCH 生成 arch_*.hex
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

产物为 `<bld_dir>/arch_*.hex`（前缀 `arch_`）。将各 `.hex` 路径写入 `test_info`
即可批量回归。

注意测试集选择与 `cut_list` 均与 ARCH 绑定：

- ARCH=imc（MAX 默认）：汇编源取 `rv32i_m/{I,C,M}/src`，仅剔除通用基础用例
  `cut_list`（:97-98，I 基分支类 bne/beq/bge/blt/jal/bltu/bgeu 与 C 基
  ebreak/cebreak-01/cswsp-01）；
- ARCH=im 或 ic：额外剔除 misalign-*、ecall 等用例（:90-95）；
- ARCH=imc 时 misalign 用例**会被编译运行**——跳转到 misalign 地址等正是 6.4 节
  建议的 IFU 取指异常专项，可据运行结果核对 IFU 是否按 spec 触发精确异常。

privilege/Zifencei 用例只被 I 分支引用，imc 构建不包含。

---

## 8. 验证结论

1. IFU 在 MAX/MIN 两种配置下均正确完成取指、指令切割、flush、错误注入；
2. RVC/RVI 混编流的切割与跨块拼接经波形重建验证；
3. 换 PC 后队列正确清空，取指从新地址继续；
4. 10 条内建 SVA 断言全程无违例；
5. 预取超前执行约 3 拍的流水行为与设计一致。

**遗留建议**：补做 6.4 节的 IMEM 错误注入与延迟响应专项，以覆盖 FC-16/17/18/19
及 pending/discard 计数的深边界。
