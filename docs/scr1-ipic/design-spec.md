# SCR1 IPIC 集成可编程中断控制器 设计规格（Design Specification）

- 模块：`scr1_ipic`（`src/core/pipeline/scr1_ipic.sv`，605 行）
- 条件：仅在 `SCR1_IPIC_EN` 时编译（`:36`、`:605`）
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:581-595`）
- 职责：外部 IRQ 线同步/边沿检测/反相、中断优先级仲裁、向 CSR 产生机器模式中断请求

---

## 1. 概述与职责

IPIC（Integrated Programmable Interrupt Controller）是 SCR1 内部的中断控制器，
支持最多 16 条外部中断线（`SCR1_IRQ_VECT_NUM`），提供：

1. **IRQ 线处理**：可选两级同步（`SCR1_IPIC_SYNC_EN`）、电平/边沿检测、极性反相；
2. **寄存器组**：CISV/CICSR/EOI/SOI/IDX/IPR/ISVR/IER/IMR/IINVR/ICSR；
3. **优先级仲裁**：固定优先级（索引小者优先），支持嵌套与 EOI 后立即升级；
4. **中断请求**：向 CSR 的 MEIP 输出（`ipic2csr_irq_m_req_o`）。

> 与 PLIC 不同，IPIC 采用**内部固定优先级**，并通过 CSR 映射的 8 个 3 位地址
> （`csr2ipic_addr_i[2:0]`）访问寄存器。

---

## 2. 端口（`:40-56`）

| 信号 | 方向 | 行号 |
|---|---|---|
| `rst_n` | in | 43 |
| `clk` | in | 44 |
| `soc2ipic_irq_lines_i[SCR1_IRQ_LINES_NUM-1:0]` | in | 47 |
| `csr2ipic_r_req_i` | in | 50 |
| `csr2ipic_w_req_i` | in | 51 |
| `csr2ipic_addr_i[2:0]` | in | 52 |
| `csr2ipic_wdata_i[XLEN-1:0]` | in | 53 |
| `ipic2csr_rdata_o[XLEN-1:0]` | out | 54 |
| `ipic2csr_irq_m_req_o` | out | 55 |

---

## 3. 参数与常量（`scr1_ipic.svh`）

| 参数 | 值 | 行号 |
|---|---|---|
| `SCR1_IRQ_VECT_NUM` | 16 | :15 |
| `SCR1_IRQ_VECT_WIDTH` | `$clog2(17)`=5 | :16 |
| `SCR1_IRQ_LINES_NUM` | 16 | :17 |
| `SCR1_IRQ_LINES_WIDTH` | `$clog2(16)`=4 | :18 |
| `SCR1_IRQ_VOID_VECT_NUM` | 16（0x10，void） | :19 |
| `SCR1_IRQ_IDX_WIDTH` | `$clog2(16)`=4 | :20 |

寄存器地址（`:23-30`）：

| 地址 | 名称 | 属性 |
|---|---|---|
| 3'h0 | CISV | RO |
| 3'h1 | CICSR | {IP, IE} |
| 3'h2 | IPR | RW1C |
| 3'h3 | ISVR | RO |
| 3'h4 | EOI | RZW（写结束） |
| 3'h5 | SOI | RZW（写开始） |
| 3'h6 | IDX | RW |
| 3'h7 | ICSR | RW |

ICSR 字段偏移（`:32-43`）：IP=0、IE=1、IM=2、INV=3、IS=4、PRV[9:8]、LN[15:12]、
PRV_M=2'b11。

---

## 4. 局部类型与函数（`:58-143`）

### 4.1 类型

- `type_scr1_search_one_2_s`（`:61-64`）：2 位“找首个 1”结果 `{vd, idx}`；
- `type_scr1_search_one_16_s`（`:66-69`）：16 位版本，`idx` 宽 4；
- `type_scr1_icsr_m_s`（`:71-78`）：ICSR 读模型；
- `type_scr1_cicsr_s`（`:80-83`）：CICSR 读模型。

### 4.2 `scr1_search_one_2`（`:89-98`）

```systemverilog
tmp.vd  = |din;
tmp.idx = ~din[0];   // din[0] 优先
```

### 4.3 `scr1_search_one_16`（`:100-143`）

四级归约树，返回 16 位输入中**最低置位索引**（即最高优先级）：

1. Stage1：8 组两两合并（`:114-119`）；
2. Stage2：4 组（`:122-127`）——`~tmp.idx` 选择保留低位分支的索引；
3. Stage3：2 组（`:130-135`）；
4. Stage4：输出 `vd` 与 4 位 `idx`（`:138-139`）。

> 该函数实现固定优先级：索引越小优先级越高。

---

## 5. IRQ 线处理（`:238-275`）

### 5.1 同步（`:242-257`，`SCR1_IPIC_SYNC_EN`）

```systemverilog
always_ff @(posedge clk, negedge rst_n) begin
    irq_lines_sync <= soc2ipic_irq_lines_i;
    irq_lines      <= irq_lines_sync;
