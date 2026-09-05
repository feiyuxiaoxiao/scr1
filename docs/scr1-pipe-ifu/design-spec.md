# SCR1 IFU 取指单元设计规格书（Design Specification）

- 模块：`scr1_pipe_ifu`
- 源文件：`src/core/pipeline/scr1_pipe_ifu.sv`（826 行）
- 位置：SCR1 RV32 内核流水线顶层 `scr1_pipe_top` 的第一级
- 版本：基于 SCR1 开源版本（Syntacore），文档对应 2026-09 学习记录

> 本文档描述 IFU 的功能规格、接口、微架构、时序与验证边界，行号均指向源文件。

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

---

## 2. 外部接口（Port List）

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
| `ifu2imem_addr_o` | out | 取指地址（[31:2]+2'b00，字对齐） | 44 |
| `imem2ifu_rdata_i` | in | 读数据 32 位 | 45 |
| `imem2ifu_resp_i` | in | 响应（NOTRDY/RDY_OK/RDY_ER） | 46 |

### 2.3 IFU <-> EXU（新 PC 接口）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `exu2ifu_pc_new_req_i` | in | 换 PC 请求（jump/branch/trap） | 49 |
| `exu2ifu_pc_new_i` | in | 新 PC 值 | 50 |

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

IFU 侧，`ifu2idu_vd_o & idu2ifu_rdy_i` 才真正消费一条指令（出队）。

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

### 3.2 关键常量（行号 77-85）

| 常量 | 值 | 说明 |
|---|---|---|
| `SCR1_IFU_Q_SIZE_WORD` | 2 | 队列能存 2 个 32 位字 |
| `SCR1_IFU_Q_SIZE_HALF` | 4 | 等价 4 个半字槽位 |
| `SCR1_TXN_CNT_W` | 3 | 在飞/丢弃计数器位宽（可数 0~7） |
| `SCR1_IFU_QUEUE_ADR_W` | $clog2(4)=2 | 队列地址位宽 |
| `SCR1_IFU_QUEUE_PTR_W` | 3 | 指针位宽（地址+1，支持相等判空） |

---

## 4. 功能规格

### 4.1 指令队列：按半字存储的环形缓冲

#### 4.1.1 为什么按半字存

RVC 使指令长度可变（16/32 位），取指块（32 位）与指令边界不再对齐。半字是
RVC(1)、RVI(2)、取指块(2) 的公共最小单位，因此队列按 16 位槽存储。

**关键场景——跨块 RVI**：一条 RVI 若起始于 2 对齐（非 4 对齐）地址，其低 16 位落在
块 A 高半槽、高 16 位落在块 B 低半槽，两次取指响应分别到达。半字存储使两个半字可在
不同拍入队并在队列中拼接。

#### 4.1.2 存储结构（行号 172-180）

- `q_data [4]`：16 位数据槽，`SCR1_IFU_Q_SIZE_HALF=4`；
- `q_err [4]`：16 位槽对应的错误标志（每槽 1 bit，按字声明但仅用低位）。

#### 4.1.3 读写指针（行号 381-411）

- `q_wptr`：写指针。`q_wptr_upd = q_flush_req | ~q_wr_none`（384）。
  正常步进：写半字 `+1`、写满字 `+2`；flush 归零（394-396）。
- `q_rptr`：读指针。步进由读大小决定：读半字 `+1`、读整字 `+2`（409-411）。

#### 4.1.4 队列状态（行号 449-458）

| 信号 | 表达式 | 含义 | 行号 |
|---|---|---|---|
| `q_ocpd_h` | wptr - rptr | 已占用半字数 | 449 |
| `q_is_empty` | rptr == wptr | 空 | 453 |
| `q_has_free_slots` | free_w > 在飞数 | 有空间容纳新响应 | 454 |
| `q_head_is_rvi` | &q_data_head[1:0] | 队头是 RVI 低半（低 2 位为 11） | 457 |
| `q_head_is_rvc` | ~q_head_is_rvi | 队头是独立 RVC | 458 |

### 4.2 指令类型译码

#### 4.2.1 判据基础（行号 279-280）

