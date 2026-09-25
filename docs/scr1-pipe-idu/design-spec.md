# SCR1 IDU 指令译码单元设计规格书（Design Specification）

- 模块：`scr1_pipe_idu`
- 源文件：`src/core/pipeline/scr1_pipe_idu.sv`（940 行）
- 位置：SCR1 RV32 内核流水线顶层 `scr1_pipe_top` 的第二级（IFU 之后、EXU 之前）
- 版本：基于 SCR1 开源版本（Syntacore），文档对应 2026-09 学习记录

> 本文档描述 IDU 的功能规格、接口、微架构、时序与验证边界，行号均指向源文件。
> 全文以 `SCR1_CFG_RV32IMC_MAX`（定义 `SCR1_RVI_EXT` / `SCR1_RVM_EXT` / `SCR1_RVC_EXT`，
> 不定义 `SCR1_RVE_EXT` / `SCR1_NO_EXE_STAGE`）为主线，配置相关的分支单独标注。

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
- 附录 B 局部参数、字段与枚举
- 附录 C 指令译码总表
- 附录 D 立即数（imm）重组规则
- 附录 E 关键信号 → 消费者映射

---

## 1. 概述与设计目标

IDU（Instruction Decoder Unit，指令译码单元）是流水线的第二级。它负责：

1. 接收 IFU 送来的指令（16 位 RVC 或 32 位 RVI，低半字对齐存放）；
2. **纯组合地**把指令展开为 EXU 可直接执行的命令字 `idu2exu_cmd_o`；
3. 提取并重组立即数、寄存器号，判定操作类型与写回来源；
4. 识别非法指令，产生 `ILLEGAL_INSTR` 精确异常；
5. 透传 IFU 的取指异常信息，把 `INSTR_ACCESS_FAULT` 绑定到具体指令；
6. 输出 `use_rs1/use_rs2/use_rd/use_imm` 供 EXU 做操作数时钟门控。

IDU **不含任何时序逻辑**（除仿真断言用的 `clk`/`rst_n`，`:25-28`），全部译码在一个 `always_comb` 内完成（`:95-909`）。它不缓存指令、不改变指令顺序、不做 flow control——`idu2exu_req_o` 与 `idu2ifu_rdy_o` 只是把 IFU 与 EXU 的握手信号**直连透传**（`:80-81`）。

### 1.1 设计权衡

- **纯组合译码换取零延迟**：IDU 不引入额外流水寄存器，译码延迟隐藏在同拍组合路径里，最大化吞吐。
- **统一命令结构**：RVI/RVC 全部指令都归约到 `type_scr1_exu_cmd_s` 的固定字段（附录 B.4），EXU 无需感知指令长度或编码。
- **默认值 + 分支复写**：先给所有字段赋默认值（`:97-131`），再由各指令分支覆写，避免锁存推断并保证未用字段确定。
- **立即数重组合并到 IDU**：RISC-V 变长立即数散落在指令各位，IDU 用拼接表达式一次成型，EXU 直接用。
- **非法判定分层**：`rvi_illegal` / `rvc_illegal` / `rve_illegal` 三个独立标志，最后统一收口（`:871-907`），便于按配置裁剪。
- **`instr_rvc` 复用为故障标记**：取指故障时 `instr_rvc` 被复用表达"故障落在 RVI 高半字"（`:137`、附录 B.4 注）。

### 1.2 术语与命名约定

| 术语 | 含义 |
|---|---|
| RVI | 32 位基础指令，低 2 位恒为 `11` |
| RVC0/RVC1/RVC2 | 压缩指令按低 2 位的三个象限（`00`/`01`/`10`） |
| RVE | RV32E 嵌入式 16 寄存器子集（`x0-x15`） |
| funct3 / funct7 / funct12 | RVI 中 `instr[14:12]` / `instr[31:25]` / `instr[31:20]`；RVC 的 funct3 取 `instr[15:13]` |
| 命令字（command） | `type_scr1_exu_cmd_s`，IDU→EXU 的打包译码结果 |
| use_* | 操作数使用标志，供 EXU 时钟门控，不影响功能语义 |
| zimm | CSR 立即数指令（CSRRxI）中的 5 位零扩展立即数，复用 `rs1_addr` 字段 |

**命名规则**：`<生产者>2<消费者>_<信号名>_<o|i>`，例如 `ifu2idu_instr_i`（IFU 送给 IDU 的指令）。
字段提取命名：`instr_type`（指令类型）、`rvi_opcode`（操作码）、`funct3/7/12`、`shamt`。

---

## 2. 外部接口（Port List）

共 12 个端口（含条件编译）。完整清单见附录 A，本节给出分组与语义。

### 2.1 控制信号（仅仿真）

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `rst_n` | in | 复位（低有效），仅 `SCR1_TRGT_SIMULATION` | 26 |
| `clk` | in | 时钟，仅 `SCR1_TRGT_SIMULATION` | 27 |

IDU 是纯组合电路，这两根线只服务于 SVA 断言（`:918-936`）。综合时（非仿真目标）端口被裁掉。

### 2.2 IFU <-> IDU

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `ifu2idu_instr_i` | in | IFU 指令（`SCR1_IMEM_DWIDTH` 位，16/32 位有效） | 32 |
| `ifu2idu_imem_err_i` | in | 取指异常（instruction access fault） | 33 |
| `ifu2idu_err_rvi_hi_i` | in | 取指故障是否落在未对齐 RVI 的高半字 | 34 |
| `ifu2idu_vd_i` | in | 指令有效 | 35 |
| `idu2ifu_rdy_o` | out | IDU 就绪（= EXU 就绪） | 31 |

