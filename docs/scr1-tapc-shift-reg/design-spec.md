# SCR1 TAPC 移位寄存器设计规格（Design Specification）

- 模块：`scr1_tapc_shift_reg`（`src/core/scr1_tapc_shift_reg.sv`，112 行；`SCR1_DBG_EN` 保护 `:8`/`:112`）
- 职责：JTAG TAPC 的参数化移位寄存器（DR 链路）

---

## 1. 概述与职责

通用移位寄存器，支持并行捕获与串行移位，被 TAPC 用于 IDCODE/BLD_ID/BYPASS 等数据寄存器：

- **捕获**：`fsm_dr_select & fsm_dr_capture` 时 `shift_reg <= din_parallel`；
- **移位**：`fsm_dr_select & fsm_dr_shift` 时 `{din_serial, shift_reg[WIDTH-1:1]}`（MSB 进，LSB 出）；
- **复位**：异步 `rst_n` 或同步 `rst_n_sync` 均装载 `SCR1_RESET_VALUE`。

---

## 2. 参数与端口

### 2.1 参数（`:10-11`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `SCR1_WIDTH` | 8 | 寄存器位宽 |
| `SCR1_RESET_VALUE` | `'0` | 复位值 |

### 2.2 端口（`:12-26`）

`clk`/`rst_n`/`rst_n_sync`；TAP FSM 控制 `fsm_dr_select/capture/shift`；
输入 `din_serial`/`din_parallel[WIDTH-1:0]`；输出 `dout_serial`（`shift_reg[0]`）、
`dout_parallel`（`shift_reg`）。

---

## 3. 实现（`:36-86`）

`generate` 分两支：

- `SCR1_WIDTH>1`（`:37-56`）：移位 `{din_serial, shift_reg[WIDTH-1:1]}`；
- `SCR1_WIDTH==1`（`:57-75`）：移位 `<= din_serial`。

优先级（两支一致）：异步复位 > 同步复位 > 捕获 > 移位。

---

## 4. 输出（`:78-86`）

`dout_parallel = shift_reg`（`:81`）、`dout_serial = shift_reg[0]`（`:86`）。

---

## 5. 断言（`:88-108`，`SCR1_TRGT_SIMULATION`）

`SCR1_SVA_TAPC_SHIFTREG_XCHECK`：检查 `rst_n_sync` 与 5 个控制/数据输入无 X（`:94-106`）。

---

## 6. 附录

### 6.1 配置开关

| 宏 | 影响 | 位置 |
|---|---|---|
| `SCR1_DBG_EN` | 模块是否存在 | `:8`/`:112` |
| `SCR1_TRGT_SIMULATION` | 使能 SVA | `:88-108` |

### 6.2 职责边界

- 无协议逻辑，仅移位/捕获；
- 控制的时序由 TAPC FSM 保证；
- 复位值参数化以适配不同 DR（IDCODE 等）。
