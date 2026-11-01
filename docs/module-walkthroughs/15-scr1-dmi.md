# scr1_dmi.sv 讲解

## 一句话职责

DMI 桥：TAPC↔DM 两步协议桥接（DTMCS + DMI access 寄存器）。

## 文件信息

- **路径**: `src/core/scr1_dmi.sv`
- **行数**: 182 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心架构

### 1. 寄存器映射（`:52-74`）

#### DTMCS（Debug Transport Module Control/Status）
| 位 | 名称 | 含义 |
|---|---|---|
| `[17:16]` | `version` | DMI 版本（1） |
| `[27:18]` | `abits` | 地址位宽（DM 地址位数） |
| `[30]` | `dmstat` | DM 就绪标志 |
| `[31]` | `dmreset` | DM 复位请求 |

#### DMI Access
| 位 | 名称 | 含义 |
|---|---|---|
| `[1:0]` | `op` | 操作：00=NOP, 01=Read, 10=Write |
| `[8:2]` | `address` | DM 寄存器地址 |
| `[31:9]` | `data` | 读写数据 |

### 2. 两步协议（`:150-178`）

**Step 1：TAPC→DMI**（`:130-144`）
```systemverilog
// TAPC 选中 DMI 链
assign tap_dr_dmi = (ir_reg == SCR1_TAP_IR_DMI);

// 移位到 DMI 寄存器
if (tapc_dr_update) begin
    dmi_reg <= dmi_tdi;
end
```

**Step 2：DMI→DM**（`:150-178`）
```systemverilog
assign dmi2dm_req = (dmi_reg.op != 2'b00);  // op!=0 表示有请求
assign dmi2dm_wr  = (dmi_reg.op == 2'b10);  // Write
assign dmi2dm_addr = dmi_reg.address;
assign dmi2dm_data = dmi_reg.data;
```

### 3. DMI 读缓存（`:114-120`）

```systemverilog
// DMI 读 op=01，总是成功返回缓存数据
if (dmi_reg.op == 2'b01) begin
    dmi_dmi_rdata <= dm2dmi_rdata;  // DM 读回数据
    dmi_dmi_op <= 2'b00;            // 成功
end
```

**为什么需要缓存？** DMI 读是异步的（DM 可能延迟），但 JTAG 移位是同步的。缓存最近一次读结果，调试器随时可读。

### 4. DTMCS 读响应（`:114-115`）

```systemverilog
assign dmi_dtmcs_rdata = {
    1'b0,           // [31] dmreset
    1'b0,           // [30] dmstat
    7'h00,          // [27:18] abits
    2'b01,          // [17:16] version
    ...
};
```

固定返回 version=1、abits=DM 地址位数。

### 5. TAP 移位填充（`:123-125`）

```systemverilog
// DTMCS 移位时，未定义位填 0
assign dmi_dtmcs_tdo = {dmi_dtmcs_rdata, 9'b0};  // 补齐长度
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `tap_dmi_access_i` | in | TAPC 选中 DMI 链 |
| `dmi_tck_i` | in | DMI 时钟（=TAPC TCK） |
| `dmi_tdi_i` | in | DMI 串行输入 |
| `dmi_tdo_o` | out | DMI 串行输出 |
| `dmi2dm_req_o` | out | DM 请求 |
| `dmi2dm_wr_o` | out | 写操作 |
| `dmi2dm_addr_o` | out | DM 地址 |
| `dmi2dm_data_o` | out | 写数据 |
| `dm2dmi_resp_i` | in | DM 响应 |
| `dm2dmi_rdata_i` | in | 读回数据 |

## DMI 完整流程

```
调试器 → JTAG TMS/TDI → TAPC Shift-DR → DMI 寄存器锁存
→ DMI 解析 op/address/data → DM 请求
→ DM 执行 → DM 响应 → DMI 缓存
→ JTAG TDO 移回 → 调试器读取
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | DMI 模块使能 |

## 配套文档

- 设计规格：`docs/scr1-dmi/design-spec.md`
- 验证文档：`docs/scr1-dmi/verification.md`
