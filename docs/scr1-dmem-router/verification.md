# SCR1 数据存储路由器验证文档（Verification）

- 被测模块：`scr1_dmem_router`（`src/top/scr1_dmem_router.sv`，278 行）
- 验证目标：三端口地址判定与优先级、FSM 迁移、请求分发、响应回选

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-DRT-1 | `port_sel` 优先级 port1>port2>port0 | `:88-95` | 三区间地址激励 |
| FC-DRT-2 | `ADDR`→`DATA` 与 `port_sel_r` 锁存 | `:103-108` | 波形观察 |
| FC-DRT-3 | `RDY_OK` 连续保持 | `:111-118` | 背靠背请求 |
| FC-DRT-4 | `RDY_ER` 回 `ADDR` | `:119-121` | 错误响应激励 |
| FC-DRT-5 | `sel_req_ack` 按 `port_sel` 选择 | `:132-143` | ack 观察 |
| FC-DRT-6 | 响应按 `port_sel_r` 回选三端口 | `:145-164` | 端口切换场景 |
| FC-DRT-7 | 三端口请求生成 | `:176-252` | 各端口波形 |
| FC-DRT-8 | `XPROP` 未选中端口处理 | `:192-264` | 两配置对比 |

---

## 2. SVA 断言检查

| 断言 | 性质 | 行号 |
|---|---|---|
| `SCR1_SVA_DMEM_RT_XCHECK` | `dmem_req` 时 `{port_sel, dmem_cmd, dmem_width}` 无 X | 271-274 |

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| port0 命中 | 默认（外部总线） |
| port1 命中 | TCM 区间 |
| port2 命中 | 定时器区间 |
| 端口切换 | 相邻事务命中不同端口 |
| 重叠区间 | 验证 port1 优先于 port2 |
| 错误响应 | 选中端口返回 `RDY_ER` |
| 写事务 | `cmd/width/wdata` 正确分发 |

---

## 4. 验证结论

路由器覆盖三端口判定优先级、请求分发与响应回选；`port_sel_r` 保证数据阶段端口一致。
