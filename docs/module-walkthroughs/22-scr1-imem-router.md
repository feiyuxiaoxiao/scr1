# scr1_imem_router.sv 讲解

## 一句话职责

指令路由器：按地址译码分发取指请求到 TCM/TCM-less 路径。

## 文件信息

- **路径**: `src/top/scr1_imem_router.sv`
- **行数**: 185 行
- **配置依赖**: `SCR1_TCM_EN`

## 核心架构

### 1. 地址译码（`:64`）

```systemverilog
assign port_sel = ((addr & PORT1_MASK) == PORT1_PATTERN);
```

| 配置 | PORT1_MASK | PORT1_PATTERN | 目标 |
|---|---|---|---|
| TCM 使能 | `~TCM_SIZE+1` | `TCM_BASE` | TCM |
| TCM 禁用 | - | - | 直通 AHB 桥 |

**示例**：TCM 在 `0x00480000`，大小 32KB：
```
PORT1_MASK   = ~0x8000 + 1 = 0xFFFF8001
PORT1_PATTERN = 0x00480000
addr & 0xFFFF8000 == 0x00480000 → 选中 TCM
```

### 2. 两态 FSM（`:66-99`）

| 状态 | 含义 | 转移 |
|---|---|---|
| `IDLE` | 空闲 | `imem_req` → `BUSY` |
| `BUSY` | 在途 | `imem_resp` → `IDLE` |

```systemverilog
always_ff @(posedge clk) begin
    case (fsm)
        IDLE: if (imem_req) fsm = BUSY;
        BUSY: if (imem_resp_received) fsm = IDLE;
    endcase
end
```

### 3. port_sel_r  late-resp 修正（`:69`/`:75`）

**问题**：IMEM 响应可能延迟多拍，期间 `port_sel` 可能因新请求而变化。

**解决**：ADDR 态锁存 `port_sel_r`，DATA 态用它选响应：

```systemverilog
if (fsm==IDLE && imem_req) port_sel_r <= port_sel;  // :69
// ...
assign imem_to_port = port_sel_r ? tcm_resp : ahb_resp;  // :75
```

### 4. 流控制（`:101-107`）

```systemverilog
assign imem_req_ack = (fsm==IDLE) & ~(tcm_busy | ahb_busy);
```

**含义**：IDLE 态且目标端口不忙时才 ack。

### 5. XPROP 处理（`:138-144`）

```systemverilog
`ifdef SCR1_XPROP_EN
assign imem_rdata = 'x;
`else
assign imem_rdata = port_sel_r ? tcm_rdata : ahb_rdata;
`endif
```

**用途**：仿真时传播 `x` 态（综合忽略）。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `imem_req_i` | in | IFU 取指请求 |
| `imem_addr_i` | in | 取指地址 |
| `imem_req_ack_o` | out | 请求确认 |
| `imem_rdata_o` | out | 读回指令 |
| `imem_resp_o` | out | 响应 |
| `port1_req_o` | out | TCM 请求 |
| `port0_req_o` | out | AHB 桥请求 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TCM_EN` | 使能 TCM 端口（port_sel 逻辑） |

## 与其他模块关系

```
IFU → ImemRouter: 取指请求
ImemRouter → TCM: port1 请求
ImemRouter → ImemAHB: port0 请求
TCM/AHB → ImemRouter: 响应
ImemRouter → IFU: 读回指令
```

## 配套文档

- 设计规格：`docs/scr1-imem-router/design-spec.md`
- 验证文档：`docs/scr1-imem-router/verification.md`
