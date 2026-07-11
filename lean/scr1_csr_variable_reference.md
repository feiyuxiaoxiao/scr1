# SCR1 CSR 变量、参数与类型参考手册

> 对应源码: `src/core/pipeline/scr1_pipe_csr.sv`
> 模块: `scr1_pipe_csr` — Control and Status Registers

---

## 1. 局部参数 (localparam)

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `PC_LSB` | 1 (RVC 扩展) / 2 (无 RVC) | PC 的最低有效位，用于 MEPC 地址对齐 |

---

## 2. 从外部头文件引入的类型与常量

### 2.1 CSR 地址常量 — `scr1_csr.svh`

#### 机器信息寄存器 (Machine Information, 只读)

| 参数名 | 地址 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_MVENDORID` | `0xF11` | 厂商标识符 |
| `SCR1_CSR_ADDR_MARCHID` | `0xF12` | 架构标识符 |
| `SCR1_CSR_ADDR_MIMPID` | `0xF13` | 实现标识符 |
| `SCR1_CSR_ADDR_MHARTID` | `0xF14` | 硬件线程标识符 |

#### 机器陷阱设置寄存器 (Machine Trap Setup, 读写)

| 参数名 | 地址 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_MSTATUS` | `0x300` | 机器状态寄存器 |
| `SCR1_CSR_ADDR_MISA` | `0x301` | ISA 与控制寄存器 |
| `SCR1_CSR_ADDR_MIE` | `0x304` | 机器中断使能寄存器 |
| `SCR1_CSR_ADDR_MTVEC` | `0x305` | 机器陷阱向量基址寄存器 |

#### 机器陷阱处理寄存器 (Machine Trap Handling, 读写)

| 参数名 | 地址 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_MSCRATCH` | `0x340` | 机器暂存寄存器 |
| `SCR1_CSR_ADDR_MEPC` | `0x341` | 机器异常返回地址 |
| `SCR1_CSR_ADDR_MCAUSE` | `0x342` | 机器异常原因 |
| `SCR1_CSR_ADDR_MTVAL` | `0x343` | 机器异常值 |
| `SCR1_CSR_ADDR_MIP` | `0x344` | 机器中断挂起 |

#### 机器计数器/定时器 (Machine Counters/Timers, 读写)

| 参数名 | 地址 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_MCYCLE` | `0xB00` | 机器周期计数器（低 32 位） |
| `SCR1_CSR_ADDR_MINSTRET` | `0xB02` | 机器退休指令计数器（低 32 位） |
| `SCR1_CSR_ADDR_MCYCLEH` | `0xB80` | 机器周期计数器（高 32 位） |
| `SCR1_CSR_ADDR_MINSTRETH` | `0xB82` | 机器退休指令计数器（高 32 位） |

#### 用户计数器/定时器阴影 (User Counters/Timers, 只读)

| 参数名 | 地址 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_TIME` | `0xC01` | 定时器值（只读阴影） |
| `SCR1_CSR_ADDR_CYCLE` | `0xC00` | 周期计数器阴影（低 32 位） |
| `SCR1_CSR_ADDR_INSTRET` | `0xC02` | 退休指令计数阴影（低 32 位） |
| `SCR1_CSR_ADDR_TIMEH` | `0xC81` | 定时器值阴影（高 32 位） |
| `SCR1_CSR_ADDR_CYCLEH` | `0xC80` | 周期计数器阴影（高 32 位） |
| `SCR1_CSR_ADDR_INSTRETH` | `0xC82` | 退休指令计数阴影（高 32 位） |

#### HPM 掩码 (硬件性能监视器地址匹配)

| 参数名 | 掩码 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_HPMCOUNTER_MASK` | `7'b110_0000` | 用户级 HPM 计数器地址掩码 |
| `SCR1_CSR_ADDR_HPMCOUNTERH_MASK` | `7'b110_0100` | 用户级 HPM 计数器高 32 位掩码 |
| `SCR1_CSR_ADDR_MHPMCOUNTER_MASK` | `7'b101_1000` | 机器级 HPM 计数器地址掩码 |
| `SCR1_CSR_ADDR_MHPMCOUNTERH_MASK` | `7'b101_1100` | 机器级 HPM 计数器高 32 位掩码 |
| `SCR1_CSR_ADDR_MHPMEVENT_MASK` | `7'b001_1001` | 机器级 HPM 事件选择器地址掩码 |