### 2.3 IDU <-> EXU

| 信号 | 方向 | 说明 | 行号 |
|---|---|---|---|
| `idu2exu_req_o` | out | IDU 请求（= IFU 有效） | 38 |
| `idu2exu_cmd_o` | out | 译码命令字 | 39 |
| `idu2exu_use_rs1_o` | out | 指令使用 rs1 | 40 |
| `idu2exu_use_rs2_o` | out | 指令使用 rs2 | 41 |
| `idu2exu_use_rd_o` | out | 指令使用 rd（仅 `SCR1_NO_EXE_STAGE` 未定义） | 43 |
| `idu2exu_use_imm_o` | out | 指令使用立即数（仅 `SCR1_NO_EXE_STAGE` 未定义） | 44 |
| `exu2idu_rdy_i` | in | EXU 就绪 | 46 |

### 2.4 握手约定：透明直连

IDU 不做接收/发送缓冲，握手信号直接透传（`:80-82`）：

| 表达式 | 行号 | 含义 |
|---|---|---|
| `idu2ifu_rdy_o = exu2idu_rdy_i` | 80 | IFU 能否出队，取决于 EXU 能否接收 |
| `idu2exu_req_o = ifu2idu_vd_i` | 81 | IFU 有指令有效，就向 EXU 提请求 |
| `instr = ifu2idu_instr_i` | 82 | 指令直接进入译码组合逻辑 |

因此 IFU↔IDU↔EXU 实际是**一条组合通路**：只要 `ifu2idu_vd_i & exu2idu_rdy_i` 同时为 1，该指令当拍即被 IFU 出队、被 EXU 接收，译码结果当拍可用。缓冲与背压完全由 IFU 队列实现（见 IFU 规格书 4.1）。

> 注意：因为 `idu2ifu_rdy_o` 反映的是 EXU 就绪，IDU 自身永远"就绪"，从不产生背压。

---

## 3. 内部结构（微架构）

### 3.1 结构框图

```
                       idu2exu_use_rs1_o/use_rs2_o/use_rd_o/use_imm_o
                                        │
 ifu2idu_instr_i ──► instr ──► 字段提取 ──► 4 层译码 always_comb ──► idu2exu_cmd_o
 ifu2idu_imem_err_i ─────────► (故障短路)                          │
 ifu2idu_err_rvi_hi_i ────────►                                    ├──► idu2exu_req_o
                                                                   │    (= ifu2idu_vd_i)
 exu2idu_rdy_i ────────────────────────────────────────────────────┴──► idu2ifu_rdy_o
                                                                        (= exu2idu_rdy_i)
```

IDU 是单组合块的"字段提取 → 分层译码 → 命令输出"结构，无内部状态。

### 3.2 译码分层

`always_comb`（`:95-909`）自上而下分四层：

| 层 | 行号 | 作用 |
|---|---|---|
| L1 默认值 | `:97-131` | 给命令字与 use_* 赋安全默认，清零三个 illegal 标志 |
| L2 故障短路 | `:134-138` | `if (ifu2idu_imem_err_i)` → 取指异常，**跳过整棵译码树** |
| L3 指令类型分派 | `:139-864` | `case (instr_type)`：RVI / RVC0 / RVC1 / RVC2 / default |
| L4 非法收口 | `:871-907` | 任一 illegal 标志置位 → 清空执行字段 + 置 `ILLEGAL_INSTR` |

L2 是 if/else，故障时 else 分支（`:138`）不执行，因此 L3 完全不参与——**取指故障指令不会被误译码**。

### 3.3 字段提取

| 信号 | 表达式 | 行号 |
|---|---|---|
| `instr_type` | `type_scr1_instr_type_e'(instr[1:0])` | 85 |
| `rvi_opcode` | `type_scr1_rvi_opcode_e'(instr[6:2])` | 88 |
| `funct3` | RVI 取 `instr[14:12]`，RVC 取 `instr[15:13]` | 89 |
| `funct7` | `instr[31:25]`（RVI） | 90 |
| `funct12` | `instr[31:20]`（RVI SYSTEM） | 91 |
| `shamt` | `instr[24:20]`（RVI 移位） | 92 |

`instr_type` 枚举见 `scr1_riscv_isa_decoding.svh:15-20`；`rvi_opcode` 枚举见 `:25-37`。
注意 **RVC 与 RVI 共用 `funct3`**：同一根线按 `instr_type` 选择位段，是 RVC 复用 RVI 译码框架的关键。

### 3.4 输出命令结构 `type_scr1_exu_cmd_s`

定义于 `scr1_riscv_isa_decoding.svh:161-182`，字段清单与用途见附录 B.4。IDU 每个字段都由某条指令分支赋值，未涉及者保持 L1 默认。

---

## 4. 功能规格

### 4.1 默认值与字段清零（`:97-131`）

每拍先把命令字置于"空操作"基线，防止组合锁存：

| 字段 | 默认值 | 行号 |
|---|---|---|
| `instr_rvc` | `1'b0` | 97 |
| `ialu_op` | `REG_REG` | 98 |
| `ialu_cmd` | `NONE` | 99 |
| `sum2_op` | `PC_IMM` | 100 |
| `lsu_cmd` | `NONE` | 101 |
| `csr_op` | `REG` | 102 |
| `csr_cmd` | `NONE` | 103 |
| `rd_wb_sel` | `NONE` | 104 |
| `jump_req/branch_req/mret_req/fencei_req/wfi_req` | `0` | 105-109 |
| `rs1_addr/rs2_addr/rd_addr/imm` | `'0` | 110-113 |
| `exc_req` | `0` | 114 |
| `exc_code` | `INSTR_MISALIGN` | 115 |
| `use_rs1/use_rs2/use_rd/use_imm` | `0` | 118-123 |
| `rvi_illegal` / `rve_illegal` / `rvc_illegal` | `0` | 125-131 |

