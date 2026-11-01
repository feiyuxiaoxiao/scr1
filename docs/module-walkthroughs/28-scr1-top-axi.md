# scr1_top_axi.sv 讲解

## 一句话职责

AXI 顶层：核心 + 存储层次 + 双 AXI 桥复用（mem_axi 模块用两次）。

## 文件信息

- **路径**: `src/top/scr1_top_axi.sv`
- **行数**: 701 行
- **配置依赖**: `SCR1_TCM_EN`、`SCR1_DBG_EN`

## 核心架构

### 1. 5 路 AXI 通道（`:30-80`）

| 通道 | 方向 | 含义 |
|---|---|---|
| `ar` | 读地址 | `arvalid`/`arready`/`araddr`/`arid` |
| `r` | 读数据 | `rvalid`/`rready`/`rdata`/`rid`/`rlast` |
| `aw` | 写地址 | `awvalid`/`awready`/`awaddr`/`awid`/`awlen` |
| `w` | 写数据 | `wvalid`/`wready`/`wdata`/`wstrb`/`wlast` |
| `b` | 写响应 | `bvalid`/`bready`/`bid`/`bresp` |

### 2. 复位选择（`:290-294`）

```systemverilog
assign axi_rst_n = (SCR1_AXI_RST_ASYNC) ? rst_n : cpu_rst_n_sync;
```

**异步复位**（`SCR1_AXI_RST_ASYNC=1`）：AXI 侧用 `rst_n`（外部）。
**同步复位**：用 `cpu_rst_n_sync`（与核心同步）。

### 3. Imem Router（`:140`）

```systemverilog
scr1_imem_router i_imem_router (
    .imem_req(pipe2imem_req),
    .port1_req(tcm_req),       // → TCM
    .port0_req(axi_imem_req),  // → AXI 桥
    ...
);
```

**无 TCM 时 bypass**（`:455-460`）：
```systemverilog
`ifndef SCR1_TCM_EN
assign axi_imem_req = pipe2imem_req;
assign pipe2imem_req_ack = axi_imem_resp != SCR1_MEM_RESP_NOTRDY;
`endif
```

### 4. Dmem Router（`:180`）

```systemverilog
scr1_dmem_router i_dmem_router (
    .dmem_req(pipe2dmem_req),
    .port1_req(tcm_req),       // → TCM
    .port2_req(timer_req),     // → Timer
    .port0_req(axi_dmem_req),  // → AXI 桥
    ...
);
```

### 5. 单一 mem_axi 模块复用两次

#### Imem AXI（`:541-613`）
```systemverilog
scr1_mem_axi #(.CORE_WIDTH(SCR1_AHB_WIDTH)) i_axi_imem (
    .core_req(axi_imem_req),
    .core_cmd(SCR1_MEM_CMD_RD),  // 指令口只读
    .core_wdata('0),             // 写数据填 0
    .arvalid(axi_imem_arvalid),
    .awvalid(axi_imem_awvalid),  // 写地址恒 0（只读）
    ...
);
```

#### Dmem AXI（`:619-691`）
```systemverilog
scr1_mem_axi #(.CORE_WIDTH(SCR1_AHB_WIDTH)) i_axi_dmem (
    .core_req(axi_dmem_req),
    .core_cmd(axi_dmem_cmd),
    .core_wdata(axi_dmem_wdata),
    .arvalid(axi_dmem_arvalid),
    .awvalid(axi_dmem_awvalid),
    .wvalid(axi_dmem_wvalid),
    ...
);
```

**对比**：
| 参数 | Imem AXI | Dmem AXI |
|---|---|---|
| `core_cmd` | 恒 `RD` | 动态 `RD/WR` |
| `core_wdata` | `'0` | 动态写数据 |
| `awvalid` | 恒 0 | 动态 |
| `wvalid` | 恒 0 | 动态 |

### 6. AXI 重初始化握手（`:693-699`）

```systemverilog
always_ff @(negedge core_rst_n_local, posedge clk) begin
    if (~core_rst_n_local) begin
        axi_reinit <= 1'b1;
    end else begin
        axi_reinit <= ~(axi_imem_idle & axi_dmem_idle);
    end
end
```

**含义**：
- 复位时 `axi_reinit=1`（桥重新初始化）
- 两个桥都 idle 后 `axi_reinit=0`（恢复正常）
- 用于 `mem_axi` 的 `core_req_ack` 门控（`:126-144`）

### 7. Timer 实例化（`:260`）

```systemverilog
scr1_timer i_timer (
    .dmem_req(timer_req),
    .irq(timer_irq),
    ...
);
```

### 8. AXI bypass 宏

| 宏 | 作用 |
|---|---|
| `SCR1_IMEM_AXI_REQ_BP` | 指令请求 bypass |
| `SCR1_IMEM_AXI_RESP_BP` | 指令响应 bypass |
| `SCR1_DMEM_AXI_REQ_BP` | 数据请求 bypass |
| `SCR1_DMEM_AXI_RESP_BP` | 数据响应 bypass |

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `rst_n` | in | 系统复位 |
| `clk` | in | 系统时钟 |
| `arvalid_i`/`arready_o` | - | AXI 读地址 |
| `rvalid_o`/`rready_i` | - | AXI 读数据 |
| `awvalid_i`/`awready_o` | - | AXI 写地址 |
| `wvalid_i`/`wready_o` | - | AXI 写数据 |
| `bvalid_o`/`bready_i` | - | AXI 写响应 |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TCM_EN` | TCM 实例化 + Router port1 使能 |
| `SCR1_DBG_EN` | Core 内调试子系统使能 |
| `SCR1_AXI_RST_ASYNC` | AXI 复位异步（vs 同步） |
| `SCR1_*_AXI_REQ_BP` | 请求 bypass |
| `SCR1_*_AXI_RESP_BP` | 响应 bypass |

## 配套文档

- 设计规格：`docs/scr1-top-axi/design-spec.md`
- 验证文档：`docs/scr1-top-axi/verification.md`
