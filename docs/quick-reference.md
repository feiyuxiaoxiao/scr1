# SCR1 快速参考手册

本文档提供 SCR1 RISC-V 核心的快速查询表，包括 ISA 指令编码、配置矩阵、寄存器映射、异常码等。

---

## 附录 A：RISC-V ISA 指令编码速查表

### A.1 R 型指令（寄存器 - 寄存器）

```
31     25 24   20 19   15 14   12 11     7 6      0
+--------+-------+-------+-------+--------+--------+
| funct7 |  rs2  |  rs1  |funct3 |  rd    | opcode |
+--------+-------+-------+-------+--------+--------+
```

| 指令 | opcode | funct3 | funct7 | 操作 |
|---|---|---|---|---|
| ADD | 0110011 | 000 | 0000000 | rd = rs1 + rs2 |
| SUB | 0110011 | 000 | 0100000 | rd = rs1 - rs2 |
| SLL | 0110011 | 001 | 0000000 | rd = rs1 << rs2[4:0] |
| SLT | 0110011 | 010 | 0000000 | rd = (rs1 < rs2) ? 1 : 0 (有符号) |
| SLTU | 0110011 | 011 | 0000000 | rd = (rs1 < rs2) ? 1 : 0 (无符号) |
| XOR | 0110011 | 100 | 0000000 | rd = rs1 ^ rs2 |
| SRL | 0110011 | 101 | 0000000 | rd = rs1 >> rs2[4:0] (逻辑) |
| SRA | 0110011 | 101 | 0100000 | rd = rs1 >>> rs2[4:0] (算术) |
| OR | 0110011 | 110 | 0000000 | rd = rs1 \| rs2 |
| AND | 0110011 | 111 | 0000000 | rd = rs1 & rs2 |
| MUL | 0110011 | 000 | 0000001 | rd = rs1 * rs2 (低 32 位) |
| MULH | 0110011 | 001 | 0000001 | rd = (rs1 * rs2)[63:32] (有符号) |
| MULHU | 0110011 | 011 | 0000001 | rd = (rs1 * rs2)[63:32] (无符号) |
| DIV | 0110011 | 100 | 0000001 | rd = rs1 / rs2 (有符号) |
| DIVU | 0110011 | 101 | 0000001 | rd = rs1 / rs2 (无符号) |
| REM | 0110011 | 110 | 0000001 | rd = rs1 % rs2 (有符号) |
| REMU | 0110011 | 111 | 0000001 | rd = rs1 % rs2 (无符号) |

### A.2 I 型指令（立即数）

```
31     20 19   15 14   12 11     7 6      0
+--------+-------+-------+--------+--------+
|  imm[11:0] |  rs1  |funct3 |  rd    | opcode |
+--------+-------+-------+--------+--------+
```

| 指令 | opcode | funct3 | 操作 |
|---|---|---|---|
| ADDI | 0010011 | 000 | rd = rs1 + imm |
| SLTI | 0010011 | 010 | rd = (rs1 < imm) ? 1 : 0 |
| SLTIU | 0010011 | 011 | rd = (rs1 < imm_u) ? 1 : 0 |
| XORI | 0010011 | 100 | rd = rs1 ^ imm |
| ORI | 0010011 | 110 | rd = rs1 \| imm |
| ANDI | 0010011 | 111 | rd = rs1 & imm |
| SLLI | 0010011 | 001 | rd = rs1 << imm[4:0] |
| SRLI | 0010011 | 101 | rd = rs1 >> imm[4:0] (逻辑，funct7=0000000) |
| SRAI | 0010011 | 101 | rd = rs1 >>> imm[4:0] (算术，funct7=0100000) |
| LB | 0000011 | 000 | rd = sign_ext(mem[rs1+imm][7:0]) |
| LH | 0000011 | 001 | rd = sign_ext(mem[rs1+imm][15:0]) |
| LW | 0000011 | 010 | rd = mem[rs1+imm][31:0] |
| LBU | 0000011 | 100 | rd = zero_ext(mem[rs1+imm][7:0]) |
| LHU | 0000011 | 101 | rd = zero_ext(mem[rs1+imm][15:0]) |
| JALR | 1100111 | 000 | rd = pc+4; pc = (rs1+imm) & ~1 |