### 4.2 IMEM 故障短路（`:134-138`）

```systemverilog
if (ifu2idu_imem_err_i) begin
    idu2exu_cmd_o.exc_req   = 1'b1;
    idu2exu_cmd_o.exc_code  = SCR1_EXC_CODE_INSTR_ACCESS_FAULT;   // 4'd1
    idu2exu_cmd_o.instr_rvc = ifu2idu_err_rvi_hi_i;
end else begin
    ...
end
```

- 取指故障优先于一切译码，直接产出 `INSTR_ACCESS_FAULT`（异常码 1，`scr1_arch_types.svh:43`）。
- `instr_rvc` 在此被**复用**：置为 `err_rvi_hi_i`，告诉 EXU "这条取指故障是否属于 RVI 的高半字"（影响增量 PC 的计算）。正常译码路径下 `instr_rvc` 表示指令是否为 RVC（如 `:499`）。
- 故障时执行字段保持默认（`NONE`/`0`），不会产生副作用。

### 4.3 指令类型分派 `case (instr_type)`（`:139-864`）

| 象限 | 行号 | 覆盖指令 |
|---|---|---|
| `SCR1_INSTR_RVI` | 140-493 | 全部 RV32I + M |
| `SCR1_INSTR_RVC0` | 498-543 | C.ADDI4SPN / C.LW / C.SW |
| `SCR1_INSTR_RVC1` | 546-719 | C.ADDI/NOP / C.JAL / C.LI / C.ADDI16SP / C.LUI / C.SRLI / C.SRAI / C.ANDI / C.SUB / C.XOR / C.OR / C.AND / C.J / C.BEQZ / C.BNEZ |
| `SCR1_INSTR_RVC2` | 722-851 | C.SLLI / C.LWSP / C.MV / C.JR / C.EBREAK / C.JALR / C.ADD / C.SWSP |
| `default` | 853-863 | 未定义象限 |

RVI 分支先统一赋 `rs1_addr/rs2_addr/rd_addr`（`:141-143`），再按 `rvi_opcode` 细分。

### 4.4 RVI 译码（`:140-493`）

#### 4.4.1 opcode 总览

| opcode | 行号 | 关键设置 |
|---|---|---|
| `AUIPC` | 145-156 | `sum2_op=PC_IMM`、`rd_wb_sel=SUM2`、`imm={instr[31:12],12'b0}` |
| `LUI` | 158-168 | `rd_wb_sel=IMM`、`imm={instr[31:12],12'b0}` |
| `JAL` | 170-182 | `sum2_op=PC_IMM`、`rd_wb_sel=INC_PC`、`jump_req=1` |
| `LOAD` | 184-204 | `use_rs1`、`sum2_op=REG_IMM`、`rd_wb_sel=LSU`、`lsu_cmd` 按 funct3 |
| `STORE` | 206-223 | `use_rs1+use_rs2`、`sum2_op=REG_IMM`、`lsu_cmd` 按 funct3 |
| `OP` | 225-273 | `use_rs1+use_rs2`、`ialu_op=REG_REG`、`rd_wb_sel=IALU`、`ialu_cmd` 按 funct7/funct3 |
| `OP_IMM` | 275-320 | `use_rs1`、`ialu_op=REG_IMM`、`rd_wb_sel=IALU`、`ialu_cmd` 按 funct3 |
| `MISC_MEM` | 322-339 | FENCE=NOP；FENCE.I 置 `fencei_req` |
| `BRANCH` | 341-363 | `use_rs1+use_rs2`、`branch_req=1`、`sum2_op=PC_IMM`、`ialu_op=REG_REG`、`ialu_cmd` 按 funct3 |
| `JALR` | 365-384 | `use_rs1`、`sum2_op=REG_IMM`、`rd_wb_sel=INC_PC`、`jump_req=1` |
| `SYSTEM` | 386-487 | 见 4.4.3 |

#### 4.4.2 LOAD / STORE 的 funct3 译码

| funct3 | LOAD 行号 | lsu_cmd | | funct3 | STORE 行号 | lsu_cmd |
|---|---|---|---|---|---|---|
| 000 | 194 | `LB` | | 000 | 215 | `SB` |
| 001 | 195 | `LH` | | 001 | 216 | `SH` |
| 010 | 196 | `LW` | | 010 | 217 | `SW` |
| 100 | 197 | `LBU` | | 其他 | 218 | illegal |
| 101 | 198 | `LHU` | | | | |
| 其他 | 199 | illegal | | | | |

#### 4.4.3 SYSTEM（`:386-487`）

统一先置 `imm = SCR1_XLEN'({funct3, instr[31:20]})`（`:391`），即把 **funct3 与 CSR 地址打包**进 imm 供 CSR 单元使用。再按 funct3：

- `funct3=000`：无 CSR，撤销 `use_rd/use_imm`（`:394-397`），按 `{instr[19:15], instr[11:7]}` 细分（`:398`）：

| funct12 | 指令 | 行号 | 动作 |
|---|---|---|---|
| `0x000` | ECALL | 401-405 | `exc_req=1, exc_code=ECALL_M(11)` |
| `0x001` | EBREAK | 406-410 | `exc_req=1, exc_code=BREAKPOINT(3)` |
| `0x302` | MRET | 411-414 | `mret_req=1` |
| `0x105` | WFI | 415-418 | `wfi_req=1` |
| 其他 | illegal | 419 | |
| `{rs1,rd}≠0` | illegal | 422 | |

