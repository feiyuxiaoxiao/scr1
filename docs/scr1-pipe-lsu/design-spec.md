# SCR1 LSU 访存单元设计规格（Design Specification）

- 模块：`scr1_pipe_lsu`（`src/core/pipeline/scr1_pipe_lsu.sv`，351 行）
- 层次：`scr1_pipe_exu` 的子模块，例化于 `scr1_pipe_exu.sv:761-791`
- 上游：EXU（发送命令/地址/存储数据）；下游：DMEM 接口、TDU（可选）
- 参考手册：`docs/scr1_um.pdf`、RISC-V 规范（misalign 与 access fault 语义）

---

## 1. 概述与职责

`scr1_pipe_lsu` 是 SCR1 的访存执行单元，负责：

1. 把 EXU 发出的 load/store 命令转换为 DMEM 读/写请求（地址、命令、宽度、写数据）；
2. 接收 DMEM 响应（OK/ER），把读数据按命令做符号/零扩展后回送 EXU；
3. 生成数据地址非对齐异常（load/store misalign）与访问异常（access fault）；
4. 把数据地址流送 TDU 监测，并可产生数据/指令硬件断点异常（`SCR1_TDU_EN`）。

**关键定位**：LSU 是 EXU 内部唯一的多周期存储访问组件。EXU 通过
`exu2lsu_req_i` 发起、以 `lsu2exu_rdy_o`/`lsu2exu_exc_o` 完成握手；
在访存期间 EXU 处于 busy（`exu_busy`），不发射后续指令。

**实现要点**：

- 组合地址非对齐检查（在发 DMEM 请求前拦截），异常当拍返回，不访问 DMEM；
- 访问异常来自 DMEM 的 `SCR1_MEM_RESP_RDY_ER` 响应；
- 命令在请求被 ack 的当拍锁存到 `lsu_cmd_ff`，供响应期做宽度/符号扩展与异常编码。

---

## 2. 端口与接口

见源码 `:30-61`。分四组：

### 2.1 公共

| 信号 | 方向 | 位宽 | 行号 | 说明 |
|---|---|---|---|---|
| `rst_n` | in | 1 | 32 | 复位（低有效） |
| `clk` | in | 1 | 33 | 时钟（上升沿） |

### 2.2 LSU ↔ EXU

| 信号 | 方向 | 位宽 | 行号 | 说明 |
|---|---|---|---|---|
| `exu2lsu_req_i` | in | 1 | 36 | EXU 请求访存 |
| `exu2lsu_cmd_i` | in | `type_scr1_lsu_cmd_sel_e` | 37 | LSU 命令 |
| `exu2lsu_addr_i` | in | XLEN | 38 | DMEM 地址 |
| `exu2lsu_sdata_i` | in | XLEN | 39 | 存储数据 |
| `lsu2exu_rdy_o` | out | 1 | 40 | 收到 DMEM 响应 |
| `lsu2exu_ldata_o` | out | XLEN | 41 | 扩展后的读取数据 |
| `lsu2exu_exc_o` | out | 1 | 42 | LSU 异常 |
| `lsu2exu_exc_code_o` | out | `type_scr1_exc_code_e` | 43 | 异常码 |

### 2.3 LSU ↔ TDU（`SCR1_TDU_EN`）

| 信号 | 方向 | 位宽 | 行号 | 说明 |
|---|---|---|---|---|
| `lsu2tdu_dmon_o` | out | `type_scr1_brkm_lsu_mon_s` | 47 | 数据地址流监测 |
| `tdu2lsu_ibrkpt_exc_req_i` | in | 1 | 48 | 指令断点异常请求（访存触发） |
| `tdu2lsu_dbrkpt_exc_req_i` | in | 1 | 49 | 数据断点异常请求 |

### 2.4 LSU ↔ DMEM

