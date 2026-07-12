# SCR1 LSU 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_lsu.sv`
> 模块: `scr1_pipe_lsu` — Load/Store Unit

---

## 1. 局部枚举类型 (typedef enum)

### 1.1 LSU FSM 状态 — `type_scr1_lsu_fsm_e`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_LSU_FSM_IDLE` | 1'b0 | 空闲，等待 EXU 发起 DMEM 请求 |
| `SCR1_LSU_FSM_BUSY` | 1'b1 | 忙碌，DMEM 请求已发出，等待响应 |

- 位宽: 1 bit (`enum logic`)
- 定义位置: `scr1_pipe_lsu.sv:67-70`

---

## 2. 从外部头文件引入的类型

### 2.1 LSU 命令 — `type_scr1_lsu_cmd_sel_e`（`scr1_riscv_isa_decoding.svh`）

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_LSU_CMD_NONE` | 4'd0 | 无操作 |
| `SCR1_LSU_CMD_LB` | 4'd1 | 字节加载（有符号扩展） |
| `SCR1_LSU_CMD_LH` | 4'd2 | 半字加载（有符号扩展） |
| `SCR1_LSU_CMD_LW` | 4'd3 | 字加载 |
| `SCR1_LSU_CMD_LBU` | 4'd4 | 字节加载（无符号扩展） |
| `SCR1_LSU_CMD_LHU` | 4'd5 | 半字加载（无符号扩展） |
| `SCR1_LSU_CMD_SB` | 4'd6 | 字节存储 |
| `SCR1_LSU_CMD_SH` | 4'd7 | 半字存储 |
| `SCR1_LSU_CMD_SW` | 4'd8 | 字存储 |

- 位宽: `$clog2(9)` = 4 bit
- 定义位置: `scr1_riscv_isa_decoding.svh:107-116`

### 2.2 异常代码 — `type_scr1_exc_code_e`（`scr1_arch_types.svh`）

| 枚举值 | 编码 | LSU 相关含义 |
|--------|:---:|------|
| `SCR1_EXC_CODE_BREAKPOINT` | 4'd3 | LSU 断点异常（TDU 触发） |
| `SCR1_EXC_CODE_LD_ADDR_MISALIGN` | 4'd4 | 加载地址未对齐 |
| `SCR1_EXC_CODE_LD_ACCESS_FAULT` | 4'd5 | 加载访问错误 |
| `SCR1_EXC_CODE_ST_ADDR_MISALIGN` | 4'd6 | 存储地址未对齐 |
| `SCR1_EXC_CODE_ST_ACCESS_FAULT` | 4'd7 | 存储访问错误 |
| `SCR1_EXC_CODE_INSTR_MISALIGN` | 4'd0 | （默认值/保护值，LSU 不应产出此码） |

- 位宽: 4 bit
- 定义位置: `scr1_arch_types.svh:41-51`

### 2.3 DMEM 命令 — `type_scr1_mem_cmd_e`（`scr1_memif.svh`）

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_MEM_CMD_RD` | 1'b0 | 存储器读 |
| `SCR1_MEM_CMD_WR` | 1'b1 | 存储器写 |

- 位宽: 1 bit (`enum logic`)
- 定义位置: `scr1_memif.svh:14-21`

### 2.4 DMEM 数据宽度 — `type_scr1_mem_width_e`（`scr1_memif.svh`）

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_MEM_WIDTH_BYTE` | 2'b00 | 字节宽度（8 位） |
| `SCR1_MEM_WIDTH_HWORD` | 2'b01 | 半字宽度（16 位） |
| `SCR1_MEM_WIDTH_WORD` | 2'b10 | 字宽度（32 位） |

- 位宽: 2 bit (`enum logic[1:0]`)
- 定义位置: `scr1_memif.svh:26-34`

### 2.5 DMEM 响应 — `type_scr1_mem_resp_e`（`scr1_memif.svh`）

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_MEM_RESP_NOTRDY` | 2'b00 | 未就绪 |
| `SCR1_MEM_RESP_RDY_OK` | 2'b01 | 就绪且成功 |
| `SCR1_MEM_RESP_RDY_ER` | 2'b10 | 就绪但出错 |

- 位宽: 2 bit (`enum logic[1:0]`)
- 定义位置: `scr1_memif.svh:39-47`

### 2.6 TDU 数据监视结构体 — `type_scr1_brkm_lsu_mon_s`（`scr1_tdu.svh`）

