# SCR1 指令取指单元（IFU）技术设计文档

> 对应源码: `src/core/pipeline/scr1_pipe_ifu.sv`
> 模块名: `scr1_pipe_ifu`
> 制定者: Syntacore LLC, 2016-2021

---

## 1. 总体架构

### 1.1 功能定位

IFU 是 SCR1 处理器流水线的第一级，负责从指令存储器（IMEM）获取指令并递交到指令译码单元（IDU）。核心职责：

1. 控制取指流程——从 IMEM 或调试 Program Buffer 获取指令
2. 处理 PC 未对齐（`PC[1]=1`），正确拼接跨边界的 RVI 指令
3. 通过指令队列缓冲取指与译码之间的速率差异
4. 支持旁路跳过队列（无译码阶段配置）
5. 响应流水线清空请求（PC 跳转、IMEM 错误）

### 1.2 模块划分

```
                          scr1_pipe_ifu
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌──────────────┐   ┌─────────────────┐   ┌──────────────────┐  │
│  │   IFU FSM    │   │  Instruction    │   │  IFU ↔ IDU       │  │
│  │   (2 states) │   │  Queue (4×16b)  │   │  Output MUX      │  │
│  └──────┬───────┘   └────────┬────────┘   └────────┬─────────┘  │
│         │                    │                      │            │
│         ▼                    ▼                      ▼            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   IFU ↔ IMEM Interface                   │   │
│  │  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐   │   │
│  │  │ Response     │ │ Address      │ │ Pending /        │   │   │
│  │  │ Logic        │ │ Register     │ │ Discard Counters │   │   │
│  │  └─────────────┘ └──────────────┘ └──────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 配置宏一览

| 宏名 | 作用 |
|------|------|
| `SCR1_DBG_EN` | 使能调试接口（HDU Program Buffer） |
| `SCR1_NO_DEC_STAGE` | 无独立译码阶段的流水线配置，开启指令旁路 |
| `SCR1_CLKCTRL_EN` | 使能时钟门控信号输出 |
| `SCR1_NEW_PC_REG` | 新 PC 是否通过额外一级寄存器（跳转惩罚换频率） |
| `SCR1_TRGT_SIMULATION` | 使能 SVA 断言（仅仿真） |

---

## 2. IFU FSM（取指有限状态机）

### 2.1 状态定义

```
                    ┌─────────────────────────────┐
                    │                             │
                    ▼                             │
              ┌───────────┐               ┌───────────────┐
     reset ──▶│   IDLE    │──────────────▶│    FETCH      │
              │ (不取指)  │ fetch_req     │   (取指中)    │
              └───────────┘               └───────────────┘
                    ▲                             │
                    │         stop_req            │
                    └─────────────────────────────┘
```

| 状态 | 含义 |
|------|------|
| `IDLE` | 空闲。不向 IMEM 发起请求，等待跳转触发 |
| `FETCH` | 取指。持续向 IMEM 发起请求，直到被停止 |

### 2.2 状态转移条件

**IDLE → FETCH**: 发生新 PC 请求 且 流水线未被停止

```systemverilog
assign ifu_fetch_req = exu2ifu_pc_new_req_i & ~pipe2ifu_stop_fetch_i;
```

**FETCH → IDLE**: 流水线被停止 或 收到 IMEM 错误响应且尚未丢弃

```systemverilog
assign ifu_stop_req = pipe2ifu_stop_fetch_i
                    | (imem_resp_er_discard_pnd & ~exu2ifu_pc_new_req_i);
```

注意：收到 IMEM 错误后回到 IDLE 的原因是——需要等待所有在途的错误响应被丢弃计数消化完毕，期间不发起新请求。

---

## 3. 指令队列

### 3.1 物理结构

```
      slot 0              slot 1              slot 2              slot 3
  ┌──────────┬───┐   ┌──────────┬───┐   ┌──────────┬───┐   ┌──────────┬───┐
  │ q_data[0]│err│   │ q_data[1]│err│   │ q_data[2]│err│   │ q_data[3]│err│
  │  (16b)   │(1b)│   │  (16b)   │(1b)│   │  (16b)   │(1b)│   │  (16b)   │(1b)│
  └──────────┴───┘   └──────────┴───┘   └──────────┴───┘   └──────────┴───┘
