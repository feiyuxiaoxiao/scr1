# SCR1 IDU 指令译码单元 技术分析

> 对应源码: `src/core/pipeline/scr1_pipe_idu.sv`
> 模块名: `scr1_pipe_idu`
> 下游模块: EXU（执行单元）

---

## 1. 模块定位

IDU 是纯组合逻辑模块（无时钟寄存器，无 FSM），接收 IFU 传来的 32 位指令原始编码，产出 `type_scr1_exu_cmd_s` 结构体直接驱动 EXU。它承担 RISC-V 处理器的全部指令译码职责——指令分类、操作数提取、立即数拼接、控制信号生成、非法指令检测。

```
IFU ──instr──→ IDU ──exu_cmd──→ EXU
                 │
                 │ 纯组合逻辑，单周期完成
                 │ 无流水线寄存器，无状态记忆
```

---

## 2. 指令类型判定

### 2.1 两级分类体系

第一级：`instr[1:0]` 区分 RVI（32 位）与 RVC（16 位，4 个象限）：

```systemverilog
assign instr_type = type_scr1_instr_type_e'(instr[1:0]);
```

| `instr[1:0]` | enum 值 | 含义 |
|:---:|------|------|
| `00` | `SCR1_INSTR_RVC0` | RVC 象限 0（寄存器操作数在 abi' 区域 `[7:2]`） |
| `01` | `SCR1_INSTR_RVC1` | RVC 象限 1（目标寄存器在 `[11:7]`） |
| `10` | `SCR1_INSTR_RVC2` | RVC 象限 2（栈指针和通用操作） |
| `11` | `SCR1_INSTR_RVI` | 标准 32 位 RV32I 指令 |

第二级（仅 RVI）：`instr[6:2]` 编码为 RV32I 操作码 `type_scr1_rvi_opcode_e`：

```systemverilog
assign rvi_opcode = type_scr1_rvi_opcode_e'(instr[6:2]);
```

### 2.2 字段提取（RVI 与 RVC 共用信号）

RVI 和 RVC 对 funct3/funct7 的定义不同，但共享变量名：

```systemverilog
// funct3: RVI 用 [14:12]，RVC 用 [15:13]
assign funct3 = (instr_type == SCR1_INSTR_RVI) ? instr[14:12] : instr[15:13];
// funct7: 仅 RVI 有
assign funct7  = instr[31:25];
// funct12: RVI SYSTEM 指令用
assign funct12 = instr[31:20];
// shamt: RVI 移位量
assign shamt  = instr[24:20];
```

RVC 的 funct3 从 bit[15:13] 提取是因为 16 位压缩指令布局不同。

---

## 3. EXU 命令结构体 `type_scr1_exu_cmd_s`

这是一条指令译码的完整输出，打包进一个 packed struct：

| 字段 | 位宽 | 含义 |
|------|:---:|------|
| `instr_rvc` | 1 | 是否为压缩指令 |
| `ialu_op` | 1 | 整数 ALU 操作数来源：`REG_IMM` 或 `REG_REG` |
| `ialu_cmd` | 5 | 整数 ALU 运算命令（15~23 种） |
| `sum2_op` | 1 | 加法器 2 操作数来源：`PC_IMM` 或 `REG_IMM` |
| `lsu_cmd` | 4 | 加载/存储命令（LB/LH/LW/LBU/LHU/SB/SH/SW） |
| `csr_op` | 1 | CSR 操作数来源：`IMM` 或 `REG`（CSRRSI 用 zimm） |
| `csr_cmd` | 2 | CSR 操作命令：`WRITE`/`SET`/`CLEAR` |
| `rd_wb_sel` | 3 | 回写到 rd 的数据来源（IALU/SUM2/IMM/INC_PC/LSU/CSR） |
| `jump_req` | 1 | 无条件跳转请求 |
| `branch_req` | 1 | 条件分支请求 |
| `mret_req` | 1 | MRET 异常返回请求 |
| `fencei_req` | 1 | FENCE.I 请求 |
| `wfi_req` | 1 | WFI 等待中断请求 |
| `rs1_addr` | 5 | rs1 寄存器地址（CSRRxI 时复用为 zimm） |
| `rs2_addr` | 5 | rs2 寄存器地址 |
| `rd_addr` | 5 | rd 目标寄存器地址 |
| `imm` | 32 | 立即数（CSR 指令时复用为 `{funct3, csr_addr}`，非法指令时复用为整条指令） |
| `exc_req` | 1 | 异常请求 |
| `exc_code` | 4 | 异常类型编码 |