| 字段 | 类型 | 位宽 | 含义 |
|------|------|:---:|------|
| `vd` | logic | 1 | 数据地址流有效 |
| `load` | logic | 1 | 当前操作为加载 |
| `store` | logic | 1 | 当前操作为存储 |
| `addr` | logic | `SCR1_XLEN` | 数据地址 |

- 定义位置: `scr1_tdu.svh:112-117`

---

## 3. 架构参数（来自 `scr1_arch_description.svh`）

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_XLEN` | 32 | RISC-V 处理器通用寄存器位宽 |
| `SCR1_DMEM_AWIDTH` | `SCR1_XLEN` (32) | 数据存储器地址位宽 |
| `SCR1_DMEM_DWIDTH` | `SCR1_XLEN` (32) | 数据存储器数据位宽 |

---

## 4. 模块端口 (I/O)

### 4.1 通用控制信号

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `rst_n` | 1 | 异步复位，低有效 |
| input | `clk` | 1 | 时钟 |

### 4.2 LSU ↔ EXU 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `exu2lsu_req_i` | 1 | EXU 向 LSU 发起访存请求 |
| input | `exu2lsu_cmd_i` | `type_scr1_lsu_cmd_sel_e` (4) | LSU 命令（LB/LH/LW/LBU/LHU/SB/SH/SW） |
| input | `exu2lsu_addr_i` | `SCR1_XLEN` (32) | DMEM 访问地址 |
| input | `exu2lsu_sdata_i` | `SCR1_XLEN` (32) | 存储数据 |
| output | `lsu2exu_rdy_o` | 1 | LSU 已收到 DMEM 响应（操作完成） |
| output | `lsu2exu_ldata_o` | `SCR1_XLEN` (32) | 加载数据（已扩展） |
| output | `lsu2exu_exc_o` | 1 | LSU 异常请求 |
| output | `lsu2exu_exc_code_o` | `type_scr1_exc_code_e` (4) | 异常代码 |

### 4.3 LSU ↔ DMEM 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `lsu2dmem_req_o` | 1 | 数据存储器请求 |
| output | `lsu2dmem_cmd_o` | `type_scr1_mem_cmd_e` (1) | 存储器命令（RD/WR） |
| output | `lsu2dmem_width_o` | `type_scr1_mem_width_e` (2) | 数据宽度（BYTE/HWORD/WORD） |
| output | `lsu2dmem_addr_o` | `SCR1_DMEM_AWIDTH` (32) | 存储器地址 |
| output | `lsu2dmem_wdata_o` | `SCR1_DMEM_DWIDTH` (32) | 写入数据 |
| input | `dmem2lsu_req_ack_i` | 1 | DMEM 确认接收请求 |
| input | `dmem2lsu_rdata_i` | `SCR1_DMEM_DWIDTH` (32) | DMEM 读取数据 |
| input | `dmem2lsu_resp_i` | `type_scr1_mem_resp_e` (2) | DMEM 响应类型 |

### 4.4 LSU ↔ TDU 接口（仅 `SCR1_TDU_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `lsu2tdu_dmon_o` | `type_scr1_brkm_lsu_mon_s` | 数据地址流监视信息 |
| input | `tdu2lsu_ibrkpt_exc_req_i` | 1 | 指令断点异常请求 |
| input | `tdu2lsu_dbrkpt_exc_req_i` | 1 | 数据断点异常请求 |

---

## 5. 内部信号 — LSU FSM 模块

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `lsu_fsm_curr` | reg (`type_scr1_lsu_fsm_e`) | 1 | FSM 当前状态寄存器 |
| `lsu_fsm_next` | wire (`type_scr1_lsu_fsm_e`) | 1 | FSM 下一状态（组合逻辑） |
| `lsu_fsm_idle` | wire | 1 | FSM 处于 IDLE 状态 |

---

## 6. 内部信号 — 命令寄存器模块

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `lsu_cmd_upd` | wire | 1 | 命令寄存器更新使能（`fsm_idle & dmem_req_vd`） |
| `lsu_cmd_ff` | reg (`type_scr1_lsu_cmd_sel_e`) | 4 | 命令寄存器值（锁存的 LSU 命令） |
| `lsu_cmd_ff_load` | wire | 1 | 锁存的命令是加载操作 |
| `lsu_cmd_ff_store` | wire | 1 | 锁存的命令是存储操作 |

