# SCR1 CSR 控制状态寄存器 技术分析

> 对应源码: `src/core/pipeline/scr1_pipe_csr.sv`
> 模块名: `scr1_pipe_csr`
> 上级模块: EXU（执行单元）— 通过读写与事件接口交互
> 外部模块: IPIC（中断控制器）、HDU（硬件调试单元）、TDU（触发调试单元）

---

## 1. 模块定位

CSR 模块负责 SCR1 处理器中所有 Machine 级别控制状态寄存器的访问管理，并处理异常(EXC)、中断(IRQ)和 MRET 三类事件。它同时生成异常/中断发生时的新 PC 值，并提供计数器信息。

```
              ┌── soc 中断源 (EXT/SOFT/TIMER) ──┐
              │     soc 熔丝 MHARTID ──┤        │
              │                       v        │
EXU ──── R/W ──────→  scr1_pipe_csr  ────→ IPIC (可选)
EXU ──── 事件 ──────→       │         ────→ HDU  (可选)
EXU ──── PC ────────→       │         ────→ TDU  (可选)
              │               │                │
              │        csr2exu_new_pc_o        │
              │        csr2exu_irq_o            │
              │        csr2exu_rw_exc_o         │
              └───────────────┘
```

CSR 模块内部包含寄存器状态（有时钟复位），读写接口为组合逻辑。

---

## 2. 事件处理机制 (Events: EXC / IRQ / MRET)

### 2.1 事件优先级与仲裁

三个事件 `e_exc`、`e_irq`、`e_mret` 互斥，由 `exu2csr_take_exc_i` 和 `exu2csr_take_irq_i` 的优先级决定：

| 优先级 | 事件 | 条件 | 对 CSR 的影响 |
|:---:|------|------|------|
| 1 (最高) | `e_exc` | `take_exc & ~hdu_no_commit` | MEPC=curr_pc, MCAUSE=exc_code, MSTATUS.MIE←0, MSTATUS.MPIE←MIE_old, MTVAL=trap_val |
| 2 | `e_irq` | `take_irq & ~take_exc & ~hdu_no_commit` | MEPC=next_pc(非MRET时), MCAUSE.I=1 且 EC=IRQ编码, MSTATUS.MIE←0, MSTATUS.MPIE←MIE_old, MTVAL=0 |
| 3 (最低) | `e_mret` | `mret_update & ~hdu_no_commit` | MSTATUS.MIE←MPIE, MSTATUS.MPIE←1 |

调试模式禁止提交时（`hdu2csr_no_commit_i=1`），所有事件被阻塞。

信号 `e_irq_nmret` 在 IRQ 发生且当前不是 MRET 指令时置位，用于区分 MEPC 保存策略。

### 2.2 中断挂起与使能信号

三个中断源的"挂起且使能"条件信号由组合逻辑生成：

```systemverilog
csr_eirq_pnd_en = csr_mip_meip & csr_mie_meie_ff;  // 外部中断
csr_sirq_pnd_en = csr_mip_msip & csr_mie_msie_ff;  // 软件中断
csr_tirq_pnd_en = csr_mip_mtip & csr_mie_mtie_ff;  // 定时器中断
```

全局中断输出通过 MSTATUS.MIE 进行最终门控：

```systemverilog
csr2exu_ip_ie_o = csr_eirq_pnd_en | csr_sirq_pnd_en | csr_tirq_pnd_en;  // 局部使能
csr2exu_irq_o   = csr2exu_ip_ie_o & csr_mstatus_mie_ff;                  // 全局使能
```

### 2.3 中断优先级编码

IRQ 中断编码（用于 MCAUSE）通过 `case(1'b1)` 优先级仲裁器确定，按外部→软件→定时器的优先级排序：

