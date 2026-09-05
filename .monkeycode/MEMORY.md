# User Instruction Memory

This file records user instructions, preferences, and teachings for reference in future interactions.

## Format

### User Instruction Entry
User instruction entries should follow this format:

[User Instruction Summary]
- Date: [YYYY-MM-DD]
- Context: [Mentioned scenario or time]
- Instructions:
  - [Content of user teaching or instruction, described line by line]

### Project Knowledge Entry
Entries discovered by the Agent during task execution should follow this format:

[Project Knowledge Summary]
- Date: [YYYY-MM-DD]
- Context: Discovered by Agent while performing [specific task description]
- Category: [Operations & Deployment|Build Methods|Testing Methods|Troubleshooting & Debugging|Workflow & Collaboration|Environment Configuration]
- Instructions:
  - [Specific knowledge points, described line by line]

## Deduplication Strategy
- Before adding a new entry, check for similar or identical instructions.
- If a duplicate is found, skip the new entry or merge it with the existing one.
- When merging, update the context or date information.
- This helps avoid redundant entries and keeps the memory file tidy.

## Entries

[Project Knowledge Summary]
- Date: 2026-08-25
- Context: Discovered by Agent while setting up Verilator simulation for SCR1 hello test
- Category: Environment Configuration
- Instructions:
  - RISC-V 工具链为 xPack 预编译版 riscv-none-elf-gcc 15.2.0，位于 /opt/toolchain/xpack-riscv-none-elf-gcc-15.2.0-1/bin，需加入 PATH 使用；源码编译在低配置环境不可行。
  - Verilator 5.006 通过 `apt install verilator` 安装，Debian 12 有现成包。
  - SCR1 仿真用 MIN 配置（SCR1_CFG_RV32EC_MIN，RTL 最小）可最快跑通，产物在 build/verilator_AHB_MIN_ec_IPIC_0_TCM_1_VIRQ_0_TRACE_0。
  - 编译 hello 需 EXT_CFLAGS=-D__RVE_EXT（RV32E 时 crt_tcm.S 只处理 x1-x15）；链接须 ADD_LDFLAGS="-lc -lnosys -Wl,--defsym,end=_end"（新版工具链 emutls 链接顺序 + link_tcm.ld 缺 end 符号）。
  - 跑通流程：先 `make -C sim/tests/hello ...` 编 hello.hex，再 `make -C sim build_verilator ...` 建仿真模型，写入 test_info 后运行 ./verilator/Vscr1_top_tb_ahb +test_info +test_results。
  - 详细命令与坑见 docs/learning-notes.md 第 9 课。
