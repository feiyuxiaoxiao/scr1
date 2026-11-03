# SCR1 数据通路全景图

本文档用 Mermaid 绘制 SCR1 RISC-V 核心的完整数据通路，涵盖指令执行主通路、中断/异常 trap 流程、调试访问链路、存储层次。

---

## 1. 指令执行主通路（IFU → IDU → EXU → MPRF → CSR）

```mermaid
flowchart LR
    subgraph IFU ["取指单元 (IFU)"]
        ifu_q["指令队列<br/>(环形缓冲 4 半字)"]
        ifu_fsm["FSM: IDLE/FETCH"]
        ifu_pend["未完成事务计数器"]
    end
    
    subgraph IMEM ["指令存储器"]
        imem_tcm["TCM"]
        imem_ahb["AHB 桥"]
        imem_axi["AXI 桥"]
    end
    
    subgraph IDU ["译码单元 (IDU)"]
        idu_decode["指令译码<br/>(纯组合逻辑)"]
        idu_illegal["非法指令检测"]
    end
    
    subgraph EXU ["执行单元 (EXU)"]
        exu_queue["指令队列<br/>(级间寄存器)"]
        exu_ialu["IALU 双加法器"]
        exu_pc["PC 逻辑"]
        exu_wb["写回选择器"]
    end
    
    subgraph MPRF ["寄存器堆"]
        mpRF_rs["2 读端口"]
        mpRF_w["1 写端口"]
    end
    
    subgraph CSR ["控制状态寄存器"]
        csr_trap["Trap 逻辑"]
        csr_regs["MSTATUS/MIE/MIP..."]
        csr_counter["MCYCLE/MINSTRET"]
    end
    
    IFU -->|ifu2idu_instr| IDU
    IDU -->|idu2exu_cmd| EXU
    EXU -->|rs1/rs2 addr| MPRF
    MPRF -->|rs1/rs2 data| EXU
    EXU -->|rd addr/rd data/w_req| MPRF
    EXU -->|exc/irq/mret| CSR
    CSR -->|new_pc| IFU
    IFU -->|取指请求 | IMEM
    IMEM -->|指令数据| IFU
    
    style IFU fill:#e1f5ff
    style IDU fill:#fff4e1
    style EXU fill:#e8f5e9
    style MPRF fill:#fce4ec
    style CSR fill:#f3e5f5
```

**关键信号流**：
1. IFU 从 IMEM 取指 → 指令队列 → IDU
2. IDU 译码 → `idu2exu_cmd`（执行命令）→ EXU 队列
3. EXU 读 MPRF（rs1/rs2）→ IALU 运算 → 写回 MPRF（rd）
4. CSR 处理中断/异常 → 更新 PC → IFU 取新 PC

---

## 2. 中断/异常 Trap 流程

```mermaid
sequenceDiagram
    participant EXU
    participant CSR
    participant IPIC
    participant MPRF
    participant IMEM
    
    EXU->>CSR: take_exc (异常) 或 take_irq (中断)
    CSR->>CSR: 保存现场 (MEPC=MPC, MCAUSE=原因，MTVAL=附加信息)
    CSR->>CSR: 关中断 (MIE=0, MPIE=旧 MIE)
    CSR->>CSR: 计算新 PC (MTVEC.base + offset)
    CSR->>EXU: csr2exu_new_pc
    EXU->>IFU: new_pc_req + new_pc
    IFU->>IMEM: 从新 PC 取指
    IMEM->>IFU: handler 第一条指令
    IFU->>IDU->>EXU->>CSR: handler 执行
    EXU->>CSR: MRET 指令
    CSR->>CSR: 恢复现场 (MIE=MPIE)
    CSR->>EXU: new_pc = MEPC
    EXU->>IFU: 返回被中断的指令
```

**异常 vs 中断**：
| 特性 | 异常 | 中断 |
|---|---|---|
| 来源 | 指令自身（非法指令/访存错误） | 外部异步（IPIC/定时器/软件） |
| 优先级 | 高（`e_irq = take_irq & ~take_exc`） | 低 |
| MEPC | 出错指令地址 | 被中断的指令地址 |
| MCAUSE | 异常码（最高位=0） | 中断码（最高位=1） |

---

## 3. 调试访问链路（JTAG → TAPC → DMI → DM → HDU）

