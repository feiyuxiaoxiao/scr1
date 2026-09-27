# SCR1 Tracelog 指令追踪记录 设计规格（Design Specification）

- 模块：`scr1_tracelog`（`src/core/pipeline/scr1_tracelog.sv`，453 行）
- 条件：整个模块位于 `` `ifdef SCR1_TRGT_SIMULATION `` 内（`:10`、`:453`），仅在仿真目标编译；
  内部数据通路再受 `` `ifdef SCR1_TRACE_LOG_EN `` 保护（`:15`、`:66`…`:449`）
- 层次：`scr1_pipe_top` 例化（`scr1_pipe_top.sv:780-821`），例化名 `i_tracelog`
- 职责：仿真期把流水线的 PC/指令流的每一次“更新事件”以及伴随的寄存器写、
  trap/中断/唤醒事件与 CSR 快照写入 `tracelog_core_<hart>.log`，供离线比对

> 说明：本模块是**纯仿真观测逻辑**，无综合意义；`SCR1_TRACE_LOG_EN` 在
> `scr1_arch_description.svh:206` 默认被注释关闭（该行使 `define` 处于注释状态）。

---

## 1. 概述与职责

Tracelog 是被动观测模块，不驱动任何流水线信号。它做三件事：

1. **维护文件名与文件句柄**：`tracelog_core_<mhartid>.log`（`:271`、`:145`）；
2. **采样流水线更新事件**：由 `exu2trace_update_pc_en_i` 或 `mprf2trace_wr_en_i`
   任一有效触发一次采样（`:313`）；
3. **格式化输出**：每个采样拍打印一行 Curr_PC / Instr / Next_PC / Reg / Value，
   事件发生时追加 CSR 快照（`:156-205`、`:370-417`）。

### 关键设计点

- **PC 流水**：`trace_pc <= trace_npc <= exu2trace_update_pc_i`（`:337-338`），
  因此“Curr_PC”相对当前更新晚一拍，是上一次事件的 Next_PC；
- **事件类型**：N/E/I/W（无事件/异常/中断/唤醒），优先级 异常 > 中断 > 唤醒
  （`:341-356`）；
- **CSR 快照**：由组合逻辑从离散输入拼装成 7 个 CSR 的完整值（`:423-447`），
  仅在事件发生的那一拍有意义地打印。

---

## 2. 端口（`:12-61`）

仅 `rst_n`/`clk` 常驻；其余端口在 `` `ifdef SCR1_TRACE_LOG_EN `` 内。

| 信号 | 方向 | 行号 | 说明 |
|---|---|---|---|
| `rst_n` | in | 13 | 复位 |
| `clk` | in | 14 | 时钟 |
| `soc2pipe_fuse_mhartid_i[XLEN-1:0]` | in | 17 | 用于构造日志文件名 |
| `mprf2trace_int_i[1:MPRF_SIZE-1]` | in | 21/23 | MPRF 寄存器堆内容（RAM/分布式两种类型） |
| `mprf2trace_wr_en_i` | in | 25 | MPRF 写使能 |
| `mprf2trace_wr_addr_i[MPRF_AWIDTH-1:0]` | in | 26 | MPRF 写地址 |
| `mprf2trace_wr_data_i[XLEN-1:0]` | in | 27 | MPRF 写数据 |
| `exu2trace_update_pc_en_i` | in | 30 | PC 更新标志（采样触发源之一） |
| `exu2trace_update_pc_i[XLEN-1:0]` | in | 31 | 下一个 PC 值 |
| `ifu2trace_instr_i[IMEM_DWIDTH-1:0]` | in | 34 | 当前 IFU 指令 |
| `csr2trace_mstatus_mie_i` / `_mpie_i` | in | 37/38 | MSTATUS 位 |
| `csr2trace_mtvec_base_i[XLEN-1:6]` / `_mode_i` | in | 39/40 | MTVEC 基址与模式 |
| `csr2trace_mie_meie_i/mtie_i/msie_i` | in | 41-43 | MIE 位 |
| `csr2trace_mip_meip_i/mtip_i/msip_i` | in | 44-46 | MIP 位 |
| `csr2trace_mepc_i` | in | 48/50 | MEPC（RVC 时 `[XLEN-1:1]`，否则 `[XLEN-1:2]`） |
| `csr2trace_mcause_irq_i` | in | 52 | MCAUSE 中断位 |
| `csr2trace_mcause_ec_i`（`type_scr1_exc_code_e`） | in | 53 | MCAUSE 异常码 |
| `csr2trace_mtval_i[XLEN-1:0]` | in | 54 | MTVAL |
| `csr2trace_e_exc_i` | in | 57 | 异常事件 |
| `csr2trace_e_irq_i` | in | 58 | 中断事件 |
| `pipe2trace_e_wake_i` | in | 59 | 流水线唤醒事件 |

端口连线（父层 `scr1_pipe_top.sv:780-821`）全部采用对子模块内部信号的**层次引用**，
例如 `i_pipe_mprf.mprf_int`（`:788`）、`i_pipe_exu.update_pc_en`（`:794`）、
`i_pipe_csr.csr_mstatus_mie_ff`（`:801`）。

---

## 3. 局部类型（`:66-113`）

### 3.1 `type_scr1_ireg_name_s`（`:67-102`）

镜像 ABI 寄存器名的结构体，字段 `INT_00_ZERO`…`INT_15_A5`（`:68-83`），
非 RVE 时再扩展 `INT_16_A6`…`INT_31_T6`（`:84-101`，受 `` `ifndef SCR1_RVE_EXT ``）。
用于 `$fwrite` 时按名打印寄存器。

