# scr1_mem_axi.sv 讲解

## 一句话职责

AXI 存储桥：环形队列 + 三指针 + 四位状态，支持 AW/W 分离跟踪。

## 文件信息

- **路径**: `src/top/scr1_mem_axi.sv`
- **行数**: 363 行
- **配置依赖**: `SCR1_AXI_REQ_BP`、`SCR1_AXI_RESP_BP`

## 核心架构

### 1. 为什么不用 FIFO+FSM？

AHB 是单通道，一次只能有一个事务在途，所以"FIFO + 两态 FSM"就够了。

AXI 有 **5 个独立通道**（AR/AW/R/W/B），且 **AW（写地址）和 W（写数据）可以任意顺序完成**。一个事务的"地址相位"和"数据相位"可能相差好几拍，桥必须能同时跟踪多个事务的不同阶段。

### 2. 环形缓冲区 + 三指针（`:111-117`）

```systemverilog
localparam SCR1_REQ_BUF_SIZE = 2;

req_fifo[SCR1_REQ_BUF_SIZE]      // 事务内容（width/addr/wdata）
req_status[SCR1_REQ_BUF_SIZE]    // 每个事务的四位状态
req_aval_ptr, req_proc_ptr, req_done_ptr
```

| 指针 | 职责 |
|---|---|
| `req_aval_ptr` | 收新请求的位置（Available，最旧） |
| `req_proc_ptr` | 发 AXI 请求的位置（Processing，中间） |
| `req_done_ptr` | 收 AXI 响应的位置（Done，最新） |

**在途事务数** = `req_done_ptr - req_aval_ptr`（环形减法）。

### 3. 四位状态（`:103-108`）

```systemverilog
typedef struct packed {
    logic req_write;   // 这是写还是读
    logic req_addr;    // 地址相位还没做完
    logic req_data;    // 数据相位还没做完（仅写）
    logic req_resp;    // 事务还在途（占用标志）
} type_scr1_req_status_s;
```

**这是整个模块的关键**：`req_addr` 和 `req_data` 分开记，正因为 AW 和 W 可能不同拍完成。

### 4. 状态更新（`:157-194`）

#### 新请求进来（`:164-175`）
```systemverilog
if (core_req_ack) begin
    req_status[aval_ptr].req_resp = 1'b1;   // 占用槽位
    req_status[aval_ptr].req_write = is_write;
    req_status[aval_ptr].req_addr  = 1'b1;  // 地址相位待完成
    req_status[aval_ptr].req_data  = is_write;  // 写操作才有数据相位
end
```

#### 地址相位完成（`:178-181`）
```systemverilog
if ((awvalid & awready) | (arvalid & arready)) begin
    req_status[proc_ptr].req_addr = 1'b0;
end
```

#### 数据相位完成（`:184-187`）
```systemverilog
if (wvalid & wready & wlast) begin
    req_status[proc_ptr].req_data = 1'b0;
end
```

#### 事务完成（`:190-193`）
```systemverilog
if (rvalid & rready & rlast | bvalid & bready) begin
    req_status[done_ptr].req_resp = 1'b0;  // 释放槽位
end
```

### 5. proc_ptr 推进条件（`:218-221`）

```systemverilog
if ((awvalid & awready & wvalid & wready & wlast) |  // 写：地址 + 数据同拍
    (~force_write & ~req_addr & awvalid & awready) | // 写：地址先完成
    (~force_write & ~req_data & wvalid & wready & wlast) |  // 写：数据后完成
    (~req_data & arvalid & arready)) begin  // 读：地址完成即可
    req_proc_ptr <= req_proc_ptr + 1;
end
```

**四种情况**，兼容 AW 先、W 先、两者同拍这三种从设备行为。

### 6. bypass 参数（`:170-174`）

```systemverilog
// 如果开了 bypass 且这一拍就完成了，就不必置标志位
req_status[aval_ptr].req_addr = ~(force_axi_req & axi_ready);
req_status[aval_ptr].req_data = is_write & ~(force_axi_req & axi_ready);
```

**省一拍**：bypass 模式下，地址/数据相位同拍完成，无需标记"待完成"。

### 7. 背压和空闲（`:126-144`）

```systemverilog
assign core_req_ack = ~axi_reinit & ~req_status[aval_ptr].req_resp & core_resp != RDY_ER;
assign core_idle = ~(|req_status.req_resp);  // 所有槽位都释放
```

`core_idle` 是上一轮讲 `scr1_top_axi` 时 `axi_reinit` 等的那个信号——两个桥都 idle 了才放开总线。

### 8. AXI 输出固定值（`:318-341`）

```systemverilog
assign awlen = 8'd0;        // 单拍突发（len=0 表示 1 拍）
assign awburst = 2'd1;      // INCR
assign awcache = 4'd2;      // modifiable + bufferable
assign awid = 1; arid = 0;  // 读写用不同 ID
assign wlast = 1;           // 永远最后一拍（因为 len=0）
```

### 9. 字节路处理（`:266-297`）

AXI 用 `wstrb` 掩码，比 AHB 的"数据搬到对应字节路"更直接：

```systemverilog
// 写掩码
case (width)
    BYTE:  wstrb = (1 << addr[1:0]);      // 1 位
    HWORD: wstrb = (3 << addr[1:0]);      // 2 位
    WORD:  wstrb = (f << addr[1:0]);      // 4 位
endcase

// 数据移位
assign wdata = core_wdata << (8*addr[1:0]);

// 读回反向右移
assign core_rdata = axi_rdata >> (8*addr[1:0]);
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `core_req_i` | in | 核访存请求 |
| `core_cmd_i` | in | RD/WR |
| `core_width_i` | in | BYTE/HWORD/WORD |
| `core_addr_i` | in | 访存地址 |
| `core_wdata_i` | in | 写数据 |
| `core_req_ack_o` | out | 请求确认 |
| `core_rdata_o` | out | 读回数据 |
| `core_resp_o` | out | 响应 |
| `core_idle_o` | out | 无在途事务 |
| `arvalid_o`/`arready_i` | - | AXI 读地址通道 |
| `awvalid_o`/`awready_i` | - | AXI 写地址通道 |
| `wvalid_o`/`wready_i` | - | AXI 写数据通道 |
| `rvalid_i`/`rready_o` | - | AXI 读数据通道 |
| `bvalid_i`/`bready_o` | - | AXI 写响应通道 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_AXI_REQ_BP` | 请求 bypass（组合 vs 打拍） |
| `SCR1_AXI_RESP_BP` | 响应 bypass（组合 vs 打拍） |

## 配套文档

- 设计规格：`docs/scr1-mem-axi/design-spec.md`
- 验证文档：`docs/scr1-mem-axi/verification.md`