#### 非标准 CSR 地址

| 参数名 | 地址 | 条件 | 含义 |
|--------|:---:|------|------|
| `SCR1_CSR_ADDR_MCOUNTEN` | `0x7E0` | `SCR1_MCOUNTEN_EN` | 计数器使能寄存器 |
| `SCR1_CSR_ADDR_TDU_MBASE` | `0x7A0` | `SCR1_TDU_EN` | TDU 寄存器基址 |
| `SCR1_CSR_ADDR_TDU_MSPAN` | `0x008` | `SCR1_TDU_EN` | TDU 寄存器跨度 |
| `SCR1_CSR_ADDR_HDU_MBASE` | `0x7B0` | `SCR1_DBG_EN` | HDU 调试 CSR 基址 |
| `SCR1_CSR_ADDR_HDU_MSPAN` | `0x004` | `SCR1_DBG_EN` | HDU 调试 CSR 跨度 |
| `SCR1_CSR_ADDR_IPIC_BASE` | `0xBF0` | `SCR1_IPIC_EN` | IPIC 寄存器基址 |

### 2.2 CSR 定义常量 — `scr1_csr.svh`

#### MSTATUS 相关

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_MSTATUS_MIE_OFFSET` | 3 | MIE 位在 MSTATUS 中的偏移 |
| `SCR1_CSR_MSTATUS_MPIE_OFFSET` | 7 | MPIE 位在 MSTATUS 中的偏移 |
| `SCR1_CSR_MSTATUS_MPP_OFFSET` | 11 | MPP 位在 MSTATUS 中的偏移 |
| `SCR1_CSR_MSTATUS_MPP` | `2'b11` | MPP 硬连线为 Machine 模式 |
| `SCR1_CSR_MSTATUS_MIE_RST_VAL` | `1'b0` | MIE 复位值 |
| `SCR1_CSR_MSTATUS_MPIE_RST_VAL` | `1'b1` | MPIE 复位值 |

#### MIE/MIP 相关

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_MIE_MSIE_OFFSET` | 3 | MSIE/MSIP 位偏移 |
| `SCR1_CSR_MIE_MTIE_OFFSET` | 7 | MTIE/MTIP 位偏移 |
| `SCR1_CSR_MIE_MEIE_OFFSET` | 11 | MEIE/MEIP 位偏移 |
| `SCR1_CSR_MIE_MSIE_RST_VAL` | `1'b0` | MSIE 复位值 |
| `SCR1_CSR_MIE_MTIE_RST_VAL` | `1'b0` | MTIE 复位值 |
| `SCR1_CSR_MIE_MEIE_RST_VAL` | `1'b0` | MEIE 复位值 |

#### MTVEC 相关

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_MTVEC_BASE_RST_VAL` | `SCR1_CSR_MTVEC_BASE_WR_RST_VAL` | MTVEC.base 复位/硬连线值 |
| `SCR1_CSR_MTVEC_MODE_DIRECT` | `1'b0` | 直接模式：所有 trap 跳转到 BASE |
| `SCR1_CSR_MTVEC_MODE_VECTORED` | `1'b1` | 向量模式：异步中断跳转到 BASE+4×cause |

#### MISA 相关

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_MISA_MXL_32` | `2'd1` | MXL 字段编码 RV32 |
| `SCR1_CSR_MISA` | 拼接值 | MISA 寄存器聚合值 |
| `` `SCR1_RVC_ENC `` | `0x0004` | RVC 扩展位（bit 2） |
| `` `SCR1_RVE_ENC `` | `0x0010` | RVE 扩展位（bit 4） |
| `` `SCR1_RVI_ENC `` | `0x0100` | RVI 扩展位（bit 8） |
| `` `SCR1_RVM_ENC `` | `0x1000` | RVM 扩展位（bit 12） |

