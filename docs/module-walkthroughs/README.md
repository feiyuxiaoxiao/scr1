# SCR1 模块讲解索引

## 流水线核心（8 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 1 | IFU | `01-scr1-pipe-ifu.md` | 826 | 取指 + 指令队列 + RVC 对齐 |
| 2 | IDU | `02-scr1-pipe-idu.md` | 940 | 指令译码 → 生成执行命令 |
| 3 | EXU | `03-scr1-pipe-exu.md` | 1086 | ALU 运算 + 异常 + PC 控制 |
| 4 | LSU | `04-scr1-pipe-lsu.md` | 351 | 访存 + 地址对齐检查 |
| 5 | IALU | `05-scr1-pipe-ialu.md` | 720 | 双加法器 + 移位 + MDU 乘除 |
| 6 | MPRF | `06-scr1-pipe-mprf.md` | 167 | 2 读 1 写寄存器堆（x0 恒 0） |
| 7 | CSR | `07-scr1-pipe-csr.md` | 1169 | 特权状态 + trap 进出 + 计数器 |
| 8 | Pipe Top | `08-scr1-pipe-top.md` | 825 | 流水线顶层（9 子模块 + glue logic） |

## 中断与调试（5 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 9 | IPIC | `09-scr1-ipic.md` | 605 | 中断控制器（优先编码 + 服务中跟踪） |
| 10 | TDU | `10-scr1-tdu.md` | 610 | 硬件触发器（4 个 MCONTROL + 事件匹配） |
| 11 | HDU | `11-scr1-hdu.md` | 904 | 调试单元（FSM + PBUF + 调试 CSR） |
| 12 | Tracelog | `12-scr1-tracelog.md` | 453 | 轨迹日志（PC/异常/中断快照输出） |
| 13 | Core Top | `13-scr1-core-top.md` | 521 | 核心顶层（复位双路径 + 调试子系统装配） |

## JTAG 与调试链（4 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 14 | TAPC | `14-scr1-tapc.md` | 457 | JTAG TAP 控制器（16 态 FSM + IR/DR） |
| 15 | DMI | `15-scr1-dmi.md` | 182 | DMI 桥（TAPC↔DM 两步协议） |
| 16 | DM | `16-scr1-dm.md` | 1427 | Debug Module（抽象命令 FSM + DHI） |
| 17 | SCU | `17-scr1-scu.md` | 516 | 系统控制单元（四路复位 + RDC qualifier） |

## 时钟与原语（4 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 18 | Clk Ctrl | `18-scr1-clk-ctrl.md` | 55 | 时钟控制器（复位路径 + 门控 FSM + ICG） |
| 19 | CG | `29-scr1-cg.md` | 32 | 时钟门控单元（锁存器抗毛刺） |
| 20 | Reset Cells | `30-scr1-reset-cells.md` | 231 | 7 个复位原语（buf/sync/qlfy/and/mux） |
| 21 | TAPC Shift Reg | `31-scr1-tapc-shift-reg.md` | 112 | 参数化 JTAG 移位寄存器 |
| 22 | TAPC Sync | `32-scr1-tapc-synchronizer.md` | 183 | TCK↔SysCLK CDC（二分频 + 边沿检测） |

## 存储桥与路由器（6 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 23 | Imem AHB | `19-scr1-imem-ahb.md` | 319 | 指令 AHB 桥（2 深 FIFO + 两态 FSM） |
| 24 | Dmem AHB | `20-scr1-dmem-ahb.md` | 480 | 数据 AHB 桥（FIFO + data_fifo + 字节路转换） |
| 25 | Mem AXI | `21-scr1-mem-axi.md` | 363 | AXI 桥（环形队列 + 三指针 + 四位状态） |
| 26 | Imem Router | `22-scr1-imem-router.md` | 185 | 指令路由器（地址判定 + 端口选择） |
| 27 | Dmem Router | `23-scr1-dmem-router.md` | 278 | 数据路由器（三端口优先级 + 请求分发） |
| 28 | TCM | `24-scr1-tcm.md` | 131 | TCM（单拍应答 + 字节路访问） |

## 存储与定时器（3 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 29 | DP Memory | `25-scr1-dp-memory.md` | 111 | 双端口 RAM（Intel/通用两种实现） |
| 30 | Timer | `26-scr1-timer.md` | 271 | 内存映射定时器（MTIME/MTIMECMP + RTC） |

## 系统顶层（2 个）

| # | 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|---|
| 31 | Top AHB | `27-scr1-top-ahb.md` | 513 | AHB 顶层（核心 + 存储 + 双 AHB 桥） |
| 32 | Top AXI | `28-scr1-top-axi.md` | 701 | AXI 顶层（核心 + 存储 + 双 AXI 桥复用） |

---

**总计**：32 个非 TB 模块讲解文件（`scr1_top_tb_axi.sv` 已单独成文，未计入）。

所有文件位于 `docs/module-walkthroughs/` 目录。
