# scr1_pipe_top.sv 讲解

## 一句话职责

流水线顶层：纯实例化 + 信号互连，不实现功能逻辑（"接线员"）。

## 文件信息

- **路径**: `src/core/pipeline/scr1_pipe_top.sv`
- **行数**: 825 行（MAX 配置）
- **配置依赖**: `SCR1_IPIC_EN`、`SCR1_DBG_EN`、`SCR1_TDU_EN`、`SCR1_TRACE_LOG_EN`

## 实例化的 9 个子模块

| 实例名 | 模块 | 作用 |
|---|---|---|
| `i_pipe_ifu` | `scr1_pipe_ifu` | 取指 |
| `i_pipe_idu` | `scr1_pipe_idu` | 译码 |
| `i_pipe_exu` | `scr1_pipe_exu` | 执行 |
| `i_pipe_mprf` | `scr1_pipe_mprf` | 寄存器堆 |
| `i_pipe_csr` | `scr1_pipe_csr` | 控制状态寄存器 |
| `i_pipe_ialu` | `scr1_pipe_ialu` | ALU（在 EXU 内） |
| `i_ipic` | `scr1_ipic` | 中断控制器（可选） |
| `i_tdu` | `scr1_tdu` | 硬件触发器（可选） |
| `i_hdu` | `scr1_hdu` | 调试单元（可选） |
| `i_tracelog` | `scr1_tracelog` | 轨迹日志（可选） |

注：`scr1_pipe_lsu` 在 EXU 内实例化，不在顶层。

## 核心互连

### 1. 取指通路

```systemverilog
i_pipe_ifu (
    .ifu2idu_instr_o(ifu2idu_instr),
    .ifu2idu_vd(ifu2idu_vd),
    .idu2ifu_rdy_i(idu2ifu_rdy),
    .exu2ifu_pc_new_req_i(exu2ifu_pc_new_req),
    .exu2ifu_pc_new_i(exu2ifu_pc_new),
    ...
);
```

### 2. 译码→执行

```systemverilog
i_pipe_idu (
    .ifu2idu_instr_i(ifu2idu_instr),
    .ifu2idu_vd_i(ifu2idu_vd),
    .idu2exu_cmd_o(idu2exu_cmd),
    .idu2exu_req_o(idu2exu_req),
    .exu2idu_rdy_i(exu2idu_rdy),
    ...
);
```

### 3. 寄存器读写

```systemverilog
i_pipe_mprf (
    .exu2mprf_rs1_addr_i(exu2mprf_rs1_addr),
    .exu2mprf_rs2_addr_i(exu2mprf_rs2_addr),
    .mprf2exu_rs1_data_o(mprf2exu_rs1_data),
    .mprf2exu_rs2_data_o(mprf2exu_rs2_data),
    .exu2mprf_w_req_i(exu2mprf_w_req),
    .exu2mprf_rd_addr_i(exu2mprf_rd_addr),
    .exu2mprf_rd_data_i(exu2mprf_rd_data),
    ...
);
```

### 4. 中断通路

```systemverilog
i_ipic (
    .ipic2csr_ip_i(ipic2csr_ip),      // 中断 pending
    .ipic2csr_ie_i(ipic2csr_ie),      // 中断使能
    .csr2ipic_meie_i(csr2ipic_meie),  // MIE.MEIE
    .csr2ipic_msie_i(csr2ipic_msie),  // MIE.MSIE
    .csr2ipic_mtie_i(csr2ipic_mtie),  // MIE.MTIE
    ...
);
```

### 5. 调试通路

```systemverilog
i_hdu (
    .hdu2pipe_halt_req_o(hdu2pipe_halt_req),    // 停机请求
    .hdu2pipe_dmode_o(hdu2pipe_dmode),          // 调试模式
    .pipe2hdu_curr_pc_o(curr_pc),               // 当前 PC
    .dm2pipe_active_i(dm2pipe_active),          // 调试活跃
    ...
);
```

## 时钟复位域

| 信号 | 域 | 说明 |
|---|---|---|
| `clk` | SysCLK | 系统时钟（所有模块共用） |
| `rst_n` | SysCLK | 系统复位（异步，同步释放） |
| `dbg_rst_n` | SysCLK | 调试复位（来自 SCU） |
| `dbg_en` | SysCLK | 调试使能（硬连线 1'b1） |

## 配置宏影响

| 宏 | 实例化变化 |
|---|---|
| `SCR1_IPIC_EN` | `i_ipic` 使能，否则 IPIC 信号直通 |
| `SCR1_DBG_EN` | `i_hdu`、`i_tdu` 使能 |
| `SCR1_TRACE_LOG_EN` | `i_tracelog` 使能 |
| `SCR1_NO_EXE_STAGE` | 不影响顶层（EXU 内部逻辑变化） |

## 与 core_top 关系

```
core_top → pipe_top: clk, rst_n, dbg_rst_n, dbg_en
pipe_top → core_top: curr_pc, instret, exu_busy, irq, halt_req
```

## 配套文档

- 设计规格：`docs/scr1-pipe-top/design-spec.md`
- 验证文档：`docs/scr1-pipe-top/verification.md`
