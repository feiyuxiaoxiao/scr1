# scr1_timer.sv 讲解

## 一句话职责

内存映射定时器：MTIME/MTIMECMP 计数 + RTC 脉冲 + 中断生成。

## 文件信息

- **路径**: `src/top/scr1_timer.sv`
- **行数**: 271 行
- **配置依赖**: 无（始终使能）

## 核心架构

### 1. 寄存器映射（`:33-39`）

| 寄存器 | 地址偏移 | 作用 |
|---|---|---|
| `CONTROL` | 0x0 | 分频使能 |
| `DIVIDER` | 0x4 | 分频系数 |
| `MTIMELO` | 0x8 | 计数器低 32 位 |
| `MTIMEHI` | 0xC | 计数器高 32 位 |
| `MTIMECMPLO` | 0x10 | 比较值低 32 位 |
| `MTIMECMPHI` | 0x14 | 比较值高 32 位 |

### 2. CONTROL 寄存器（`:41-42`）

| 位 | 名称 | 作用 |
|---|---|---|
| `[0]` | `timer_en` | 计数器使能 |
| `[1]` | `rtc_en` | RTC 脉冲使能 |

### 3. MTIME 计数器（`:101-113`）

```systemverilog
// 分频
always_ff @(posedge clk) begin
    if (timer_en) begin
        div_cnt <= div_cnt + 1;
        if (div_cnt == divider) begin
            time_posedge <= 1'b1;  // :101
            time_posedge <= 1'b0;
        end
    end
end

// MTIME+1
always_ff @(posedge clk) begin
    if (time_posedge) mtime <= mtime + 1;  // :106
end

// 软件写
if (wr_mtime) mtime <= wdata;  // :108-113
```

### 4. RTC 脉冲生成（`:184-207`）

```systemverilog
// 翻转
always_ff @(posedge clk) begin
    if (rtc_en && time_posedge) rtc_toggle <= ~rtc_toggle;
end

// 3 级同步
always_ff @(posedge clk) begin
    rtc_sync1 <= rtc_toggle;
    rtc_sync2 <= rtc_sync1;
    rtc_sync3 <= rtc_sync2;
end

// 边沿检测
assign rtc_ext_pulse = rtc_sync3 & ~rtc_sync2;  // :187
```

**用途**：外部 RTC 模块用（如系统定时器）。

### 5. 比较器 + 中断（`:150-164`）

```systemverilog
// 比较
assign mtimecmp_new = (mtime >= mtimecmp);

// Sticky 中断
always_ff @(posedge clk) begin
    if (mtimecmp_new) irq <= 1'b1;  // :156-162
    if (rd_mtimecmp) irq <= 1'b0;   // 读比较寄存器清中断
end
```

**为什么 sticky？** MTIME 可能等于 MTIMECMP 多拍，中断只需触发一次。读比较器清中断（类似 1149.1 的 clear-on-read）。

### 6. 内存接口（`:212-244`）

**只支持字对齐访问**：
```systemverilog
if (addr[1:0] == 2'b00) begin
    // 正常访问
end else begin
    dmem_resp = SCR1_MEM_RESP_RDY_ER;  // :237 错位返回错误
end
```

**读 mux**（`:220-230`）：
```systemverilog
case (addr[4:2])
    3'h2: dmem_rdata = CONTROL;
    3'h3: dmem_rdata = DIVIDER;
    3'h4: dmem_rdata = MTIMELO;
    3'h5: dmem_rdata = MTIMEHI;
    3'h6: dmem_rdata = MTIMECMPLO;
    3'h7: dmem_rdata = MTIMECMPHI;
endcase
```

**写 decoder**（`:215-218`）：
```systemverilog
assign dmem_req_ack = 1'b1;  // 永远 ack（单周期）
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `dmem_req_i` | in | 访存请求 |
| `dmem_cmd_i` | in | RD/WR |
| `dmem_addr_i` | in | 寄存器地址 |
| `dmem_wdata_i` | in | 写数据 |
| `dmem_req_ack_o` | out | 请求确认（恒 1） |
| `dmem_rdata_o` | out | 读回数据 |
| `dmem_resp_o` | out | 响应 |
| `irq_o` | out | 定时器中断 |
| `rtc_ext_pulse_o` | out | RTC 脉冲 |

## 中断生命周期

```
MTIME 递增 → MTIME>=MTIMECMP → irq 置 1
→ CSR 采样 → 触发定时器中断
→ Handler 读 MTIMECMP → irq 清 0
→ Handler 写新 MTIMECMP → 下一轮中断
```

## 配套文档

- 设计规格：`docs/scr1-timer/design-spec.md`
- 验证文档：`docs/scr1-timer/verification.md`
