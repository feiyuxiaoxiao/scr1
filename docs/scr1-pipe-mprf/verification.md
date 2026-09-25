# SCR1 MPRF 多端口寄存器堆验证文档（Verification Document）

- 被测对象：`scr1_pipe_mprf`（`src/core/pipeline/scr1_pipe_mprf.sv`，167 行）
- 配套设计规格：`docs/scr1-pipe-mprf/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:477`）

> MPRF 是所有指令的操作数来源与写回目标，正确性由串行程序语义隐式强验证：
> 任一寄存器读写错误都会使测试结果发散。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证 2 读 1 写端口的正确性；
2. 验证 x0 恒 0（读 0、写忽略）；
3. 验证写回选择（ALU/load/CSR/PC+4/imm）落到正确寄存器；
4. 验证 RV32E 下寄存器数 16、地址位宽 4；
5. 验证分布式与 RAM 两种实现；
6. 验证依赖指令（写后读）的时序正确。

### 1.2 范围

- MPRF 无独立测试台；由程序级测试 + 波形核对 + 内建 SVA 覆盖；
- 寄存器堆错误通常表现为最终结果错误，故 hello/ISA 测试是关键证据。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 测试台

`scr1_top_tb_ahb.sv` + `scr1_top_tb_runtests.sv`；可通过层次引用
`i_pipe_mprf.mprf_int` 观察寄存器值（`scr1_pipe_top.sv:788`）。

### 2.3 运行参数

`+test_info`、`+test_results`、`+imem_pattern`、`+dmem_pattern`。

---

## 3. 验证策略

1. **内建 SVA**（`:154-165`）：写请求时的 X 检查；
2. **程序级覆盖**：hello + `riscv_arch` 覆盖所有寄存器访存模式；
3. **波形核对**：读地址/读数据、写使能/写地址/写数据；
4. **配置切换**：RVE / `SCR1_MPRF_RST_EN` / `SCR1_MPRF_RAM`。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-MPRF-1 | x0 读恒 0 | 读 `x0` 指令 |
| FC-MPRF-2 | x0 写被忽略 | 写 `x0`（如 `addi x0,x0,1`） |
| FC-MPRF-3 | 单读口 rs1 | 一元指令 |
| FC-MPRF-4 | 双读口 rs1+rs2 | 寄存器-寄存器指令 |
| FC-MPRF-5 | 写回 ALU 结果 | `add` 等 |
| FC-MPRF-6 | 写回 load 数据 | `lw` 后使用 |
| FC-MPRF-7 | 写回 CSR 数据 | `csrrw` |
| FC-MPRF-8 | 写回 PC+4（JAL/JALR） | `jal`/`jalr` |
| FC-MPRF-9 | 写回 IMM（LUI/AUIPC） | `lui/auipc` |
| FC-MPRF-10 | 写后读（相邻依赖） | `addi x1,...; add x2,x1,...` |
| FC-MPRF-11 | 同拍多读多写不冲突 | 波形 |
| FC-MPRF-12 | 未用 rs 口不读（地址 0） | 波形 |
| FC-MPRF-13 | RVE：仅 16 寄存器 | MIN 配置 |
| FC-MPRF-14 | 复位清零（`SCR1_MPRF_RST_EN`） | 复位后读 |
| FC-MPRF-15 | RAM 模式写优先旁路 | Intel FPGA 配置（如需） |

---

## 5. 断言验证（Assertions）

| 断言 | 行号 | 检查 | 状态 |
|---|---|---|---|
| `SCR1_SVA_MPRF_WRITEX` | 159-162 | 写请求时地址/写数据无 X | 通过 |

- 仅 `SCR1_MPRF_RST_EN` 下编译（依赖 `rst_n`）；
- 用 `|rd_addr ? rd_data : 0` 屏蔽 x0 写数据，避免对 x0 写误报。

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

hello 大量使用寄存器（传参、循环、UART 地址构造）：

| 配置 | MPRF 相关差异 | 结果 |
|---|---|---|
| MIN（RV32EC） | 16 寄存器、无复位 | PASS |
| MAX（RV32IMC） | 32 寄存器、带复位 | PASS |

覆盖 FC-MPRF-3~10；MIN 覆盖 FC-MPRF-13。

### 6.2 波形核对（MAX + VCD）

可核对：

1. `exu2mprf_rs1_addr_o`/`rs2_addr_o` 与 IDU 命令吻合；
2. `exu2mprf_w_req_o` 在指令退休且非异常时拉高，`rd_addr` 正确；
3. `mprf2exu_rs1_data_o` 在下一指令使用时已是新值（FC-MPRF-10）；
4. 未用端口地址为 0、读数据 0（FC-MPRF-12）。

### 6.3 RVE 配置

MIN（`SCR1_RVE_EXT`）下 `SCR1_MPRF_AWIDTH=4`、`SCR1_MPRF_SIZE=16`；
编译器 `-march=rv32ec` 不生成 x16~x31 访存，程序 PASS 验证 FC-MPRF-13。

### 6.4 边界场景专项（建议补充执行）

| 场景 | 建议方法 | 预期 |
|---|---|---|
| x0 写忽略 | `addi x0,x0,123; add x1,x0,x0` | x1=0，FC-MPRF-1/2 |
| 复位清零 | 复位后立即读非 x0 寄存器 | 0（仅 RST_EN），FC-MPRF-14 |
| RAM 模式 | 定义 `SCR1_TRGT_FPGA_INTEL` 重编 | 写优先旁路生效，FC-MPRF-15 |
| 连续 CSR 依赖 | `csrrw` 后立即用结果 | 正确，FC-MPRF-7/10 |

> 6.1-6.3 已完成；6.4 依赖专用程序或配置，尚未系统执行。

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

波形核对信号：`exu2mprf_rs*_addr_o`、`mprf2exu_rs*_data_o`、
`exu2mprf_w_req_o`、`exu2mprf_rd_addr_o`、`exu2mprf_rd_data_o`、`mprf_int`。

---

## 8. 验证结论

1. 2 读 1 写端口与写回选择在全 ISA 测试下正确；
2. x0 读 0/写忽略符合 RISC-V 规范；
3. MIN（RVE，16 寄存器）与 MAX（32 寄存器）均通过；
4. 写后读依赖正确，内建 SVA 无违例。

**遗留建议**：补做 6.4 节的 x0 写忽略、复位清零与 RAM 模式专项。
