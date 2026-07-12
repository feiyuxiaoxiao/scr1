# SCR1 IFU 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_ifu.sv`
> 模块: `scr1_pipe_ifu` — Instruction Fetch Unit

---

## 1. 局部参数 (localparam)

### 1.1 队列容量参数

| 参数名 | 值 | 位宽 | 含义 |
|--------|:---:|------|------|
| `SCR1_IFU_Q_SIZE_WORD` | 2 | — | 指令队列以字（32位）计的容量：2 个字 |
| `SCR1_IFU_Q_SIZE_HALF` | 4 | — | 指令队列以半字（16位）计的容量：4 个半字。等于 `Q_SIZE_WORD × 2` |

### 1.2 队列地址与指针位宽

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_IFU_QUEUE_ADR_W` | `$clog2(4)` = 2 | 队列物理地址位宽，用于索引 4 个半字槽 |
| `SCR1_IFU_QUEUE_PTR_W` | `2+1` = 3 | 队列读写指针位宽，比地址多 1 位用于区分空/满（环形队列圈数标记） |
| `SCR1_IFU_Q_FREE_H_W` | `$clog2(4+1)` = `$clog2(5)` = 3 | 队列空闲半字计数器位宽，能表示 0~4 共 5 个状态 |
| `SCR1_IFU_Q_FREE_W_W` | `$clog2(2+1)` = `$clog2(3)` = 2 | 队列空闲字计数器位宽，能表示 0~2 共 3 个状态 |

### 1.3 IMEM 事务跟踪参数

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_TXN_CNT_W` | 3 | 在途事务计数器位宽，最大值 7，决定流水线中最多可同时存在的未完成 IMEM 请求数 |

---

## 2. 枚举类型 (typedef enum)

### 2.1 IFU FSM 状态 — `type_scr1_ifu_fsm_e`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_IFU_FSM_IDLE` | 1'b0 | 空闲状态，不取指 |
| `SCR1_IFU_FSM_FETCH` | 1'b1 | 取指状态，持续发送 IMEM 请求 |

- 位宽: 1 bit (`enum logic`)

### 2.2 队列写入模式 — `type_scr1_ifu_queue_wr_e`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_IFU_QUEUE_WR_NONE` | 2'b00 | 不写入队列（丢弃或旁路） |
| `SCR1_IFU_QUEUE_WR_FULL` | 2'b01 | 写入 32 位（两个半字槽），完整 RVI 或两 RVC |
| `SCR1_IFU_QUEUE_WR_HI` | 2'b10 | 仅写入高 16 位（一个半字槽），PC 未对齐场景 |

- 位宽: 2 bit (`enum logic[1:0]`)

### 2.3 队列读取模式 — `type_scr1_ifu_queue_rd_e`

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_IFU_QUEUE_RD_NONE` | 2'b00 | 不读取队列（下游未就绪） |
| `SCR1_IFU_QUEUE_RD_HWORD` | 2'b01 | 读取 1 个半字（队首是 RVC 或错误标志置位） |
| `SCR1_IFU_QUEUE_RD_WORD` | 2'b10 | 读取 1 个字（队首是 RVI 头部） |

- 位宽: 2 bit (`enum logic[1:0]`)

### 2.4 指令旁路类型 — `type_scr1_bypass_e`（仅 `SCR1_NO_DEC_STAGE`）

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_BYPASS_NONE` | 2'b00 | 不旁路，走正常队列路径 |
| `SCR1_BYPASS_RVC` | 2'b01 | 旁路 RVC 指令（16 位直接送出） |
| `SCR1_BYPASS_RVI_RDATA_QUEUE` | 2'b10 | 旁路跨边界 RVI：IMEM 低16位 + 队首16位拼接 |
| `SCR1_BYPASS_RVI_RDATA` | 2'b11 | 旁路完整 RVI 指令（32 位直接送出） |

- 位宽: 2 bit (`enum logic[1:0]`)

### 2.5 IMEM 返回字中的指令分布类型 — `type_scr1_ifu_instr_e`

命名规范: `SCR1_IFU_INSTR_<高16位类型>_<低16位类型>`（`HI` = RVI 高半即 funct/imm 部分，`LO` = RVI 低半含 `[1:0]=11`）

