# scr1_pipe_idu.sv 讲解

## 一句话职责

译码单元：纯组合逻辑，把 32 位指令拆字段，生成给 EXU 的完整"执行命令单"。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_idu.sv`
- **行数**: 940 行
- **配置依赖**: `SCR1_RVC_EXT`、`SCR1_RVM_EXT`、`SCR1_RVE_EXT`

## 核心设计原则

### 1. 纯组合逻辑

**IDU 没有状态、没有寄存器**，只有一大块 `always_comb` 译码。这意味着：
- 输入指令变化 → 输出命令立刻变化（无时钟延迟）
- 必须用"默认值先行"避免锁存器推断

### 2. 默认值先行（`:96-123`）

```systemverilog
always_comb begin
    // 开头全部赋安全默认值
    idu2exu_cmd_o = '0;
    idu2exu_cmd_o.rd_wb_sel = SCR1_RD_WB_SEL_INC_PC;  // 默认写回 PC+4
    // ...
    
    // 然后逐个 opcode 覆盖
    case (rvi_opcode)
        ...
    endcase
end
```

**为什么？** 如果不先赋默认值，`case` 语句未覆盖的分支会推断出锁存器，导致时序违例。

### 3. 字段切分（`:85-92`）

```systemverilog
instr_type   = instr[1:0];           // RVC 类型判定
rvi_opcode   = instr[6:2];           // RVI opcode
funct3       = (instr_type==RVC) ? instr[15:13] : instr[14:12];
funct7       = instr[31:25];
```

RVC 字段位置和 RVI 不同，`funct3` 用三目选择（`:89`）。

## 输出结构体：`idu2exu_cmd_o`

`type_scr1_exu_cmd_s` 是打包结构体（定义在 `scr1_riscv_isa_decoding.svh`），是"给 EXU 的完整指令单"：

| 字段 | 含义 | 例子 |
|---|---|---|
| `ialu_cmd` | ALU 操作 | ADD/SUB/MUL/XOR/SLL… |
| `ialu_op` | 运算数来源 | 寄存器 - 寄存器 或 寄存器 - 立即数 |
| `sum2_op` | 第二个加法器的源 | PC+ 立即数（跳转/取地址）或 寄存器 + 立即数（访存） |
| `lsu_cmd` | 访存操作 | LB/LH/LW/LBU/LHU/SB/SH/SW |
| `csr_cmd/csr_op` | CSR 读写操作 | CSRRW/CSRRS/CSRRC |
| `rd_wb_sel` | 结果写回来源 | IALU / LSU / CSR / IMM / INC_PC |
| `jump_req` / `branch_req` | 跳转/分支请求 | JAL / BEQ |
| `exc_req` / `exc_code` | 异常请求与异常码 | 非法指令 / ECALL / EBREAK |
| `rs1_addr/rs2_addr/rd_addr` | 三个寄存器号 | x1, x2, x31 |
| `imm` | 符号扩展后的立即数 | JAL 的 21 位立即数 |

## 立即数重组（重点）

RISC-V 立即数位散落在指令里，译码器重排。

### JAL 立即数（`:178`）

```systemverilog
imm = {{12{instr[31]}}, instr[19:12], instr[20], instr[30:21], 1'b0};
```

- `{12{instr[31]}}`：符号扩展（最高位复制到 31:21）
- `instr[19:12]`：imm[19:12]
- `instr[20]`：imm[20]
- `instr[30:21]`：imm[30:21]
- `1'b0`：最低位恒 0（JAL 地址必 2 对齐）

### 其他指令立即数

| 指令类型 | 立即数位来源 | 用途 |
|---|---|---|
| I 型（ADDI） | `instr[31:20]` | 算术/逻辑立即数 |
| S 型（SW） | `instr[31:25]` + `instr[11:7]` |  Store 地址偏移 |
| B 型（BEQ） | `instr[31]` + `instr[7]` + `instr[30:25]` + `instr[11:8]` + `1'b0` | 分支偏移 |
| U 型（LUI） | `instr[31:12]` | 高 20 位立即数 |
| J 型（JAL） | 见上 | 跳转目标 |

## 非法指令检查

每个分支检查编码合法性，非法则置 `rvi_illegal`。末尾（`:871-907`）把命令清零并改为：

```systemverilog
idu2exu_cmd_o.exc_req = 1'b1;
idu2exu_cmd_o.exc_code = SCR1_EXC_CODE_ILLEGAL_INSTR;
```

**触发条件**：
- opcode 未定义
- funct3/funct7 不匹配
- RVC 编码非法（如 `11` 开头）
- 寄存器号越界（RV32E 时 rs>=16）

## RVC 特殊处理

压缩指令的译码更复杂，因为：
1. 字段位置与 RVI 不同
2. 隐含寄存器（如 `C.ADDI4SPN` 隐含 rd=x2）
3. 立即数缩放（`C.ADDI` 立即数×1，`C.ADDI16SP` ×16）

示例（`:500-550`）：
```systemverilog
case (c_opcode)
    C_ADDI4SPN: begin
        idu2exu_cmd_o.rd_addr = {2'b01, c_rd};  // 隐含 rd=x2+(c_rd<<1)
        idu2exu_cmd_o.ialu_op = SCR1_IALU_OP_REG_IMM;
        idu2exu_cmd_o.imm = c_imm_scaled;
    end
    ...
endcase
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|
| `SCR1_RVC_EXT` | 增加 RVC 分支，字段切分用三目选择 |
| `SCR1_RVM_EXT` | 增加 MUL/DIV 译码分支 |
| `SCR1_RVE_EXT` | rs/rd 检查 `>=16` 时判非法 |
| `SCR1_NO_DEC_STAGE` | 无影响（IDU 本身无寄存器） |

## 与其他模块关系

```
IFU → IDU: ifu2idu_instr_o + ifu2idu_vd
IDU → EXU: idu2exu_cmd_o（组合输出，无时钟打拍）
IDU → MPRF: idu2exu_rs1_addr_o / rs2_addr_o（读寄存器号）
```

## 定位

IDU 只是一张**查表**：opcode+funct3+funct7 → 控制信号。译码器负责"翻译"，执行单元负责"干活"。

## 配套文档

- 设计规格：`docs/scr1-pipe-idu/design-spec.md`
- 验证文档：`docs/scr1-pipe-idu/verification.md`
