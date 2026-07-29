# SCR1 RISC-V 处理器 -- 模块连接关系可视化

## 图一：系统顶层架构

```mermaid
graph TD
  subgraph SOC["SOC / FPGA"]
    JTAG["JTAG 调试器<br/>TCK, TMS, TDI, TDO"]
    IRQ_SRC["外部中断源<br/>16 条 IRQ 线"]
    TIMER_VAL["定时器时钟源<br/>RTC / 系统时钟"]
    EXT_MEM["外部存储器<br/>AHB-Lite / AXI4"]
  end

  subgraph SCR1_TOP["scr1_top_ahb / scr1_top_axi"]
    subgraph CORE["scr1_core_top"]
      SCU["scr1_scu<br/>系统控制单元<br/>复位生成 / 系统控制"]
      TAPC["scr1_tapc<br/>JTAG TAP 控制器<br/>16 状态 FSM"]
      TAPCSYNC["scr1_tapc_synchronizer<br/>跨时钟域同步"]
      DMI["scr1_dmi<br/>调试模块接口<br/>JTAG ←→ DM 桥接"]
      DM["scr1_dm<br/>调试模块<br/>HART 控制 / 抽象命令"]
      PIPE["scr1_pipe_top<br/>流水线顶层"]
    end
    TIMER["scr1_timer<br/>内存映射定时器<br/>mtime / mtimecmp"]
    IMEM_RTR["scr1_imem_router<br/>指令存储器路由器"]
    DMEM_RTR["scr1_dmem_router<br/>数据存储器路由器"]
    TCM["scr1_tcm<br/>紧耦合存储器<br/>双端口 SRAM"]
    IMEM_AHB["scr1_imem_ahb<br/>指令 AHB 桥"]
    DMEM_AHB["scr1_dmem_ahb<br/>数据 AHB 桥"]
  end

  JTAG --> TAPC
  TAPC --> TAPCSYNC --> DMI --> DM
  DM --> PIPE
  SCU --> PIPE
  SCU --> DM
  IRQ_SRC --> CORE
  TIMER_VAL --> CORE
  PIPE --> IMEM_RTR
  PIPE --> DMEM_RTR
  IMEM_RTR --> TCM
  IMEM_RTR --> IMEM_AHB --> EXT_MEM
  DMEM_RTR --> TCM
  DMEM_RTR --> TIMER
  DMEM_RTR --> DMEM_AHB --> EXT_MEM
```

---

## 图二：流水线内部 -- 数据通路

```mermaid
graph LR
  subgraph IFU_BLK["IFU 取指单元"]
    IFU["scr1_pipe_ifu<br/>FSM: IDLE ←→ FETCH<br/>4x16 位指令队列"]
  end

  subgraph IDU_BLK["IDU 译码单元"]
    IDU["scr1_pipe_idu<br/>纯组合逻辑<br/>RVI + RVC 译码"]
  end

  subgraph EXU_BLK["EXU 执行单元"]
    EXU["scr1_pipe_exu<br/>操作数准备 + 提交判断<br/>异常仲裁 + 指令流控制"]
    IALU["scr1_pipe_ialu<br/>整数 ALU + 乘除法器<br/>地址加法器 sum2"]
    LSU["scr1_pipe_lsu<br/>加载存储单元<br/>FSM: IDLE → WAIT_MEM"]
    EXU --> IALU
    EXU --> LSU
  end

  MPRF["scr1_pipe_mprf<br/>多端口寄存器文件<br/>2 读 + 1 写<br/>x0 硬连线零"]
  CSR["scr1_pipe_csr<br/>控制状态寄存器<br/>MSTATUS, MIE, MIP, MEPC...<br/>计数器: mcycle, minstret, time"]
  IPIC["scr1_ipic<br/>集成可编程中断控制器<br/>16 路 IRQ 配置"]

  IMEM["指令存储器<br/>IMEM"]
  DMEM["数据存储器<br/>DMEM"]

  IMEM -->|"指令字 [31:0]"| IFU
  IFU -->|"ifu2idu_instr<br/>+ valid + err"| IDU
  IDU -->|"idu2exu_cmd<br/>操作码 + 操作数 + 立即数"| EXU
  EXU -->|"rs1_addr, rs2_addr"| MPRF
  MPRF -->|"rs1_data, rs2_data"| EXU
  EXU -->|"rd_addr + rd_data<br/>写请求"| MPRF
  EXU -->|"CSR 读写 + 异常/中断事件"| CSR
  CSR -->|"CSR 数据 + IRQ 信号 + 新 PC"| EXU
  EXU -->|"new_pc_req + new_pc"| IFU
  CSR -->|"中断配置读写"| IPIC
  IPIC -->|"中断向量"| CSR
  LSU -->|"访存请求 + 地址 + 数据"| DMEM
  DMEM -->|"读取数据 + 响应"| LSU
```

---

## 图三：流水线控制流 -- 暂停 / 清空 / 跳转 / 异常

