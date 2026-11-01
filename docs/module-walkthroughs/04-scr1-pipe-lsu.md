# scr1_pipe_lsu.sv 讲解

## 一句话职责

访存单元：LOAD/STORE 请求发往 DMEM，地址对齐检查，读数据符号扩展。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_lsu.sv`
- **行数**: 351 行
- **配置依赖**: `SCR1_TDU_EN`

## 核心功能

### 1. 两态 FSM（`:67-191`）

```
IDLE: 收到请求（dmem_req_vd）→ BUSY
BUSY: 收到响应（dmem_resp_received）→ IDLE
```

**为什么需要 FSM？** DMEM 响应可能延迟多周期（AHB 桥/外部内存），LSU 必须"记住"自己在等响应。

### 2. 地址对齐检查（`:206-209`）

```systemverilog
dmem_addr_mslgn = (dmem_wdth_hword & addr[0]) | (dmem_wdth_word & |addr[1:0]);
```

- halfword 必须 2 字节对齐 → `addr[0]==0`
- word 必须 4 字节对齐 → `addr[1:0]==00`

### 3. 异常码生成（`:212-224`）

| 错误类型 | LOAD 异常码 | STORE 异常码 |
|---|---|---|
| 地址错位 | `LD_ADDR_MISALIGN` | `ST_ADDR_MISALIGN` |
| 访问错误 | `LD_ACCESS_FAULT` | `ST_ACCESS_FAULT` |

**访问错误**：地址落在未映射区域（如无外设、无 TCM）。

### 4. 读数据符号扩展（`:240-248`）

```systemverilog
case (dmem_width)
    BYTE:  dmem_rdata = {32{lb_data[7]}} ? lb_data[7:0];   // 符号扩展
    HWORD: dmem_rdata = {16{lb_data[15]}} ? lb_data[15:0]; // 符号扩展
    WORD:  dmem_rdata = lb_data;                            // 无需扩展
endcase
```

`LBU`/`LHU`（无符号）用零扩展：`{24'b0, lb_data[7:0]}`。

### 5. TDU 集成（`:254-322`）

TDU（硬件触发器）监控 LOAD/STORE 地址，命中时触发调试异常：
- `tdu2lsu_exec_match_i`：PC 匹配
- `tdu2lsu_ldst_match_i`：地址匹配
- 命中 → `lsu2exu_tdu_exc_o=1`

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `exu2lsu_req_i` | in | EXU 发来访存请求 |
| `exu2lsu_cmd_i` | in | RD/WR |
| `exu2lsu_width_i` | in | BYTE/HWORD/WORD |
| `exu2lsu_addr_i` | in | 访存地址 |
| `exu2lsu_wdata_i` | in | 写数据 |
| `lsu2exu_rdy_o` | out | 响应到达（= `dmem_resp_received`） |
| `lsu2exu_rdata_o` | out | 读回数据（已扩展） |
| `dmem_req_o` | out | 向 DMEM 发请求 |
| `dmem_cmd_o` | out | RD/WR |
| `dmem_width_o` | out | 访问宽度 |
| `dmem_addr_o` | out | 对齐后的地址 |
| `dmem_wdata_o` | out | 写数据 |
| `dmem_resp_i` | in | NOTRDY/RDY_OK/RDY_ER |
| `dmem_rdata_i` | in | DMEM 返回数据 |

## 关键等式

```systemverilog
lsu2exu_rdy_o = dmem_resp_received;  // :236
```

**这就是"LOAD 是多周期指令"的物理来源**——EXU 的 `exu_rdy` 在访存时以 LSU 收到响应为准。

## 访存完整路径

```
LSU 发出请求 → Router 按地址选端口 → TCM 单周期响应 或 Timer/AHB 慢速响应
→ 响应沿原路返回 → LSU 做符号扩展 → EXU 写回寄存器
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|
| `SCR1_TDU_EN` | 启用 TDU 地址匹配检查 |
| `SCR1_NO_EXE_STAGE` | 无影响（LSU 本身有时序） |

## 与其他模块关系

```
EXU → LSU: 访存请求（cmd/width/addr/wdata）
LSU → DMEM: memif 接口（req/ack/cmd/width/addr/wdata/rdata/resp）
LSU → TDU: 地址监控（ldst_addr）
LSU → EXU: 读回数据 + rdy + 异常
```

## 配套文档

- 设计规格：`docs/scr1-pipe-lsu/design-spec.md`
- 验证文档：`docs/scr1-pipe-lsu/verification.md`