| 信号 | 方向 | 位宽 | 行号 | 说明 |
|---|---|---|---|---|
| `lsu2dmem_req_o` | out | 1 | 53 | DMEM 请求 |
| `lsu2dmem_cmd_o` | out | `type_scr1_mem_cmd_e` | 54 | 读/写命令 |
| `lsu2dmem_width_o` | out | `type_scr1_mem_width_e` | 55 | 数据宽度 |
| `lsu2dmem_addr_o` | out | `SCR1_DMEM_AWIDTH` | 56 | DMEM 地址 |
| `lsu2dmem_wdata_o` | out | `SCR1_DMEM_DWIDTH` | 57 | 写数据 |
| `dmem2lsu_req_ack_i` | in | 1 | 58 | 请求应答 |
| `dmem2lsu_rdata_i` | in | `SCR1_DMEM_DWIDTH` | 59 | 读数据 |
| `dmem2lsu_resp_i` | in | `type_scr1_mem_resp_e` | 60 | 响应类型 |

> 数据/地址位宽使用 `SCR1_DMEM_DWIDTH`/`SCR1_DMEM_AWIDTH`（由 `scr1_arch_description.svh` 按 TCM 配置给出），与 XLEN 解耦。

---

## 3. 局部类型与信号

### 3.1 FSM 类型（`:67-70`）

```systemverilog
typedef enum logic {
    SCR1_LSU_FSM_IDLE,
    SCR1_LSU_FSM_BUSY
} type_scr1_lsu_fsm_e;
```

### 3.2 关键信号（`:77-107`）

| 分组 | 信号 | 行号 |
|---|---|---|
| FSM | `lsu_fsm_curr` / `lsu_fsm_next` / `lsu_fsm_idle` | 77-79 |
| 命令寄存器 | `lsu_cmd_upd`、`lsu_cmd_ff`、`lsu_cmd_ff_load`、`lsu_cmd_ff_store` | 82-85 |
| DMEM 命令/宽度标志 | `dmem_cmd_load`、`dmem_cmd_store`、`dmem_wdth_word`、`dmem_wdth_hword`、`dmem_wdth_byte` | 88-92 |
| 响应/请求控制 | `dmem_resp_ok`、`dmem_resp_er`、`dmem_resp_received`、`dmem_req_vd` | 95-98 |
| 异常 | `lsu_exc_req`、`dmem_addr_mslgn`、`dmem_addr_mslgn_l`、`dmem_addr_mslgn_s`、`lsu_exc_hwbrk` | 101-106 |

---

## 4. 组合控制逻辑（`:109-137`）

### 4.1 DMEM 响应/请求控制（`:114-117`）

```systemverilog
assign dmem_resp_ok       = (dmem2lsu_resp_i == SCR1_MEM_RESP_RDY_OK);
assign dmem_resp_er       = (dmem2lsu_resp_i == SCR1_MEM_RESP_RDY_ER);
assign dmem_resp_received = dmem_resp_ok | dmem_resp_er;
assign dmem_req_vd        = exu2lsu_req_i & dmem2lsu_req_ack_i & ~lsu_exc_req;
```

- `dmem_resp_received`：OK 或 ER 均视为“完成”；
- `dmem_req_vd`：请求被 ack **且无异常**时有效，用于命令锁存与 FSM 跳转。

### 4.2 load/store 命令标志（`:120-127`）

`dmem_cmd_load` 覆盖 LB/LH/LW/LBU/LHU；`dmem_cmd_store` 覆盖 SB/SH/SW。
两者由**当前组合命令** `exu2lsu_cmd_i` 解码，用于地址非对齐分类与写命令选择。

### 4.3 数据宽度标志（`:130-137`）

- word：LW/SW；hword：LH/LHU/SH；byte：LB/LBU/SB。
- 三者互斥；对 `SCR1_LSU_CMD_NONE` 全为 0。

---

## 5. LSU 命令寄存器（`:139-158`）

