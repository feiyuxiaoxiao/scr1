# SCR1 IFU 取指单元设计规格书（Design Specification）

- 模块：`scr1_pipe_ifu`
- 源文件：`src/core/pipeline/scr1_pipe_ifu.sv`（826 行）
- 位置：SCR1 RV32 内核流水线顶层 `scr1_pipe_top` 的第一级
- 版本：基于 SCR1 开源版本（Syntacore），文档对应 2026-09 学习记录

> 本文档描述 IFU 的功能规格、接口、微架构、时序与验证边界，行号均指向源文件。
> 全文以 `SCR1_CFG_RV32IMC_MAX`（不定义 `SCR1_NO_DEC_STAGE` / `SCR1_NEW_PC_REG`）
> 为主线，配置相关的分支单独标注。

---

## 目录

1. 概述与设计目标
2. 外部接口
3. 内部结构（微架构）
4. 功能规格
5. 配置宏对行为的影响
6. 时序行为示例
7. 断言规格
8. 假设与约束
- 附录 A 端口清单
- 附录 B 局部参数与类型
- 附录 C 内部信号清单
- 附录 D 一条指令的生命周期
- 附录 E 关键信号 → 消费者映射

---

## 1. 概述与设计目标

IFU（Instruction Fetch Unit，指令取指单元）是流水线的第一级。它负责：

1. 维护取指 PC 并向 IMEM（指令内存）发起读请求；
2. 接收 IMEM 的 32 位响应数据，**将指令流切分为 16 位（RVC）或 32 位（RVI）指令**；
3. 将解析出的指令送入 IDU，同时携带取指异常信息；
4. 响应 EXU 发来的控制转移请求（分支/跳转/mret/trap），实现换 PC；
5. 处理跨取指块边界的指令拼接（RVC 变长指令带来的对齐问题）。

IFU 不包含任何"架构可见状态"——PC 变化全部来自 EXU/CSR 通过 `exu2ifu_pc_new_i` 的注入，IFU 自身不产生跳转，只按顺序取指并处理 flush。

### 1.1 设计权衡

- **队列 + 对齐处理是支持 RVC 压缩指令付出的代价**：若纯 RV32I，指令 4 字节对齐，取指块即指令，IFU 无需队列与拼接逻辑。
- IMEM 采用 **请求-应答 + 多拍响应** 协议，用 pending 计数器支持"请求已发、响应未归"的在飞事务。
- 错误响应（总线错误）不即时处理，而是伪装成指令注入队列随流水前进，保证 RISC-V 精确异常语义。
- 队列只有 2 个 32 位字（4 个半字槽），与"最多 2 笔在飞事务"配套，用最小存储换取吞吐。

### 1.2 术语与命名约定

| 术语 | 含义 |
|---|---|
| 取指块（fetch block） | 一次 IMEM 读响应返回的 4 字节（32 位）数据 |
| 半字（halfword） | 16 位单元，队列的最小存储粒度 |
| 低半字 / 高半字 | 取指块内 `rdata[15:0]` / `rdata[31:16]`，低半字地址较小 |
| RVI | 32 位指令，低 2 位恒为 `11` |
| RVC | 16 位压缩指令，低 2 位为 `00/01/10` |
| 在飞事务（pending txn） | 请求已握手、响应尚未返回的 IMEM 读事务 |
| 废单（discard） | 因换 PC 或取指错误而被判定作废的在飞事务 |
| 欠账 | 上一条 RVI 的高半字落在当前取指块低半字，尚未配齐 |

**指令组合命名规则**：`SCR1_IFU_INSTR_<高半字内容>_<低半字内容>`（`:118`）。
四个内容代号：`RVI_LO`（某 RVI 的低半）、`RVI_HI`（某 RVI 的高半）、
`RVC`（独立压缩指令）、`NV`（No Valid，无效）。

---

## 2. 外部接口（Port List）

共 22 个端口（含条件编译）。完整清单见附录 A，本节给出分组与语义。

### 2.1 控制信号

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `rst_n` | in | 复位（低有效） | 36 |
| `clk` | in | 时钟 | 37 |
| `pipe2ifu_stop_fetch_i` | in | 停止取指（WFI 停机 / 调试取指） | 38 |

### 2.2 IFU <-> IMEM（指令内存接口）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `imem2ifu_req_ack_i` | in | 请求应答 | 41 |
| `ifu2imem_req_o` | out | 读请求 | 42 |
| `ifu2imem_cmd_o` | out | 命令（固定 `READ`） | 43 |
| `ifu2imem_addr_o` | out | 取指地址（`[31:2]+2'b00`，字对齐） | 44 |
| `imem2ifu_rdata_i` | in | 读数据 32 位 | 45 |
| `imem2ifu_resp_i` | in | 响应（`NOTRDY`/`RDY_OK`/`RDY_ER`） | 46 |

三段式协议：`req` 提出 → `ack` 接受（握手）→ 后续某拍 `resp` 返回。
握手式 `imem_handshake_done = ifu2imem_req_o & imem2ifu_req_ack_i`（`:513`）。

### 2.3 IFU <-> EXU（新 PC 接口）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `exu2ifu_pc_new_req_i` | in | 换 PC 请求（jump/branch/trap） | 49 |
| `exu2ifu_pc_new_i` | in | 新 PC 值 | 50 |

这是 IFU 唯一的"指令流方向改变"来源。IFU 不产生跳转，只被动接受。

### 2.4 IFU <-> HDU（调试，条件编译）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `hdu2ifu_pbuf_fetch_i` | in | 从程序缓冲取指 | 54 |
| `ifu2hdu_pbuf_rdy_o` | out | 程序缓冲就绪 | 55 |
| `hdu2ifu_pbuf_vd_i` | in | 程序缓冲指令有效 | 56 |
| `hdu2ifu_pbuf_err_i` | in | 程序缓冲错误 | 57 |
| `hdu2ifu_pbuf_instr_i` | in | 程序缓冲指令 | 58 |

仅在 `SCR1_DBG_EN` 下存在。MAX 配置启用。

### 2.5 IFU <-> 时钟控制（条件编译）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `ifu2pipe_imem_txns_pnd_o` | out | 存在在飞 IMEM 事务（门控时钟用） | 62 |

### 2.6 IFU <-> IDU（指令输出接口）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `idu2ifu_rdy_i` | in | IDU 就绪（可接收） | 66 |
| `ifu2idu_instr_o` | out | 指令（16/32 位） | 67 |
| `ifu2idu_imem_err_o` | out | 取指异常（instruction access fault） | 68 |
| `ifu2idu_err_rvi_hi_o` | out | 取指 RVI 高半字错误 | 69 |
| `ifu2idu_vd_o` | out | 指令有效 | 70 |

### 2.7 握手约定

SCR1 流水级间采用 **valid-ready** 单握手：

- 有效侧拉 `*_vd_o`，接收侧就绪拉 `*_rdy_i`；
- 两者同时为 1 的拍，数据交接完成。

IFU 侧，`ifu2idu_vd_o & idu2ifu_rdy_i` 才真正消费一条指令（出队，`:328`）。

### 2.8 IFU↔IDU 端口语义详解

这 5 个端口构成取指流水最后一段的契约。IDU 是**纯组合译码级**，没有缓冲，
其 ready 与 valid 均为对 EXU 的透传（`scr1_pipe_idu.sv:80-82`）：

```
assign idu2ifu_rdy_o  = exu2idu_rdy_i;    // EXU 的 ready 直接反压 IFU
assign idu2exu_req_o  = ifu2idu_vd_i;     // IFU 的 valid 直接转发 EXU
assign instr          = ifu2idu_instr_i;  // 指令直接进入译码逻辑
```

因此整条握手链路的实质是 **IFU（生产者）↔ EXU（消费者）**，IDU 只做中继。

| 端口 | 语义 |
|---|---|
| `ifu2idu_instr_o` | 指令本体。RVC 为 16 位有效数据（高 16 位补 0），RVI 为完整 32 位。指令长度由 IDU 依据 `instr[1:0]` 自行判定，接口不额外传递长度 |
| `ifu2idu_vd_o` | 本拍指令有效。队头是 RVC 或带错标志即可用；队头是 RVI 但高半未到（`q_has_1_ocpd_hw`）时为 0，等待下一拍 |
| `idu2ifu_rdy_i` | 反向流控。为 1 时 IFU 才推进读指针（`q_rd_vd = ~q_is_empty & ifu2idu_vd_o & idu2ifu_rdy_i`，`:328`） |
| `ifu2idu_imem_err_o` | 本拍指令的取指发生访问错误。与 `instr_o` 同拍有效；IDU 收到后不译码其字段，直接产生 `INSTR_ACCESS_FAULT`（`scr1_pipe_idu.sv:134-137`） |
| `ifu2idu_err_rvi_hi_o` | 错误的**位置**标志：错误发生在非对齐 RVI 的高半时置 1。仅在 `imem_err=1` 时有意义，平时恒 0。完整链路见 §4.5.3 |