### 3.2 `type_scr1_csr_trace_s`（`:104-112`）

`packed struct`，打包 7 个 CSR 快照：`mstatus`/`mtvec`/`mie`/`mip`/`mepc`/`mcause`/`mtval`。

---

## 4. 局部信号（`:118-148`）

| 信号 | 类型 | 行号 | 说明 |
|---|---|---|---|
| `mprf_int_alias` | `type_scr1_ireg_name_s` | 120 | MPRF 内容按名别名 |
| `current_time` | `time` | 123 | 采样时刻（Verilator 用 `time_cnt`） |
| `trace_flag` | `logic` | 126 | 常 1（`:312`） |
| `trace_update` | `logic` | 127 | 采样触发 |
| `trace_update_r` | `logic` | 128 | 采样触发打一拍 |
| `event_type` | `byte` | 129 | "N"/"E"/"I"/"W" |
| `trace_pc` / `trace_npc` | `logic[XLEN-1:0]` | 131/132 | Curr/Next PC |
| `trace_instr` | `logic[IMEM_DWIDTH-1:0]` | 133 | 当前指令 |
| `csr_trace1` | `type_scr1_csr_trace_s` | 135 | CSR 快照 |
| `trace_fhandler_core` | `int unsigned` | 138 | 文件句柄 |
| `mprf_up` / `mprf_addr` / `mprf_wdata` | — | 141-143 | 寄存器写流水 |
| `hart` / `test_name` | `string` | 145/146 | 文件名后缀 / 测试名（`test_name` 由 TB 层次写） |

> `test_name` 在本模块内只读，实际由 `scr1_top_tb_runtests.sv:175` 通过
> `...i_tracelog.test_name = test_file` 赋值。

---

## 5. 局部任务（`:156-205`）

### 5.1 `trace_write_common`（`:156-163`）

输出公共前缀：`current_time`、`event_type`、`trace_pc`、`trace_instr`、`trace_npc`。
等价于“换行后重写一行行首”。

### 5.2 `trace_write_int_walias`（`:165-205`）

按 `mprf_addr` 用 case 打印寄存器别名（`x00_zero`…`x31_t6`），
非 RVE 时含 16-31（`:183-200`）；`default` 打印 `xxx`（`:201-203`）。

---

## 6. MPRF 别名赋值（`:210-243`）

`mprf_int_alias.INT_00_ZERO = '0`（`:210`，x0 恒零），
其余 `INT_01_RA`…`INT_31_T6` 直接绑定 `mprf2trace_int_i[n]`（`:211-242`）。
非 RVE 部分受 `` `ifndef SCR1_RVE_EXT ``（`:226`）。

---

## 7. 遗留时间计数器（`:251-259`）

```
time_cnt: 复位为 0，否则每拍 +1
```
注释 `:249` 明确其为“兼容当前 UVM 环境的遗留计数器”。Verilator 下
`current_time` 取 `time_cnt`（`:333`），其它仿真器取 `$time()`（`:335`）。

---

## 8. 文件打开与头部（`:267-306`）

### 8.1 initial（`:267-284`）

1. `$timeformat(-9,0," ns",10)` 设定时间打印格式（`:268`）；
2. `#1 hart.hextoa(soc2pipe_fuse_mhartid_i)` 生成 hart 字符串（`:269`）；
3. 打开 `tracelog_core_<hart>.log`（`:271`）；
4. 写入 RTL_ID 与事件图例（N/E/I/W，`:274-283`）。

### 8.2 复位头部（`:287-306`）

`always @(posedge rst_n)`，复位上升沿打印分隔线、`# Test: <test_name>`（`:295`）
与列头 `Time / Ev / Curr_PC / Instr / Next_PC / Reg / Value`（`:296-303`）。
非 Verilator 时打印 `$time()`（`:290`），Verilator 下省略时间（`:292`）。

---

## 9. 采样触发与时序（`:312-364`）

```
trace_flag   = 1'b1                                        (:312)
trace_update = (exu2trace_update_pc_en_i | mprf2trace_wr_en_i) & trace_flag   (:313)
```

`always_ff @(posedge clk)`（`:315-364`）：