```systemverilog
assign lsu_cmd_upd = lsu_fsm_idle & dmem_req_vd;   // :140
always_ff @(posedge clk, negedge rst_n) begin      // :142-148
    if (~rst_n)           lsu_cmd_ff <= SCR1_LSU_CMD_NONE;
    else if (lsu_cmd_upd) lsu_cmd_ff <= exu2lsu_cmd_i;
end
```

- 复位值 `SCR1_LSU_CMD_NONE`；
- 仅在 **IDLE 且请求被 ack** 时锁存，供响应返回到达时（可能多个周期后）使用；
- `lsu_cmd_ff_load`/`lsu_cmd_ff_store`（`:151-158`）对寄存值再解码，
  用于**访问异常码**与**读数据符号扩展**。

> 为什么需要寄存器：DMEM 响应可能晚于请求，`exu2lsu_cmd_i` 到响应时就不可靠，
> 故必须锁存。地址与写数据使用组合路径（`:255-256`），因为请求当拍即被 DMEM 采样。

---

## 6. LSU FSM（`:160-191`）

### 6.1 状态

- `IDLE`：无进行中的访问；
- `BUSY`：已发出请求，等待响应。

### 6.2 状态更新（`:169-175`）

同步复位到 `IDLE`，否则每拍 `lsu_fsm_curr <= lsu_fsm_next`。

### 6.3 次态逻辑（`:178-189`）

```systemverilog
SCR1_LSU_FSM_IDLE: lsu_fsm_next = dmem_req_vd        ? BUSY : IDLE;
SCR1_LSU_FSM_BUSY: lsu_fsm_next = dmem_resp_received ? IDLE : BUSY;
```

`lsu_fsm_idle = (lsu_fsm_curr == SCR1_LSU_FSM_IDLE)`（`:191`）。

> 非对齐异常时 `dmem_req_vd=0`（被 `~lsu_exc_req` 屏蔽）且 `dmem_resp_received=0`，
> 故 FSM 恒在 `IDLE`，不产生任何 DMEM 事务。

---

## 7. 异常逻辑（`:193-230`）

### 7.1 地址非对齐（`:206-209`）

```systemverilog
assign dmem_addr_mslgn   = exu2lsu_req_i & ( (dmem_wdth_hword & exu2lsu_addr_i[0])
                                           | (dmem_wdth_word  & |exu2lsu_addr_i[1:0]));
assign dmem_addr_mslgn_l = dmem_addr_mslgn & dmem_cmd_load;
assign dmem_addr_mslgn_s = dmem_addr_mslgn & dmem_cmd_store;
```

- 半字要求 `addr[0]==0`；字要求 `addr[1:0]==0`；字节无对齐约束；
- 仅在 `exu2lsu_req_i` 有效时判定；
- 按 load/store 拆分为两类异常。

### 7.2 异常码编码（`:212-224`）

优先级 `case(1'b1)`：

| 优先级 | 条件 | 异常码 | 行号 |
|---|---|---|---|
| 1 | `dmem_resp_er` | load→`LD_ACCESS_FAULT`；store→`ST_ACCESS_FAULT`；否则 `INSTR_MISALIGN` | 214-216 |
| 2 | `lsu_exc_hwbrk`（TDU） | `BREAKPOINT` | 218 |
| 3 | `dmem_addr_mslgn_l` | `LD_ADDR_MISALIGN` | 220 |
| 4 | `dmem_addr_mslgn_s` | `ST_ADDR_MISALIGN` | 221 |
| default | — | `INSTR_MISALIGN`（占位） | 222 |

> `dmem_resp_er` 的 load/store 判定使用**寄存命令** `lsu_cmd_ff_*`，
> 因为该异常在响应期产生（`:214-215`）。

### 7.3 异常请求聚合（`:226-230`）

```systemverilog
assign lsu_exc_req = dmem_addr_mslgn_l | dmem_addr_mslgn_s
`ifdef SCR1_TDU_EN
                   | lsu_exc_hwbrk
`endif
;
```