---

## 4. 核心译码流程

译码是一个大型 `always_comb` 块（`scr1_pipe_idu.sv:95-909`），结构如下：

```
always_comb begin
    // Step 1: 所有输出信号设为默认值（全 0 / NONE）
    ...
    // Step 2: IMEM 错误检查（优先级最高）
    if (ifu2idu_imem_err_i) → 设置 exc_req + exc_code
    else begin
        // Step 3: 指令类型分发
        case (instr_type)
            SCR1_INSTR_RVI  → rvi_opcode 二级 case
            SCR1_INSTR_RVC0 → funct3 二级 case（象限 0 指令）
            SCR1_INSTR_RVC1 → funct3 二级 case（象限 1 指令）
            SCR1_INSTR_RVC2 → funct3 二级 case（象限 2 指令）
            default         → rvi_illegal / rvc_illegal
        endcase
    end
    // Step 4: 非法指令检测（优先级最低，覆盖所有输出）
    if (rvi_illegal | rvc_illegal | rve_illegal)
        → 清空所有控制信号 + 设置 exc_req = ILLEGAL_INSTR
end
```

---

## 5. RV32I 指令译码

### 5.1 译码表

| 操作码 | funct3 | funct7 | 译出的关键信号 | RISC-V 指令 |
|--------|:---:|:---:|------|------|
| `AUIPC` | — | — | `sum2_op=PC_IMM`, `rd_wb_sel=SUM2`, `imm=U-type` | AUIPC |
| `LUI` | — | — | `rd_wb_sel=IMM`, `imm=U-type` | LUI |
| `JAL` | — | — | `sum2_op=PC_IMM`, `rd_wb_sel=INC_PC`, `jump_req=1`, `imm=J-type` | JAL |
| `LOAD` | 000~101 | — | `sum2_op=REG_IMM`, `rd_wb_sel=LSU`, `lsu_cmd=LB/LH/LW/LBU/LHU`, `imm=I-type` | LB/LH/LW/LBU/LHU |
| `STORE` | 000~010 | — | `sum2_op=REG_IMM`, `lsu_cmd=SB/SH/SW`, `imm=S-type` | SB/SH/SW |
| `OP` | 000~111 | 0000000 | `ialu_op=REG_REG`, `rd_wb_sel=IALU`, `ialu_cmd=ADD/SLL/SLT/SLTU/XOR/SRL/OR/AND` | 基础算术逻辑 |
| `OP` | 000/101 | 0100000 | `ialu_cmd=SUB/SRA` | SUB/SRA |
| `OP` | 000~111 | 0000001 | `ialu_cmd=MUL/MULH/MULHSU/MULHU/DIV/DIVU/REM/REMU`（M 扩展） | MUL/DIV 等 |
| `OP-IMM` | 000~111 | 多种 | `ialu_op=REG_IMM`, `rd_wb_sel=IALU`, `imm=I-type` | ADDI/SLTI/XORI/ORI/ANDI |
| `OP-IMM` | 001 | 0000000 | `ialu_cmd=SLL`, `imm=shamt`（零扩展） | SLLI |
| `OP-IMM` | 101 | 0000000/0100000 | `ialu_cmd=SRL/SRA`, `imm=shamt`（零扩展） | SRLI/SRAI |
| `BRANCH` | 000~111 | — | `sum2_op=PC_IMM`, `ialu_op=REG_REG`, `branch_req=1`, `ialu_cmd=SUB_EQ/NE/LT/GE/LTU/GEU`, `imm=B-type` | BEQ/BNE/BLT/BGE/BLTU/BGEU |
| `JALR` | 000 | — | `sum2_op=REG_IMM`, `rd_wb_sel=INC_PC`, `jump_req=1`, `imm=I-type` | JALR |
| `MISC-MEM` | 000/001 | — | `fencei_req=1`（FENCE.I）或 NOP（FENCE） | FENCE/FENCE.I |
| `SYSTEM` | 000 | funct12 | `exc_req`/`mret_req`/`wfi_req` | ECALL/EBREAK/MRET/WFI |
| `SYSTEM` | 001~111 | — | `rd_wb_sel=CSR`, `csr_cmd=WRITE/SET/CLEAR`, `csr_op=REG/IMM` | CSRRW/CSRRS/CSRRC 及其立即数形式 |

