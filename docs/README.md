# SCR1 RISC-V Core 文档导航

本文档是 SCR1 RISC-V 处理器核心完整文档体系的入口，按学习路径和使用场景组织。

---

## 快速开始

**我是新手，第一次接触 SCR1** → 从 [`docs/learning-notes.md`](./learning-notes.md) 开始

**我已经了解 SCR1，需要查某个模块的细节** → 跳转 [`docs/module-walkthroughs/`](./module-walkthroughs/)

**我需要运行仿真/回归测试** → 查看 [`docs/verification-guide.md`](./verification-guide.md)

**我需要查指令编码/CSR 映射/内存地图** → 翻阅 [`docs/quick-reference.md`](./quick-reference.md)

**我想看整体数据通路/架构全景** → 浏览 [`docs/data-path-overview.md`](./data-path-overview.md)

---

## 文档体系总览

### 1. 学习笔记（1 份，约 1300 行）

**文件**: [`docs/learning-notes.md`](./learning-notes.md)

**内容**:
- RISC-V ISA 基础 + CPU 执行原理
- 流水线概念（2~4 级可配置）
- 11 课系统讲解（IFU→IDU→EXU→CSR→LSU→MPRF→仿真实战）
- MIN vs MAX 配置对比实测
- VCD 波形分析与指令流重建

**适合人群**: 初学者，希望系统学习 SCR1 架构

**前置知识**: 数字电路基础，了解触发器/FSM/流水线概念

---

### 2. 模块讲解（32 份，约 4300 行）

**目录**: [`docs/module-walkthroughs/`](./module-walkthroughs/)

**组织结构**:

| 分组 | 模块数 | 文件编号 |
|---|---|---|
| 流水线核心 | 8 | 01-08 |
| 中断与调试 | 5 | 09-13 |
| JTAG 与调试链 | 4 | 14-17 |
| 时钟与原语 | 5 | 18, 29-32 |
| 存储桥与路由器 | 6 | 19-24 |
| 存储与定时器 | 3 | 25-27 |
| 系统顶层 | 2 | 28-29 |

**每份讲解包含**:
- 一句话职责 + 文件信息 + 配置依赖
- 核心架构分解（带行号引用的代码片段）
- 关键信号表
- 配置宏影响
- 与其他模块关系图
- 配套设计/验证文档索引

**适合人群**: 需要深入了解特定模块的开发者

**使用建议**: 配合 `learning-notes.md` 的全景描述使用，先整体后局部

---

### 3. 数据通路全景图（1 份，7 张 Mermaid 图）

**文件**: [`docs/data-path-overview.md`](./data-path-overview.md)

**图表列表**:
1. 指令执行主通路（IFU→IDU→EXU→MPRF→CSR）
2. 中断/异常 Trap 流程（时序图）
3. 调试访问链路（JTAG→TAPC→DMI→DM→HDU）
4. 存储访问层次（LSU→Router→TCM/Timer/AHB/AXI）
5. 复位层次（Sys⊃Core⊃HDU⊃DM，Qualifier 早一拍）
6. 时钟门控（ClkCtrl→ICG 抗毛刺原理）
7. 完整 SoC 视图（AHB/AXI 顶层 + 外部总线）

**适合人群**: 需要全局视角的架构师/验证工程师

**使用场景**: 设计评审/代码审查/问题定位

---

### 4. 快速参考手册（1 份，7 个附录）

**文件**: [`docs/quick-reference.md`](./quick-reference.md)

**附录列表**:
- **A**: RISC-V ISA 指令编码速查表（R/I/S/B/U/J 型 + RVC）
- **B**: SCR1 配置矩阵（MAX/BASE/MIN/CUSTOM + 宏开关）
- **C**: CSR 寄存器映射（只读/Trap/计数器/调试）
- **D**: 异常/中断码表（MCAUSE 编码）
- **E**: 调试寄存器映射（DM 地址空间）
- **F**: 内存映射（TCM/Timer/AHB/AXI）
- **G**: JTAG IR 编码 + IDCODE 格式

**适合人群**: 所有用户（案头必备速查表）

**使用场景**: 写汇编/读反汇编/调试 CSR/查异常码

---

### 5. 验证与回归测试指南（1 份）

**文件**: [`docs/verification-guide.md`](./verification-guide.md)

