# SCR1 TCM（紧耦合存储器）设计规格（Design Specification）

- 模块：`scr1_tcm`（`src/top/scr1_tcm.sv`，131 行；`SCR1_TCM_EN` 保护 `:9`/`:131`）
- 职责：把双端口存储包装为指令/数据两套 TCM 接口

---

## 1. 概述与职责

TCM 由单个双端口存储器 `scr1_dp_memory` 构成：

- **端口 A**：指令读（`imem_rd`，地址 `imem_addr[clog2(SIZE)-1:2]`）；
- **端口 B**：数据读写（`dmem_rd/dmem_wr`，含字节选通）。

核心侧应答固定为单拍 OK：

1. `imem_req_ack = 1'b1`、`dmem_req_ack = 1'b1`（`:71-72`）；
2. 响应 `resp` 由请求产生，持续到下一拍按 `req_en` 变化（`:52-69`）。

---

## 2. 参数与端口

### 2.1 参数（`:11-13`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `SCR1_TCM_SIZE` | `` `SCR1_IMEM_AWIDTH'h00010000 `` | TCM 字节容量 |

### 2.2 端口（`:14-35`）

`clk`/`rst_n`；指令接口 `imem_req_ack/req/addr/rdata/resp`（`:19-24`）；
数据接口 `dmem_req_ack/req/cmd/width/addr/wdata/rdata/resp`（`:26-34`）。

---

## 3. 响应生成（`:52-69`）

```systemverilog
assign imem_req_en = (imem_resp == SCR1_MEM_RESP_RDY_OK) ^ imem_req;
```

请求有效时置 `RDY_OK`，否则回 `NOTRDY`；数据侧同理（`:53`/`:63-69`）。实现单拍
吞吐：只要核心持续请求，`resp` 保持 `RDY_OK`。

---

## 4. 读/写与控制（`:73-95`）

| 信号 | 逻辑 | 行号 |
|---|---|---|
| `imem_rd` | `imem_req` | 76 |
| `dmem_rd` | `dmem_req & (cmd==RD) ` | 77 |
| `dmem_wr` | `dmem_req & (cmd==WR) ` | 78 |
| `dmem_writedata` | BYTE 复制 `wdata[7:0]`×4；HWORD 复制 `[15:0]`×2；否则直通 | 80-95 |
| `dmem_byteen` | BYTE `4'b0001<<addr[1:0]`；HWORD `4'b0011<<{addr[1],1'b0}`；否则 `4'b1111` | 80-95 |

---

## 5. 存储器例化（`:96-117`）

`scr1_dp_memory #(WIDTH=32, SIZE=SCR1_TCM_SIZE)`：

- 端口 A `.rena(imem_rd)`、`.addra(imem_addr[clog2-1:2])`、`.qa(imem_rdata)`；
- 端口 B `.renb/.wenb/.webb/.addrb/.qb/.datab`。

---

## 6. 读数据回对齐（`:118-127`）

`dmem_rdata_shift_reg` 在 `dmem_rd` 捕获 `dmem_addr[1:0]`，输出
`dmem_rdata = dmem_rdata_local >> (8*shift)`，实现字节/半字定位。

---

## 7. 附录

### 7.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_TCM_EN` | 决定模块是否存在 | `:9`/`:131` |

### 7.2 职责边界

- TCM 无地址译码（由路由器保证只命中区间）；
- 单周期访问，无等待态；
- 无 SVA。