**握手时序示例**（队列中已有一条 RVI、一条 RVC）：

```
周期           T0    T1    T2
ifu2idu_vd_o   1     1     0     ← T2 队列空
idu2ifu_rdy_i  1     0     1
传输成立       Y     N     -
q_rd_vd        1     0     0
q_rd_size      WORD  -     -     ← T0 弹 2 槽（RVI）
```

T1 时 `rdy=0`（下游未就绪），IFU 保持 `instr_o` 与指针不变，T2 继续尝试。

### 2.9 与 IMEM 接口的对比

| 维度 | IFU↔IDU | IFU↔IMEM |
|---|---|---|
| 协议 | valid/ready 单握手 | req/ack/response 三段式 |
| 时延 | 数据与 valid 同拍 | 请求、ack、响应可跨多拍 |
| 缓冲 | 队列在 IFU 内 | IFU 侧无数据缓冲，靠在飞计数记账 |
| 反压 | `idu2ifu_rdy_i` 直接反压队列出队 | `q_has_free_slots` 间接通过请求准入 |

---

## 3. 内部结构（微架构）

### 3.1 结构框图

```
                 ┌────────────────────────────────────────────┐
  IMEM 响应 ───▶ │ IMEM接口    指令类型译码   指令队列(4×16bit) │
   (rdata/resp)  │ resp译码   instr_type     q_data/q_err      │
                 │ req生成                   环形 rptr/wptr     │
                 │ addr_ff      │                 │             │
                 │              ▼                 ▼             │
                 │       跨块欠账标志     队列状态/输出多路选择   │
                 │       instr_hi_rvi_lo_ff       │             │
                 │              ▲                 ▼             │
                 │  IFU FSM     │    ifu2idu_instr_o ──▶ IDU    │
                 │  IDLE/FETCH  │                              │
                 └──────────────────────────────────────────────┘
    新PC请求/新PC ◀────────────────────────────────────────────────
```

### 3.2 关键常量（`:77-85`）

| 常量 | 值 | 说明 |
|---|---|---|
| `SCR1_IFU_Q_SIZE_WORD` | 2 | 队列能存 2 个 32 位字 |
| `SCR1_IFU_Q_SIZE_HALF` | `SIZE_WORD*2` = 4 | 等价 4 个半字槽位 |
| `SCR1_TXN_CNT_W` | 3 | 在飞/丢弃计数器位宽（可数 0~7） |
| `SCR1_IFU_QUEUE_ADR_W` | `$clog2(4)` = 2 | 队列槽地址位宽 |
| `SCR1_IFU_QUEUE_PTR_W` | `ADR_W+1` = 3 | 指针位宽（多 1 位用于判空） |
| `SCR1_IFU_Q_FREE_H_W` | `$clog2(4+1)` = 3 | 空闲半字计数位宽（0~4） |
| `SCR1_IFU_Q_FREE_W_W` | `$clog2(2+1)` = 2 | 空闲字计数位宽（0~2） |

`+1` 是为了让计数能表示"全空/全满"的上界值本身。

### 3.3 数据通路总览（一条指令的生命周期）

```
IMEM 响应 (32 位)
  ① 提取两个原始判据位：
       instr_lo_is_rvi = &rdata[1:0]        // 低半字起始 32 位指令
       instr_hi_is_rvi = &rdata[17:16]      // 高半字起始 32 位指令
  ② 结合两个上下文寄存器定出 instr_type（8 类之一）
       new_pc_unaligned_ff / instr_hi_rvi_lo_ff
  ③ 写大小译码 → 入队 0/1/2 个半字，错误位随数据同写
       （旁路配置下扣除已直送 IDU 的部分）
  ④ 读侧按队头类型出队：RVC/错误 → 1 槽，RVI → 2 槽
  ⑤ 输出多路选择拼出 16/32 位指令 → ifu2idu_instr_o
```

### 3.4 内部信号分组（`:129-243`）

| 分组 | 行号 | 内容 |
|---|---|---|
| 指令队列信号 | 133-190 | 非对齐标志、类型译码、欠账标志、读写大小、指针、队列数据/错误、状态 |
| IFU FSM 信号 | 192-201 | `ifu_fetch_req`/`ifu_stop_req`、状态寄存器 |
| IMEM 信号 | 203-233 | 响应译码、地址寄存器、在飞计数、丢弃计数 |
| 条件编译信号 | 235-243 | `new_pc_req_ff`、旁路类型/有效 |

---

## 4. 功能规格

### 4.1 指令队列：按半字存储的环形缓冲

#### 4.1.1 为什么按半字存

RVC 使指令长度可变（16/32 位），取指块（32 位）与指令边界不再对齐。半字是
RVC(1)、RVI(2)、取指块(2) 的公共最小单位，因此队列按 16 位槽存储。

**关键场景——跨块 RVI**：一条 RVI 若起始于 2 对齐（非 4 对齐）地址，其低 16 位落在
块 A 高半槽、高 16 位落在块 B 低半槽，两次取指响应分别到达。半字存储使两个半字可在
不同拍入队并在队列中拼接。

#### 4.1.2 存储结构（`:172-180`）

- `q_data [4]`：16 位数据槽，`SCR1_IFU_Q_SIZE_HALF=4`；
- `q_err [4]`：槽对应的错误标志（每槽 1 bit）。

两块存储同步写入、独立读出。

#### 4.1.3 写入块与写入顺序（`:419-439`）

```
q_wr_en = imem_resp_vd & ~q_flush_req;      // 有效响应且不在冲刷拍

case (q_wr_size)
  WR_HI   : q_data[wptr]   <= imem_rdata_hi;  q_err[wptr]   <= imem_resp_er;
  WR_FULL : q_data[wptr]   <= imem_rdata_lo;  q_err[wptr]   <= imem_resp_er;
            q_data[wptr+1] <= imem_rdata_hi;  q_err[wptr+1] <= imem_resp_er;
endcase
```

**顺序约定**：槽位顺序代表程序地址递增，因此**先写低半字 `lo`（较小地址），
再写高半字 `hi`**。

以 `RVC_RVI_HI` 为例验算：低半字是上一条 RVI 的高半（地址较小），高半字是新 RVC
（地址较大）。`WR_FULL` 把 `lo`(RVI_HI) 写进 `wptr`、`hi`(RVC) 写进 `wptr+1`，
正好是"RVI 先、RVC 后"的程序顺序。`RVI_LO_RVC` 同理（低位 RVC 在前、高位那条
RVI 随后）。

`q_err` 与数据同步写；`imem_resp_er` 是整次响应的属性，故 `WR_FULL` 下两个槽位
写入同一错误标志。

**`WR_HI` 的语义**：只把**高半字**写入当前 `wptr`。用于 `RVC_NV`/`RVI_LO_NV`
（新 PC 落在块中部，低半字无效）以及旁路配置下"低半字已直送 IDU"的场景。

#### 4.1.4 读侧组合读出（`:441-444`）

```
q_data_head = q_data[ADR_W'(q_rptr)];          // 队头槽
q_data_next = q_data[ADR_W'(q_rptr + 1'b1)];   // 次槽
q_err_head  = q_err [ADR_W'(q_rptr)];
q_err_next  = q_err [ADR_W'(q_rptr + 1'b1)];
```

- 纯组合读出，无额外寄存器；
- `q_rptr` 是 3 位指针，数组只有 4 槽，索引时用 `ADR_W'(...)` 取低 2 位（`mod 4`）；
- 指针最高位不参与寻址，专用于判空；
- `q_rptr + 1'b1` 的 `+1` 在指针宽度内回绕（`7+1→0`）；
- `q_data_next`/`q_err_next` 在队头是 RVI 时用于拼高半与归属错误。

#### 4.1.5 读写指针（`:378-411`）

- `q_flush_req = exu2ifu_pc_new_req_i | pipe2ifu_stop_fetch_i`（`:381`）；
- **写指针** `q_wptr`：`q_wptr_upd = q_flush_req | ~q_wr_none`（`:384`）。
  正常步进：写半字 `+1`、写满字 `+2`；flush 归零（`:394-396`）。
- **读指针** `q_rptr`：`q_rptr_upd = q_flush_req | ~q_rd_none`（`:399`）。
  步进由读大小决定：读半字 `+1`、读整字 `+2`；flush 归零（`:409-411`）。
