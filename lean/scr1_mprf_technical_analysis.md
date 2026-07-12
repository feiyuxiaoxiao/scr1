# SCR1 多端口寄存器文件（MPRF）技术设计文档

> 对应源码: `src/core/pipeline/scr1_pipe_mprf.sv`
> 模块名: `scr1_pipe_mprf`
> 下游模块: EXU（执行单元）

---

## 1. 模块定位

MPRF 是 SCR1 处理器的通用寄存器文件，提供 **2 读 1 写**（2R1W）的多端口访问能力。它位于 IDU 译码之后、EXU 执行之前，为执行单元提供源操作数（rs1、rs2）并接收回写结果（rd）。

```
IDU ──rs1_addr, rs2_addr──→ MPRF ──rs1_data, rs2_data──→ EXU
EXU ──rd_addr, rd_data, w_req──→ MPRF
```

核心设计约束：
- x0 寄存器硬连线为零，不可写入
- 同周期读写冲突必须给出正确的写数据（写优先语义）
- 支持两种物理实现：RAM 块（FPGA 优化）与分布式逻辑（ASIC/通用）

---

## 2. x0 寄存器硬连线为零

### 2.1 存储层面

寄存器阵列的索引范围是 `[1 : SCR1_MPRF_SIZE-1]`，物理上不存在索引 0。x0 没有存储单元。

| 参数 | RV32I | RV32E |
|------|:---:|:---:|
| `SCR1_MPRF_SIZE` | 32 | 16 |
| `SCR1_MPRF_AWIDTH` | 5 | 4 |
| 阵列索引范围 | `[1:31]` | `[1:15]` |
| 物理寄存器数 | 31 | 15 |

### 2.2 读 x0：输出为 0

读地址为 0 时，通过地址有效性判定门控输出：

```systemverilog
assign rs1_addr_vd = |exu2mprf_rs1_addr_i;  // 全位按位或，地址 0 → 结果 0
```

- RAM 实现：`rs1_addr_vd_ff == 0` → `mprf2exu_rs1_data_o = '0`
- 分布逻辑实现：`rs1_addr_vd == 0` → `mprf2exu_rs1_data_o = '0`

### 2.3 写 x0：被抑制

```systemverilog
assign wr_req_vd = exu2mprf_w_req_i & |exu2mprf_rd_addr_i;
```

写地址为 0 时，`wr_req_vd = 0`，`always_ff` 块中的 `if (wr_req_vd)` 条件不满足，写入不发生。x0 始终保持为 0。

---

## 3. 实现方式一：RAM 实现（`SCR1_MPRF_RAM`）

### 3.1 触发条件

`SCR1_TRGT_FPGA_INTEL` 宏定义时自动使能 `SCR1_MPRF_RAM`。此模式下，综合工具将 `mprf_int` / `mprf_int2` 映射到 FPGA 的专用 RAM 块（M9K / M10K），同时强制关闭 `SCR1_NO_EXE_STAGE` 和 `SCR1_MPRF_RST_EN`。

### 3.2 双份 RAM 副本架构

```
                    ┌─────────────────────────────────────┐
                    │         RAM Implementation          │
                    │                                     │
  rs1_addr ────────▶│  ┌──────────────┐                   │
                    │  │   mprf_int    │──▶ rs1_data_ff ──┤──▶ MUX ──▶ rs1_data
  rd_addr ──┬──────▶│  │  (RAM copy 1) │                   │
            │       │  └──────────────┘                   │
            │       │         │ wr_data                    │
            │       │         ▼                            │
            │       │  ┌──────────────┐                   │
            │       │  │  mprf_int2   │──▶ rs2_data_ff ──┤──▶ MUX ──▶ rs2_data
  rs2_addr ─┴──────▶│  │  (RAM copy 2) │                   │
                    │  └──────────────┘                   │
                    │                                     │
                    │  冲突检测 ──▶ rd_data_ff（旁路）     │
                    └─────────────────────────────────────┘
```

使用两份 RAM 副本的原因：FPGA RAM 块是简单双端口（1 读 + 1 写），而 MPRF 需要支持 **2 读 + 1 写**（3 个同时操作）。用两份独立的 RAM 分别服务 rs1 和 rs2 的读端口，写操作同步写入两份副本。

