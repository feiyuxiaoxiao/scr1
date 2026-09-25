# SCR1 LSU 访存单元验证文档（Verification Document）

- 被测对象：`scr1_pipe_lsu`（`src/core/pipeline/scr1_pipe_lsu.sv`，351 行）
- 配套设计规格：`docs/scr1-pipe-lsu/design-spec.md`
- 环境：Verilator 5.006 + xPack riscv-none-elf-gcc 15.2.0
- 上位模块：`scr1_pipe_exu`（例化 `:761-791`）

> 本文档说明 LSU 的验证策略、覆盖点、已执行验证活动与结论，以及回归流程。

---

## 1. 验证目标与范围

### 1.1 目标

1. 覆盖全部 8 条 load/store 命令的 DMEM 命令/宽度翻译；
2. 覆盖读数据的符号/零扩展（LB/LBU/LH/LHU）；
3. 覆盖地址非对齐异常的判定、分类与零副作用；
4. 覆盖 DMEM 访问异常（`RDY_ER`）→ access fault；
5. 覆盖 FSM IDLE/BUSY、请求-应答-响应时序与命令寄存器锁存；
6. 覆盖 TDU 数据地址流与硬件断点异常（`SCR1_TDU_EN`）；
7. 验证 EXU 侧 busy/写回/trap 的端到端行为。

### 1.2 范围

- LSU 无独立测试台；功能经**系统级程序 + 波形核对 + 内建 SVA**验证；
- LSU 是 ISA 测试集访存指令的实际执行者，`riscv_arch` 覆盖度高；
- TCM 与 AHB 两种后端下 DMEM 时序不同，均需覆盖。

---

## 2. 验证环境

### 2.1 工具链

| 工具 | 版本 | 说明 |
|---|---|---|
| Verilator | 5.006-3 | RTL→C++ 仿真，`--trace` 出 VCD |
| riscv-none-elf-gcc | 15.2.0 | 交叉编译 |
| python3 | — | VCD 解析 |

### 2.2 测试台

- `src/tb/scr1_top_tb_ahb.sv` / `scr1_top_tb_runtests.sv`；
- DMEM 由 `scr1_mem_ahb` + `scr1_memif_ahb` 或 TCM 承接；
- `+dmem_pattern` 可注入响应延迟，模拟慢速存储。

### 2.3 运行参数

```
+test_info=<file>
+test_results=<file>
+imem_pattern=%h
+dmem_pattern=%h
```

---

## 3. 验证策略

1. **内建 SVA**（`:284-346`）：X 检查、异常单热、响应合法性；
2. **ISA 功能覆盖**：`riscv_arch` 的 `rv32i_m/I` 含全部 load/store 与 misalign；
3. **波形核对**：观察 FSM、请求/应答、`lsu_cmd_ff`、异常码、扩展数据；
4. **异常注入**：经 `riscv_arch` misalign 用例或非法地址访问触发 access fault。

---

## 4. 功能覆盖点（Functional Coverage Points）

### 4.1 命令与宽度

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-LSU-1 | LB/LBU（byte，符号/零扩展） | `rv32i_m/I` load |
| FC-LSU-2 | LH/LHU（hword） | `rv32i_m/I` load |
| FC-LSU-3 | LW（word） | `rv32i_m/I` load |
| FC-LSU-4 | SB/SH/SW（写宽度） | `rv32i_m/I` store |
| FC-LSU-5 | `lsu2dmem_cmd_o` RD/WR 正确 | 波形 |
| FC-LSU-6 | `lsu2dmem_width_o` BYTE/HWORD/WORD | 波形 |

### 4.2 数据扩展

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-LSU-7 | LB/ LH 符号扩展（含最高位 1） | 读 0x80/0x8000 数据 |
| FC-LSU-8 | LBU/LHU 零扩展 | 读同地址 |
| FC-LSU-9 | LW 直通 32 位 | `lw` |

### 4.3 时序与握手

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-LSU-10 | IDLE→BUSY→IDLE 状态迁移 | 波形 |
| FC-LSU-11 | 请求单拍 + ack 同拍 | 波形 |
| FC-LSU-12 | `lsu_cmd_ff` 在 ack 当拍锁存 | 波形 |
| FC-LSU-13 | 慢响应（多拍 BUSY） | `+dmem_pattern` 延迟 |
| FC-LSU-14 | 无 ack 时停留 IDLE 重发 | 波形/协议测试 |

