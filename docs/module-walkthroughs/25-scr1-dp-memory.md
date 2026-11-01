# scr1_dp_memory.sv 讲解

## 一句话职责

双端口 RAM：Intel FPGA 字节数组 vs 通用字数组两种实现。

## 文件信息

- **路径**: `src/top/scr1_dp_memory.sv`
- **行数**: 111 行
- **配置依赖**: `SCR1_TRGT_INTEL`

## 两套实现

### 1. Intel FPGA（`:30-66`）

```systemverilog
`ifdef SCR1_TRGT_INTEL
// 字节数组 + ramstyle 推断
logic [7:0] mem_array [0:MEM_SIZE/8-1];
(* ramstyle = "M9K" *)
```

**特点**：
- 字节粒度（方便字节使能）
- 推断成 M9K/M10K 块
- 需要 `ramstyle` 属性

### 2. 通用实现（`:68-107`）

```systemverilog
`else
// 字数组
logic [31:0] mem_array [0:MEM_SIZE/4-1];
```

**特点**：
- 字粒度（32 位）
- 综合成标准单元寄存器或 RAM
- 无需特殊属性

## 端口定义

### Port A（只读，`:85-89`）

```systemverilog
always_ff @(posedge clk_a) begin
    if (ren_a) data_a <= mem_array[addr_a];
end
```

**用途**：LSU 读端口（或指令读）。

### Port B（读写，字节使能，`:94-105`）

```systemverilog
always_ff @(posedge clk_b) begin
    if (we_b) begin
        if (byte_en_b[0]) mem_array[addr_b][7:0]   <= wdata_b[7:0];
        if (byte_en_b[1]) mem_array[addr_b][15:8]  <= wdata_b[15:8];
        if (byte_en_b[2]) mem_array[addr_b][23:16] <= wdata_b[23:16];
        if (byte_en_b[3]) mem_array[addr_b][31:24] <= wdata_b[31:24];
    end
end
```

**用途**：LSU 写端口（支持字节/半字/字访问）。

## 无写旁路

```systemverilog
// 同周期写 rd、读 rs=rd → 读到旧值
// 需要上层模块（如 MPRF）自己处理旁路
```

**为什么没有？** 简化 RAM 设计，旁路逻辑交给使用者（如 `scr1_pipe_mprf.sv` 的 `rs1_new_data_req` 旁路）。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `clk_a` | in | Port A 时钟 |
| `addr_a` | in | Port A 读地址 |
| `ren_a` | in | Port A 读使能 |
| `data_a` | out | Port A 读数据 |
| `clk_b` | in | Port B 时钟 |
| `addr_b` | in | Port B 地址 |
| `we_b` | in | Port B 写使能 |
| `byte_en_b` | in | Port B 字节使能 |
| `wdata_b` | in | Port B 写数据 |
| `data_b` | out | Port B 读数据 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TRGT_INTEL` | Intel FPGA 实现（字节数组 + M9K） |
| 未定义 | 通用实现（字数组） |

## 与其他模块关系

```
MPRF → DP Memory: Port A/B 地址 + Port B 写
DP Memory → MPRF: Port A/B 读数据
```

## 配套文档

- 设计规格：`docs/scr1-dp-memory/design-spec.md`
- 验证文档：`docs/scr1-dp-memory/verification.md`
