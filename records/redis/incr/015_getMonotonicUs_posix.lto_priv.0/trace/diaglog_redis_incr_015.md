# RISC-V 性能分析报告：getMonotonicUs_posix.lto_priv.0

**Functions under analysis: [getMonotonicUs_posix.lto_priv.0]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单

| Evidence | 状态 |
|---|---|
| 单函数完整 annotate | 已提供（`015-getMonotonicUs_posix.lto_priv.0-annotate.txt`，71 行，函数 0x2001ca–0x200244，4 samples，event=`cpu-clock:u`，percent=local period） |
| perf stat | 已提供（仅吞吐/延迟指标，无 counter，详见 Phase 1） |
| workload/binary/source context | 已提供（`src/monotonic.c:196-204` getMonotonicUs_posix；dispatch `src/monotonic.c:221-237`；RISC-V mtime 路径 `src/monotonic.c:141-194` 受 `USE_PROCESSOR_CLOCK` 门控（源码注释中未定义）；调用点 `src/ae.c:228/279/291/313/338`、`src/server.c:1599-2192` 等 54 处） |
| readelf -A（build ISA） | 已提供（`rv64i2p1_..._v1p0_zvl128b1p0`，含 `zicsr2p0`） |
| hardware ISA | 已提供（`rv64imafdcv_..._zicntr_...`——time CSR 用户态可读；C920v2） |
| vlenb | 已提供（VLEN=128 bits） |
| 采样元数据 | 部分（`baseline_gap: sampling metadata`） |
| Sampling IP precision | 缺失（`baseline_gap: sampling IP precision`） |

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zicntr_..._sstc`（含 `v`、**`zicntr`**；用户态 `csrr time` 可读 mtime） |
| Build ISA | `rv64i2p1_..._v1p0_zicsr2p0_...`（含 `zicsr`；`csrr` 可用） |
| Vector flavor | 全 scalar、零 `v*`；无 flavor mismatch |
| VLEN | 128 bits |
| Bound type | `baseline_gap: bound type` |
| Sampling semantics | event=`cpu-clock`；percent=local period；同一窗口；函数 workload 级贡献未知 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（4/4 样本聚集 prologue/epilogue 为 syscall 周围 skid 特征） |

L0 gates：hardware 有 `v` ∧ build 有 `v` → 无 mismatch；无 `th.v*`。关键 baseline 观察：hardware 暴露 `zicntr`（用户态 `time` CSR），build 含 `zicsr2p0`（`csrr` 指令可用），但源码中 RISC-V mtime 快速路径（`monotonic.c:141-194`）被 `USE_PROCESSOR_CLOCK` 宏门控且该宏在源码注释块内（`monotonic.c:26-30`）未定义 → 快速路径未编译，dispatch 落入 POSIX 回退。

## Phase 2 — Scope / 分析边界

函数清单：**[getMonotonicUs_posix.lto_priv.0]**（rank 015，LTO 私有化版本）。POSIX 单调时钟回退实现（`monotonic.c:196-204`）：stack canary 设置 + `clock_gettime(CLOCK_MONOTONIC)` **syscall**（经 PLT）+ `tv_sec*1000000 + tv_nsec/1000` 换算 + canary 检查。无循环。

hot interval：**整个函数**——成本主体是 `clock_gettime@plt` syscall（2001ea/2001ee），样本在 syscall 周围 prologue/epilogue 聚集（skid）。trace anchors：
- `50.00 : 2001cc: sd s0,48(sp)`（prologue，skid）
- `25.00 : 2001ce: sd s1,40(sp)`（prologue，skid）
- `25.00 : 200224: ld a2,0(s1)`（epilogue canary 重载）

annotate 覆盖完整；IP precision 未确认 → interval 层归因。**incr workload 事实**：`getMonotonicUs` 在 ae 事件循环每迭代调用多次（`ae.c:228/279/291/313/338` 定时器处理、`server.c:2141/2192` 每迭代 el 计时），每次调用在 POSIX 回退下都付一次完整 syscall。

## Phase 3 — Pattern scan / 模式扫描：getMonotonicUs_posix.lto_priv.0

### Class selection trace（8 项）

| Class | include/exclude | 触发观察 |
|---|---|---|
| `rows-asm.md` | exclude | compiler-generated C（monotonic.c），无 `.S` |
| `rows-operator-rvv.md` | exclude | 无循环/算子语义合同 |
| `rows-string-memory.md` | exclude | 无 memory/string 语义 |
| `rows-vectorized-tuning.md` | exclude | 零 `v*` |
| `rows-codegen.md` | **include** | runtime feature dispatch 信号：syscall fallback（clock_gettime）执行而 hardware 支持 `zicntr`（用户态 rdtime/csrr time）；dispatch（monotonicInit）未把硬件能力接到函数入口（宏门控未编译快速路径） |
| `rows-offload.md` | exclude | 无矩阵引擎上下文 |
| `rows-crypto.md` | exclude | 无密码学原语 |
| `rows-runtime-os.md` | exclude | 热点函数是用户态 syscall wrapper（非 kernel 侧 timer/CSR 路径）；修复点在用户态 dispatch/build 门控，不在内核 timer 记账 |

Classes scanned: **`rows-codegen.md`**

### Local performance pattern scan: `getMonotonicUs_posix.lto_priv.0`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Runtime CPU Feature Dispatch for ISA-Specific Intrinsics and VDSO（primary） | 每事件路径执行 clock_gettime syscall 回退，而 hardware 有 `zicntr`（`csrr time` 数周期）；dispatch 因 `USE_PROCESSOR_CLOCK` 宏未定义未接入 RISC-V mtime 快速路径 | High | Low | `patterns/runtime_isa_specific_instruction_dispatch.md` |

**三件套：**

(a) 逐字 evidence 引用（hot interval 全函数）：
- `0.00 : 2001ea: auipc ra,0xffe99` / `0.00 : 2001ee: jalr -1514(ra) # 98c00 <clock_gettime@plt>` —— 每次调用一次 clock_gettime syscall（函数成本主体）
- `50.00 : 2001cc: sd s0,48(sp)` / `25.00 : 2001ce: sd s1,40(sp)` / `25.00 : 200224: ld a2,0(s1)` —— syscall 周围 prologue/epilogue skid 聚集
- `0.00 : 2001f2-200232`（`lui/addi/slli/mulh/mul/srai/sub/add` 序列）—— `tv_sec*1000000 + tv_nsec/1000` 换算
- 源码锚：`monotonic.c:26-30`（`USE_PROCESSOR_CLOCK` 宏在注释块内，未定义）；`monotonic.c:141-194`（RISC-V mtime 路径 `read_mtime`=`csrr %0, time`、`monotonicInit_riscv`）；`monotonic.c:230-234`（dispatch：riscv init 未编译 → posix 回退）；`monotonic.c:196-204`（posix 实现）
- 调用点锚：`ae.c:228/279/291/313/338`、`server.c:2141/2192`（每事件循环迭代多次调用）
- baseline 锚：hardware isa 含 `zicntr`（用户态 time CSR）；build 含 `zicsr2p0`