### 3.3 同步读 + 一拍延迟

FPGA RAM 块的读取是同步的（地址在时钟沿采样，数据下一拍有效）。因此读路径经过一级 `_ff` 寄存器：

```systemverilog
always_ff @(posedge clk) begin
    rs1_data_ff <= mprf_int[exu2mprf_rs1_addr_i];   // 同步读，延迟一拍
    rs2_data_ff <= mprf_int2[exu2mprf_rs2_addr_i];
end
```

控制信号也同步延迟一拍（`rs1_addr_vd_ff` 等），与数据对齐。

### 3.4 写优先旁路（Write-First Bypass）

同步读引入了一拍延迟，导致同周期"写后读"冲突场景下，读端口拿到的是一拍前的旧数据。解决方案：

**检测冲突**（组合逻辑，当拍）：

```systemverilog
assign rs1_new_data_req = wr_req_vd & (exu2mprf_rs1_addr_i == exu2mprf_rd_addr_i);
assign rs2_new_data_req = wr_req_vd & (exu2mprf_rs2_addr_i == exu2mprf_rd_addr_i);
```

**保存新数据**（`rd_data_ff`，仅冲突时更新）：

```systemverilog
always_ff @(posedge clk) begin
    if (read_new_data_req) begin
        rd_data_ff <= exu2mprf_rd_data_i;
    end
end
```

**输出旁路**（下一拍，冲突信号也延迟一拍对齐）：

```systemverilog
assign mprf2exu_rs1_data_o = (rs1_new_data_req_ff) ? rd_data_ff
                            : (rs1_addr_vd_ff)      ? rs1_data_ff
                                                    : '0;
```

### 3.5 时序图：读-写冲突旁路

```
          T0            T1            T2            T3
clk    ───┐───────┐───────┐───────┐───────┐───────
          │       │       │       │       │
rs1_addr ─┤  x5   │  x5   │  x6   │       │
          │       │       │       │       │
rd_addr  ─┤  x5   │       │       │       │
          │       │       │       │       │
wr_data  ─┤ 0xAA  │       │       │       │
          │       │       │       │       │
wr_req   ─┤   1   │       │       │       │
          │       │       │       │       │
new_data ─┼───1───┼───0───┤       │       │  (rs1_new_data_req)
          │       │       │       │       │
new_ff   ─┼───────┼───1───┼───0───┤       │  (rs1_new_data_req_ff)
          │       │       │       │       │
rd_ff    ─┼───────┼─0xAA──┤       │       │  (rd_data_ff)
          │       │       │       │       │
rs1_data ─┼───────┼─0xAA──┤ old_6 │       │  (mprf2exu_rs1_data_o)
          │       │       │       │       │
```

- **T0**: rs1 读 x5，同时写 x5=0xAA。冲突检测置位，写数据打入 rd_data_ff
- **T1**: 冲突信号延迟一拍到达 `rs1_new_data_req_ff=1`，输出 MUX 选择 `rd_data_ff`（0xAA），旁路生效
- **T2**: 无冲突，正常输出从 RAM 读出的 x6 旧值

### 3.6 写操作

两份 RAM 副本同步写入，保持一致：

```systemverilog
always_ff @(posedge clk) begin
    if (wr_req_vd) begin
        mprf_int[exu2mprf_rd_addr_i]  <= exu2mprf_rd_data_i;
        mprf_int2[exu2mprf_rd_addr_i] <= exu2mprf_rd_data_i;
    end
end
```

### 3.7 FPGA RAM 风格属性

```systemverilog
`ifdef SCR1_TRGT_FPGA_INTEL_MAX10
(* ramstyle = "M9K" *)  logic [...] mprf_int  [1:SIZE-1];
`elsif SCR1_TRGT_FPGA_INTEL_ARRIAV
(* ramstyle = "M10K" *) logic [...] mprf_int  [1:SIZE-1];
`else
logic [...] mprf_int  [1:SIZE-1];   // 无属性，由综合器自动推断
`endif
```

### 3.8 RAM 实现要点总结