```

- 容量: 4 个半字槽 (16 bit × 4 = 2 × 32-bit 字)
- 每槽附带 1 bit 错误标志 `q_err[n]`
- 物理寻址: 2 bit 地址 (`[1:0]`)
- 读写指针: 3 bit（额外 1 位用于环形空满判定）

### 3.2 以半字为粒度的设计原因

RVC 指令占 16 位，RVI 指令占 32 位。如果以字（32 位）为槽，存储 RVC 时会浪费一半空间。以半字为最小粒度，RVC 和 RVI 的高/低半可以自由混排。

### 3.3 读/写粒度决策

**写入粒度** 由 `instr_type` 决定（参见第 4 节）：

| `instr_type` | 写粒度 | 写入内容 |
|--------------|:---:|----------|
| `RVI_HI_RVI_LO` / `RVC_RVC` / `RVI_LO_RVC` / `RVC_RVI_HI` / `RVI_LO_RVI_HI` | FULL (2 半字) | 完整 32 位 |
| `RVC_NV` / `RVI_LO_NV`（PC 未对齐） | HI (1 半字) | 仅高 16 位 |
| `NONE`（丢弃/错误） | NONE | — |

**读取粒度** 由队首指令类型决定：

| 队首类型 | 读粒度 | 理由 |
|---------|:---:|------|
| RVC 或错误 | HWORD (1 半字) | RVC 只需 16 位 |
| RVI | WORD (2 半字) | RVI 需要拼 2 个半字 |

### 3.4 环形队列空/满判定

读写指针位宽 `SCR1_IFU_QUEUE_PTR_W = 3`（地址位宽 `2 + 1`）：

- **判空**: `q_rptr == q_wptr`（指针数值相等）
- **判满**: 不直接判满，通过 `q_ocpd_h`（占用度）与容量比较
- **占用度**: `q_ocpd_h = q_wptr - q_rptr`，由额外 1 位圈数位保证差值范围 0~4

### 3.5 队列清空（Flush）

两种触发条件：

```
q_flush_req = exu2ifu_pc_new_req_i | pipe2ifu_stop_fetch_i
```

清空时读写指针同时归零。PC 跳转后旧指令全部废弃，停止取指时也不应残留数据。

### 3.6 预测性空间检查

```systemverilog
q_free_w_next = (4 - (q_wptr - q_rptr_next)) >> 1
q_has_free_slots = (q_free_w_next > imem_vd_pnd_txns_cnt)
```

用**下一拍读指针**（`q_rptr_next`）而非当前值计算剩余空间。原因是：读指针下一拍可能已经前移（当前周期同时有读操作），此处用保守估计，确保即使读操作发生，队列仍有足够空间容纳所有在途响应。

---

## 4. 指令分布类型译码

### 4.1 RVI vs RVC 判定

RISC-V 指令编码中，`[1:0] = 2'b11` 标识 32 位指令（RVI），否则为 16 位压缩指令（RVC）：

```systemverilog
assign instr_hi_is_rvi = &imem2ifu_rdata_i[17:16];  // 高16位 ∈ RVI
assign instr_lo_is_rvi = &imem2ifu_rdata_i[1:0];    // 低16位 ∈ RVI
```

### 4.2 指令分布译码真值表

`instr_type` 由 `{new_pc_unaligned_ff, instr_hi_rvi_lo_ff, instr_hi_is_rvi, instr_lo_is_rvi}` 决定：

