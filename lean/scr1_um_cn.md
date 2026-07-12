# SCR1 用户手册

Syntacore, scr1@syntacore.com

版本 1.1.4, 2022-01-13

## 目录

修订历史  
1. SCR1 概述  
  1.1. SCR1 版本  
  1.2. 主要特性  
2. 代码库概览  
  2.1. SCR1 仓库内容  
  2.2. SCR1 RTL 源文件和测试平台文件  
3. 核心配置  
  3.1. 核心与设备标识符  
  3.2. 推荐配置  
  3.3. 自定义配置的精细调节选项  
  3.4. 核心集成选项  
  3.5. 仿真选项  
4. 仿真环境  
  4.1. 环境要求  
    4.1.1. 操作系统  
    4.1.2. RISC-V GCC 工具链  
    4.1.3. HDL 仿真器  
    4.1.4. 测试准备  
  4.2. 运行仿真  
    4.2.1. 仿真器选择  
    4.2.2. 架构配置  
  4.3. 测试目标  
  4.4. 仿真代码  
    4.4.1. 跟踪日志(Tracelog)  
  4.5. 测试平台说明  
5. SDK 信息  
6. 技术支持

## 修订历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2018-05-07 | 初始版本 |
| 1.0.1 | 2018-09-19 | RTL 配置和仿真脚本更新 |
| 1.0.2 | 2018-10-09 | 更新 MIMPID |
| 1.0.3 | 2019-03-19 | 更新兼容 MIMPID=0x19031802 |
| 1.0.4 | 2019-04-11 | 更新兼容 MIMPID=0x19040301 |
| 1.0.5 | 2019-05-07 | 仿真环境更新 |
| 1.0.6 | 2019-08-30 | 更新 MIMPID=0x19083000 |
| 1.0.7 | 2019-10-28 | 更新测试子集章节 |
| 1.1.0 | 2019-12-13 | 新增 SCR1 集群图；文件列表详细说明；仿真设置流程更新；SCR1 SDK 仓库更多信息 |
| 1.1.1 | 2020-07-15 | 更新合规性测试，新增 TCM 选项 |
| 1.1.2 | 2020-11-13 | 更新兼容 MIMPID=0x20111300 |
| 1.1.3 | 2021-02-08 | 更新兼容 MIMPID=0x21020800 |
| 1.1.4 | 2022-01-13 | MIMPID=0x22011200 |

## 1. SCR1 概述

SCR1 是一个开源且免费的 RISC-V 兼容 MCU 级核心，由 Syntacore 设计和维护。详细信息请参见根目录下的 LICENSE 文件。

### 1.1. SCR1 版本

本文档适用于 MIMPID 值为 0x22011200 的 SCR1 核心。

### 1.2. 主要特性

- 基于 SHL 许可证开源（详见 LICENSE 文件）—— 允许无限制商业使用
- RV32I 或 RV32E 基础 ISA，可选 RVM 和 RVC 标准扩展
- 仅支持机器(Machine)特权模式
- 2 至 4 级流水线
- 可选集成可编程中断控制器(IPIC)，含 16 条 IRQ 线
- 可选 RISC-V 调试子系统，带 JTAG 接口
- 可选片上紧耦合存储器(TCM)
- 32 位 AXI4/AHB-Lite 外部接口
- 使用 SystemVerilog 编写
- 针对面积和功耗进行了优化
- 3 种预定义推荐配置
- 多种自定义配置微调选项
- 提供验证套件
- 丰富的文档

关于核心架构的更多信息，请参见 SCR1 外部架构规格书(EAS)。

## 2. 代码库概览

### 2.1. SCR1 仓库内容

| 文件夹 | 描述 |
|--------|------|
| dependencies | 依赖的子模块 |
| riscv-tests | RISC-V ISA 测试的通用源文件 |
| riscv-compliance | RISC-V 合规性测试的通用源文件 |
| coremark | EEMBC CoreMark® 基准测试的通用源文件 |
| docs | SCR1 文档 |
| scr1_eas.pdf | SCR1 外部架构规格书 |
| scr1_um.pdf | SCR1 用户手册 |
| sim | 仿真测试和脚本 |
| tests/common | 测试通用源文件 |
| tests/riscv_isa | RISC-V ISA 测试平台特定源文件 |
| tests/riscv_compliance | RISC-V 合规性测试平台特定源文件 |
| tests/benchmarks/dhrystone21 | Dhrystone 2.1 基准测试源文件 |
| tests/benchmarks/coremark | EEMBC CoreMark® 基准测试平台特定源文件 |
| tests/isr_sample | 中断服务例程示例程序 |
| tests/hello | "Hello" 示例程序 |
| verilator_wrap | Verilator 仿真的封装器 |
| src | SCR1 RTL 源文件和测试平台文件 |
| includes | 头文件 |
| core | 核心顶层源文件 |
| top | 集群源文件 |
| tb | 测试平台文件 |