#### 信息寄存器

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_MVENDORID` | `` `SCR1_MVENDORID `` (`0x00000000`) | 厂商标识符 |
| `SCR1_CSR_MARCHID` | `32'd8` | 架构标识符 |
| `SCR1_CSR_MIMPID` | `` `SCR1_MIMPID `` (`0x22011200`) | 实现标识符 |

#### MCOUNTEN 相关

| 参数名 | 值 | 条件 | 含义 |
|--------|:---:|------|------|
| `SCR1_CSR_MCOUNTEN_CY_OFFSET` | 0 | `SCR1_MCOUNTEN_EN` | cycle 使能位偏移 |
| `SCR1_CSR_MCOUNTEN_IR_OFFSET` | 2 | `SCR1_MCOUNTEN_EN` | instret 使能位偏移 |

#### 计数器宽度

| 参数名 | 值 | 条件 | 含义 |
|--------|:---:|------|------|
| `SCR1_CSR_COUNTERS_WIDTH` | 32 | `SCR1_CSR_REDUCED_CNT` | 计数器位宽（缩减模式） |
| `SCR1_CSR_COUNTERS_WIDTH` | 64 | 默认 | 计数器位宽（完整模式） |

### 2.3 架构类型定义 — `scr1_arch_types.svh`

#### CSR 地址与 MTVEC 基础参数

| 参数名 | 值 | 含义 |
|--------|:---:|------|
| `SCR1_CSR_ADDR_WIDTH` | 12 | CSR 地址位宽 |
| `SCR1_CSR_MTVEC_BASE_ZERO_BITS` | 6 | MTVEC.base 低 6 位硬连线为 0 |
| `SCR1_CSR_MTVEC_BASE_VAL_BITS` | `XLEN - 6` (26) | MTVEC.base 有效位宽 |
| `SCR1_CSR_MTVEC_BASE_RO_BITS` | `32 - 6 - WR_BITS` | MTVEC.base 只读位数量 |
| `` `SCR1_XLEN `` | 32 | 通用寄存器位宽 |
| `` `SCR1_MPRF_AWIDTH `` | 5 (RVI) / 4 (RVE) | 寄存器文件地址位宽 |

#### 异常与中断编码

| 枚举值 | 编码 | 含义 |
|--------|:---:|------|
| `SCR1_EXC_CODE_WIDTH_E` | 4 | 异常编码位宽 |
| `SCR1_EXC_CODE_INSTR_MISALIGN` | 4'd0 | 指令地址不对齐 |
| `SCR1_EXC_CODE_INSTR_ACCESS_FAULT` | 4'd1 | 指令访问故障 |
| `SCR1_EXC_CODE_ILLEGAL_INSTR` | 4'd2 | 非法指令 |
| `SCR1_EXC_CODE_BREAKPOINT` | 4'd3 | 断点 |
| `SCR1_EXC_CODE_LD_ADDR_MISALIGN` | 4'd4 | 加载地址不对齐 |
| `SCR1_EXC_CODE_LD_ACCESS_FAULT` | 4'd5 | 加载访问故障 |
| `SCR1_EXC_CODE_ST_ADDR_MISALIGN` | 4'd6 | 存储地址不对齐 |
| `SCR1_EXC_CODE_ST_ACCESS_FAULT` | 4'd7 | 存储访问故障 |
| `SCR1_EXC_CODE_ECALL_M` | 4'd11 | M-mode 环境调用 |
| `SCR1_EXC_CODE_IRQ_M_SOFTWARE` | 4'd3 | 机器软件中断 |
| `SCR1_EXC_CODE_IRQ_M_TIMER` | 4'd7 | 机器定时器中断 |
| `SCR1_EXC_CODE_IRQ_M_EXTERNAL` | 4'd11 | 机器外部中断 |
| `SCR1_EXC_CODE_RESET` | 4'd0 | 复位默认编码 |

### 2.4 CSR 命令类型 — `scr1_riscv_isa_decoding.svh`

| 枚举值 | 含义 |
|--------|------|
| `SCR1_CSR_CMD_NONE` | 无操作（默认值） |
| `SCR1_CSR_CMD_WRITE` | CSR 直接写入（CSRRW） |
| `SCR1_CSR_CMD_SET` | CSR 置位（CSRRS） |
| `SCR1_CSR_CMD_CLEAR` | CSR 清位（CSRRC） |

### 2.5 CSR 响应类型 — `scr1_csr.svh`

| 枚举值 | 含义 |
|--------|------|
| `SCR1_CSR_RESP_OK` | 响应正常 |
| `SCR1_CSR_RESP_ER` | 响应错误 |
| `SCR1_CSR_RESP_ERROR` | X 传播（仅 `SCR1_XPROP_EN`） |

### 2.6 MCAUSE 向量类型 — `scr1_csr.svh`

| 类型名 | 定义 | 含义 |
|--------|------|------|
| `type_scr1_csr_mcause_ec_v` | `logic [XLEN-2:0]` | MCAUSE 异常编码向量的位宽类型 |

---

## 3. 模块端口 (I/O)

### 3.1 公共信号

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `rst_n` | 1 | 低有效异步复位 |
| input | `clk` | 1 | 门控时钟 |
| input | `clk_alw_on` | 1 | 非门控时钟（仅 MCYCLE 使用，条件：`~SCR1_CSR_REDUCED_CNT && SCR1_CLKCTRL_EN`） |

### 3.2 SOC 信号

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `soc2csr_irq_ext_i` | 1 | 外部中断请求 |
| input | `soc2csr_irq_soft_i` | 1 | 软件中断请求 |
| input | `soc2csr_irq_mtimer_i` | 1 | 定时器中断请求 |
| input | `soc2csr_mtimer_val_i` | 64 | 外部定时器 64 位值 |
| input | `soc2csr_fuse_mhartid_i` | `XLEN` (32) | MHARTID 熔丝值 |

### 3.3 CSR ↔ EXU 读写接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `exu2csr_r_req_i` | 1 | CSR 读请求 |
| input | `exu2csr_rw_addr_i` | `SCR1_CSR_ADDR_WIDTH` (12) | CSR 读写地址 |
| output | `csr2exu_r_data_o` | `XLEN` (32) | CSR 读数据 |
| input | `exu2csr_w_req_i` | 1 | CSR 写请求 |
| input | `exu2csr_w_cmd_i` | `type_scr1_csr_cmd_sel_e` (2) | CSR 写命令 |
| input | `exu2csr_w_data_i` | `XLEN` (32) | CSR 写数据 |
| output | `csr2exu_rw_exc_o` | 1 | CSR 读写访问异常 |

### 3.4 CSR ↔ EXU 事件接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `exu2csr_take_irq_i` | 1 | 触发 IRQ 陷阱 |
| input | `exu2csr_take_exc_i` | 1 | 触发异常陷阱 |
| input | `exu2csr_mret_update_i` | 1 | MRET 更新 CSR |
| input | `exu2csr_mret_instr_i` | 1 | MRET 指令标志 |
| input | `exu2csr_exc_code_i` | `type_scr1_exc_code_e` (4) | 异常编码 |
| input | `exu2csr_trap_val_i` | `XLEN` (32) | 陷阱值 |
| output | `csr2exu_irq_o` | 1 | IRQ 请求（全局使能后的中断） |
| output | `csr2exu_ip_ie_o` | 1 | 有中断挂起且本地使能 |
| output | `csr2exu_mstatus_mie_up_o` | 1 | MSTATUS 或 MIE 本周期更新 |

### 3.5 CSR ↔ IPIC 接口（仅 `SCR1_IPIC_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| output | `csr2ipic_r_req_o` | 1 | IPIC 读请求 |
| output | `csr2ipic_w_req_o` | 1 | IPIC 写请求 |
| output | `csr2ipic_addr_o` | 3 | IPIC 寄存器地址 |
| output | `csr2ipic_wdata_o` | `XLEN` (32) | IPIC 写数据 |
| input | `ipic2csr_rdata_i` | `XLEN` (32) | IPIC 读数据 |

