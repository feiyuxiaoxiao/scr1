# SCR1 数据存储 AHB 桥验证文档（Verification）

- 被测模块：`scr1_dmem_ahb`（`src/top/scr1_dmem_ahb.sv`，480 行）
- 验证目标：宽度/字节偏移转换、读写事务、FIFO 与 FSM

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-DAHB-1 | `dmem_req_ack=~req_fifo_full` | `:226` | 握手观察 |
| FC-DAHB-2 | 宽度转换 BYTE/HWORD/WORD | `:82-103` | 三种宽度激励 |
| FC-DAHB-3 | 写数据按 addr 对齐 | `:105-156` | 4 种字节偏移 |
| FC-DAHB-4 | 读数据按 hwidth/haddr 回对齐 | `:158-197` | 读响应观察 |
| FC-DAHB-5 | 响应映射 OKAY/ERROR | `:231-235` | `hresp` 激励 |
| FC-DAHB-6 | bypass 请求 FIFO | `:240-267` | `OUT_BP` 配置 |
| FC-DAHB-7 | 深度 2 FIFO 及 `cnt=1` 覆盖 | `:269-333` | `{rd,wr}` 组合 |
| FC-DAHB-8 | FSM `ADDR`/`DATA` 迁移 | `:338-364` | 波形观察 |
| FC-DAHB-9 | `data_fifo` 锁存地址/宽度/写数据 | `:385-411` | 波形观察 |
| FC-DAHB-10 | 响应 IN_BP 组合 vs 寄存 | `:416-438` | 两配置对比 |
| FC-DAHB-11 | `hprot[DATA]=1`、`hsize=req_fifo[0].hwidth` | `:444`/`:450` | 信号检查 |
| FC-DAHB-12 | `hwrite`/`hwdata` 输出 | `:477-478` | 写事务观察 |

---

## 2. 转换函数验证

覆盖宽度×偏移的笛卡尔组合：

| 宽度 | 偏移取值 | 关注 |
|---|---|---|
| BYTE | `addr[1:0]`=0/1/2/3 | wdata 落到对应字节，rdata 从对应字节抽取 |
| HWORD | `addr[1]`=0/1 | 低/高 16 位 |
| WORD | 无关 | 直通 |

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| 写事务 | `hwrite=1`、`hwdata` 正确 |
| 读事务 | `hrdata` 回对齐 |
| 混合读写 | 背靠背不同宽度 |
| 反压 | FIFO 满时 `req_ack=0` |
| 错误响应 | `hresp=ERROR` 回 IDLE |
| bypass/缓冲 | 四种宏组合 |

---

## 4. 断言

本模块无 SVA；转换函数与 FSM 通过定向/随机激励覆盖。

---

## 5. 验证结论

数据桥覆盖读写、三宽度与字节偏移转换；`data_fifo` 保证响应阶段地址/宽度一致，
支持读数据正确回对齐。