**注意**：`lsu_exc_req` 只含**非对齐与硬件断点**；访问异常（`dmem_resp_er`）不经此信号，
而是直接进入 `lsu2exu_exc_o`（`:237`）。

---

## 8. LSU ↔ EXU 接口（`:232-248`）

```systemverilog
assign lsu2exu_rdy_o = dmem_resp_received;                 // :236
assign lsu2exu_exc_o = dmem_resp_er | lsu_exc_req;          // :237
```

- `rdy_o` 仅在收到 DMEM 响应时拉高（**不含非对齐异常**）；
- `exc_o` 包含 DMEM 访问异常 + 非对齐/hwbrk。

### 8.1 读数据扩展（`:240-248`）

| `lsu_cmd_ff` | 扩展方式 | 行号 |
|---|---|---|
| `LH` | 符号扩展到 32 位 | 242 |
| `LHU` | 零扩展 | 243 |
| `LB` | 符号扩展到 32 位 | 244 |
| `LBU` | 零扩展 | 245 |
| 其它（含 LW/NONE） | 直通 `dmem2lsu_rdata_i` | 246 |

> 使用**寄存命令** `lsu_cmd_ff`；数据取自组合 `dmem2lsu_rdata_i`，在响应当拍有效。

### 8.2 EXU 侧如何用这些信号

在 EXU 中（`scr1_pipe_exu.sv:798-807`）：

```systemverilog
lsu_req    → exu_rdy = lsu_rdy | lsu_exc_req   // 完成或非对齐异常都算 ready
```

因此非对齐异常时 `lsu_rdy=0` 但 `lsu_exc_req=1`，EXU 仍能退休该指令并走 trap。
访问异常时 `lsu_rdy=1`（`dmem_resp_received`）且 `lsu_exc_req` 由 EXU 单独并入
（EXU `exu_exc_req` 包含 `lsu_exc_req`，`:483-493`，其中 `lsu_exc_req` 指 EXU 采样的
`lsu2exu_exc_o`，即已含 `dmem_resp_er`）。

---

## 9. LSU ↔ DMEM 接口（`:250-260`）

```systemverilog
assign lsu2dmem_req_o   = exu2lsu_req_i & ~lsu_exc_req & lsu_fsm_idle;  // :254
assign lsu2dmem_addr_o  = exu2lsu_addr_i;                               // :255
assign lsu2dmem_wdata_o = exu2lsu_sdata_i;                              // :256
assign lsu2dmem_cmd_o   = dmem_cmd_store  ? SCR1_MEM_CMD_WR : SCR1_MEM_CMD_RD;   // :257
assign lsu2dmem_width_o = dmem_wdth_byte  ? SCR1_MEM_WIDTH_BYTE                  // :258
                        : dmem_wdth_hword ? SCR1_MEM_WIDTH_HWORD
                                          : SCR1_MEM_WIDTH_WORD;                 // :260
```

- `req_o` 是**单拍脉冲**（仅 IDLE），请求当拍由 DMEM 用 `dmem2lsu_req_ack_i` 应答；
- `addr`/`wdata` 直通组合信号；
- `cmd` 对非 store 一律为 RD；`width` 对非 byte/hword 一律为 WORD（NONE 不会产生请求）。

### 9.1 DMEM 事务时序

1. `IDLE`：EXU 给 `req_i=1`，`lsu2dmem_req_o=1`；DMEM 同拍 `req_ack_i=1` → `dmem_req_vd=1`，
   锁存 `lsu_cmd_ff`，FSM→`BUSY`；
2. `BUSY`：`req_o=0`，等待 `dmem2lsu_resp_i`；
3. 收到 `RDY_OK`/`RDY_ER` → `rdy_o=1`（`resp_er` 时 `exc_o=1`），FSM→`IDLE`。

---

## 10. LSU ↔ TDU 接口（`:262-275`）

仅 `SCR1_TDU_EN` 编译。