### 3.6 CSR ↔ HDU 接口（仅 `SCR1_DBG_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| output | `csr2hdu_req_o` | 1 | HDU 请求 |
| output | `csr2hdu_cmd_o` | `type_scr1_csr_cmd_sel_e` (2) | HDU 写命令 |
| output | `csr2hdu_addr_o` | `SCR1_HDU_DEBUGCSR_ADDR_WIDTH` | HDU 寄存器地址 |
| output | `csr2hdu_wdata_o` | `XLEN` (32) | HDU 写数据 |
| input | `hdu2csr_rdata_i` | `XLEN` (32) | HDU 读数据 |
| input | `hdu2csr_resp_i` | `type_scr1_csr_resp_e` | HDU 响应状态 |
| input | `hdu2csr_no_commit_i` | 1 | 禁止指令提交 |

### 3.7 CSR ↔ TDU 接口（仅 `SCR1_TDU_EN`）

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| output | `csr2tdu_req_o` | 1 | TDU 请求 |
| output | `csr2tdu_cmd_o` | `type_scr1_csr_cmd_sel_e` (2) | TDU 写命令 |
| output | `csr2tdu_addr_o` | `SCR1_CSR_ADDR_TDU_OFFS_W` | TDU 寄存器地址 |
| output | `csr2tdu_wdata_o` | `XLEN` (32) | TDU 写数据 |
| input | `tdu2csr_rdata_i` | `XLEN` (32) | TDU 读数据 |
| input | `tdu2csr_resp_i` | `type_scr1_csr_resp_e` | TDU 响应状态 |

