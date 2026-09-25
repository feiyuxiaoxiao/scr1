# SCR1 CSR 控制状态寄存器设计规格（Design Specification）

- 模块：`scr1_pipe_csr`（`src/core/pipeline/scr1_pipe_csr.sv`，1169 行）
- 层次：`scr1_pipe_top` 的子模块，例化于 `scr1_pipe_top.sv:506-539`
- 职责：RISC-V 机器模式 CSR、trap（异常/中断/MRET）进出、计数器、对外设 CSR 的桥接
- 参考：`docs/scr1_um.pdf`、RISC-V Privileged 规范

---

## 1. 概述与职责

CSR 是 CPU 的“特权状态与 trap 中心”，功能分六类：

1. **CSR 读写接口**：把 EXU 的 CSRRW/S/C 命令翻译为寄存器读写，未实现地址产生非法指令异常；
2. **事件逻辑**：接收异常/中断/MRET 三类事件，按优先级仲裁；
3. **Trap Setup**：MSTATUS、MISA、MIE、MTVEC；
4. **Trap Handling**：MSCRATCH、MEPC、MCAUSE、MTVAL、MIP；
5. **计数器**：MCYCLE、MINSTRET（64 位）+ 非标准 MCOUNTEN；
6. **外设桥接**：IPIC / HDU（调试）/ TDU（硬件触发）寄存器的读写转发。

**核心契约**：CSR 在事件拍完成 MEPC/MCAUSE/MTVAL/MSTATUS 的原子更新，并给出
`csr2exu_new_pc_o`（trap 入口或 MRET 返回地址）。

---

## 2. 端口与接口

见源码 `:47-126`。

### 2.1 公共（`:48-55`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `rst_n` | in | 49 | 复位 |
| `clk` | in | 50 | 门控时钟 |
| `clk_alw_on` | in | 53 | 非门控时钟（`SCR1_CLKCTRL_EN` 且非 `REDUCED_CNT`） |

### 2.2 SOC 信号（`:57-67`）

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `soc2csr_irq_ext_i` | in | 59 | 外部中断请求 → MEIP |
| `soc2csr_irq_soft_i` | in | 60 | 软件中断请求 → MSIP |
| `soc2csr_irq_mtimer_i` | in | 61 | 定时器中断请求 → MTIP |
| `soc2csr_mtimer_val_i` | in | 64 | 外部定时器 64 位值（供 `mtime` CSR） |
| `soc2csr_fuse_mhartid_i` | in | 67 | MHARTID 熔丝值 |

### 2.3 CSR ↔ EXU 读写（`:69-76`）

| 信号 | 方向 | 行号 |
|---|---|---|
| `exu2csr_r_req_i` | in | 70 |
| `exu2csr_rw_addr_i[11:0]` | in | 71 |
| `csr2exu_r_data_o` | out | 72 |
| `exu2csr_w_req_i` | in | 73 |
| `exu2csr_w_cmd_i` | in | 74 |
| `exu2csr_w_data_i` | in | 75 |
| `csr2exu_rw_exc_o` | out | 76 |

### 2.4 CSR ↔ EXU 事件（`:78-87`）

| 信号 | 方向 | 行号 |
|---|---|---|
| `exu2csr_take_irq_i` | in | 79 |
| `exu2csr_take_exc_i` | in | 80 |
| `exu2csr_mret_update_i` | in | 81 |
| `exu2csr_mret_instr_i` | in | 82 |
| `exu2csr_exc_code_i` | in | 83 |
| `exu2csr_trap_val_i` | in | 84 |
| `csr2exu_irq_o` | out | 85 |
| `csr2exu_ip_ie_o` | out | 86 |
| `csr2exu_mstatus_mie_up_o` | out | 87 |

### 2.5 外设接口

- **IPIC**（`:89-96`）：`csr2ipic_r_req_o`/`w_req_o`/`addr_o[2:0]`/`wdata_o`、`ipic2csr_rdata_i`；
- **HDU**（`:98-107`）：`csr2hdu_req_o`/`cmd_o`/`addr_o`/`wdata_o`、`hdu2csr_rdata_i`/`resp_i`/`no_commit_i`；
- **TDU**（`:109-117`）：`csr2tdu_*` 与 `tdu2csr_rdata_i`/`resp_i`。

