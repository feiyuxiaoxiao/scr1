# scr1_dmem_router.sv 讲解

## 一句话职责

数据路由器：三端口优先级译码（TCM > Timer > AHB）+ 请求分发。

## 文件信息

- **路径**: `src/top/scr1_dmem_router.sv`
- **行数**: 278 行
- **配置依赖**: `SCR1_TCM_EN`

## 核心架构

### 1. 三端口优先级选择（`:88-95`）

```systemverilog
if ((addr & PORT1_MASK) == PORT1_PATTERN) begin
    port_sel = 2'b01;  // TCM（最高优先级）
end else if ((addr & PORT2_MASK) == PORT2_PATTERN) begin
    port_sel = 2'b10;  // Timer（中优先级）
end else begin
    port_sel = 2'b00;  // AHB 桥（最低）
end
```

**地址映射**：
| 端口 | 地址范围 | 掩码 | 模式 |
|---|---|---|---|
| TCM | `0x00480000` | `~0x8000+1` | `(addr & MASK)==PATTERN` |
| Timer | `0x00490000` | `~0x1000+1` | 同上 |
| AHB | 其他 | - | default |

**为什么有优先级？** 避免地址重叠（TCM 和 Timer 地址空间相邻）。

### 2. port_sel_r 锁存（`:80`/`:146-164`）

```systemverilog
// ADDR 态锁存
if (fsm==ADDR && dmem_req) port_sel_r <= port_sel;  // :80

// DATA 态用锁存值选响应
case (port_sel_r)
    2'b01: dmem_rdata = tcm_rdata;
    2'b10: dmem_rdata = timer_rdata;
    2'b00: dmem_rdata = ahb_rdata;
endcase
```

**为什么需要？** 响应可能延迟，期间 `port_sel` 可能变化（新请求）。

### 3. 两态 FSM（`:97-130`）

| 状态 | 含义 | 转移 |
|---|---|---|
| `ADDR` | 地址相位（选端口） | `dmem_req` → `DATA` |
| `DATA` | 数据相位（等响应） | `dmem_resp_received` → `ADDR` |

### 4. 请求生成（`:176-264`）

```systemverilog
case (port_sel_r)
    TCM: begin
        tcm_req = dmem_req;
        tcm_cmd = dmem_cmd;
        tcm_addr = dmem_addr;
    end
    Timer: begin
        timer_req = dmem_req;
        timer_cmd = dmem_cmd;
        timer_addr = dmem_addr;
    end
    AHB: begin
        ahb_req = dmem_req;
        ahb_cmd = dmem_cmd;
        ahb_addr = dmem_addr;
    end
endcase
```

### 5. 响应用回选（`:266-275`）

```systemverilog
always_comb begin
    case (port_sel_r)
        TCM:   dmem_resp = tcm_resp;
        Timer: dmem_resp = timer_resp;
        AHB:   dmem_resp = ahb_resp;
    endcase
end
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `dmem_req_i` | in | LSU 访存请求 |
| `dmem_cmd_i` | in | RD/WR |
| `dmem_width_i` | in | BYTE/HWORD/WORD |
| `dmem_addr_i` | in | 访存地址 |
| `dmem_wdata_i` | in | 写数据 |
| `dmem_req_ack_o` | out | 请求确认 |
| `dmem_rdata_o` | out | 读回数据 |
| `dmem_resp_o` | out | 响应 |
| `port1_*` | - | TCM 接口 |
| `port2_*` | - | Timer 接口 |
| `port0_*` | - | AHB 桥接口 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TCM_EN` | 使能 TCM 端口（port1 逻辑） |

## 与其他模块关系

```
LSU → DmemRouter: 访存请求
DmemRouter → TCM: port1 请求
DmemRouter → Timer: port2 请求
DmemRouter → DmemAHB: port0 请求
TCM/Timer/AHB → DmemRouter: 响应
DmemRouter → LSU: 读回数据 + 响应
```

## 配套文档

- 设计规格：`docs/scr1-dmem-router/design-spec.md`
- 验证文档：`docs/scr1-dmem-router/verification.md`