- 步进常量写成 `SCR1_IFU_QUEUE_PTR_W'('b001)` / `'b010)`，即显式位宽转换，
  使加法在 3 位内进行、进位自然丢弃，得到环形队列所需的 `mod 8` 回绕。

#### 4.1.6 队列状态逻辑（`:446-458`）

**占用与空闲计数**

| 信号 | 表达式 | 含义 |
|---|---|---|
| `q_ocpd_h` | `FREE_H_W'(q_wptr - q_rptr)` | 当前已占用半字数（0~4） |
| `q_free_h_next` | `FREE_H_W'(SIZE_HALF - (q_wptr - q_rptr_next))` | 下一拍空闲半字数 |
| `q_free_w_next` | `FREE_W_W'(q_free_h_next >> 1)` | 下一拍空闲字数（0~2） |

- `q_wptr - q_rptr` 在 3 位内相减，依赖"队列永不溢出"这一不变量，差值不会超过 4；
- `q_free_h_next` 用 **`q_rptr_next`（下一拍读指针）**计算：本拍 `q_rd_vd` 可能
  推进读指针，故"下拍空闲量 = 4 − 下拍占用量"，用于判断下拍能否再发请求；
- `q_free_w_next` 右移一位等价除 2（2 半字 = 1 字），向下取整偏保守。

**布尔查询**

| 信号 | 表达式 | 含义 |
|---|---|---|
| `q_is_empty` | `q_rptr == q_wptr` | 队列空（指针**全等**判定） |
| `q_has_free_slots` | `TXN_CNT_W'(q_free_w_next) > imem_vd_pnd_txns_cnt` | 还能再发取指请求 |
| `q_has_1_ocpd_hw` | `q_ocpd_h == FREE_H_W'(1)` | 恰好只占 1 个半字 |
| `q_head_is_rvi` | `&q_data_head[1:0]` | 队头是 RVI 低半（低 2 位为 11） |
| `q_head_is_rvc` | `~q_head_is_rvi` | 队头是独立 RVC |

- **`q_is_empty` 用全等判定**：指针比槽地址多 1 位，正是为了区分"空"与"满"——
  若只有 2 位，两者低位都相等而无法区分；多出的最高位让"空"成为唯一确定的指针全等状态。
- **`q_has_free_slots` 是"预留座位"判据**：左边是下拍队列可容纳的**字数**，
  右边是已发出但尚未入队的**有效**在飞事务数（`pnd − discard`，`:590`）。
  必须**严格大于**才准入，这样最坏情况下所有在飞响应都能落进队列，队列绝不溢出。
  以字而非半字为单位是保守处理——一个在飞请求可能返回整字（RVI）。
- **`q_has_1_ocpd_hw` 处理"半条指令待续"**：队列只剩 1 个半字时，若队头是 RVI
  低半，其高半未到、数据不完整，不能整字出队；若队头是 RVC 或错误标志，
  半字即可成条。读侧据此区分（见 §4.5）。
- **`q_head_is_rvi` 复用同一判据**：与类型译码的 `instr_lo_is_rvi`（`:280`）同一规则。

### 4.2 指令类型译码

#### 4.2.1 判据基础（`:279-280`）

RISC-V 长度编码规则：32 位指令低 2 位恒为 `11`；压缩指令只用 `00/01/10`。
据此：

```
instr_hi_is_rvi = &imem2ifu_rdata_i[17:16]   // 高半字的低 2 位
instr_lo_is_rvi = &imem2ifu_rdata_i[1:0]     // 低半字的低 2 位
```

这两个位检测的是"该半字**开始**了一条 32 位指令"，而不是"它属于一条 32 位指令"。
区别在于：**低半能判起始，高半不能**——高半的比特是上一半字的延续，
其低 2 位没有独立含义。这一点决定了译码树的分支结构。

#### 4.2.2 类型全集（`:117-127`、`:282-302`）

命名 `SCR1_IFU_INSTR_<高半内容>_<低半内容>`：

| 类型 | 高半字 | 低半字 | 场景 |
|---|---|---|---|
| `NONE` | - | - | 无效/被丢弃 |
| `RVC_RVC` | RVC | RVC | 两条独立 RVC |
| `RVI_LO_RVC` | RVI_LO | RVC | 低位 RVC + 高位起始一条 RVI |
| `RVI_HI_RVI_LO` | RVI_HI | RVI_LO | 一个完整 RVI（块内对齐） |
| `RVC_RVI_HI` | RVC | RVI_HI | 低半拼接上一 RVI + 高半新 RVC（欠账） |
| `RVI_LO_RVI_HI` | RVI_LO | RVI_HI | 低半拼接上一 RVI + 高半新 RVI 低半（欠账） |
| `RVC_NV` | RVC | NV | 非对齐 new_pc 后丢弃低半，高半为 RVC |
| `RVI_LO_NV` | RVI_LO | NV | 非对齐 new_pc 后丢弃低半，高半起始一条 RVI |

#### 4.2.3 三分支译码树（`:282-302`）

译码由 `imem_resp_ok & ~imem_resp_discard_req` 门控：
只有**正常且未被丢弃**的响应才参与译码。

**分支 1：`new_pc_unaligned_ff`（`:286-288`）**

新 PC 的 bit[1]=1，起始半字落在本块**高位**，低半字落在新 PC 之前、无效：

```
高位起始 RVI → RVI_LO_NV
高位是 RVC   → RVC_NV
```

**分支 2：`instr_hi_rvi_lo_ff`（`:290-292`）**

本块**低位半字是上一条 RVI 的高半**（跨字延续），低位被它占满，新指令起点在高位：

```
高位起始 RVI → RVI_LO_RVI_HI
高位是 RVC   → RVC_RVI_HI
```

**分支 3：常态（`:293-299`）**

```
case ({instr_hi_is_rvi, instr_lo_is_rvi})
    2'b00   : RVC_RVC
    2'b10   : RVI_LO_RVC
    default : RVI_HI_RVI_LO
endcase
```

`default` 覆盖 `2'b01` 与 `2'b11`：只要**低位**起始了一条 32 位指令，
高位半字无论如何都是它的高半，无需再看高位的低 2 位。这就是为什么
`RVI_HI_RVI_LO` 用 `default` 而非精确匹配 `2'b11`。

#### 4.2.4 跨块欠账标志 `instr_hi_rvi_lo_ff`（`:304-322`）

- 1 bit 寄存器，记忆"上一取指块高半槽是否为某 RVI 的低半（欠账未还）"；
- 更新规则：

```
if (exu2ifu_pc_new_req_i)  instr_hi_rvi_lo_ff <= 1'b0;          // 换 PC 作废
else if (imem_resp_vd)     instr_hi_rvi_lo_ff <= instr_hi_rvi_lo_next;

instr_hi_rvi_lo_next = (type == RVI_LO_NV)
                     | (type == RVI_LO_RVI_HI)
                     | (type == RVI_LO_RVC);
```

- 这三个类型的共同特征：**高位半字是 `RVI_LO`**。即本块结束时，有一条 32 位指令
  刚好在本块高位半字处起步，其高半必然落在下一块低半字里——所以下拍进入分支 2。
- EXU 换 PC 请求到来时清零（`:312-313`），因为新 PC 冲刷所有旧队列状态。

### 4.3 非对齐 new_pc（`:259-274`）

RISC-V 压缩指令要求 2 字节对齐。若 `exu2ifu_pc_new_i[1]=1`（2 对齐、非 4 对齐），
新 PC 落在取指块中部：

- `new_pc_unaligned_ff` 置位；
- 后续解码将**丢弃块低半**，只认高半（`RVC_NV`/`RVI_LO_NV`）；
- 下次取指地址仍为 4 对齐块（IFU 请求地址恒字对齐），但解析跳过半个字。

标志寄存器更新（`:262-274`）：

```
new_pc_unaligned_upd  = exu2ifu_pc_new_req_i | imem_resp_vd;
new_pc_unaligned_next = exu2ifu_pc_new_req_i ? exu2ifu_pc_new_i[1]
                      : ~imem_resp_vd        ? new_pc_unaligned_ff   // 等待期保持
                                             : 1'b0;                 // 首响应后清零
```

语义：换 PC 时装载新 PC 的 bit[1]；等待首个响应期间保持不变；首响应到达后清零。
与 `instr_hi_rvi_lo_ff` 配合，保证"半字边界错位"这一状态只跨越一次取指。

### 4.4 队列读写大小译码

#### 4.4.1 读大小（`:324-337`）