| 枚举值 | 编码 | 高16位 `[31:16]` | 低16位 `[15:0]` | 触发条件 |
|--------|:---:|------------------|-----------------|----------|
| `SCR1_IFU_INSTR_NONE` | 3'b000 | — | — | 无有效指令（响应被丢弃或错误） |
| `SCR1_IFU_INSTR_RVI_HI_RVI_LO` | 3'b001 | RVI 高半 | RVI 低半 | 完整对齐的 RVI 指令 |
| `SCR1_IFU_INSTR_RVC_RVC` | 3'b010 | RVC | RVC | 两个独立压缩指令 |
| `SCR1_IFU_INSTR_RVI_LO_RVC` | 3'b011 | RVI 低半 | RVC | 高16位是下一个 RVI 的前半截 |
| `SCR1_IFU_INSTR_RVC_RVI_HI` | 3'b100 | RVC | RVI 高半 | 低16位是上一个 RVI 的后半截 |
| `SCR1_IFU_INSTR_RVI_LO_RVI_HI` | 3'b101 | RVI 低半 | RVI 高半 | 两个 RVI 指令的头尾拼接（跨字边界） |
| `SCR1_IFU_INSTR_RVC_NV` | 3'b110 | RVC | 无效 | PC[1]=1 时仅高16位有效的 RVC |
| `SCR1_IFU_INSTR_RVI_LO_NV` | 3'b111 | RVI 低半 | 无效 | PC[1]=1 时仅高16位有效的 RVI 半截 |

- 位宽: 3 bit (`enum logic[2:0]`)

---

## 3. 模块端口 (I/O)

### 3.1 通用控制信号

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `rst_n` | 1 | 异步复位，低有效 |
| input | `clk` | 1 | 时钟 |
| input | `pipe2ifu_stop_fetch_i` | 1 | 流水线暂停取指请求 |

### 3.2 IFU ↔ IMEM 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `imem2ifu_req_ack_i` | 1 | IMEM 已确认接收 IFU 请求 |
| output | `ifu2imem_req_o` | 1 | IFU 向 IMEM 发起读取请求 |
| output | `ifu2imem_cmd_o` | `type_scr1_mem_cmd_e` | 存储器命令类型（固定为 READ） |
| output | `ifu2imem_addr_o` | `SCR1_IMEM_AWIDTH` | 存储器读地址（字地址，低2位为0） |
| input | `imem2ifu_rdata_i` | `SCR1_IMEM_DWIDTH` | 存储器返回的读取数据 |
| input | `imem2ifu_resp_i` | `type_scr1_mem_resp_e` | 存储器响应类型（OK / ERROR） |

### 3.3 IFU ↔ EXU 新 PC 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `exu2ifu_pc_new_req_i` | 1 | 执行单元请求新 PC（跳转/分支/陷阱） |
| input | `exu2ifu_pc_new_i` | `SCR1_XLEN` | 新 PC 值 |

### 3.4 IFU ↔ HDU 调试接口（仅 `SCR1_DBG_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `hdu2ifu_pbuf_fetch_i` | 1 | 使用 Program Buffer 提供的指令 |
| output | `ifu2hdu_pbuf_rdy_o` | 1 | IFU 准备好接受 Program Buffer 指令 |
| input | `hdu2ifu_pbuf_vd_i` | 1 | Program Buffer 指令有效 |
| input | `hdu2ifu_pbuf_err_i` | 1 | Program Buffer 指令错误 |
| input | `hdu2ifu_pbuf_instr_i` | `SCR1_HDU_CORE_INSTR_WIDTH` | Program Buffer 注入的指令 |

### 3.5 时钟门控接口（仅 `SCR1_CLKCTRL_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| output | `ifu2pipe_imem_txns_pnd_o` | 1 | 存在未完成的 IMEM 事务（时钟不能关） |

### 3.6 IFU ↔ IDU 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|------|------|
| input | `idu2ifu_rdy_i` | 1 | IDU 准备好接收新指令（反压信号） |
| output | `ifu2idu_instr_o` | `SCR1_IMEM_DWIDTH` | 送给 IDU 的指令（32位，RVC 时高16位填0） |
| output | `ifu2idu_imem_err_o` | 1 | 指令访问异常 |
| output | `ifu2idu_err_rvi_hi_o` | 1 | 取 RVI 高半时 IMEM 出错（跨边界场景） |
| output | `ifu2idu_vd_o` | 1 | IFU 输出指令有效 |

---

## 4. 内部信号 — 指令队列模块

### 4.1 新 PC 未对齐标志

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `new_pc_unaligned_ff` | reg | 1 | 当前取指地址的 bit[1] 是否为 1（仅高16位有效） |
| `new_pc_unaligned_next` | wire | 1 | 下一时钟周期的未对齐标志值 |
| `new_pc_unaligned_upd` | wire | 1 | 未对齐标志寄存器更新使能 |