end
```

未使能时 `irq_lines = soc2ipic_irq_lines_i`（`:256`）。

### 5.2 电平（`:262`）

```systemverilog
assign irq_lvl = irq_lines ^ ipic_iinvr_next;
```

反相寄存器作用于输入，得到“有效电平”。

### 5.3 边沿检测（`:267-275`）

```systemverilog
always_ff ... irq_lines_dly <= irq_lines;              // :267-273
assign irq_edge_detected = (irq_lines_dly ^ irq_lines) & irq_lvl;  // :275
```

上升沿检测（与反相后的电平做 AND）。

---

## 6. 读写接口（`:277-359`）

### 6.1 读多路器（`:285-328`）

默认 `ipic2csr_rdata_o='0`，`if (csr2ipic_r_req_i) case(addr)`：

| 地址 | 读内容 | 行号 |
|---|---|---|
| CISV | 服务中向量或 `VOID`（16） | 290-294 |
| CICSR | `{IE(1), IP(0)}` | 295-298 |
| IPR | `ipic_ipr_ff` | 299-301 |
| ISVR | `ipic_isvr_ff` | 302-304 |
| EOI/SOI | 0 | 305-308 |
| IDX | `ipic_idxr_ff` | 309-311 |
| ICSR | `{line, IS, PRV=11, INV, IM, IE, IP}` | 312-322 |
| default | `'x` | 323-325 |

### 6.2 写选择（`:334-359`）

默认各 `*_wr_req=0`，`if (csr2ipic_w_req_i) case(addr)`：

| 地址 | 动作 | 行号 |
|---|---|---|
| CISV | 静默只读 | 342 |
| CICSR | `cicsr_wr_req=1` | 343 |
| IPR | 静默只读（实际经 `ipic_ipr_clr_req`） | 344 |
| ISVR | 静默只读 | 345 |
| EOI | `eoi_wr_req=1` | 346 |
| SOI | `soi_wr_req=1` | 347 |
| IDX | `idxr_wr_req=1` | 348 |
| ICSR | `icsr_wr_req=1` | 349 |
| default | `'x` | 350-356 |

---

## 7. IPIC 寄存器（`:361-581`）

### 7.1 CISV（`:379-401`）

- 服务中向量；复位为 `VOID`(16)，即 bit[4]=1 → `irq_serv_vd=0`；
- 更新：`ipic_cisv_upd = irq_start_vd | ipic_eoi_req`（`:385`）；
- next（`:395-398`）：start→待服务 idx；EOI→若存在 EOI 后请求则取之，否则 VOID；
- 读回：`irq_serv_idx=cisv_ff[3:0]`（`:400`）、`irq_serv_vd=~cisv_ff[4]`（`:401`）。

### 7.2 CICSR（`:403-408`）

反映**当前服务向量**的 pending/enable：

```systemverilog
ipic_cicsr.ip = ipic_ipr_ff[irq_serv_idx] & irq_serv_vd;
ipic_cicsr.ie = ipic_ier_ff[irq_serv_idx] & irq_serv_vd;
```

### 7.3 EOI（`:410-414`）

```systemverilog
assign ipic_eoi_req = eoi_wr_req & irq_serv_vd;
```

写 EOI 结束当前服务中断。

### 7.4 SOI（`:416-424`）

```systemverilog
assign ipic_soi_req = soi_wr_req & irq_req_vd;
```

写 SOI 在存在待服务中断时启动之（真正开始还需 `irq_start_vd` 条件，见第 10 节）。

### 7.5 IDX（`:426-437`）

`idxr_wr_req` 时由 `wdata[3:0]` 更新，作为 ICSR 的向量选择。

### 7.6 IPR（`:439-477`）

- 更新：`ipic_ipr_upd = (next != ff)`（`:443`）；
- **清除请求**（`:453-465`）：

| 来源 | 清除位 |
|---|---|
| CICSR 写 IP | `[irq_serv_idx]` |
| IPR 写 | 全 `wdata[15:0]`（RW1C） |
| SOI 写 | `[irq_req_idx]` |
| ICSR 写 IP | `[ipic_idxr_ff]` |

- 清除条件（`:467-468`）：

```systemverilog
ipic_ipr_clr_cond = ~irq_lvl | ipic_imr_next;   // 电平模式需线已拉低；边沿模式恒可清
ipic_ipr_clr      = ipic_ipr_clr_req & ipic_ipr_clr_cond;
```

- next（`:470-477`）：

```systemverilog
ipic_ipr_next[i] = ipic_ipr_clr[i] ? 1'b0
                 : ~ipic_imr_ff[i] ? irq_lvl[i]              // 电平：跟随
                                   : ipic_ipr_ff[i] | irq_edge_detected[i];  // 边沿：锁存
```

