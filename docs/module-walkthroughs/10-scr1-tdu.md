# scr1_tdu.sv 讲解

## 一句话职责

硬件触发器：4 个 MCONTROL 寄存器 + PC/地址匹配 + 调试异常请求。

## 文件信息

- **路径**: `src/core/scr1_tdu.sv`
- **行数**: 610 行
- **配置依赖**: `SCR1_TDU_EN`

## 核心功能

### 1. MCONTROL 寄存器（4 个，`:100-200`）

每个触发器对应一个 `MCONTROL` CSR（`tselect` 选择当前访问哪个）：

| 位 | 名称 | 含义 |
|---|---|---|
| `type` | [3:0] | 触发器类型（2=PC 匹配，3=地址匹配） |
| `dmode` | [11] | 命中后进入调试模式 |
| `action` | [12] | 0=调试异常，1=trace 日志 |
| `match` | [14:12] | 匹配类型（0=等值，1=NA，2=后沿） |
| `m` | [6] | M 模式使能 |
| `s` | [5] | S 模式使能（SCR1 不支持，恒 0） |
| `exec` | [3] | 执行匹配（PC） |
| `store` | [2] | Store 地址匹配 |
| `load` | [1] | Load 地址匹配 |

**SCR1 只支持 type=2（PC）和 type=3（地址）**。

### 2. PC 匹配（`:250-320`）

```systemverilog
if (mcontrol.type == 2 && mcontrol.exec && curr_pc[31:2] == tdata2[31:2]) begin
    tdu2csr_exec_match = 1'b1;
end
```

**用途**：断点。PC 匹配触发调试异常，handler 可以单步/查看寄存器。

### 3. 地址匹配（`:330-400`）

```systemverilog
if (mcontrol.type == 3) begin
    if (mcontrol.load && lsu_is_load && lsu_addr == tdata2)
        tdu2csr_ldst_match = 1'b1;
    if (mcontrol.store && lsu_is_store && lsu_addr == tdata2)
        tdu2csr_ldst_match = 1'b1;
end
```

**用途**：数据断点。监控特定地址的读写（如内存映射寄存器）。

### 4. 匹配时序（`:420-500`）

```systemverilog
always_ff @(posedge clk) begin
    if (match_condition) begin
        exec_match_ff <= mcontrol.exec;
        ldst_match_ff <= mcontrol.load | mcontrol.store;
    end
end
```

匹配结果打一拍送 CSR/HDU，确保稳定。

### 5. 与 HDU 交互

TDU 命中 → `tdu2csr_dmode_req=1` → HDU 接收 → `halt_req=1` → 流水线停取 → 进入调试模式。

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `csr_tdu_tselect_i` | in | 选择当前 MCONTROL（0~3） |
| `csr_tdu_tdata1_i` | in | 写 MCONTROL |
| `csr_tdu_tdata1_o` | out | 读 MCONTROL |
| `csr_tdu_tdata2_i` | in | 写匹配值（PC 或地址） |
| `csr_tdu_tdata2_o` | out | 读匹配值 |
| `tdu2csr_exec_match_o` | out | PC 匹配 |
| `tdu2csr_ldst_match_o` | out | 地址匹配 |
| `lsu_addr_i` | in | LSU 访存地址 |
| `curr_pc_i` | in | 当前 PC |

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_TDU_EN` | TDU 模块使能，否则匹配信号恒 0 |
| `SCR1_DBG_EN` | 必须同时开启（HDU 接收 dmode_req） |

## 配套文档

- 设计规格：`docs/scr1-tdu/design-spec.md`
- 验证文档：`docs/scr1-tdu/verification.md`