```systemverilog
assign lsu2tdu_dmon_o.vd    = exu2lsu_req_i & lsu_fsm_idle & ~tdu2lsu_ibrkpt_exc_req_i; // :267
assign lsu2tdu_dmon_o.addr  = exu2lsu_addr_i;      // :268
assign lsu2tdu_dmon_o.load  = dmem_cmd_load;       // :269
assign lsu2tdu_dmon_o.store = dmem_cmd_store;      // :270

assign lsu_exc_hwbrk = (exu2lsu_req_i & tdu2lsu_ibrkpt_exc_req_i)  // :272
                     | tdu2lsu_dbrkpt_exc_req_i;                    // :273
```

- 数据地址流（`dmon`）在 IDLE、请求当拍上报，且**指令断点命中时不报**
  （`~ibrkpt_exc_req_i`，避免把断点视为正常访存）；
- 硬件断点异常：指令型必须在访存请求当拍命中，数据型则直接命中。

---

## 11. 配置宏影响

| 宏 | 位置 | 影响 |
|---|---|---|
| `SCR1_TDU_EN` | `:26-28`、`:45-50`、`:105-107`、`:217-219`、`:227-229`、`:262-275`、`:341-347` | 增加 TDU 端口、`lsu_exc_hwbrk`、BREAKPOINT 编码、dmon 输出与 cover |
| `SCR1_TRGT_SIMULATION` | `:277-349` | 编译内建 SVA/cover |
| `SCR1_XPROP_EN` | `scr1_memif.svh:17-20` 等 | 为 mem 枚举追加 `ERROR='x` 成员（LSU 不直接用） |

无 `SCR1_RVM_EXT`/`SCR1_RVC_EXT`/`SCR1_DBG_EN` 直接分支；`SCR1_DMEM_*WIDTH` 影响端口位宽。

---

## 12. 时序约束与假设

1. **请求-应答同拍**：`lsu2dmem_req_o` 与 `dmem2lsu_req_ack_i` 在 IDLE 同拍握手；
   ack 丢失则停留 IDLE 重发（FSM 不进入 BUSY）。
2. **响应不早于请求**：IDLE 下断言 `lsu_fsm_idle |-> ~dmem_resp_received`
   （SVA `:331-334`），即 DMEM 不得在无请求时给响应。
3. **异常单热**：`{resp_er, mslgn_l, mslgn_s}` 至多一热（SVA `:326-329`）。
4. **异常必有请求**：`lsu2exu_exc_o |-> exu2lsu_req_i`（SVA `:336-339`）。
5. **非对齐零副作用**：非对齐时 `req_o=0`、FSM 恒 IDLE，DMEM 无任何事务。
6. **命令稳定性**：`exu2lsu_cmd_i`/`addr_i`/`sdata_i` 在请求被 ack 当拍有效；
   ack 后 LSU 不再依赖组合命令（异常/扩展改用 `lsu_cmd_ff`）。

---

## 13. 内建断言（`:277-347`）

| 断言 | 行号 | 检查内容 |
|---|---|---|
| `SCR1_SVA_LSU_XCHECK_CTRL` | 284-291 | `req`/FSM/TDU 控制无 X |
| `SCR1_SVA_LSU_XCHECK_CMD` | 293-296 | `req` 时 cmd/addr 无 X |
| `SCR1_SVA_LSU_XCHECK_SDATA` | 298-301 | 写请求时 sdata 无 X |
| `SCR1_SVA_LSU_XCHECK_EXC` | 303-306 | 异常时异常码无 X |
| `SCR1_SVA_LSU_IMEM_CTRL` | 308-311 | DMEM 请求时 cmd/width/addr 无 X |
| `SCR1_SVA_LSU_IMEM_ACK` | 313-316 | DMEM 请求时 ack 无 X |
| `SCR1_SVA_LSU_IMEM_WDATA` | 318-322 | 写请求时 wdata 无 X |
| `SCR1_SVA_LSU_EXC_ONEHOT` | 326-329 | 异常来源单热 |
| `SCR1_SVA_LSU_UNEXPECTED_DMEM_RESP` | 331-334 | IDLE 不应收响应 |
| `SCR1_SVA_LSU_REQ_EXC` | 336-339 | 异常必伴随请求 |
| `SCR1_COV_LSU_MISALIGN_BRKPT` | 343-346 | 覆盖：非对齐且 hwbrk 同拍 |