- 复位：`current_time<=0`、`event_type<="N"`、PC/指令取 `'x`、寄存器流水清零（`:316-328`）；
- `trace_update_r <= trace_update`（`:330`）——输出 always 块用它作为打印使能；
- 当 `trace_update`：
  - `current_time` 取 `time_cnt`（Verilator）或 `$time()`（`:332-336`）；
  - `trace_pc <= trace_npc`；`trace_npc <= exu2trace_update_pc_i`；
    `trace_instr <= ifu2trace_instr_i`（`:337-339`）；
  - 事件类型选择（`:341-356`）：`e_exc`→"E"，否则 `e_irq`→"I"，否则 `e_wake`→"W"，否则 "N"；
- 寄存器写流水无条件更新（`:360-362`）：`mprf_up<=wr_en`，
  `mprf_addr/wdata` 在无写时取 `'x`。

> `trace_update_r` 与输出块配合：采样拍更新数据，下一拍（`trace_update_r` 高）打印。

---

## 10. 日志输出（`:370-417`）

`always_ff @(negedge rst_n, posedge clk)`，当 `trace_update_r` 时：

1. 先 `trace_write_common()` 打印公共列（`:375`）；
2. 按 `event_type` 分支（`:377-413`）：
   - **"W"**：若 `csr_trace1.mip & csr_trace1.mie` 非零，打印 `mip`、`mie`（`:378-385`）；
   - **"N"**：`mprf_up && mprf_addr!=0` 时打印别名 + 写数据，否则打印 `--- --------`（`:386-395`）；
   - **"R"**：打印 `mstatus`（`:396-399`，**死分支**，见 §12）；
   - **"E"/"I"**：逐行打印 `mstatus`、`mepc`、`mcause`、`mtval`（`:400-409`）；
   - default：仅换行（`:410-412`）；
3. 末尾统一换行（`:414`）。

---

## 11. CSR 快照组合逻辑（`:423-447`）

| 字段 | 组装 | 行号 |
|---|---|---|
| `mtvec` | `{base[XLEN-1:6], 4'd0, 2'(mode)}` | 424 |
| `mepc` | RVC: `{mepc_i,1'b0}`；否则 `{mepc_i,2'b00}` | 425-430 |
| `mcause` | `{irq, type_scr1_csr_mcause_ec_v'(ec)}` | 431 |
| `mtval` | 直连 | 432 |
| `mstatus` | 先清零，再置 MIE(bit3)、MPIE(bit7)、MPP=`2'b11` | 434-440 |
| `mie` | 先清零，再置 MSIE(bit3)/MTIE(bit7)/MEIE(bit11) | 435/441-443 |
| `mip` | 先清零，再置 MSIP/MTIP/MEIP 于同三位 | 436/444-446 |

常量来源：`SCR1_CSR_MSTATUS_MPP=2'b11`（`scr1_csr.svh:142`）、
`MIE_OFFSET=3`（`:143`）、`MPIE_OFFSET=7`（`:144`）、`MSIE_OFFSET=3`（`:157`）。

---

## 12. 配置宏与死代码

| 宏 | 作用 | 位置 |
|---|---|---|
| `SCR1_TRGT_SIMULATION` | 整个模块是否存在的总开关 | `:10` / `:453` |
| `SCR1_TRACE_LOG_EN` | 内部观测逻辑开关，默认注释关闭 | `scr1_arch_description.svh:206` |
| `SCR1_RVE_EXT` | 决定 x16-x31 是否参与 | `:84`/`:183`/`:226` |
| `SCR1_RVC_EXT` | 决定 MEPC 低位对齐 | `:47-51`/`:426-430` |
| `SCR1_MPRF_RAM` | 决定 MPRF 输入类型 | `:20-24` |
| `VERILATOR` | 时间源与复位头部格式 | `:289-292`/`:332-336` |

**死分支**：`:396-399` 的 `"R"`（MRET）分支不可达——`event_type` 仅在
`:318`（"N"）、`:343`（"E"）、`:347`（"I"）、`:351`（"W"）、`:355`（"N"）
被赋值，从不为 `"R"`。属遗留代码。

---

## 13. 附录

### 13.1 端口清单
见 §2。

### 13.2 局部信号清单
见 §4。

### 13.3 输出格式与信号映射

| 日志列 | 来源 | 打印处 |
|---|---|---|
| Time | `current_time` | :157 |
| Ev | `event_type` | :159 |
| Curr_PC | `trace_pc` | :160 |
| Instr | `trace_instr` | :161 |
| Next_PC | `trace_npc` | :162 |
| Reg | `trace_write_int_walias` | :390 |
| Value | `mprf_wdata` | :391 |
| 事件附加行 | `csr_trace1.*` | :381-408 |

### 13.4 职责边界

- 只读观测，不影响流水线行为；
- 依赖父层层次引用取内部信号（非端口解耦），故与 `scr1_pipe_top` 内部命名强耦合；
- 文件名由 hart id 决定，多 hart 会分别生成独立日志。