### A.3 S 型指令（Store）

```
31     25 24   20 19   15 14   12 11     7 6      0
+--------+-------+-------+-------+--------+--------+
|imm[11:5]|  rs2  |  rs1  |funct3 |imm[4:0] | opcode |
+--------+-------+-------+-------+--------+--------+
```

| 指令 | opcode | funct3 | 操作 |
|---|---|---|---|
| SB | 0100011 | 000 | mem[rs1+imm][7:0] = rs2[7:0] |
| SH | 0100011 | 001 | mem[rs1+imm][15:0] = rs2[15:0] |
| SW | 0100011 | 010 | mem[rs1+imm][31:0] = rs2[31:0] |

### A.4 B 型指令（Branch）

```
31     25 24   20 19   15 14   12 11     7 6      0
+--------+-------+-------+-------+--------+--------+
|imm[12\|10:5]| rs2 | rs1 |funct3 |imm[4:1\|11] | opcode |
+--------+-------+-------+-------+--------+--------+
```

| 指令 | opcode | funct3 | 条件 |
|---|---|---|---|
| BEQ | 1100011 | 000 | pc += (rs1 == rs2) ? imm : 0 |
| BNE | 1100011 | 001 | pc += (rs1 != rs2) ? imm : 0 |
| BLT | 1100011 | 100 | pc += (rs1 < rs2) ? imm : 0 (有符号) |
| BGE | 1100011 | 101 | pc += (rs1 >= rs2) ? imm : 0 (有符号) |
| BLTU | 1100011 | 110 | pc += (rs1 < rs2) ? imm : 0 (无符号) |
| BGEU | 1100011 | 111 | pc += (rs1 >= rs2) ? imm : 0 (无符号) |

### A.5 U 型指令（上层立即数）

```
31     12 11     7 6      0
+--------+--------+--------+
|  imm[31:12] |  rd    | opcode |
+--------+--------+--------+
```

| 指令 | opcode | 操作 |
|---|---|---|
| LUI | 0110111 | rd = imm[31:12] << 12 |
| AUIPC | 0010111 | rd = pc + (imm[31:12] << 12) |

### A.6 J 型指令（Jump）

```
31     12 11     7 6      0
+--------+--------+--------+
|imm[20\|10:1\|11\|19:12]| rd | opcode |
+--------+--------+--------+
```

| 指令 | opcode | 操作 |
|---|---|---|
| JAL | 1101111 | rd = pc+4; pc += imm (符号扩展) |

### A.7 RVC 压缩指令（16 位）

```
15   13 12  11 10   7 6   2 1    0
+-----+---+-------+-----+------+
|funct3| |  rd/rs1 | ... | opcode |
+-----+---+-------+-----+------+
```