### 5.2 立即数拼接

每种指令格式的立即数分散在指令的不同位域，IDU 负责拼接为 32 位有符号数：

| 格式 | 拼接表达式 | 示例指令 |
|------|------|------|
| I-type | `{{21{instr[31]}}, instr[30:20]}` | ADDI, LOAD, JALR |
| S-type | `{{21{instr[31]}}, instr[30:25], instr[11:7]}` | STORE |
| B-type | `{{20{instr[31]}}, instr[7], instr[30:25], instr[11:8], 1'b0}` | BRANCH |
| U-type | `{instr[31:12], 12'b0}` | LUI, AUIPC |
| J-type | `{{12{instr[31]}}, instr[19:12], instr[20], instr[30:21], 1'b0}` | JAL |
| shamt | `{27'b0, instr[24:20]}`（零扩展） | SLLI, SRLI, SRAI |
| CSR | `{funct3, instr[31:20]}` | CSR 指令（立即数域复用为 CSR 地址） |

### 5.3 特殊指令处理

**FENCE**（`funct3=000`）：当 `instr[31:28]`, `instr[19:15]`, `instr[11:7]` 全为零时视为 NOP（SCR1 是单核顺序处理器，FENCE 无需实际动作）。

**FENCE.I**（`funct3=001`）：当 `instr[31:15]`, `instr[11:7]` 全为零时发出 `fencei_req=1`，触发指令缓存刷新。

**ECALL/EBREAK/MRET/WFI**：走 `SYSTEM` 操作码 `funct3=000`，且要求 `{instr[19:15], instr[11:7]} = 10'd0`（即为 PRIV 指令），由 `funct12` 区分。

**CSR 指令的立即数形式**：`.csr_op=IMM` 表示源操作数是 zimm（`instr[19:15]` 零扩展），而非 rs1 寄存器值。此时 `use_rs1=1` 被复用为"需要读取 zimm"标志，实际数据在 `rs1_addr` 域传输。

---

## 6. RVC 压缩指令译码

### 6.1 象限 0（`SCR1_INSTR_RVC0`，`instr[1:0]=00`）

寄存器操作数使用压缩地址空间 `{2'b01, instr[x:y]}`（映射到 x8-x15）：

| funct3 | 指令 | 关键信号 |
|:---:|------|------|
| `000` | C.ADDI4SPN | `ialu_op=REG_IMM`, `ialu_cmd=ADD`, `rs1=SP`, `imm=拼接`。`instr[12:5]=0` 则为非法 |
| `010` | C.LW | `sum2_op=REG_IMM`, `lsu_cmd=LW`, `rd_wb_sel=LSU` |
| `110` | C.SW | `sum2_op=REG_IMM`, `lsu_cmd=SW` |

### 6.2 象限 1（`SCR1_INSTR_RVC1`，`instr[1:0]=01`）

| funct3 | 指令 | 关键信号 |
|:---:|------|------|
| `000` | C.ADDI / C.NOP | `ialu_op=REG_IMM`, `ialu_cmd=ADD`, `rs1=rd` |
| `001` | C.JAL | `sum2_op=PC_IMM`, `rd_wb_sel=INC_PC`, `jump_req=1`, `rd=RA` |
| `010` | C.LI | `rd_wb_sel=IMM`, `imm=符号扩展` |
| `011` | C.LUI / C.ADDI16SP | 若 `rd==SP` 则为 C.ADDI16SP（`ialu_op=REG_IMM`, `ialu_cmd=ADD`, `rs1=SP`），否则 C.LUI（`rd_wb_sel=IMM`） |
| `100` | C.MISC-ALU | 嵌套 `instr[11:10]` 二级译码：C.SRLI/C.SRAI/C.ANDI/或 `instr[12:6:5]` 选 C.SUB/C.XOR/C.OR/C.AND |
| `101` | C.J | `sum2_op=PC_IMM`, `jump_req=1` |
| `110` | C.BEQZ | `sum2_op=PC_IMM`, `ialu_cmd=SUB_EQ`, `branch_req=1`, `rs2=x0` |
| `111` | C.BNEZ | `sum2_op=PC_IMM`, `ialu_cmd=SUB_NE`, `branch_req=1`, `rs2=x0` |

