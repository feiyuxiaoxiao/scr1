# SCR1 MPRF 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_mprf.sv`
> 模块: `scr1_pipe_mprf` — Multi Port Register File

---

## 1. 外部宏定义（编译时参数）

### 1.1 架构宏 — `scr1_arch_description.svh`

| 宏名 | 作用 | 定义条件 |
|------|------|------|
| `SCR1_MPRF_RST_EN` | 使能 MPRF 复位逻辑 | 推荐配置（`RV32IMC_MAX`、`RV32IC_BASE`）默认使能；自定义配置中可选开启；RAM 实现时被强制 `undef` |
| `SCR1_MPRF_RAM` | 使用专用 RAM 块实现 MPRF（双份同步读副本） | `SCR1_TRGT_FPGA_INTEL` 定义时自动使能；Intel FPGA 专用 |
| `SCR1_NO_EXE_STAGE` | 无独立执行阶段寄存器（RAM 实现时被强制 `undef`） | RAM 实现时禁止 |
| `SCR1_TRGT_FPGA_INTEL_MAX10` | Intel MAX10 FPGA 目标 | 影响 RAM 风格的 `ramstyle` 属性为 `"M9K"` |
| `SCR1_TRGT_FPGA_INTEL_ARRIAV` | Intel Arria V FPGA 目标 | 影响 RAM 风格的 `ramstyle` 属性为 `"M10K"` |
| `SCR1_TRGT_SIMULATION` | 仿真模式 | 使能 SVA 断言 |
| `SCR1_TRGT_FPGA_INTEL` | Intel FPGA 通用目标宏 | 触发 `SCR1_MPRF_RAM` 定义 |

### 1.2 类型宏 — `scr1_arch_types.svh`

| 宏/类型名 | 值 | 含义 |
|-----------|:---:|------|
| `SCR1_MPRF_SIZE` | 32（RV32I）或 16（RV32E） | 寄存器文件物理条目数（不含 x0） |
| `SCR1_MPRF_AWIDTH` | 5（RV32I）或 4（RV32E） | 寄存器地址位宽 |
| `type_scr1_mprf_v` | `logic [XLEN-1:0]` | MPRF 单个条目的数据类型（32 或 64 位） |

### 1.3 宏依赖关系

```
SCR1_TRGT_FPGA_INTEL
    └── SCR1_MPRF_RAM ────────┐
                              ├── undef SCR1_NO_EXE_STAGE
                              └── undef SCR1_MPRF_RST_EN
```

---

## 2. 类型声明

### 2.1 分布逻辑实现中的寄存器阵列类型

| 类型 | 含义 | 索引范围 |
|------|------|:---:|
| `type_scr1_mprf_v [1:SCR1_MPRF_SIZE-1]` | 寄存器数组，每个元素位宽 = `SCR1_XLEN`，从索引 1 开始（x0 不使用物理存储） | 1 ~ 31（RV32I）或 1 ~ 15（RV32E） |

### 2.2 RAM 实现中的存储器阵列

| 类型 | 含义 | 索引范围 |
|------|------|:---:|
| `logic [XLEN-1:0] [1:SCR1_MPRF_SIZE-1]` | 双端口 RAM 风格存储器，可选 `ramstyle` 属性 | 1 ~ 31（或 1 ~ 15） |

---

## 3. 模块端口 (I/O)

### 3.1 通用控制信号

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `rst_n` | 1 | 异步复位，低有效（仅 `SCR1_MPRF_RST_EN` 定义时存在） |
| input | `clk` | 1 | 时钟 |

### 3.2 EXU ↔ MPRF 接口 — 读端口 1 (rs1)

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `exu2mprf_rs1_addr_i` | `SCR1_MPRF_AWIDTH` | rs1 读地址。为 0 时表示"读 x0"，输出强制为 0 |
| output | `mprf2exu_rs1_data_o` | `SCR1_XLEN` | rs1 读数据。读 x0 时为 0，地址无效（全零地址）时为 0 |

### 3.3 EXU ↔ MPRF 接口 — 读端口 2 (rs2)

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `exu2mprf_rs2_addr_i` | `SCR1_MPRF_AWIDTH` | rs2 读地址。为 0 时表示"读 x0"，输出强制为 0 |
| output | `mprf2exu_rs2_data_o` | `SCR1_XLEN` | rs2 读数据。读 x0 时为 0，地址无效时为 0 |