| `unaligned` | `hi_prev_rvi_lo` | `hi_is_rvi` | `lo_is_rvi` | `instr_type` |
|:---:|:---:|:---:|:---:|------|
| 1 | - | 1 | - | `RVI_LO_NV` |
| 1 | - | 0 | - | `RVC_NV` |
| 0 | 1 | 1 | 1 | `RVI_LO_RVI_HI` |
| 0 | 1 | 1 | 0 | `RVC_RVI_HI` |
| 0 | 0 | 0 | 0 | `RVC_RVC` |
| 0 | 0 | 1 | 0 | `RVI_LO_RVC` |
| 0 | 0 | - | 1 | `RVI_HI_RVI_LO` |
| - | - | - | - | `NONE`（imem_resp 无效时） |

### 4.3 跨边界状态记忆

`instr_hi_rvi_lo_ff` 寄存器记录"上一轮 IMEM 返回的高 16 位是 RVI 的低半部分"。这是必要的状态记忆，因为一个 RVI 指令可能被 32 位字边界切开：

```
   字 N-1                  字 N                  字 N+1
┌─────────┬─────────┐ ┌─────────┬─────────┐
│ RVC     │ RVI_LO  │ │ RVI_HI  │ RVC     │
│ [15:0]  │ [31:16] │ │ [15:0]  │ [31:16] │
└─────────┴─────────┘ └─────────┴─────────┘
                   ↑                    ↑
            这一半是 RVI 低半       这一半是 RVI 高半
            需要记住，下次拼接      和上次记住的低半拼成完整指令
```

置位条件（三种情形都会导致下一轮需要跨边界拼接）：

```systemverilog
instr_hi_rvi_lo_next = (instr_type == RVI_LO_NV)
                     | (instr_type == RVI_LO_RVI_HI)
                     | (instr_type == RVI_LO_RVC);
```

清零条件：新 PC 跳转时强制清零（跨边界状态属于旧 PC 流，跳过无效），或正常 IMEM 响应时根据 `instr_hi_rvi_lo_next` 更新。

---

## 5. IMEM 接口

### 5.1 请求生成

以无 `SCR1_NEW_PC_REG` 版本为例：

```systemverilog
ifu2imem_req_o = (exu2ifu_pc_new_req_i & ~imem_pnd_txns_q_full & ~pipe2ifu_stop_fetch_i)
               | (ifu_fsm_fetch        & ~imem_pnd_txns_q_full & q_has_free_slots);
```

两个来源（通过 OR 合并）：
1. **新 PC 跳转**: 跳转请求优先级最高，当拍发出。受限于 `~q_full`（流水线满不发）和 `~pipe2ifu_stop_fetch_i`（暂停时不发）。
2. **正常取指**: FSM 在 FETCH 状态时持续发出。受限于 `~q_full` 和 `q_has_free_slots`（队列必须有足够空间，包含对在途事务的预测）。

### 5.2 地址生成

地址寄存器 `imem_addr_ff` 存储字地址 `[XLEN-1:2]`（去掉低 2 位）：

```systemverilog
imem_addr_next = exu2ifu_pc_new_req_i ? exu2ifu_pc_new_i[XLEN-1:2] + imem_handshake_done
                                      : imem_addr_ff + imem_handshake_done;
```

- 新 PC 时: 地址设为新 PC 的字地址，若同拍握手完成则 +1（因请求已发出）
- 正常取指时: 每握手成功一次地址 +1（即 +4 字节）

IMEM 输出地址补回低 2 位：

```systemverilog
ifu2imem_addr_o = {imem_addr_ff, 2'b00};
```

### 5.3 在途事务跟踪

```
                    +1                        -1
               ┌─────────┐              ┌─────────┐
               │ 握手完成 │              │ 收到响应 │
               └────┬────┘              └────┬────┘
                    ▼                        ▼
              ┌─────────────────────────────────────┐
              │       imem_pnd_txns_cnt (0~7)        │
              └─────────────────────────────────────┘
                          │              │
                    ┌─────┴─┐      ┌─────┴──────────┐
                    ▼       ▼      ▼                ▼
               q_full  有效在途数  丢弃计数器初始值  时钟门控
```

- **溢出保护**: `imem_pnd_txns_q_full = 1` 时停止发新请求
- **有效在途数**: `imem_vd_pnd_txns_cnt = imem_pnd_txns_cnt - imem_resp_discard_cnt`，用于队列容量预测
- **XNOR 使能**: `imem_pnd_txns_cnt_upd = imem_handshake_done ^ imem_resp_received`，仅当单边发生时更新寄存器