### 3.8 CSR ↔ EXU PC 接口

| 方向 | 端口名 | 位宽 | 含义 |
|------|--------|:---:|------|
| input | `exu2csr_instret_no_exc_i` | 1 | 指令已退休（无异常）— 仅 `~SCR1_CSR_REDUCED_CNT` |
| input | `exu2csr_pc_curr_i` | `XLEN` (32) | 当前 PC |
| input | `exu2csr_pc_next_i` | `XLEN` (32) | 下一条 PC |
| output | `csr2exu_new_pc_o` | `XLEN` (32) | 异常/IRQ/MRET 新 PC |

---

## 4. 内部信号

### 4.1 事件标志

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `e_exc` | logic | 1 | 成功触发异常陷阱（优先级最高） |
| `e_irq` | logic | 1 | 成功触发 IRQ 陷阱（优先级次之） |
| `e_mret` | logic | 1 | MRET 指令执行 |
| `e_irq_nmret` | logic | 1 | IRQ 陷阱且非 MRET 指令 |

### 4.2 中断挂起与使能

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_eirq_pnd_en` | logic | 1 | 外部中断挂起且本地使能 (MEIP & MEIE) |
| `csr_sirq_pnd_en` | logic | 1 | 软件中断挂起且本地使能 (MSIP & MSIE) |
| `csr_tirq_pnd_en` | logic | 1 | 定时器中断挂起且本地使能 (MTIP & MTIE) |

### 4.3 异常标志

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_w_exc` | logic | 1 | CSR 写访问异常（非法地址） |
| `csr_r_exc` | logic | 1 | CSR 读访问异常（非法地址） |
| `exu_req_no_exc` | logic | 1 | EXU 请求且无异常 |

### 4.4 请求信号

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_ipic_req` | logic | 1 | IPIC 读写请求合并 |
| `csr_hdu_req` | logic | 1 | 内部 HDU 请求 |
| `csr_brkm_req` | logic | 1 | 内部 TDU/BRKM 请求 |

### 4.5 Machine Trap Setup 寄存器信号

#### MSTATUS

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mstatus_upd` | logic | 1 | MSTATUS 写更新使能 |
| `csr_mstatus` | logic | `XLEN` | MSTATUS 聚合值（MIE/MPIE/MPP 拼接） |
| `csr_mstatus_mie_ff` | logic | 1 | 全局中断使能（当前值） |
| `csr_mstatus_mie_next` | logic | 1 | 全局中断使能（下一周期值） |
| `csr_mstatus_mpie_ff` | logic | 1 | 陷阱前的中断使能（当前值） |
| `csr_mstatus_mpie_next` | logic | 1 | 陷阱前的中断使能（下一周期值） |

