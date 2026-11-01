# scr1_dm.sv 讲解

## 一句话职责

Debug Module：RISC-V 调试规范实现——寄存器映射 + 抽象命令 FSM + 程序缓冲区 + DHI（Debug Module Interface）。

## 文件信息

- **路径**: `src/core/scr1_dm.sv`
- **行数**: 1427 行
- **配置依赖**: `SCR1_DBG_EN`

## 核心架构

### 1. 寄存器映射（`:434-448`）

| 寄存器 | 地址 | 作用 |
|---|---|---|
| `DMCONTROL` | 0x10 | 调试控制（haltreq、dresumereq、dmactive） |
| `DMSTATUS` | 0x11 | 调试状态（halted、running、version） |
| `HARTINFO` | 0x12 | Hart 信息（nscratch、dataaddr） |
| `ABSTRACTCS` | 0x16 | 抽象命令控制 |
| `COMMAND` | 0x17 | 抽象命令（读/写/访问寄存器） |
| `DATA0-3` | 0x04-07 | 数据寄存器（抽象命令参数） |
| `PROGBUF0-15` | 0x20-2F | 程序缓冲区（16 条指令） |
| `DSCRATCH0-1` | 0x40-41 | 调试临时寄存器 |

### 2. 抽象命令 FSM（13 状态，`:90-104`/`:804-903`）

| 状态 | 含义 | 转移 |
|---|---|---|
| `IDLE` | 空闲 | `cmd_valid` → DECODE |
| `DECODE` | 命令译码 | 根据 type → 各执行状态 |
| `EXEC_ACCESS_REGISTER` | 访问寄存器 | 读/写 x1~x31 |
| `EXEC_QUICK_ACCESS` | 快速访问 | 特殊寄存器 |
| `EXEC_MEMORY` | 内存访问 | LOAD/STORE |
| `EXEC_CSR` | CSR 访问 | 读/写 CSR |
| `...` | ... | ... |
| `CMD_DONE` | 命令完成 | 写 `abstractcs.cmderr` → IDLE |

**命令格式**（`COMMAND` 寄存器）：
```
[31:24] type: 0=访问寄存器，1=快速访问，2=内存，3=CSR
[23:20] regno: 寄存器号（x0~x31 或 CSR）
[19:16] size: 访问宽度（0=8 位，1=16 位，2=32 位）
[15:12] control: 后增量/符号扩展
```

### 3. 内存访问（`:707-771`）

**绝对地址访问**（Abstract Command）：
```systemverilog
// 解析命令
addr = DATA1;        // 内存地址
data = DATA0;        // 写数据（或读回）
size = COMMAND[19:16];  // 字节/半字/字

// 生成 LSU 请求
dmem_req = 1'b1;
dmem_cmd = (is_write) ? SCR1_MEM_CMD_WR : SCR1_MEM_CMD_RD;
dmem_width = size;
dmem_addr = addr;
```

**限制**：`memsize<3`（`:754`）——只支持字访问，不支持块传输。

### 4. DHI FSM（7 状态，`:106-114`/`:1244-1302`）

DHI（Debug Module Interface）是 DM→CPU 核的私有接口：

| 状态 | 含义 |
|---|---|
| `IDLE` | 空闲 |
| `HALT` | 请求停机 |
| `RESUME` | 请求恢复 |
| `STEP` | 单步执行 |
| `READ_REG` | 读核寄存器 |
| `WRITE_REG` | 写核寄存器 |
| `READ_PC` | 读 PC |

**响应协议**（`:1301-1302`）：
```systemverilog
assign dhi_resp = exec_halt & dhalt;  // 停机确认
assign dhi_resp_exc = ...;            // 异常标志
```

### 5. 程序缓冲区执行（`:1360-1376`）

```systemverilog
// PROGBUF mux
assign pipe_instr = (progbuf_en) ? progbuf[pc[3:0]] : 32'h00300073;  // EBREAK 默认
```

**执行流程**：
1. 调试器写 `PROGBUF[0..N]`
2. 写 `DMCONTROL.go=1`
3. DM 从 `PROGBUF[0]` 开始"取指"
4. 执行到 `EBREAK`（`32'h00300073`）停止
5. `halted=1`

**关键技巧**（`:1018-1045`）：DSCRATCH0 用 trampoline 指令（`csrrw x5, DSCRATCH0, x5`）实现寄存器交换。

### 6. cmderr 错误码（`:116-122`/`:1144-1149`）

| 错误码 | 含义 |
|---|---|
| `0` | 无错误 |
| `1` | 超时 |
| `2` | 忙（命令在飞） |
| `3` | 非法命令 |
| `4` | 其他错误 |

**busy 标志**（`:1149`）：
```systemverilog
assign cmderr_busy = ~idle & ~err;  // 非空闲且无错 = 忙
```

### 7. HART 状态跟踪（`:667-694`）

| 标志 | 含义 |  sticky |
|---|---|---|
| `havereset` | Hart 经历过复位 | 是（写 1 清） |
| `halted` | Hart 已停机 | 否 |
| `resumeack` | Hart 已恢复 | 是（写 1 清） |

**havereset_skip_pwrup**（`:652-665`）：上电复位不触发 `havereset`（避免调试器误判）。

### 8. dmactive 全局使能（`:635-639`/`:682-694`）

```systemverilog
if (~dmactive) begin
    // dmactive=0 时清零所有状态
    halted <= 1'b0;
    cmderr <= 5'b0;
    // ...
end
```

**dmactive 写 1 后需等待**（`:595-605`）：
```systemverilog
always_ff @(posedge clk) begin
    if (dmactive_w) begin
        clk_en_dm <= 1'b1;  // 自保持，直到 dmactive=0
    end
end
```

## 关键信号

| 信号 | 方向 | 含义 |
|---|---|---|
| `dmi2dm_req_i` | in | DMI 请求 |
| `dmi2dm_addr_i` | in | DM 寄存器地址 |
| `dmi2dm_data_i` | in | 写数据 |
| `dm2dmi_resp_o` | out | DMI 响应 |
| `dm2dmi_rdata_o` | out | 读回数据 |
| `dm2hdu_active_o` | out | 调试活跃标志 |
| `dm2hdu_halt_req_o` | out | 停机请求 |
| `dm2hdu_resume_req_o` | out | 恢复请求 |
| `dm2hdu_progbuf_o` | out | PROGBUF 内容 |
| `dhi2dm_resp_i` | in | DHI 响应 |

## 调试器完整流程

```
调试器 → DMI 写 DMCONTROL.haltreq → DM 发 DHI 停机请求
→ HDU 停流水线 → DMSTATUS.halted=1
→ 调试器 DMI 读寄存器 → DM 执行抽象命令
→ 调试器 DMI 写 PROGBUF → DM 执行指令序列
→ 调试器 DMI 写 DMCONTROL.resumereq → 恢复执行
```

## 配置宏影响

| 宏 | 行为变化 |
|---|---|---|
| `SCR1_DBG_EN` | DM 模块使能 |

## 配套文档

- 设计规格：`docs/scr1-dm/design-spec.md`
- 验证文档：`docs/scr1-dm/verification.md`