### 5.4 响应丢弃机制

当发生 PC 跳转或 IMEM 错误时，流水线中所有未返回的响应都不可用：

```systemverilog
// PC 跳转时：丢弃计数 = 在途事务数 - 同拍握手的补偿
imem_resp_discard_cnt_next = exu2ifu_pc_new_req_i ? imem_pnd_txns_cnt_next - imem_handshake_done
// IMEM 错误时（且错误尚未丢弃）: 丢弃计数 = 全部在途事务数
                          : imem_resp_er_discard_pnd ? imem_pnd_txns_cnt_next
// 正在丢弃中: 每收到一个响应减 1
                          : imem_resp_discard_cnt - 1;
```

丢弃窗口期间：

```systemverilog
imem_resp_discard_req = |imem_resp_discard_cnt;  // 丢弃计数 > 0
imem_resp_vd = imem_resp_received & ~imem_resp_discard_req;  // 有效响应被屏蔽
```

所有经过 `imem_resp_vd` 的下游逻辑（指令类型译码、队列写入）都自动跳过被丢弃的响应。

---

## 6. 指令旁路（Bypass）

### 6.1 触发条件（仅 `SCR1_NO_DEC_STAGE`）

当没有独立译码阶段时，IFU 可以直接将指令送到 IDU 而不经过队列，减少延迟：

| 旁路类型 | 触发条件 | 数据来源 |
|----------|----------|----------|
| `BYPASS_RVC` | 队列空 + 新指令为 RVC | IMEM 高/低 16 位 |
| `BYPASS_RVI_RDATA` | 队列空 + 新指令为完整 RVI | IMEM 32 位数据 |
| `BYPASS_RVI_RDATA_QUEUE` | 队列有 1 个半字且是 RVI 头 + 新指令可拼接 | IMEM 低 16 位 + 队首半字 |
| `BYPASS_NONE` | 其他情况 | 走队列路径 |

### 6.2 旁路对队列写入的影响

旁路有效时 (`instr_bypass_vd & idu2ifu_rdy_i`)，指令直接送入 IDU，不写入队列：

| instr_type | 无旁路写入 | 有旁路写入 |
|------------|:---:|:---:|
| `RVC_NV` | HI | NONE |
| `RVI_HI_RVI_LO` | FULL | NONE |
| 其他跨边界类型 | FULL | HI（仅写需要暂存的半字） |

---

## 7. IDU 输出接口

### 7.1 指令拼接

输出到 IDU 的 32 位指令按以下方式拼接：

```
RVC 场景:     { 16'b0 , q_data_head }
               └─高16位补0──┘└─队首16位──┘

RVI 场景:     { q_data_next , q_data_head }
               └─队首+1（高半）──┘└─队首（低半，含[1:0]=11）┘

旁路 RVC:     { 16'b0 , 高/低16位 }  （由 new_pc_unaligned 选择）
旁路 RVI:     IMEM 原始返回数据（32位）
旁路 拼接:    { imem_rdata_lo , q_data_head }
```

### 7.2 错误信息传递

IFU 输出两条错误信号给 IDU：

| 信号 | 含义 | 触发条件 |
|------|------|----------|
| `ifu2idu_imem_err_o` | 指令访问异常 | IMEM 返回错误响应 |
| `ifu2idu_err_rvi_hi_o` | RVI 高半取指错误 | 队首是指令头（低半）正常、但队首+1（高半）有错误标志 |

两条错误信号独立的原因: RVI 指令跨两个半字槽，可能只有高半槽出错，需要精确定位。

### 7.3 有效信号生成

`ifu2idu_vd_o` 决定 IDU 是否能采集当前指令：

- **旁路有效**: `vd = 1`
- **队列非空 + 只有 1 个半字**: `vd = q_head_is_rvc | q_err_head`（RVC 可直接送出，有错误也必需送出）
- **队列非空 + 有 ≥2 个半字**: `vd = 1`（有完整的半字对，可直接拼成 32 位指令）

