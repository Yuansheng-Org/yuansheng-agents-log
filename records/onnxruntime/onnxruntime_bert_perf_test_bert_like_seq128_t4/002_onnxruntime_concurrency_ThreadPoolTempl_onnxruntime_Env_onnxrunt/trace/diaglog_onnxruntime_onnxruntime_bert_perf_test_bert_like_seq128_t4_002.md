Functions under analysis: [onnxruntime::concurrency::ThreadPoolTempl<onnxruntime::Env, onnxruntime::concurrency::WorkNoCallbackPolicy>::WorkerLoop(int)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`onnxruntime_pybind11_state.so` 中 `ThreadPoolTempl<Env, WorkNoCallbackPolicy>::WorkerLoop(int)`，f384f6–f39fbe，共 3268 行，109 samples，percent: local period）
- perf stat（可选 bound/context）：已提供（`perf_stat_onnxruntime_bert_perf_test_bert_like_seq128_t4.txt`）
- workload/binary/DSO/source context：已提供（onnxruntime 仓库 commit d4d792f545760ab292c5e68b2bb547d89306a7f0，本工作区含源码；热点 DSO 为 onnxruntime_pybind11_state.so，perf stat 来自 onnxruntime_perf_test）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries` 中 libonnxruntime.so / onnxruntime_perf_test / onnxruntime_test_runner 的 ELF attributes；annotate 所指 DSO 未单独提供 readelf -A，见 Phase 1）
- hardware ISA（cpuinfo / hwprobe）：已提供（metadata `cpuinfo.isa`，SpacemiT X100）
- `vlenb`：已提供（hardware_profile_snapshot.vector：vlenb=32 → VLEN=256 bits）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=cpu-clock、percent=local period、单次运行窗口；函数对整体 workload 的贡献未知）
- Sampling IP precision：缺失（无 `precise_ip` / Exact-IP / PMU skid 信息，详见 Phase 1）
- 源码（source context）：已提供（`onnxruntime/core/common/spin_pause.cc`、`include/onnxruntime/core/platform/EigenNonBlockingThreadPool.h`、`onnxruntime/core/common/threadpool.cc`、`onnxruntime/core/util/thread_utils.h`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | SpacemiT X100 `rv64imafdcvh_zicbom_..._zihintntl_zihintpause_..._zvbb_...` —— 含 `v`（RVV 1.0）与 `zihintpause`；含 `zaamo`/`zalrsc`（A 扩展） |
| Build ISA | `Tag_RISCV_arch: "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_..._zvl128b1p0..."` —— 含 `v`，**不含 `zihintpause`、不含 Zbb/Zba**；annotate 所指 DSO 未单独提供，按同构建的 libonnxruntime.so 属性推定 |
| Vector flavor | RVV 1.0 `v*` mnemonic（`vsetivli`、`vle64.v`、`vse64.v`、`vmv.x.s`、`vmv.v.i`），无 `th.v*`；硬件/构建均为 RVV 1.0，无 flavor mismatch |
| VLEN | `vlenb`=32 → VLEN=256 bits（hardware_profile_snapshot） |
| Bound type | 整轮 perf stat：duration 7.30s，task-clock 13.21s → **CPUs utilized ≈ 1.81**（4 线程 bert 测试中 worker 大量空闲）；cycles 29.04B / instructions 35.58B → IPC 1.23；无 cache/memory/branch counter → `baseline_gap: bound type`（cache/branch 分类未知） |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=local period；同一运行窗口；**函数对整体 workload 贡献未知** → 收益上界只能表述为该函数局部样本份额，不可称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 `precise_ip`/Exact-IP/skid 信息）——单行占比只锚定 basic block / loop interval，不承担单指令 latency 归因 |

L0 baseline gate：hardware 有 `v`、build 有 `v`，无 mismatch；无 `th.v*`。另发现：hardware 有 `zihintpause` 而 build ISA 无 `zihintpause`（且源码 `SpinPause()` 对 RISC-V 无任何 pause 发射路径）——这是本报告 primary finding 的准入事实。Bound-type gate：`baseline_gap: bound type`，本轮 performance-impact confidence 因此封顶 Medium。Sampling semantics 四项中 workload 贡献未知 → 收益上界仅为函数内局部份额。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：1 个函数。

hot loop / 区间边界：主循环为 `while (!should_exit)`（f3859a→f38a1a）；**热点区间为 idle 自旋/等待机制区间 f38658–f38fba**（外层 `for (int i = 0; i < spin_count_ && !done_; i++)` 自旋循环 + 其内部的 `ThreadPoolWaiter::wait()` 退避扩展），区间 back-edge 为 ` 0.92 :   f38fba: blt a4,a5,f38658`。任务执行路径（f389bc–f38a16：SetActive / callback Execute / LogRun）与阻塞路径（f38fbe–f39948：SetBlocked / cv.wait）在 annotate 中均为 0.00%——与源码语义一致：WorkerLoop 的 self-time 几乎全部是队列探测/自旋/等待机制，任务计算本身落在被调用的 kernel/lambda 中。

最高占比行（trace anchor）：
- ` 6.42 :   f38664: zext.b  a5,a5`（`done_` seq_cst 装载链）
- ` 6.42 :   f38670: addiw   a5,a5,-1`（`--steal_countdown`）
- ` 4.59 :   f386d2: li      s8,0`、` 4.59 :   f38f54: sd      a4,-224(s0)`（空 `std::function` 构造/swap 机制）
- ` 2.75 :   f3865c: lb      a5,0(s4)`、` 2.75 :   f38660: fence   r,rw`（`done_` seq_cst load + fence）
- ` 0.00 :   f38816: jal     f2fb10 <onnxruntime::concurrency::SpinPause()>`（退避循环内的 SpinPause 调用点）

区间样本汇总：f38658–f38fba 区间各行 local period 百分比相加 ≈ **89.9%**（该函数 109 samples 的绝大多数落在 idle 自旋/等待机制上）。Sampling IP precision 未知 → 结论收敛到 interval-level mechanism（自旋/等待机制成本），不做单指令 latency 归因。

## Phase 3 — Pattern scan / 模式扫描：onnxruntime::concurrency::ThreadPoolTempl<onnxruntime::Env, onnxruntime::concurrency::WorkNoCallbackPolicy>::WorkerLoop(int)

### Class selection trace（8 项）

| Class 文件 | include/exclude | 触发观察 |
|---|---|---|
| `rows-asm.md` | exclude | 当前代码来源不是手写 `.S`（compiler-generated 模板实例化代码，annotate 无 `.S` provenance）；无 dispatch-slot / policy 证据指向缺失 `.S` |
| `rows-operator-rvv.md` | exclude | 热点不是算子语义计算循环（无 elementwise/matmul/normalization 等计算合同）；是同步/协调循环 |
| `rows-string-memory.md` | exclude | 无 copy/fill/sentinel/compare/checksum 原语 |
| `rows-vectorized-tuning.md` | include | annotate 含 compiler-generated `v*`（`vsetivli`/`vle64.v`/`vse64.v`/`vmv.x.s`，用于 `std::_Any_data` 16B swap），非手写 `.S` —— Step 0d 判定必选 |
| `rows-codegen.md` | include | compiler-generated 代码 + 同步/原子/轮询/自旋 + 指令形态与分派 —— 主归属 class |
| `rows-offload.md` | exclude | 无矩阵引擎 / packed-SIMD / GEMM 卸载信号 |
| `rows-crypto.md` | exclude | 无密码学原语 |
| `rows-runtime-os.md` | exclude | 用户态应用热点，非 RTOS/kernel timer/ISR/CSR 路径 |

### Classes scanned: `rows-codegen.md`、`rows-vectorized-tuning.md`

### Local performance pattern scan: `onnxruntime::concurrency::ThreadPoolTempl<Env, WorkNoCallbackPolicy>::WorkerLoop(int)`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RISC-V ISA Extension-Specific Instruction Substitution（primary） | 自旋/等待区间 f38658–f38fba 主导 ~90% 函数本地样本；busy-wait 结构（有界迭代 + ThreadPoolWaiter 退避 + SetBlocked 阻塞 crossover）已在源码中存在；但 `SpinPause()` 对 RISC-V 落入 `std::atomic_signal_fence(seq_cst)` 纯编译器屏障（spin_pause.cc:57-62），**不发 `pause` hint**；hardware ISA 含 `zihintpause`、build ISA 不含；默认 `spin_backoff_max=1` 使退避斜坡不启用 | High | Medium | `patterns/isa_extension_specific_instruction_substitution.md` |
| Eliminate Redundant Sign/Zero Extensions via Word-Sized Operations（supporting） | ` 2.75 :   f3865c: lb a5,0(s4)` + ` 6.42 :   f38664: zext.b a5,a5`（`done_` load 链），同形态 `f3869a/f3869e`、`f3885a/f3885e`、`f38f06/f38f0a` —— typed load 后接冗余 zero-extension，`lbu` 可单指令完成 | — | — | `patterns/native_word_size_instruction_selection.md` |

**(a) 逐字 evidence 引用（primary）**：
- ` 2.75 :   f3865c: lb      a5,0(s4)`、` 2.75 :   f38660: fence   r,rw`、` 6.42 :   f38664: zext.b  a5,a5` —— 自旋循环条件 `!done_` 的 **seq_cst** 装载链（2 fence + load + extend，f38658–f38664 区间）
- ` 6.42 :   f38670: addiw   a5,a5,-1` —— `--steal_countdown`（steal_interval_ = spin_count_/100）
- ` 4.59 :   f386d2: li      s8,0`、` 2.75 :   f386c6: sd zero,-160(s0)`、` 2.75 :   f387a6: jalr ... <memset@plt>`、` 4.59 :   f38f54: sd a4,-224(s0)`、` 2.75 :   f38f50: vle64.v v1,(s9)` —— 空队列路径上每次迭代构造/移动/销毁空 `std::function`（memset + `_Any_data` swap）
- ` 0.92 :   f38fba: blt a4,a5,f38658` —— 自旋循环 back-edge
- ` 1.83 :   f3882c: bnez a5,f389b4`、` 1.83 :   f38836: beqz a5,f389b4` —— waiter lambda 内 `done_`/`spin_loop_status_` relaxed 轮询
- ` 0.00 :   f38816: jal     f2fb10 <onnxruntime::concurrency::SpinPause()>` —— 退避循环调用的 SpinPause（当前实现不发任何 RISC-V pause 指令）

源码证据（同一机制的直接 provenance）：
- `onnxruntime/core/common/spin_pause.cc:30-63`：`SpinPause()` 的 RISC-V 分支不存在 —— `#elif defined(__aarch64__)`/`__arm__` 之后直接 `#else // Generic fallback: a compiler barrier... std::atomic_signal_fence(std::memory_order_seq_cst);`
- `include/onnxruntime/core/platform/EigenNonBlockingThreadPool.h:1618-1620`：`ComputeSpinCount(spin_duration_us<0) = 1<<20`（默认 ~1M 次迭代）
- `include/onnxruntime/core/platform/EigenNonBlockingThreadPool.h:1658-1681`：`ThreadPoolWaiter::wait()` 退避结构（pause_count = 1,2,4,…,cap）；`onnxruntime/core/util/thread_utils.h:48`：`spin_backoff_max = 1`（默认无退避斜坡）
- `include/onnxruntime/core/platform/EigenNonBlockingThreadPool.h:1751-1772`：自旋循环（`!done_` seq_cst 条件 + `q.PopFront()` + `waiter.wait` lambda）

**(b) 互斥邻居排除**：
- **Bounded Spin-Wait Backoff row**：该 row 要求"循环没有有界/递增退避或阻塞 crossover"。此处源码结构**已有**有界迭代（`spin_count_`）、递增退避（`ThreadPoolWaiter`，当 backoff_max>1）与阻塞 crossover（`SetBlocked`/`cv.wait`，f38fbe–f39728）——因此该 row 的行内互斥判据"已有健康的有界 backoff、仅缺单条 `pause` hint → ISA-substitution row"直接命中本 primary；bounded-spin-wait 不另行认领（该 row 认领整个轮询频率/退避 progression 为根因，而此处根因是 RISC-V 上退避的"pause 载体"缺失）。
- **Redundant Synchronization Elimination in Uncontended Contexts row**：要求"可证明无并发写者"。此处 `done_` 由池析构并发写、队列由提交线程并发写，同步**必要**——gate 不成立，排除。
- **Hardware Atomic Operations for Hardware Synchronization row**：要求 atomic helper 未 lowering 为 AMO/LR-SC。反汇编显示 CAS 已用 `lr.w.aq`/`sc.w` 内联序列（f3888e–f388a6 等），exchange 用 `lr.w.aqrl`/`sc.w.rl`（f39082–f39090）——已 lowering，排除。
- **RVV Vector-State Management row**（rows-vectorized-tuning.md）：函数内确有 ~30 处 `vsetivli zero,2,e64,m1,ta,ma` 前置的 16B `_Any_data` 向量移动（如 `f38f50: vle64.v`、`f38f54: sd`），但向量指令本身样本占比极小（0–2.75%，多数 0.00%），样本主导在标量的空函数构造/swap 机制上；向量移动只是 `std::function` 机制的实现细节，不是主导机制，且修复对象不是 vector-state bookkeeping（源码层不可直接作为 vtype 收敛目标）——排除为命中，记录为症状观察。
- **rows-vectorized-tuning 其余行（autovec control / operand-form / LMUL sizing / inactive-lane / register-budgeted unrolling / maximal-LMUL）**：本函数不是向量计算 kernel，无对应 vector loop 语义，全部排除。
- **Zero-Based Comparison / Control-Flow / Register-Pressure / Load-Store Fusion / IV Strength Reduction 等 codegen 行**：无对应主导信号（无 zero-temp 物化 pattern、无分支/跳转主导、无 spill/reload 主导、无 index-scaling 循环寻址主导），排除。

**(c) 双 Confidence 推导式**：
- route: 源码直接证明 `SpinPause()` 对 RISC-V 无 pause 发射（spin_pause.cc 的 `#else` fallback）+ hardware `zihintpause`（cpuinfo）+ build ISA 缺 `zihintpause` + busy-wait 结构已有有界退避与阻塞 crossover + 行内互斥已逐项排除相邻 row → **High**
- impact: 自旋/等待区间占该函数本地样本 ~89.9%（local period，同窗口）但采样语义缺"函数对 workload 贡献"、`baseline_gap: sampling IP precision`、`baseline_gap: bound type`（缺 cache/branch counter；IPC=1.23、CPUs utilized≈1.81 佐证 worker 空闲期长）→ 缺项但 sample share 与采样语义（函数内局部）成立 → **Medium**

### 仲裁追加小段（入口条件 A）

顶层 finding 仅 1 个 primary（ISA-substitution / Zihintpause）；supporting 1 条（native-word 扩展消除）。证据机制分账：primary 的 evidence sample share 汇总为自旋/等待区间 f38658–f38fba ≈ **89.9% 函数内局部样本**（109 samples 中约 98 个）；supporting 与 primary 共享同一轮询区间（`done_` load 链），归 primary 之下的 supporting（`supporting because:` 同一机制——自旋轮询路径每轮固定指令开销的放大），不另计顶层命中数、不单独排序。L0–L4 归属：本 finding 属于 **L4 compute/codegen micro-structure**（指令选择/缺失 hint），非 L0 build（build 缺 zihintpause 是准入事实而非独立根因——即使 build 含 zihintpause，源码也没有发射 pause 的路径）。

## Phase 4 — Root-cause blueprint / 根因蓝图：onnxruntime::concurrency::ThreadPoolTempl<onnxruntime::Env, onnxruntime::concurrency::WorkNoCallbackPolicy>::WorkerLoop(int)

对应 Phase 3 通过 gate 的 row：`RISC-V ISA Extension-Specific Instruction Substitution`（rows-codegen.md 行内判据命中；已读 `patterns/isa_extension_specific_instruction_substitution.md`）。

### 1. Root cause

`WorkerLoop` 的 self-time 几乎全部是 idle 自旋/等待机制（f38658–f38fba ≈ 89.9% 函数内局部样本；任务执行与阻塞路径为 0.00%）。该机制在 RISC-V 上按全流水线速度运行，因为其唯一的"停顿"载体 `onnxruntime::concurrency::SpinPause()` 在 RISC-V 上没有实现——源码对 x86/ARM 分支之后直接落入 `#else` 的 `std::atomic_signal_fence(std::memory_order_seq_cst)`，即**纯编译器屏障，不发射任何架构指令**（依据 `patterns/isa_extension_specific_instruction_substitution.md` §The fix 第 4 节 "busy-wait loop body has no architecture spin hint ... Zihintpause `pause` hint"；原文要点："它不替代 acquire/release、fence 或锁协议本身……收益来自具体微架构可能降低资源争用、功耗或无效循环压力"）。

放大因素（同一机制）：
1. 默认 `spin_duration_us=-1` → `spin_count_ = 1<<20`（约 100 万次迭代，EigenNonBlockingThreadPool.h:1620），每次 idle 窗口每个 worker 可全速自旋约 100 万轮；
2. 默认 `spin_backoff_max=1`（thread_utils.h:48）→ `ThreadPoolWaiter` 的指数退避斜坡不启用，每轮只调 1 次 `SpinPause()`；
3. 每轮自旋开销重：`!done_` **seq_cst** 装载（f38658 `fence rw,rw` + f3865c `lb` + f38660 `fence r,rw` + f38664 `zext.b`，4 条指令/轮）+ 队列前端状态探测 + 空队列时构造/销毁空 `std::function`（memset + `_Any_data` swap，f386ac–f38744 区间）。
4. 整轮 perf stat 显示 CPUs utilized ≈ 1.81（4 线程 bert 测试），说明 worker 大量时间处于空闲自旋而非执行任务——与该区间样本主导一致。

### 2. The fix / 修复方式

修复对象：`onnxruntime/core/common/spin_pause.cc` 的 `SpinPause()`（以及使 RISC-V 构建包含 `zihintpause`）。这是架构 spin hint 缺失的指令选择修复，不改变同步语义。

修复前（现状）：
```cpp
// onnxruntime/core/common/spin_pause.cc, SpinPause()
#elif defined(__arm__)
  __asm__ __volatile__("yield" ::: "memory");
#else
  // Generic fallback: a compiler barrier. ... intentionally much cheaper than
  // std::this_thread::yield() so that callers in worker spin loops do not pay
  // scheduler overhead.
  std::atomic_signal_fence(std::memory_order_seq_cst);
#endif
```

修复后（在 `#else` 之前加入 RISC-V 分支，方向与 `patterns/isa_extension_specific_instruction_substitution.md` §The fix 第 4 节一致——"feature-gated spin hint preserves synchronization semantics"）：
```cpp
#elif defined(__riscv)
#if defined(__riscv_zihintpause)
  // 架构 spin hint：Zihintpause `pause`。只作流水线/资源提示，不改变
  // 任何内存序或同步语义；在不支持该 hint 的实现上按 spec 退化为无害
  // 的 hint（类似 NOP/fence 编码空间），但为稳妥仍保留 hwprobe/macro gate。
  __asm__ __volatile__("pause" ::: "memory");
#else
  // RISC-V 且构建不含 zihintpause：保留原编译器屏障 fallback。
  std::atomic_signal_fence(std::memory_order_seq_cst);
#endif
#else
  std::atomic_signal_fence(std::memory_order_seq_cst);
#endif
```

适用前提与 gate：
- 工具链/构建需要把 `zihintpause` 加入 RISC-V `-march`（当前 build `Tag_RISCV_arch` 缺 `zihintpause`；hardware cpuinfo 含 `zihintpause`）。若不想动全局 `-march`，可照 pattern §The fix 第 4 节的等价编码形式在 feature gate 下发射（`.word 0x0100000f`，即 Zihintpause 的 `pause` 编码）——该编码方案与 `__riscv_zihintpause` 宏 gate 二选一，不得两者混用导致重复发射。
- 正确性 contract：`pause` 是 hint，**不得替代**任何 fence/acquire/release/atomic/退出条件；ORT 的同步协议（`done_` seq_cst、互斥锁、条件变量、队列 LR/SC CAS）原样保留。`SpinPause()` 的调用者（`ThreadPoolWaiter::wait` 退避循环、`CalibrateSpinPauseNs` 校准循环、stream_execution_context 等）语义不变。
- 风险与限制：`pause` 的收益是微架构相关的（X100 为 OoO 核，需实测 throughput/latency/energy）；`CalibrateSpinPauseNs()` 的校准值会随真实 pause 延迟变化，但 `ComputeSpinCount` 由校准值推导迭代数（自校正），time-based spin 窗口语义保持；迭代数语义（默认 -1 → 1<<20 次迭代）不变，因此单次 idle 窗口的墙钟时长会变长（每轮被 pause 拉长），这是预期行为——目的不是缩短自旋线程的墙钟，而是降低其资源消耗密度与对 producer/其他 worker 的争用。
- 附加杠杆（同一轮询路径，供同一修复窗口评估，非独立 pattern 命中）：
  - 自旋循环条件 `!done_`（EigenNonBlockingThreadPool.h:1751）改为 `done_.load(std::memory_order_relaxed)`：去掉每轮 2 条 fence（f38658/f38660）。退出正确性由后续 `waiter.wait` lambda（relaxed）+ `SetBlocked` 持锁复查（f38fbe 起）保证，无丢失唤醒风险（SetBlocked 内已在锁下重查队列与 `done_`）。
  - 默认 `spin_backoff_max=1` 使退避斜坡不生效；配合真实 `pause` 后，建议评估将默认值提到小整数（如 8），让每轮 pause 计数随等待增长（1,2,4,8,…），显著降低长 idle 窗口的轮询密度。
  - （编译器侧观察，非源码直接可改）GCC 对 `std::atomic<bool>` 的 seq_cst load 生成 `lb`+`zext.b` 而非 `lbu`；改用 relaxed load 后该形态消失，`lbu` 单指令完成。

### 3. Baseline facts 回填

hardware ISA：rv64imafdcvh + zihintpause + zaamo/zalrsc（SpacemiT X100，OoO）；build ISA：rv64i2p1_..._v1p0_...（含 `v`，不含 zihintpause）；VLEN：256 bits（vlenb=32）；bound type：`baseline_gap: bound type`（IPC 1.23、CPUs utilized≈1.81，缺 cache/branch counter）。

### 4. 收益上界

入口条件 A：primary 的 evidence sample share 加总 = 自旋/等待区间 f38658–f38fba ≈ **89.9% 函数内局部样本**（109 samples 中约 98 个）。表述为「当前 sampled event（cpu-clock）下该函数的局部样本份额」；因采样语义四条中"函数对 workload 贡献"未知，**不得**称 workload 级 Amdahl 上界。修复的直接效果是降低自旋轮询密度（单位墙钟内轮询/原子/内存流量下降）与资源争用；对端到端 bert 推理时延的收益方向为正但幅度需实机验证（X100 微架构属性）。

### 5. 三维路由判定

- current source：compiler-generated C++（模板实例化，含源码路径 EigenNonBlockingThreadPool.h/spin_pause.cc），非 `.S`、非 JIT；
- implementation existence/reachability：退避/自旋结构已存在且可达（默认配置下必然执行），缺的是 `SpinPause()` 中 RISC-V `pause` hint 的实现路径——reachability 成立、实现缺失；
- function-level policy：`SpinPause` 是 ORT 公共并发原语（`core/common/spin_pause.cc`），其跨架构契约要求"spin-loop 内在指令"；RISC-V 分支缺失是源码层事实，与项目对 x86/ARM 的既有处理一致。不进入 policy-backed missing `.S` 分支（非汇编载体问题）。

### 6. Related PRs 小节（依据 `patterns/isa_extension_specific_instruction_substitution.md` ## Related PRs / 关联提交）

Related PRs（Zihintpause spin-wait hint 直接相关）：
- https://github.com/openjdk/jdk/commit/8cb9b479c529c058aee50f83920db650b0c18045 （OpenJDK Zihintpause Spin-Wait Hint）
- https://github.com/openjdk/jdk/pull/11921 （OpenJDK Zihintpause Spin-Wait Hint）
- https://github.com/ggml-org/llama.cpp/pull/17784 （llama.cpp Zihintpause Spin-Wait Hint，thread-pool barrier busy-wait）

同一 pattern 表的其余 substitution 类别 PR（同表 URL 去重后列出）：
- Go：https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0 ；https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c ；https://github.com/golang/go/pull/59488 ；https://github.com/golang/go/commit/63ab68ddc5f1307e552cf27ae7a6f0dfda2bb962 ；https://github.com/golang/go/commit/1951afc9193f8e197cb7dfaf6afed70ea02404cb
- OpenCV：https://github.com/opencv/opencv/commit/a00818047ff5671586c7b296cac6175b250f85d3 ；https://github.com/opencv/opencv/commit/7e2c8cc9f4c49a3afeee084a2f01f7bac1d9a2dc ；https://github.com/opencv/opencv/commit/f0d29cd33c5d2b8f6c9c5c3177cbb3a359ee6b33 ；https://github.com/opencv/opencv/pull/21351
- OpenSSL：https://github.com/openssl/openssl/commit/03ce37e11729 ；https://github.com/openssl/openssl/commit/ca6286c382a7 ；https://github.com/openssl/openssl/commit/48b6776678d7 ；https://github.com/openssl/openssl/commit/6136408e6abf ；https://github.com/openssl/openssl/commit/e4fd3fc379d7 ；https://github.com/openssl/openssl/commit/80c664db430d ；https://github.com/openssl/openssl/commit/08c8dd6b8ced；https://github.com/openssl/openssl/commit/49a3e7adc392；https://github.com/openssl/openssl/commit/a41f9135f082；https://github.com/openssl/openssl/commit/4dbb537bd1ea；https://github.com/openssl/openssl/commit/608cadfbdbdb；https://github.com/openssl/openssl/commit/b1b889d1b3fc；https://github.com/openssl/openssl/commit/657d1927c68b；https://github.com/openssl/openssl/commit/611685adc04a；https://github.com/openssl/openssl/commit/7ae2bc9df6e0
- Linux Kernel RISC-V：https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712 ；https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49 ；https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92 ；https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de ；https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355 ；https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb ；https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68
- OpenJDK（其余）：https://github.com/openjdk/jdk/commit/6b89954c65342bc601633d24075dab4f4b248f4b ；https://github.com/openjdk/jdk/commit/a7631ccf18e468d6ecba121865f7fed29cbf2186 ；https://github.com/openjdk/jdk/pull/22752 ；https://github.com/openjdk/jdk/commit/08d563ba15047020fd5f5fea80547e18898bbab2 ；https://github.com/openjdk/jdk/commit/b1a21b563e3ae13fa5c409a4f0c04686c3f5b34a ；https://github.com/openjdk/jdk/pull/22410 ；https://github.com/openjdk/jdk/commit/9f582e56baee0e7f5af20da0f395cd935bf5a962 ；https://github.com/openjdk/jdk/pull/24096 ；https://github.com/openjdk/jdk/commit/1a4bbb0027ae9e6df3b668454fa155861d531f72 ；https://github.com/openjdk/jdk/commit/2ed7ad4b5c7d2344ae6571c186f8a2903770aa57 ；https://github.com/openjdk/jdk/commit/edfe28541a6e94357f873aa69778c7eba707cbb ；https://github.com/openjdk/jdk/pull/22386 ；https://github.com/openjdk/jdk/commit/b891bfa7e67c21478475642e2bfa2cdc65a3bffe ；https://github.com/openjdk/jdk/commit/3d3b78203710 ；https://github.com/openjdk/jdk/commit/5866b16dbca3f63770c8792d204dabdf49b59839 ；https://github.com/openjdk/jdk/commit/bcc33d5ef3bd
- V8：https://github.com/v8/v8/commit/6f100865663fb99df2628144fc65977e55ab7e68 ；https://github.com/v8/v8/commit/e62c1e307d207e1e56219c172929289a1b474530 ；https://github.com/v8/v8/commit/200b5212ae4752f6345dbcc6ecf23f651434bbf8 ；https://github.com/v8/v8/commit/5136fb5200c1f3a33939419fed9de32e8d85bc1e ；https://github.com/v8/v8/commit/e02d2238f6ea6f920a6ac31f888b4727bb0b0f80 ；https://github.com/v8/v8/commit/94a3c420e2a4f76d42e367e7d3b3f85ecd1b0a3f ；https://github.com/v8/v8/commit/1818e36d54d1 ；https://github.com/v8/v8/commit/5f433dd5024a566fa051317dd0c9c1d9921a9162 ；https://github.com/v8/v8/commit/9bbbde26bb90bd493afe33777d7b6be0c4f2afb4 ；https://github.com/v8/v8/commit/758956654f28861ba0d5e94f03fff6016b3a2998 ；https://github.com/v8/v8/commit/32c5d22333c14a9d6bb645af3ea4d3fb28b541fe ；https://github.com/v8/v8/commit/89719bc239f48735bbffe83e0803fd504b3c485a ；https://github.com/v8/v8/commit/223d7fb26b12145bdd335d51da2fa2ed1c58ce3a ；https://github.com/v8/v8/commit/b2852080c401cb0980f332918531597967de36c1 ；https://github.com/v8/v8/commit/8b842dbb9f6d7ea03990d80dc945ef8c6c6ca144 ；https://github.com/v8/v8/commit/e855c14cab2c6522a2a2d9b6be51e886fd3aae74 ；https://github.com/v8/v8/commit/8033cc56e2afb7b19214aa9a6f776a59fc509d2c ；https://github.com/v8/v8/commit/a654e27b50feb457a99ad27582dd555316eab587 ；https://github.com/v8/v8/commit/21289aa92c80a4161cc5f0d9915a0002a644d8b4 ；https://github.com/v8/v8/commit/19a0f69f4c4b1257b567290412acda411568d565
- QEMU：https://github.com/qemu/qemu/commit/3de1fb712a072992d72bc99c2b70978132ee44d0 ；https://github.com/qemu/qemu/commit/6ef5843182382f6a84995590ad91047b0f2bc1fa
- LLVM：https://github.com/llvm/llvm-project/pull/170824 ；https://github.com/llvm/llvm-project/commit/4c1e1e05cb901a2ed9055e5d6ac6ce60b826a288 ；https://github.com/llvm/llvm-project/pull/92926 ；https://github.com/llvm/llvm-project/pull/152744 ；https://github.com/llvm/llvm-project/pull/122698 ；https://github.com/llvm/llvm-project/commit/13e32a8a3c95b23af51f081865db1bd259d269a4 ；https://github.com/llvm/llvm-project/commit/787eeb8597fac22decb366a42176b11f52ec1bf0 ；https://github.com/llvm/llvm-project/commit/c705b7b04dba467a67871a1bbb77907d0ed7fc19

## Phase 5 — Verification forecast / 验证预测：onnxruntime::concurrency::ThreadPoolTempl<onnxruntime::Env, onnxruntime::concurrency::WorkNoCallbackPolicy>::WorkerLoop(int)

primary（ISA-substitution / Zihintpause）验证预测：

- 应消失/缩小（锚定 Phase 3(a) 引用行）：对同一函数重新 annotate 后，自旋/等待区间 f38658–f38fba 内 `f3865c: lb a5,0(s4)`、`f38660: fence r,rw`、`f38664: zext.b a5,a5`、`f38670: addiw a5,a5,-1`、`f386d2: li s8,0`、`f38f54: sd a4,-224(s0)` 等轮询行的**单位墙钟样本密度**应显著下降（pause 拉长每轮、轮询频率下降）；在相同 workload/窗口下该区间占函数样本的比例应下降。
- 应出现（锚定 pattern 文件）：`patterns/isa_extension_specific_instruction_substitution.md` §Verification —— "confirm `pause` / encoded Zihintpause appears only when the feature gate is enabled, preserves lock and memory-model semantics, and is benchmarked separately for throughput, latency, and energy on the target microarchitecture"。即：反汇编 `SpinPause()` 出现 `pause`（或 `0x0100000f` 编码）且仅在 feature gate 开启时出现；不使用 zihintpause 构建时原 fallback 复现。
- 正确性与边界：memory ordering / 可见性 / 进展性 / 退出条件不变（pause 为纯 hint）；`CalibrateSpinPauseNs()` 校准值与 `ComputeSpinCount` 自校正关系保持（time-based spin 窗口语义不变）；覆盖立即有工作、短等待、长等待（1<<20 全窗口）、shutdown/cancel 路径。
- 实机对比：同一 X100 上分别测 throughput（端到端 bert_like_seq128_t4 时延）、latency、energy/power 与 IPC；低争用（任务间空隙大）场景收益预期最明显；另测默认配置（spin=1<<20, backoff=1）与 backoff=8 的组合，验证退避斜坡与真实 pause 叠加后的轮询密度下降。
- supporting（native-word `lb`+`zext.b`）不单独验证，随 primary 的轮询路径变化一并观察（修复 primary 的 relaxed-load 杠杆后该形态自然消失）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 全部出现（载荷：`1/1 组；ThreadPoolTempl<Env, WorkNoCallbackPolicy>::WorkerLoop(int)`） | ✅ |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 结论；gap 标签 `baseline_gap: bound type`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） | ✅ |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；`Classes scanned: rows-codegen.md、rows-vectorized-tuning.md`；顶层 finding 1（primary）；evidence 锚点 `f3865c: lb`、`f38664: zext.b`、`f38670: addiw`、`f38fba: blt`、`f38816: jal SpinPause`、spin_pause.cc:57-62；supporting 1（native-word）；排除 8+ 条含行内互斥推导） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 `patterns/isa_extension_specific_instruction_substitution.md`；命中 row `RISC-V ISA Extension-Specific Instruction Substitution`；引用短语首词 `busy-wait loop body`、`The fix` 第 4 节、`§Verification`；`The fix` 的 before/after（spin_pause.cc `#else`→RISC-V `pause` 分支）、correctness（hint 不替代 fence/atomic）、风险（微架构相关、CalibrateSpinPauseNs 自校正）、预期 Profile 信号（轮询行密度下降）齐全；Related PRs 列表含 Zihintpause 3 条 + 同表 URL 全量） | ✅ |
| 5 | 路径合规（载荷：模式 A profile-backed；路径 rows-codegen（L4 compute/codegen micro-structure）+ rows-vectorized-tuning（必选，v* 存在）；顶层 finding 按 A 排序（89.9% 函数内局部份额）） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `f3865c/f38660/f38664/f38670/f386d2/f38f54`；出现侧 `patterns/isa_extension_specific_instruction_substitution.md` §Verification spin-wait hint 项） | ✅ |
| 7 | 契约边界合规（载荷：无实施询问/无代码修改/无补丁生成/无契约外分支；交付止于 Profile 证据 + 根因蓝图 + The fix + 验证预测） | ✅ |

修正记录：无