### 2.2. SCR1 RTL 源文件和测试平台文件

SCR1 源文件列表位于 ./src 目录下：

- core.files - SCR1 核心所有可综合源文件
- ahb_top.files - AHB 集群可综合源文件
- axi_top.files - AXI 集群可综合源文件
- ahb_tb.files - AHB 集群测试平台源文件（仅仿真）
- axi_tb.files - AXI 集群测试平台源文件（仅仿真）

头文件库位于 ./src/includes/ 目录。

以下是所有源文件的完整列表。

| 路径 | 描述 |
|------|------|
| **SCR1 头文件** | |
| includes/scr1_ahb.svh | AHB 头文件 |
| includes/scr1_arch_description.svh | 架构描述文件 |
| includes/scr1_arch_types.svh | 流水线类型描述文件 |
| includes/scr1_csr.svh | CSR 映射/描述文件 |
| includes/scr1_dm.svh | DM 头文件 |
| includes/scr1_hdu.svh | HDU 头文件 |
| includes/scr1_ipic.svh | IPIC 头文件 |
| includes/scr1_memif.svh | 存储器接口定义文件 |
| includes/scr1_riscv_isa_decoding.svh | RISC-V ISA 定义文件 |
| includes/scr1_scu.svh | SCU 头文件 |
| includes/scr1_search_ms1.svh | 最高有效1搜索函数 |
| includes/scr1_tapc.svh | TAPC 头文件 |
| includes/scr1_tdu.svh | TM 头文件 |
| **SCR1 核心源文件** | |
| core/pipeline/scr1_ipic.sv | 集成可编程中断控制器(IPIC) |
| core/pipeline/scr1_pipe_csr.sv | 控制状态寄存器(CSR) |
| core/pipeline/scr1_pipe_exu.sv | 执行单元(EXU) |
| core/pipeline/scr1_pipe_hdu.sv | HART调试单元(HDU) |
| core/pipeline/scr1_pipe_ialu.sv | 整数算术逻辑单元(IALU) |
| core/pipeline/scr1_pipe_idu.sv | 指令译码单元(IDU) |
| core/pipeline/scr1_pipe_ifu.sv | 指令取指单元(IFU) |
| core/pipeline/scr1_pipe_lsu.sv | 加载/存储单元(LSU) |
| core/pipeline/scr1_pipe_mprf.sv | 多端口寄存器文件(MPRF) |
| core/pipeline/scr1_pipe_tdu.sv | 触发器调试单元(TDU) |
| core/pipeline/scr1_pipe_top.sv | SCR1 流水线顶层 |
| core/pipeline/scr1_tracelog.sv | 核心跟踪日志模块（仅仿真） |
| core/primitives/scr1_cg.sv | SCR1 时钟门控原语 |
| core/primitives/scr1_reset_cells.sv | SCR1 复位逻辑原语 |
| core/scr1_clk_ctrl.sv | SCR1 时钟控制 |
| core/scr1_core_top.sv | SCR1 核心顶层 |
| core/scr1_dm.sv | 调试模块(DM) |
| core/scr1_dmi.sv | 调试模块接口(DMI) |
| core/scr1_scu.sv | 系统控制单元 |
| core/scr1_tapc.sv | TAP 控制器(TAPC) |
| core/scr1_tapc_shift_reg.sv | TAPC 移位寄存器 |
| core/scr1_tapc_synchronizer.sv | TAPC 跨时钟域同步器 |
| **SCR1 顶层集群源文件** | |
| top/scr1_dmem_ahb.sv | 数据存储器 AHB 桥 |
| top/scr1_dmem_router.sv | 数据存储器路由器 |
| top/scr1_dp_memory.sv | 带字节使能的同步双端口存储器 |
| top/scr1_imem_ahb.sv | 指令存储器 AHB 桥 |
| top/scr1_imem_router.sv | 指令存储器路由器 |
| top/scr1_mem_axi.sv | 存储器 AXI 桥 |
| top/scr1_tcm.sv | 紧耦合存储器(TCM) |
| top/scr1_timer.sv | 内存映射定时器 |
| top/scr1_top_ahb.sv | SCR1 AHB 顶层 |
| top/scr1_top_axi.sv | SCR1 AXI 顶层 |
| **测试平台文件** | |
| tb/scr1_memory_tb_ahb.sv | AHB 存储器测试平台 |
| tb/scr1_memory_tb_axi.sv | AXI 存储器测试平台 |
| tb/scr1_top_tb_ahb.sv | SCR1 AHB 顶层测试平台 |
| tb/scr1_top_tb_axi.sv | SCR1 AXI 顶层测试平台 |
| tb/scr1_top_tb_runtests.sv | 测试平台运行测试 |

