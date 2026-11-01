# scr1_dmem_ahb.sv 讲解

## 一句话职责

数据 AHB 桥：FIFO + data_fifo + 三转换函数，支持读/写/字节路对齐。

## 文件信息

- **路径**: `src/top/scr1_dmem_ahb.sv`
- **行数**: 480 行
- **配置依赖**: `SCR1_DMEM_AHB_OUT_BP`、`SCR1_DMEM_AHB_IN_BP`

## 核心架构

### 1. 请求 FIFO（`:59-64`）

```systemverilog
typedef struct packed {
    logic hwrite;                    // 读还是写
    logic [2:0] hwidth;              // HSIZE
    logic [ADDR_WIDTH-1:0] haddr;
    logic [ADDR_WIDTH-1:0] hwdata;   // 预对齐后的写数据
} type_scr1_req_fifo_s;
```

**对比 imem_ahb**：数据口需要存 `hwrite`/`hwidth`/`hwdata`，因为 AHB 写操作的数据在地址相位之后才出现。

### 2. Data FIFO（`:219`、`:388-411`）

**为什么需要？**

AHB 写操作时序：
```
ADDR 相位：发 haddr → 从设备采样
DATA 相位：发 hwdata → 从设备采样（晚一拍或多拍）
```

核的写请求是"地址 + 数据同时"，但 AHB 是"先地址后数据"。所以得把写数据先扣住，等 AHB 数据相位再发：

```systemverilog
always_ff @(posedge clk) begin
    case (fsm)
        ADDR: if (~req_fifo_empty) data_fifo <= req_fifo[0];  // :391-395
        DATA: if (hready && hresp==OK && ~req_fifo_empty) data_fifo <= req_fifo[0];
    endcase
end

assign hwdata = data_fifo.hwdata;  // :478
```

### 3. 三转换函数

#### `scr1_conv_mem2ahb_width`（`:82-103`）

```systemverilog
case (dmem_width)
    BYTE:  tmp = SCR1_HSIZE_8B;
    HWORD: tmp = SCR1_HSIZE_16B;
    WORD:  tmp = SCR1_HSIZE_32B;
endcase
```

#### `scr1_conv_mem2ahb_wdata`（`:105-156`）—— 字节路对齐

**问题**：核的写数据总在低位（如 `wdata[7:0]`），但 AHB 要求数据落在**实际地址对应的字节路**上。

**解决**：
```systemverilog
case (dmem_addr[1:0])
    2'b00: tmp[7:0]   = dmem_wdata[7:0];   // 对齐
    2'b01: tmp[15:8]  = dmem_wdata[7:0];   // 偏 1 字节
    2'b10: tmp[23:16] = dmem_wdata[7:0];   // 偏 2 字节
    2'b11: tmp[31:24] = dmem_wdata[7:0];   // 偏 3 字节
endcase
```

HWORD 类似（`:137-143`），WORD 原样（`:149`）。

#### `scr1_conv_ahb2mem_rdata`（`:158-197`）—— 反向抽取

读回时也要从字节路抽回低位：

```systemverilog
case (hwidth)
    HSIZE_8B: case (haddr[1:0])
        00: tmp[7:0] = hrdata[7:0];
        01: tmp[7:0] = hrdata[15:8];
        ...
    endcase
    HSIZE_32B: tmp = hrdata;  // 原样
endcase
```

**关键**：用 `resp_fifo.hwidth`/`haddr`（`:229`）——响应相位必须记住当初请求的宽度和地址。

### 4. FSM（`:338-383`）

与 imem_ahb 相同（ADDR/DATA 两态），但 `req_fifo_rd` 逻辑多了 `hresp` 检查（`:366-383`）：

```systemverilog
case (fsm)
    ADDR: if (hready) req_fifo_rd = ~req_fifo_empty;
    DATA: if (hready) req_fifo_rd = ~req_fifo_empty & (hresp==OK);
endcase
```

**为什么？** 写操作 `hresp=ER` 时不能 pop FIFO（要重发）。

### 5. AHB 输出（`:444-478`）

```systemverilog
assign hprot[HPROT_DATA]  = 1'b1;   // 数据口（与 imem 的 0 不同）
assign hburst = SCR1_HBURST_SINGLE;
assign hsize  = req_fifo[0].hwidth;  // 动态宽度
assign htrans = (fsm==ADDR && ~req_fifo_empty) ? NONSEQ : IDLE;
assign haddr  = req_fifo[0].haddr;
assign hwrite = req_fifo[0].hwrite;
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `dmem_req_i` | in | 核访存请求 |
| `dmem_cmd_i` | in | RD/WR |
| `dmem_width_i` | in | BYTE/HWORD/WORD |
| `dmem_addr_i` | in | 访存地址 |
| `dmem_wdata_i` | in | 写数据 |
| `dmem_req_ack_o` | out | 请求确认 |
| `dmem_rdata_o` | out | 读回数据 |
| `dmem_resp_o` | out | 响应（NOTRDY/RDY_OK/RDY_ER） |

## 与 imem_ahb 对比

| 特性 | imem_ahb | dmem_ahb |
|---|---|---|
| FIFO 内容 | 地址 | 地址 + 写数据 + 宽度 |
| Data FIFO | 无 | 有（存 hwdata） |
| hprot[DATA] | 0（指令口） | 1（数据口） |
| hsize | 固定 32B | 动态（BYTE/HWORD/WORD） |
| 转换函数 | 无 | 3 个（宽度/对齐/抽取） |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DMEM_AHB_OUT_BP` | 请求 FIFO bypass |
| `SCR1_DMEM_AHB_IN_BP` | 响应 FIFO bypass |

## 配套文档

- 设计规格：`docs/scr1-dmem-ahb/design-spec.md`
- 验证文档：`docs/scr1-dmem-ahb/verification.md`
