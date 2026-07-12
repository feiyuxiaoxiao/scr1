# SCR1 加载/存储单元（LSU）技术设计文档

> 对应源码: `src/core/pipeline/scr1_pipe_lsu.sv`
> 模块名: `scr1_pipe_lsu`
> 制定者: Syntacore LLC, 2016-2021

---

## 1. 总体架构

### 1.1 功能定位

LSU 是 SCR1 处理器流水线中负责数据存储器（DMEM）访问的执行单元。它接收 EXU 的访存命令，通过标准握手协议与 DMEM 交互，完成加载和存储操作。核心职责：

1. 管理 LSU ↔ DMEM 接口协议，控制请求/响应时序
2. 译码 EXU 的 LSU 命令（LB/LH/LW/LBU/LHU/SB/SH/SW），生成 DMEM 命令和宽度
3. 检测地址未对齐异常（加载和存储分别检查）
4. 处理 DMEM 返回的访问错误（Access Fault）
5. 对加载数据进行符号/零扩展
6. 向 TDU 提供数据地址流监视信息并生成断点异常

### 1.2 模块划分

```
                          scr1_pipe_lsu
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌──────────────┐   ┌──────────────────┐   ┌────────────────┐   │
│  │   LSU FSM    │   │  Command Register │   │  Exceptions    │   │
│  │   (2 states) │   │  (lsu_cmd_ff)     │   │  Logic         │   │
│  └──────┬───────┘   └────────┬─────────┘   └───────┬────────┘   │
│         │                    │                      │            │
│         ▼                    ▼                      ▼            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   LSU ↔ DMEM Interface                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │   │
│  │  │ Command  │ │  Width   │ │ Address  │ │ Load Data   │  │   │
│  │  │ Decode   │ │ Decode   │ │ Passthru │ │ Sign-Extend │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   LSU ↔ TDU Interface                     │   │
│  │  Data Monitoring → Breakpoint Exception ← Hardware        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 配置宏一览

| 宏名 | 作用 |
|------|------|
| `SCR1_TDU_EN` | 使能触发调试单元（TDU），启用数据断点和指令断点接口 |
| `SCR1_TRGT_SIMULATION` | 使能 SVA 断言（仅仿真） |

---

## 2. LSU FSM（访存有限状态机）

### 2.1 状态定义

```
          ┌──────────────────────────────────┐
          │                                  │
          ▼                                  │
    ┌───────────────┐   dmem_req_vd  ┌───────────────┐
    │     IDLE      │──────────────▶ │     BUSY      │
    │  (等待请求)   │                │  (等待响应)   │
    └───────────────┘                └───────────────┘
          ▲                                  │
          │       dmem_resp_received         │
          └──────────────────────────────────┘
```

| 状态 | 编码 | 含义 |
|------|:---:|------|
| `SCR1_LSU_FSM_IDLE` | 1'b0 | 空闲。LSU 未向 DMEM 发出请求，等待 EXU 传来新访存命令 |
| `SCR1_LSU_FSM_BUSY` | 1'b1 | 忙碌。DMEM 请求已发出，等待 DMEM 返回响应数据/错误 |

### 2.2 状态转移条件

**IDLE → BUSY**: DMEM 请求有效（三重条件同时满足）

```systemverilog
assign dmem_req_vd = exu2lsu_req_i & dmem2lsu_req_ack_i & ~lsu_exc_req;
```

三要素：EXU 发起请求、DMEM 确认握手、当前无异常。缺少任一条件则不发起 DMEM 请求。

**BUSY → IDLE**: 收到 DMEM 响应

```systemverilog
assign dmem_resp_received = dmem_resp_ok | dmem_resp_er;
```

DMEM 响应 OK 或 ERROR 均可结束 BUSY 状态——前者读出数据正常返回给 EXU，后者触发访问错误异常。

### 2.3 状态机实现

```systemverilog
// 状态寄存器
always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)     lsu_fsm_curr <= SCR1_LSU_FSM_IDLE;
    else            lsu_fsm_curr <= lsu_fsm_next;