## 3. 核心配置

### 3.1. 核心与设备标识符

下表展示了 SCR1 核心和设备的标识符。

| 标识符名称 | 描述 |
|-----------|------|
| MIMPID | SCR1 核心实现 ID，通过对应 CSR 读取。此编号唯一标识 SCR1 核心 RTL 的版本 |
| MARCHID | SCR1 核心架构 ID，通过对应 CSR 读取。该编号将 SCR1 核心与其他 RISC-V 核心区分开。硬连线为 0x00000008 |
| MVENDORID | SCR1 核心供应商 ID，通过对应 CSR 读取。用于商业制造，请将此字段重写为您的 JEDEC 标准制造商 ID 代码。默认值为 0x00000000 |
| TAP_IDCODE | 通过 JTAG 从对应 TAPC 寄存器读取的 IDCODE。用于商业制造，请将此字段重写为您的 JEDEC 标准制造商 ID 代码+部件号+版本。默认值为 0xDEB11001（非 JEDEC） |
| BUILD_ID | 设备构建 ID。该编号需在外部 arch_custom.svh 文件中为特定设备构建设置（如 SCR1-SDK 中的 FPGA 构建）。仿真默认值 = MIMPID |

### 3.2. 推荐配置

下表展示了针对典型使用场景的三种 SCR1 推荐配置。这些配置可以在 scr1_arch_description.svh 文件的"推荐核心架构配置"部分轻松启用。要选择配置，请取消列表中唯一对应 define 的注释。

| 选项 | SCR1_CFG_RV32EC_MIN | SCR1_CFG_RV32IC_BASE | SCR1_CFG_RV32IMC_MAX |
|------|:---:|:---:|:---:|
| 指令集 | RV32EC | RV32IC | RV32IMC |
| 流水线级数 | 2 | 3 | 4 |
| GPR 数量 | 16 | 32 | 32 |
| 硬件乘法器 | - | - | + |
| 快速单周期乘法器 | - | - | + |
| 压缩指令 | + | + | + |
| MTVEC 基址可写位 | 0 | 16 | 26 |
| MTVEC 模式可写 | - | + | + |
| 外部 IRQ 线数 | 1 | 16 | 16 |
| 调试子系统 | - | + | + |
| 硬件触发器数量 | 0 | 2 | 4 |
| TCM | + | + | + |

### 3.3. 自定义配置的精细调节选项

SCR1 提供多种自定义配置微调选项，如下表所述。要设计您自己的配置，需编辑 scr1_arch_description.svh 文件：

- 在"推荐核心架构配置"部分取消定义所有推荐配置以启用自定义配置
- 在"自定义核心架构配置"部分选择所需选项：
  - 禁用/启用选项 —— 注释/取消注释相应的 define
  - 数值参数 —— 更改其值

