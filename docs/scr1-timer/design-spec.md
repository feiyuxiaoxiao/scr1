# SCR1 内存映射定时器设计规格（Design Specification）

- 模块：`scr1_timer`（`src/top/scr1_timer.sv`，271 行）
- 职责：提供 64 位 `mtime`/`mtimecmp` 及控制/分频寄存器，并产生定时器中断

---

## 1. 概述与职责

定时器是内存映射外设，寄存器按 5 位偏移寻址：

| 寄存器 | 偏移 | 行号 |
|---|---|---|
| CONTROL | `5'h0` | 34 |
| DIVIDER | `5'h4` | 35 |
| MTIMELO | `5'h8` | 36 |
| MTIMEHI | `5'hC` | 37 |
| MTIMECMPLO | `5'h10` | 38 |
| MTIMECMPHI | `5'h14` | 39 |

CONTROL 位：使能 `bit0`、时钟源 `bit1`（`:41-42`）；DIVIDER 宽度 10（`:43`）。

---

## 2. 端口（`:9-28`）

`rst_n`/`clk`/`rtc_clk`；数据接口 `dmem_req/cmd/width/addr/wdata/req_ack/rdata/resp`；
输出 `timer_val[63:0]`、`timer_irq`。

---

## 3. 寄存器与计数

- **CONTROL**（`:77-87`）：复位 `timer_en=1`、`timer_clksrc_rtc=0`；写时更新；
- **DIVIDER**（`:90-98`）：写 `dmem_wdata[9:0]`；
- **MTIME**（`:101-124`）：`time_posedge` 时自增；软件写 LOW/HIGH 覆盖对应 32 位；
- **MTIMECMP**（`:127-145`）：软件写覆盖。

---

## 4. 中断（`:147-164`）

```systemverilog
assign time_cmp_flag = (mtime_reg >= ((mtimecmplo_up | mtimecmphi_up) ? mtimecmp_new : mtimecmp_reg));
```

`timer_irq` 在 `time_cmp_flag` 时置 1；若已是 1，仅当再次写 `mtimecmp` 时按新比较值刷新
（`:152-164`，即未写 cmp 时不自动清中断）。

---

## 5. 分频与 RTC 同步（`:166-207`）

- `timeclk_cnt_en = (~clksrc_rtc ? 1 : rtc_ext_pulse) & timer_en`（`:169`）；
- 计数到 0 产生 `time_posedge`（`:101`）；写 DIVIDER 装载，`time_posedge` 重装 `timer_div`（`:171-182`）；
- `rtc_sync[0]` 在 `rtc_clk` 域翻转（`:189-197`），`rtc_sync[3:1]` 在 `clk` 域采样，
  `rtc_ext_pulse = rtc_sync[3]^rtc_sync[2]`（`:187`/`:199-207`）。

---

## 6. 存储接口（`:209-264`）

| 项 | 逻辑 | 行号 |
|---|---|---|
| `dmem_req_valid` | WORD 且 `addr[1:0]==0` 且偏移 ≤ MTIMECMPHI | 212-213 |
| `dmem_req_ack` | `1'b1` | 215 |
| 读响应 | 有效→`RDY_OK` 并按偏移返回；否则 `RDY_ER`；无请求 `NOTRDY`+清零 | 217-244 |
| 写译码 | `dmem_req & valid & cmd==WR` 按偏移产生 6 个 `*_up` | 246-264 |

`timer_val = mtime_reg`（`:269`）。

---

## 7. 附录

### 7.1 职责边界

- 仅字（32 位）访问，非对齐或越界返回错误；
- `mtime` 与 `mtimecmp` 均 64 位、分高低寄存器；
- 无 SVA。
