# 用户指令记忆

本文件记录了用户的指令、偏好和教导，用于在未来的交互中提供参考。

## 格式

### 用户指令条目
用户指令条目应遵循以下格式：

[用户指令摘要]
- Date: [YYYY-MM-DD]
- Context: [提及的场景或时间]
- Instructions:
  - [用户教导或指示的内容，逐行描述]

### 项目知识条目
Agent 在任务执行过程中发现的条目应遵循以下格式：

[项目知识摘要]
- Date: [YYYY-MM-DD]
- Context: Agent 在执行 [具体任务描述] 时发现
- Category: [运维部署|构建方法|测试方法|排错调试|工作流协作|环境配置]
- Instructions:
  - [具体的知识点，逐行描述]

## 去重策略
- 添加新条目前，检查是否存在相似或相同的指令
- 若发现重复，跳过新条目或与已有条目合并
- 合并时，更新上下文或日期信息
- 这有助于避免冗余条目，保持记忆文件整洁

## 条目

### SCR1 仿真环境构建与运行要点
- Date: 2026-08-01
- Context: Agent 复现 SCR1 的 Verilator 仿真流程，构建快速版（无波形）仿真用于学习迭代时发现
- Category: 构建方法 / 排错调试
- Instructions:
  - 工具链位置：RISC-V 交叉编译器在 `/usr/bin/riscv64-unknown-elf-gcc`，Verilator 在 `/usr/bin/verilator`（5.006 版本），均为系统安装
  - 构建测试：在项目根目录运行 `make TARGETS=<test> run_verilator`（快速版，秒级完成），不要用 `run_verilator_wf`（波形版，会生成 1.7GB+ 的 VCD 且极慢）
  - 必须移除 `sim/tests/common/common.mk` 第 15 行的 `--specs=nano.specs` 才能链接成功——picolibc 环境不支持该 spec 文件，报错 `cannot read spec file 'nano.specs'`。此为环境适配 hack，不提交 git
  - 仿真运行产物在 `build/verilator_AHB_MAX_imc_IPIC_1_TCM_1_VIRQ_1_TRACE_0/`，可执行文件为 `verilator/Vscr1_top_tb_ahb`，测试列表由 `+test_info=` 参数指向
  - 用 `make TARGETS=hello run_verilator` 验证环境：应输出 "Hello from SCR1!" 且 Summary: 1/1 tests passed
