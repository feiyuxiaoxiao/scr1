# scr1_ipic.sv 讲解

## 一句话职责

中断控制器：多中断源优先级编码 + 服务中跟踪 + CSR 桥接。

## 文件信息

- **路径**: `src/core/scr1_ipic.sv`
- **行数**: 605 行
- **配置依赖**: `SCR1_IPIC_EN`、`SCR1_IPIC_SYNC_EN`

## 核心功能

### 1. 中断源

| 中断源 | 优先级 | 触发方式 | 来源 |
|---|---|---|---|
| 外部中断 7 | 最高 | 电平/脉冲 | 外设 |
| ... | ... | ... | ... |
| 外部中断 0 | 最低 | 电平/脉冲 | 外设 |
| 软件中断 | - | 写 MIP.MSIP | CSR |
| 定时器中断 | - | mtimecmp 匹配 | Timer |

**外部中断 0~7**：`ipic2csr_ip[7:0]`，优先级编码（7 最高）。

### 2. 优先级编码器（`:200-280`）

```systemverilog
always_comb begin
    casez (ipic_irq_pnd_en)
        8'b1??_????: ipic2csr_ip_id = 7;
        8'b01??_????: ipic2csr_ip_id = 6;
        8'b001?_????: ipic2csr_ip_id = 5;
        8'b0001_????: ipic2csr_ip_id = 4;
        8'b0000_1???: ipic2csr_ip_id = 3;
        8'b0000_01??: ipic2csr_ip_id = 2;
        8'b0000_001?: ipic2csr_ip_id = 1;
        8'b0000_0001: ipic2csr_ip_id = 0;
        default:     ipic2csr_ip_id = 0;
    endcase
end
```

**作用**：同时多个中断 pending 时，选最高优先级送 CSR。

### 3. 服务中跟踪（`:300-380`）

```systemverilog
always_ff @(posedge clk) begin
    if (csr_ipic_take_i) begin      // CSR 接受中断
        ipic_service_id <= ipic2csr_ip_id;
        ipic_service_vd <= 1'b1;
    end else if (csr_ipic_eoi_i) begin  // End Of Interrupt
        ipic_service_vd <= 1'b0;
    end
end
```

**为什么需要？** 中断 handler 执行期间，同级中断必须屏蔽（防止无限嵌套）。IPIC 记住"正在服务哪个中断"，`ipic2csr_ip` 输出时排除它。

### 4. 同步模式（`SCR1_IPIC_SYNC_EN`，`:400-480`）

异步中断信号需同步到 SysCLK：
- 2 级触发器同步
- 脉冲展宽（确保 CSR 采样到）

### 5. 与 CSR 接口

| 信号 | 方向 | 含义 |
|---|---|---|
| `ipic2csr_ip_i` | out | 中断 pending（按优先级编码） |
| `ipic2csr_ie_i` | out | 中断使能（MIE 对应位） |
| `csr2ipic_meie_i` | in | MIE.MEIE（外部中断使能） |
| `csr2ipic_msie_i` | in | MIE.MSIE（软件中断使能） |
| `csr2ipic_mtie_i` | in | MIE.MTIE（定时器中断使能） |
| `csr_ipic_take_i` | in | CSR 接受中断（锁存 service_id） |
| `csr_ipic_eoi_i` | in | 中断结束（清除 service_vd） |

## 中断完整流程

```
外设拉高 irq → IPIC 优先级编码 → ip & ie 两级门 → EXU 察觉
→ CSR 接受（take） → 锁存 service_id → 屏蔽同级
→ handler 执行 → 写 MIP 清 pending → CSR 发 EOI → IPIC 清除 service_vd
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_IPIC_EN` | IPIC 模块使能，否则外部中断直通 CSR |
| `SCR1_IPIC_SYNC_EN` | 中断输入同步 + 脉冲展宽 |

## 配套文档

- 设计规格：`docs/scr1-ipic/design-spec.md`
- 验证文档：`docs/scr1-ipic/verification.md`
