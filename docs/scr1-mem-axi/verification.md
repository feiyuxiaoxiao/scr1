# SCR1 存储器 AXI 桥验证文档（Verification）

- 被测模块：`scr1_mem_axi`（`src/top/scr1_mem_axi.sv`，363 行）
- 验证目标：核心握手、状态队列、三指针、AXI 事务与数据适配、bypass

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-MAXI-1 | `core_req_ack` 条件 | `:126-128` | 握手观察 |
| FC-MAXI-2 | `rready`/`bready` 按读写分离 | `:131-132` | 读/写事务 |
| FC-MAXI-3 | `core_idle` 汇总 | `:139-144` | 空闲判定 |
| FC-MAXI-4 | 请求缓冲写入 | `:146-152` | 波形观察 |
| FC-MAXI-5 | 新请求状态初始化 | `:164-175` | 状态位检查 |
| FC-MAXI-6 | 地址/数据阶段清零状态 | `:178-187` | 波形观察 |
| FC-MAXI-7 | 完成清 `req_resp` | `:190-193` | 波形观察 |
| FC-MAXI-8 | 三指针递增 | `:209-237` | 多笔事务 |
| FC-MAXI-9 | `arvalid/awvalid/wvalid` 生成 | `:241-243` | 事务波形 |
| FC-MAXI-10 | `rcvd_resp` 映射 | `:248-260` | `bresp`/`rresp` 激励 |
| FC-MAXI-11 | `wstrb` 宽度/偏移 | `:266-281` | 三宽度写 |
| FC-MAXI-12 | `wdata` 移位对齐 | `:285-286` | 三宽度写 |
| FC-MAXI-13 | `rcvd_rdata` 回对齐 | `:290-297` | 三宽度读 |
| FC-MAXI-14 | 响应 bypass 组合 vs 寄存 | `:300-313` | 两配置对比 |
| FC-MAXI-15 | AXI 常量（id/len/burst/cache） | `:317-341` | 信号检查 |

---

## 2. SVA 断言检查（`:344-361`）

| 断言 | 性质 | 行号 |
|---|---|---|
| `SCR1_SVA_AXI_X_CHECK0` | 握手输入无 X | 350-351 |
| `SCR1_SVA_AXI_X_CHECK1` | `core_req` 时 cmd/width/addr 无 X | 352-354 |
| `SCR1_SVA_AXI_X_CHECK2` | `bvalid` 时 `{bid,bresp}` 无 X | 355-357 |
| `SCR1_SVA_AXI_X_CHECK3` | `rvalid` 时 `{rid,rresp}` 无 X | 358-360 |

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| 单笔读 | AR→R，数据回对齐 |
| 单笔写 | AW+W→B，`wstrb`/`wdata` |
| 背靠背 | 多笔在途，三指针滚动 |
| bypass 通路 | `REQ_BP=1` 时 `force_*` |
| 满队列 | `req_resp` 反压 `core_req_ack` |
| 错误响应 | `bresp/rresp!=0` → `RDY_ER` |
| reinit | `axi_reinit` 时 `core_resp=NOTRDY` |

---

## 4. 验证结论

AXI 桥覆盖单拍读写、在途队列管理、宽度/字节偏移适配与 bypass 配置；SVA 保证无 X 传播。