---

## 附录 A：端口速查

见第 2 节。共 22 个端口（含 TDU 时 +3）。

## 附录 B：相关类型/枚举

| 类型 | 定义位置 | 成员 |
|---|---|---|
| `type_scr1_lsu_cmd_sel_e` | `scr1_riscv_isa_decoding.svh:107-117` | NONE, LB, LH, LW, LBU, LHU, SB, SH, SW（9 个） |
| `type_scr1_mem_cmd_e` | `scr1_memif.svh:14-21` | RD=0, WR=1 |
| `type_scr1_mem_width_e` | `scr1_memif.svh:26-34` | BYTE=00, HWORD=01, WORD=10 |
| `type_scr1_mem_resp_e` | `scr1_memif.svh:39-47` | NOTRDY=00, RDY_OK=01, RDY_ER=10 |
| `type_scr1_brkm_lsu_mon_s` | `scr1_tdu.svh:112-117` | vd, load, store, addr |
| `type_scr1_exc_code_e` | `scr1_arch_types.svh:41-51` | 见附录 D |

## 附录 C：命令 → load/store/宽度映射

| 命令 | load | store | byte | hword | word |
|---|---|---|---|---|---|
| NONE | 0 | 0 | 0 | 0 | 0 |
| LB / LBU | 1 | 0 | 1 | 0 | 0 |
| LH / LHU | 1 | 0 | 0 | 1 | 0 |
| LW | 1 | 0 | 0 | 0 | 1 |
| SB | 0 | 1 | 1 | 0 | 0 |
| SH | 0 | 1 | 0 | 1 | 0 |
| SW | 0 | 1 | 0 | 0 | 1 |

（由 `:120-137` 解码）

## 附录 D：异常码映射

| 码 | 值 | 触发 |
|---|---|---|
| `LD_ADDR_MISALIGN` | 4 | load 半字/字地址非对齐 |
| `LD_ACCESS_FAULT` | 5 | load 收到 `RDY_ER` |
| `ST_ADDR_MISALIGN` | 6 | store 半字/字地址非对齐 |
| `ST_ACCESS_FAULT` | 7 | store 收到 `RDY_ER` |
| `BREAKPOINT` | 3 | TDU 指令/数据断点命中 |
| `INSTR_MISALIGN` | 0 | 默认占位（不应作为 LSU 真实异常） |

（码值见 `scr1_arch_types.svh:41-51`）

## 附录 E：信号 → 消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `lsu2exu_rdy_o` | EXU `exu_rdy` | 完成握手 |
| `lsu2exu_exc_o` / `_exc_code_o` | EXU 异常逻辑 | 生成精确 trap |
| `lsu2exu_ldata_o` | EXU MPRF 写回 | load 结果 |
| `lsu2dmem_*` | DMEM/AHB 或 TCM | 总线事务 |
| `lsu2tdu_dmon_o` | TDU 数据断点比较 | 硬件断点 |

## 附录 F：与 EXU/边界

- **上游边界**：EXU 提供 `exu2lsu_req_i`、命令、地址（`ialu_addr_res`）、存储数据（`mprf2exu_rs2_data_i`）；
- **下游边界**：LSU 驱动 DMEM 接口，响应回来经 EXU 选择写回 MPRF；
- **异常汇总**：LSU 异常在 EXU 异常优先级编码中排第 3（`:514-529`），
  低于 IDU 的 `exc_req`、高于 CSR 与分支非对齐；
- **busy 传播**：`lsu_req` 使 `exu_busy=1`，阻塞 IDU 发射直到完成
  （EXU `:798-812`）。