| 名称 | 描述 |
|------|------|
| **RISC-V ISA 选项** | |
| SCR1_RVE_EXT | 启用 RV32E 基础整数指令集，否则使用 RV32I |
| SCR1_RVM_EXT | 启用标准扩展"M"，用于整数硬件乘法器和除法器 |
| SCR1_RVC_EXT | 启用标准扩展"C"，用于压缩指令 |
| SCR1_MTVEC_BASE_WR_BITS | MTVEC.base 字段中可写位数 |
| SCR1_MTVEC_MODE_EN | 启用可写 MTVEC.mode 字段以允许向量中断模式，否则仅直接模式可用 |
| **核心流水线选项（功耗-性能-面积优化）** | |
| SCR1_NO_DEC_STAGE | 禁用 IFU 和 IDU 之间的寄存器 |
| SCR1_NO_EXE_STAGE | 禁用 IDU 和 EXU 之间的寄存器 |
| SCR1_NEW_PC_REG | 在 IFU 中启用新 PC 值寄存器 |
| SCR1_FAST_MUL | 启用快速单周期乘法，否则乘法需要 32 个周期 |
| SCR1_CLKCTRL_EN | 启用全局时钟门控 |
| SCR1_MPRF_RST_EN | 启用 MPRF 复位 |
| SCR1_MCOUNTEN_EN | 启用自定义 MCOUNTEN CSR 用于计数器控制 |
| **非核心选项** | |
| SCR1_DBG_EN | 启用调试子系统（TAPC, DM, SCU, HDU） |
| SCR1_TDU_EN | 启用触发器调试单元（硬件断点） |

综合时若启用 SCR1_CLKCTRL_EN，scr1_cg.sv 中的代码需替换为具体实现的时钟门控。

| 名称 | 描述 |
|------|------|
| SCR1_TDU_TRIG_NUM | 硬件触发器数量 |
| SCR1_TDU_ICOUNT_EN | 启用指令计数器硬件触发器 |
| SCR1_IPIC_EN | 启用集成可编程中断控制器 |
| SCR1_IPIC_SYNC_EN | 启用 IRQ 线 2 级输入同步器 |
| SCR1_TCM_EN | 启用紧耦合存储器，默认大小 64K |

### 3.4. 核心集成选项

SCR1 提供多种集成到上层设计中的选项。这些选项可以在 scr1_arch_description.svh 文件的"核心集成选项"部分修改：

- 禁用/启用选项 —— 注释/取消注释相应的 define
- 数值参数 —— 更改其值

部分选项可以在外部文件 scr1_arch_custom.svh 中定义（该文件不在 SCR1 仓库中，但可用于上层项目，如 SCR1-SDK 及任何自定义 FPGA/ASIC/SoC 项目）。

| 名称 | 描述 |
|------|------|
| **存储器桥接旁路选项** | |
| SCR1_IMEM_AHB_IN_BP | 启用指令存储器 AHB 桥输入端旁路 |
| SCR1_IMEM_AHB_OUT_BP | 启用指令存储器 AHB 桥输出端旁路 |
| SCR1_DMEM_AHB_IN_BP | 启用数据存储器 AHB 桥输入端旁路 |
| SCR1_DMEM_AHB_OUT_BP | 启用数据存储器 AHB 桥输出端旁路 |
| SCR1_IMEM_AXI_REQ_BP | 启用指令存储器 AXI 桥请求旁路 |
| SCR1_IMEM_AXI_RESP_BP | 启用指令存储器 AXI 桥响应旁路 |
| SCR1_DMEM_AXI_REQ_BP | 启用数据存储器 AXI 桥请求旁路 |
| SCR1_DMEM_AXI_RESP_BP | 启用数据存储器 AXI 桥响应旁路 |
| **地址常量** | |
| SCR1_ARCH_RST_VECTOR | 复位向量值（复位后起始地址，默认 0x200） |
| SCR1_ARCH_MTVEC_BASE | MTVEC.base 字段复位值，或硬连线 MTVEC.base 位的常量值（默认 0x1C0） |
| SCR1_TCM_ADDR_MASK | 设置 TCM 掩码和大小；字节大小为掩码值的补码（默认 0xFFFF0000） |
| SCR1_TCM_ADDR_PATTERN | 设置 TCM 地址匹配模式（默认 0x00480000） |
| SCR1_TIMER_ADDR_MASK | 设置定时器掩码（默认 0xFFFFFE0） |
| SCR1_TIMER_ADDR_PATTERN | 设置定时器地址匹配模式（默认 0x00490000） |
| **目标平台（启用特定平台构造）** | |
| SCR1_TRGT_FPGA_INTEL | 目标平台为 Intel FPGA |
| SCR1_TRGT_FPGA_INTEL_MAX10 | 目标平台为 Intel MAX 10 FPGA（SCR1-SDK 项目使用） |
| SCR1_TRGT_FPGA_INTEL_ARRIAV | 目标平台为 Intel Arria V FPGA（SCR1-SDK 项目使用） |
| SCR1_TRGT_FPGA_XILINX | 目标平台为 Xilinx FPGA（SCR1-SDK 项目使用） |
| SCR1_TRGT_AASIC | 目标平台为 ASIC |