(b) 互斥邻居排除：
- vs **Kernel Selection and Runtime Specialization**：行内互斥"硬件 ISA feature detection 缺口 → runtime-ISA-dispatch row"——本场景失败点恰是 feature/build 门控（`USE_PROCESSOR_CLOCK` 未定义导致快速路径未编译），归 runtime-ISA-dispatch → 不命中本 row。
- vs **Resource-Aware Instruction Scheduling / 其它 codegen 行**：函数成本是 syscall 而非指令调度 → 不命中。
- vs **rows-runtime-os（timer 硬件行）**：热点函数为用户态 syscall wrapper，非 RTOS/kernel 侧 timer 记账/ISR/CSR 路径 → class 排除（见上）。

(c) 双 Confidence 推导式：
- `route: 源码宏门控（monotonic.c:26-30 注释块）+ dispatch 逻辑（230-234）+ hardware zicntr + build zicsr 四重直接证据 → High`
- `impact: 4 samples、skid、bound type 未知、函数 workload 级 share 未知；每调用 syscall(50-150+ cycles) vs csrr time(数 cycles) 的机制收益明确但动态频率未量化 → Low`

**多候选仲裁小段：** 单顶层 finding（runtime-ISA-dispatch），无 supporting。Evidence-mechanism layer：**L0**（已有适配实现却因注册/优先级/属性 gate 未被正确选择——arbitration.md L0 层点名该机制；本场景为编译期宏 gate 使 dispatch 分支不可达，归 L0 认领）。动态优先级（模式 A）：4/4 局部样本（1.0）归属 syscall 回退路径。顶层 finding 数：**1**；supporting：0；排除条数：26；推导式：1。

## Phase 4 — Root-cause blueprint / 根因蓝图：getMonotonicUs_posix.lto_priv.0

**纳入蓝图的 row：** `Runtime CPU Feature Dispatch for ISA-Specific Intrinsics and VDSO`（Phase 3 通过 gate；L0 归属）。