### 7.4 反压机制

IDU 通过 `idu2ifu_rdy_i` 反压 IFU。当 `idu2ifu_rdy_i = 0` 时：
- `q_rd_vd = 0`，队列读指针不前进，数据保持在队列中
- 旁路写入判断中的 `idu2ifu_rdy_i` 为 0，数据改为写入队列

---

## 8. 调试接口

由 `SCR1_DBG_EN` 控制。支持 HDU 通过 Program Buffer 向流水线注入调试指令：

- 当 `hdu2ifu_pbuf_fetch_i = 1` 时，IFU 输出完全由 Program Buffer 驱动
- `ifu2idu_instr_o` 被覆写为 `hdu2ifu_pbuf_instr_i`（零扩展至 32 位）
- `ifu2idu_vd_o` 覆写为 `hdu2ifu_pbuf_vd_i`
- `ifu2idu_imem_err_o` 覆写为 `hdu2ifu_pbuf_err_i`
- `ifu2hdu_pbuf_rdy_o` 直通 `idu2ifu_rdy_i`（IDU 就绪时 IFU 即可接受调试指令）

---

## 9. 时钟门控支持

由 `SCR1_CLKCTRL_EN` 控制：

```systemverilog
ifu2pipe_imem_txns_pnd_o = |imem_pnd_txns_cnt;
```

当存在未完成 IMEM 事务时，此信号为 1，时钟管理单元不能关闭 IFU 时钟——否则在途响应无法被接收，状态机锁死。

---

## 10. 关键设计决策与分析

### 10.1 指令队列 vs 直通设计

队列缓冲了 IFU 和 IDU 之间的速率差异。IDU 暂时无法接收时（`idu2ifu_rdy_i=0`），取指可以继续进行直到队列填满，避免流水线气泡。

### 10.2 半字粒度存储 vs 字粒度

以半字为粒度避免了 RVC 的空间浪费。代价是读写逻辑需要区分 HWORD/WORD 两种粒度，增加了指针和状态机的复杂度。但 RISC-V 混编场景下（RVI 和 RVC 交替出现）这是必要的。

### 10.3 `SCR1_NEW_PC_REG` 的时序频率权衡

| 维度 | 无此宏 | 有此宏 |
|------|--------|--------|
| 跳转到首次取指 | 0 周期 | 1 周期 |
| IMEM 地址路径 | EXU→MUX→IFU→IMEM（长组合路径） | 寄存器→IFU→IMEM（短路径） |
| 最高频率 | 受限于 EXU→IMEM 组合路径 | 更高 |
| 接口复杂度 | `req_o` 有 MUX，`addr_next` 需补偿 `handshake_done` | 单一来源，逻辑简洁 |

### 10.4 丢弃计数器 vs 即时清空

PC 跳转时，如果不设计丢弃计数器而试图"撤回"已发出的请求，需要修改 IMEM 接口语义（需要 abort 信号）。保持 IMEM 接口简单的前提下，丢弃计数器是代价最低的方案——只需多一个 3 位计数器和一个门控信号，就实现了"不看他、不用他、不理他"的效果。

### 10.5 预测性空间检查

用 `q_rptr_next` 而非 `q_rptr` 计算可用空间，确保即使当前周期读指针会前进，也以保守的最低可用空间做判断。这是一种避免欠估计导致溢出的防御性设计。

---

## 11. 验证断言概要

8 条 SVA 断言覆盖以下检查维度：

- **X 态检测**: 输入信号、IMEM 请求输出信号不能有未知值
- **计数器边界**: 丢弃计数器不下溢、不超过在途事务数
- **队列安全**: 队列不上溢（写入粒度受限于剩余容量）
- **行为正确性**: IMEM 错误后回 IDLE、新 PC 后队列清空、停止取指后 FSM 回 IDLE
- **接口约束**: IMEM 地址字对齐、`err_rvi_hi` 隐含 `imem_err`