### 3.5. 仿真选项

以下参数仅用于仿真，用于启用仿真代码和设置测试平台。这些选项可以在 scr1_arch_description.svh 文件的"仿真选项"部分修改：

- 禁用/启用选项 —— 注释/取消注释相应的 define
- 数值参数 —— 更改其值

| 名称 | 描述 |
|------|------|
| **仿真选项** | |
| SCR1_TRGT_SIMULATION | 启用仿真代码（由根 Makefile 自动定义） |
| SCR1_TRACE_LOG_EN | 启用跟踪日志 |
| **测试平台使用的地址** | |
| SCR1_SIM_EXIT_ADDR | 写入此地址以退出仿真（默认 0x000000F8） |
| SCR1_SIM_PRINT_ADDR | 写入此地址以在控制台打印字符（默认 0xF0000000） |
| SCR1_SIM_EXT_IRQ_ADDR | 写入此地址以生成外部中断（默认 0xF0000100） |
| SCR1_SIM_SOFT_IRQ_ADDR | 写入此地址以生成软件中断（默认 0xF0000200） |

## 4. 仿真环境

项目包含测试平台、测试源文件和脚本，可快速启动 SCR1 仿真。开始仿真前，请确保：

- 已安装 RISC-V GCC 工具链
- 已安装一个受支持的仿真器
- 已初始化包含测试源的子模块

### 4.1. 环境要求

#### 4.1.1. 操作系统

GCC 工具链和 make 脚本受大多数类 Linux 操作系统支持。

在 Windows 上运行可以使用额外的兼容层，如 WSL 或 Cygwin。

#### 4.1.2. RISC-V GCC 工具链

需要 RISC-V GCC 工具链来编译软件。您可以使用预构建的二进制文件或从源码构建工具链。

##### 4.1.2.1. 使用预构建二进制工具

支持所有 SCR1 架构配置的预构建 RISC-V GCC 工具链可从 http://syntacore.com/page/products/sw-tools 下载。

1. 下载适用于您平台的压缩包
2. 解压到首选目录 `<GCC_INSTALL_PATH>`
3. 将 `<GCC_INSTALL_PATH>/bin` 文件夹添加到 $PATH 环境变量：

```bash
export PATH=<GCC_INSTALL_PATH>/bin:$PATH
```

##### 4.1.2.2. 从源码构建工具

您可以从官方仓库 https://github.com/riscv/riscv-gnu-toolchain 中的源码构建 RISC-V GCC 工具链。

构建和准备的说明参见 https://github.com/riscv/riscv-gnu-toolchain/blob/master/README.md

推荐使用 multilib 编译器。注意 RV32IC、RV32E、RV32EM、RV32EMC、RV32EC 架构配置默认不包含在编译器中。如果计划使用它们，需在构建前自行添加相应的库。

构建完成后，务必将 `<GCC_INSTALL_PATH>/bin` 文件夹添加到 $PATH 环境变量中。

#### 4.1.3. HDL 仿真器

当前支持的仿真器：

- Verilator（最后验证版本: v4.102）
- Intel ModelSim（最后验证版本: INTEL FPGA STARTER EDITION vsim 2020.1_3）
- Mentor Graphics ModelSim（最后验证版本: Modelsim PE Student Edition 10.4a）
- Synopsys VCS（最后验证版本: vcs-mx_vL-2016.06）
- Cadence NCSim

注意 RTL 仿真器可执行文件应在 $PATH 变量中。

#### 4.1.4. 测试准备

仿真包包含以下测试：

- hello —— "Hello" 示例程序
- isr_sample —— "中断服务例程"示例程序
- riscv_isa —— RISC-V ISA 测试（子模块）
- riscv_compliance —— RISC-V 合规性测试（子模块）
- dhrystone21 —— Dhrystone 2.1 基准测试
- coremark —— EEMBC CoreMark® 基准测试（子模块）

克隆主 SCR1 仓库后执行以下命令：

```bash
git submodule update --init --recursive
```

此命令将初始化包含测试源的子模块。

### 4.2. 运行仿真

要从仓库根目录构建 RTL、编译和运行测试，需要调用 Makefile。默认情况下，可以不带任何参数直接调用：

```bash
make
```