RISC-V 长度编码规则：32 位指令低 2 位恒为 `11`；压缩指令只用 `00/01/10`。
据此 `instr_lo_is_rvi = &imem2ifu_rdata_i[1:0]`、`instr_hi_is_rvi = &[17:16]`。

#### 4.2.2 类型全集（行号 282-302）

命名 `SCR1_IFU_INSTR_<高半内容>_<低半内容>`（行号 117-127）：

| 类型 | 场景 |
|---|---|
| `NONE` | 无效 |
| `RVI_HI_RVI_LO` | 一个完整 RVI（块内对齐） |
| `RVC_RVC` | 两条 RVC |
| `RVI_LO_RVC` | 低半 RVC + 高半为 RVI 低半（跨块欠账） |
| `RVC_RVI_HI` | 低半为上一 RVI 高半（拼接）+ 高半 RVC |
| `RVI_LO_RVI_HI` | 低半拼接上一 RVI + 高半为新 RVI 低半 |
| `RVC_NV` / `RVI_LO_NV` | 非对齐 new_pc 后丢弃低半，只解析高半 |

解码顺序（组合逻辑）：非对齐 new_pc → 跨块欠账标志 → hi/lo 两位组合。

#### 4.2.3 跨块欠账标志 `instr_hi_rvi_lo_ff`（行号 304-322）

- 1 bit 寄存器，记忆"上一取指块高半槽是否为某 RVI 的低半（欠账未还）"。
- `instr_hi_rvi_lo_next`：`RVI_LO_NV | RVI_LO_RVI_HI | RVI_LO_RVC` 均置位
  （即高半槽内容为 RVI_LO）。
- EXU 换 PC 请求到来时清零（312-313），因为新 PC 冲刷所有旧队列状态。

### 4.3 非对齐 new_pc（行号 259-274）

RISC-V 压缩指令要求 2 字节对齐。若 `exu2ifu_pc_new_i[1]=1`（2 对齐、非 4 对齐），
新 PC 落在取指块中部：

- `new_pc_unaligned_ff` 置位（272-274）；
- 后续解码将**丢弃块低半**，只认高半（`RVC_NV`/`RVI_LO_NV`）；
- 下次取指地址仍为 4 对齐块（IFU 请求地址恒字对齐），但解析跳过半个字。

### 4.4 队列读写大小译码

#### 4.4.1 写大小（行号 340-373）

- `WR_FULL`（2 槽）：完整 RVI、两 RVC、含欠账组合；
- `WR_HI`（1 槽）：`RVC_NV` / `RVI_LO_NV`（丢弃低半，仅存高半）；
- `WR_NONE`：IMEM 错误响应但被丢弃标记、或响应无效；
- **错误响应写 `WR_FULL` 但 `q_err` 置位**（369-370）。

#### 4.4.2 读大小（行号 328-337）

- `q_rd_hword`：队头是 RVC 或队头错误；
- `q_rd_word`：队头是 RVI（读 2 槽拼 `{next, head}`）。

### 4.5 指令输出（行号 745-753，MAX 非 bypass 路径）

```
ifu2idu_instr_o = q_head_is_rvc ? q_data_head          // 16 位 RVC
                                : {q_data_next, q_data_head};  // 32 位 RVI
```

### 4.6 IFU FSM（行号 91-94, 464-490）

仅两态：

```
IDLE ──(ifu_fetch_req)──▶ FETCH ──(ifu_stop_req)──▶ IDLE
                            ▲                           │
                            └──────── 保持 ─────────────┘
```

- 进入条件 `ifu_fetch_req = exu2ifu_pc_new_req_i & ~pipe2ifu_stop_fetch_i`（465）；
- 退出条件 `ifu_stop_req = stop_fetch | (imem 错误未丢弃 & 无新PC)`（466-467）；
- `FETCH` 态持续向 IMEM 发请求，直到停取或队列满。

### 4.7 IMEM 接口与计数逻辑

#### 4.7.1 响应译码（行号 507-513）