```mermaid
graph TD
  EXU["EXU 执行单元<br/>异常仲裁 + 指令流控制<br/>异常优先级:<br/>ECALL > EBREAK > 访存异常<br/>> 非法指令 > 中断"]

  IFU["IFU<br/>PC 更新 + 取指"]
  IDU["IDU<br/>译码"]
  MPRF["MPRF<br/>寄存器文件"]

  EXU -->|"new_pc_req / new_pc<br/>跳转/分支/异常/MRET<br/>目标地址"| IFU
  EXU -->|"flush 清空流水线"| IFU
  EXU -->|"stall 暂停取指"| IFU
  EXU -->|"flush 清空"| IDU
  EXU -->|"stall 暂停"| IDU
  EXU -->|"rd_addr + rd_data<br/>写回"| MPRF
  CSR["CSR"] -->|"csr2exu_new_pc<br/>异常/中断向量地址"| EXU
  LSU["LSU"] -->|"访存异常"| EXU
  IPIC["IPIC"] -->|"IRQ 中断请求"| CSR
  CSR -->|"ip_ie 中断待处理且已使能"| EXU
```

---

## 图四：调试子系统

```mermaid
graph TD
  subgraph JTAG_CHAIN["JTAG 扫描链"]
    TAPC["scr1_tapc<br/>TAP 控制器<br/>16 状态 FSM<br/>IR / DR 寄存器"]
    TAPCSYNC["scr1_tapc_synchronizer<br/>TCK ←→ 系统时钟 同步"]
  end

  DMI["scr1_dmi<br/>调试模块接口<br/>地址+数据+操作 三字段协议"]
  DM["scr1_dm<br/>调试模块<br/>DMCONTROL / DMSTATUS / ABSTRACTCS<br/>程序缓冲区 PROGBUF0-5<br/>抽象数据寄存器 DATA0-1"]
  HDU["scr1_pipe_hdu<br/>HART 调试单元<br/>HALT / RESUME / SSTEP<br/>程序缓冲区执行<br/>抽象寄存器访问"]
  TDU["scr1_pipe_tdu<br/>触发调试单元<br/>指令断点 ibrkpt<br/>数据断点 dbrkpt<br/>指令计数断点 icount"]

  EXU["EXU 执行单元"]
  IFU["IFU 取指单元"]
  CSR["CSR 控制状态寄存器"]
  LSU["LSU 加载存储单元"]

  TAPC --> TAPCSYNC --> DMI --> DM --> HDU
  HDU -->|"程序缓冲区指令"| IFU
  HDU -->|"调试控制: halt/run/sstep<br/>no_commit/irq_dsbl 等"| EXU
  HDU -->|"硬件断点禁用"| TDU
  TDU -->|"进入调试模式请求"| HDU
  CSR -->|"CSR 读写 tdata/tselect/tinfo"| TDU
  CSR -->|"CSR 读写 dcsr/dpc/dscratch"| HDU
  EXU -->|"指令监视器 imon"| TDU
  LSU -->|"数据监视器 dmon"| TDU
  TDU -->|"指令断点匹配/异常"| EXU
  TDU -->|"数据断点匹配/异常"| LSU
  EXU -->|"硬件断点触发"| HDU
  DM -->|"ndm_rst_n / hart_rst_n"| SCU["SCU 系统控制单元"]
```

---

## 图五：存储器系统 -- TCM + AHB 桥 + 定时器

```mermaid
graph TD
  CORE["处理器核心<br/>scr1_core_top"]
  IMEM_RTR["scr1_imem_router<br/>地址译码<br/>TCM 范围 vs 外部总线"]
  DMEM_RTR["scr1_dmem_router<br/>地址译码<br/>TCM / 定时器 / 外部总线"]
  TCM["scr1_tcm<br/>紧耦合存储器<br/>64KB 双端口 SRAM<br/>1 周期访问延迟"]
  TIMER["scr1_timer<br/>内存映射定时器<br/>mtime (64位) / mtimecmp<br/>时钟源: RTC 或 系统时钟"]
  IMEM_AHB["scr1_imem_ahb<br/>AHB 指令桥<br/>FIFO 缓冲<br/>请求/响应分离"]
  DMEM_AHB["scr1_dmem_ahb<br/>AHB 数据桥<br/>FIFO 缓冲<br/>请求/响应分离"]
  AHB_BUS["AHB-Lite 总线<br/>HTRANS, HADDR, HWDATA<br/>HRDATA, HREADY, HRESP"]
  EXT_SRAM["外部 SRAM / Flash"]

  CORE -->|"指令取指"| IMEM_RTR
  CORE -->|"数据访存"| DMEM_RTR
  IMEM_RTR -->|"TCM 命中"| TCM
  IMEM_RTR -->|"外部总线"| IMEM_AHB
  DMEM_RTR -->|"TCM 命中"| TCM
  DMEM_RTR -->|"定时器空间"| TIMER
  DMEM_RTR -->|"外部总线"| DMEM_AHB
  IMEM_AHB --> AHB_BUS --> EXT_SRAM
  DMEM_AHB --> AHB_BUS --> EXT_SRAM
```

---

## 图六：中断系统