1. **Root cause**：`monotonicInit()`（`monotonic.c:221-237`）选中了 POSIX 回退 `getMonotonicUs_posix`，因为 RISC-V mtime 快速路径（`monotonic.c:141-194`，`getMonotonicUs_riscv` 用 `csrr %0, time` 读用户态 `time` CSR）被 `USE_PROCESSOR_CLOCK` 宏门控，而该宏位于源码注释块内（`monotonic.c:26-30`）且构建未传 `-DUSE_PROCESSOR_CLOCK` → 快速路径**未编译**。结果：ae 事件循环每迭代多次调用（`ae.c:228/279/291/313/338`、`server.c:2141/2192`）的单调时钟读取每次执行完整 `clock_gettime` syscall（用户态-内核态切换，通常 50–150+ cycles），而 hardware（`zicntr`）支持用户态 `csrr time`（数 cycles）。依据 `patterns/runtime_isa_specific_instruction_dispatch.md` §Why this is slow #2："时间计数器读取必须通过系统调用... 单次调用开销通常为 50–150 个周期。RISC-V 的 rdtime 指令允许用户态直接读取 mtime 硬件寄存器（Zicntr 扩展），延迟仅为数周期"。4/4 样本聚集在 syscall 周围 prologue/epilogue（skid）即该 syscall 路径的热证据。

