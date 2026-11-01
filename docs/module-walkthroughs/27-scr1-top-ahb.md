# scr1_top_ahb.sv 讲解

## 一句话职责

AHB 顶层：核心 + 存储层次 + 双 AHB 桥（imem/dmem 各一个）。

## 文件信息

- **路径**: `src/top/scr1_top_ahb.sv`
- **行数**: 513 行
- **配置依赖**: `SCR1_TCM_EN`、`SCR1_DBG_EN`

## 核心架构

### 1. 复位同步（`:174-207`）

三个独立同步器（STAGES=2，`:86`）：
```systemverilog
scr1_reset_sync_cell #(.STAGES_AMOUNT(2)) i_rst_sync (
    .rst_n(rst_n),
    .clk(clk),
    .rst_n_in(cpu_rst_n),
    .rst_n_out(cpu_rst_n_sync)
);
```

**TAPC 特殊处理**（`:211-216`）：
```systemverilog
scr1_reset_and2_cell i_tapc_rst (
    .rst_n_in({trst_n, pwrup_rst_n}),
    .tapc_trst_n(tapc_trst_n)  // trst_n & pwrup_rst_n
);
```

### 2. 核心实例化（`:220-270`）

```systemverilog
scr1_core_top i_core (
    .rst_n(cpu_rst_n_sync),
    .pwrup_rst_n(pwrup_rst_n),
    .clk(clk),
    .tapc_tck(tapc_tck),
    .tapc_trst_n(tapc_trst_n),
    .curr_pc(curr_pc),
    ...
);
```

### 3. TCM 实例化（`:289`）

```systemverilog
`ifdef SCR1_TCM_EN
scr1_tcm i_tcm (
    .tcd_req(tcd_req),
    .tcd_addr(tcd_addr),
    .tcd_wdata(tcd_wdata),
    .tcd_rdata(tcd_rdata),
    ...
);
localparam TCM_SIZE = SCR1_TCM_ADDR_MASK + 1;  // :290
`endif
```

### 4. 路由器实例化

#### Imem Router（`:330`）
```systemverilog
scr1_imem_router i_imem_router (
    .imem_req(pipe2imem_req),
    .imem_addr(pipe2imem_addr),
    .port1_req(tcm_req),      // → TCM
    .port0_req(ahb_imem_req), // → AHB 桥
    ...
);
```

#### Dmem Router（`:360`）
```systemverilog
scr1_dmem_router i_dmem_router (
    .dmem_req(pipe2dmem_req),
    .dmem_addr(pipe2dmem_addr),
    .port1_req(tcd_req),       // → TCM
    .port2_req(timer_req),     // → Timer
    .port0_req(ahb_dmem_req),  // → AHB 桥
    ...
);
```

### 5. 双 AHB 桥

#### Imem AHB（`:460`）
```systemverilog
scr1_imem_ahb i_imem_ahb (
    .imem_req(ahb_imem_req),
    .imem_addr(ahb_imem_addr),
    .haddr(imem_haddr),
    .htrans(imem_htrans),
    ...
);
```

#### Dmem AHB（`:485`）
```systemverilog
scr1_dmem_ahb i_dmem_ahb (
    .dmem_req(ahb_dmem_req),
    .dmem_cmd(ahb_dmem_cmd),
    .dmem_addr(ahb_dmem_addr),
    .haddr(dmem_haddr),
    .hwrite(dmem_hwrite),
    ...
);
```

### 6. Timer 实例化（`:440`）

```systemverilog
scr1_timer i_timer (
    .dmem_req(timer_req),
    .dmem_addr(timer_addr),
    .irq(timer_irq),
    .rtc_ext_pulse(rtc_pulse),
    ...
);
```

### 7. AHB 总线粘合逻辑（`:300-330`）

**无 TCM 时 bypass**（`SCR1_TCM_EN` 未定义）：
```systemverilog
`ifndef SCR1_TCM_EN
assign pipe2imem_req_ack = imem2pipe_resp != SCR1_MEM_RESP_NOTRDY;
assign imem_htrans = (pipe2imem_req) ? SCR1_HTRANS_NONSEQ : SCR1_HTRANS_IDLE;
`endif
```

**DMEM 无 TCM 时 port1 响应**（`:427-434`）：
```systemverilog
`ifndef SCR1_TCM_EN
assign tcd_resp = SCR1_MEM_RESP_RDY_ER;  // 无 TCM 返回错误
assign tcd_rdata = '0;
`endif
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `rst_n` | in | 系统复位 |
| `pwrup_rst_n` | in | 上电复位 |
| `clk` | in | 系统时钟 |
| `tapc_tck` | in | JTAG 时钟 |
| `tapc_trst_n` | in | JTAG 复位 |
| `imem_haddr`/`imem_htrans` | out | 指令 AHB 输出 |
| `dmem_haddr`/`dmem_hwrite` | out | 数据 AHB 输出 |
| `timer_irq` | in | 定时器中断 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TCM_EN` | TCM 实例化 + Router port1 使能 |
| `SCR1_DBG_EN` | Core 内调试子系统使能 |

## 配套文档

- 设计规格：`docs/scr1-top-ahb/design-spec.md`
- 验证文档：`docs/scr1-top-ahb/verification.md`