**内容**:
- 工具链安装（Verilator/RISC-V GCC/其他仿真器）
- 构建 Verilator 模型（MAX/MIN/带波形）
- 编译测试程序（Hello/ISA/CoreMark）
- 运行仿真流程 + 预期输出
- 回归测试脚本 + 覆盖率统计
- 常见错误排查表（编译/仿真/波形）
- 性能基准（Hello/CoreMark）
- 验证覆盖点清单（指令/边界情况）

**适合人群**: 验证工程师/需要运行仿真的开发者

**前置知识**: 熟悉 Linux 命令行，了解 Makefile 基本用法

---

### 6. 配套设计与验证文档（66 份）

**目录**: `docs/scr1-*/design-spec.md` + `docs/scr1-*/verification.md`

**覆盖模块**:
- `scr1-pipe-ifu`, `scr1-pipe-idu`, `scr1-pipe-exu`, `scr1-pipe-lsu`, `scr1-pipe-ialu`, `scr1-pipe-mprf`, `scr1-pipe-csr`, `scr1-pipe-top`
- `scr1-ipic`, `scr1-tdu`, `scr1-hdu`, `scr1-tracelog`
- `scr1-core-top`, `scr1-tapc`, `scr1-dmi`, `scr1-dm`, `scr1-scu`, `scr1-clk-ctrl`
- `scr1-imem-ahb`, `scr1-dmem-ahb`, `scr1-mem-axi`, `scr1-imem-router`, `scr1-dmem-router`
- `scr1-tcm`, `scr1-dp-memory`, `scr1-timer`
- `scr1-top-ahb`, `scr1-top-axi`
- `scr1-cg`, `scr1-reset-cells`, `scr1-tapc-shift-reg`, `scr1-tapc-synchronizer`

**每份文档包含**:
- **设计规格**: 接口定义/微架构/时序图/配置宏影响/SVA 断言
- **验证文档**: 覆盖点 (FC-*)/断言核对/回归流程/结论

**适合人群**: 模块负责人/验证工程师/代码审查者

**使用场景**: 模块级验证/断言覆盖率检查/回归测试

---

## 学习路径推荐

### 路径 A：系统学习（适合新手，约 40 小时）

```
第 1 周: learning-notes.md 第 1-6 课 (RISC-V 基础 + 流水线核心)
       ↓
第 2 周: learning-notes.md 第 7-11 课 (存储/调试/仿真实战)
       ↓
第 3 周: module-walkthroughs/ 01-08 (流水线核心模块深入)
       ↓
第 4 周: verification-guide.md + 跑通 Hello/ISA 回归测试
```

**产出**: 能独立修改 SCR1 代码 + 运行完整回归测试

---

### 路径 B：快速上手（适合有 RISC-V 经验者，约 10 小时）

```
第 1 天: learning-notes.md 第 1-2 课 (快速回顾) + data-path-overview.md
       ↓
第 2 天: module-walkthroughs/ 按需阅读（根据修改的模块）
       ↓
第 3 天: quick-reference.md 附录 A/B/C (指令/配置/CSR) + 跑仿真
```

**产出**: 能快速定位代码 + 修改特定模块

---

### 路径 C：验证导向（适合验证工程师，约 20 小时）

```
第 1 周: verification-guide.md (工具链 + 回归流程)
       ↓
第 2 周: 配套验证文档 (66 份 verification.md) + 补充覆盖点
       ↓
第 3 周: data-path-overview.md + module-walkthroughs/ 理解架构
```

**产出**: 能独立编写验证计划 + 运行覆盖分析

---

## 文档更新记录

| 日期 | 更新内容 | Commit |
|---|---|---|
| 2026-02-05 | 创建完整文档体系（学习笔记 +32 模块讲解 + 全景图 + 速查表 + 验证指南） | `18125ca` |
| 2026-02-05 | 追加数据通路全景图（7 张 Mermaid 图） | `8a472e5` |
| 2026-02-05 | 追加快速参考手册（7 个附录） | `ae3f734` |
| 2026-02-05 | 追加验证指南 | `18125ca` |

---

## 反馈与贡献

如发现文档错误或需要补充内容，请：
1. 在 GitHub 提交 Issue
2. 或直接提交 PR 修改 `docs/` 下对应文件

---

## 许可证

本文档与 SCR1 RTL 代码使用相同许可证（详见项目根目录 `LICENSE` 文件）。

---

*文档版本：v1.0 | 最后更新：2026-02-05 | 适用于 SCR1 RV32IMC 配置*