- `imem_resp_ok` = resp == RDY_OK；`imem_resp_er` = resp == RDY_ER；
- `imem_resp_received = ok | er`；
- `imem_resp_vd = received & ~discard_req`（有效且未被丢弃）；
- 握手完成 `imem_handshake_done = req & ack`（513）。

#### 4.7.2 取指地址寄存器（行号 528-536）

- 正常顺序：字地址 `[31:2]` 每请求 `+1`（对齐拼 `2'b00`）；
- 换 PC：`imem_addr_next = exu2ifu_pc_new_i[31:2] + handshake_done`（528-531）；
- 特殊技巧：若 `imem_addr_ff[5:2]==1111` 才整字 +1，否则仅翻转低 4 位——
  限制加法器宽度（低 [5:2] 计数，高位仅在边界进位）。

#### 4.7.3 pending 在飞事务计数（行号 538-554）

- `imem_pnd_txns_cnt_upd = handshake_done ^ resp_received`（543）；
- 请求握手 +1、收到响应 -1（553）；
- 全满（`&cnt`，即 7 笔在飞）→ `q_full`，暂停再发请求（554）。

#### 4.7.4 discard 丢弃计数（行号 556-591）

丢弃在飞响应两种情形：

1. **换 PC（new_pc 请求）**：已取到的旧指令全部无效；
2. **收到错误响应**：无法保证错误之后的数据有效，需丢弃后续全部在飞响应。

实现要点：

- `imem_resp_discard_cnt_upd = new_pc_req | imem_resp_er | (ok & discard_req)`（569）；
- 有效在飞数 `imem_vd_pnd_txns_cnt = pnd - discard`（590），决定队列可收空间；
- `discard_req` 拉高后，后续到达的指令响应被丢弃不入队（342、419）。

### 4.8 错误注入与精确异常

设计决策：**IMEM 错误不即时在 IFU 报告，而是作为一条指令的携带标志入队**。

- 错误响应照常写满 2 槽，数据是垃圾，但 `q_err[]` 随槽存储（419-444）；
- 出队时错误转为 `ifu2idu_imem_err_o`；RVI 高半槽错误有独立标志
  `ifu2idu_err_rvi_hi_o`（724-733）；
- IDU 收到 `imem_err` 不译码，直接产生 `INSTR_ACCESS_FAULT`（异常码 1）命令
  （scr1_pipe_idu.sv:134-137），随流水走到 EXU/CSR 完成 trap（MCAUSE=1、MEPC=出错指令）。

**为什么必须入队而不是当场报错**：异常必须精确绑定具体指令，且后续指令不能产生副作用。
IFU 出错时队列可能已囤正常指令、错误响应可能多条在飞，IFU 无法预知哪条指令对应哪个错。
让错误按序入队、由 IDU/EXU 消费时翻牌，异常码、出错 PC、顺序天然精确。

### 4.9 新 PC 路径（行号 596-605，MAX 无 NEW_PC_REG）

`SCR1_NEW_PC_REG` 未定义（MAX）时：

- `ifu2imem_req_o`：换 PC 请求或 FETCH 态请求，且非队列满、有空间、未停取指；
- `ifu2imem_addr_o`：new_pc 或当前顺序地址二选一；
- `ifu2imem_cmd_o` 恒为 `READ`。

### 4.10 调试接口（HDU 程序缓冲，SCR1_DBG_EN）

当 `hdu2ifu_pbuf_fetch_i`（调试器经程序缓冲注入指令）：

- 输出数据/错误/有效全部改由 HDU 程序缓冲提供（681-685, 708-712, 734-753）；
- `stop_fetch` 随 `fetch_pbuf` 拉高（scr1_pipe_top.sv:278-282），停止正常取指。

### 4.11 时钟控制接口（SCR1_CLKCTRL_EN）

`ifu2pipe_imem_txns_pnd_o = |imem_pnd_txns_cnt`（610）——门控时钟时若有在飞
IMEM 事务不能休眠。

---

## 5. 配置宏对行为的影响