### 7.7 ISVR（`:479-507`）

- 更新：`irq_start_vd | ipic_eoi_req`（`:483`）；
- `ipic_isvr_eoi`：清除当前服务位（`:493-498`）；
- next：start 置位 `[irq_req_idx]`，EOI 用 `isvr_eoi`（`:500-507`）。

### 7.8 IER（`:509-532`）

- 使能寄存器；更新：`cicsr_wr_req | icsr_wr_req`（`:513`）；
- CICSR 写 IE → `[irq_serv_idx]`；ICSR 写 IE → `[ipic_idxr_ff]`（`:523-531`）。

### 7.9 IMR（`:534-551`）

边沿/电平模式；仅 `icsr_wr_req` 更新 `[ipic_idxr_ff]`。

### 7.10 IINVR（`:553-570`）

极性反相；仅 `icsr_wr_req` 更新 `[ipic_idxr_ff]`。

### 7.11 ICSR（`:572-581`）

组合读模型：

```systemverilog
ipic_icsr.ip  = ipic_ipr_ff  [ipic_idxr_ff];
ipic_icsr.ie  = ipic_ier_ff  [ipic_idxr_ff];
ipic_icsr.im  = ipic_imr_ff  [ipic_idxr_ff];
ipic_icsr.inv = ipic_iinvr_ff[ipic_idxr_ff];
ipic_icsr.isv = ipic_isvr_ff [ipic_idxr_ff];
ipic_icsr.line= SCR1_IRQ_LINES_WIDTH'(ipic_idxr_ff);
```

---

## 8. 优先级与中断请求（`:583-601`）

### 8.1 待服务向量（`:587-591`）

```systemverilog
irq_req_v   = ipic_ipr_ff & ipic_ier_ff;          // pending 且 enabled
irr_priority= scr1_search_one_16(irq_req_v);      // 最低索引优先
irq_req_vd  = irr_priority.vd;
irq_req_idx = irr_priority.idx;
```

### 8.2 EOI 后候选（`:593-595`）

```systemverilog
isvr_priority_eoi = scr1_search_one_16(ipic_isvr_eoi);
irq_eoi_req_vd    = isvr_priority_eoi.vd;
irq_eoi_req_idx   = isvr_priority_eoi.idx;
```

### 8.3 优先级比较与请求（`:597-601`）

```systemverilog
irq_hi_prior_pnd     = irq_req_idx < irq_serv_idx;          // 固定优先级
ipic2csr_irq_m_req_o = irq_req_vd & (~irq_serv_vd | irq_hi_prior_pnd);
irq_start_vd         = ipic2csr_irq_m_req_o & ipic_soi_req;
```

- 有 pending(enabled) 向量，且（无服务中断 或 其优先级更高）→ 向 CSR 请求中断；
- 实际开始（更新 CISV/ISVR）还需软件写 SOI（`ipic_soi_req`）。

---

## 9. 中断处理流程

1. 外部线有效 → `irq_lvl`/`irq_edge_detected` → IPR 置 pending；
2. pending & IER → `irq_req_vd`，向 CSR 拉 `ipic2csr_irq_m_req_o`；
3. CSR 转 MEIP，CPU 取中断进入 trap；
4. 软件写 SOI → `irq_start_vd` → CISV/ISVR 记录服务向量；
5. 服务完成，软件写 EOI → 清除 ISVR/CISV；若存在更高优先级待服务则连续升级（`irq_eoi_req_*`）。

---

## 10. 假设与约束

1. `SCR1_IRQ_VECT_NUM` 为 2 的幂（`:15` 注释）；
2. 优先级固定：索引越小越高（`scr1_search_one_16`）；
3. 软件通过 CSR 的 8 个地址访问，地址全定义，default 分支不可达；
4. 电平模式 pending 在输入线仍高时不可清除；
5. `ipic2csr_irq_m_req_o` 可在无服务中断时就绪，但需 SOI 才真正启动。

---

## 附录 A：寄存器语义速查

| 寄存器 | 读 | 写 |
|---|---|---|
| CISV | 服务向量/void | 忽略 |
| CICSR | 当前向量 {IP,IE} | IP 清 pending；IE 设 enable |
| IPR | pending 位图 | RW1C |
| ISVR | 服务位图 | 忽略 |
| EOI | 0 | 结束当前服务 |
| SOI | 0 | 启动待服务中断 |
| IDX | 索引 | 设索引 |
| ICSR | 索引向量的全部状态 | 更新 IER/IMR/IINVR（按索引） |

## 附录 B：关键信号→消费者

| 信号 | 消费者 | 用途 |
|---|---|---|
| `ipic2csr_irq_m_req_o` | CSR（`soc2csr_irq_ext_i`） | MEIP |
| `ipic2csr_rdata_o` | CSR 读多路 | IPIC CSR 读 |