end

// 下一状态组合逻辑
always_comb begin
    case (lsu_fsm_curr)
        SCR1_LSU_FSM_IDLE: lsu_fsm_next = dmem_req_vd        ? SCR1_LSU_FSM_BUSY : SCR1_LSU_FSM_IDLE;
        SCR1_LSU_FSM_BUSY: lsu_fsm_next = dmem_resp_received ? SCR1_LSU_FSM_IDLE : SCR1_LSU_FSM_BUSY;
    endcase
end
```

### 2.4 FSM 对 DMEM 请求的门控

LSU 向 DMEM 发出请求不仅要求 `dmem_req_vd` 为真，还要求 FSM 处于 IDLE 状态：

```systemverilog
assign lsu2dmem_req_o = exu2lsu_req_i & ~lsu_exc_req & lsu_fsm_idle;
```

这是 LSU FSM 的核心约束：LSU 同一时刻最多只存在一个未完成的 DMEM 事务。一旦进入 BUSY 状态，不会发出新请求，直到当前事务完成。

---

## 3. DMEM 命令与数据宽度译码

### 3.1 加载/存储命令判定

EXU 发来的 `exu2lsu_cmd_i` 被分类为加载或存储命令：

```systemverilog
assign dmem_cmd_load  = (exu2lsu_cmd_i == SCR1_LSU_CMD_LB )
                      | (exu2lsu_cmd_i == SCR1_LSU_CMD_LBU)
                      | (exu2lsu_cmd_i == SCR1_LSU_CMD_LH )
                      | (exu2lsu_cmd_i == SCR1_LSU_CMD_LHU)
                      | (exu2lsu_cmd_i == SCR1_LSU_CMD_LW );

assign dmem_cmd_store = (exu2lsu_cmd_i == SCR1_LSU_CMD_SB )
                      | (exu2lsu_cmd_i == SCR1_LSU_CMD_SH )
                      | (exu2lsu_cmd_i == SCR1_LSU_CMD_SW );
```

### 3.2 数据宽度译码

| LSU 命令 | `dmem_wdth_word` | `dmem_wdth_hword` | `dmem_wdth_byte` |
|----------|:---:|:---:|:---:|
| `LB` / `LBU` / `SB` | 0 | 0 | 1 |
| `LH` / `LHU` / `SH` | 0 | 1 | 0 |
| `LW` / `SW` | 1 | 0 | 0 |

```systemverilog
assign dmem_wdth_word  = (exu2lsu_cmd_i == SCR1_LSU_CMD_LW )
                       | (exu2lsu_cmd_i == SCR1_LSU_CMD_SW );
assign dmem_wdth_hword = (exu2lsu_cmd_i == SCR1_LSU_CMD_LH )
                       | (exu2lsu_cmd_i == SCR1_LSU_CMD_LHU)
                       | (exu2lsu_cmd_i == SCR1_LSU_CMD_SH );
assign dmem_wdth_byte  = (exu2lsu_cmd_i == SCR1_LSU_CMD_LB )
                       | (exu2lsu_cmd_i == SCR1_LSU_CMD_LBU)
                       | (exu2lsu_cmd_i == SCR1_LSU_CMD_SB );
```

### 3.3 DMEM 接口输出生成

```systemverilog
assign lsu2dmem_cmd_o   = dmem_cmd_store ? SCR1_MEM_CMD_WR : SCR1_MEM_CMD_RD;
assign lsu2dmem_width_o = dmem_wdth_byte  ? SCR1_MEM_WIDTH_BYTE
                        : dmem_wdth_hword ? SCR1_MEM_WIDTH_HWORD
                                          : SCR1_MEM_WIDTH_WORD;