| 宏 | 未定义（MAX 默认） | 定义（MIN/IC_BASE） |
|---|---|---|
| `SCR1_NO_DEC_STAGE` | 指令先入队再出队（本文 745-753 路径） | 增加 bypass，指令可直接旁路给 IDU（623-713） |
| `SCR1_NEW_PC_REG` | new_pc 直接驱动请求线（596-601） | new_pc 先进寄存器再发请求（602-605） |
| `SCR1_DBG_EN` | 无 HDU 程序缓冲 | 增加调试指令注入与 pbuf_rdy 反馈 |
| `SCR1_CLKCTRL_EN` | 无 | 输出在飞事务指示供门控时钟 |

> 注：`SCR1_CFG_RV32IMC_MAX` 均未定义这四个中的前两个 → IFU 走完整队列路径。

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

- TCM 等零延迟内存：应答与响应可同拍或错拍，pending 计数在 0/1 间摆动。

### 6.2 换 PC（flush）

```
new_pc_req  ____█____________________
flush      ____█____________________  (wptr/rptr 清零, discard 置为 pnd)
队列清空    旧指令全部作废；FSM 转向新 PC 取指
```

断言 `SCR1_SVA_IFU_NEW_PC_REQ_BEH`：换 PC 下一拍队列必须为空（806）。

### 6.3 错误响应与精确异常

```
IMEM 响应 RDY_ER：
  ① imem_resp_er → 该响应照常入队 + q_err 置位
  ② discard_cnt 增加 → 后续在飞响应全部丢弃
  ③ FSM → IDLE（停止取指）
  ④ 队头带错指令出队 → ifu2idu_imem_err_o=1
  ⑤ IDU 生成 INSTR_ACCESS_FAULT → EXU → CSR（MCAUSE=1）
```

### 6.4 非对齐 new_pc（跨块 RVI 场景）

```
new_pc = 0x...2 （2 对齐非 4 对齐）
  ① new_pc_unaligned_ff 置位
  ② 下一取指块：解码丢弃低半字，只认高半字（RVC_NV 或 RVI_LO_NV）
  ③ 若高半是 RVI 低半 → instr_hi_rvi_lo_ff 置位（欠账）
  ④ 再下一块：低半字作为上一 RVI 的高半拼接 → 完整指令还原
```

---

## 7. 断言规格（行号 769-822，仅 SCR1_TRGT_SIMULATION）

| 断言 | 内容 |
|---|---|
| `SCR1_SVA_IFU_XCHECK` | 关键输入无 X |
| `SCR1_SVA_IFU_XCHECK_REQ` | 请求有效时地址/命令无 X |
| `SCR1_SVA_IFU_DRC_UNDERFLOW` | discard 计数不下溢 |
| `SCR1_SVA_IFU_DRC_RANGE` | discard 计数 ∈ [0, pnd] |
| `SCR1_SVA_IFU_QUEUE_OVF` | 队列不溢出 |
| `SCR1_SVA_IFU_IMEM_ERR_BEH` | 内存错误后 FSM 回 IDLE 且 discard 收敛 |
| `SCR1_SVA_IFU_NEW_PC_REQ_BEH` | new_pc 请求后队列空 |
| `SCR1_SVA_IFU_IMEM_ADDR_ALIGNED` | IMEM 访问地址 [1:0]=0（恒字对齐） |
| `SCR1_SVA_IFU_STOP_FETCH` | 停取指请求后 FSM 回 IDLE |
| `SCR1_SVA_IFU_IMEM_FAULT_RVI_HI` | RVI 高半错误时 imem_err 必为 1 |

---

## 8. 假设与约束

1. IMEM 请求地址必须 4 字节对齐（IFU 自身保证，见断言 IFU_IMEM_ADDR_ALIGNED）。
2. IMEM 最大在飞事务数 < 8（`SCR1_TXN_CNT_W=3`），超出即暂停发请求。
3. 队列最多容纳 2 个 32 位字 = 4 个半字；跨块 RVI + 后续指令依赖该容量不丢取指。
4. IFU 假设取指流为合法指令流；对 `[1:0]==11` 的半字按 RVI 处理，若落入数据区
   是程序/链接层错误而非 IFU 责任。
5. `pipe2ifu_stop_fetch_i` 来自 WFI 或调试模式，IFU 不自行判断停机。
