Functions under analysis: [dnnl::impl::cpu::rv64::jit_rvv_softmax_f16_exp_sub_sum(...)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`dnnl::impl::cpu::rv64::jit_rvv_softmax_f16_exp_sub_sum(dnnl::impl::float16_t const*, float*, long, float, float*)`，`libdnnl.so.3.14`，5 samples，event=cpu-clock:u，percent type=local period）。样本量极小（5），进入 confidence 推导但不改变小节结构。
- perf stat（bound/context）：已提供（同 run 共享 perf stat：IPC 0.800229；L1_dcache_load_miss_rate 0.418%；LLC_load_miss_rate 21.232%）。
- workload/binary/DSO/source context：已提供（oneDNN main @ d22de940f；该符号是 `jit_rvv_softmax_f16_exp_sub_sum` 的 C++ wrapper（`src/cpu/rv64/jit_rvv_softmax_kernel.cpp` 行 174–179：构造 `call_params_t p{...}` 并调用 `dispatch_f16_exp_sub_sum(&p)`），JIT kernel 本体在独立的 jitted DSO 中，不在本符号内）。
- readelf -A：缺失（详见 Phase 1）。
- hardware ISA：已提供（RVV 1.0，含 `v`/`zvfh`）。
- `vlenb`：已提供（vlen_bits=128）。
- 采样元数据：部分（cpu-clock:u；local period；单窗口；函数级贡献未知）。
- Sampling IP precision：缺失（precise_ip 未知）。

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zvfh_...`：RVV 1.0；C920v2/SG2044（OoO，VLEN=128 实测） |
| Build ISA | `baseline_gap: build ISA`。libdnnl.so 未提供 readelf -A 输出；该 wrapper 为静态编译 C++ 代码（含 stack canary `__stack_chk_guard@GLIBC_2.27` 与 magic-static guard），JIT kernel 部分以 generated-code dump 为准 |
| Vector flavor | 本符号内无向量指令（仅 wrapper 逻辑）；JIT kernel 本体为 RVV 1.0（见 006/007 的 generated-code dump） |
| VLEN | 128 bits（vlenb=16）→ exp_sub_sum JIT kernel 在 e16/m2 下 VLMAX=16 元素/轮 |
| Bound type | 本符号样本全在 wrapper 的调用设置区（prologue/spill/guard）→ 归类为 per-call dispatch/latency 开销，非计算/访存主体（本符号无计算） |
| Sampling semantics | cpu-clock:u；local period；单窗口；函数级贡献未知 → `baseline_gap: sampling metadata` |
| Sampling IP precision | `baseline_gap: sampling IP precision`；单行占比只锚定 wrapper 调用设置 interval（c71f80–c71fc2） |

L0 baseline gate：hardware 有 `v`，无 mismatch 证据；无 `th.v*`。Bound-type gate：本符号为 per-call dispatch 开销，compute/vector pattern 不适用。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致。该符号是 exp_sub_sum JIT kernel 的 C++ 调用 wrapper：构造 `call_params_t p{src, tmp, len, sub, sum}`（栈上 struct），经 magic-static guard 后取 static kernel 对象内嵌的 `jit_ker_` 函数指针并 `jalr` 调用。JIT kernel 本体（exp 多项式循环）在另一个 DSO（jitted-*.so）中，不归本符号。

Hot interval：c71f80–c71fe6（wrapper 全程）。样本分布：
- `40.00 : c71f82: sd s0,64(sp)`（frame 建立）
- `20.00 : c71fb6: sd a3,-48(s0)`（参数 spill 到 struct）
- `20.00 : c71fba: lb a5,0(a5)`（magic-static guard 加载）
- `20.00 : c71fc2: zext.b a5,a5`（guard 零扩展）
- ` 0.00 : c71fd0: jalr a5`（JIT kernel 调用点）
Trace anchor（最高占比行）：`40.00 : c71f82: sd s0,64(sp)`，interval 内样本 100%（5/5）。IP precision 未知 → 只锚定 wrapper 调用设置 interval。annotate 覆盖完整，入口条件 A（profile_backed）。

## Phase 3 — Pattern scan / 模式扫描：jit_rvv_softmax_f16_exp_sub_sum

### Class selection trace（8 项）
1. `rows-asm.md` — exclude：本符号是编译器生成的 C++ wrapper，非手写 `.S`。
2. `rows-operator-rvv.md` — exclude：本符号无算子计算体（计算在 JIT kernel / 其它 DSO）；类内全部 row 需 `v*` 或算子语义热点，本符号内无向量指令。
3. `rows-string-memory.md` — exclude：非 string/memory 语义。
4. `rows-vectorized-tuning.md` — exclude：本符号 hot interval 无 `v*`（`v*` 在 JIT kernel DSO 中），类内 7 个 row 均要求已向量化 interval 作为证据基础。
5. `rows-codegen.md` — include（必选）：compiler-generated wrapper 的指令形态、调用/分派/同步开销。
6. `rows-offload.md` — exclude：无矩阵引擎/权重重排证据。
7. `rows-crypto.md` — exclude：无密码原语。
8. `rows-runtime-os.md` — exclude：用户态，无 timer/ISR/CSR。

### Classes scanned: rows-codegen.md（rows-operator-rvv.md 与 rows-vectorized-tuning.md 已按类内第一级判据整组排除：本符号无向量指令与算子计算）

### Local performance pattern scan: `jit_rvv_softmax_f16_exp_sub_sum`

**No local pattern matched。**

### 逐 row 排除表（rows-codegen.md，27 行）
| Row | 判定 | 判别性观察 |
|---|---|---|
| Kernel Selection / Runtime Specialization | exclude | wrapper 无 lookup/fallback 路由；static kernel 固定，guard 检查不是选择逻辑；样本不在选择循环内 |
| Cache-Aware Blocking | exclude | 无 tiled kernel/工作集证据 |
| Kernel Operation Fusion | exclude | 样本是 per-call 设置开销而非 pass 间中间流量；exp_sub_sum→affine 有 inv_sum 跨 pass 依赖，无法合成单 pass |
| Compiler Optimization Workaround Retirement | exclude | 无历史 `-O0/-fno-*` workaround 证据 |
| Forced Inlining for Hot Specialization Helpers | exclude | 被调 callee 是 JIT 生成代码（函数指针），不可内联；非 missed-inline 的静态 helper |
| Control-Flow Layout and Transfer | exclude | 单次间接调用；无 branch diamond/错误转移/RAS 污染证据 |
| Code Layout and Constant-Pool | exclude | 无 constant-pool/trampoline/frontend 证据 |
| Load/Store Addressing-Mode Fusion | exclude | 无可折叠地址生成序列 |
| Register Pressure / Save-Restore | exclude（作 micro-analysis 素材） | prologue 为标准 frame；canary 帧 + struct spill 由 ABI 驱动（JIT kernel 单指针参数），非 RA 可消除 |
| GP-Relative Small-Data | exclude | 无 gp small-data 对象 |
| Floating-Point Semantic Lowering | exclude | 本符号无 FP 语义序列 |
| Trap-Based Guard | exclude | 无 fault/trap 语义 |
| Resource-Aware Scheduling | exclude | 无 target-core latency 模型与依赖/barrier 证据 |
| Algebraic Simplification | exclude | 无可代数消除序列（`li a5,0` 为 guard 初始化惯用） |
| ALU Constant Materialization | exclude | 常量物化（auipc/addi）每调用仅 3 条，非主导 |
| Eliminate Precision Conversions | exclude | 无 f32↔f64 往返 |
| Hardware Atomic / Spin-Wait | exclude | 无 AMO/lr-sc/轮询 |
| ISA Extension Substitution | exclude | 无 Zbb 等单指令替换主体 |
| Native-Width State | exclude | 无窄状态循环 |
| Native Word Size / Redundant Ext | exclude | `zext.b a5,a5` 紧跟 `lb` 是 guard 字节语义所需，非冗余 |
| Runtime CPU Feature Dispatch | exclude | guard 是 magic-static 线程安全协议，非 feature dispatch |
| Tail Call Optimization | exclude | wrapper 必须在调用后校验 canary，无法 tail-call；callee 为函数指针 |
| Zero-Based Comparison | exclude | `li a5,0` 是 guard 初始化路径赋值 |
| Redundant Synchronization Elimination | exclude（最近候选，记入 micro-analysis） | guard 快速路径的 `fence r,rw` 是 C++ magic-static acquire 语义所需；benchdnn 多线程（parallel(nthr)）下无法证明无并发初始化 → row gate 不成立 |
| JIT-Generated Code Quality | exclude | 本符号是静态编译器生成 wrapper（该 row 行内互斥「静态 compiler codegen → 对应 codegen row」）；JIT kernel 本体在其它 DSO，不属本符号 |
| Loop Induction Variable Strength Reduction | exclude | 无归纳变量循环 |

### 零命中后第一性原理 micro-analysis（不依赖本目录的 bottleneck 假设）
按 arbitration "When no row matches" §5，给出基于反汇编静态计数的假设，confidence 单独标注：

每调用一次的 wrapper 固定指令数（静态统计，来自本 annotate）≈ 31 条：
- prologue/canary：`addi sp,-80`、`sd s0`、`sd s1`、`addi s0,sp,80`、`sd ra`、`auipc s1`、`ld s1(canary)`、`ld a5,0(s1)`、`sd canary` = 9
- 参数 struct spill：`fsw fa0`、`sd a0`、`sd a1`、`sd a2`、`sd a3` = 5
- magic-static guard 快速路径：`auipc a4`、`addi a4`、`addi a5,a4,216`、`lb a5,0(a5)`、`fence r,rw`、`zext.b a5,a5`、`beqz` = 7
- 取指针+调用：`ld a5,568(a4)`、`addi a0,s0,-80`、`jalr a5` = 3
- epilogue/canary 校验：`ld canary-saved`、`ld canary`、`xor`、`li a4,0`、`bnez`、`ld ra`、`ld s0`、`ld s1`、`addi sp,80`、`ret` = 10

对 axis_size=16 的 softmax 行（exp_sub_sum JIT kernel 在 e16/m2 下 VLMAX=16，主循环仅 1 轮 + 收尾 vfredosum/fsw，动态指令数与 wrapper 同量级），wrapper 的 31 条固定指令约为该调用总成本的一半。f16 路径对 exp_sub_sum 无最小长度 gate（对比 f32 路径有 `exp_jit_min_len=16`），因此短 axis 行也会付出完整 JIT dispatch 成本。

**Hypothesis candidate（zero-match micro-analysis，非 matched row）**：per-call JIT dispatch 固定开销（magic-static guard + `fence r,rw` + stack canary + params struct spill + 间接调用）在短 axis/高行数 softmax shape 上是本符号样本的主导载体（本符号 100% 样本落在该区间）。route≤Medium、impact=Low（无 row gate、样本 5、无 crossover 实测）。

## Phase 4 — Root-cause blueprint / 根因蓝图：jit_rvv_softmax_f16_exp_sub_sum

- 无通过 gate 的 matched row（零命中）；本蓝图来自 arbitration §5 的第一性原理 micro-analysis，整体标注为 hypothesis。
- **Root cause（hypothesis）**：`jit_rvv_softmax_f16_exp_sub_sum` 的编译器生成 wrapper 每调用承担约 31 条固定指令（canary 帧 9 + struct spill 5 + magic-static guard 快速路径 7 + 取指针/调用 3 + epilogue 10），其中 guard 快速路径含 `fence r,rw`（OoO 上具内存序串行成本）且 struct spill/canary 为 per-call 固定成本。对 axis_size≈VLMAX（16）的 softmax 行，该成本与 JIT kernel 单轮计算同量级；f16 路径缺少 f32 路径已有的短 axis JIT gate（`exp_jit_min_len`），所有行都支付该固定成本。
- **The fix / 修复方式（hypothesis，方向性建议，需实测确认）**：
  1. 把 static kernel 的 `jit_ker_` 函数指针在 `execute_forward` 级解析并缓存（wrapper 每行调用改为直接间接调用，消除每调用的 magic-static guard + fence 路径与 `auipc/addi` 地址物化）；或把 function-local static 改为在 primitive 构造时初始化的成员/文件级单例，让 guard 检查只发生在构造期。
  2. 为 f16 exp_sub_sum（及 affine/reduce_max）增加与 f32 路径一致的短 axis 长度 gate，短 axis 路由到既有 scalar path（`compute_softmax_f16_scalar`），避免 JIT dispatch 固定成本大于计算本身；crossover 需按目标 shape 实测。
  3. 若安全策略允许，热 TU 编译去掉 stack-protector（消除 per-call canary 的 load/store/xor/branch）——此为 build config 选项，涉及安全权衡，不作为主推。
  - correctness contract：guard 的 acquire 语义（fence）不可在未消除并发首次初始化风险时移除；params struct 布局不变；JIT kernel 行为不变。
  - 预期 Profile signals：wrapper 区（c71f80–c71fc6）样本下降；`jalr` 调用点保留但前置 guard/地址物化指令消失；短 axis 行整体耗时下降。
- **Baseline facts 回填**：hardware ISA=含 `v`；build ISA=`baseline_gap: build ISA`（无 readelf -A；wrapper 含 `__stack_chk_guard`，JIT 部分 RVV 1.0）；VLEN=128；bound type=per-call dispatch/latency。
- **收益上界**：本符号局部样本份额 100%（5/5）落在 wrapper 调用设置区间；但该份额不能直接等于可修收益——假设只覆盖 wrapper 固定成本部分，且需跨 shape 实测；`hypothesis_only` 语义下不估算幅度，不做 workload 级 Amdahl 表述。
- **三维路由判定**：current source=compiler-generated C++ wrapper（静态代码）；implementation existence/reachability=JIT kernel 可达（006/007/本符号证明调用链工作）；function-level policy=JIT dispatch 为既定 ABI（单指针参数）→ 修正方向在调用侧（缓存指针/长度 gate），不改 JIT kernel 本体。
- **Related PRs**：无命中 pattern（zero-match），无 Related PRs 表可引。

## Phase 5 — Verification forecast / 验证预测：jit_rvv_softmax_f16_exp_sub_sum
- 应消失/缩小（锚定 Phase 3(a) 引用行）：若实施「函数指针缓存/构造期初始化」，`c71fba: lb a5,0(a5)`、`c71fbe: fence r,rw`、`c71fc2: zext.b a5,a5`、`c71fc6: beqz a5,<init>` 与 `c71f92/c71f96/c71fa2` 地址物化应在热路径消失，`jalr a5` 保留；若实施短 axis gate，短 shape 下本符号样本应整体显著下降（转入 scalar path）。
- 应出现（锚定第一性原理分析）：缓存后的热路径仅保留 struct spill + `jalr`；短 axis 路由后对应行不再产生 wrapper 样本。
- 要把该 hypothesis 升级为 profile-backed，最少需补采：`perf annotate --percent-type=global-period` 确认该 wrapper 在 workload 内的真实贡献；对 axis_size=16/32/128/384 的 shape 分别测 benchdnn_min_ms 找到 JIT 与 scalar 的 crossover；`perf evlist -v` 确认 precise_ip。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status / Anchor |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（载荷：`1/1 组；jit_rvv_softmax_f16_exp_sub_sum`） | ✅ |
| 2 | Phase 1 输出要求满足：7 行 baseline + L0 gate + bound gate；gap 标签 `baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行（载荷：7 行；3 个 gap 标签） | ✅ |
| 3 | Phase 3 输出要求满足：8 项 `Class selection trace`；`Classes scanned: rows-codegen.md`（另两组按第一级判据整组排除并声明）；零命中路径完整执行（证据充分性→class 覆盖→bound-type→negative-evidence→micro-analysis）；`No local pattern matched` 已声明；逐 row 排除表 27 行；hypothesis candidate 标 route≤Medium/impact=Low（载荷：零命中；27 行排除；micro-analysis 假设） | ✅ |
| 4 | Phase 4 输出要求满足（zero-match 分支）：根因蓝图基于 micro-analysis 假设；The fix 含方向性 before/after、correctness、风险、Profile signals；收益上界按 `hypothesis_only: 无 sample share` 语义表述（局部份额 100% 仅作观察上界）；Related PRs：无命中 pattern → 无表可引 | ✅ |
| 5 | 路径合规：8 项 trace 集合；零命中无硬凑 row；未把 hypothesis 写成 matched root cause；局部份额表述、无 workload 级 Amdahl（载荷：模式 A 零命中路径；class 列表） | ✅ |
| 6 | Phase 5 两侧锚定：消失侧=`c71fba lb`/`c71fbe fence r,rw`/`c71fc2 zext.b`/`c71fc6 beqz`/`c71f92/c71f96/c71fa2`；出现侧=缓存后热路径保留 `jalr`、短 axis 路由后 wrapper 样本消失 | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；未把 guard fence 建议为弱化同步语义；交付止于证据+蓝图+The fix+验证预测 | ✅ |

修正记录：无