assign lsu2dmem_addr_o  = exu2lsu_addr_i;    // 地址直通
assign lsu2dmem_wdata_o = exu2lsu_sdata_i;   // 写数据直通
```

设计决策：`lsu2dmem_cmd_o` 仅用 load/store 二分类判定，因为 DMEM 只需知道读写方向即可。数据宽度由 `lsu2dmem_width_o` 单独告知。地址和写数据直接透传，LSU 不做任何变换。

---

## 4. 命令寄存器（Command Register）

### 4.1 设计动机

EXU 提供的 `exu2lsu_cmd_i` 只在请求发出的那个周期有效。DMEM 响应可能若干周期后才返回，此时 LSU 需要知道当初发起的是什么指令，才能正确：

- 对加载数据进行符号/零扩展
- 在访问错误时输出正确的异常代码（区分加载错误和存储错误）

### 4.2 实现

```systemverilog
assign lsu_cmd_upd = lsu_fsm_idle & dmem_req_vd;

always_ff @(posedge clk, negedge rst_n) begin
    if (~rst_n)
        lsu_cmd_ff <= SCR1_LSU_CMD_NONE;
    else if (lsu_cmd_upd)
        lsu_cmd_ff <= exu2lsu_cmd_i;
end
```

`lsu_cmd_ff` 在 FSM 从 IDLE 跳转到 BUSY 的同拍锁存 EXU 命令，整段 BUSY 期间保持不变。复位后为 `NONE`。

### 4.3 锁存值的二次译码

```systemverilog
assign lsu_cmd_ff_load  = (lsu_cmd_ff == SCR1_LSU_CMD_LB ) | (lsu_cmd_ff == SCR1_LSU_CMD_LBU)
                        | (lsu_cmd_ff == SCR1_LSU_CMD_LH ) | (lsu_cmd_ff == SCR1_LSU_CMD_LHU)
                        | (lsu_cmd_ff == SCR1_LSU_CMD_LW );
assign lsu_cmd_ff_store = (lsu_cmd_ff == SCR1_LSU_CMD_SB ) | (lsu_cmd_ff == SCR1_LSU_CMD_SH )
                        | (lsu_cmd_ff == SCR1_LSU_CMD_SW );
```

锁存值按 load/store 再次分类，用于异常代码生成——访问错误时需要根据锁存的 load/store 属性选择 `LD_ACCESS_FAULT` 或 `ST_ACCESS_FAULT`。

---

## 5. 地址未对齐检测

### 5.1 检测逻辑

```systemverilog
assign dmem_addr_mslgn = exu2lsu_req_i & ( (dmem_wdth_hword & exu2lsu_addr_i[0])
                                         | (dmem_wdth_word  & |exu2lsu_addr_i[1:0]));
```

| 数据宽度 | 对齐要求 | 检测条件 | 未对齐示例地址 |
|----------|------|------|------|
| BYTE | 无（任意地址） | 不检测 | — |
| HWORD | 2 字节对齐 | `addr[0] != 0` | 0x0001, 0x0003, 0x1001 |
| WORD | 4 字节对齐 | `|addr[1:0] != 0` | 0x0001, 0x0002, 0x0003, 0x1001 |

表达式 `|exu2lsu_addr_i[1:0]` 等效于 `addr[1:0] != 2'b00`，即最低两位任一位非零即为未对齐。

`exu2lsu_req_i` 作为使能——只在 EXU 确实发起请求时才检测，避免 EXU 不请求时地址总线的随机值被误判为未对齐。

### 5.2 加载/存储分离

```systemverilog
assign dmem_addr_mslgn_l = dmem_addr_mslgn & dmem_cmd_load;
assign dmem_addr_mslgn_s = dmem_addr_mslgn & dmem_cmd_store;
```

未对齐检测结果按加载/存储分开，用于生成精确的异常代码（`LD_ADDR_MISALIGN` / `ST_ADDR_MISALIGN`）。这种分离不是多余的——RISC-V 特权架构规范对不同来源的异常提供了不同的 mtval/mcause 编码，分离能精确定位异常源。