| 维度 | 特征 |
|------|------|
| 存储 | 两份独立 RAM 副本（mprf_int + mprf_int2） |
| 读时序 | 同步读，数据延迟 1 拍 |
| 冲突处理 | 组合逻辑冲突检测 + 延迟对齐的旁路 MUX |
| 写操作 | 同步写入两份副本 |
| 复位 | 不支持（`SCR1_MPRF_RST_EN` 被强制 `undef`） |
| 综合目标 | Intel FPGA M9K/M10K RAM 块 |
| 附加约束 | `SCR1_NO_EXE_STAGE` 被强制关闭（EXU 需要寄存器级来抵消 RAM 读延迟） |

---

## 4. 实现方式二：分布逻辑实现（`~SCR1_MPRF_RAM`）

### 4.1 架构

```
                    ┌─────────────────────────────────────┐
                    │    Distributed Logic Implementation  │
                    │                                     │
  rs1_addr ────────▶│  ┌──────────────────────────────┐   │
                    │  │  mprf_int[1:SIZE-1]           │───┼──▶ rs1_data
                    │  │  (Register Array, async read) │   │
  rs2_addr ────────▶│  │                               │───┼──▶ rs2_data
                    │  └──────────────────────────────┘   │
  rd_addr ─────────▶│         ▲ wr_data                   │
                    │         │ (sync write)              │
                    └─────────┼───────────────────────────┘
```

### 4.2 异步读（无延迟）

寄存器阵列支持组合逻辑读取，当拍即可获得数据：

```systemverilog
assign mprf2exu_rs1_data_o = (rs1_addr_vd) ? mprf_int[exu2mprf_rs1_addr_i] : '0;
assign mprf2exu_rs2_data_o = (rs2_addr_vd) ? mprf_int[exu2mprf_rs2_addr_i] : '0;
```

无延迟，不需要旁路逻辑。同周期写入的数据在同一周期即可被读端口看到（取决于综合后的时序路径）。

### 4.3 写操作（支持复位）

```systemverilog
`ifdef SCR1_MPRF_RST_EN
always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n) begin
        mprf_int <= '{default: '0};
    end else if (wr_req_vd) begin
        mprf_int[exu2mprf_rd_addr_i] <= exu2mprf_rd_data_i;
    end
end
`else
always_ff @(posedge clk) begin
    if (wr_req_vd) begin
        mprf_int[exu2mprf_rd_addr_i] <= exu2mprf_rd_data_i;
    end
end
`endif
```

### 4.4 分布逻辑实现要点总结

| 维度 | 特征 |
|------|------|
| 存储 | 单份寄存器阵列 (`type_scr1_mprf_v`) |
| 读时序 | 异步读，当拍有效 |
| 冲突处理 | 无需旁路——同一周期的写入在同一拍组合路径上可见 |
| 写操作 | 同步写，可配置异步复位 |
| 资源 | 综合为 LUT + FF（分布式逻辑） |
| 功耗 | 读操作功耗与地址变化相关（组合路径翻转） |

---

## 5. 两种实现方式对比

| 维度 | RAM 实现 (`SCR1_MPRF_RAM`) | 分布逻辑实现 (`~SCR1_MPRF_RAM`) |
|------|------|------|
| **存储资源** | FPGA 专用 RAM 块（M9K/M10K） | 通用 LUT + FF |
| **RAM 副本数** | 2 份（mprf_int + mprf_int2） | 1 份（mprf_int） |
| **读时序** | 同步读，延迟 1 拍 | 异步读，0 延迟 |
| **冲突处理** | 冲突检测 + rd_data_ff 旁路 | 无需处理（组合同拍可见） |
| **复位支持** | 不支持 | 可选支持（`SCR1_MPRF_RST_EN`） |
| **控制信号** | 多 6 个 `_ff` 寄存器 + 3 个冲突检测信号 | 无额外寄存器 |
| **EXU 约束** | 强制 `~SCR1_NO_EXE_STAGE`（需要 EXU 寄存器抵消读延迟） | 无约束 |
| **目标平台** | Intel FPGA | ASIC、非 Intel FPGA、仿真 |
| **面积** | 小（RAM 块密度高） | 大（LUT 实现寄存器） |
| **功耗** | 低（RAM 块优化） | 较高（组合路径翻转） |

---

## 6. 读写冲突详析（Write-After-Read Hazard）

### 6.1 RAM 实现冲突场景

当同一周期内 rs1 或 rs2 的读地址与 rd 的写地址相同，RAM 的同步读在一拍后才能返回数据。若不处理，读端口将拿到写入前的旧值。

冲突检测为纯组合逻辑（当拍），结果延迟一拍后作为旁路选择信号：

```
T0: 冲突检测（组合） → rs1_new_data_req = 1
    写数据打入 rd_data_ff