| 指令 | opcode[1:0] | funct3[15:13] | 操作 |
|---|---|---|---|
| C.ADDI4SPN | 00 | 000 | x2 += imm (仅 SP 相对) |
| C.LW | 00 | 010 | rd = mem[rs1'+imm] |
| C.SW | 00 | 110 | mem[rs1'+imm] = rs2' |
| C.ADDI | 01 | xxx | rd += imm |
| C.J | 01 | 101 | pc += imm |
| C.BEQZ | 11 | 011 | if (rs==0) pc+=imm |
| C.BNEZ | 11 | 111 | if (rs!=0) pc+=imm |

---

## 附录 B：SCR1 配置矩阵

| 配置名 | ISA | 流水线 | MPRF | 特性 |
|---|---|---|---|---|
| `SCR1_CFG_RV32IMC_MAX` | RV32IMC | 4 级 | 32 regs | IPIC, TDU, DBG, TCM, TRACE |
| `SCR1_CFG_RV32IC_BASE` | RV32IC | 3 级 | 32 regs | 无 M 扩展，无 DBG |
| `SCR1_CFG_RV32EC_MIN` | RV32EC | 2 级 | 16 regs | RVE, 无 M 扩展，无 DBG |
| `SCR1_CFG_CUSTOM` | 可定制 | 2~4 级 | 16/32 | 用户定义 |

### 配置宏开关

| 宏 | 功能 | 依赖 |
|---|---|---|
| `SCR1_RVE_EXT` | 16 通用寄存器 | - |
| `SCR1_RVM_EXT` | M 扩展（乘除） | - |
| `SCR1_RVC_EXT` | C 扩展（压缩指令） | - |
| `SCR1_IPIC_EN` | 中断控制器 | - |
| `SCR1_DBG_EN` | 调试子系统 | - |
| `SCR1_TDU_EN` | 硬件触发器 | DBG_EN |
| `SCR1_TRACE_LOG_EN` | 轨迹日志 | - |
| `SCR1_TCM_EN` | 紧耦合内存 | - |
| `SCR1_CLKCTRL_EN` | 时钟门控 | - |
| `SCR1_MPRF_RAM` | MPRF 用 RAM 实现 | - |

---

## 附录 C：CSR 寄存器映射

### C.1 机器模式只读寄存器

| 地址 | 寄存器 | 位宽 | 描述 |
|---|---|---|---|
| 0xF11 | MVENDORID | 32 | 厂商 ID（SCR1=0） |
| 0xF12 | MARCHID | 32 | 架构 ID（SCR1=0） |
| 0xF13 | MIMPID | 32 | 实现版本 |
| 0xF14 | MHARTID | 32 | Hart 号（0） |

### C.2 机器模式 Trap 设置寄存器

| 地址 | 寄存器 | 位宽 | 可写位 | 描述 |
|---|---|---|---|---|
| 0x300 | MSTATUS | 32 | MIE, MPIE, MPP | 全局中断使能 |
| 0x301 | MISA | 32 | MXL, Extensions | ISA 扩展使能 |
| 0x304 | MIE | 32 | MEIE, MSIE, MTIE | 中断使能 |
| 0x305 | MTVEC | 32 | BASE[31:2], MODE | trap 入口地址 |

### C.3 机器模式 Trap 处理寄存器

| 地址 | 寄存器 | 位宽 | 可写位 | 描述 |
|---|---|---|---|---|
| 0x340 | MSCRATCH | 32 | 全部 | 临时寄存器 |
| 0x341 | MEPC | 32 | 全部 | trap 前 PC |
| 0x342 | MCAUSE | 32 | Exception Code, Interrupt | trap 原因 |
| 0x343 | MTVAL | 32 | 全部 | 附加信息 |
| 0x344 | MIP | 32 | MEIP, MSIP, MTIP | 中断 pending |

### C.4 机器模式计数器

| 地址 | 寄存器 | 位宽 | 可写位 | 描述 |
|---|---|---|---|---|
| 0xB00 | MCYCLE | 32 | 全部 | 周期计数器（低 32） |
| 0xB80 | MCYCLEH | 32 | 全部 | 周期计数器（高 32） |
| 0xB02 | MINSTRET | 32 | 全部 | 退休指令数（低 32） |
| 0xB82 | MINSTRETH | 32 | 全部 | 退休指令数（高 32） |

### C.5 调试 CSR（`SCR1_DBG_EN`）

| 地址 | 寄存器 | 位宽 | 描述 |
|---|---|---|---|
| 0x7B0 | DCSR | 32 | 调试控制 |
| 0x7B1 | DPC | 32 | 调试 PC |
| 0x7B2 | DSCRATCH0 | 32 | 调试临时寄存器 0 |
| 0x7B3 | DSCRATCH1 | 32 | 调试临时寄存器 1 |

---

## 附录 D：异常/中断码

### D.1 异常码（MCAUSE[30:0]）

| 码值 | 类型 | 描述 |
|---|---|---|
| 0 | 异常 | 指令地址错位 (INST_ADDR_MISALIGN) |
| 1 | 异常 | 指令访问错误 (INST_ACCESS_FAULT) |
| 2 | 异常 | 非法指令 (ILLEGAL_INSTR) |
| 3 | 异常 | 断点 (BREAKPOINT) |
| 4 | 异常 | Load 地址错位 (LD_ADDR_MISALIGN) |
| 5 | 异常 | Load 访问错误 (LD_ACCESS_FAULT) |
| 6 | 异常 | Store 地址错位 (ST_ADDR_MISALIGN) |
| 7 | 异常 | Store 访问错误 (ST_ACCESS_FAULT) |
| 8 | 异常 | 环境调用 (ECALL/U-mode) |
| 9 | 异常 | 环境调用 (ECALL/S-mode，SCR1 不支持) |
| 11 | 异常 | 环境调用 (ECALL/M-mode) |
| 12 | 异常 | 指令页面错误 (INST_PAGE_FAULT，SCR1 不支持) |
| 13 | 异常 | Load 页面错误 (SCR1 不支持) |
| 15 | 异常 | Store 页面错误 (SCR1 不支持) |

### D.2 中断码（MCAUSE[31]=1）

| 码值 | 类型 | 描述 |
|---|---|---|
| 3 | 中断 | 软件中断 (MSI) |
| 7 | 中断 | 外部中断 (MEI) |
| 11 | 中断 | 定时器中断 (MTI) |

---

## 附录 E：调试寄存器映射（DM）

| 偏移 | 寄存器 | 位宽 | 描述 |
|---|---|---|---|
| 0x10 | DMCONTROL | 32 | 调试控制（haltreq, dresumereq, dmactive） |
| 0x11 | DMSTATUS | 32 | 调试状态（halted, running, version） |
| 0x12 | HARTINFO | 32 | Hart 信息（nscratch, dataaddr） |
| 0x16 | ABSTRACTCS | 32 | 抽象命令控制 |
| 0x17 | COMMAND | 32 | 抽象命令 |
| 0x04-0x07 | DATA0-3 | 32 | 数据寄存器 |
| 0x20-0x2F | PROGBUF0-15 | 32 | 程序缓冲区 |
| 0x40 | DSCRATCH0 | 32 | 调试临时寄存器 |
| 0x41 | DSCRATCH1 | 32 | 调试临时寄存器 1 |

---

## 附录 F：内存映射

| 起始地址 | 结束地址 | 大小 | 模块 | 描述 |
|---|---|---|---|---|
| 0x00000000 | 0x003FFFFF | 4MB | RAM | 代码加载区（TCM-less） |
| 0x00480000 | 0x00487FFF | 32KB | TCM | 紧耦合内存（`SCR1_TCM_EN`） |
| 0x00490000 | 0x00490FFF | 4KB | Timer | 内存映射定时器 |
| 0x004A0000 | 0xFFFFFFFF | ~ | AHB/AXI | 外部总线 |

### Timer 寄存器布局

| 偏移 | 寄存器 | 访问 | 描述 |
|---|---|---|---|
| 0x0 | CONTROL | RW | 分频使能 |
| 0x4 | DIVIDER | RW | 分频系数 |
| 0x8 | MTIMELO | RW | 计数器低 32 位 |
| 0xC | MTIMEHI | RW | 计数器高 32 位 |
| 0x10 | MTIMECMPLO | RW | 比较值低 32 位 |
| 0x14 | MTIMECMPHI | RW | 比较值高 32 位 |

---

## 附录 G：JTAG IR 编码

| IR[3:0] | 名称 | 描述 |
|---|---|---|
| 0000 | 保留 | - |
| 0001 | IDCODE | 设备 ID 寄存器 |
| 0010 | DMI_ACCESS | 调试管理接口 |
| 0011 | SCU_ACCESS | 系统控制接口 |
| 1110 | BYPASS | 直通寄存器 |
| 1111 | 保留 | - |

### IDCODE 格式（32 位）

```
31        28 27    12 11   1 0
+----------+--------+------+---+
| version  | part#  | mfg  | 1 |
+----------+--------+------+---+
   4 bits   16 bits  11 bits 1
```

SCR1 IDCODE: `0x20000001`
- version = 2
- part# = 0x0000
- mfg = 0x000 (Syntacore)

---

*本速查表配合 `docs/learning-notes.md` 和 `docs/module-walkthroughs/` 使用。*