- `funct3 ∈ {001,010,011}`（CSRRW/S/C）：`use_rs1=1`、`rd_wb_sel=CSR`、`csr_op=REG`，`csr_cmd` 分别为 `WRITE/SET/CLEAR`（`:425-454`）。
- `funct3 ∈ {101,110,111}`（CSRRWI/SI/CI）：`use_rs1=1`（作为 zimm）、`rd_wb_sel=CSR`、`csr_op=IMM`，`csr_cmd` 分别为 `WRITE/SET/CLEAR`（`:455-484`）。
- 其余 funct3 illegal（`:485`）。

CSR 指令不设置 `sum2_op`/`lsu_cmd`/`jump_req`；`csr_op` 决定操作数第二来源是 rs1 还是 zimm。

#### 4.4.4 OP / OP_IMM 的字段译码

**OP（funct7/funct3，`:233-269`）**

| funct7 | funct3 | IALU 命令 | 行号 |
|---|---|---|---|
| `0000000` | 000/001/010/011/100/101/110/111 | ADD/SLL/SLT/SLTU/XOR/SRL/OR/AND | 236-243 |
| `0100000` | 000/101 | SUB/SRA（其他 illegal） | 249-251 |
| `0000001`（M 扩展） | 000-111 | MUL/MULH/MULHSU/MULHU/DIV/DIVU/REM/REMU | 257-264 |
| 其他 funct7 | — | illegal | 268 |

**OP_IMM（`:284-316`）**

| funct3 | 指令 | 行号 | 特殊 |
|---|---|---|---|
| 000 | ADDI | 285 | |
| 010 | SLTI | 286 | |
| 011 | SLTIU | 287 | |
| 100 | XORI | 288 | |
| 110 | ORI | 289 | |
| 111 | ANDI | 290 | |
| 001 | SLLI | 291-300 | 仅 funct7=`0000000`，imm=`XLEN'(shamt)` |
| 101 | SRLI/SRAI | 301-315 | funct7=`0000000`/`0100000` |

移位立即数用 `SCR1_XLEN'(shamt)`（`:295`、`:305`、`:310`）显式零扩展为 XLEN 位。

### 4.5 RVC0（`:498-543`）

进入即置 `instr_rvc=1`、`use_rs1=1`、`use_imm=1`（`:499-503`）。`funct3` 取 `instr[15:13]`。

| funct3 | 指令 | 行号 | 要点 |
|---|---|---|---|
| 000 | C.ADDI4SPN | 505-517 | `{instr[12:5]}==0` 则 illegal；rs1=SP；`ialu=ADD/REG_IMM`；rd=`{2'b01,instr[4:2]}` |
| 010 | C.LW | 518-529 | `sum2=REG_IMM`、`lsu=LW`、`rd_wb_sel=LSU`；rs1=`{2'b01,instr[9:7]}` |
| 110 | C.SW | 530-538 | `sum2=REG_IMM`、`lsu=SW`；rs1/rs2 均为 `{2'b01,...,}` |
| default | illegal | 539-541 | |