```mermaid
graph LR
  SOC_IRQ["外部中断源<br/>soc2csr_irq_ext<br/>soc2csr_irq_soft<br/>soc2csr_irq_mtimer"]

  CSR["scr1_pipe_csr<br/>CSR 寄存器文件<br/>MIE / MIP / MTVEC / MCAUSE<br/>MEPC / MSTATUS / MTVAL"]
  IPIC["scr1_ipic<br/>可编程中断控制器<br/>16 路 IRQ<br/>IER 使能 / IMR 模式 / IINVR 极性<br/>ISVR 向量 / IPR 挂起<br/>SOI 开始 / EOI 结束"]
  EXU["EXU 执行单元<br/>中断响应:<br/>保存 PC → MEPC<br/>保存原因 → MCAUSE<br/>跳转 MTVEC 向量"]

  SOC_IRQ --> CSR
  SOC_IRQ --> IPIC
  CSR -->|"csr2ipic 配置读写"| IPIC
  IPIC -->|"ipic2csr 中断向量/CISV"| CSR
  CSR -->|"irq / ip_ie 中断信号"| EXU
  EXU -->|"take_irq 接收中断<br/>MTVEC 模式: direct / vectored"| CSR
```

---

## 模块间信号命名规范

所有模块间信号遵循统一格式:

```
<源模块>2<目标模块>_<信号名>_<方向后缀>
```

| 元素 | 含义 | 示例 |
|------|------|------|
| 源模块 | 信号发出方 | `exu`, `idu`, `ifu`, `mprf`, `csr`, `hdu`, `tdu`, `dmem`, `imem` |
| `2` | "to" 连接符 | -- |
| 目标模块 | 信号接收方 | `exu`, `idu`, `ifu`, `mprf`, `csr`, `hdu`, `tdu`, `dmem`, `imem` |
| `_i` 后缀 | input | 本模块视角的输入端口 |
| `_o` 后缀 | output | 本模块视角的输出端口 |

### 典型连接示例

| 信号全名 | 含义 |
|----------|------|
| `exu2ifu_pc_new_req_o` | EXU 向 IFU 发送的新 PC 请求，在 EXU 是输出 |
| `ifu2idu_instr_vd_o` | IFU 向 IDU 发送的指令有效标志，在 IFU 是输出 |
| `mprf2exu_rs1_data_o` | MPRF 向 EXU 返回的 rs1 数据，在 MPRF 是输出 |
| `csr2exu_irq_o` | CSR 向 EXU 发送的中断请求，在 CSR 是输出 |
| `hdu2exu_dbg_halted_o` | HDU 向 EXU 发送的调试暂停状态，在 HDU 是输出 |
| `dmem2lsu_rdata_i` | DMEM 向 LSU 返回的读数据，在 LSU 是输入 |

---

## 核心模块功能速查

| 模块 | 文件 | 核心功能 | 复杂度 |
|------|------|---------|--------|
| IFU | `scr1_pipe_ifu.sv` | 取指 FSM、指令队列、RVC 拼接、PC 管理 | 中 |
| IDU | `scr1_pipe_idu.sv` | RVI + RVC 译码、立即数生成、纯组合逻辑 | 中 |
| EXU | `scr1_pipe_exu.sv` | 操作数准备、异常仲裁、指令流控制、提交逻辑 | 高 |
| IALU | `scr1_pipe_ialu.sv` | 整数运算、乘除法、地址加法器、标志位 | 中 |
| LSU | `scr1_pipe_lsu.sv` | 访存 FSM、对齐检查、符号扩展、数据断点 | 低 |
| MPRF | `scr1_pipe_mprf.sv` | 2 读 1 写寄存器堆、x0 零寄存器、FPGA RAM 优化 | 低 |
| CSR | `scr1_pipe_csr.sv` | 全部 M-mode CSR、计数器、中断接口 | 高 |
| IPIC | `scr1_ipic.sv` | 16 路 IRQ 配置、向量化响应、SOI/EOI 协议 | 中 |
| HDU | `scr1_pipe_hdu.sv` | HART 调试控制、程序缓冲区执行、抽象寄存器访问 | 高 |
| TDU | `scr1_pipe_tdu.sv` | 硬件断点、指令/数据/计数触发、tdata 寄存器 | 中 |
| SCU | `scr1_scu.sv` | 系统复位生成、复位域交叉同步、粘滞状态 | 低 |
| TAPC | `scr1_tapc.sv` | JTAG TAP 16 状态 FSM、IR/DR 扫描链 | 高 |
| DMI | `scr1_dmi.sv` | JTAG ←→ DM 桥接、三字段协议 | 低 |
| DM | `scr1_dm.sv` | 调试模块、抽象命令、程序缓冲区、HART 控制 | 高 |
| TIMER | `scr1_timer.sv` | 64 位定时器、mtime/mtimecmp、时钟源选择 | 低 |
| TCM | `scr1_tcm.sv` | 紧耦合存储器、双端口 SRAM、零等待访问 | 低 |

---

生成时间: 2026-07-19
涵盖范围: SCR1 全部可综合模块及调试子系统