---

## 6. 异常生成逻辑

### 6.1 LSU 产出的异常类型

| 异常代码 | 编码 | 触发条件 | 来源 |
|------|:---:|------|------|
| `SCR1_EXC_CODE_LD_ADDR_MISALIGN` | 4'd4 | 加载地址未对齐 | LSU 内部检测 |
| `SCR1_EXC_CODE_LD_ACCESS_FAULT` | 4'd5 | DMEM 返回加载错误 | DMEM 响应 |
| `SCR1_EXC_CODE_ST_ADDR_MISALIGN` | 4'd6 | 存储地址未对齐 | LSU 内部检测 |
| `SCR1_EXC_CODE_ST_ACCESS_FAULT` | 4'd7 | DMEM 返回存储错误 | DMEM 响应 |
| `SCR1_EXC_CODE_BREAKPOINT` | 4'd3 | TDU 触发断点 | TDU 接口 |

### 6.2 异常代码优先级（case 语句顺序）

LSU 使用 `case (1'b1)` 优先级编码器而非 `unique case` 确定异常代码，确保了即使多种异常条件同时满足，也能输出确定的优先级最高者：

```
优先级 1（最高）: dmem_resp_er        → LD_ACCESS_FAULT / ST_ACCESS_FAULT
优先级 2        : lsu_exc_hwbrk       → BREAKPOINT
优先级 3        : dmem_addr_mslgn_l   → LD_ADDR_MISALIGN
优先级 4        : dmem_addr_mslgn_s   → ST_ADDR_MISALIGN
优先级 5（最低）: default             → INSTR_MISALIGN（保护值，实际不应触发）
```

关键设计决策：

- **DMEM 响应错误优先级最高**: 一旦 DMEM 返回错误，无论是否有地址未对齐或断点，都报告 Access Fault——因为此时数据已不可用。
- **断点优先于地址未对齐**: TDU 断点异常优先级高于地址未对齐检测，确保调试器能捕获到程序流中的断点事件。
- **加载优先于存储**: 在未对齐类别内部，加载地址未对齐优先于存储地址未对齐被输出。这是因为 RISC-V 处理器在同一拍不可能同时进行加载和存储，但防御性地用优先级规避不确定性。

### 6.3 异常请求汇总

```systemverilog
assign lsu_exc_req = dmem_addr_mslgn_l | dmem_addr_mslgn_s
`ifdef SCR1_TDU_EN
                   | lsu_exc_hwbrk
`endif // SCR1_TDU_EN
;
```

`lsu_exc_req` 汇总所有 LSU 内部可检测的异常（不包括 DMEM 返回的错误——那是通过 `dmem_resp_er` 路径直接驱动 `lsu2exu_exc_o`）。

### 6.4 EXU 异常输出

```systemverilog
assign lsu2exu_exc_o = dmem_resp_er | lsu_exc_req;
```

DMEM 返回错误和 LSU 内部检测异常二者 OR 后驱动 EXU 异常信号——两种来源覆盖了 LSU 能触发的所有异常路径。

### 6.5 异常对 DMEM 请求的影响

```systemverilog
assign dmem_req_vd   = exu2lsu_req_i & dmem2lsu_req_ack_i & ~lsu_exc_req;
assign lsu2dmem_req_o = exu2lsu_req_i & ~lsu_exc_req & lsu_fsm_idle;
```

地址未对齐或断点异常一旦检测到，`~lsu_exc_req` 为 0，阻止向 DMEM 发出请求——异常指令不真正访问存储器，直接向 EXU 报告异常。

---

## 7. 加载数据符号/零扩展

### 7.1 扩展逻辑

LSU 根据锁存的命令寄存器值（而非当前 EXU 命令），选择正确的扩展方式：