#### MIE

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mie_upd` | logic | 1 | MIE 写更新使能 |
| `csr_mie` | logic | `XLEN` | MIE 聚合值 |
| `csr_mie_mtie_ff` | logic | 1 | 定时器中断使能 |
| `csr_mie_meie_ff` | logic | 1 | 外部中断使能 |
| `csr_mie_msie_ff` | logic | 1 | 软件中断使能 |

#### MTVEC

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mtvec_upd` | logic | 1 | MTVEC 写更新使能 |
| `csr_mtvec_base` | logic | 26:0 | 陷阱向量基址（高 26 位） |
| `csr_mtvec_mode` | logic | 1 | 陷阱模式（0=direct, 1=vectored） |
| `csr_mtvec_mode_ff` | logic | 1 | MTVEC.mode 寄存器值（仅 `SCR1_MTVEC_MODE_EN`） |
| `csr_mtvec_mode_vect` | logic | 1 | 当前为向量模式（仅 `SCR1_MTVEC_MODE_EN`） |

### 4.6 Machine Trap Handling 寄存器信号

#### MSCRATCH

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mscratch_upd` | logic | 1 | MSCRATCH 写更新使能 |
| `csr_mscratch_ff` | logic | `XLEN` | MSCRATCH 寄存器值 |

#### MEPC

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mepc_upd` | logic | 1 | MEPC 写更新使能 |
| `csr_mepc_ff` | logic | `32-PC_LSB` | MEPC 寄存器值（对齐后） |
| `csr_mepc_next` | logic | `32-PC_LSB` | MEPC 下一周期值 |
| `csr_mepc` | logic | `XLEN` | MEPC 扩展到 32 位（低 PC_LSB 位补 0） |

#### MCAUSE

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mcause_upd` | logic | 1 | MCAUSE 写更新使能 |
| `csr_mcause_i_ff` | logic | 1 | 中断标志（bit XLEN-1）：1=中断, 0=异常 |
| `csr_mcause_i_next` | logic | 1 | 中断标志下一值 |
| `csr_mcause_ec_ff` | `type_scr1_exc_code_e` | 4 | 异常/中断编码（当前值） |
| `csr_mcause_ec_next` | `type_scr1_exc_code_e` | 4 | 异常/中断编码（下一值） |
| `csr_mcause_ec_new` | `type_scr1_exc_code_e` | 4 | 中断编码（IRQ 优先级仲裁结果） |

#### MTVAL

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mtval_upd` | logic | 1 | MTVAL 写更新使能 |
| `csr_mtval_ff` | logic | `XLEN` | MTVAL 寄存器值 |
| `csr_mtval_next` | logic | `XLEN` | MTVAL 下一周期值 |

#### MIP

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mip` | logic | `XLEN` | MIP 聚合值 |
| `csr_mip_mtip` | logic | 1 | 定时器中断挂起 |
| `csr_mip_meip` | logic | 1 | 外部中断挂起 |
| `csr_mip_msip` | logic | 1 | 软件中断挂起 |

### 4.7 Machine Counters/Timers 信号

#### MCYCLE（仅 `~SCR1_CSR_REDUCED_CNT`）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mcycle_upd` | logic | 2 | MCYCLE 写更新：`[0]`=低 32 位, `[1]`=高 32 位 |
| `csr_mcycle` | logic | `SCR1_CSR_COUNTERS_WIDTH` | MCYCLE 完整值 |
| `csr_mcycle_lo_inc` | logic | 1 | MCYCLE 低 8 位递增使能 |
| `csr_mcycle_lo_upd` | logic | 1 | MCYCLE 低 8 位更新使能 |
| `csr_mcycle_lo_ff` | logic | 8 | MCYCLE 低 8 位寄存器 |
| `csr_mcycle_lo_next` | logic | 8 | MCYCLE 低 8 位下一值 |
| `csr_mcycle_hi_inc` | logic | 1 | MCYCLE 高 56 位递增使能 |
| `csr_mcycle_hi_upd` | logic | 1 | MCYCLE 高 56 位更新使能 |
| `csr_mcycle_hi_ff` | logic | 56 | MCYCLE 高 56 位寄存器 |
| `csr_mcycle_hi_next` | logic | 56 | MCYCLE 高 56 位下一值 |
| `csr_mcycle_hi_new` | logic | 56 | MCYCLE 高 56 位 +1 结果 |

