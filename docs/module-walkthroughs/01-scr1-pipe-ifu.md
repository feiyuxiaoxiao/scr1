# scr1_pipe_ifu.sv 讲解

## 一句话职责

取指单元：维护 PC、向指令内存发请求、把指令喂给 IDU，支持 RVC 压缩指令对齐。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_ifu.sv`
- **行数**: 826 行（MAX 配置）
- **配置依赖**: `SCR1_NO_DEC_STAGE`、`SCR1_NEW_PC_REG`、`SCR1_RVC_EXT`

## 核心功能分解

### 1. 指令队列（环形缓冲）

```systemverilog
localparam SCR1_IFU_Q_SIZE_WORD = 2;      // 2 个 32 位字
localparam SCR1_IFU_Q_SIZE_HALF = 4;      // 4 个半字（16 位）
```

**为什么按半字存储？**

RVC 压缩指令 16 位，RVI 指令 32 位。指令边界不与取指块（4 字节）对齐，必须用半字作为公共最小单位：
- 入队写宽度：`WR_HI`/`WR_FULL`（`:340-373`）
- 出队读宽度：`RD_HWORD`/`RD_WORD`（`:328-337`）
- 读指针步进：`+1`（RVC）/`+2`（RVI）（`:409-411`）

**关键判据**（`:279-280`）：
```systemverilog
assign instr_lo_is_rvi = &imem2ifu_rdata_i[1:0];     // 低半字 [1:0]==11
assign instr_hi_is_rvi = &imem2ifu_rdata_i[17:16];   // 高半字 [1:0]==11
```

规范出处：RISC-V Unprivileged ISA Manual — 32 位指令低两位恒 `11`；16 位压缩指令低两位只取 `00/01/10`。

### 2. 指令类型译码（`:282-302`）

| `{hi_is_rvi,lo_is_rvi}` | 含义 | instr_type |
|---|---|---|
| `00` | L、H 都不是 RVI 开头 | `RVC_RVC`（两个 16 位指令） |
| `10` | H 是某 RVI 的低半 | `RVI_LO_RVC`（L 完整 RVC + H 是跨块 RVI 低半） |
| `01`/`11` | L 是 RVI 低半 | `RVI_HI_RVI_LO`（完整 32 位 RVI） |

### 3. 跨块拼接状态（`:148`, `:308-322`）

```systemverilog
instr_hi_rvi_lo_next = (instr_type==RVI_LO_NV) | (instr_type==RVI_LO_RVI_HI) | (instr_type==RVI_LO_RVC);
```

**含义**：本块把 RVI 低半放进了高半字槽 → 该指令从块中部（2 对齐）开始、跨块边界，块内拼不完，置位让下一块用低半字槽来还账。

### 4. 非对齐 new_pc（`:262-274`）

分支/跳转目标可 2 字节对齐（bit1=1、bit0=0），而 IMEM 取指总是 4 字节块。new_pc 落在块中间时块低半字是废数据须丢弃：

```systemverilog
assign new_pc_unaligned_next = exu2ifu_pc_new_req_i ? exu2ifu_pc_new_i[1] : ...;  // :272-274
```

### 5. FSM（`:91-94`, `:477-488`）

```
IDLE: 等待 exu2ifu_pc_new_req_i
FETCH: 持续向 IMEM 发请求直到 pipe2ifu_stop_fetch_i 或错误待丢弃
```

MAX（无 `SCR1_NEW_PC_REG`）下 new_pc 请求绕过 FSM 直接驱动请求线（`:597`）。

### 6. IMEM 接口：两个计数器

- `imem_pnd_txns_cnt`（未完成事务，`:553`）：握手完成 `+1`、收响应 `-1`，全满（`:554`）停发请求。
- `imem_resp_discard_cnt`（待丢弃响应，`:569-591`）：两种丢弃来源——(1) new_pc 到来作废所有在飞指令；(2) 收到 IMEM 错误。

### 7. 错误注入队列（`:419-444`, `:724-733`）

**IMEM 出错时错误被"伪装成一条指令"注入队列**：
- ER 响应照样写满 2 槽（`:369-370`），数据是垃圾，但 `q_err[]` 与槽位绑定（`:429/:433`）一起流动；
- 出队时错误转为 `ifu2idu_imem_err_o`，IDU 收后不译码，直接生成 `exc_code=INSTR_ACCESS_FAULT(=1)` 命令（`scr1_pipe_idu.sv:134-137`），经 EXU 触发 CSR trap。

**为什么必须入队**：RISC-V 要求异常精确且绑定具体指令。错误作为一条指令的属性排进队列、逐槽流动，到 IDU/EXU 才翻牌成异常——异常码、出错 PC、顺序天然精确。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `ifu2idu_instr_o` | out | 给 IDU 的指令（RVC 低 16 位 / RVI 32 位） |
| `ifu2idu_vd` | out | 指令有效 |
| `idu2ifu_rdy_i` | in | IDU 就绪，握手成功则消费指令 |
| `ifu2imem_req_o` | out | 向 IMEM 发请求 |
| `imem2ifu_req_ack_i` | in | IMEM 确认请求 |
| `imem2ifu_rdata_i` | in | IMEM 返回数据 |
| `exu2ifu_pc_new_req_i` | in | EXU 要求换 PC（分支/跳转/异常/中断） |
| `exu2ifu_pc_new_i` | in | 新 PC 值 |
| `ifu2idu_imem_err_o` | out | IMEM 错误标志（随指令流动） |

## 输出多路选择器（MAX 分支，`:745-753`）

```systemverilog
ifu2idu_instr_o = q_head_is_rvc ? q_data_head : {q_data_next, q_data_head};
```

RVI 在队里低半先入（地址小）、高半相邻，读两口拼回 `{next, head}` 即完整指令（little-endian 指令字）。

## SVA 断言（`:769-822`）

| 断言名 | 验证内容 |
|---|---|
| `IFU_DRC_UNDERFLOW` | 丢弃计数不越界 |
| `IFU_DRC_RANGE` | 丢弃计数不越界（另一路） |
| `IFU_QUEUE_OVF` | 写满保护 |
| `IFU_IMEM_ERR_BEH` | ER 后进 IDLE 且丢弃计数收敛 |
| `IFU_NEW_PC_REQ_BEH` | 换 PC 后队列必空 |
| `IFU_IMEM_ADDR_ALIGNED` | 发地址 `[1:0]==0` |

## 配置宏影响

| 宏 | 定义时行为 | 未定义时行为 |
|---|---|---|
| `SCR1_NO_DEC_STAGE` | 无指令队列，IDU 与 EXU 直通 | 完整指令队列（MAX） |
| `SCR1_NEW_PC_REG` | new_pc 请求打一拍 | 直接驱动请求线（MAX） |
| `SCR1_RVC_EXT` | 支持 RVC 译码/对齐 | 纯 RVI 模式 |

## 与其他模块关系

```
IFU → IDU: ifu2idu_instr_o + ifu2idu_vd + ifu2idu_imem_err_o
IFU ← EXU: exu2ifu_pc_new_req_i + exu2ifu_pc_new_i（换 PC 请求）
IFU ↔ IMEM: 取指请求/响应（可能多周期延迟）
```

## 实测验证（VCD 波形）

`pipe2imem_addr_o` 领先 `curr_pc` 约 3 拍 = IFU 在 `FETCH` 态持续预取、指令囤进 4 槽队列。crt 中 `j`/`addi` 步长 2 与 4 交替 = `RD_HWORD`/`RD_WORD` 交替消费的波形表现。

## 配套文档

- 设计规格：`docs/scr1-pipe-ifu/design-spec.md`
- 验证文档：`docs/scr1-pipe-ifu/verification.md`
- 图形化：`docs/scr1-ifu-graphics/index.html`（9 张 Mermaid 图）