### 4.2 指令类型译码

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `instr_hi_is_rvi` | wire | 1 | IMEM 返回字的高 16 位 `[17:16]` 是否全 1（即为 RVI 低半） |
| `instr_lo_is_rvi` | wire | 1 | IMEM 返回字的低 16 位 `[1:0]` 是否全 1（即为 RVI 低半） |
| `instr_type` | wire | `type_scr1_ifu_instr_e` | 当前 IMEM 返回字中的指令分布类型 |
| `instr_hi_rvi_lo_ff` | reg | 1 | 上一轮 IMEM 返回的高 16 位是 RVI 低半（跨边界拼接状态记忆） |
| `instr_hi_rvi_lo_next` | wire | 1 | 下一时钟周期的跨边界状态值 |

### 4.3 队列读控制

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `q_rd_size` | wire | `type_scr1_ifu_queue_rd_e` | 队列读取粒度（NONE/HWORD/WORD） |
| `q_rd_vd` | wire | 1 | 本次需要读取队列（队列非空 & 输出有效 & IDU 就绪） |
| `q_rd_none` | wire | 1 | 本次不读取队列 |
| `q_rd_hword` | wire | 1 | 本次读取 1 个半字（队首 RVC / 错误 / 旁路 RVI） |

### 4.4 队列写控制

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `q_wr_size` | wire | `type_scr1_ifu_queue_wr_e` | 队列写入粒度（NONE/HI/FULL） |
| `q_wr_none` | wire | 1 | 本次不写入队列 |
| `q_wr_full` | wire | 1 | 本次写入完整 32 位 |
| `q_wr_en` | wire | 1 | 队列写入使能（响应有效 & 未 flush） |
| `q_flush_req` | wire | 1 | 队列清空请求（新 PC 或停止取指） |

### 4.5 队列读写指针

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `q_rptr` | reg | `SCR1_IFU_QUEUE_PTR_W` | 读指针寄存器（队首位置） |
| `q_rptr_next` | wire | `SCR1_IFU_QUEUE_PTR_W` | 下一周期的读指针值 |
| `q_rptr_upd` | wire | 1 | 读指针更新使能 |
| `q_wptr` | reg | `SCR1_IFU_QUEUE_PTR_W` | 写指针寄存器（队尾位置） |
| `q_wptr_next` | wire | `SCR1_IFU_QUEUE_PTR_W` | 下一周期的写指针值 |
| `q_wptr_upd` | wire | 1 | 写指针更新使能 |

### 4.6 队列存储

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `q_data[0..3]` | reg array | 16 bit × 4 | 4 个半字槽的指令数据 |
| `q_err[0..3]` | reg array | 1 bit × 4 | 4 个半字槽的错误标志位 |
| `q_data_head` | wire | 16 bit | 队首半字数据（读指针指向的槽） |
| `q_data_next` | wire | 16 bit | 队首下一个半字数据（RVI 的高半部分） |
| `q_err_head` | wire | 1 bit | 队首半字错误标志 |
| `q_err_next` | wire | 1 bit | 队首下一个半字错误标志 |

### 4.7 队列状态

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `q_is_empty` | wire | 1 | 队列空（读写指针相等） |
| `q_has_free_slots` | wire | 1 | 队列剩余字空间 > 有效在途事务数（可以发新请求） |
| `q_has_1_ocpd_hw` | wire | 1 | 队列恰好占用 1 个半字 |
| `q_head_is_rvc` | wire | 1 | 队首是指令的 `[1:0]` ≠ 11，即为 RVC 指令 |
| `q_head_is_rvi` | wire | 1 | 队首是指令的 `[1:0]` = 11，即为 RVI 指令头部 |
| `q_ocpd_h` | wire | `SCR1_IFU_Q_FREE_H_W` | 已占用的半字数（`q_wptr - q_rptr`，0~4） |
| `q_free_h_next` | wire | `SCR1_IFU_Q_FREE_H_W` | 预测性空闲半字数（用下一拍读指针计算） |
| `q_free_w_next` | wire | `SCR1_IFU_Q_FREE_W_W` | 预测性空闲字数（`q_free_h_next >> 1`） |

---

