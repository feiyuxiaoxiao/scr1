# SCR1 双端口存储器设计规格（Design Specification）

- 模块：`scr1_dp_memory`（`src/top/scr1_dp_memory.sv`，111 行；`SCR1_TCM_EN` 保护 `:8`/`:111`）
- 职责：带字节使能的双端口同步存储器，作为 TCM 的存储阵列

---

## 1. 概述与职责

- **端口 A**：只读，`rena`/`addra`/`qa`（同步读，寄存输出）；
- **端口 B**：读写，`renb`/`wenb`/`webb`/`addrb`/`datab`/`qb`；
- 提供两种实现：Intel FPGA（M9K/M10K）与通用（Xilinx block / ASIC / 仿真）。

---

## 2. 参数与端口

### 2.1 参数（`:9-14`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `SCR1_WIDTH` | 32 | 数据位宽 |
| `SCR1_SIZE` | `` `SCR1_IMEM_AWIDTH'h00010000 `` | 字节容量 |
| `SCR1_NBYTES` | `SCR1_WIDTH/8` | 字节数（4） |

### 2.2 端口（`:15-28`）

`clk`；端口 A（`:17-20`）；端口 B（`:21-27`）。地址位宽 `[$clog2(SIZE)-1:2]`
（按字寻址，`:19`/`:25`）。

---

## 3. Intel FPGA 实现（`:30-66`）

- 存储体按字节数组声明：`memory_array[0:SIZE/NBYTES-1]`（`:35`/`:37`），
  `SCR1_TRGT_FPGA_INTEL_MAX10` 用 `M9K`、`SCR1_TRGT_FPGA_INTEL_ARRIAV` 用 `M10K`；
- `wenbb = {4{wenb}} & webb`（`:43`），四字节各自条件写（`:44-58`）；
- `qb <= memory_array[addrb]`（`:59`）；`qa <= memory_array[addra]`（`:64-66`）。

---

## 4. 通用实现（`:68-107`）

- `RAM_SIZE_WORDS = SCR1_SIZE/SCR1_NBYTES`（`:72`）；
- Xilinx 平台标注 `ram_style="block"`（`:77-78`）；
- 端口 A 同步读（`:85-89`）；
- 端口 B：按 `webb[i]` 逐字节写，再同步读（`:94-105`）。

---

## 5. 附录

### 5.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_TCM_EN` | 模块是否存在 | `:8`/`:111` |
| `SCR1_TRGT_FPGA_INTEL` | 选择 Intel 实现 | `:30`/`:68` |
| `SCR1_TRGT_FPGA_INTEL_MAX10` / `_ARRIAV` | RAM 风格 M9K/M10K | `:34`/`:36` |
| `SCR1_TRGT_FPGA_XILINX` | block RAM 标注 | `:77` |

### 5.2 职责边界

- 纯存储阵列，无地址译码/协议逻辑；
- 写入优先：端口 B 同拍读写时输出为写后值语义由工具实现决定（无 bypass 声明）；
- 无 SVA。
