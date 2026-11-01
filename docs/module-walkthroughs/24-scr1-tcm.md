# scr1_tcm.sv 讲解

## 一句话职责

紧耦合内存：单周期应答 + 字节路访问 + 读数据对齐。

## 文件信息

- **路径**: `src/top/scr1_tcm.sv`
- **行数**: 131 行
- **配置依赖**: `SCR1_TCM_EN`

## 核心架构

### 1. 单周期应答（`:71-72`）

```systemverilog
assign tcd_req_ack = 1'b1;  // 永远就绪（单周期）
```

**为什么能永远就绪？** TCM 是片上 SRAM，访问延迟 1 拍（与 CPU 同频）。

### 2. 响应 XOR 技巧（`:52-53`）

```systemverilog
assign tcd_resp = (tcd_req_ack & ~tcd_resp_err) ? SCR1_MEM_RESP_RDY_OK : SCR1_MEM_RESP_RDY_ER;
```

**简化设计**：TCM 不会出错（无 ECC/校验），`resp_err` 恒 0，所以 `resp` 恒 `RDY_OK`。

### 3. 写对齐（`:80-95`）

```systemverilog
case (tcd_width)
    BYTE: begin
        tcd_wmask = 4'b0001 << tcd_addr[1:0];
        tcd_wdata = wdata << (8*tcd_addr[1:0]);
    end
    HWORD: begin
        tcd_wmask = 4'b0011 << tcd_addr[1:0];
        tcd_wdata = wdata << (8*tcd_addr[1:0]);
    end
    WORD: begin
        tcd_wmask = 4'b1111;
        tcd_wdata = wdata;
    end
endcase
```

**为什么需要移位？** TCM 是字数组（32 位宽），但访问可能是字节/半字。

### 4. 读数据对齐（`:121-127`）

```systemverilog
always_ff @(posedge clk) begin
    case (tcd_width)
        BYTE:  tcd_rdata <= tcm_rdata >> (8*tcd_addr_latched[1:0]);
        HWORD: tcd_rdata <= tcm_rdata >> (8*tcd_addr_latched[1:0]);
        WORD:  tcd_rdata <= tcm_rdata;
    endcase
end
```

**为什么要锁存地址？** 读响应晚一拍，需要记住当时的 `addr[1:0]` 来决定右移几位。

### 5. 尺寸参数（`:20-30`）

```systemverilog
localparam TCM_SIZE = SCR1_TCM_ADDR_MASK + 1;  // 如 0x8000 = 32KB
localparam TCM_ADDR_WIDTH = $clog2(TCM_SIZE/4);  // 字地址宽度
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `tcd_req_i` | in | 访存请求 |
| `tcd_cmd_i` | in | RD/WR |
| `tcd_width_i` | in | BYTE/HWORD/WORD |
| `tcd_addr_i` | in | 访存地址 |
| `tcd_wdata_i` | in | 写数据 |
| `tcd_req_ack_o` | out | 请求确认（恒 1） |
| `tcd_rdata_o` | out | 读回数据 |
| `tcd_resp_o` | out | 响应（恒 RDY_OK） |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TCM_EN` | TCM 模块使能 |
| `SCR1_TCM_ADDR_MASK` | TCM 尺寸（如 `0x7FFF` = 32KB） |

## 配套文档

- 设计规格：`docs/scr1-tcm/design-spec.md`
- 验证文档：`docs/scr1-tcm/verification.md`