### 2.6 PC 接口（`:119-125`）

| 信号 | 方向 | 行号 |
|---|---|---|
| `exu2csr_instret_no_exc_i` | in | 121（非 `REDUCED_CNT`） |
| `exu2csr_pc_curr_i` | in | 123 |
| `exu2csr_pc_next_i` | in | 124 |
| `csr2exu_new_pc_o` | out | 125 |

---

## 3. 局部参数/信号

### 3.1 PC 低位（`:132-136`）

```systemverilog
`ifdef SCR1_RVC_EXT
    localparam PC_LSB = 1;   // 压缩指令可 2 字节对齐
`else
    localparam PC_LSB = 2;
`endif
```

用于 MEPC 只保存地址的高位（低位恒 0）。

### 3.2 关键信号分组（`:142-275`）

| 分组 | 信号 | 行号 |
|---|---|---|
| MSTATUS | `csr_mstatus*` | 146-151 |
| MIE | `csr_mie*` | 154-158 |
| MTVEC | `csr_mtvec*` | 161-167 |
| MSCRATCH/MEPC | `csr_mscratch*`、`csr_mepc*` | 173-180 |
| MCAUSE | `csr_mcause*` | 183-188 |
| MTVAL | `csr_mtval*` | 191-193 |
| MIP | `csr_mip*` | 196-199 |
| 计数器 | `csr_minstret*`、`csr_mcycle*` | 206-229 |
| MCOUNTEN | `csr_mcounten*` | 237-240 |
| 事件 | `e_exc/e_irq/e_mret/e_irq_nmret` | 253-256 |
| 中断 pnd&en | `csr_eirq/sirq/tirq_pnd_en` | 259-261 |
| 异常 flags | `csr_w_exc/csr_r_exc/exu_req_no_exc` | 264-266 |

---

## 4. 事件逻辑（`:277-315`）

### 4.1 事件优先级（`:282-297`）

```systemverilog
assign e_exc  = exu2csr_take_exc_i  [ & ~hdu2csr_no_commit_i ];   // :282-286
assign e_irq  = exu2csr_take_irq_i & ~exu2csr_take_exc_i [ & ~no_commit ]; // :287-291
assign e_mret = exu2csr_mret_update_i [ & ~no_commit ];           // :292-296
assign e_irq_nmret = e_irq & ~exu2csr_mret_instr_i;               // :297
```

- **异常优先于中断**（`e_irq` 显式排除 `take_exc`）；
- 三者均受 `~hdu2csr_no_commit_i` 门控（调试禁止提交时冻结）；
- `e_irq_nmret`：中断 trap 但不是 MRET 指令（用于 MEPC 保存 pc_next 而非 pc_curr）。

### 4.2 中断 pending & enable（`:299-302`）

```systemverilog
csr_eirq_pnd_en = csr_mip_meip & csr_mie_meie_ff;   // 外部
csr_sirq_pnd_en = csr_mip_msip & csr_mie_msie_ff;   // 软件
csr_tirq_pnd_en = csr_mip_mtip & csr_mie_mtie_ff;   // 定时器
```

两级闸门：MIP.pending（SOC 输入）与 MIE.enable 的 AND。

### 4.3 中断异常码优先级（`:304-312`）

```systemverilog
case (1'b1)
    csr_eirq_pnd_en: ec_new = IRQ_M_EXTERNAL;  // 11
    csr_sirq_pnd_en: ec_new = IRQ_M_SOFTWARE;  // 3
    csr_tirq_pnd_en: ec_new = IRQ_M_TIMER;     // 7
    default        : ec_new = IRQ_M_EXTERNAL;
endcase
```

优先级：外部 > 软件 > 定时器（`case(1'b1)` 自上而下）。

### 4.4 无异常请求（`:314-315`）

```systemverilog
exu_req_no_exc = (r_req & ~csr_r_exc) | (w_req & ~csr_w_exc);
```

用于门控对 HDU/TDU 的转发请求：**本地地址译码已出错时不转发**。

---

## 5. CSR 读接口（`:317-469`）

### 5.1 结构

`always_comb`（`:324-467`）：先给默认值（`csr_r_data=0`、`csr_r_exc=0`、
各外设 req=0），再 `casez(exu2csr_rw_addr_i)` 分发。

### 5.2 地址译码表

| 地址 | CSR | 读数据 | 行号 |
|---|---|---|---|
| `0xF11` | MVENDORID | `SCR1_CSR_MVENDORID` | 339 |
| `0xF12` | MARCHID | `SCR1_CSR_MARCHID`（=8） | 340 |
| `0xF13` | MIMPID | `SCR1_CSR_MIMPID` | 341 |
| `0xF14` | MHARTID | `soc2csr_fuse_mhartid_i` | 342 |
| `0x300` | MSTATUS | `csr_mstatus` | 345 |
| `0x301` | MISA | `SCR1_CSR_MISA` | 346 |
| `0x304` | MIE | `csr_mie` | 347 |
| `0x305` | MTVEC | `{base,4'd0,2'(mode)}` | 348 |
| `0x340` | MSCRATCH | `csr_mscratch_ff` | 351 |
| `0x341` | MEPC | `csr_mepc` | 352 |
| `0x342` | MCAUSE | `{i_ff, ec_ff}` | 353 |
| `0x343` | MTVAL | `csr_mtval_ff` | 354 |
| `0x344` | MIP | `csr_mip` | 355 |
| `0xC00` 段 | HPMCOUNTER（用户，只读） | 见下 | 358-369 |
| `0xC80` 段 | HPMCOUNTERH | 见下 | 371-382 |
| `0xB00` 段 | MHPMCOUNTER（RW） | 见下 | 385-396 |
| `0xB80` 段 | MHPMCOUNTERH | 见下 | 398-409 |
| `0x320` 段 | MHPMEVENT | 见下 | 411-420 |
| MCOUNTEN | 非标准 | `csr_mcounten` | 422-424 |
| IPIC | 8 个地址 | `ipic2csr_rdata_i` + `csr2ipic_r_req_o` | 426-439 |
| HDU | DCSR/DPC/SCRATCH0/1 | `hdu2csr_rdata_i` + `csr_hdu_req` | 441-450 |
| TDU | TSELECT/TDATA1/2/TINFO | `tdu2csr_rdata_i` + `csr_brkm_req` | 452-461 |
| default | — | `csr_r_exc = r_req`（非法） | 463-465 |

### 5.3 计数器子译码

低 5 位（`:359-368` 等）选择索引：

| 索引 | MCYCLE/MINSTRET（`0xB00`/`0xB80`） | HPMCOUNTER（`0xC00`/`0xC80`） |
|---|---|---|
| 0 | `mcycle`（lo/hi） | `mcycle`（lo/hi） |
| 1 | `csr_r_exc`（读触发异常） | `mtimer_val` 低/高 32 位 |
| 2 | `minstret`（lo/hi） | `minstret`（lo/hi） |
| 其它 | 返回 0 | 返回 0 |

- **用户只读计数器 `mtime`（索引 1）** 由外部 `soc2csr_mtimer_val_i` 提供；
- MHPMCOUNTER 索引 1 读会产生 `csr_r_exc`（`0xB01` 未实现）；
- MHPMEVENT 索引 0/1/2 读产生 `csr_r_exc`（`:411-420`）。

### 5.4 读数据输出

```systemverilog
assign csr2exu_r_data_o = csr_r_data;   // :469
```

---

## 6. CSR 写接口（`:471-600`）

### 6.1 写数据语义（`:474-481`）

```systemverilog
case (exu2csr_w_cmd_i)
    SCR1_CSR_CMD_WRITE : csr_w_data =  exu2csr_w_data_i;            // CSRRW
    SCR1_CSR_CMD_SET   : csr_w_data =  exu2csr_w_data_i | csr_r_data; // CSRRS
    SCR1_CSR_CMD_CLEAR : csr_w_data = ~exu2csr_w_data_i & csr_r_data; // CSRRC
    default            : csr_w_data = '0;
endcase
```

注意 SET/CLEAR 使用**读数据** `csr_r_data`（当前值）参与运算。

### 6.2 更新使能与写异常（`:483-600`）

同样先清默认值，再 `if (w_req) casez(addr)`：

| 地址 | 行为 | 行号 |
|---|---|---|
| MSTATUS | `csr_mstatus_upd=1` | 508 |
| MISA | 忽略（无异常） | 509 |
| MIE | `csr_mie_upd=1` | 510 |
| MTVEC | `csr_mtvec_upd=1` | 511 |
| MSCRATCH | `csr_mscratch_upd=1` | 514 |
| MEPC | `csr_mepc_upd=1` | 515 |
| MCAUSE | `csr_mcause_upd=1` | 516 |
| MTVAL | `csr_mtval_upd=1` | 517 |
| MIP | 忽略（MIP 只读） | 518 |
| MHPMCOUNTER idx 0/2 | 对应 `mcycle/minstret_upd` 半字 | 521-532 |
| MHPMCOUNTER idx 1 | `csr_w_exc=1` | 523 |
| MHPMCOUNTERH idx 0/2 | 高半字更新 | 534-545 |
| MHPMEVENT idx 0/1/2 | `csr_w_exc=1` | 547-556 |
| MCOUNTEN | `csr_mcounten_upd=1` | 558-559 |
| IPIC 可写地址 | `csr2ipic_w_req_o=1` | 562-576 |
| HDU/TDU | 转发（`csr_hdu_req`/`csr_brkm_req` 已在读路径置位） | 578-593 |
| default | `csr_w_exc=1`（非法） | 595-597 |

> `MISA`/`MIP` 是“可读但写忽略”：写它们不报异常，值不变。

---

## 7. Trap Setup 寄存器（`:602-737`）

### 7.1 MSTATUS（`:612-653`）

只实现 3 个字段：MIE(bit3)、MPIE(bit7)、MPP[12:11]（硬连线 `2'b11`）。

```systemverilog
always_ff ... 复位: mie=1'b0, mpie=1'b1;        // :617-625
case (1'b1)
    e_exc, e_irq : mie_next = 1'b0; mpie_next = mie_ff;   // 进 trap
    e_mret       : mie_next = mpie_ff; mpie_next = 1'b1;  // 出 trap
    csr_mstatus_upd: mie_next/mpie_next 来自 csr_w_data;  // 软件写
    default      : hold;
endcase                                          // :627-646
```

聚合（`:648-653`）：MIE/MPIE 从 ff 取，MPP 固定 `2'b11`。

### 7.2 MIE（`:655-676`）

实现 MSIE(3)/MTIE(7)/MEIE(11)，复位 0，`csr_mie_upd` 时按位写。

### 7.3 MTVEC（`:678-737`）

**base 部分用 generate 三分支**（`:685-717`）按 `SCR1_MTVEC_BASE_WR_BITS`：

| 分支 | 条件 | 行为 |
|---|---|---|
| `mtvec_base_ro` | 写位数=0 | 全只读，硬连线复位值 |
| `mtvec_base_rw` | 写位数=全部 | 全可写 |
| `mtvec_base_ro_rw` | 其余 | 低位只读、高位可写（寄存器 + 拼接） |

**mode 部分**（`:719-737`）：`SCR1_MTVEC_MODE_EN` 时寄存器可写
（0=direct、1=vectored），否则硬连线 `DIRECT`。

---

## 8. Trap Handling 寄存器（`:739-862`）

### 8.1 MSCRATCH（`:751-761`）

完整可读写，绑定 `csr_mscratch_upd`。

### 8.2 MEPC（`:763-789`）

```systemverilog
case (1'b1)
    e_exc       : mepc_next = pc_curr_i[XLEN-1:PC_LSB];  // 异常指令地址
    e_irq_nmret : mepc_next = pc_next_i[XLEN-1:PC_LSB];  // 被中断指令的下一条
    csr_mepc_upd: mepc_next = w_data[XLEN-1:PC_LSB];
    default     : hold;
endcase                                            // :776-783
// 读回低位补 0：RVC→{ff,1'b0}；非 RVC→{ff,2'b00}   // :785-789
```

- **异常保存“出错指令”地址**，**中断保存“下一条”地址**（RISC-V 语义）；
- 只存高位，读回时低位补 0。

### 8.3 MCAUSE（`:791-825`）

结构 `{i_ff, ec_ff}`；复位 `ec=SCR1_EXC_CODE_RESET`、`i=0`。

```systemverilog
e_exc         : i=0; ec=exc_code_i;      // 同步异常
e_irq         : i=1; ec=csr_mcause_ec_new; // 中断 + 源码
csr_mcause_upd: 来自 w_data;
default       : hold;
```

### 8.4 MTVAL（`:827-847`）

```systemverilog
e_exc        : mtval = trap_val_i;   // 异常附加信息
e_irq        : mtval = '0;           // 中断清零
csr_mtval_upd: mtval = w_data;
default      : hold;
```

### 8.5 MIP（`:849-862`）

只读聚合，来自 SOC：

```systemverilog
csr_mip_msip = soc2csr_irq_soft_i;
csr_mip_mtip = soc2csr_irq_mtimer_i;
csr_mip_meip = soc2csr_irq_ext_i;
```

---

## 9. 计数器（`:864-951`）

### 9.1 MCYCLE（`:873-913`）

- 64 位拆为 `lo[7:0]` 与 `hi[63:8]`，**按字节低位进位**；
- `csr_mcycle_lo_inc = 1'b1 [& mcounten_cy_ff]`（`:879-883`）：每周期 +1；
- `clk_alw_on` 时钟（`SCR1_CLKCTRL_EN` 时），保证休眠计数不停；
- 写支持**部分字**更新（`upd[0]` 写 lo 字节 + hi 低位，`upd[1]` 写 hi）。

### 9.2 MINSTRET（`:915-950`）

- `csr_minstret_lo_inc = exu2csr_instret_no_exc_i [& mcounten_ir_ff]`（`:920-924`）：
  **仅无异常退休指令计数**；
- 结构与 MCYCLE 相同。

---

## 10. 非标准 MCOUNTEN（`:953-977`）

- 位 CY(0)：MCYCLE 计数使能；位 IR(2)：MINSTRET 计数使能；
- 复位均 1（默认计数）。

---

## 11. CSR ↔ EXU 接口（`:979-1020`）

### 11.1 异常与中断输出

```systemverilog
csr2exu_rw_exc_o = csr_r_exc | csr_w_exc
                 | (csr2hdu_req_o & (hdu2csr_resp_i != OK))   // :984-991
                 | (csr2tdu_req_o & (tdu2csr_resp_i != OK));
csr2exu_ip_ie_o  = eirq_pnd_en | sirq_pnd_en | tirq_pnd_en;   // :992
csr2exu_irq_o    = csr2exu_ip_ie_o & csr_mstatus_mie_ff;      // :993
```

- `ip_ie`：本地 pending&enable（不含全局 MIE，用于 WFI 唤醒判断）；
- `irq`：叠加全局 `MSTATUS.MIE`。

### 11.2 MIE 更新脉冲（`:995`）

```systemverilog
csr2exu_mstatus_mie_up_o = csr_mstatus_upd | csr_mie_upd | e_mret;
```

供 EXU 判断“本拍中断使能刚发生变化”，避免重复触发（SVA `:1126-1129` 约束单拍）。

### 11.3 New PC 多路选择（`:997-1020`）

**无 MTVEC_MODE_EN**（`:998-1001`）：MRET 且非同时中断 → `csr_mepc`；否则 `base`。

**有 MTVEC_MODE_EN**（`:1002-1019`）：

1. MRET 且非 `take_irq` → `csr_mepc`；
2. 否则若 vectored：
   - `take_exc` → `base`；
   - 外部中断 → `{base, IRQ_M_EXTERNAL, 2'd0}`（offset=11×4）；
   - 软件中断 → `{base, IRQ_M_SOFTWARE, 2'd0}`（offset=3×4）；
   - 定时器 → `{base, IRQ_M_TIMER, 2'd0}`（offset=7×4）；
3. 否则（direct）→ `base`。

> 拼接把 4 位中断码放在地址 `[5:2]`，即 `base + 4×cause`。

---

## 12. CSR ↔ 外设桥接（`:1022-1052`）

- **IPIC**（`:1022-1030`）：`csr2ipic_addr_o` 取地址低 3 位；读/写请求分别转发；
- **HDU**（`:1032-1041`）：`csr2hdu_req_o = csr_hdu_req & exu_req_no_exc`，
  转发 cmd/addr/wdata；
- **TDU**（`:1043-1052`）：`csr2tdu_req_o = csr_brkm_req & exu_req_no_exc`。

---

## 13. 配置宏影响

| 宏 | 位置 | 影响 |
|---|---|---|
| `SCR1_RVC_EXT` | `:132-136`、`:785-789` | `PC_LSB`=1，MEPC 低位补 0 方式 |
| `SCR1_MTVEC_MODE_EN` | `:164-167`、`:724-737`、`:998-1020` | 是否支持 vectored 模式 |
| `SCR1_MCOUNTEN_EN` | `:236-241`、`:422-424`、`:497-499`、`:558-560`、`:880-882`、`:921-923`、`:957-977` | MCOUNTEN 计数器使能寄存器 |
| `SCR1_CSR_REDUCED_CNT` | `:51-55`、`:120-122`、`:204-230`、`:361-364` 等 | 精简为 32 位计数器、去掉 instret 输入 |
| `SCR1_CLKCTRL_EN` | `:52-54`、`:889-893`、`:1136-1140` | MCYCLE 用非门控时钟 |
| `SCR1_IPIC_EN` | `:37-39`、`:89-96`、`:333-335`、`:426-439`、`:501-503`、`:562-576`、`:1022-1030`、`:1080-1090` | IPIC 桥接 |
| `SCR1_DBG_EN` | `:40-42`、`:98-107`、`:270-272`、`:283-296`、`:327-329`、`:441-450`、`:578-584`、`:985-987`、`:1032-1041` | HDU 桥接与 no_commit 门控 |
| `SCR1_TDU_EN` | `:43-45`、`:109-117`、`:273-275`、`:330-332`、`:452-461`、`:586-593`、`:988-990`、`:1043-1052` | TDU 桥接 |
| `SCR1_TRGT_SIMULATION` | `:1054-1167` | 内建断言 |

---

## 14. 时序与流程

### 14.1 Trap 进入（异常或中断）

1. EXU 拉高 `take_exc_i` 或 `take_irq_i`；
2. CSR：MSTATUS.MIE←0、MPIE←旧 MIE；MEPC←出错/被中断 PC；MCAUSE←码；
   MTVAL←trap_val（异常）或 0（中断）；
3. `csr2exu_new_pc_o` 给出 trap 入口（base 或 base+4×cause）；
4. EXU 用它作为 New PC。

### 14.2 MRET

1. `mret_update_i` 拉高 → `e_mret`；
2. MSTATUS.MIE←MPIE、MPIE←1；
3. `csr2exu_new_pc_o = csr_mepc`（返回地址）。

### 14.3 CSR 读写

- 读：组合译码，当拍返回 `csr2exu_r_data_o`；未实现地址 → `csr_r_exc`；
- 写：cmd 决定数据，更新使能决定是否落寄存器；未实现 → `csr_w_exc`；
- `csr_r_exc`/`csr_w_exc` 经 EXU 变为非法指令异常。

---

## 15. 假设与约束

1. `exu2csr_rw_addr_i` 在读写期间稳定；
2. 异常优先级高于中断（硬件强制）；
3. MIP 完全由 SOC 输入决定，软件不可写；
4. 事件三者互斥（SVA `:1116-1119`）；
5. `csr2exu_mstatus_mie_up_o` 单拍有效（SVA `:1126-1129`）；
6. 计数器在与 MCOUNTEN 使能、时钟门控配合下单调递增（SVA `:1134-1163`）。

---

## 16. 内建断言（`:1054-1167`）

| 断言 | 行号 | 检查 |
|---|---|---|
| `XCHECK_CTRL` | 1061-1068 | 控制输入无 X |
| `XCHECK_READ` | 1070-1073 | 读时地址/数据/异常无 X |
| `XCHECK_WRITE` | 1075-1078 | 写时地址/cmd/数据/异常无 X |
| `XCHECK_READ_IPIC` | 1081-1084 | IPIC 读无 X |
| `XCHECK_WRITE_IPIC` | 1086-1089 | IPIC 写无 X |
| `MRET` | 1094-1097 | MRET 后 MEPC/MTVAL 稳定 |
| `MRET_IRQ` | 1099-1103 | MRET+IRQ 时 MEPC 稳定且 PC≠MEPC |
| `EXC_IRQ` | 1105-1114 | 异常+中断时 MIE=0、cause 为异常、PC=base |
| `EVENTS` | 1116-1119 | 三事件单热 |
| `RW_EXC` | 1121-1124 | 异常必有读写请求 |
| `MSTATUS_MIE_UP` | 1126-1129 | MIE 更新脉冲单拍 |
| `CYCLE_INC` | 1134-1148 | MCYCLE 递增正确 |
| `INSTRET_INC` | 1150-1158 | MINSTRET 递增正确 |
| `CYCLE_INSTRET_UP` | 1160-1163 | 计数器更新值合法 |

---

## 附录 A：CSR 地址表（`scr1_csr.svh:26-42`）

| 地址 | 名称 | 属性 |
|---|---|---|
| 0xF11 | MVENDORID | RO |
| 0xF12 | MARCHID | RO |
| 0xF13 | MIMPID | RO |
| 0xF14 | MHARTID | RO（外部熔丝） |
| 0x300 | MSTATUS | RW（MIE/MPIE，MPP=11） |
| 0x301 | MISA | RO |
| 0x304 | MIE | RW |
| 0x305 | MTVEC | RW |
| 0x340 | MSCRATCH | RW |
| 0x341 | MEPC | RW |
| 0x342 | MCAUSE | RW |
| 0x343 | MTVAL | RW |
| 0x344 | MIP | RO |
| 0xB00/0xB80 | MCYCLE/MINSTRET（MHPMCOUNTER） | RW |
| 0xC00/0xC80 | HPMCOUNTER 用户视图 | RO |
| 0xC01/0xC81 | `mtime`（外部定时器） | RO |
| 0x320 段 | MHPMEVENT | 访问异常 |

## 附录 B：异常/中断码

| 码 | 值 | 来源 |
|---|---|---|
| RESET | （复位占位） | `scr1_arch_types.svh` |
| IRQ_M_SOFTWARE | 3 | `scr1_arch_types.svh:54` |
| IRQ_M_TIMER | 7 | `:55` |
| IRQ_M_EXTERNAL | 11 | `:56` |

同步异常码由 EXU 传入 `exu2csr_exc_code_i`。

## 附录 C：关键字段偏移（`scr1_csr.svh`）

| 字段 | 偏移 |
|---|---|
| MSTATUS.MIE | 3（`:143`） |
| MSTATUS.MPIE | 7（`:144`） |
| MSTATUS.MPP | 11（`:145`） |
| MIE.MSIE/MTIE/MEIE | 3/7/11（`:157-159`） |
| MCOUNTEN.CY/IR | 0/2（`:163-164`） |

## 附录 D：信号 → 消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `csr2exu_r_data_o` | EXU 写回 mux | CSRR* 读结果 |
| `csr2exu_rw_exc_o` | EXU 异常 | 非法 CSR → ILLEGAL_INSTR |
| `csr2exu_irq_o` | EXU 中断判定 | 取中断 |
| `csr2exu_ip_ie_o` | EXU WFI | 本地中断唤醒 |
| `csr2exu_new_pc_o` | EXU New PC | trap 入口/MRET 返回 |
| `csr2exu_mstatus_mie_up_o` | EXU / CSR | 单拍更新标志 |
| `csr2ipic/hdu/tdu_*` | 外设 | CSR 转发 |