2. **The fix / 修复方式**（构建配置 + dispatch 接线，不改函数语义）：
   - Before：`USE_PROCESSOR_CLOCK` 未定义 → `monotonicInit_riscv` 未编译 → `getMonotonicUs = getMonotonicUs_posix`，每次调用 `clock_gettime` syscall。
   - After（blueprint，实现层方向）：① 构建时定义宏（`CFLAGS="-DUSE_PROCESSOR_CLOCK"` 或在 `monotonic.c` 为 `__riscv` 分支放开宏），使 `monotonicInit_riscv` 编译并在 `monotonicInit()` 中先于 posix 被 dispatch 选中 → `getMonotonicUs = getMonotonicUs_riscv`（`csrr time` + `/ mono_ticksPerMicrosecond`，约 5 条用户态指令）；② 增强 `get_timebase_frequency()` 的鲁棒性：`/proc/device-tree/cpus/timebase-frequency` 缺失时可用 `riscv_hwprobe`/auxv 兜底，避免 DT 文件缺失导致回退。
   - 适用前提：hardware `zicntr`（C920v2 已证实 isa 含 zicntr）；build `zicsr`（已含）；内核允许用户态读 `time` CSR（Linux 标准行为）。
   - correctness contract：mtime 单调性由 Zicntr 架构保证；`mono_ticksPerMicrosecond` 必须 >0（否则 `monotonicInit_riscv` 自身回退，语义不变）；`csrr time` 读值与 `clock_gettime(CLOCK_MONOTONIC)` 的时间域换算各自独立，`getMonotonicUs` 的 monotime 语义（微秒、单调）必须保持；VDSO 路径（若使用）需保持相同单调语义。
   - 限制/风险：`-DUSE_PROCESSOR_CLOCK` 同时激活 x86 TSC 路径（x86 构建下行为与现有相同，需 x86 侧回归）；若目标环境 DT 文件缺失，需第二项兜底否则仍回退 posix；宏为全二进制生效，需全量回归。
   - 预期 Profile signals：`getMonotonicUs_posix`（及 `.lto_priv.0` 变体）从热点中消失；`getMonotonicUs_riscv` 被 dispatch 选中；调用点时间读取降为用户态 `csrr time` 序列；syscall 计数（strace/perf syscalls）显著下降。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zicntr_...`（含 `v`、`zicntr`）；build ISA=`rv64i2p1_..._zicsr2p0_...`；VLEN=128 bits；bound type=`baseline_gap: bound type`。

4. **收益上界**：「当前 sampled event（cpu-clock）下的局部样本份额」= 4/4（1.0）——整个函数为 syscall 回退路径的产物，修复后该函数样本应归零（时间读取转入 `getMonotonicUs_riscv`）。函数 workload 级贡献未知 → 不表述 Amdahl。

5. **三维路由判定**：
   - `current source`：compiler-generated C（monotonic.c posix 实现）——直接证据。
   - `implementation existence/reachability`：RISC-V mtime 快速路径**存在**于源码（monotonic.c:141-194）但编译期 gate 未开启 → dispatch 不可达（L0）。非 runtime dispatch 缺失，是编译期宏 gate 缺失。
   - `function-level policy`：N/A（无 `.S` 载体 policy）。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。

7. **Related PRs 小节**：`Related PRs：23 条 URL` — https://github.com/golang/go/commit/4d10d4ad849467f12a1a16a5ade26cc03d8f1a1f 、https://github.com/golang/go/commit/b70244ff7a043786c211775b68259de6104ff91c 、https://github.com/golang/go/pull/53466 、https://github.com/torvalds/linux/commit/c51c26e687a649df9a792f71759105e12e5d082f 、https://github.com/torvalds/linux/commit/93b63f68d00a0483b450b446e2ea5386a1b94213 、https://github.com/torvalds/linux/commit/4c0b5a451675e9a95be98a16ddb889bb0486d2ad 、https://github.com/torvalds/linux/commit/55ca8d7aa2af3ebdb6f85cccf1b0703d031c1678 、https://github.com/torvalds/linux/commit/457926b253200bd9bdfae9a016a3b1d1dc661d55 、https://github.com/torvalds/linux/commit/650ea2a1dd964ca0a9c55f68dcb614d359c6b7d7 、https://github.com/torvalds/linux/commit/4b740779ac03f42866059a33f5454e1ac5393cdd 、https://github.com/torvalds/linux/commit/95bc69a47be2d5cdccf40ba3f23c99e9a6c57597 、https://github.com/qemu/qemu/commit/ae4a37f57818e47e212272821a5a86ad54620eb8 、https://github.com/qemu/qemu/commit/93cb52b7a3ccc64e8d28813324818edae07e21d5 、https://github.com/qemu/qemu/commit/7a999d4dd704aa71fe6416871ada69438b56b1e5 、https://github.com/qemu/qemu/commit/ebd476488d1369d602534e82234d410ca1734f07 、https://github.com/qemu/qemu/commit/abe2d74032d6d12a6918715086bbdf8843296f36 、https://github.com/qemu/qemu/commit/629ccdaa4e43c39d38d67b5ba1cec2bbedb6104d 、https://github.com/qemu/qemu/commit/7689b028ca56268fde3d492038413582b494c489 、https://github.com/qemu/qemu/commit/43740e3a3b3bb66456103684e622ba4e9baae297 、https://github.com/openjdk/jdk/commit/34412da52b41e9374168e67e3b6129576c8e4402 、https://github.com/openjdk/jdk/commit/538a722c2e9123cc575355879ff230444cf2dadc 、https://github.com/openjdk/jdk/commit/af9af7e90f7dab5adc7b89b76eb978d269e863de 、https://github.com/openjdk/jdk/pull/27512

## Phase 5 — Verification forecast / 验证预测：getMonotonicUs_posix.lto_priv.0

- **应消失/缩小**：`getMonotonicUs_posix.lto_priv.0` 从 perf 热点消失；`2001ea: auipc ra,0xffe99`/`2001ee: jalr ... <clock_gettime@plt>` 调用不再出现于时间读取路径。
- **应出现**：`getMonotonicUs_riscv`（`csrr time` 序列）被 dispatch 采用；与 `patterns/runtime_isa_specific_instruction_dispatch.md` §Verification 一致：`strace -c -e trace=clock_gettime` 确认 syscall 计数显著下降；单调性验证（无回跳）；`mono_ticksPerMicrosecond` 校准与 DT 缺失兜底路径测试；启用宏后全量回归。
- **升级到指令级结论所需数据**：perf stat cycles/instructions 与 syscall 计数确认函数级收益；precise 事件重采消除 skid；全程序样本总数计算函数级 share。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（含 Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ `1/1 组；getMonotonicUs_posix.lto_priv.0` |
| 2 | Phase 1 输出要求满足 | ✅ 7 行 baseline 表（含 `Sampling IP precision`）+ L0 观察（zicntr/zicsr 可用 + 宏门控）；gap：`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision` |
| 3 | Phase 3 输出要求满足 | ✅ 8 项 Class selection trace；`Classes scanned: rows-codegen.md`；顶层 finding 1（runtime-ISA-dispatch）；evidence 锚点 `2001cc: sd s0,48(sp)`、`2001ee: jalr ... <clock_gettime@plt>`、`200224: ld a2,0(s1)`；supporting 0；排除 26 条；推导式 1 |
| 4 | Phase 4 输出要求满足 | ✅ 已读 pattern：`patterns/runtime_isa_specific_instruction_dispatch.md`（命中 row，引用 §Why this is slow #2 "时间计数器读取必须通过系统调用…"）；`The fix` 含 before/after、correctness（mtime 单调、校准、VDSO）、风险、Profile signals；Related PRs：23 条 URL |
| 5 | 路径合规 | ✅ 模式 A；8 项 trace；leaf 来自 gate 通过 row；L0 归属；收益 1.0 局部表述 |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧 `2001ea`/`2001ee`；出现侧标注 pattern §Verification |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁 |

修正记录：无