此时仿真将在 Verilator 上以以下参数运行：CFG=MAX BUS=AHB TRACE=0 TARGETS="hello isr_sample riscv_isa riscv_compliance dhrystone21 coremark"。

Makefile 支持：

- 仿真器选择 —— run_<SIMULATOR>
- 架构设置 —— CFG, BUS, ARCH, VECT_IRQ, IPIC, TCM
- 测试子集选择 —— TARGETS
- 启用跟踪日志 —— TRACE
- 以及传递给仿真器的任何附加选项 —— SIM_BUILD_OPTS

示例：

```bash
make run_vcs CFG=CUSTOM BUS=AXI ARCH=I VECT_IRQ=1 IPIC=1 TCM=0 TARGETS="hello isr_sample" TRACE=1 SIM_BUILD_OPTS="-gui"
```

构建和运行参数可以在 ./Makefile 中配置。

所有测试完成后，结果可以在 build/<SIM_CFG>/test_results.txt 中找到。

**重要**：为确保正确重新构建，请在仿真运行之间调用 `make clean`。

#### 4.2.1. 仿真器选择

您可以指定以下受支持的仿真器之一：run_<SIMULATOR> = <run_vcs, run_modelsim, run_ncsim, run_verilator, run_verilator_wf>：

```bash
make run_modelsim
```

仿真器运行选项：

- run_verilator —— Verilator（默认）
- run_verilator_wf —— 带波形生成的 Verilator
- run_modelsim —— Mentor Graphics 或 Intel 的 ModelSim
- run_vcs —— Synopsys VCS
- run_ncsim —— Cadence NCSim

对于 run_verilator_wf 选项，所有执行的测试都会生成波形并保存在 ./build/<SIM_CFG>/simx.vcd 中。该文件可以用 GTKWave 等波形查看器打开。

#### 4.2.2. 架构配置

您可以指定配置 CFG = <MAX, BASE, MIN, CUSTOM> 和外部接口 BUS = <AHB, AXI>：

```bash
make CFG=BASE BUS=AXI
```

配置展开如下：

- MAX —— 设置预定义配置 SCR1_CFG_RV32IMC_MAX（默认）
- BASE —— 设置预定义配置 SCR1_CFG_RV32IC_BASE
- MIN —— 设置预定义配置 SCR1_CFG_RV32EC_MIN
- CUSTOM —— 可用于其他自定义配置

对于所有预定义配置，其他架构参数会自动设置为确定状态，包括编译测试和 SCR1 RTL。

对于 CUSTOM 配置，可以指定附加参数：

- ARCH = <IMC, IC, IM, I, EMC, EM, EC, E> —— RISC-V 指令集架构。此参数定义用于编译测试的 RISC-V 指令集架构（由 RISC-V 工具链自动使用）：RV32I 或 RV32E 基础 + 可选标准扩展 M 和 C。RTL 选项 SCR1_RVE_EXT、SCR1_RVM_EXT 和 SCR1_RVC_EXT 必须相应定义。
- VECT_IRQ = <0, 1> —— 向量模式处理中断，否则使用直接模式。VECT_IRQ 参数在 "isr_sample" 测试中用于展示各种中断调用和处理场景。对于向量模式，必须定义 RTL 选项 SCR1_MTVEC_MODE_EN。
- IPIC = <0, 1> —— 使用集成可编程中断控制器。IPIC 参数在 "isr_sample" 测试中用于展示各种中断调用和处理场景。RTL 选项 SCR1_IPIC_EN 必须相应定义。
- TCM = <0, 1> —— 使用紧耦合存储器。设置 TCM 选项为 1 会将某些测试的执行位置定义为紧耦合存储器而非外部测试平台存储器。RTL 选项 SCR1_TCM_EN 必须相应定义。

**重要**：CUSTOM 配置的附加参数集不会启用 SCR1 RTL 参数。请不要忘记在 ./src/includes/scr1_arch_description.svh 文件中手动设置相应参数。

**注意**：附加参数不能用于预定义配置，因为它们是硬编码的。

示例：

```bash
make CFG=CUSTOM ARCH=I VECT_IRQ=1 IPIC=1 TCM=0
```

### 4.3. 测试目标

您可以指定要在仿真中运行的测试子集：

- TARGETS = <hello, isr_sample, riscv_isa, riscv_compliance, dhrystone21, coremark>

要仅选择列表中的一个目标，指定其名称即可，例如：

```bash
make TARGETS=hello
```