RVC 压缩寄存器映射规则：`{2'b01, instr[4:2]}`（rd' 组）与 `{2'b01, instr[9:7]}`（rs1' 组）——把 3 位压缩寄存器号映射到 x8-x15。

### 4.6 RVC1（`:546-719`）

进入即置 `instr_rvc=1`、`use_rd=1`、`use_imm=1`（`:547-551`，`SCR1_NO_EXE_STAGE` 未定义时）。

| funct3 | 指令 | 行号 | 要点 |
|---|---|---|---|
| 000 | C.ADDI / C.NOP | 553-565 | rs1=rd=`instr[11:7]`；`ADD/REG_IMM`；C.NOP 是 rd=x0 的 ADDI |
| 001 | C.JAL | 566-573 | 仅 RV32；`sum2=PC_IMM`、`rd_wb_sel=INC_PC`、`jump_req`、rd=RA |
| 010 | C.LI | 574-582 | `rd_wb_sel=IMM`；rd=`instr[11:7]` |
| 011 | C.ADDI16SP / C.LUI | 583-603 | `rd==SP` 走 ADDI16SP（rs1=rd=SP，ADD/REG_IMM）；否则 C.LUI（`rd_wb_sel=IMM`，imm 高 20 位） |
| 100 | 移位/逻辑组 | 604-678 | rs1=rd=`{2'b01,instr[9:7]}`，rs2=`{2'b01,instr[4:2]}`，见下 |
| 101 | C.J | 679-687 | `sum2=PC_IMM`、`jump_req` |
| 110 | C.BEQZ | 688-702 | rs1=`{2'b01,instr[9:7]}`、rs2=x0、`SUB_EQ`、`branch_req` |
| 111 | C.BNEZ | 703-717 | 同上，`SUB_NE` |

RVC1 的 8 个 funct3 取值**全覆盖**，因此 `case` 无 `default` 分支。

**funct3=100 子译码（`:612-677`）**，先按 `instr[11:10]`：

| instr[11:10] | 指令 | 行号 | ialu_op |
|---|---|---|---|
| 00 | C.SRLI | 613-623 | REG_IMM |
| 01 | C.SRAI | 624-634 | REG_IMM |
| 10 | C.ANDI | 635-644 | REG_IMM |
| 11 | 再按 `{instr[12], instr[6:5]}` | 645-676 | REG_REG |

`instr[11:10]=11` 时 `{instr[12],instr[6:5]}`：`000→C.SUB`、`001→C.XOR`、`010→C.OR`、`011→C.AND`，其余 illegal（`:672-674`）。C.SRLI/C.SRAI 要求 `instr[12]==0`，C.ANDI 的 `instr[12]` 作为立即数符号位。

### 4.7 RVC2（`:722-851`）

进入即置 `instr_rvc=1`、`use_rs1=1`（`:723-724`）。

| funct3 | 指令 | 行号 | 要点 |
|---|---|---|---|
| 000 | C.SLLI | 726-742 | `instr[12]==0`；rs1=rd=`instr[11:7]`；imm=`{27'd0,instr[6:2]}`；`SLL/REG_IMM` |
| 010 | C.LWSP | 743-759 | `instr[11:7]==0` 则 illegal；rs1=SP；rd=`instr[11:7]`；`lsu=LW` |
| 100 | C.MV / C.JR / C.EBREAK / C.JALR / C.ADD | 760-830 | 按 `instr[12]` 再分，见下 |
| 110 | C.SWSP | 831-846 | rs1=SP、rs2=`instr[6:2]`；`lsu=SW` |
| default | illegal | 847-849 | |

**funct3=100 子译码**

`instr[12]==0`（`:761-790`）：
- `instr[6:2]!=0` → **C.MV**（`:763-776`）：`ADD/REG_REG`，rs1=x0、rs2=`instr[6:2]`、rd=`instr[11:7]`。
- 否则 → **C.JR**（`:779-789`）：要求 rd≠0；`sum2=REG_IMM`、`jump_req`、rs1=`instr[11:7]`、imm=0。

`instr[12]==1`（`:791-829`）：
- `instr[11:2]==0` → **C.EBREAK**（`:793-795`）：`exc_req=1, exc_code=BREAKPOINT`。
- 否则 `instr[6:2]==0` → **C.JALR**（`:796-811`）：rs1=`instr[11:7]`、rd=RA、`INC_PC`、`jump_req`、imm=0。
- 否则 → **C.ADD**（`:812-827`）：`ADD/REG_REG`，rs1=`instr[11:7]`、rs2=`instr[6:2]`、rd=`instr[11:7]`。

C.JR / C.JALR / C.JAL / C.J 的 imm=0（`:786`、`:808`、`:572` 等）；跳转目标由 SUM2 依据 `sum2_op` 计算（C.J 用 PC+imm，C.JR 用 rs1+0）。

### 4.8 `default` 象限（`:853-863`）

| 条件 | 行为 | 行号 |
|---|---|---|
| `SCR1_XPROP_EN` | `rvi_illegal=1`（让未定义象限可被 X-prop 检测） | 853-857 |
| 非 RVC 配置 | `instr_rvc=1; rvi_illegal=1` | 858-862 |

当 `SCR1_RVC_EXT` 未定义时，`instr[1:0]≠11` 的编码在本 ISA 子集下非法。

### 4.9 非法指令收口（`:871-907`）

```systemverilog
if (rvi_illegal | rvc_illegal | rve_illegal) begin
    ialu_cmd=NONE; lsu_cmd=NONE; csr_cmd=NONE; rd_wb_sel=NONE;
    jump_req=0; branch_req=0; mret_req=0; fencei_req=0; wfi_req=0;
    use_rs1=0; use_rs2=0; use_rd=0;
    use_imm = <0 或 1>;  imm = <instr 或 保持>;
    exc_req=1; exc_code=ILLEGAL_INSTR;
end
```

- 三个 illegal 标志按配置裁剪：`rvc_illegal` 仅 `SCR1_RVC_EXT`（`:873-875`）、`rve_illegal` 仅 `SCR1_RVE_EXT`（`:876-878`）。
- **清空所有执行字段**是必须的：非法指令不应触发写回、访存、跳转等副作用。
- `SCR1_MTVAL_ILLEGAL_INSTR_EN`（在 `scr1_arch_types.svh:33` **无条件定义**）决定是否把整条指令编码放进 `imm`（`:902`），供 EXU 写入 `mtval`：
  - 定义且未 `SCR1_NO_EXE_STAGE`：`use_imm=1`、`imm=instr`；
  - 未定义：`use_imm=0`、`imm` 保持默认。
- 异常码 `ILLEGAL_INSTR = 4'd2`（`scr1_arch_types.svh:44`）。

### 4.10 `use_*` 时钟门控信号

`use_rs1/use_rs2/use_rd/use_imm` 是纯性能提示，参与 EXU 的操作数读/写使能，不改变功能。典型置位：

- `use_rs1`：携带 rs1 的指令（LOAD/STORE/OP/OP_IMM/BRANCH/JALR/CSR）；RVC0/RVC2 进入时无条件置位（`:500`、`:724`）。
- `use_rs2`：双源指令（STORE/OP/BRANCH）与 C.SW/C.SWSP/C.BEQZ/C.BNEZ/C.SUB 等。
- `use_rd`：有写回的指令；`SCR1_NO_EXE_STAGE` 未定义时存在。
- `use_imm`：使用立即数的指令；`SCR1_NO_EXE_STAGE` 未定义时存在。

> RVC2 无条件置 `use_rs1`（`:724`）会让 C.MV/C.EBREAK 等也读一次 rs1，属于安全的过度使能（rs1=x0 或结果被忽略），只影响门控效率。

---

## 5. 配置宏对行为的影响

| 宏 | 定义位置 | 对 IDU 的影响 |
|---|---|---|
| `SCR1_RVC_EXT` | MAX `arch_description.svh:73` | 编译 RVC0/1/2 译码（`:495-851`）；否则 default 象限置 illegal（`:858-862`） |
| `SCR1_RVE_EXT` | MIN `:102` | 启用 `rve_illegal` 检查，禁止写 x16+（如 `:202`、`:271`、`:318`、`:361`、`:382`） |
| `SCR1_RVM_EXT` | MAX `:72` | 启用 OP 的 `funct7=0000001`（M 扩展，`:254-267`）；同时抬高 IALU 命令合法上界（断言 `:930-934`） |
| `SCR1_NO_EXE_STAGE` | MIN `:106`、custom `:135` | 裁掉 `use_rd/use_imm` 端口与赋值（`:42-45`、`:120-123` 等）；MAX **不定义** |
| `SCR1_MTVAL_ILLEGAL_INSTR_EN` | `scr1_arch_types.svh:33`（无条件） | 非法指令把 `instr` 放入 `imm` 供 `mtval`（`:896-903`） |
| `SCR1_XPROP_EN` | 仿真选项 `:207` | default 象限置 `rvi_illegal`（`:853-857`） |
| `SCR1_TRGT_SIMULATION` | 仿真选项 `:205` | 暴露 `clk/rst_n` 端口并启用断言（`:25-28`、`:911-938`） |

**MAX 参考路径**：定义 `RVI/RVM/RVC`，不定义 `RVE/NO_EXE_STAGE`。即完整 RVI+RVC+M、x0-x31、保留 `use_rd/use_imm`、非法指令带 `mtval`。

---

## 6. 时序行为示例

IDU 是组合级，以下"拍"只表示其输入/输出在同一有效拍内的对应关系。

### 6.1 顺序 `addi x1, x1, 1`

`ifu2idu_vd_i=1`、`exu2idu_rdy_i=1`，当拍 `idu2exu_req_o=1`、`cmd.ialu_cmd=ADD`、`ialu_op=REG_IMM`、`rd_wb_sel=IALU`、`rs1_addr=1`、`rd_addr=1`、`imm=1`。

### 6.2 取指故障

`ifu2idu_imem_err_i=1` 时，无论 `instr` 内容如何，当拍 `cmd.exc_req=1`、`exc_code=INSTR_ACCESS_FAULT`，执行字段全默认，`instr_rvc=err_rvi_hi_i`。

### 6.3 非法指令（如 funct7 非法的 OP）

译码分支置 `rvi_illegal=1`，随后收口清空所有执行字段，`exc_req=1`、`exc_code=ILLEGAL_INSTR`、`imm=instr`（若 `MTVAL...` 使能）。

### 6.4 分支

`beq`：`branch_req=1`、`sum2_op=PC_IMM`、`ialu_cmd=SUB_EQ`、`ialu_op=REG_REG`、`imm` 为符号扩展的分支偏移。EXU 依 `sum2` 结果决定是否跳转。

### 6.5 背压

若 `exu2idu_rdy_i=0`，则 `idu2ifu_rdy_o=0`，IFU 不出队，`instr` 保持不变（IFU 队列保持队首），译码输出稳定。

---

## 7. 断言规格（`:911-938`，仅 `SCR1_TRGT_SIMULATION`）

| 断言 | 行号 | 检查内容 |
|---|---|---|
| `SCR1_SVA_IDU_XCHECK` | 918-921 | `negedge clk` 上 `{ifu2idu_vd_i, exu2idu_rdy_i}` 无 X |
| `SCR1_SVA_IDU_IALU_CMD_RANGE` | 925-936 | 有效且非取指故障时，`ialu_cmd` 落在 `[NONE, SRA]`（无 M）或 `[NONE, REMU]`（含 M） |

`IALU_CMD_RANGE` 上界随 `SCR1_RVM_EXT` 切换（`:930-934`），保证译码不会产生枚举越界值。

---

## 8. 假设与约束

1. 上游 IFU 保证 `ifu2idu_instr_i` 的低半字有效且指令长度由低 2 位标识；
2. 下游 EXU 在 `exu2idu_rdy_i=1` 时接收命令，IDU 不缓冲；
3. `use_*` 信号只作门控提示，其过度置位不影响正确性；
4. 非法指令必须无副作用——本设计通过收口清空执行字段保证；
5. 立即数重组遵循 RISC-V 规范的位序，详见附录 D；
6. RVE 配置下，任何写 x16-x31 的编码都判非法（`rve_illegal`）。

---

## 附录 A：端口清单（12）

### A.1 控制（`:25-28`，仅 `SCR1_TRGT_SIMULATION`）

| 端口 | 方向 | 行号 |
|---|---|---|
| `rst_n` | in | 26 |
| `clk` | in | 27 |

### A.2 IFU ↔ IDU（`:30-35`）

| 端口 | 方向 | 行号 |
|---|---|---|
| `idu2ifu_rdy_o` | out | 31 |
| `ifu2idu_instr_i` | in | 32 |
| `ifu2idu_imem_err_i` | in | 33 |
| `ifu2idu_err_rvi_hi_i` | in | 34 |
| `ifu2idu_vd_i` | in | 35 |

### A.3 IDU ↔ EXU（`:37-46`）

| 端口 | 方向 | 行号 |
|---|---|---|
| `idu2exu_req_o` | out | 38 |
| `idu2exu_cmd_o` | out | 39 |
| `idu2exu_use_rs1_o` | out | 40 |
| `idu2exu_use_rs2_o` | out | 41 |
| `idu2exu_use_rd_o` | out（仅无 `SCR1_NO_EXE_STAGE`） | 43 |
| `idu2exu_use_imm_o` | out（仅无 `SCR1_NO_EXE_STAGE`） | 44 |
| `exu2idu_rdy_i` | in | 46 |

---

## 附录 B：局部参数、字段与枚举

### B.1 localparam（`:53-55`）

| 名称 | 值 | 用途 |
|---|---|---|
| `SCR1_MPRF_ZERO_ADDR` | 5'd0 | x0 |
| `SCR1_MPRF_RA_ADDR` | 5'd1 | x1（返回地址） |
| `SCR1_MPRF_SP_ADDR` | 5'd2 | x2（栈指针） |

### B.2 内部信号（`:61-74`）

| 信号 | 位宽 | 行号 |
|---|---|---|
| `instr` | `SCR1_IMEM_DWIDTH` | 61 |
| `instr_type` | `type_scr1_instr_type_e` | 62 |
| `rvi_opcode` | `type_scr1_rvi_opcode_e` | 63 |
| `rvi_illegal` | 1 | 64 |
| `funct3` | 3 | 65 |
| `funct7` | 7 | 66 |
| `funct12` | 12 | 67 |
| `shamt` | 5 | 68 |
| `rvc_illegal` | 1（仅 `SCR1_RVC_EXT`） | 70 |
| `rve_illegal` | 1（仅 `SCR1_RVE_EXT`） | 73 |

### B.3 指令类型枚举（`scr1_riscv_isa_decoding.svh:15-20`）

| 名称 | 值 |
|---|---|
| `SCR1_INSTR_RVC0` | 2'b00 |
| `SCR1_INSTR_RVC1` | 2'b01 |
| `SCR1_INSTR_RVC2` | 2'b10 |
| `SCR1_INSTR_RVI` | 2'b11 |

### B.4 命令结构 `type_scr1_exu_cmd_s`（`scr1_riscv_isa_decoding.svh:161-182`）

| 字段 | 类型 | 含义 |
|---|---|---|
| `instr_rvc` | logic | 是否 RVC；取指故障时表示"故障在 RVI 高半字" |
| `ialu_op` | `type_scr1_ialu_op_sel_e` | IALU 操作数选择（REG_IMM/REG_REG） |
| `ialu_cmd` | `type_scr1_ialu_cmd_sel_e` | IALU 运算命令 |
| `sum2_op` | `type_scr1_ialu_sum2_op_sel_e` | 第二加法器操作数（PC_IMM/REG_IMM） |
| `lsu_cmd` | `type_scr1_lsu_cmd_sel_e` | 访存命令（LB..SW） |
| `csr_op` | `type_scr1_csr_op_sel_e` | CSR 操作数（IMM/REG） |
| `csr_cmd` | `type_scr1_csr_cmd_sel_e` | CSR 命令（NONE/WRITE/SET/CLEAR） |
| `rd_wb_sel` | `type_scr1_rd_wb_sel_e` | 写回来源（IALU/SUM2/IMM/INC_PC/LSU/CSR） |
| `jump_req` | logic | 无条件跳转 |
| `branch_req` | logic | 条件分支 |
| `mret_req` | logic | MRET |
| `fencei_req` | logic | FENCE.I |
| `wfi_req` | logic | WFI |
| `rs1_addr` | 5 | rs1；CSRRxI 时复用为 zimm |
| `rs2_addr` | 5 | rs2 |
| `rd_addr` | 5 | rd |
| `imm` | XLEN | 立即数；CSR 时为 `{funct3,csr}`；非法时为整条 `instr` |
| `exc_req` | logic | 异常请求 |
| `exc_code` | `type_scr1_exc_code_e` | 异常码 |

### B.5 异常码（`scr1_arch_types.svh:41-51`）

| 名称 | 值 | 来源 |
|---|---|---|
| `INSTR_MISALIGN` | 0 | EXU |
| `INSTR_ACCESS_FAULT` | 1 | IFU（经 IDU 透传） |
| `ILLEGAL_INSTR` | 2 | IDU / CSR |
| `BREAKPOINT` | 3 | IDU / BRKM |
| `ECALL_M` | 11 | IDU |

---

## 附录 C：指令译码总表

### C.1 RVI（`:140-493`）

| opcode | 指令 | use_rs1/rs2/rd/imm | ialu_cmd | rd_wb_sel | 其他 |
|---|---|---|---|---|---|
| AUIPC | `:145` | -/-/✓/✓ | — | SUM2 | sum2=PC_IMM |
| LUI | `:158` | -/-/✓/✓ | — | IMM | |
| JAL | `:170` | -/-/✓/✓ | — | INC_PC | jump, sum2=PC_IMM |
| LOAD | `:184` | ✓/-/✓/✓ | — | LSU | lsu_cmd 按 funct3 |
| STORE | `:206` | ✓/✓/-/✓ | — | — | lsu_cmd 按 funct3 |
| OP | `:225` | ✓/✓/✓/- | 按 f7/f3 | IALU | REG_REG |
| OP_IMM | `:275` | ✓/-/✓/✓ | 按 funct3 | IALU | REG_IMM |
| MISC_MEM | `:322` | -/-/-/- | — | — | FENCE=NOP; FENCE.I→fencei_req |
| BRANCH | `:341` | ✓/✓/-/✓ | 按 funct3 | — | branch, sum2=PC_IMM |
| JALR | `:365` | ✓/-/✓/✓ | — | INC_PC | jump, sum2=REG_IMM |
| SYSTEM | `:386` | 见 4.4.3 | — | CSR | csr_cmd/csr_op |

### C.2 RVC0（`:498-543`）

| funct3 | 指令 | 关键字段 |
|---|---|---|
| 000 | C.ADDI4SPN | ADD/REG_IMM，rd=`{2'b01,instr[4:2]}` |
| 010 | C.LW | LSU/LW，rd=`{2'b01,instr[4:2]}` |
| 110 | C.SW | LSU/SW，rs2=`{2'b01,instr[4:2]}` |

### C.3 RVC1（`:546-719`）

| funct3 | 指令 | 关键字段 |
|---|---|---|
| 000 | C.ADDI/C.NOP | ADD/REG_IMM，rs1=rd=`instr[11:7]` |
| 001 | C.JAL | jump，INC_PC，rd=RA |
| 010 | C.LI | IMM |
| 011 | C.ADDI16SP / C.LUI | rd==SP ? ADD : IMM |
| 100 | 移位/逻辑 | 见 4.6 |
| 101 | C.J | jump |
| 110 | C.BEQZ | branch，SUB_EQ，rs2=x0 |
| 111 | C.BNEZ | branch，SUB_NE，rs2=x0 |

### C.4 RVC2（`:722-851`）

| funct3 | 指令 | 关键字段 |
|---|---|---|
| 000 | C.SLLI | SLL/REG_IMM |
| 010 | C.LWSP | LSU/LW，rs1=SP |
| 100 | C.MV / C.JR / C.EBREAK / C.JALR / C.ADD | 见 4.7 |
| 110 | C.SWSP | LSU/SW，rs1=SP |

---

## 附录 D：立即数（imm）重组规则

| 指令 | 表达式 | 行号 |
|---|---|---|
| AUIPC / LUI | `{instr[31:12], 12'b0}` | 152、164 |
| JAL | `{{12{instr[31]}}, instr[19:12], instr[20], instr[30:21], 1'b0}` | 178 |
| LOAD | `{{21{instr[31]}}, instr[30:20]}` | 192 |
| STORE | `{{21{instr[31]}}, instr[30:25], instr[11:7]}` | 213 |
| BRANCH | `{{20{instr[31]}}, instr[7], instr[30:25], instr[11:8], 1'b0}` | 347 |
| JALR | `{{21{instr[31]}}, instr[30:20]}` | 377 |
| OP_IMM（算术） | `{{21{instr[31]}}, instr[30:20]}` | 281 |
| OP_IMM（移位） | `` `SCR1_XLEN'(shamt) `` | 295、305、310 |
| SYSTEM | `` `SCR1_XLEN'({funct3, instr[31:20]}) `` | 391 |
| C.ADDI4SPN | `{22'd0, instr[10:7], instr[12:11], instr[5], instr[6], 2'b00}` | 516 |
| C.LW / C.SW | `{25'd0, instr[5], instr[12:10], instr[6], 2'b00}` | 528、537 |
| C.ADDI / C.LI / C.ANDI | `{{27{instr[12]}}, instr[6:2]}` | 561、578、643 |
| C.ADDI16SP | `{{23{instr[12]}}, instr[4:3], instr[5], instr[2], instr[6], 4'd0}` | 593 |
| C.LUI | `{{15{instr[12]}}, instr[6:2], 12'd0}` | 598 |
| C.SLLI / C.SRLI / C.SRAI | `{27'd0, instr[6:2]}` | 735、619、630 |
| C.JAL / C.J | `{{21{instr[12]}}, instr[8], instr[10:9], instr[6], instr[7], instr[2], instr[11], instr[5:3], 1'b0}` | 572、686 |
| C.BEQZ / C.BNEZ | `{{24{instr[12]}}, instr[6:5], instr[2], instr[11:10], instr[4:3], 1'b0}` | 701、716 |
| C.LWSP | `{24'd0, instr[3:2], instr[12], instr[6:4], 2'b00}` | 755 |
| C.SWSP | `{24'd0, instr[8:7], instr[12:9], 2'b00}` | 842 |
| C.JR / C.JALR | `0` | 786、808 |

---

## 附录 E：关键信号 → 消费者映射

| IDU 信号 | 生产者 | 消费者 | 作用 |
|---|---|---|---|
| `ifu2idu_instr_i` | IFU `ifu2idu_instr_o` | IDU `instr` | 待译码指令 |
| `ifu2idu_vd_i` | IFU `ifu2idu_vd_o` | IDU `idu2exu_req_o` | 指令有效 → EXU 请求 |
| `ifu2idu_imem_err_i` | IFU `ifu2idu_imem_err_o` | IDU 故障短路 | 取指异常 |
| `ifu2idu_err_rvi_hi_i` | IFU `ifu2idu_err_rvi_hi_o` | IDU `instr_rvc` | RVI 高半字故障 |
| `exu2idu_rdy_i` | EXU `exu2idu_rdy_o` | IDU `idu2ifu_rdy_o` | EXU 就绪 → IFU 可出队 |
| `idu2exu_cmd_o` | IDU 译码 | EXU `idu2exu_cmd_i` | 命令字 |
| `idu2exu_use_rs1_o/use_rs2_o/use_rd_o/use_imm_o` | IDU 译码 | EXU 门控 | 操作数使用提示 |
| `idu2exu_req_o` | IFU 有效 | EXU | 请求 |

顶层例化连接见 `src/core/pipeline/scr1_pipe_top.sv:340-359`（IDU）、`:329-334`（IFU 侧）、`:373-381`（EXU 侧）。

---

## 附录 F：与 IFU/EXU 的职责边界

| 职责 | IFU | IDU | EXU |
|---|---|---|---|
| 取指、指令流切割 | ✓ | — | — |
| 命令译码 | — | ✓ | — |
| 立即数重组 | 部分（跨块拼接） | ✓ | — |
| 非法指令判定 | — | ✓ | — |
| 取指异常 | 产生 | 透传并绑定 | 写 `mepc/mtval` |
| flow control（flush/新 PC） | 响应 | 不参与 | 产生 |
