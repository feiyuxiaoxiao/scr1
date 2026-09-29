# SCR1 TAPC 同步器验证文档（Verification）

- 被测模块：`scr1_tapc_synchronizer`（`src/core/scr1_tapc_synchronizer.sv`，183 行）
- 验证目标：TCK/sys 跨域同步、沿脉冲产生、各输出锁存与复位域

---

## 1. 功能检查项（Feature Checklist）

| ID | 功能点 | 依据行号 | 检查方法 |
|---|---|---|---|
| FC-SYNC-1 | TCK 上升/下降沿分频 | `:68-82` | 波形观察 |
| FC-SYNC-2 | 4 级系统域同步 | `:84-92` | 同步链观察 |
| FC-SYNC-3 | rise/fall load/reset 脉冲 | `:94-97` | 脉冲宽度与单拍 |
| FC-SYNC-4 | `ch_update` 装载/清零 | `:99-109` | fall 沿触发 |
| FC-SYNC-5 | capture/shift 采样与同步 | `:111-129` | TCK→sys 传递 |
| FC-SYNC-6 | tdi 3 级同步 | `:131-137` | 数据同步 |
| FC-SYNC-7 | capture/shift/tdi 输出锁存 | `:139-155` | rise 沿触发 |
| FC-SYNC-8 | `dmi_ch_sel`/`ch_id` 锁存（`dm_rst_n`） | `:157-167` | 复位域检查 |
| FC-SYNC-9 | `scu_ch_sel` 锁存（`pwrup_rst_n`） | `:169-177` | 复位域检查 |
| FC-SYNC-10 | tdo 直通 | `:179` | 端口检查 |

---

## 2. 覆盖建议

| 场景 | 说明 |
|---|---|
| 单次扫描 | update 在 fall、capture/shift 在 rise |
| 连续扫描 | 多脉冲重复 |
| tck 快/慢 | 与 `clk` 频率比变化 |
| 复位 | `pwrup_rst_n`/`dm_rst_n` 分别复位 |
| tdo 回读 | 链输出直通 |

---

## 3. 断言

本模块无 SVA。

---

## 4. 验证结论

同步器覆盖 TCK↔sys 的控制/数据同步与沿脉冲语义；跨域正确性依赖同步链级数与复位域
划分的验证。
