# scr1_pipe_csr.sv 讲解

## 一句话职责

控制状态寄存器：特权模式状态 + trap（异常 + 中断）进出流程 + 计数器。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_csr.sv`
- **行数**: 1169 行
- **配置依赖**: `SCR1_IPIC_EN`、`SCR1_DBG_EN`、`SCR1_MCOUNTEN_EN`

## 主线：Trap 机制

### 事件：trap 的三种来源（`:282-297`）

```systemverilog
e_exc  = exu2csr_take_exc_i;    // 异常（指令自身导致的错误）
e_irq  = exu2csr_take_irq_i;    // 中断（外部异步请求）
e_mret = exu2csr_mret_update_i; // MRET 指令（从 trap 返回）
```

异常优先于中断（`e_irq = take_irq & ~take_exc`）。SVA 断言 `$onehot0({e_irq, e_exc, e_mret})` 确保三者互斥。

### 中断使能的"两级闸门"（重点，`:330-380`）

```systemverilog
csr2exu_ip_ie_o = csr_eirq_pnd_en | csr_sirq_pnd_en | csr_tirq_pnd_en;  // 有 pending 且本地使能
csr2exu_irq_o   = csr2exu_ip_ie_o & csr_mstatus_mie_ff;                 // 全局使能
```

- **每一路**：`MIP.pending & MIE.enable`
  - `csr_eirq_pnd_en = csr_mip_meip & csr_mie_meie_ff`（外部中断）
  - `csr_sirq_pnd_en = csr_mip_msip & csr_mie_msie_ff`（软件中断）
  - `csr_tirq_pnd_en = csr_mip_mtip & csr_mie_mtie_ff`（定时器中断）
- **总开关**：`MSTATUS.MIE`（全局中断使能位）

**进入中断时自动关总闸**，防止嵌套。

### Trap 发生时 CSR 自动做的事（`:627-646` 关键）

```systemverilog
case (1'b1)
    e_exc, e_irq:   // 进入 trap
        MIE  = 0;         // 关闭全局中断（防止嵌套）
        MPIE = 旧 MIE;     // 保存旧值
    e_mret:         // 从 trap 返回
        MIE  = MPIE;      // 恢复中断
        MPIE = 1;
    ...
endcase
```

同时并行写入：
- **MEPC** ← 出错的指令地址（异常）或被中断的指令地址（中断）
- **MCAUSE** ← 原因码：中断时最高位置 1 + 中断源编号
- **MTVAL** ← 附加信息（异常时是 trap 值，中断时清零）

**一个异常四个动作**（关中断、存 PC、存原因、存细节），硬件一个周期内完成。

### New PC：trap 跳到哪（`:1003-1019`）

MRET 跳回 `csr_mepc`；否则看 MTVEC 的 mode：

| MTVEC.mode | 异常入口 | 外部中断入口 | 软件中断入口 | 定时器中断入口 |
|---|---|---|---|---|
| `0` (direct) | `base` | `base` | `base` | `base` |
| `1` (vectored) | `base` | `base+11*4` | `base+3*4` | `base+7*4` |

**Vectored 模式**：不同 trap 跳不同地址，handler 无需查 cause 分发。

## 寄存器分类总览

### 1. 信息寄存器（只读）

| 寄存器 | 地址 | 含义 |
|---|---|---|
| `MVENDORID` | 0xF11 | 厂商 ID（Syntacore=0） |
| `MARCHID` | 0xF12 | 架构 ID（SCR1=0） |
| `MIMPID` | 0xF13 | 实现版本 |
| `MHARTID` | 0xF14 | Hart 号（0，单核） |

### 2. Trap Setup 寄存器

| 寄存器 | 地址 | 作用 |
|---|---|---|
| `MSTATUS` | 0x300 | 全局中断使能 MIE、MPIE、MPP |
| `MISA` | 0x301 | ISA 扩展使能（RVMC 等） |
| `MIE` | 0x304 | 中断使能（MEIE/MSIE/MTIE） |
| `MTVEC` | 0x305 | trap 入口地址 + mode |

### 3. Trap Handling 寄存器

| 寄存器 | 地址 | 作用 |
|---|---|---|
| `MSCRATCH` | 0x340 | 临时寄存器（handler 专用） |
| `MEPC` | 0x341 | trap 前 PC |
| `MCAUSE` | 0x342 | trap 原因码 |
| `MTVAL` | 0x343 | 附加信息 |
| `MIP` | 0x344 | 中断 pending 标志 |

### 4. 计数器

| 寄存器 | 地址 | 作用 |
|---|---|---|
| `MCYCLE` | 0xB00/B80 | 周期数（64 位，低 32+ 高 32） |
| `MINSTRET` | 0xB02/B82 | 已退休指令数（64 位） |

`csr_mcycle` 每周期 +1，`csr_minstret` 每退休一条**无异常**指令 +1（`instret_no_exc`）。

## CSR 读写接口（`:474-481`）

实现 CSRRW/CSRRS/CSRRC 三种写语义：

```systemverilog
WRITE:  w_data = 写入值            // CSRRW 直接覆盖
SET:    w_data = 写入值 | 旧值     // CSRRS 置位
CLEAR:  w_data = ~写入值 & 旧值    // CSRRC 清位
```

读写未实现的地址会置 `csr_r_exc`/`csr_w_exc`，最终变成非法指令异常（`:984-991`）。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `exu2csr_take_exc_i` | in | 触发异常 |
| `exu2csr_take_irq_i` | in | 触发中断 |
| `exu2csr_mret_update_i` | in | MRET 指令 |
| `csr2exu_new_pc_o` | out | trap 目标 PC |
| `csr2exu_ip_ie_o` | out | 有 pending 且使能的中断 |
| `csr2exu_irq_o` | out | 全局中断请求（经 MIE 门控） |
| `csr2exu_csr_rdata_o` | out | CSR 读回数据 |
| `exu2csr_csr_wdata_i` | in | CSR 写数据 |
| `exu2csr_csr_addr_i` | in | CSR 地址 |
| `exu2csr_csr_op_i` | in | 读/写/置位/清位 |

## 中断完整生命周期

```
外部拉高 → MIP 置位 → IP&IE 两级门 → EXU 察觉中断 → CSR 保存现场
→ PC 跳到 MTVEC → handler 运行 → MRET 恢复现场
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_IPIC_EN` | 启用 IPIC 中断控制器（多中断源优先级编码） |
| `SCR1_IPIC_SYNC_EN` | IPIC 同步模式（中断脉冲同步到 SysCLK） |
| `SCR1_DBG_EN` | 启用调试 CSR（DSCRATCH0/1、DPC、DCSR） |
| `SCR1_MCOUNTEN_EN` | MCOUNTEN 寄存器使能（控制计数器开关） |
| `SCR1_MTVEC_MODE_EN` | MTVEC.mode 可写（支持 vectored 模式） |

## 与其他模块关系

```
EXU → CSR: 异常/中断/MRET 请求 + CSR 读写
CSR → EXU: new_pc（trap 目标）+ ip_ie（中断 pending）+ CSR 读回
CSR ↔ IPIC: 中断源输入 + 服务中反馈
CSR ↔ HDU/TDU: 调试事件
```

## 配套文档

- 设计规格：`docs/scr1-pipe-csr/design-spec.md`
- 验证文档：`docs/scr1-pipe-csr/verification.md`