---

## 7. 内部信号 — DMEM 命令与数据宽度译码

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `dmem_cmd_load` | wire | 1 | 当前 EXU 命令是加载（LB/LBU/LH/LHU/LW） |
| `dmem_cmd_store` | wire | 1 | 当前 EXU 命令是存储（SB/SH/SW） |
| `dmem_wdth_word` | wire | 1 | 数据宽度为字（LW/SW） |
| `dmem_wdth_hword` | wire | 1 | 数据宽度为半字（LH/LHU/SH） |
| `dmem_wdth_byte` | wire | 1 | 数据宽度为字节（LB/LBU/SB） |

---

## 8. 内部信号 — DMEM 响应与控制模块

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `dmem_resp_ok` | wire | 1 | DMEM 响应为 RDY_OK（成功） |
| `dmem_resp_er` | wire | 1 | DMEM 响应为 RDY_ER（错误） |
| `dmem_resp_received` | wire | 1 | 收到 DMEM 响应（OK 或 ER） |
| `dmem_req_vd` | wire | 1 | DMEM 请求有效（EXU 请求 & DMEM 确认 & 无异常） |

---

## 9. 内部信号 — 异常检测模块

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `lsu_exc_req` | wire | 1 | LSU 异常请求（未对齐或断点的 OR） |
| `dmem_addr_mslgn` | wire | 1 | 地址未对齐（加载或存储，通用信号） |
| `dmem_addr_mslgn_l` | wire | 1 | 加载地址未对齐 |
| `dmem_addr_mslgn_s` | wire | 1 | 存储地址未对齐 |
| `lsu_exc_hwbrk` | wire | 1 | LSU 硬件断点异常（仅 `SCR1_TDU_EN`） |

---

## 10. 断言信号概要（仅 `SCR1_TRGT_SIMULATION`）

| 断言名 | 类型 | 检查内容 |
|--------|------|------|
| `SCR1_SVA_LSU_XCHECK_CTRL` | X 态检查 | 控制信号不能为未知值 |
| `SCR1_SVA_LSU_XCHECK_CMD` | X 态检查 | 请求有效时命令和地址不能为未知值 |
| `SCR1_SVA_LSU_XCHECK_SDATA` | X 态检查 | 写请求时存储数据不能为未知值 |
| `SCR1_SVA_LSU_XCHECK_EXC` | X 态检查 | 异常输出时异常代码不能为未知值 |
| `SCR1_SVA_LSU_IMEM_CTRL` | X 态检查 | DMEM 请求时控制信号不能为未知值 |
| `SCR1_SVA_LSU_IMEM_ACK` | X 态检查 | DMEM 请求时确认信号不能为未知值 |
| `SCR1_SVA_LSU_IMEM_WDATA` | X 态检查 | DMEM 写请求时写数据 `[8:0]` 不能为未知值 |
| `SCR1_SVA_LSU_EXC_ONEHOT` | 行为检查 | 异常原因互斥——不可同时出现多种异常 |
| `SCR1_SVA_LSU_UNEXPECTED_DMEM_RESP` | 行为检查 | IDLE 状态不应收到 DMEM 响应 |
| `SCR1_SVA_LSU_REQ_EXC` | 行为检查 | 异常输出时必须伴随 EXU 请求 |
| `SCR1_COV_LSU_MISALIGN_BRKPT` | 覆盖率 | 地址未对齐与断点同时触发的场景 |

---

## 11. 外部依赖头文件

| 文件名 | 提供内容 |
|--------|------|
| `scr1_arch_description.svh` | `SCR1_XLEN`、`SCR1_DMEM_AWIDTH`、`SCR1_DMEM_DWIDTH` 等架构宏 |
| `scr1_arch_types.svh` | `type_scr1_exc_code_e` 异常编码枚举 |
| `scr1_memif.svh` | `type_scr1_mem_cmd_e`（RD/WR）、`type_scr1_mem_width_e`（BYTE/HWORD/WORD）、`type_scr1_mem_resp_e`（NOTRDY/RDY_OK/RDY_ER） |
| `scr1_riscv_isa_decoding.svh` | `type_scr1_lsu_cmd_sel_e`（LB/LH/LW/LBU/LHU/SB/SH/SW） |
| `scr1_tdu.svh` | `type_scr1_brkm_lsu_mon_s`（仅 `SCR1_TDU_EN`） |