**C.BEQZ/C.BNEZ 的复用技巧**：这两条指令仅有一个寄存器源操作数（rs1'），但 IDU 始终设置 `use_rs2=1`, `rs2_addr=0`（x0）。这是因为 EXU 中的整型 ALU 需要两个操作数做减法比较，`rs1 - 0 == 0` 等价于 `rs1 == 0`。

### 6.3 象限 2（`SCR1_INSTR_RVC2`，`instr[1:0]=10`）

| funct3 | `instr[12]` | 子条件 | 指令 | 关键信号 |
|:---:|:---:|------|------|------|
| `000` | 0 | — | C.SLLI | `ialu_op=REG_IMM`, `ialu_cmd=SLL` |
| `010` | — | — | C.LWSP | `sum2_op=REG_IMM`, `lsu_cmd=LW`, `rs1=SP`，`rd≠0` |
| `100` | 0 | `instr[6:2]≠0` | C.MV | `ialu_op=REG_REG`, `ialu_cmd=ADD`, `rs1=x0` |
| `100` | 0 | `instr[6:2]=0, rd≠0` | C.JR | `sum2_op=REG_IMM`, `jump_req=1`, `imm=0` |
| `100` | 1 | `instr[11:2]=0` | C.EBREAK | `exc_req=1`, `exc_code=BREAKPOINT` |
| `100` | 1 | `instr[6:2]=0, instr[11:7]≠0` | C.JALR | `sum2_op=REG_IMM`, `jump_req=1`, `rd=RA`, `imm=0` |
| `100` | 1 | 其他 | C.ADD | `ialu_op=REG_REG`, `ialu_cmd=ADD` |
| `110` | — | — | C.SWSP | `sum2_op=REG_IMM`, `lsu_cmd=SW`, `rs1=SP` |

**C.MV 的实现技巧**：不是专门的 MOVE 命令，而是 `ADD rd, x0, rs2`，即 `rs2 + 0 → rd`。译码为 `ialu_op=REG_REG`, `ialu_cmd=ADD`, `rs1_addr=x0`。

---

## 7. 非法指令检测

三种非法标志并行检测，最后汇总：

```systemverilog
if (rvi_illegal | rvc_illegal | rve_illegal) begin
    // 所有控制信号清空
    idu2exu_cmd_o.exc_req  = 1'b1;
    idu2exu_cmd_o.exc_code = SCR1_EXC_CODE_ILLEGAL_INSTR;
end
```

### 7.1 RVI 非法条件

`rvi_illegal` 在以下情况下置位：
- `rvi_opcode` 的 `default` 分支（未定义操作码）
- 合法操作码下 `funct3` 的 `default` 分支（如 LOAD `funct3=011`）
- 合法 opcode+funct3 下 `funct7` 不匹配（如 OP-IMM `funct3=001` 但 `funct7≠0000000`）
- SYSTEM 指令中 `{rs1,rd}≠10'd0` 且非 CSR 指令
- FENCE/FENCE.I 中非零字段存在

### 7.2 RVC 非法条件

- `rvc_illegal`：各象限中 `funct3` 的 `default` 分支，或特定保留编码（如 C.ADDI4SPN 中 `instr[12:5]=0`）
- 若未定义 `SCR1_RVC_EXT`，所有 `instr[1:0]≠11` 的指令直接置 `rvi_illegal`

### 7.3 RVE 非法条件（嵌入式扩展限制）

RVE 限制可用寄存器为 x0-x15，因此检查 `instr[11]`, `instr[19]`, `instr[24]` 等高寄存器地址位是否为 1：

```systemverilog
// 例如 OP 指令
if (instr[11] | instr[19] | instr[24]) rve_illegal = 1'b1;
```

### 7.4 `SCR1_MTVAL_ILLEGAL_INSTR_EN`

