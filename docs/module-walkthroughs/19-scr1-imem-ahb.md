# scr1_imem_ahb.sv 讲解

## 一句话职责

指令 AHB 桥：2 深 FIFO + 两态 FSM，把内部取指请求翻译成 AHB 读事务。

## 文件信息

- **路径**: `src/top/scr1_imem_ahb.sv`
- **行数**: 319 行
- **配置依赖**: `SCR1_IMEM_AHB_OUT_BP`、`SCR1_IMEM_AHB_IN_BP`

## 核心架构

### 1. 请求 FIFO（`:67-174`）

**默认深度 2**（`:38`）：
```systemverilog
localparam SCR1_FIFO_WIDTH = 2;
localparam SCR1_FIFO_CNT_WIDTH = $clog2(SCR1_FIFO_WIDTH+1);
```

**Bypass 模式**（`:99-120`，`SCR1_IMEM_AHB_OUT_BP=1`）：
```systemverilog
// 退化成 1 深（组合直通）
assign req_fifo_empty = ~(req_fifo_full | imem_req);
assign req_fifo_full = imem_req & ~req_fifo_rd;
```

**背压**（`:85`）：
```systemverilog
assign imem_req_ack = ~req_fifo_full;  // FIFO 满了就压住核
```

### 2. 两态 FSM（`:179-222`）

| 状态 | 含义 | 转移条件 |
|---|---|---|
| `ADDR` | 地址相位 | `hready` 有效且有请求 → `DATA` |
| `DATA` | 数据相位 | `hready` 有效且 `hresp=OK` → 有请求留 `DATA`，否则回 `ADDR` |

```systemverilog
case (fsm)
    SCR1_FSM_ADDR: if (hready && !req_fifo_empty) fsm = DATA;
    SCR1_FSM_DATA: if (hready) fsm = (hresp==OK && !req_fifo_empty) ? DATA : ADDR;
endcase
```

### 3. 响应 FIFO（1 深，`:227-246`）

**默认打一拍**（`:237-246`）：
```systemverilog
always_ff @(posedge clk) begin
    if (hready && fsm==DATA) begin
        resp_fifo.hresp  <= hresp;
        resp_fifo.hrdata <= hrdata;
    end
end
```

**Bypass 模式**（`SCR1_IMEM_AHB_IN_BP=1`）：
```systemverilog
assign resp_fifo_hready = (fsm==DATA) ? hready : 1'b0;  // 组合直通
assign resp_fifo.hresp  = hresp;
assign resp_fifo.hrdata = hrdata;
```

### 4. AHB 输出常量（`:251-281`）

```systemverilog
assign hprot[HPROT_DATA]  = 1'b0;   // 指令口（数据口=1）
assign hprot[HPROT_PRV]   = 1'b0;   // 非特权
assign hprot[HPROT_BUF]   = 1'b0;   // 非缓冲
assign hprot[HPROT_CACHE] = 1'b0;   // 非缓存

assign hburst = SCR1_HBURST_SINGLE;  // 单拍突发
assign hsize  = SCR1_HSIZE_32B;      // 32 位宽度
assign hmastlock = 1'b0;             // 无锁
```

**关键点**：`hprot[DATA]=0` 告诉从设备"这是取指，不是数据访问"。

### 5. HTRANS 控制（`:260-278`）

```systemverilog
case (fsm)
    ADDR: htrans = (!req_fifo_empty) ? NONSEQ : IDLE;
    DATA: htrans = (hready && hresp==OK && !req_fifo_empty) ? NONSEQ : IDLE;
endcase
```

**NONSEQ**：非顺序传输（每次都是新地址，无突发）。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `imem_req_i` | in | 核取指请求 |
| `imem_addr_i` | in | 取指地址 |
| `imem_req_ack_o` | out | 请求确认 |
| `imem_rdata_o` | out | 读回指令 |
| `imem_resp_o` | out | 响应（OK/ER） |
| `haddr_o` | out | AHB 地址 |
| `htrans_o` | out | AHB 传输类型 |
| `hready_i` | in | AHB 从设备就绪 |
| `hrdata_i` | in | AHB 读回数据 |
| `hresp_i` | in | AHB 响应 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_IMEM_AHB_OUT_BP` | 请求 FIFO bypass（1 深 vs 2 深） |
| `SCR1_IMEM_AHB_IN_BP` | 响应 FIFO bypass（组合 vs 打一拍） |

## 与其他模块关系

```
IFU → ImemAHB: 取指请求
ImemAHB → AHB 总线：HADDR/HTRANS/HREADY
AHB 从设备 → ImemAHB: HRDATA/HRESP
ImemAHB → IFU: 读回指令 + 响应
```

## 配套文档

- 设计规格：`docs/scr1-imem-ahb/design-spec.md`
- 验证文档：`docs/scr1-imem-ahb/verification.md`