## 5. 内部信号 — IFU FSM 模块

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `ifu_fetch_req` | wire | 1 | 启动取指请求（有新 PC 且未被停止） |
| `ifu_stop_req` | wire | 1 | 停止取指请求（流水线暂停 或 有未决错误待丢弃） |
| `ifu_fsm_curr` | reg | `type_scr1_ifu_fsm_e` | FSM 当前状态寄存器 |
| `ifu_fsm_next` | wire | `type_scr1_ifu_fsm_e` | FSM 下一状态 |
| `ifu_fsm_fetch` | wire | 1 | FSM 处于 FETCH 状态 |

---

## 6. 内部信号 — IMEM 接口模块

### 6.1 IMEM 响应

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `imem_resp_ok` | wire | 1 | IMEM 返回 OK 响应 |
| `imem_resp_er` | wire | 1 | IMEM 返回 ERROR 响应 |
| `imem_resp_er_discard_pnd` | wire | 1 | 有未丢弃的错误响应待处理（触发 FSM 回 IDLE） |
| `imem_resp_discard_req` | wire | 1 | 当前响应需要被丢弃（丢弃计数 > 0） |
| `imem_resp_received` | wire | 1 | 收到了 IMEM 响应（OK 或 ERROR） |
| `imem_resp_vd` | wire | 1 | IMEM 响应有效（已收到 且 不需要丢弃） |
| `imem_handshake_done` | wire | 1 | IFU-IMEM 握手完成（请求发出 & 确认收到） |
| `imem_rdata_lo` | wire | 16 bit | IMEM 返回数据的低 16 位 `[15:0]` |
| `imem_rdata_hi` | wire | 16 bit | IMEM 返回数据的高 16 位 `[31:16]` |

### 6.2 IMEM 地址

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `imem_addr_upd` | wire | 1 | 地址寄存器更新使能（握手完成 或 新 PC） |
| `imem_addr_ff` | reg | `SCR1_XLEN-1:2` | 地址寄存器（字地址，不含低 2 位） |
| `imem_addr_next` | wire | `SCR1_XLEN-1:2` | 下一周期的地址值 |

### 6.3 在途事务计数器

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `imem_pnd_txns_cnt_upd` | wire | 1 | 在途计数更新使能（handshake 和 resp 仅单边发生） |
| `imem_pnd_txns_cnt` | reg | `SCR1_TXN_CNT_W` (3) | 在途事务计数器（已发请求-已收响应） |
| `imem_pnd_txns_cnt_next` | wire | `SCR1_TXN_CNT_W` (3) | 下一周期在途计数值 |
| `imem_vd_pnd_txns_cnt` | wire | `SCR1_TXN_CNT_W` (3) | 有效在途计数（总数 - 正在丢弃数） |
| `imem_pnd_txns_q_full` | wire | 1 | 在途计数器满（=7），禁止发新请求 |

### 6.4 丢弃计数器

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `imem_resp_discard_cnt_upd` | wire | 1 | 丢弃计数更新使能 |
| `imem_resp_discard_cnt` | reg | `SCR1_TXN_CNT_W` (3) | 需要丢弃的响应数量 |
| `imem_resp_discard_cnt_next` | wire | `SCR1_TXN_CNT_W` (3) | 下一周期丢弃计数值 |

---

## 7. 内部信号 — 旁路模块（仅 `SCR1_NO_DEC_STAGE`）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `instr_bypass_type` | wire | `type_scr1_bypass_e` | 当前指令的旁路类型 |
| `instr_bypass_vd` | wire | 1 | 旁路数据有效（旁路类型非 NONE） |

---

## 8. 内部信号 — 新 PC 寄存器（仅 `SCR1_NEW_PC_REG`）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|------|------|
| `new_pc_req_ff` | reg | 1 | 新 PC 请求寄存器（延迟一拍后由 FSM 发出） |

---

## 9. 外部依赖的头文件

| 文件名 | 提供内容 |
|--------|----------|
| `scr1_memif.svh` | `type_scr1_mem_cmd_e`（READ/WRITE 枚举）、`type_scr1_mem_resp_e`（RDY_OK/RDY_ER 枚举）、`SCR1_MEM_CMD_RD` / `SCR1_MEM_RESP_RDY_OK` / `SCR1_MEM_RESP_RDY_ER` 宏 |
| `scr1_arch_description.svh` | `SCR1_XLEN`（32 / 64）、`SCR1_IMEM_AWIDTH`、`SCR1_IMEM_DWIDTH` 等架构宏 |
| `scr1_hdu.svh` | （仅 `SCR1_DBG_EN`）`SCR1_HDU_CORE_INSTR_WIDTH` |
