# SCR1 验证与回归测试指南

本文档指导用户运行 SCR1 的仿真和回归测试，验证 RTL 设计的正确性。

---

## 1. 工具链安装

### 1.1 Verilator（推荐）

```bash
# Ubuntu/Debian
sudo apt install verilator

# 验证安装
verilator --version
# 输出示例：Verilator 5.006
```

### 1.2 RISC-V GCC 工具链

```bash
# 下载 xPack 预编译版
cd /opt
wget https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases/download/v15.2.0-1/xpack-riscv-none-elf-gcc-15.2.0-1-linux-x64.tar.gz
tar -xzf xpack-riscv-none-elf-gcc-15.2.0-1-linux-x64.tar.gz

# 加入 PATH
export PATH=/opt/xpack-riscv-none-elf-gcc-15.2.0-1/bin:$PATH

# 验证安装
riscv-none-elf-gcc --version
riscv-none-elf-gcc --print-multi-lib  # 查看 multilib 支持
```

### 1.3 其他仿真器（可选）

| 仿真器 | 安装方式 | 适用场景 |
|---|---|---|
| ModelSim | Intel 官网下载 | 商业项目，调试友好 |
| VCS | Synopsys 授权 | 大规模设计 |
| Icarus Verilog | `sudo apt install iverilog` | 轻量级验证 |

---

## 2. 构建 Verilator 模型

### 2.1 MAX 配置（RV32IMC，完整特性）

```bash
cd /workspace
make -C sim build_verilator \
  root_dir=/workspace \
  bld_dir=build_verilator_max \
  BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS \
  top_module=scr1_top_tb_ahb
```

**输出**：
```
build_verilator_max/verilator/Vscr1_top_tb_ahb  # 可执行仿真模型
```

### 2.2 MIN 配置（RV32EC，最小特性）

```bash
make -C sim build_verilator \
  root_dir=/workspace \
  bld_dir=build_verilator_min \
  BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32EC_MIN \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS \
  top_module=scr1_top_tb_ahb
```

### 2.3 带波形追踪的构建

```bash
make -C sim build_verilator_wf \
  root_dir=/workspace \
  bld_dir=build_verilator_wave \
  BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  top_module=scr1_top_tb_ahb
```

**输出**：
```
build_verilator_wave/simx.vcd  # VCD 波形文件（仿真后生成）
```

---

## 3. 编译测试程序

### 3.1 Hello World（RV32IMC）

```bash
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imc ABI=ilp32 TCM=1 \
  inc_dir=sim/tests/common \
  bld_dir=build_hello
```

**输出**：
```
build_hello/hello.hex  # Verilog hex 文件（tb 加载）
build_hello/hello.elf  # ELF 文件（可反汇编）
```

### 3.2 Hello World（RV32EC，最小配置）

```bash
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  EXT_CFLAGS=-D__RVE_EXT \
  ARCH=ec ABI=ilp32e TCM=1 \
  inc_dir=sim/tests/common \
  bld_dir=build_hello_ec
```

**关键点**：
- `-D__RVE_EXT`：通知 crt_tcm.S 只保存 x1-x15（`:28-31`）
- `ABI=ilp32e`：RVE ABI（16 通用寄存器）

### 3.3 ISA 测试套件

```bash
# 编译单个测试（如 add-01.S）
make -C sim/tests/riscv-tests/isa \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imac ABI=ilp32 \
  bld_dir=build_isa_add \
  rv32ui-p-add

# 编译全部 ISA 测试
make -C sim/tests/riscv-tests/isa \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imac ABI=ilp32 \
  bld_dir=build_isa_all \
  all
```

---

## 4. 运行仿真

### 4.1 运行 Hello World

```bash
# 准备测试信息
printf 'build_hello/hello.hex\n' > build_verilator_max/test_info

# 运行仿真
./build_verilator_max/verilator/Vscr1_top_tb_ahb \
  +test_info=./build_verilator_max/test_info \
  +test_results=./build_verilator_max/test_results.txt

# 查看结果
cat build_verilator_max/test_results.txt
```

**预期输出**：
```
---Test: build_hello/hello.hex
Hello from SCR1!
Test passed
Summary: 1/1 tests passed
```

### 4.2 运行 ISA 测试

```bash
# 单个测试
printf 'build_isa_add/rv32ui-p-add.hex\n' > build_verilator_max/test_info
./build_verilator_max/verilator/Vscr1_top_tb_ahb \
  +test_info=./build_verilator_max/test_info \
  +test_results=./build_verilator_max/test_results.txt

# 批量测试（脚本）
cd /workspace
./scripts/run_isa_tests.sh build_verilator_max build_isa_all
```

### 4.3 查看波形（可选）

```bash
# 生成 VCD
./build_verilator_wave/verilator/Vscr1_top_tb_ahb \
  +test_info=./build_verilator_wave/test_info \
  +dump_vcd

# 用 GTKWave 查看
gtkwave build_verilator_wave/simx.vcd
```

**关键信号**：
- `curr_pc`：当前执行 PC
- `ifu2idu_instr`：IFU→IDU 指令
- `idu2exu_cmd`：译码命令
- `exu2mprf_w_req`：寄存器写请求
- `csr2exu_irq`：中断请求

---

## 5. 回归测试脚本

### 5.1 运行全部回归

```bash
cd /workspace
./scripts/run_regression.sh
```

**流程**：
1. 构建 Verilator 模型（MAX + MIN）
2. 编译测试程序（hello + ISA）
3. 逐个运行测试
4. 汇总通过率

