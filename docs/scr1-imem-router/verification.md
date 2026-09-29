# SCR1 指令存储路由器验证文档（Verification）

- 被测模块：`scr1_imem_router`（`src/top/scr1_imem_router.sv`，185 行）
- 验证目标：地址判定、端口选择、FSM 迁移、请求分发

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-IRT-1 | `port_sel` 地址匹配 | `:64` | 命中/未命中地址 |
| FC-IRT-2 | `ADDR`→`DATA` 转换与 `port_sel_r` 锁存 | `:72-77` | 波形观察 |
| FC-IRT-3 | `DATA` 连续 OKAY 保持 | `:80-87` | 背靠背请求 |
| FC-IRT-4 | `RDY_ER` 回 `ADDR` | `:88-90` | 错误响应激励 |
| FC-IRT-5 | `sel_req_ack` 使能条件 | `:101-107` | ack 观察 |
| FC-IRT-6 | 响应按 `port_sel_r` 回选 | `:109-110` | 端口切换场景 |
| FC-IRT-7 | port0/port1 请求分发 | `:122-136`/`:149-163` | 两端口波形 |
| FC-IRT-8 | `XPROP` 未选中端口处理 | `:138-144`/`:165-171` | 两配置对比 |

---

## 2. SVA 断言检查

| 断言 | 性质 | 行号 |
|---|---|---|
| `SCR1_SVA_IMEM_RT_XCHECK` | `imem_req` 时 `{port_sel, imem_cmd}` 无 X | 178-181 |

---

## 3. 覆盖建议

| 场景 | 说明 |
|---|---|
| port1 命中 | 地址匹配 → TCM 端口 |
| port0 命中 | 地址不匹配 → 总线桥 |
| 端口切换 | 相邻事务命中不同端口，验证 `port_sel_r` |
| 错误响应 | 选中端口返回 `RDY_ER` |
| 背靠背 | `DATA` 状态连续 OKAY |

---

## 4. 验证结论

路由器覆盖地址判定、端口分发与单 outstanding 的两态时序；`port_sel_r` 保证响应阶段
端口一致性。