```
q_rd_vd    = ~q_is_empty & ifu2idu_vd_o & idu2ifu_rdy_i;   // 真正弹出一个条目
q_rd_hword = q_head_is_rvc | q_err_head
`ifdef SCR1_NO_DEC_STAGE
           | (q_head_is_rvi & instr_bypass_vd)
`endif
           ;
q_rd_size  = ~q_rd_vd   ? SCR1_IFU_QUEUE_RD_NONE
           : q_rd_hword ? SCR1_IFU_QUEUE_RD_HWORD
                        : SCR1_IFU_QUEUE_RD_WORD;
```

两条规则：队头是 RVC 或带错误标志时按**半字**弹（1 槽），队头是 RVI 时按**字**弹
（2 槽）。旁路配置下再加一项 `q_head_is_rvi & instr_bypass_vd`——被旁路直送的
RVI 已消费，队列里只剩它的高半，按半字推进指针才能对齐。

#### 4.4.2 写大小（`:339-373`）

先判 `~imem_resp_discard_req`（废单窗口内一律不写），再分正常/错误响应。

**MAX 路径（`else` 分支，`:362-367`）**

| `instr_type` | 写大小 | 理由 |
|---|---|---|
| `NONE` | `WR_NONE` | 无有效数据 |
| `RVC_NV` / `RVI_LO_NV` | `WR_HI` | 只有高位半字有效 |
| 其余 | `WR_FULL` | 高低两半字都要入队 |

**旁路路径（`SCR1_NO_DEC_STAGE`，`:345-360`）**

凡本条指令已同拍直送 IDU（`instr_bypass_vd & idu2ifu_rdy_i`）就扣除对应半字：

| `instr_type` | 旁路且 ready | 否则 |
|---|---|---|
| `NONE` | `WR_NONE` | `WR_NONE` |
| `RVI_LO_NV` | `WR_HI` | `WR_HI` |
| `RVC_NV` | `WR_NONE` | `WR_HI` |
| `RVI_HI_RVI_LO` | `WR_NONE` | `WR_FULL` |
| `RVC_RVC`、`RVI_LO_RVC`、`RVC_RVI_HI`、`RVI_LO_RVI_HI` | `WR_HI` | `WR_FULL` |

**错误响应固定 `WR_FULL`**（`:369-370`）：让错误占据一个完整字槽，
以便在正确的指令位置交给 IDU。

### 4.5 IFU→IDU 输出（MAX 路径，`:715-755`）

#### 4.5.1 状态标志生成（`:720-740`）

默认三个输出清零。仅在 `~q_is_empty`（队列非空）时产生输出，分两种情况：

**情况 A：队列只有 1 个半字（`q_has_1_ocpd_hw`，`:725-727`）**

```
ifu2idu_vd_o       = q_head_is_rvc | q_err_head;
ifu2idu_imem_err_o = q_err_head;
```

"半条指令待续"窗口。只有两种情况可宣告有效：队头是完整 RVC（`q_head_is_rvc`），
或队头本身就是坏数据（`q_err_head`，立即报错让异常前进）。若队头是 RVI 低半、
高半未到，`vd=0` 等待。`err_rvi_hi` 保持 0——只剩 1 个半字时物理上不可能出现
"高半取错"。

**情况 B：队列占用 >= 2 个半字（`:728-732`）**

```
ifu2idu_vd_o          = 1'b1;
ifu2idu_imem_err_o    = q_err_head ? 1'b1 : (q_head_is_rvi & q_err_next);
ifu2idu_err_rvi_hi_o  = ~q_err_head & q_head_is_rvi & q_err_next;
```

数据足够组成一整条，`vd` 无条件为 1。错误按**队头优先**归因：
队头半字自身错则报错；否则若队头是 RVI 且其高半（`q_err_next`）错，则整条 RVI
取指失败。**关键限定 `q_head_is_rvi`**：若队头是 RVC，`q_err_next` 属于下一条
指令，不能归到本条上。

#### 4.5.2 指令数据多路选择（`:745-753`）

```
ifu2idu_instr_o = q_head_is_rvc ? `SCR1_IMEM_DWIDTH'(q_data_head)   // 16 位，高 16 位补 0
                                : {q_data_next, q_data_head};        // 32 位 RVI