```mermaid
flowchart LR
    subgraph JTAG ["JTAG 接口"]
        jtag_tck["TCK"]
        jtag_tms["TMS"]
        jtag_tdi["TDI"]
        jtag_tdo["TDO"]
    end
    
    subgraph TAPC ["TAP 控制器"]
        tapc_fsm["16 态 TAP FSM"]
        tapc_ir["IR 寄存器"]
        tapc_dr["DR 链选择"]
    end
    
    subgraph DMI ["DMI 桥"]
        dmi_dtmcs["DTMCS"]
        dmi_access["DMI Access"]
    end
    
    subgraph DM ["Debug Module"]
        dm_regs["DM CSR<br/>DMCONTROL/DMSTATUS"]
        dm_abstract["抽象命令 FSM"]
        dm_progbuf["PROGBUF"]
    end
    
    subgraph HDU ["调试单元"]
        hdu_fsm["调试 FSM"]
        hdu_halt["停机请求"]
    end
    
    subgraph CORE ["CPU 核心"]
        core_pipe["流水线"]
    end
    
    JTAG --> TAPC
    TAPC -->|IR=DMI_ACCESS| DMI
    DMI --> DM
    DM -->|haltreq/dresumereq| HDU
    HDU -->|halt_req| CORE
    CORE -->|halted| HDU
    HDU -->|dmode| DM
    
    style JTAG fill:#ffe0b2
    style TAPC fill:#c8e6c9
    style DMI fill:#bbdefb
    style DM fill:#e1bee7
    style HDU fill:#ffcdd2
    style CORE fill:#f5f5f5
```

**调试会话流程**：
1. 调试器 → JTAG TMS/TDI → TAPC 移位 → IR 译码 → 选中 DMI 链
2. DMI 解析 op/address/data → DM 寄存器访问
3. DM 写 `DMCONTROL.haltreq=1` → HDU 发 `halt_req` → 流水线停机
4. 调试器通过 DM 读/写核寄存器（抽象命令或 PROGBUF）
5. 调试器写 `DMCONTROL.dresumereq=1` → 恢复执行

---

## 4. 存储访问层次（LSU → Router → TCM/Timer/AHB/AXI）

```mermaid
flowchart TD
    subgraph LSU ["访存单元 (LSU)"]
        lsu_fsm["FSM: IDLE/BUSY"]
        lsu_align["地址对齐检查"]
        lsu_ext["符号扩展"]
    end
    
    subgraph ROUTER ["数据路由器"]
        router_decode["地址译码"]
        router_port["三端口优先级<br/>TCM > Timer > AHB"]
    end
    
    subgraph TCM ["紧耦合内存"]
        tcm_sram["片上 SRAM"]
        tcm_align["字节路对齐"]
    end
    
    subgraph TIMER ["定时器"]
        timer_regs["MTIME/MTIMECMP"]
        timer_irq["中断输出"]
    end
    
    subgraph AHB ["AHB 桥"]
        ahb_fifo["请求/响应 FIFO"]
        ahb_fsm["两态 FSM"]
    end
    
    subgraph AXI ["AXI 桥"]
        axi_queue["环形队列"]
        axi_ptr["三指针"]
    end
    
    LSU -->|dmem_req| ROUTER
    ROUTER -->|port1| TCM
    ROUTER -->|port2| TIMER
    ROUTER -->|port0| AHB
    ROUTER -->|port0| AXI
    
    TCM -->|单周期响应 | ROUTER
    TIMER -->|慢速响应 | ROUTER
    AHB -->|HREADY 应答 | ROUTER
    AXI -->|RLAST/BLAST| ROUTER
    
    ROUTER -->|dmem_rdata + resp| LSU
    
    style LSU fill:#e3f2fd
    style ROUTER fill:#fff9c4
    style TCM fill:#dcedc8
    style TIMER fill:#f8bbd0
    style AHB fill:#b2dfdb
    style AXI fill:#e1bee7
```

**地址译码示例**（TCM 在 `0x00480000`，Timer 在 `0x00490000`）：
```
addr[31:0] & PORT1_MASK == PORT1_PATTERN → TCM (32KB @ 0x480000)
addr[31:0] & PORT2_MASK == PORT2_PATTERN → Timer (4KB @ 0x490000)
其他 → AHB/AXI 桥（外部总线）
```

