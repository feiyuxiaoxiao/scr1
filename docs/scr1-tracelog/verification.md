# SCR1 Tracelog 验证文档（Verification Document）

- 被测对象：`scr1_tracelog`（`src/core/pipeline/scr1_tracelog.sv`，453 行）
- 编译条件：模块整体在 `` `ifdef SCR1_TRGT_SIMULATION ``（`:10`/`:453`），
  观测逻辑另需 `` `ifdef SCR1_TRACE_LOG_EN ``
- 配套设计规格：`docs/scr1-tracelog/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:780-821`，例化名 `i_tracelog`）

> `SCR1_TRACE_LOG_EN` 默认注释关闭（`scr1_arch_description.svh:206`），
> 因此常规回归**不编译**观测逻辑；需要显式打开该宏后单独构建验证。

---

## 1. 验证目标与范围

### 1.1 目标

1. 验证日志文件按 hart id 正确命名与打开；
2. 验证复位头部与列头格式；
3. 验证采样触发条件（PC 更新或 MPRF 写）；
4. 验证 PC/指令/寄存器写流水的时序；
5. 验证事件类型优先级（E > I > W > N）；
6. 验证 CSR 快照组合组装正确（含 MEPC 对齐、MCAUSE 拼接）；
7. 验证异常/中断行的多行输出。

### 1.2 范围

- 纯仿真观测逻辑，不驱动 RTL 行为；
- 需在开启 `SCR1_TRACE_LOG_EN` 的构建下观察日志文件。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 |
|---|---|
| Verilator | 5.006-3 |
| riscv-none-elf-gcc | 15.2.0 |

### 2.2 观察点

- 文件产物：`tracelog_core_<hart>.log`（`:271`）；
- 层次引用 `i_top.i_core_top.i_pipe_top.i_tracelog.*`：
  `test_name`、`trace_update`、`trace_update_r`、`event_type`、
  `trace_pc`、`trace_npc`、`trace_instr`、`mprf_up`/`mprf_addr`/`mprf_wdata`、
  `csr_trace1.*`。

### 2.3 构建开关

| 宏 | 作用 | 设置方式 |
|---|---|---|
| `SCR1_TRGT_SIMULATION` | 模块存在的前提 | 仿真目标自带 |
| `SCR1_TRACE_LOG_EN` | 启用观测逻辑 | 构建宏（默认关） |
| `VERILATOR` | 时间源/头部格式 | 工具定义 |

---

## 3. 验证策略

1. **静态核对**：确认宏保护范围与例化连线（层次引用）一致；
2. **日志比对**：开启 `SCR1_TRACE_LOG_EN` 跑 `hello`，逐列核对格式与时序；
3. **波形交叉**：用 `trace_update`/`event_type`/CSR 快照与日志行对照；
4. **边界**：x0 写不打印寄存器、多事件同拍优先级、RVC 对齐。

---