```

`{q_data_next, q_data_head}` 中 `q_data_head` 是低半（低地址）、`q_data_next` 是高半，
符合小端程序顺序。标志生成与数据选择由同一套判据驱动，保证"标志有效"与"数据有效"对齐。

#### 4.5.3 错误标志 `err_rvi_hi` 的完整链路

`ifu2idu_err_rvi_hi_o` 存在的唯一目的是让异常时的 `mtval` 指向真正出错的地址。
完整跨模块链路：

```
IFU :731  ifu2idu_err_rvi_hi_o
  → top :333  ifu2idu_err_rvi_hi
  → IDU :137  idu2exu_cmd_o.instr_rvc = ifu2idu_err_rvi_hi_i   // 字段被复用
  → EXU :341  exu_queue.instr_rvc <= idu2exu_cmd_i.instr_rvc
  → EXU :534  instr_fault_rvi_hi = exu_queue.instr_rvc
  → EXU :550  exc_trap_val = instr_fault_rvi_hi ? inc_pc : pc_curr_ff
              inc_pc = pc_curr_ff + (`SCR1_XLEN'd2 或 'd4)       // :695
```

对非对齐 32 位指令，低半在 `pc`、高半在 `pc+2`。错误发生在高半时，
trap value 应指向**真正出错的地址 `pc+2`**；错误在低半则指向指令起始 `pc`。
`instr_rvc` 字段在异常路径上被赋予另一层语义（`scr1_riscv_isa_decoding.svh:162`
注释注明 "used with a different meaning for IFU access fault exception"）。

### 4.6 旁路路径（`SCR1_NO_DEC_STAGE`，`:623-713`）

当 IFU 与 IDU 之间不设流水寄存器（DEC 级）时，指令必须能在响应到达的**同一拍**
直接送进 IDU，否则每次取指白白多一拍的延迟。旁路路径即为此设计。

#### 4.6.1 旁路类型（`:108-115`）

```
SCR1_BYPASS_NONE             不走旁路，仍从队列取
SCR1_BYPASS_RVC              本次响应里有一条完整 RVC
SCR1_BYPASS_RVI_RDATA        本次响应里有一条完整 RVI（整字）
SCR1_BYPASS_RVI_RDATA_QUEUE  RVI 低半在队列、高半在本次响应，需拼接
```

#### 4.6.2 旁路译码（`:630-652`）

`instr_bypass_vd = (instr_bypass_type != SCR1_BYPASS_NONE)`（`:628`）。
译码只在 `imem_resp_vd` 时进行，分两种情况。

**队列为空（`:634`，连续取指稳态）**

| `instr_type` | 旁路类型 | 原因 |
|---|---|---|
| `RVC_NV` / `RVC_RVC` / `RVI_LO_RVC` | `BYPASS_RVC` | 低位半字是完整 RVC，可立即发射 |
| `RVI_HI_RVI_LO` | `BYPASS_RVI_RDATA` | 低半=RVI 低半、高半=RVI 高半，整条已齐 |

**队列卡着 RVI 低半（`:646-650`）**

```
q_has_1_ocpd_hw & q_head_is_rvi & instr_hi_rvi_lo_ff → BYPASS_RVI_RDATA_QUEUE
```

RVI 跨越字边界：低半已进队列，高半落在本次响应的 `imem_rdata_lo`。
`instr_hi_rvi_lo_ff` 用来确认本次响应的低位确实是它的续半。

#### 4.6.3 旁路状态输出（`:657-686`）

门控 `ifu_fsm_fetch | ~q_is_empty`，分两支：

**有旁路（`:663-668`）**

```
ifu2idu_vd_o          = 1'b1;
ifu2idu_imem_err_o    = (type == RVI_RDATA_QUEUE) ? (imem_resp_er | q_err_head)
                                                   : imem_resp_er;
ifu2idu_err_rvi_hi_o  = (type == RVI_RDATA_QUEUE) & imem_resp_er;
```

拼接型旁路的错误可能来自**两个来源**：队列里那条低半（`q_err_head`）或本次响应的
高半（`imem_resp_er`）。只有"高半错"才算"错在 RVI 高半"（`err_rvi_hi`），
因为此时低半是好的。

**无旁路（`:669-678`）**

退回队列路径，逻辑与 MAX 分支一致。差异一处：旁路分支在"只有 1 个半字"时
**也**计算 `err_rvi_hi = ~q_err_head & q_head_is_rvi & q_err_next`（`:673`），
而 MAX 分支该情形下保持为 0，反映旁路配置下"低半在队列、高半同拍到达"的组合。

#### 4.6.4 旁路输出多路选择（`:691-713`）

```
case (instr_bypass_type)
  BYPASS_RVC             : new_pc_unaligned_ff ? imem_rdata_hi : imem_rdata_lo
  BYPASS_RVI_RDATA       : imem2ifu_rdata_i                  // 整条 32 位
  BYPASS_RVI_RDATA_QUEUE : {imem_rdata_lo, q_data_head}      // 高半在前、低半在后
  default                : q_head_is_rvc ? q_data_head : {q_data_next, q_data_head}
endcase
```

- `BYPASS_RVC` 用 `new_pc_unaligned_ff` 选高低半：非对齐跳转目标的起始半字落在
  响应**高位**，故取 `hi`；正常取 `lo`；
- `BYPASS_RVI_RDATA_QUEUE` 拼接顺序 `{高半, 低半}` 与队列路径 `{q_data_next, q_data_head}`
  保持同一约定。

#### 4.6.5 与队列写大小的配合

旁路是"同拍直送 + 队列相应少写半字"，避免同一条指令被消费两次，见 §4.4.2 的旁路表。
改变的是**延迟**，不是每拍条数。

### 4.7 IFU FSM（`:91-94`、`:460-490`）

两个状态，语义是"一个取指窗口开着还是关着"：

```
SCR1_IFU_FSM_IDLE     不取指
SCR1_IFU_FSM_FETCH    取指窗口打开，可持续发请求
```

控制信号：

```
ifu_fetch_req = exu2ifu_pc_new_req_i & ~pipe2ifu_stop_fetch_i      // 开窗
ifu_stop_req  = pipe2ifu_stop_fetch_i
              | (imem_resp_er_discard_pnd & ~exu2ifu_pc_new_req_i) // 关窗
```

- **开窗**：EXU 送来新 PC，且管道未要求停取；
- **关窗**：管道要求停止（WFI、FENCE、调试等），或收到一个**需要丢弃的取指错误**。
  错误之后后续指令的有效性无法保证，应停止继续取，交给异常流程重新定向；
- **优先级**：`ifu_stop_req` 中 `imem_resp_er_discard_pnd` 与 `~exu2ifu_pc_new_req_i`
  相与——同拍既有错误要丢弃、又有新 PC 请求时，**新 PC 优先**，窗口不关。

转移关系：

```
IDLE  --ifu_fetch_req-->  FETCH
FETCH --ifu_stop_req -->  IDLE
其余情况保持
```

`ifu_fsm_fetch = (ifu_fsm_curr == FETCH)`（`:490`）是请求准入的一个条件。

### 4.8 IMEM 接口与计数逻辑

#### 4.8.1 响应译码（`:504-513`）

| 信号 | 表达式 | 含义 |
|---|---|---|
| `imem_resp_ok` | `resp == RDY_OK` | 正常响应 |
| `imem_resp_er` | `resp == RDY_ER` | 错误响应 |
| `imem_resp_received` | `ok \| er` | 收到任意有效响应 |
| `imem_resp_vd` | `received & ~discard_req` | 该响应**可用**（不在废单窗口内） |
| `imem_resp_er_discard_pnd` | `er & ~discard_req` | 一个尚未被判定丢弃的新错误 |
| `imem_handshake_done` | `req & ack` | 请求被接受 |

`imem_resp_vd` 与 `imem_resp_er_discard_pnd` 都带 `~discard_req`，是整个丢弃机制的
过滤口：窗口内到达的响应一律视为废料，正常路径看不到它们。

#### 4.8.2 取指地址寄存器与窄加法器（`:515-536`）

```
imem_addr_upd  = imem_handshake_done | exu2ifu_pc_new_req_i;

imem_addr_next = pc_new_req ? new_pc 的 word 地址
             : &imem_addr_ff[5:2] ? imem_addr_ff + imem_handshake_done
                                  : {imem_addr_ff[XLEN-1:6], imem_addr_ff[5:2] + imem_handshake_done};
```

`imem_addr_ff` 存**字地址**（`[XLEN-1:2]`），输出时补两个 0。

**窄进位加法技巧**：给字地址 +1 时，低 4 位 `[5:2]` 全为 1 才会向高位产生进位。

- 若 `&imem_addr_ff[5:2]`（即将进位）→ 做整宽 `+1`；
- 否则只对 `[5:2]` 加 1，高位原样拼接。

把长加法器拆成"4 位检测 + 4 位加法 + 条件全宽加法"，比无条件跑整宽加法器更省。

**`+ imem_handshake_done` 而非 `+ 1`**：`imem_addr_upd` 成立且走递增路径时
`handshake_done` 必为 1，两者等价；写成信号名让"地址前进"与"确实完成一次握手"
在语义上绑定。

`SCR1_NEW_PC_REG` 两版（`:529` 与 `:533`）的差别只有一处：非寄存器版在新 PC 分支
上多了 `+ imem_handshake_done`，用于补偿"新 PC 由 EXU 组合给出、IFU 无寄存器缓冲、
同拍可能已发生一次握手"的情形。

#### 4.8.3 pending 在飞事务计数（`:538-554`）

```
imem_pnd_txns_cnt_upd  = imem_handshake_done ^ imem_resp_received;
imem_pnd_txns_cnt_next = imem_pnd_txns_cnt + (imem_handshake_done - imem_resp_received);
imem_pnd_txns_q_full   = &imem_pnd_txns_cnt;
```

- 记录"请求已 ack、响应未归"的事务数；
- **异或做更新使能**：只有"握手与响应恰好发生其一"时计数才变化；同拍两者都发生
  时净变化 0，无需更新；
- 增量用布尔相减：`1-0=+1`、`0-1=-1`、`1-1=0`；
- 宽度 `SCR1_TXN_CNT_W=3`，最大 7；`q_full = &cnt`（全 1）作为请求的硬闸门，
  与 `q_has_free_slots` 一起保证事务数与队列容量双重不越界。

#### 4.8.4 discard 丢弃计数（`:556-591`）

需要丢弃在飞响应的两种情形（`:558-567` 注释）：

1. **换 PC（new_pc 请求）**：PC 已变，未取回和未发出的指令都不再需要；
2. **出现了一个未被丢弃的错误响应**：错误之后 IMEM 返回内容的有效性无法保证。

两种情况下"要丢弃的条数 = 在飞事务数"。

**更新条件（`:569-570`）**

```
imem_resp_discard_cnt_upd = pc_new_req | imem_resp_er | (imem_resp_ok & discard_req)
```

三种触发：新 PC（重装计数）、任意错误响应（重装）、窗口内收到正常响应（递减）。

**次态（`:580-588`）**

```
非 NEW_PC_REG：
  next = pc_new_req ? (imem_pnd_txns_cnt_next - imem_handshake_done)
       : er_discard_pnd ? imem_pnd_txns_cnt_next
                        : imem_resp_discard_cnt - 1'b1
NEW_PC_REG：
  next = (pc_new_req | er_discard_pnd) ? imem_pnd_txns_cnt_next
                                       : imem_resp_discard_cnt - 1'b1
```

- `pc_new_req` 时装载 `pnd_next - handshake_done`：只丢弃**真正在飞**的那些
  （同拍完成的握手不计入）；
- 新错误（`er_discard_pnd`，排除窗口内的错误）时装载 `pnd_next`：
  把当前所有在飞事务全部作废；
- 窗口内每收到一个响应就 `-1`，直到归零。

**派生信号（`:590-591`）**

```
imem_vd_pnd_txns_cnt  = imem_pnd_txns_cnt - imem_resp_discard_cnt   // 仍"有用"的在飞数
imem_resp_discard_req = |imem_resp_discard_cnt                      // 窗口是否开着
```

- `imem_vd_pnd_txns_cnt` 只在 `q_has_free_slots`（`:454`）使用：废单不入队，
  不占队列座位，容量只按有效在飞数折算；
- `imem_resp_discard_req` 是全模块的"废料窗口"哨兵，同时门控 `imem_resp_vd`（`:510`）、
  `imem_resp_er_discard_pnd`（`:511`）、写大小（`:342`）、`q_wr_en`（`:419`）等。

**窗口内错误按普通废料处理**：丢弃、计数减一、不产生新异常。这正是
`imem_resp_er_discard_pnd = er & ~discard_req` 要排除的情形，也是它能作为
"重装触发器"的原因。

#### 4.8.5 接口输出信号（`:593-611`）

```
非 SCR1_NEW_PC_REG：
  req  = (pc_new_req & ~q_full & ~stop)                  // 跳转后首拍立即发出
       | (ifu_fsm_fetch & ~q_full & q_has_free_slots)
  addr = pc_new_req ? {new_pc[XLEN-1:2], 2'b00}
                    : {imem_addr_ff, 2'b00}

SCR1_NEW_PC_REG：
  req  = ifu_fsm_fetch & ~q_full & q_has_free_slots      // 只能由 FSM 发起
  addr = {imem_addr_ff, 2'b00}
```

非寄存器版允许"新 PC 请求"直接旁路 FSM 发一次请求，跳转后零延迟起取；
寄存器版依赖 FSM 状态和已捕获地址，时序更干净但多一拍。

三处闸门始终一致：`~imem_pnd_txns_q_full`（事务槽未满）、`~pipe2ifu_stop_fetch_i`
（管道未要求停）、`q_has_free_slots`（队列有空位）。

`ifu2imem_cmd_o = SCR1_MEM_CMD_RD`（`:607`）。

### 4.9 错误注入与精确异常

设计决策：**IMEM 错误不即时在 IFU 报告，而是作为一条指令的携带标志入队**。

- 错误响应照常写满 2 槽，数据是垃圾，但 `q_err[]` 随槽存储（`:419-439`）；
- 出队时错误转为 `ifu2idu_imem_err_o`；RVI 高半槽错误有独立标志
  `ifu2idu_err_rvi_hi_o`（`:724-733`）；
- IDU 收到 `imem_err` 不译码，直接产生 `INSTR_ACCESS_FAULT`（异常码 1）命令
  （`scr1_pipe_idu.sv:134-137`），随流水走到 EXU/CSR 完成 trap
  （MCAUSE=1、MEPC=出错指令）。

**为什么必须入队而不是当场报错**：异常必须精确绑定具体指令，且后续指令不能产生
副作用。IFU 出错时队列可能已囤正常指令、错误响应可能多条在飞，IFU 无法预知哪条
指令对应哪个错。让错误按序入队、由 IDU/EXU 消费时翻牌，异常码、出错 PC、顺序
天然精确。

### 4.10 吞吐与队列深度

#### 4.10.1 交付粒度

IFU→IDU 每拍最多交付**一条指令**，`ifu2idu_instr_o` 只有一个字段。RVC 占 1 个
半字槽、RVI 占 2 个，所以"每拍一条"不等于"每拍一个半字"：

| 队头类型 | 单拍交付 | 消耗槽位 |
|---|---|---|
| RVC | 1 条 16 位 | 1 |
| RVI | 1 条 32 位（`{next, head}` 同拍拼出） | 2 |

#### 4.10.2 一个 32 位响应的四种构成与出队节奏

| 响应内容 | 写大小 | 出队节奏 |
|---|---|---|
| RVC + RVC | `WR_FULL` | 2 拍，各 1 条 RVC |
| RVI（低半+高半） | `WR_FULL` | 1 拍，1 条 32 位 |
| 低半 RVC + 高半 RVI_LO | `WR_FULL` | 第 1 拍送 RVC；那条 RVI 的高半在下一次响应里 |
| 低半 RVI_HI + 高半 RVC | `WR_FULL` | 第 1 拍完成上一条 RVI，第 2 拍送 RVC |

队列里既可能存 RVC，也可能存"待续半条 RVI"，`q_head_is_rvc` 正是读侧区分这两种
节奏的依据。

#### 4.10.3 在飞事务数与队列深度的配套

- 队列空时 `q_free_w_next = 2`。发起第 1 笔后 `vd_pnd = 1`：`2 > 1` 成立，可发第 2 笔；
  再发后 `vd_pnd = 2`：`2 > 2` 不成立，停止；
- 即**最多 2 笔在飞事务**，每笔最多 2 条指令，合计 4 个半字 = 队列容量，完全匹配；
- `imem_pnd_txns_cnt` 的位宽上限 7 是硬保护，正常流控下不会触及。

#### 4.10.4 稳态吞吐

- **峰值交付**：1 指令/周期；
- **补给**：1 事务最多 2 条；连续 RVC 时需 IMEM 每 2 周期完成 1 个字，
  连续 RVI 时需每周期完成 1 个字；
- **缓冲**：队列 2 个字，可吸收约 2 个周期的 IMEM 延迟；
- **受限场景**：连续 RVI 每拍吃掉 2 个半字槽，若 IMEM 补给不及时即出现取指气泡。

### 4.11 调试接口（HDU 程序缓冲，`SCR1_DBG_EN`，`:52-59`、`:680-685`、`:708-712`、`:734-753`）

当 `hdu2ifu_pbuf_fetch_i`（调试器经程序缓冲注入指令）：

- 输出数据/错误/有效全部改由 HDU 程序缓冲提供（`:681-685`、`:708-712`、`:734-753`）；
- `ifu2hdu_pbuf_rdy_o = idu2ifu_rdy_i`（`:757-759`）：程序缓冲接口的就绪即 IDU 就绪；
- `stop_fetch` 随 `fetch_pbuf` 拉高（`scr1_pipe_top.sv:278-282`），停止正常取指。

### 4.12 时钟控制接口（`SCR1_CLKCTRL_EN`，`:61-63`、`:609-611`）

`ifu2pipe_imem_txns_pnd_o = |imem_pnd_txns_cnt`（`:610`）——只要还有在飞 IMEM 事务，
该位为 1。经 `scr1_pipe_top.sv:285` 与 WFI 停机状态相与（`wfi_halted & ~imem_txns_pending`），
最终送到 `scr1_clk_ctrl.sv:35` 门控流水线时钟，防止核在取指事务未回时进入睡眠。

---

## 5. 配置宏对行为的影响

| 宏 | 未定义（MAX 默认） | 定义（MIN/IC_BASE 等） | 行号 |
|---|---|---|---|
| `SCR1_NO_DEC_STAGE` | 指令先入队再出队，输出用队列路径 | 增加旁路，指令可直接送 IDU（§4.6） | 623-713 |
| `SCR1_NEW_PC_REG` | new_pc 直接驱动请求线 | new_pc 先进寄存器再发请求 | 596-605 |
| `SCR1_DBG_EN` | 无 HDU 程序缓冲 | 增加调试指令注入与 `pbuf_rdy` 反馈 | 52-59, 680-685, 708-712, 734-753 |
| `SCR1_CLKCTRL_EN` | 无 | 输出在飞事务指示供门控时钟 | 61-63, 609-611 |
| `SCR1_TRGT_SIMULATION` | 无 | 使能 SVA 断言块 | 761-824 |

> 注：`SCR1_CFG_RV32IMC_MAX` 均未定义前两个宏 → IFU 走完整队列路径。
> `SCR1_NO_DEC_STAGE` 的宏名含义为"禁用 IFU 与 IDU 之间的寄存器（DEC 级）"。

### 5.1 两条输出路径对比

| 维度 | 旁路路径 `SCR1_NO_DEC_STAGE` | 标准路径（MAX） |
|---|---|---|
| 取指延迟 | 响应同拍可直达 IDU | 经队列，至少多一拍 |
| 组合路径 | 更长（IMEM 响应直穿到 IDU） | 较短 |
| 输出来源 | `instr_bypass_type` 四态选择 | 只看队头/次槽 |
| 错误归属 | 需区分高半来自响应还是队列 | 按队头/次槽两个固定位置 |
| 适用配置 | 追求短延迟的 MIN/低端配置 | RV32IMC MAX 等完整配置 |

---

## 6. 时序行为示例

### 6.1 顺序取指（无 RVC、无停歇）

```
clk    ▔▔|▁▁|▔▔|▁▁|▔▔|▁▁|▔▔|▁▁|▔▔
req      ___█______█______█______█____   (FETCH 态连续发)
ack      ____█______█______█______█___   (立即应答)
addr     ___0x200___0x204___0x208___0x20c
resp     ________________█______█______  (OK, OK...)
pnd      0 →1→0  1→0  1→0  1→0
```

TCM 等零延迟内存：应答与响应可同拍或错拍，pending 计数在 0/1 间摆动。

### 6.2 换 PC（flush）

```
new_pc_req  ____█____________________
flush       ____█____________________  (wptr/rptr 清零, discard 置为 pnd)
队列清空     旧指令全部作废；FSM 转向新 PC 取指
```

断言 `SCR1_SVA_IFU_NEW_PC_REQ_BEH`：换 PC 下一拍队列必须为空（`:806`）。

### 6.3 错误响应与精确异常

```
IMEM 响应 RDY_ER：
  ① imem_resp_er → 该响应照常入队 + q_err 置位
  ② discard_cnt 装载为 pnd → 后续在飞响应全部丢弃
  ③ FSM → IDLE（停止取指）
  ④ 队头带错指令出队 → ifu2idu_imem_err_o=1
  ⑤ IDU 生成 INSTR_ACCESS_FAULT → EXU → CSR（MCAUSE=1）
```

### 6.4 非对齐 new_pc（跨块 RVI 场景）

```
new_pc = 0x...2 （2 对齐非 4 对齐）
  ① new_pc_unaligned_ff 置位（装载 new_pc[1]）
  ② 下一取指块：解码丢弃低半字，只认高半字（RVC_NV 或 RVI_LO_NV）
  ③ 若高半是 RVI 低半 → instr_hi_rvi_lo_ff 置位（欠账）
  ④ 再下一块：低半字作为上一 RVI 的高半拼接 → 完整指令还原
```

### 6.5 一个 32 位响应的两拍排空（RVC + RVC）

```
周期      T0        T1        T2
resp      RDY_OK    -         -
instr_type RVC_RVC  -         -
q_wr_size WR_FULL   -         -
q_ocpd_h  0 → 2    2 → 1     1 → 0
ifu2idu_vd_o  0     1         1
instr_o   -        RVC(lo)   RVC(hi)
q_rd_size -        RD_HWORD  RD_HWORD
```

两次响应之间的任何时刻，队列里都可能有 1 个半字处于"待续"状态。

---

## 7. 断言规格（`:769-822`，仅 `SCR1_TRGT_SIMULATION`）

| 断言 | 检查内容 | 行号 |
|---|---|---|
| `SCR1_SVA_IFU_XCHECK` | `req_ack`/`rdy`/`pc_new_req` 无 X | 769 |
| `SCR1_SVA_IFU_XCHECK_REQ` | 请求有效时地址/命令无 X | 774 |
| `SCR1_SVA_IFU_DRC_UNDERFLOW` | discard 计数不下溢 | 781 |
| `SCR1_SVA_IFU_DRC_RANGE` | discard 计数 ∈ `[0, pnd]` | 786 |
| `SCR1_SVA_IFU_QUEUE_OVF` | 占用达 3 个半字时禁止 `WR_FULL`；达 4 时禁止任何写 | 791 |
| `SCR1_SVA_IFU_IMEM_ERR_BEH` | 错误后 FSM 回 IDLE 且 `discard_cnt == pnd` | 798 |
| `SCR1_SVA_IFU_NEW_PC_REQ_BEH` | new_pc 请求后下一拍队列为空 | 804 |
| `SCR1_SVA_IFU_IMEM_ADDR_ALIGNED` | IMEM 访问地址 `[1:0]==0` | 809 |
| `SCR1_SVA_IFU_STOP_FETCH` | 停取指请求后 FSM 回 IDLE | 814 |
| `SCR1_SVA_IFU_IMEM_FAULT_RVI_HI` | `err_rvi_hi` 置位时 `imem_err` 必为 1 | 819 |

> 所有断言采样于 `negedge clk` 且 `disable iff (~rst_n)`，即避开时钟上升沿的
> 组合稳定期采样。`SCR1_SVA_IFU_QUEUE_OVF` 的含义是：写 `WR_FULL` 需要 2 个空槽，
> 因此当占用已达 `SIZE_HALF-1=3` 时必须禁止。

---

## 8. 假设与约束

1. IMEM 请求地址必须 4 字节对齐（IFU 自身保证，见断言 `IFU_IMEM_ADDR_ALIGNED`）。
2. IMEM 最大在飞事务数 < 8（`SCR1_TXN_CNT_W=3`），超出即暂停发请求；
   正常流控下最大 2 笔在飞（§4.10.3）。
3. 队列最多容纳 2 个 32 位字 = 4 个半字；跨块 RVI + 后续指令依赖该容量不丢取指。
4. IFU 假设取指流为合法指令流；对 `[1:0]==11` 的半字按 RVI 处理，若落入数据区
   是程序/链接层错误而非 IFU 责任。
5. `pipe2ifu_stop_fetch_i` 来自 WFI 或调试模式，IFU 不自行判断停机。
6. 队列永不溢出的前提是写侧准入严格：`q_has_free_slots` 要求空闲字数**严格大于**
   有效在飞数，配合 `imem_pnd_txns_q_full` 双重约束。
7. 跨块 RVI 的正确性依赖 `instr_hi_rvi_lo_ff` 与 `new_pc_unaligned_ff` 两个上下文
   标志的更新时序；换 PC 时二者同步清零。

---

## 附录 A：端口清单（22）

### A.1 控制（`:36-38`）
| 端口 | 方向 | 功能 |
|---|---|---|
| `rst_n` / `clk` | in | 复位（低有效）、时钟 |
| `pipe2ifu_stop_fetch_i` | in | 停取指（WFI 停机 / 调试） |

### A.2 IFU ↔ IMEM（`:41-46`）
| 端口 | 方向 | 功能 |
|---|---|---|
| `imem2ifu_req_ack_i` | in | 读请求应答 |
| `ifu2imem_req_o` | out | 读请求 |
| `ifu2imem_cmd_o` | out | 命令（恒 `READ`） |
| `ifu2imem_addr_o` | out | 取指地址（字对齐） |
| `imem2ifu_rdata_i` | in | 32 位读数据 |
| `imem2ifu_resp_i` | in | 响应（`NOTRDY`/`RDY_OK`/`RDY_ER`） |

### A.3 IFU ↔ EXU（`:49-50`）
| 端口 | 方向 | 功能 |
|---|---|---|
| `exu2ifu_pc_new_req_i` | in | 换 PC 请求（jump/branch/trap/mret） |
| `exu2ifu_pc_new_i` | in | 新 PC 值 |

### A.4 IFU ↔ HDU 调试（`:54-58`，仅 `SCR1_DBG_EN`）
| 端口 | 方向 | 功能 |
|---|---|---|
| `hdu2ifu_pbuf_fetch_i` | in | 改从程序缓冲取指 |
| `hdu2ifu_pbuf_vd_i` / `hdu2ifu_pbuf_err_i` / `hdu2ifu_pbuf_instr_i` | in | 程序缓冲指令的有效/错误/内容 |
| `ifu2hdu_pbuf_rdy_o` | out | 程序缓冲接口就绪 |

### A.5 时钟控制（`:62`，仅 `SCR1_CLKCTRL_EN`）
| 端口 | 方向 | 功能 |
|---|---|---|
| `ifu2pipe_imem_txns_pnd_o` | out | 有在飞 IMEM 事务（禁止休眠门控时钟） |

### A.6 IFU ↔ IDU（`:66-70`）
| 端口 | 方向 | 功能 |
|---|---|---|
| `idu2ifu_rdy_i` | in | IDU 可接收 |
| `ifu2idu_instr_o` | out | 指令（16/32 位） |
| `ifu2idu_imem_err_o` | out | 取指异常（instruction access fault） |
| `ifu2idu_err_rvi_hi_o` | out | RVI 高半字取指错误 |
| `ifu2idu_vd_o` | out | 指令有效 |

---

## 附录 B：局部参数与类型

### B.1 localparam（`:77-85`）
| 名称 | 值 | 功能 |
|---|---|---|
| `SCR1_IFU_Q_SIZE_WORD` | 2 | 队列容量（字） |
| `SCR1_IFU_Q_SIZE_HALF` | 4 | 队列容量（半字槽） |
| `SCR1_TXN_CNT_W` | 3 | 计数器位宽（0~7） |
| `SCR1_IFU_QUEUE_ADR_W` | 2 | 队列地址位宽 |
| `SCR1_IFU_QUEUE_PTR_W` | 3 | 指针位宽（地址+1，可判空） |
| `SCR1_IFU_Q_FREE_H_W` | 3 | 空闲半字数计数位宽（0~4） |
| `SCR1_IFU_Q_FREE_W_W` | 2 | 空闲字数计数位宽（0~2） |

### B.2 typedef（`:91-127`）
| 类型 | 取值 | 功能 |
|---|---|---|
| `type_scr1_ifu_fsm_e` | `IDLE`/`FETCH` | FSM 两态（`:91-94`） |
| `type_scr1_ifu_queue_wr_e` | `WR_NONE`/`WR_FULL`/`WR_HI` | 写队列大小（`:96-100`） |
| `type_scr1_ifu_queue_rd_e` | `RD_NONE`/`RD_HWORD`/`RD_WORD` | 读队列大小（`:102-106`） |
| `type_scr1_bypass_e` | `BYPASS_NONE`/`_RVC`/`_RVI_RDATA`/`_RVI_RDATA_QUEUE` | 旁路类型（`:108-115`，仅 `SCR1_NO_DEC_STAGE`） |
| `type_scr1_ifu_instr_e` | 8 型 | 指令半字组合类型（`:117-127`） |

---

## 附录 C：内部信号清单

### C.1 非对齐 new_pc 标志（`:137-139`）
| 变量 | 功能 |
|---|---|
| `new_pc_unaligned_ff` | 记忆下一取指块需丢弃低半字 |
| `new_pc_unaligned_next` / `_upd` | 次态 / 更新使能 |

### C.2 指令类型译码（`:142-144`）
| 变量 | 功能 |
|---|---|
| `instr_hi_is_rvi` | 高半字 `[17:16]==11` |
| `instr_lo_is_rvi` | 低半字 `[1:0]==11` |
| `instr_type` | 8 型译码结果 |

### C.3 跨块欠账（`:148-149`）
| 变量 | 功能 |
|---|---|
| `instr_hi_rvi_lo_ff` | 上一块高半是某 RVI 低半（欠账待还） |
| `instr_hi_rvi_lo_next` | 次态 |

### C.4 读写大小译码（`:152-158`）
| 变量 | 功能 |
|---|---|
| `q_rd_size` / `q_rd_none` / `q_rd_hword` / `q_rd_vd` | 读大小枚举 / 无读 / 读半字 / 读有效 |
| `q_wr_size` / `q_wr_none` / `q_wr_full` | 写大小枚举 / 无写 / 整字写 |

### C.5 读写指针（`:161-166`）
| 变量 | 功能 |
|---|---|
| `q_rptr` / `q_rptr_next` / `q_rptr_upd` | 读指针 / 次态 / 使能 |
| `q_wptr` / `q_wptr_next` / `q_wptr_upd` | 写指针 / 次态 / 使能 |

### C.6 队列控制与数据（`:169-180`）
| 变量 | 功能 |
|---|---|
| `q_wr_en` | 写队列使能（`imem_resp_vd & ~q_flush_req`） |
| `q_flush_req` | 清队列（换 PC / 停取指） |
| `q_data[4]` | 4 个 16 位半字数据槽 |
| `q_data_head` / `q_data_next` | 队头槽 / 下一槽数据 |
| `q_err[4]` | 每槽取指错误位 |
| `q_err_head` / `q_err_next` | 队头 / 次槽错误位 |

### C.7 队列状态（`:183-190`）
| 变量 | 功能 |
|---|---|
| `q_is_empty` | 队列空 |
| `q_has_free_slots` | 有空间容纳在飞响应 |
| `q_has_1_ocpd_hw` | 仅 1 个半字 |
| `q_head_is_rvc` / `q_head_is_rvi` | 队头是 RVC / RVI 低半 |
| `q_ocpd_h` | 已占半字数 |
| `q_free_h_next` / `q_free_w_next` | 下一拍空闲半字/字数 |

### C.8 FSM（`:196-201`）
| 变量 | 功能 |
|---|---|
| `ifu_fetch_req` / `ifu_stop_req` | 进/退 FETCH 条件 |
| `ifu_fsm_curr` / `ifu_fsm_next` | 当前/次态 |
| `ifu_fsm_fetch` | 处于 FETCH 态 |

### C.9 IMEM 响应（`:207-216`）
| 变量 | 功能 |
|---|---|
| `imem_resp_ok` / `imem_resp_er` | 响应为 `RDY_OK` / `RDY_ER` |
| `imem_resp_received` | 终结响应（ok 或 er） |
| `imem_resp_vd` | 有效可入队（received 且非丢弃窗口） |
| `imem_resp_er_discard_pnd` | 首个错误（自身入队并触发装载） |
| `imem_resp_discard_req` | 丢弃窗口开启（`|discard_cnt`） |
| `imem_handshake_done` | 请求被应答（推进地址/计数） |
| `imem_rdata_lo[15:0]` / `imem_rdata_hi[31:16]` | 读数据低/高半字切片 |

### C.10 IMEM 地址（`:219-221`）
| 变量 | 功能 |
|---|---|
| `imem_addr_ff` | 取指地址寄存器 `[31:2]`（字地址） |
| `imem_addr_next` | 地址次态（顺序 +1 / 新 PC） |
| `imem_addr_upd` | 地址更新使能 |

### C.11 在飞计数（`:224-228`）
| 变量 | 功能 |
|---|---|
| `imem_pnd_txns_cnt` | 在飞总笔数 |
| `imem_pnd_txns_cnt_next` / `_upd` | 次态 / 更新使能 |
| `imem_vd_pnd_txns_cnt` | 有效在飞（`pnd - discard`） |
| `imem_pnd_txns_q_full` | 在飞满（停发请求） |

### C.12 丢弃计数（`:231-233`）
| 变量 | 功能 |
|---|---|
| `imem_resp_discard_cnt` | 待丢弃废单笔数 |
| `imem_resp_discard_cnt_next` / `_upd` | 次态 / 更新使能 |

### C.13 条件编译（`:235-243`）
| 变量 | 条件 | 功能 |
|---|---|---|
| `new_pc_req_ff` | `SCR1_NEW_PC_REG` | new_pc 请求经寄存器后再发 |
| `instr_bypass_type` / `instr_bypass_vd` | `SCR1_NO_DEC_STAGE` | 指令旁路类型/有效 |

---

## 附录 D：一条指令的生命周期

以一条普通 RV32I 指令（4 对齐）为例：

```
① 请求        FSM 进入 FETCH，ifu2imem_req_o 拉高，地址 = imem_addr_ff
② 握手        imem2ifu_req_ack_i 到达 → imem_handshake_done=1
              → 地址 +1（窄加法器）、pnd_txns_cnt +1
③ 响应        imem2ifu_resp_i = RDY_OK → imem_resp_received=1
              → imem_resp_vd=1（若不在丢弃窗口）、pnd_txns_cnt -1
④ 判据提取    instr_lo_is_rvi = &rdata[1:0]（=1）、instr_hi_is_rvi（任意）
⑤ 类型译码    两个上下文标志为 0 → default 分支 → RVI_HI_RVI_LO
⑥ 写大小      MAX：WR_FULL（旁路：bypass&ready 时 WR_NONE）
⑦ 写队列      q_data[wptr]<=rdata_lo、q_data[wptr+1]<=rdata_hi，q_err 同写
              q_wptr += 2
⑧ 读侧        队头 q_head_is_rvi=1 → q_rd_size=RD_WORD（需 2 槽）
⑨ 握手输出    ifu2idu_vd_o=1，ifu2idu_instr_o={q_data_next, q_data_head}
              idu2ifu_rdy_i=1 → 交接完成，q_rptr += 2
⑩ 错误归属    若任一槽 q_err=1 → ifu2idu_imem_err_o=1（高半错时另置 err_rvi_hi）
```

跨块 RVI 的差异出现在 ④⑤：首次响应给出 `RVI_LO_NV` 或 `RVI_LO_RVC`，
`instr_hi_rvi_lo_ff` 被置位；下一次响应走分支 2，低半作为高半被拼接。

---

## 附录 E：关键信号 → 消费者映射

| 信号 | 消费者 | 作用 |
|---|---|---|
| `q_is_empty` | `:328`、`:634`、`:669`、`:724`、`:806` | 无数据可出；空时清输出；断言 |
| `q_has_free_slots` | `:598`、`:603` | 写侧准入：能否发 IMEM 请求 |
| `q_has_1_ocpd_hw` | `:646`、`:670`、`:725` | 出队决策：区分"半条指令"与"完整条目" |
| `q_head_is_rvi` / `q_head_is_rvc` | `:329`、`:646`、`:671-676`、`:704`、`:726-731`、`:746` | 读出粒度、指令拼接、错误归属 |
| `q_ocpd_h` | `:793-794` | 断言：占用接近满时禁止 `WR_FULL` |
| `imem_vd_pnd_txns_cnt` | `:454` | 队列预留座位计算 |
| `imem_resp_discard_req` | `:342`、`:419`、`:510`、`:511`、`:570`、`:590` | 废料窗口哨兵 |
| `imem_pnd_txns_cnt` | `:543`、`:553`、`:590`、`:610` | 计数、有效在飞、时钟门控 |
| `ifu_fsm_fetch` | `:598`、`:603`、`:662` | 请求准入与旁路门控 |
| `instr_hi_rvi_lo_ff` | `:290`、`:647` | 跨块译码分支、旁路拼接确认 |
| `new_pc_unaligned_ff` | `:286`、`:694` | 非对齐译码分支、旁路高低半选择 |
| `ifu2idu_err_rvi_hi_o` | IDU `:137` → EXU `:534`、`:550` | 异常 `mtval` 指向出错高半 |