---

## 5. 复位层次（Sys ⊃ Core ⊃ HDU ⊃ DM）

```mermaid
flowchart TB
    subgraph SYS ["系统级"]
        pwrup_rst["pwrup_rst_n<br/>(上电复位)"]
        sys_rst["sys_rst_n<br/>(rst_n & cpu_rst_n & pwrup_rst_n)"]
    end
    
    subgraph CORE ["核心级"]
        core_sync["2 级同步器"]
        core_qlfy["Qualifier (早一拍)"]
        core_rst["core_rst_n_out"]
    end
    
    subgraph HDU ["HDU 级"]
        hdu_rst["hdu_rst_n<br/>(core_rst & ~halt_req)"]
    end
    
    subgraph DM ["DM 级"]
        dm_rst["dm_rst_n"]
    end
    
    SYS --> CORE
    CORE --> HDU
    HDU --> DM
    
    SYS -.->|SCU 分发 | CORE
    SYS -.->|SCU 分发 | HDU
    SYS -.->|SCU 分发 | DM
    
    style SYS fill:#ffcdd2
    style CORE fill:#f8bbd0
    style HDU fill:#f48fb1
    style DM fill:#f06292
```

**关键设计**：
- `reset_n_qlfy` 比 `reset_n_out` 早一拍拉低（给 RDC 同步器留时间）
- HDU 复位独立于核心（系统复位时调试器不能失联）
- DM 用 `scr1_reset_buf_cell` 缓冲（`:454-462`）

---

## 6. 时钟门控（ClkCtrl → ICG → 全芯片）

```mermaid
flowchart LR
    subgraph CLKCTRL ["时钟控制器"]
        clk_en_reg["clk_en 寄存器"]
        wake["wake_req<br/>(中断/调试)"]
        sleep["sleep_req<br/>(WFI)"]
    end
    
    subgraph ICG ["时钟门控单元"]
        latch["锁存器<br/>(clk=0 采样)"]
        and["与门"]
    end
    
    subgraph LOADS ["负载"]
        core["核心逻辑"]
        debug["调试模块 (常开)"]
    end
    
    wake -->|置 1| clk_en_reg
    sleep -->|清 0| clk_en_reg
    clk_en_reg -->|clk_en| ICG
    ICG -->|clk_out| LOADS
    
    latch -.->|抗毛刺 | and
    and -.-> clk_out
    
    style CLKCTRL fill:#e0f7fa
    style ICG fill:#b2ebf2
    style LOADS fill:#80deea
```

**ICG 抗毛刺原理**：
```systemverilog
always_latch begin
    if (~clk) latch_en <= clk_en;  // 只在 clk=0 时采样
end
assign clk_out = latch_en & clk;   // 高电平永远完整
```

---

## 7. 完整 SoC 视图（Top 层级）

```mermaid
flowchart TB
    subgraph TOP_AHB ["AHB 顶层"]
        core_top["scr1_core_top"]
        imem_router["Imem Router"]
        dmem_router["Dmem Router"]
        tcm["TCM"]
        timer["Timer"]
        imem_ahb["Imem AHB 桥"]
        dmem_ahb["Dmem AHB 桥"]
    end
    
    subgraph TOP_AXI ["AXI 顶层"]
        imem_axi["Imem AXI 桥"]
        dmem_axi["Dmem AXI 桥"]
    end
    
    subgraph EXT_AHB ["外部 AHB 总线"]
        ahb_sram["SRAM/Flash"]
        ahb_periph["外设"]
    end
    
    subgraph EXT_AXI ["外部 AXI 总线"]
        axi_ddr["DDR"]
        axi_dma["DMA"]
    end
    
    TOP_AHB -->|AHB 总线 | EXT_AHB
    TOP_AXI -->|AXI 总线 | EXT_AXI
    
    style TOP_AHB fill:#e8eaf6
    style TOP_AXI fill:#c5cae9
    style EXT_AHB fill:#9fa8da
    style EXT_AXI fill:#7986cb
```

---

## 附录：32 个模块索引

详见 [`docs/module-walkthroughs/README.md`](../module-walkthroughs/README.md)

---

*本文档配合 32 个模块讲解使用，提供全局视角。*
