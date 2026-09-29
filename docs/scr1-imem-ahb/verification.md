# SCR1 指令存储 AHB 桥验证文档（Verification）

- 被测模块：`scr1_imem_ahb`（`src/top/scr1_imem_ahb.sv`，319 行）
- 验证目标：请求 FIFO、FSM 状态迁移、响应处理、AHB 事务生成

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-IAHB-1 | `imem_req_ack=~req_fifo_full` | `:85` | 握手观察 |
| FC-IAHB-2 | 响应映射 OKAY→`RDY_OK`，否则 `RDY_ER` | `:90-94` | 激励不同 `hresp` |
| FC-IAHB-3 | bypass 请求 FIFO 行为 | `:99-120` | `OUT_BP` 配置 |
| FC-IAHB-4 | 深度 2 FIFO 读写/覆盖 | `:122-174` | `{rd,wr}` 四组合 |
| FC-IAHB-5 | `full/empty` 边界 | `:166-167` | cnt=0/2 检查 |
| FC-IAHB-6 | FSM `ADDR→DATA→ADDR` | `:179-203` | 波形状态观察 |
| FC-IAHB-7 | 错误响应回 `ADDR` | `:193-195` | `hresp=ERROR` 激励 |
| FC-IAHB-8 | `req_fifo_rd` 条件 | `:205-222` | 读使能观察 |
| FC-IAHB-9 | 响应 IN_BP 组合 vs 寄存 | `:227-245` | 两配置对比 |
| FC-IAHB-10 | AHB 常量（hprot/hburst/hsize/hmastlock） | `:251-258` | 信号检查 |
| FC-IAHB-11 | `htrans` 生成 | `:260-281` | 波形观察 |

---

## 2. SVA 断言检查（`:285-317`）

| 断言 | 性质 | 行号 |
|---|---|---|
| `..._REQ_XCHECK` | `imem_req` 无 X | 291-294 |
| `..._ADDR_XCHECK` | `imem_req` 时 `imem_addr` 无 X | 296-299 |
| `..._ADDR_ALLIGN` | 地址 4 字节对齐 | 301-304 |
| `..._HREADY_XCHECK` | `hready` 无 X | 307-310 |
| `..._HRESP_XCHECK` | `hresp` 无 X | 312-315 |

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| 单次取指 | FIFO 空 → 一次 ADDR/DATA |
| 背靠背取指 | FIFO 非空连续 NONSEQ |
| 满 FIFO | `imem_req_ack=0` 反压 |
| 错误响应 | `hresp=ERROR` 后回 IDLE |
| 等待插入 | `hready=0` 时状态保持 |
| bypass/缓冲组合 | 四种宏组合 |

---

## 4. 验证结论

FSM 与 FIFO 覆盖单 outstanding 取指的握手、反压与错误路径；SVA 保证地址对齐与无 X 传播。