若此宏定义，非法指令时将整条指令原值写入 `imm` 域，供异常处理程序通过 `mtval` CSR 读取：

```systemverilog
idu2exu_use_imm_o = 1'b1;
idu2exu_cmd_o.imm = instr;
```

---

## 8. 时钟门控信号

IDU 向 EXU 输出 4 个"使用标志"，告知时钟门控单元哪些寄存器文件读端口需要打开：

| 信号 | 置位条件 | 用途 |
|------|------|------|
| `idu2exu_use_rs1_o` | 指令有 rs1 源操作数 | 打开寄存器文件读端口 1 |
| `idu2exu_use_rs2_o` | 指令有 rs2 源操作数 | 打开寄存器文件读端口 2 |
| `idu2exu_use_rd_o` | 指令有 rd 目标寄存器 | 标记需要回写（无译码阶段时不存在） |
| `idu2exu_use_imm_o` | 指令使用立即数 | 标记立即数有效（无译码阶段时不存在） |

注意：CSR 立即数形式（CSRRWI/CSRRSI/CSRRCI）将 `use_rs1=1` 复用为"需要 zimm"标志，但 zimm 实际编码在 `rs1_addr` 域中传输。

---

## 9. IFU-EXU 透传信号

IDU 本身不产生有效/就绪逻辑，而是直接透传：

```systemverilog
assign idu2ifu_rdy_o = exu2idu_rdy_i;   // IDU 的就绪 = EXU 的就绪（直通）
assign idu2exu_req_o = ifu2idu_vd_i;    // 指令有效 = IFU 数据有效（直通）
```

IDU 是纯组合逻辑，没有内部流水线状态，所以反压和有效信号直接穿通，IDU 本身不引入额外延迟。

---

## 10. IMEM 错误处理（优先级最高）

译码 `always_comb` 块的最外层是一个 IMEM 错误检查：

```systemverilog
if (ifu2idu_imem_err_i) begin
    idu2exu_cmd_o.exc_req  = 1'b1;
    idu2exu_cmd_o.exc_code = SCR1_EXC_CODE_INSTR_ACCESS_FAULT;
    idu2exu_cmd_o.instr_rvc = ifu2idu_err_rvi_hi_i;
end else begin
    // 正常译码流程 ...
end
```

`instr_rvc` 字段在此处被复用：当 `ifu2idu_err_rvi_hi_i=1` 时（RVI 高半取指错误），`instr_rvc` 置 1，告知 EXU 这原本应该是一条 RVI 指令的前半部分或 RVC 指令。这帮助 EXU 正确计算异常返回地址（mepc）。

IMEM 错误直接终止译码——不走 `case(instr_type)` 分支，因为指令数据已损坏、译码结果无意义。

---

## 11. 三条算术路径的设计思想

IDU 译码出的控制信号驱动 EXU 中的三条并行算术通路：

| 通路 | 操作数 | 控制信号 | 产生的值 |
|------|--------|------|------|
| **IALU 主通路** | `ialu_op` + `ialu_cmd` | rs1, rs2/imm | 算术逻辑结果 → 可回写到 rd |
| **SUM2 加法器** | `sum2_op` | PC/rs1, imm | 跳转目标地址 / 访存地址 |
| **LSU** | `lsu_cmd` | SUM2 结果作地址 | DMEM 读写 |

三条路径并行计算，`rd_wb_sel` 决定最终哪个结果写回 rd：

- `SCR1_RD_WB_IALU` — 普通 ALU 指令
- `SCR1_RD_WB_SUM2` — AUIPC
- `SCR1_RD_WB_IMM` — LUI
- `SCR1_RD_WB_INC_PC` — JAL/JALR（PC+4）
- `SCR1_RD_WB_LSU` — LOAD 指令
- `SCR1_RD_WB_CSR` — CSR 读

---

## 12. 验证断言

两条 SVA 断言（仅 `SCR1_TRGT_SIMULATION`）：

1. **X 态检测**：`ifu2idu_vd_i` 和 `exu2idu_rdy_i` 不能有未知值
2. **IALU 命令范围**：译码出的 `ialu_cmd` 必须在有效枚举范围内（`NONE` ~ `SRA`，M 扩展时到 `REMU`），防止未定义命令进入 EXU ALU