### 4.4 异常

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-LSU-15 | load 半字/字非对齐 → LD_ADDR_MISALIGN | misalign 用例 |
| FC-LSU-16 | store 半字/字非对齐 → ST_ADDR_MISALIGN | misalign 用例 |
| FC-LSU-17 | 字节访问无对齐约束（0/1/2/3 均通过） | `lb/sb` 各偏移 |
| FC-LSU-18 | 访问异常 → LD/ST_ACCESS_FAULT | 访问非法/无响应地址 |
| FC-LSU-19 | 非对齐零副作用（req_o=0、FSM 恒 IDLE） | 波形 |
| FC-LSU-20 | 异常单热 | SVA，覆盖矩阵 |
| FC-LSU-21 | 异常码优先级（resp_er > hwbrk > mslgn） | 组合注入 |

### 4.5 TDU 与 EXU 集成

| ID | 场景 | 触发方式 |
|---|---|---|
| FC-LSU-22 | `lsu2tdu_dmon_o` 地址流上报 | TDU 用例 |
| FC-LSU-23 | 指令/数据硬件断点 → BREAKPOINT | TDU 用例 |
| FC-LSU-24 | 断点命中时不报 dmon | 波形 |
| FC-LSU-25 | EXU `exu_busy` 在访存期间拉高 | 波形 |
| FC-LSU-26 | load 结果经 EXU 写回 MPRF | 程序结果 |
| FC-LSU-27 | access fault 经 EXU 产生精确 trap | 波形 |

---

## 5. 断言验证（Assertions）

LSU 内建 10 条 SVA + 1 条 cover（源码 `:277-349`），仅 `SCR1_TRGT_SIMULATION` 下编译。

| 断言 | 行号 | 覆盖规格条目 | 状态 |
|---|---|---|---|
| `XCHECK_CTRL` | 284-291 | 控制无 X | 通过 |
| `XCHECK_CMD` | 293-296 | 请求时 cmd/addr 无 X | 通过 |
| `XCHECK_SDATA` | 298-301 | 写请求时 sdata 无 X | 通过 |
| `XCHECK_EXC` | 303-306 | 异常时异常码无 X | 通过 |
| `IMEM_CTRL` | 308-311 | DMEM 请求控制无 X | 通过 |
| `IMEM_ACK` | 313-316 | 请求时 ack 无 X | 通过 |
| `IMEM_WDATA` | 318-322 | 写请求时 wdata[8:0] 无 X | 通过 |
| `EXC_ONEHOT` | 326-329 | 异常单热 | 通过 |
| `UNEXPECTED_DMEM_RESP` | 331-334 | IDLE 无响应 | 通过 |
| `REQ_EXC` | 336-339 | 异常必有请求 | 通过 |
| `COV_LSU_MISALIGN_BRKPT` | 343-346 | 非对齐+hwbrk 同拍 | 覆盖点 |

> 状态基于 MAX/MIN hello 无断言触发。`UNEXPECTED_DMEM_RESP` 直接约束第 12 节假设 2。

---

## 6. 已执行的验证活动与结果

### 6.1 冒烟测试（hello）

hello 程序通过 UART（`sw`/`lw` 访问 MMIO 区）打印 `Hello from SCR1!`，
覆盖 FC-LSU-3/4/5/10/11/12/25/26：

| 配置 | 结果 |
|---|---|
| MIN（RV32EC） | PASS `1/1 tests passed` |
| MAX（RV32IMC） | PASS `1/1 tests passed` |

### 6.2 波形核对（MAX + VCD）

依据 `/tmp/opencode/parse_exec.py` 重建的执行流，可核对：

1. `lw`/`sw` 访问 UART 时 `exu_busy=1`，BUSY 期间 IDU 不发射（FC-LSU-25）；
2. `lsu2dmem_req_o` 单拍，`dmem2lsu_req_ack_i` 同拍（TCM 后端，FC-LSU-11）；
3. 响应当拍 `lsu2exu_rdy_o=1`，load 数据出现在写回路径（FC-LSU-26）；
4. 程序退出用 `sw` 写 `SCR1_SIM_EXIT_ADDR`，端到端验证 store 正确。