#### MINSTRET（仅 `~SCR1_CSR_REDUCED_CNT`）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_minstret_upd` | logic | 2 | MINSTRET 写更新：`[0]`=低 32 位, `[1]`=高 32 位 |
| `csr_minstret` | logic | `SCR1_CSR_COUNTERS_WIDTH` | MINSTRET 完整值 |
| `csr_minstret_lo_inc` | logic | 1 | MINSTRET 低 8 位递增使能 |
| `csr_minstret_lo_upd` | logic | 1 | MINSTRET 低 8 位更新使能 |
| `csr_minstret_lo_ff` | logic | 8 | MINSTRET 低 8 位寄存器 |
| `csr_minstret_lo_next` | logic | 8 | MINSTRET 低 8 位下一值 |
| `csr_minstret_hi_inc` | logic | 1 | MINSTRET 高 56 位递增使能 |
| `csr_minstret_hi_upd` | logic | 1 | MINSTRET 高 56 位更新使能 |
| `csr_minstret_hi_ff` | logic | 56 | MINSTRET 高 56 位寄存器 |
| `csr_minstret_hi_next` | logic | 56 | MINSTRET 高 56 位下一值 |
| `csr_minstret_hi_new` | logic | 56 | MINSTRET 高 56 位 +1 结果 |

### 4.8 非标准 CSR 信号

#### MCOUNTEN（仅 `SCR1_MCOUNTEN_EN`）

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_mcounten_upd` | logic | 1 | MCOUNTEN 写更新使能 |
| `csr_mcounten` | logic | `XLEN` | MCOUNTEN 聚合值 |
| `csr_mcounten_cy_ff` | logic | 1 | Cycle 计数器使能 |
| `csr_mcounten_ir_ff` | logic | 1 | Instret 计数器使能 |

### 4.9 CSR 读/写接口中间信号

| 信号名 | 类型 | 位宽 | 含义 |
|--------|------|:---:|------|
| `csr_r_data` | logic | `XLEN` | 读数据组合逻辑输出 |
| `csr_w_data` | logic | `XLEN` | 写数据（经 WRITE/SET/CLEAR 处理） |

---

## 5. 外部依赖头文件

| 文件名 | 提供的关键内容 |
|--------|------|
| `scr1_arch_description.svh` | `SCR1_XLEN`、ISA 扩展宏（`SCR1_RVC_EXT`/`SCR1_RVM_EXT`）、`SCR1_MTVEC_BASE_WR_BITS`、`SCR1_MCOUNTEN_EN`、`SCR1_CSR_REDUCED_CNT`、`SCR1_MTVEC_MODE_EN`、`SCR1_CLKCTRL_EN` |
| `scr1_csr.svh` | 所有 CSR 地址参数、MSTATUS/MIE/MISA/MTVEC/MCOUNTEN 的定义常量、`type_scr1_csr_resp_e` 枚举、`type_scr1_csr_mcause_ec_v` 类型 |
| `scr1_arch_types.svh` | `SCR1_CSR_ADDR_WIDTH`、`SCR1_CSR_MTVEC_BASE_ZERO_BITS`、`type_scr1_exc_code_e` 枚举、`type_scr1_mprf_v`、`type_scr1_pc_v` |
| `scr1_riscv_isa_decoding.svh` | `type_scr1_csr_cmd_sel_e` (WRITE/SET/CLEAR) |
| `scr1_ipic.svh` | IPIC 寄存器偏移地址、`SCR1_IPIC_CISV` 等常量 |
| `scr1_hdu.svh` | HDU CSR 地址常量、`SCR1_HDU_DEBUGCSR_ADDR_WIDTH` |
| `scr1_tdu.svh` | TDU CSR 地址常量、`SCR1_CSR_ADDR_TDU_OFFS_W` |