T1: rs1_new_data_req_ff = 1 → 输出 MUX 选择 rd_data_ff
```

**非冲突情况优化**：`rd_data_ff` 仅在 `read_new_data_req == 1` 时更新。无冲突时它保持上次数值，但这不影响正确性——因为 `rs*_new_data_req_ff == 0` 时 MUX 不会选择 `rd_data_ff`。

### 6.2 分布逻辑实现冲突场景

分布逻辑实现中，读取是异步组合路径：

```systemverilog
assign mprf2exu_rs1_data_o = mprf_int[exu2mprf_rs1_addr_i];
```

同周期写入的值通过 `always_ff` 在时钟沿更新 `mprf_int`。异步读路径在时钟沿之前读取的是旧值，在时钟沿之后读取的是新值。实际行为取决于综合后的时序——通常 RISC-V 处理器设计中 ID 阶段在时钟上升沿之前完成读操作，同一指令的 EX 阶段回写发生在下一个时钟周期，因此分布逻辑实现中不存在同一指令的自冲突。

### 6.3 冲突类型总结

| 冲突类型 | RAM 实现 | 分布逻辑实现 |
|----------|----------|------|
| 同一指令 rs1/rs2 == rd（RAW） | 旁路处理 | 指令间自然无冲突（ID 读在 EX 写之前） |
| 前一条指令写，后一条读（RAW） | 同步写后 RAM 内容已更新，读正常 | 同上 |
| 同一周期两条指令（双发射） | SCR1 为单发射顺序核，不存在此场景 | — |

---

## 7. 复位行为

### 7.1 分布逻辑 + 复位使能

```systemverilog
always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n) begin
        mprf_int <= '{default: '0};   // 所有寄存器清零
    end
end
```

x0 本身没有物理存储，因此复位仅清零 x1~x31（或 x1~x15）。

### 7.2 RAM 实现

`SCR1_MPRF_RAM` 强制 `undef SCR1_MPRF_RST_EN`，不生成复位逻辑。原因：
- FPGA RAM 块通常不支持复位（或复位需要额外资源）
- RISC-V ABI 不要求寄存器文件上电清零——软件应在使用前初始化

---

## 8. 验证断言

仅 `SCR1_TRGT_SIMULATION` + `SCR1_MPRF_RST_EN` 时使能：

```systemverilog
SCR1_SVA_MPRF_WRITEX : assert property (
    @(negedge clk) disable iff (~rst_n)
    exu2mprf_w_req_i |-> !$isunknown({exu2mprf_rd_addr_i,
        (|exu2mprf_rd_addr_i ? exu2mprf_rd_data_i : `SCR1_XLEN'd0)})
) else $error("MPRF error: unknown values");
```

检查当写请求有效时，写地址和写数据（仅地址非零时检查数据）不能包含 X 态。采样边沿为 `negedge clk`，在半周期点检查信号稳定性。

---

## 9. 关键设计决策

### 9.1 两份 RAM 副本 vs 多端口 RAM

FPGA 的块 RAM 通常只支持简单双端口（1R1W）。要实现 2R1W，选择用两份 RAM 副本比例化真双端口 RAM（如果 FPGA 支持）更通用——两份简单双端口 RAM 在任何 FPGA 上均可综合。

代价是写操作需要同时更新两份副本，写功耗翻倍，但面积仅增加到约 2 倍（而非 3 倍）。

### 9.2 同步读 + 旁路 vs 异步读

RAM 实现选择同步读是为了利用 FPGA RAM 块的输出寄存器，提高时序性能。代价是需要旁路逻辑处理冲突。分布逻辑实现用异步读避免了旁路复杂度。

### 9.3 阵列从 1 开始索引

x0 不分配存储空间，节省了 1/32（或 1/16）的存储资源。门控逻辑仅需在输出端加一个 MUX 选择（地址有效 → 阵列数据，地址无效 → 0）。