```systemverilog
always_comb begin
    case (lsu_cmd_ff)
        SCR1_LSU_CMD_LH : lsu2exu_ldata_o = {{16{dmem2lsu_rdata_i[15]}}, dmem2lsu_rdata_i[15:0]};
        SCR1_LSU_CMD_LHU: lsu2exu_ldata_o = { 16'b0,                     dmem2lsu_rdata_i[15:0]};
        SCR1_LSU_CMD_LB : lsu2exu_ldata_o = {{24{dmem2lsu_rdata_i[7]}},  dmem2lsu_rdata_i[7:0]};
        SCR1_LSU_CMD_LBU: lsu2exu_ldata_o = { 24'b0,                     dmem2lsu_rdata_i[7:0]};
        default         : lsu2exu_ldata_o = dmem2lsu_rdata_i;
    endcase
end
```

| LSU 命令 | 扩展方式 | 32 位结果构成 |
|----------|------|------|
| `LH` | 符号扩展 | `{dmem[15] × 16, dmem[15:0]}` |
| `LHU` | 零扩展 | `{16'd0, dmem[15:0]}` |
| `LB` | 符号扩展 | `{dmem[7] × 24, dmem[7:0]}` |
| `LBU` | 零扩展 | `{24'd0, dmem[7:0]}` |
| `LW` | 直通 | `dmem[31:0]` |

### 7.2 使用锁存命令而非当前命令的设计原因

这个设计决策至关重要：`exu2lsu_cmd_i` 仅在请求发出的那一个时钟周期有效，DMEM 的响应可能若干周期后才到达。如果 `always_comb` 中用 `exu2lsu_cmd_i` 而非 `lsu_cmd_ff`，延迟响应时 EXU 可能已经发起下一个不相关的请求，导致错误的扩展逻辑被激活。`lsu_cmd_ff` 在 BUSY 期间保持稳定，保证扩展方式与发起时的指令一致。

### 7.3 存储数据处理

存储路径上数据不做变换——`exu2lsu_sdata_i` 直通到 `lsu2dmem_wdata_o`：

```systemverilog
assign lsu2dmem_wdata_o = exu2lsu_sdata_i;
```

DMEM 内部根据 `lsu2dmem_width_o` 只取对应低字节/半字写入，高位字节被 DMEM 忽略。

---

## 8. TDU 断点接口（仅 `SCR1_TDU_EN`）

### 8.1 数据地址流监视

LSU 向 TDU 输出 `lsu2tdu_dmon_o` 结构体，供 TDU 监视数据地址流并匹配数据断点：

```systemverilog
assign lsu2tdu_dmon_o.vd    = exu2lsu_req_i & lsu_fsm_idle & ~tdu2lsu_ibrkpt_exc_req_i;
assign lsu2tdu_dmon_o.addr  = exu2lsu_addr_i;
assign lsu2tdu_dmon_o.load  = dmem_cmd_load;
assign lsu2tdu_dmon_o.store = dmem_cmd_store;
```

`vd` 有效的条件：
1. EXU 发起了请求
2. FSM 处于 IDLE（防止已在进行中的事务重复触发监视）
3. 指令断点未激活（`~tdu2lsu_ibrkpt_exc_req_i`）——指令断点异常会拦截数据监视，因为此时流水线应将当前指令当作已命中断点处理

### 8.2 断点异常生成

```systemverilog
assign lsu_exc_hwbrk = (exu2lsu_req_i & tdu2lsu_ibrkpt_exc_req_i)
                     | tdu2lsu_dbrkpt_exc_req_i;
```

两种来源：

| 断点类型 | 信号 | 触发时机 | 语义 |
|------|------|------|------|
| 指令断点 | `tdu2lsu_ibrkpt_exc_req_i` | 当前指令本身是断点目标 | 仅当 EXU 确实发起 LSU 请求时才生效（非访存指令不会被 LSU 断点拦截） |
| 数据断点 | `tdu2lsu_dbrkpt_exc_req_i` | 当前访问的数据地址命中数据断点 | TDU 在上一拍匹配到数据地址后，本拍置位，LSU 直接转为异常 |