## 4. 功能覆盖点（Functional Coverage Points）

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-TRACE-1 | 文件名按 hart id 生成 | 设置 MHARTID 后启动 |
| FC-TRACE-2 | 复位头部与列头打印 | 复位上升沿 |
| FC-TRACE-3 | `# Test: <name>` 写入 test_name | TB 层次赋值后复位 |
| FC-TRACE-4 | PC 更新触发采样 | `exu2trace_update_pc_en_i` |
| FC-TRACE-5 | MPRF 写触发采样 | `mprf2trace_wr_en_i` |
| FC-TRACE-6 | `trace_pc`/`trace_npc` 一拍流水 | 连续更新 |
| FC-TRACE-7 | `trace_instr` 采样 | IFU 指令随更新 |
| FC-TRACE-8 | 事件类型 "N" | 无事件更新 |
| FC-TRACE-9 | 事件类型 "E" | `csr2trace_e_exc_i` |
| FC-TRACE-10 | 事件类型 "I" | `csr2trace_e_irq_i` |
| FC-TRACE-11 | 事件类型 "W" | `pipe2trace_e_wake_i` |
| FC-TRACE-12 | 事件优先级 E>I | E/I 同拍 |
| FC-TRACE-13 | 事件优先级 I>W | I/W 同拍 |
| FC-TRACE-14 | 唤醒且 mip&mie 非零打印 mip/mie | W + 中断挂起 |
| FC-TRACE-15 | 普通行打印寄存器别名 + 值 | xN 写 |
| FC-TRACE-16 | x0 写不打印别名（`mprf_addr==0` 走 `---`） | 写 x0 |
| FC-TRACE-17 | 无写时打印 `--- --------` | 更新但无写 |
| FC-TRACE-18 | 异常/中断行打印 mstatus/mepc/mcause/mtval | E/I |
| FC-TRACE-19 | `mtvec` 重组（base<<6 \| mode） | 读快照 |
| FC-TRACE-20 | `mepc` RVC 对齐 | `SCR1_RVC_EXT` |
| FC-TRACE-21 | `mcause` 拼接 irq+ec | E/I 快照 |
| FC-TRACE-22 | `mstatus` MIE/MPIE/MPP=11 | 事件快照 |
| FC-TRACE-23 | `mie`/`mip` 三位映射 | 事件快照 |
| FC-TRACE-24 | 非 RVE 时 x16-x31 别名 | 配置构建 |
| FC-TRACE-25 | Verilator 下时间取 `time_cnt` | Verilator 构建 |
| FC-TRACE-26 | 关闭 `SCR1_TRACE_LOG_EN` 端口仅 rst/clk | 默认构建 |
| FC-TRACE-27 | `"R"`（MRET）分支不可达（死代码） | 代码核对 |

---

## 5. 断言验证（Assertions）

本模块**未内建 SVA**。观测正确性通过日志文件与波形交叉核对（§6.2）。

---

## 6. 已执行的验证活动与结果

### 6.1 静态核对

- 模块存在性由 `SCR1_TRGT_SIMULATION` 保护（`:10`/`:453`），符合“仅仿真”定位；
- 端口与父层例化连线一一对应（`scr1_pipe_top.sv:780-821`）；
- 发现死分支 `"R"`（`:396-399`），已记入设计规格 §12。

### 6.2 默认回归（`SCR1_TRACE_LOG_EN` 关闭）

`SCR1_CFG_RV32IMC_MAX` + `SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS` 构建，
观测逻辑不参与编译，`hello` PASS 说明：

- 模块壳（仅 `rst_n`/`clk`）不引入 X 或不期望的驱动；
- 未破坏流水线正常执行。

### 6.3 专项（建议执行）

| 场景 | 方法 | 预期 | FC |
|---|---|---|---|
| 打开观测 | 构建定义 `SCR1_TRACE_LOG_EN` | 生成 `tracelog_core_*.log` | 1-3 |
| 普通指令流 | 跑 `hello` | 每行 PC/Instr/Next_PC 连续 | 4-7,15 |
| 事件行 | 触发异常/中断/唤醒 | 事件列 E/I/W 且附加 CSR 行 | 8-14,18 |
| 无写更新 | 仅 PC 更新 | 打印 `--- --------` | 17 |
| x0 写 | 写 x0 | 不打印别名 | 16 |
| 对齐/拼接 | 读快照 | mtvec/mepc/mcause 正确 | 19-23 |

> 6.3 尚未系统执行；默认回归未覆盖观测逻辑。

---

## 7. 回归流程

默认构建（观测关闭）：

```
make -C sim build_verilator root_dir=/workspace bld_dir=<bld_dir> \
  BUS=AHB SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS \
  top_module=scr1_top_tb_ahb
```

启用观测比对时，把 `SIM_TRACE_DEF` 改为 `SCR1_TRACE_LOG_EN` 后重新构建，
跑 `hello` 并检查工作目录下的 `tracelog_core_<hart>.log`。

---

## 8. 验证结论

1. Tracelog 定位清晰：仅仿真、纯观测、默认不编译观测逻辑；
2. 设计规格所述采样时序、事件优先级、CSR 快照组装均与源码一致；
3. 存在 `"R"` 死分支（遗留代码），不影响功能；
4. 观测逻辑需显式开启 `SCR1_TRACE_LOG_EN` 才能验证，建议按 §6.3 建立日志比对流程。

**遗留建议**：建立开启 `SCR1_TRACE_LOG_EN` 的日志比对回归，覆盖 FC-TRACE-1~27。