### 3.4 EXU ↔ MPRF 接口 — 写端口 (rd)

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `exu2mprf_w_req_i` | 1 | 写请求 |
| input | `exu2mprf_rd_addr_i` | `SCR1_MPRF_AWIDTH` | 写目标寄存器地址。为 0 时写被抑制（写 x0 无效果） |
| input | `exu2mprf_rd_data_i` | `SCR1_XLEN` | 写数据 |

---

## 4. 内部信号 — 通用控制（两种实现共用）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `rs1_addr_vd` | wire | 1 | rs1 地址有效：全位按位或结果。`exu2mprf_rs1_addr_i != 0` 时为 1 |
| `rs2_addr_vd` | wire | 1 | rs2 地址有效：`exu2mprf_rs2_addr_i != 0` 时为 1 |
| `wr_req_vd` | wire | 1 | 写请求有效：写使能 & 写地址非零（不写 x0） |

---

## 5. 内部信号 — RAM 实现专用（`SCR1_MPRF_RAM`）

### 5.1 读写冲突检测

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `rs1_new_data_req` | wire | 1 | rs1 读地址与 rd 写地址冲突（同寄存器、同周期读写） |
| `rs2_new_data_req` | wire | 1 | rs2 读地址与 rd 写地址冲突 |
| `read_new_data_req` | wire | 1 | 存在任意读端口与写端口地址冲突 |

### 5.2 延迟一拍的控制/数据寄存器（`_ff` 后缀）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `rs1_addr_vd_ff` | reg | 1 | 上一周期 rs1 地址有效标志 |
| `rs2_addr_vd_ff` | reg | 1 | 上一周期 rs2 地址有效标志 |
| `rs1_new_data_req_ff` | reg | 1 | 上一周期 rs1 读写冲突标志 |
| `rs2_new_data_req_ff` | reg | 1 | 上一周期 rs2 读写冲突标志 |
| `rs1_data_ff` | reg | `SCR1_XLEN` | 上一周期 rs1 同步读出的数据（RAM 读延迟一拍） |
| `rs2_data_ff` | reg | `SCR1_XLEN` | 上一周期 rs2 同步读出的数据 |
| `rd_data_ff` | reg | `SCR1_XLEN` | 上一周期写入的数据（旁路用，仅在冲突时更新） |

### 5.3 存储器阵列

| 信号名 | 类型 | 位宽 × 深度 | 含义 |
|--------|------|------|------|
| `mprf_int` | reg array | `XLEN × (SIZE-1)` | 第一份 RAM 副本，用于 rs1 同步读 + 写 |
| `mprf_int2` | reg array | `XLEN × (SIZE-1)` | 第二份 RAM 副本，用于 rs2 同步读，写入时与 mprf_int 同步更新 |

---

## 6. 内部信号 — 分布逻辑实现专用（`~SCR1_MPRF_RAM`）

| 信号名 | 类型 | 位宽 × 深度 | 含义 |
|--------|------|------|------|
| `mprf_int` | reg array (`type_scr1_mprf_v`) | `XLEN × (SIZE-1)` | 单份寄存器阵列，异步读，同步写。索引从 1 开始，x0 无物理存储。复位使能时初始化为全 0 |

---

## 7. 外部依赖头文件

| 文件名 | 提供内容 |
|--------|----------|
| `scr1_arch_description.svh` | `SCR1_MPRF_RST_EN`、`SCR1_MPRF_RAM`、`SCR1_NO_EXE_STAGE`、`SCR1_XLEN`、FPGA 目标宏 |
| `scr1_arch_types.svh` | `SCR1_MPRF_SIZE`、`SCR1_MPRF_AWIDTH`、`type_scr1_mprf_v` |

---

## 8. x0 寄存器为零的实现机制

阵列索引范围为 `[1 : SCR1_MPRF_SIZE-1]`（不含索引 0），物理上不存在 x0 的存储位置。

读 x0 的逻辑通过两级门控实现：

1. **地址有效性判定**：`rs1_addr_vd = |exu2mprf_rs1_addr_i`（地址全位或归约），地址为 0 时 `rs1_addr_vd = 0`
2. **输出多路选择**：
   - RAM 实现：`rs1_addr_vd_ff == 0` 时 `mprf2exu_rs1_data_o = '0`
   - 分布逻辑实现：`rs1_addr_vd == 0` 时 `mprf2exu_rs1_data_o = '0`

写 x0 的逻辑通过 `wr_req_vd = exu2mprf_w_req_i & |exu2mprf_rd_addr_i` 抑制——当 `rd_addr_i == 0` 时，`wr_req_vd = 0`，写操作被门控，不会修改任何存储器内容。
