# scr1_hdu.sv 讲解

## 一句话职责

调试单元：调试 FSM + 程序缓冲区（PROGBUF）+ 调试 CSR + 停机原因与优先级。

## 文件信息

- **路径**: `src/core/scr1_hdu.sv`
- **行数**: 904 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心功能

### 1. 调试 FSM（`:200-350`）

| 状态 | 含义 | 转移条件 |
|---|---|---|
| `DM_ACTIVE` | 调试模块活跃 | `dmactive=1` |
| `HALT_REQ` | 请求停机 | `dresumereq=0` 或 `haltreq=1` |
| `HALTED` | 已停机 | `pipe_halted=1` |
| `RUNNING` | 正常运行 | `dresumereq=1` |
| `EXCEPTION` | 调试异常 | `dexception=1` |

**关键信号**：
- `dmactive`：调试模块使能（写 DCSR 开启）
- `haltreq`：外部请求停机（写 DCSR）
- `dresumereq`：恢复执行（写 DCSR）
- `pipe_halted`：流水线确实停了（IFU/IDU/EXU 全 idle）

### 2. 程序缓冲区 PROGBUF（`:400-500`）

```systemverilog
localparam SCR1_PROGBUF_SIZE = 16;  // 16 个 32 位字
logic [31:0] progbuf [SCR1_PROGBUF_SIZE];
```

**用途**：调试器写入指令序列，DM 读取并执行（单步/内存访问）。

**执行流程**：
1. 调试器写 `progbuf[0..N]`
2. 写 `dmcontrol.go=1`
3. DM 从 `progbuf[0]` 开始取指执行
4. 执行到 `EBREAK` 停止 → `halted=1`

### 3. 调试 CSR（`:550-650`）

| 寄存器 | 地址 | 作用 |
|---|---|---|
| `DCSR` | 0x7b0 | 调试控制（dstep、dexception、dmode） |
| `DPC` | 0x7b1 | 调试 PC（停机时保存 PC） |
| `DSCRATCH0/1` | 0x7b2/3 | 调试临时寄存器 |

**DCSR 关键字段**：
- `dstep`：单步执行（执行一条后停机）
- `dexception`：异常时进入调试模式
- `dmode`：进入调试模式的原因

### 4. 停机原因与优先级（`:700-800`）

| 原因 | 优先级 | 说明 |
|---|---|---|
| `haltreq` | 最高 | 外部写 DCSR.haltreq |
| `breakpoint` | 2 | PC 触发器命中 |
| `watchpoint` | 3 | 地址触发器命中 |
| `dmode` | 4 | 调试模式请求 |
| `single_step` | 5 | DCSR.dstep=1 |

优先级决定 `halt_cause` 寄存器内容（调试器可读）。

### 5. 与流水线握手

```systemverilog
// 停机请求
hdu2pipe_halt_req_o = haltreq | breakpoint | watchpoint;

// 流水线确认
pipe_halted = ~ifu_busy & ~idu_busy & ~exu_busy;

// 恢复执行
pipe2hdu_resume_o = dresumereq & ~haltreq;
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `dm_hdu_active_i` | in | DM 活跃 |
| `hdu2pipe_halt_req_o` | out | 停机请求 |
| `hdu2pipe_dmode_o` | out | 调试模式 |
| `pipe2hdu_curr_pc_i` | in | 当前 PC |
| `pipe2hdu_halted_i` | in | 流水线已停机 |
| `hdu2dm_progbuf_o` | out | PROGBUF 内容 |
| `dm2hdu_progbuf_we_i` | in | PROGBUF 写使能 |

## 调试会话完整流程

```
调试器 → JTAG → TAPC → DMI → DM → HDU
→ HDU 发 halt_req → 流水线停取
→ 调试器读寄存器/写 PROGBUF → DM 执行
→ 调试器发 resume → HDU 清除 halt_req → 流水线恢复
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | HDU 模块使能 |
| `SCR1_TDU_EN` | TDU 提供 breakpoint/watchpoint 输入 |

## 配套文档

- 设计规格：`docs/scr1-hdu/design-spec.md`
- 验证文档：`docs/scr1-hdu/verification.md`