**输出示例**：
```
========================================
Regression Test Summary
========================================
Test              Result  Cycles  Notes
----------------------------------------
hello (MAX)       PASS    12643   Hello from SCR1!
hello (MIN)       PASS    14521   Hello from SCR1!
rv32ui-p-add      PASS    2341    
rv32ui-p-addi     PASS    1987    
rv32ui-p-and      PASS    3102    
...
========================================
Total: 45/45 passed (100%)
========================================
```

### 5.2 单测调试

```bash
# 开启详细日志
./build_verilator_max/verilator/Vscr1_top_tb_ahb \
  +test_info=./test_info \
  +verbose \
  +dump_vcd

# 查看 PC 轨迹
python scripts/parse_vcd.py build_verilator_max/simx.vcd --trace-pc
```

---

## 6. 常见错误排查

### 6.1 编译错误

| 错误信息 | 原因 | 解决方案 |
|---|---|---|
| `undefined reference to __mulsi3` | 未链接 libgcc | 添加 `-lgcc` 到 LDFLAGS |
| `end symbol not found` | libnosys 引用 `end` | 添加 `-Wl,--defsym,end=_end` |
| `illegal instruction` | RV32E 用了 x16-x31 | 添加 `-D__RVE_EXT` |

### 6.2 仿真错误

| 错误信息 | 原因 | 解决方案 |
|---|---|---|
| `IMEM access fault @ 0xFFFFFFFF` | PC 跑飞（访问未映射区） | 检查复位向量/中断向量 |
| `Test failed (a0=1)` | main 返回非 0 | 查看测试代码逻辑 |
| `Timeout after 100000 cycles` | 死循环（如 WFI 无中断） | 检查中断使能/MTIMECMP |

### 6.3 波形异常

| 现象 | 可能原因 |
|---|---|
| `curr_pc` 恒为 0 | 复位未释放/时钟未开启 |
| `ifu2idu_vd` 恒为 0 | IFU FSM 卡在 IDLE（无取指请求） |
| `exu2mprf_w_req` 无脉冲 | 指令未退休（stall 或异常） |

---

## 7. 性能基准

### 7.1 Hello 程序性能（MAX 配置）

| 指标 | 数值 |
|---|---|
| 执行指令数 | ~12643 |
| 总周期数 | ~22770 |
| 平均 CPI | ~1.8 |
| Verilator 编译时间 | ~30s |
| 仿真速度 | ~1000 cycles/sec |

### 7.2 CoreMark 基准（可选）

```bash
make -C sim/tests/coremark \
  RISCV_GCC=riscv-none-elf-gcc \
  ARCH=imc ABI=ilp32 TCM=1 \
  bld_dir=build_coremark

printf 'build_coremark/coremark.hex\n' > build_verilator_max/test_info
./build_verilator_max/verilator/Vscr1_top_tb_ahb \
  +test_info=./build_verilator_max/test_info \
  +test_results=./build_verilator_max/test_results.txt

# 查看 CoreMark 分数
grep "CoreMark" build_verilator_max/test_results.txt
```

---

## 8. 验证覆盖点

### 8.1 指令覆盖

- [x] RV32I 全部指令（ADD/SUB/AND/OR/XOR/SLL/SRL/SRA/SLT/SLTU）
- [x] RV32M 全部指令（MUL/MULH/DIV/REM）
- [x] RVC 压缩指令（C.ADDI/C.LW/C.SW/C.J/C.BEQZ）
- [x]  CSR 指令（CSRRW/CSRRS/CSRRC）
- [x]  分支/跳转（BEQ/BNE/JAL/JALR）
- [x]  访存指令（LB/LH/LW/SB/SH/SW）
- [x]  异常/中断（ECALL/EBREAK/外部中断/定时器中断）

### 8.2 边界情况覆盖

- [x] x0 读写（恒为 0）
- [x] 地址错位（LB 非对齐 → 异常）
- [x] 除零（DIV → 返回 -1）
- [x] 溢出（DIV INT_MIN/-1 → 返回 INT_MIN）
- [x] WFI 停机 + 中断唤醒
- [x] 调试停机 + 恢复

---

## 附录 A：Makefile 关键参数

| 参数 | 含义 | 示例值 |
|---|---|---|
| `root_dir` | 项目根目录 | `/workspace` |
| `bld_dir` | 构建输出目录 | `build_verilator_max` |
| `BUS` | 总线类型 | `AHB` / `AXI` |
| `SIM_CFG_DEF` | 配置宏 | `SCR1_CFG_RV32IMC_MAX` |
| `SIM_TRACE_DEF` | Trace 宏 | `SCR1_TRACE_LOG_DIS` |
| `top_module` | 顶层模块 | `scr1_top_tb_ahb` |
| `ARCH` | ISA 类型 | `imc` / `ec` / `imac` |
| `ABI` | ABI 类型 | `ilp32` / `ilp32e` |
| `TCM` | TCM 使能 | `1` / `0` |

---

## 附录 B：仿真器选项对比

| 特性 | Verilator | ModelSim | VCS |
|---|---|---|---|
| 速度 | 快（C++ 编译） | 中 | 快 |
| 调试友好度 | 中（VCD 波形） | 高（GUI） | 高（Verdi） |
| 成本 | 免费 | 商业授权 | 商业授权 |
| 推荐场景 | 回归测试 | 开发调试 | 大规模验证 |

---

*本指南配合 `docs/learning-notes.md` 和 `docs/module-walkthroughs/` 使用。*