| 优先级 | 条件 | `csr_mcause_ec_new` |
|:---:|------|------|
| 1 | `csr_eirq_pnd_en` | `SCR1_EXC_CODE_IRQ_M_EXTERNAL` (4'd11) |
| 2 | `csr_sirq_pnd_en` | `SCR1_EXC_CODE_IRQ_M_SOFTWARE` (4'd3) |
| 3 | `csr_tirq_pnd_en` | `SCR1_EXC_CODE_IRQ_M_TIMER` (4'd7) |
| default | — | `SCR1_EXC_CODE_IRQ_M_EXTERNAL` (4'd11) |

---

## 3. CSR 读写接口

### 3.1 写命令处理

有三种写命令类型，在 `csr_w_data` 生成阶段处理：

| 命令 | 操作 | 公式 | 对应 RISC-V 指令 |
|------|------|------|------|
| `SCR1_CSR_CMD_WRITE` | 直接写入 | `csr_w_data = exu2csr_w_data_i` | CSRRW |
| `SCR1_CSR_CMD_SET` | 读-修改-置位 | `csr_w_data = exu2csr_w_data_i \| csr_r_data` | CSRRS |
| `SCR1_CSR_CMD_CLEAR` | 读-修改-清位 | `csr_w_data = ~exu2csr_w_data_i & csr_r_data` | CSRRC |

### 3.2 读地址译码（always_comb 块）

读逻辑通过 `casez` 匹配 CSR 地址，按优先级列出：

| CSR 地址 | 读数据来源 | 条件/说明 |
|------|------|------|
| `0xF11` (MVENDORID) | `SCR1_CSR_MVENDORID` | 硬连线常量 `0x00000000` |
| `0xF12` (MARCHID) | `SCR1_CSR_MARCHID` | 硬连线常量 `32'd8` |
| `0xF13` (MIMPID) | `SCR1_CSR_MIMPID` | 硬连线常量 `0x22011200` |
| `0xF14` (MHARTID) | `soc2csr_fuse_mhartid_i` | 外部熔丝值 |
| `0x300` (MSTATUS) | `csr_mstatus` | MIE/MPIE/MPP 拼接 |
| `0x301` (MISA) | `SCR1_CSR_MISA` | 硬连线常量（扩展位编码） |
| `0x304` (MIE) | `csr_mie` | MEIE/MTIE/MSIE 拼接 |
| `0x305` (MTVEC) | `{csr_mtvec_base, 6'b0, csr_mtvec_mode}` | BASE(26) + 4'd0(保留) + MODE(1) |
| `0x340` (MSCRATCH) | `csr_mscratch_ff` | 通用暂存器 |
| `0x341` (MEPC) | `csr_mepc` | 对齐扩展后的异常返回地址 |
| `0x342` (MCAUSE) | `{csr_mcause_i_ff, mcause_ec_ff}` | I 位(1) + EC(31) |
| `0x343` (MTVAL) | `csr_mtval_ff` | 陷阱辅助值 |
| `0x344` (MIP) | `csr_mip` | MEIP/MTIP/MSIP 拼接 |
| `0xC0x` (HPMCOUNTER) | `csr_mcycle[31:0]` / `csr_minstret[31:0]` / `soc2csr_mtimer_val_i` | 用户计数器阴影（只读） |
| `0xC8x` (HPMCOUNTERH) | `csr_mcycle[63:32]` / `csr_minstret[63:32]` / `soc2csr_mtimer_val_i[63:32]` | 用户计数器高 32 位阴影（只读） |
| `0xB0x` (MHPMCOUNTER) | `csr_mcycle[31:0]` / `csr_minstret[31:0]` / `r_exc` | 机器计数器 |
| `0xB8x` (MHPMCOUNTERH) | `csr_mcycle[63:32]` / `csr_minstret[63:32]` / `r_exc` | 机器计数器高 32 位 |
| `0x32x` (MHPMEVENT) | `r_exc` | 事件选择器：对 0/1/2 返回异常，其余返回 0 |
| `0x7E0` (MCOUNTEN) | `csr_mcounten` | 非标准 CSR（条件编译） |
| IPIC 地址范围 | `ipic2csr_rdata_i` | 8 个 IPIC 寄存器（`0xBFx`） |
| HDU 地址范围 | `hdu2csr_rdata_i` | DCSR/DPC/DSCRATCH0/1 |
| TDU 地址范围 | `tdu2csr_rdata_i` | TSELECT/TDATA1/2/TINFO |
| 其他 | 0 | 默认返回 0 |

### 3.3 写地址译码

写逻辑通过 `exu2csr_w_req_i` 使能后，匹配地址设置对应的 `_upd` 标志：

| CSR 地址 | 操作 | 写更新信号 |
|------|------|------|
| `0x300` (MSTATUS) | 允许写入 MIE、MPIE 位 | `csr_mstatus_upd` |
| `0x301` (MISA) | 写入被忽略（只读） | 无 |
| `0x304` (MIE) | 允许写入 | `csr_mie_upd` |
| `0x305` (MTVEC) | 允许写入 BASE 可写位 + MODE | `csr_mtvec_upd` |
| `0x340` (MSCRATCH) | 允许写入 | `csr_mscratch_upd` |
| `0x341` (MEPC) | 允许写入 | `csr_mepc_upd` |
| `0x342` (MCAUSE) | 允许写入 | `csr_mcause_upd` |
| `0x343` (MTVAL) | 允许写入 | `csr_mtval_upd` |
| `0x344` (MIP) | 写入被忽略（只读） | 无 |
| `0xB0x` (MHPMCOUNTER) | 0=mcycle[31:0], 1=r_exc, 2=minstret[31:0] | `csr_mcycle_upd[0]` / `csr_w_exc` / `csr_minstret_upd[0]` |
| `0xB8x` (MHPMCOUNTERH) | 0=mcycle[63:32], 1=r_exc, 2=minstret[63:32] | `csr_mcycle_upd[1]` / `csr_w_exc` / `csr_minstret_upd[1]` |
| `0x32x` (MHPMEVENT) | 0/1/2 触发异常 | `csr_w_exc` |
| `0x7E0` (MCOUNTEN) | 允许写入 | `csr_mcounten_upd` |
| IPIC 地址 | 除 CISV/ISVR 外触发 IPIC 写 | `csr2ipic_w_req_o` |
| 其他 | 未定义地址 | `csr_w_exc` |

### 3.4 访问异常综合

```systemverilog
csr2exu_rw_exc_o = csr_r_exc | csr_w_exc
                  | (csr2hdu_req_o & hdu2csr_resp_i != SCR1_CSR_RESP_OK)  // HDU 错误
                  | (csr2tdu_req_o & tdu2csr_resp_i != SCR1_CSR_RESP_OK); // TDU 错误
```

任何非法地址访问（读或写）、HDU/TDU 响应错误都会触发 CSR 访问异常。

---

## 4. Machine Trap Setup 寄存器

### 4.1 MSTATUS（机器状态寄存器, `0x300`）

#### 位域结构

| 位 | 名称 | 含义 | 访问 |
|:---:|------|------|:---:|
| `[11:12]` (2 位) | MPP | 先前特权级，硬连线为 `2'b11` (Machine) | 只读 |
| `[7]` (1 位) | MPIE | 陷阱前中断使能 | 读写 |
| `[3]` (1 位) | MIE | 全局中断使能 | 读写 |
| 其他位 | — | 硬连线为 0 | 只读 |

#### 状态转移逻辑

| 事件 | MIE ← | MPIE ← | 说明 |
|------|:---:|:---:|------|
| 发生异常/中断 (`e_exc` / `e_irq`) | 0 | MIE_old | 禁用中断，保存旧使能 |
| MRET (`e_mret`) | MPIE_old | 1 | 恢复中断使能 |
| 软件写入 (`csr_mstatus_upd`) | `csr_w_data[3]` | `csr_w_data[7]` | 直接写入对应位 |
| 无事件 | 保持 | 保持 | — |

### 4.2 MIE（机器中断使能寄存器, `0x304`）

#### 位域结构

| 位 | 名称 | 含义 | 复位值 |
|:---:|------|------|:---:|
| `[11]` | MEIE | 外部中断使能 | 0 |
| `[7]` | MTIE | 定时器中断使能 | 0 |
| `[3]` | MSIE | 软件中断使能 | 0 |
| 其他位 | — | 硬连线为 0 | 0 |

写入时仅将 `csr_w_data` 对应偏移位的值写入 FF。该寄存器没有事件触发更新，仅受软件写入控制。

### 4.3 MTVEC（机器陷阱向量基址寄存器, `0x305`）

#### 位域结构

| 位 | 名称 | 含义 | 访问 |
|:---:|------|------|:---:|
| `[31:6]` (26 位) | BASE | 陷阱向量基址 | 部分可写 (取决于 `SCR1_MTVEC_BASE_WR_BITS`) |
| `[5:1]` (5 位) | — | 保留 | 硬连线为 0 |
| `[0]` (1 位) | MODE | 0=直接模式, 1=向量模式 | 条件可写 (`SCR1_MTVEC_MODE_EN`) |

#### BASE 可写性

MTVEC.base 通过 `generate` 块实现三种变体：

| `SCR1_MTVEC_BASE_WR_BITS` | 行为 |
|:---:|------|
| 0 | 全部只读，硬连线为复位值（如 `0x1C0`） |
| 26 | 全部可写（`csr_w_data[31:6]`） |
| 中间值 (如 16) | 高 `WR_BITS` 位可写，低 `RO_BITS` 位硬连线为复位值 |

RW 模式（`mtvec_base_rw` 和 `mtvec_base_ro_rw`）在 `csr_mtvec_upd` 有效时从 `csr_w_data` 对应位域写入。

#### MODE 工作模式

| mode | 值 | 说明 |
|:---:|:---:|------|
| Direct | `0` | 所有 trap 统一跳转到 `{BASE, 6'b0}` |
| Vectored | `1` | 异常跳转到 `{BASE, 6'b0}`；中断跳转到 `{BASE, 4×IRQ_code, 2'b0}` |

`SCR1_MTVEC_MODE_EN` 未定义时 mode 硬连线为 0（仅直接模式）。

---

## 5. Machine Trap Handling 寄存器

### 5.1 MSCRATCH（机器暂存寄存器, `0x340`）

通用 32 位读/写寄存器，用于存储 M-mode 上下文指针。无事件更新逻辑，仅通过 CSR 写入。

### 5.2 MEPC（机器异常程序计数器, `0x341`）

#### 位域

实际存储 `[31:PC_LSB]` 位，低 `PC_LSB` 位硬连线为 0。读取时扩展到完整的 32 位。

- RVC 扩展：`PC_LSB=1`，低 1 位补 0
- 无 RVC：`PC_LSB=2`，低 2 位补 0

#### 保存策略

| 事件 | `csr_mepc_next` | 含义 |
|------|------|------|
| `e_exc` | `curr_pc[31:PC_LSB]` | 保存发生异常的指令的 PC |
| `e_irq_nmret` | `next_pc[31:PC_LSB]` | 保存下一条指令的 PC（将被中断的指令尚未执行） |
| `csr_mepc_upd` | `csr_w_data[31:PC_LSB]` | 软件写入 |
| default | `csr_mepc_ff` | 保持不变 |

MRET 指令时若同时有 IRQ（`e_irq & ~e_irq_nmret`），MEPC 保持原值（由断言 `SCR1_SVA_CSR_MRET_IRQ` 检查）。

### 5.3 MCAUSE（机器异常原因寄存器, `0x342`）

#### 位域

| 位 | 名称 | 含义 |
|:---:|------|------|
| `[31]` | Interrupt | 1=中断, 0=异常 |
| `[30:0]` | Exception Code | 异常/中断编码 |

异常编码仅使用低 4 位（`SCR1_EXC_CODE_WIDTH_E=4`），高 27 位为 0。

读取时通过 `type_scr1_csr_mcause_ec_v`（`logic[30:0]`）进行类型转换。

#### 编码更新逻辑

| 事件 | I 位 | Exception Code | 来源 |
|------|:---:|------|------|
| `e_exc` | 0 | `exu2csr_exc_code_i` | EXU 传入的异常编码 |
| `e_irq` | 1 | `csr_mcause_ec_new` | 中断优先级仲裁结果 |
| `csr_mcause_upd` | `csr_w_data[31]` | `csr_w_data[3:0]` | 软件写入 |
| 复位 | 0 | `SCR1_EXC_CODE_RESET` (0) | — |

#### 异常编码映射表

| 编码 | 符号 | 来源 |
|:---:|------|------|
| 0 | INSTR_MISALIGN | EXU |
| 1 | INSTR_ACCESS_FAULT | IFU |
| 2 | ILLEGAL_INSTR | IDU / CSR |
| 3 | BREAKPOINT / IRQ_M_SOFTWARE | IDU/BRKM (异常) / IPIC (中断) |
| 4 | LD_ADDR_MISALIGN | LSU |
| 5 | LD_ACCESS_FAULT | LSU |
| 6 | ST_ADDR_MISALIGN | LSU |
| 7 | ST_ACCESS_FAULT / IRQ_M_TIMER | LSU (异常) / SoC (中断) |
| 11 | ECALL_M / IRQ_M_EXTERNAL | IDU (异常) / SoC (中断) |

### 5.4 MTVAL（机器陷阱值寄存器, `0x343`）

| 事件 | `csr_mtval_next` | 说明 |
|------|------|------|
| `e_exc` | `exu2csr_trap_val_i` | 异常特定信息（如非法指令编码、访问地址等） |
| `e_irq` | 0 | 中断时写入 0 |
| `csr_mtval_upd` | `csr_w_data` | 软件写入 |
| default | `csr_mtval_ff` | 保持 |

### 5.5 MIP（机器中断挂起寄存器, `0x344`）

只读寄存器，位域映射与 MIE 完全对应（使用相同偏移常量），但值为中断挂起状态：

| 位 | 名称 | 来源 |
|:---:|------|------|
| `[11]` | MEIP | `soc2csr_irq_ext_i` |
| `[7]` | MTIP | `soc2csr_irq_mtimer_i` |
| `[3]` | MSIP | `soc2csr_irq_soft_i` |

MIP 寄存器由 SOC 输入直接驱动（无 FF），写入被静默忽略。

---

## 6. Machine Counters/Timers

### 6.1 MCYCLE（机器周期计数器, `0xB00`/`0xB80`）

64 位计数器（仅在 `~SCR1_CSR_REDUCED_CNT` 时存在），每个时钟周期递增 1。

#### 递增控制链

```
csr_mcycle_lo_inc = 1'b1 & csr_mcounten_cy_ff    // 低 8 位每周期递增
csr_mcycle_hi_inc = csr_mcycle_lo_inc & (&csr_mcycle_lo_ff)   // 低 8 位全 1 时递增高 56 位
```

流水线化递增（低 8 位直接 FF，高 56 位在低 8 位溢出后更新）以优化时序。

#### 时钟选择

当 `SCR1_CLKCTRL_EN` 定义时，MCYCLE 使用非门控时钟 `clk_alw_on`，确保在核心时钟门控时仍然计数。

#### 写入支持

通过 `csr_mcycle_upd` 标志，支持低 32 位（`upd[0]`）和高 32 位（`upd[1]`）独立写入：
- 写低 32 位时：`lo_next = w_data[7:0]`，`hi_next` 取 `hi_new` 的高位和 `w_data[31:8]`
- 写高 32 位时：`hi_next = {w_data[31:0], hi_new[31:8]}`
- 同时写入两者时原子更新全部 64 位

### 6.2 MINSTRET（机器退休指令计数器, `0xB02`/`0xB82`）

64 位计数器（仅在 `~SCR1_CSR_REDUCED_CNT` 时存在），每条无异常退休的指令递增 1。

#### 递增控制链

```
csr_minstret_lo_inc = exu2csr_instret_no_exc_i & csr_mcounten_ir_ff   // 指令退休且计数使能
csr_minstret_hi_inc = csr_minstret_lo_inc & (&csr_minstret_lo_ff)
```

递增使能由 EXU 传入的 `exu2csr_instret_no_exc_i` 信号驱动。

写入机制与 MCYCLE 完全对称（`csr_minstret_upd[1:0]`）。

### 6.3 用户计数器阴影（`0xC0x`/`0xC8x`）

- `0xC00`/`0xC80` (CYCLE/CYCLEH): 映射到 MCYCLE 同值（只读）
- `0xC02`/`0xC82` (INSTRET/INSTRETH): 映射到 MINSTRET 同值（只读）
- `0xC01`/`0xC81` (TIME/TIMEH): 映射到 `soc2csr_mtimer_val_i`（64 位外部定时器值，只读）

---

## 7. Machine Information 寄存器（只读）

| CSR | 地址 | 值 | 含义 |
|------|:---:|------|------|
| MVENDORID | `0xF11` | `0x00000000` | 非商业实现（JEDEC 厂商 ID） |
| MARCHID | `0xF12` | `32'd8` | 架构 ID |
| MIMPID | `0xF13` | `0x22011200` | 实现版本 ID |
| MHARTID | `0xF14` | `soc2csr_fuse_mhartid_i` | 硬件线程 ID（外部熔丝） |

### 7.1 MISA（`0x301`）

硬连线编码，指示支持的 ISA 扩展：

| 条件 | MISA 位 | 含义 |
|------|:---:|------|
| 始终 | `[31:30] = 2'b01` | MXL=RV32 |
| 始终 | `[25] = 1` | Zicsr (CSR 指令扩展) |
| `SCR1_RVI_EXT` | `[8] = 1` | RV32I 基础整数指令集 |
| `SCR1_RVE_EXT` | `[4] = 1` | RV32E 嵌入式基础指令集 |
| `SCR1_RVM_EXT` | `[12] = 1` | M 扩展（整数乘除法） |
| `SCR1_RVC_EXT` | `[2] = 1` | C 扩展（压缩指令） |

---

## 8. 非标准 CSR

### 8.1 MCOUNTEN（计数器使能寄存器, `0x7E0`）

仅在 `SCR1_MCOUNTEN_EN` 定义时存在：

| 位 | 名称 | 含义 | 复位值 |
|:---:|------|------|:---:|
| `[0]` | CY | Cycle 计数器使能 | 1 |
| `[2]` | IR | Instret 计数器使能 | 1 |
| 其他 | — | 硬连线为 0 | 0 |

当 CY=0 时 MCYCLE 停止递增；当 IR=0 时 MINSTRET 停止递增。

---

## 9. 新 PC 生成

### 9.1 无 Vectored 模式 (`~SCR1_MTVEC_MODE_EN`)

```systemverilog
csr2exu_new_pc_o = (mret_instr & ~take_irq)
                 ? csr_mepc                                    // MRET: 跳转到 MEPC
                 : {csr_mtvec_base, 6'b0};                     // Trap: 跳转到 MTVEC.base
```

### 9.2 Vectored 模式 (`SCR1_MTVEC_MODE_EN`)

```
if (mret_instr & ~take_irq) → csr_mepc                        // MRET 返回
else if (mtvec_mode_vect):
    take_exc    → {csr_mtvec_base, 6'b0}                       // 异常: 固定 BASE
    eirq_pnd_en → {csr_mtvec_base, IRQ_M_EXTERNAL, 2'b0}      // 外部中断: BASE+0x2C
    sirq_pnd_en → {csr_mtvec_base, IRQ_M_SOFTWARE, 2'b0}      // 软件中断: BASE+0x0C
    tirq_pnd_en → {csr_mtvec_base, IRQ_M_TIMER, 2'b0}         // 定时器中断: BASE+0x1C
    default     → {csr_mtvec_base, 6'b0}                       // 默认: BASE
else → {csr_mtvec_base, 6'b0}                                  // Direct 模式
```

Vectored 模式下，各中断类型的向量地址 = `BASE + 4 × IRQ_exc_code`。

---

## 10. 外部模块接口

### 10.1 IPIC 接口

CSR 模块通过地址范围 `0xBF0`（`SCR1_CSR_ADDR_IPIC_BASE`）将 8 个 IPIC 寄存器暴露给 EXU：

| 偏移 | 寄存器 | 读 | 写 |
|:---:|------|:---:|:---:|
| 0 | CISV | 直通 `ipic2csr_rdata_i` | 静默忽略 |
| 1 | CICSR | 直通 `ipic2csr_rdata_i` | 转发 `csr2ipic_w_req_o` |
| 2 | IPR | 直通 `ipic2csr_rdata_i` | 转发 `csr2ipic_w_req_o` |
| 3 | ISVR | 直通 `ipic2csr_rdata_i` | 静默忽略 |
| 4 | EOI | 直通 `ipic2csr_rdata_i` | 转发 `csr2ipic_w_req_o` |
| 5 | SOI | 直通 `ipic2csr_rdata_i` | 转发 `csr2ipic_w_req_o` |
| 6 | IDX | 直通 `ipic2csr_rdata_i` | 转发 `csr2ipic_w_req_o` |
| 7 | ICSR | 直通 `ipic2csr_rdata_i` | 转发 `csr2ipic_w_req_o` |

IPIC 地址 = `exu2csr_rw_addr_i[2:0]`，写数据直通 `exu2csr_w_data_i`。

### 10.2 HDU 接口

仅在 `SCR1_DBG_EN` 定义时存在。4 个调试 CSR（DCSR/DPC/DSCRATCH0/DSCRATCH1）通过 `SCR1_HDU_DEBUGCSR_ADDR_WIDTH` 宽地址访问 HDU。

- `csr2hdu_req_o` = 读/写请求且当前无异常
- `csr2hdu_cmd_o` = 直通 EXU 写命令
- 读数据从 `hdu2csr_rdata_i` 获取
- 非 OK 响应触发 CSR 访问异常

HDU 的 `hdu2csr_no_commit_i` 信号阻塞所有事件处理（exe/irq/mret）。

### 10.3 TDU 接口

仅在 `SCR1_TDU_EN` 定义时存在。4 个触发寄存器（TSELECT/TDATA1/TDATA2/TINFO）按地址范围访问。

信号传输方式与 HDU 一致，但地址为 `exu2csr_rw_addr_i[SCR1_CSR_ADDR_TDU_OFFS_W-1:0]`。

---

## 11. 断言（仿真专用, `SCR1_TRGT_SIMULATION`）

### 11.1 X 态检查

| 断言名 | 检查内容 |
|--------|------|
| `SCR1_SVA_CSR_XCHECK_CTRL` | 控制信号 `r_req/w_req/take_irq/take_exc/mret_update/instret_no_exc` 不得为 X |
| `SCR1_SVA_CSR_XCHECK_READ` | 读请求时 `rw_addr/r_data/rw_exc` 不得为 X |
| `SCR1_SVA_CSR_XCHECK_WRITE` | 写请求时 `rw_addr/w_cmd/w_data/rw_exc` 不得为 X |
| `SCR1_SVA_CSR_XCHECK_READ_IPIC` | IPIC 读时 `csr2ipic_addr_o/ipic2csr_rdata_i` 不得为 X |
| `SCR1_SVA_CSR_XCHECK_WRITE_IPIC` | IPIC 写时 `csr2ipic_addr_o/csr2ipic_wdata_o` 不得为 X |

### 11.2 行为检查

| 断言名 | 检查内容 |
|--------|------|
| `SCR1_SVA_CSR_MRET` | MRET 后 MEPC 和 MTVAL 保持稳定 |
| `SCR1_SVA_CSR_MRET_IRQ` | MRET+IRQ 同时发生时 MEPC 保持且 `curr_pc ≠ mepc` |
| `SCR1_SVA_CSR_EXC_IRQ` | 异常+中断同时发生时：MIE=0, I=0, PC=MTVEC.base |
| `SCR1_SVA_CSR_EVENTS` | e_irq/e_exc/e_mret 互斥（onehot0） |
| `SCR1_SVA_CSR_RW_EXC` | `csr2exu_rw_exc_o` 仅在 `r_req\|w_req` 时有效 |
| `SCR1_SVA_CSR_MSTATUS_MIE_UP` | `csr2exu_mstatus_mie_up_o` 最多持续一个周期 |
| `SCR1_SVA_CSR_CYCLE_INC` | MCYCLE 每周期递增 1（未被写入时） |
| `SCR1_SVA_CSR_INSTRET_INC` | MINSTRET 在 `instret_no_exc` 时递增 1 |
| `SCR1_SVA_CSR_CYCLE_INSTRET_UP` | 计数器不能同时写高低 32 位（仅 32 位原子写入） |

---

## 12. RISC-V 特权规范兼容性说明

### 12.1 已实现的 Machine 级别 CSR（MVendorID 到 MHartID）

CSR 地址空间按标准 RISC-V Privileged Architecture v1.10+ 分配。SCR1 仅实现 Machine 级别（M-mode），所有 trap 都进入 M-mode。

### 12.2 未实现的寄存器

以下标准 CSR 未实现，访问将触发 `csr2exu_rw_exc_o`：

- `mstatush (0x310)` — 未实现
- `mtinst (0x34A)` — 未实现
- `mtval2 (0x34B)` — 未实现
- `menvcfg (0x30A)` — 未实现
- `mcounteren (0x306)` — 未实现（仅 U-mode 需要）
- `mhpmevent3-mhpmevent31` — 读返回 0，写对 0/1/2 返回异常
- `pmpcfg0-pmpcfg15, pmpaddr0-pmpaddr63` — 物理内存保护未实现

### 12.3 特权级支持

SCR1 仅实现 Machine mode，没有 User mode，因此：
- `mstatus.MPP` 硬连线为 `2'b11` (Machine)
- 所有读写操作均视为在最高特权级下执行，无特权级检查
- 写入只读 CSR 时静默忽略（MISA、MIP）