要选择多个目标，用引号括起以空格分隔，例如：

```bash
make TARGETS="dhrystone21 coremark"
```

某些测试依赖于所选架构，因此不能用于所有核心配置（这些会被自动跳过）。

要从集合中选择单个测试，需要：

- 对于 riscv_isa 集合，进入 ./sim/tests/riscv_isa/rv32_tests.inc 并在 rv32_isa_tests 中列出所需测试。
- 对于 riscv_compliance 集合，进入 ./sim/tests/riscv_compliance/Makefile 并在 compliance_set 中列出所需测试（指定每个测试的完整路径）。

### 4.4. 仿真代码

您可以添加关于仿真过程的有用信息：断言、跟踪日志和指令统计。必须在 ./src/includes/scr1_arch_description.svh 文件的"仿真选项"部分定义 SCR1_TRGT_SIMULATION 参数以启用所有仿真代码。运行 make 脚本时此参数会自动启用。

#### 4.4.1. 跟踪日志(Tracelog)

仿真过程中，以下信息可写入构建目录中的专用文件 tracelog_core_N.log：

- RTL_ID 值
- 核心复位事件及发生时间
- MPRF 和 CSR 寄存器更新信息，格式如下：

| 时间 | 事件 | 当前PC | 指令 | 下一PC | 寄存器 | 值 |

使用以下事件缩写：

- N —— 无事件（常规寄存器值更新或无更新的周期）
- E —— 异常
- I —— 中断
- W —— 唤醒

必须在 src/includes/scr1_arch_description.svh 中定义 SCR1_TRACE_LOG_EN 和 SCR1_TRGT_SIMULATION 参数以启用跟踪日志。使用 make 脚本时，可传入 TRACE=1 参数自动启用所有选中测试的跟踪日志生成。

### 4.5. 测试平台说明

SCR1 测试平台由顶层模块和外部存储器组成。根据所使用的存储器接口（AXI 或 AHB），有两种测试平台配置可用。

所需配置可通过 make 命令的 BUS=<AHB, AXI> 选项选择。默认为 BUS=AHB。

两种配置的文件列表如下表所示。

| 路径 | 描述 |
|------|------|
| **AXI 测试平台** | |
| tb/scr1_memory_tb_axi.sv | AXI 存储器测试平台 |
| tb/scr1_top_tb_axi.sv | SCR1 AXI 顶层测试平台 |
| **AHB 测试平台** | |
| tb/scr1_memory_tb_ahb.sv | AHB 存储器测试平台 |
| tb/scr1_top_tb_ahb.sv | SCR1 AHB 顶层测试平台 |

两种测试平台存储器大小均为 1024 kB。

**注意**：如果启用了 TCM，其存储器地址范围将从外部存储器地址范围中移除。

测试平台存储器提供了通过向特定地址写入数据来生成中断和在仿真控制台打印字符的机制。同时，加载仿真退出地址值到程序计数器会终止仿真。这些地址的定义位于 scr1_arch_description.svh 的"仿真选项"部分。默认地址映射如下：

| 地址 | 描述 |
|------|------|
| 0xF0000100 | 外部 IRQ |
| 0xF0000200 | 软件 IRQ |
| 0xF0000000 | 打印字符 |
| 0x000000F8 | 仿真退出 |

外部单引脚 IRQ 和 IPIC IRQ 线共享同一地址。请根据是否启用 IPIC 确保使用正确的宽度。外部单引脚 IRQ（IPIC 禁用时）和软件 IRQ 值应放在 bit 0 中。IRQ 线值（IPIC 启用时）使用写数据的低 16 位。

**注意**：如果需要使用默认地址映射以外的地址映射，请确保 RTL 和测试程序使用相同的映射。

## 5. SDK 信息

基于 FPGA 的开源 SDK 可在 https://github.com/syntacore/scr1-sdk 获取。

仓库包含：

- 针对多种标准 FPGA 开发板的预构建镜像和开源设计：
  - Digilent Arty (Xilinx)
  - Digilent Nexys 4 DDR (Xilinx)
  - Arria V GX Starter (Intel)
  - Terasic DE10-Lite (Intel)
- 软件包：
  - Bootloader
  - Zephyr RTOS
  - 测试和软件示例
- SDK 和工具的用户指南

## 6. 技术支持

如需更多 SCR1 核心信息，请发送邮件至 scr1@syntacore.com。
