# scr1_pipe_mprf.sv 讲解

## 一句话职责

多端口寄存器堆：32 个通用寄存器（x0~x31），2 读 1 写，x0 恒为 0。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_mprf.sv`
- **行数**: 167 行
- **配置依赖**: `SCR1_RVE_EXT`、`SCR1_MPRF_RAM`

## 核心设计

### 1. 2 读 1 写三端口

指令最多同时读 rs1、rs2，写 rd，所以需要三端口：
- 读端口 A：`rs1_addr` → `rs1_data`
- 读端口 B：`rs2_addr` → `rs2_data`
- 写端口：`rd_addr` + `rd_data` + `wr_req`

### 2. x0 恒为 0 的硬件实现（`:71-74`）

```systemverilog
rs1_addr_vd = |exu2mprf_rs1_addr_i;           // 地址非零才算有效读
wr_req_vd   = exu2mprf_w_req_i & |exu2mprf_rd_addr_i;  // 写 x0 直接忽略
```

- 读 x0：地址 0 → `rs1_addr_vd=0` → 输出 `32'b0`
- 写 x0：地址 0 → `wr_req_vd=0` → 写入被丢弃

**为什么有效？** RISC-V 规范规定 x0 读恒为 0、写无效。硬件直接无视地址 0 的访问，天然满足规范。

### 3. 两套实现

#### 分布式逻辑实现（`:127-152`，默认）

```systemverilog
always_ff @(posedge clk) begin
    if (wr_req_vd) regs[rd_addr] <= rd_data;
end
assign rs1_data = rs1_addr_vd ? regs[rs1_addr] : '0;
assign rs2_data = rs2_addr_vd ? regs[rs2_addr] : '0;
```

**特点**：
- 触发器构成
- **异步读**（组合逻辑输出）
- 代码直白，综合成标准单元寄存器

#### RAM 实现（`:90-126`，Intel FPGA）

```systemverilog
// 两块 M9K/M10K 片上 RAM
intel_ram_portA: entity work.scr1_dp_memory
    .addr(rs1_addr_ff), .data(rs1_data_ff);  // 只读
intel_ram_portB: entity work.scr1_dp_memory
    .addr(rs2_addr_ff), .data(rd_data), .we(wr_req_vd);  // 读写
```

**特点**：
- 片上 RAM 块
- **同步读**（地址打一拍出数据）
- 需要 EXU 侧配合打拍

### 4. RAM 实现的"写读冲突"旁路（重点，`:78-79`）

同步读意味着同周期写 rd、读 rs1=rd 读到旧值。SCR1 检测"读地址==写地址"后前向旁路：

```systemverilog
mprf2exu_rs1_data_o = (rs1_new_data_req_ff) ? rd_data_ff   // 冲突时旁路写数据
                    : (rs1_addr_vd_ff)      ? rs1_data_ff   // 正常同步读
                    : '0;
```

**为什么需要？**

考虑 `add x1, x1, x2`：
- 周期 N：EXU 算出结果，写 x1
- 周期 N+1：新指令读 x1 作为 rs1

如果 RAM 同步读，周期 N 的写操作要周期 N+1 才读出，新指令读到旧值。旁路逻辑检测到 `rs1_addr == rd_addr`，直接取 `rd_data_ff`（刚写入的值），避免 stall。

### 5. 为什么 RAM 实现要两块内存

一个周期内可能 2 读 + 1 写 = 3 个同时独立操作，单端口 RAM 做不到：
- 用两块简单双口 RAM：各服务一个读端口
- 写端口同时写两块（广播）

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `exu2mprf_rs1_addr_i` | in | 读 rs1 寄存器号 |
| `exu2mprf_rs2_addr_i` | in | 读 rs2 寄存器号 |
| `exu2mprf_w_req_i` | in | 写请求 |
| `exu2mprf_rd_addr_i` | in | 写回寄存器号 |
| `exu2mprf_rd_data_i` | in | 写回数据 |
| `mprf2exu_rs1_data_o` | out | rs1 寄存器值 |
| `mprf2exu_rs2_data_o` | out | rs2 寄存器值 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_RVE_EXT` | 只实现 16 个寄存器（x0~x15），rs>=16 读返回 0 |
| `SCR1_MPRF_RAM` | RAM 实现（同步读 + 旁路）vs 分布式（异步读） |
| `SCR1_MPRF_RST_EN` | 复位时清零寄存器（默认不复位，节省面积） |

## 与其他模块关系

```
EXU → MPRF: rs1/rs2 地址 + rd 地址/数据/写请求
MPRF → EXU: rs1/rs2 数据
```

## 配套文档

- 设计规格：`docs/scr1-pipe-mprf/design-spec.md`
- 验证文档：`docs/scr1-pipe-mprf/verification.md`