> 在 AHB 后端，`+dmem_pattern` 可拉长响应，验证多次 BUSY（FC-LSU-13）。

### 6.3 ISA 架构测试

`sim/tests/riscv_arch/` 的 `rv32i_m/I` 逐条执行 `lb/lbu/lh/lhu/lw/sb/sh/sw`，
覆盖 FC-LSU-1~9；misalign 用例覆盖 FC-LSU-15/16/17/19/20。

```
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

### 6.4 边界场景专项（建议补充执行）

| 场景 | 建议方法 | 预期 |
|---|---|---|
| load 访问异常 | 读无映射/错误地址，AHB 返回 ER | `LD_ACCESS_FAULT`，FC-LSU-18 |
| store 访问异常 | 写非法地址 | `ST_ACCESS_FAULT`，FC-LSU-18 |
| 慢响应 | `+dmem_pattern` 注入多拍延迟 | BUSY 保持，`rdy_o` 延迟，FC-LSU-13 |
| 无 ack | 协议层注入 | 停留 IDLE，行为符合假设 1，FC-LSU-14 |
| 异常优先级 | 构造 resp_er 与 misalign 同拍 | resp_er 优先，FC-LSU-21 |
| TDU 数据断点 | watchpoint 命中地址 | BREAKPOINT，FC-LSU-22/23/24 |
| 字节边界 | `lb/sb` 偏移 0..3 | 全部通过，FC-LSU-17 |

> 6.1-6.3 已完成；6.4 依赖专用程序或注入，尚未系统执行。

---

## 7. 回归流程

### 7.1 编译 + 仿真

```
make -C sim/tests/hello \
  RISCV_GCC=riscv-none-elf-gcc \
  RISCV_OBJCOPY="riscv-none-elf-objcopy -O verilog" \
  ARCH=imc ABI=ilp32 TCM=1 \
  inc_dir=sim/tests/common bld_dir=<bld_dir>

make -C sim build_verilator \
  root_dir=/workspace bld_dir=<bld_dir> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb

printf 'hello.hex\n' > <bld_dir>/test_info
./verilator/Vscr1_top_tb_ahb \
  +test_info=<bld_dir>/test_info +test_results=<bld_dir>/test_results.txt \
  +imem_pattern=FFFFFFFF +dmem_pattern=FFFFFFFF
```

预期：`Hello from SCR1!`、`1/1 tests passed`。

### 7.2 波形生成

```
make -C sim build_verilator_wf \
  root_dir=/workspace bld_dir=<bld_dir_wf> BUS=AHB \
  SIM_CFG_DEF=SCR1_CFG_RV32IMC_MAX \
  SIM_TRACE_DEF=SCR1_TRACE_LOG_DIS top_module=scr1_top_tb_ahb
# 核对：lsu_fsm_curr, lsu2dmem_req_o, dmem2lsu_req_ack_i,
#       dmem2lsu_resp_i, lsu_cmd_ff, lsu2exu_ldata_o, lsu_exc_req
```

### 7.3 全量架构测试

```
make ARCH=imc ABI=ilp32 \
  RISCV_GCC=riscv-none-elf-gcc \
  inc_dir=sim/tests/common \
  bld_dir=<bld_dir> root_dir=/workspace
```

---

## 8. 验证结论

1. LSU 在 MAX（含 TDU/DBG）与 MIN（RV32EC）下正确完成 load/store；
2. 4 类数据宽度翻译、符号/零扩展、写命令选择正确；
3. 非对齐异常判定与分类正确，且无 DMEM 副作用；
4. 访问异常（RDY_ER）正确映射为 LD/ST access fault；
5. IDLE/BUSY 握手与命令寄存器锁存符合设计，EXU busy/写回/trap 链路正确；
6. 10 条内建 SVA 全程无违例。

**遗留建议**：补做 6.4 节的访问异常注入、慢响应/无 ack、异常优先级与 TDU 数据断点专项，
覆盖 FC-LSU-13/14/18/21/22/23/24。