`lsu_exc_hwbrk` 被归入 `lsu_exc_req`，会阻止向 DMEM 发出请求，断点异常不产生实际的存储器访问。

---

## 9. LSU ↔ EXU 完成信号

### 9.1 就绪信号

```systemverilog
assign lsu2exu_rdy_o = dmem_resp_received;
```

就绪信号直接绑定 DMEM 响应——EXU 只有在 DMEM 返回结果后才能继续流水线。这保证了加载指令的 RAW（Read-After-Write）依赖被正确处理。

### 9.2 异常信号

```systemverilog
assign lsu2exu_exc_o = dmem_resp_er | lsu_exc_req;
```

异常信号有三种来源：DMEM 访问错误、地址未对齐、硬件断点。EXU 收到异常后进入陷阱处理流程。

---

## 10. 关键设计决策与分析

### 10.1 双状态 FSM vs 流水线化

LSU 仅用 IDLE/BUSY 两个状态管理 DMEM 交互，不支持多事务流水线化。这意味着 LSU 同时只能有一个未完成的 DMEM 请求。代价是访存延迟等于 DMEM 访问延迟（无吞吐量提升），但在 SCR1 作为顺序单发射核心的目标定位下，这种简单性优于性能——多事务流水线化需要 OoO（乱序）完成和重排序缓冲，对面积和功耗不利。

### 10.2 命令寄存器的必要性

如果没有 `lsu_cmd_ff`，DMEM 响应到达时原始命令信息已丢失，无法区分加载错误的类型、无法正确扩展加载数据。这种"状态记忆"是任何存在可变延迟接口的模块的基本需求。

### 10.3 未对齐异常前置检测

LSU 在向 DMEM 发送请求之前就完成了地址未对齐检测。当检测到未对齐时，`lsu2dmem_req_o = 0`——请求根本没有发送到 DMEM。这避免了无效访问浪费 DMEM 带宽，也避免了 DMEM 因未对齐访问自身报错而导致异常来源模糊不清。

### 10.4 store 未对齐的处理

store 地址未对齐检测使用了和 load 完全相同的逻辑（基于 `dmem_wdth_*`）。对于 store 指令，数据宽度依然有效（SB=byte, SH=hword, SW=word），所以对齐检测逻辑直接复用。分离的 `dmem_addr_mslgn_l` 和 `dmem_addr_mslgn_s` 仅用于异常代码区别。

### 10.5 TDU 断点与访存的仲裁

`tdu2lsu_ibrkpt_exc_req_i` 参与两处逻辑：

1. **阻止数据监视**（`lsu2tdu_dmon_o.vd` 中取反）：避免指令断点在 TDU 中被重复记录
2. **触发断点异常**（`lsu_exc_hwbrk` 的子项）：阻止实际的 DMEM 访问

这种设计确保指令断点异常"劫持"了当前 LSU 指令——不访问存储器、不产生数据监视记录、直接进入异常处理。

---

## 11. 验证断言概要

11 条 SVA 断言覆盖以下检查维度：

- **X 态检测**（7 条）：控制信号、命令、地址、存储数据、异常代码、DMEM 请求输出、DMEM 确认信号不能为未知值
- **行为正确性**（3 条）：
  - `SCR1_SVA_LSU_EXC_ONEHOT`: 异常原因互斥——`dmem_resp_er`、`dmem_addr_mslgn_l`、`dmem_addr_mslgn_s` 三者不可同时成立
  - `SCR1_SVA_LSU_UNEXPECTED_DMEM_RESP`: IDLE 状态不应收到 DMEM 响应
  - `SCR1_SVA_LSU_REQ_EXC`: 异常输出时必须伴随 EXU 请求
- **覆盖率**（1 条）：`SCR1_COV_LSU_MISALIGN_BRKPT` — 地址未对齐与断点同时触发的边界场景覆盖
