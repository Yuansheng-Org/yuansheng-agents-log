Functions under analysis: [void vPowx<float>(int, float const*, float, float*)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`011-void vPowx＜float＞(int, float const＊, float, float＊)-annotate.txt`，DSO `libcaffe.so.1.0.0`，event=`cpu-clock`，`percent: local period`，6 samples，含 hot loop body）
- perf stat（可选 bound/context）：已提供（`12-caffe-benchmark-riscv-bvlc_alexnet.txt`：cycles=56,413,930,048；instructions=129,543,036,792；branches=4,984,941,186；branch_misses=7,963,448；L1_dcache_loads=53,701,921,459；L1_dcache_load_misses=150,729,109；LLC 系 NA）
- workload/binary/DSO/source context：已提供（caffe @ commit 9b891540183ddc834a02b2bd81b31afae71b2153；testcase bvlc_alexnet；热点函数为 `mkl_alternate.hpp` 中宏 `DEFINE_VSL_UNARY_FUNC_WITH_PARAM(Powx, y[i] = pow(a[i], b))` 的模板实例，经 `caffe_powx<float>`→`vsPowx` 调用；source 行 47–62 直证语义）
- readelf -A（承载热点地址 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries.caffe-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`）
- hardware ISA（/proc/cpuinfo / riscv_hwprobe）：已提供（SpacemiT X100 / K3 snapshot；isa 含 `v`、`zve32f`、`zve64f`、`zvbb`、`zvbc`、`zvfh`、`zvk*`、`zfa` 等）
- `vlenb`：已提供（hardware profile snapshot：`vlen_bits=256`，`vlenb=32`）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock`，`percent: local period`，单次运行同一窗口；函数级 workload 贡献未知）
- Sampling IP precision（precise_ip / Exact-IP）：缺失（annotate 未含 `precise_ip` 信息，PMU skid 能力未知 → `baseline_gap: sampling IP precision`）

样本量声明：本函数 annotate 仅 6 samples。按契约，样本量写入本清单并进入 confidence 推导（结论降级、结构不降级）。

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | SpacemiT X100（K3 SoC），OoO 乱序执行；`isa` 含标准 RVV 1.0 `v` 及 `zvbb/zvbc/zvfh/zvk*` 等扩展（hwprobe snapshot 直证） |
| Build ISA | 承载热点地址的 `libcaffe.so.1.0.0`：`Tag_RISCV_arch` 含 `v1p0` 与 `zve32f/zve64f`（build 已具备 RVV 1.0）；fixed-VLEN 声明为 `zvl128b`（与硬件 VLEN 256 不一致，见下） |
| Vector flavor | 本函数 annotate 内 zero `v*`、zero `th.v*`（全 scalar），无 flavor mismatch 可言；hot loop 未使用 RVV |
| VLEN | 硬件 `vlenb=32` → VLEN=256 bits（SEW=32、LMUL=1 时 8 lanes）；build 侧 `zvl128b` 声明 128-bit 固定下限 → 无 flavor mismatch（同 RVV 1.0 flavor），但记录 build/hardware VLEN 差：向量化修复须用 scalable `vsetvlmax` 或精确 `-march=rv64gcv_zvl256b` 重建，否则 256-bit 硬件只能吃 128-bit 配置 |
| Bound type | compute-bound：IPC = 129.54G/56.41G ≈ 2.30；L1D load miss rate 0.281%、branch miss rate 0.160% → 非 memory-bound 非 branch-bound；本地 RVV rewrite 是第一杠杆候选 |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；`percent: local period`（非 global-period）；单运行窗口；函数占 workload 总耗时未知 → 四项中两项不成立 → `baseline_gap: sampling metadata`，禁止称 cycle/workload 级 Amdahl 上界，收益只能表述为「当前 sampled event 下的函数内局部样本份额」 |
| Sampling IP precision | `precise_ip`/Exact-IP/skid 能力未知 → `baseline_gap: sampling IP precision`；单条指令占比只能锚定所属 loop interval，不承担 instruction-latency 归因 |

L0 baseline gate 1（hardware 有 `v` 而 build 无 `v`）：不成立 —— hardware 与 build 均含 `v1p0`；不是 mismatch。构建为含 V 的二进制，但编译器未向量化本循环（powf 为外部 libm 调用，非 `-ffast-math` + vector-libm 不可自动向量化）。继续扫描全部可见 row。
L0 baseline gate 2（`th.v*` flavor gate）：annotate 无 `th.v*`、无 mismatch，无 flavor 冻结。
Bound-type gate：compute-bound 成立，无 memory-bound 压制；本地向量化修复的 performance-impact confidence 不受 memory 门压制。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：1 个函数 `void vPowx<float>(int, float const*, float, float*)`（符号地址 `0x231b74`，DSO `libcaffe.so.1.0.0`，rank 011 / index 4 / total 6）。

hot basic block / loop interval 边界：`0x231bbc`–`0x231bd8`（scalar 逐元素循环：load → spill b → 调 powf@plt → store → reload b → 指针递增 → backward branch）。6/6 个函数内 sample 全部落在此区间，annotate 覆盖完整（hot loop body 已覆盖）。

trace anchor（最高占比行，cpu-clock / local period）：
```
50.00 :   231bbc: flw     fa0,0(s1)          # a[i] 载入（3/6 samples）
33.33 :   231bc6: auipc   ra,0xffeb2        # powf@plt 调用序列 setup（2/6 samples）
16.67 :   231bce: fsw     fa0,0(s2)          # y[i] 写回（1/6 samples）
```
该区间完整指令序列（逐字，含调用与 spill/reload）：
```
231bbc: flw     fa0,0(s1)
231bc0: fsw     fa1,-168(s0)     # scalar 参数 b 每轮 spill
231bc4: addi    s1,s1,4
231bc6: auipc   ra,0xffeb2
231bca: jalr    1162(ra) # e4050 <powf@plt>
231bce: fsw     fa0,0(s2)
231bd2: flw     fa1,-168(s0)     # b 每轮 reload
231bd6: addi    s2,s2,4
231bd8: bne     s1,s3,231bbc
```
`Sampling IP precision` 未确认 → 单行占比只锚定该 loop interval；根因结论收敛到 interval-level mechanism（标量逐元素 powf 调用循环）。

冷区间（prologue CHECK_GT / CHECK(a) / CHECK(y) 错误路径与 stack canary epilogue，`0x231b74`–`0x231bb2`、`0x231bfa`–`0x231cd8`）全部为 0.00%，不参与热点归因。

## Phase 3 — Pattern scan / 模式扫描：void vPowx<float>(int, float const*, float, float*)

### Class selection trace（8 项）
1. `rows-operator-rvv.md` — **include** — compiler-generated scalar 逐元素循环；source 语义 `y[i]=pow(a[i],b)` 为 lane-independent 算子合同；hot main loop 全 scalar、zero `v*`。
2. `rows-codegen.md` — **include** — compiler-generated 指令形态；可独立观察的每轮 PLT 调用序列（`231bc6: auipc`+`231bca: jalr`→`powf@plt`）与每轮 spill/reload（`231bc0: fsw fa1,-168(s0)` / `231bd2: flw fa1,-168(s0)`）；须评估 call/spill 形态 row 后再归因。
3. `rows-string-memory.md` — **exclude** — 本循环是 float 逐元素 pow 算术，非 copy/fill/sentinel/compare/checksum/back-reference 语义。
4. `rows-vectorized-tuning.md` — **exclude** — 该组要求 annotate 已有 `v*`（修正已向量化 loop 的 RVV 配置）；本函数 zero `v*`。
5. `rows-asm.md` — **exclude** — 当前代码来源为 compiler-generated C++ 模板（`mkl_alternate.hpp:47-62` 宏实例），非手写 `.S`；caffe 无 SIMD/assembly-default 策略、无 CPU 侧 dispatch slot，policy-backed missing `.S` 四证不成立。
6. `rows-offload.md` — **exclude** — 无矩阵引擎 / packed-SIMD / 权重重排 / 可移植层 / 并行分块参与。
7. `rows-crypto.md` — **exclude** — 无 AES/SHA/SM/GHASH/CRC 等密码学原语信号。
8. `rows-runtime-os.md` — **exclude** — 用户态库代码，非 timer/CSR/特权路径。

### Classes scanned: rows-operator-rvv.md, rows-codegen.md

### Local performance pattern scan: `void vPowx<float>(int, float const*, float, float*)`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Elementwise Activation Kernels（primary） | 逐元素 lane-independent 超越函数 pow；sample 主导在标量 load（`flw` 50.00%）、数学 helper 调用 setup（`auipc` 33.33% → `powf@plt`）与 store（`fsw` 16.67%）；无跨 lane statistics | High | Medium | `patterns/rvv_elementwise_activation_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`；只解释缺少向量执行，不决定贡献载体 | — | — | `patterns/no-vectorization.md` |

### 顶层 finding 三件套

**(a) 逐字 evidence 引用**（所属 loop interval `0x231bbc`–`0x231bd8`）：
```
50.00 :   231bbc: flw     fa0,0(s1)
33.33 :   231bc6: auipc   ra,0xffeb2
16.67 :   231bce: fsw     fa0,0(s2)
```
`231bc6` 与其后继 `231bca: jalr 1162(ra) # e4050 <powf@plt>` 构成每元素一次的 libm 调用序列；`231bc0/231bd2`（b 的 spill/reload）在同一 interval 内。

**(b) 互斥邻居排除**：
- vs `RVV Contiguous Elementwise Arithmetic Kernels`：该 row 行内互斥判据「mask/分段/**超越函数** → elementwise-activation row」；本行 pow 为超越函数，且有 `231bca: jalr … <powf@plt>` 与 source `y[i]=pow(a[i],b)`（`mkl_alternate.hpp:62`）直证 → 归 activation row，不归 arithmetic row。
- vs `RVV Normalization Kernels`：loop 内无 max/sum/variance 等跨 lane 统计归约，仅 per-lane load→call→store → 排除。
- vs `No vectorization`：该 row 行内互斥「operator semantic shape → 各自更具体 row；本行只认领 generic scalar main loop」；本循环已被更具体的 elementwise-activation semantic row 认领 → 降为 supporting，不独立顶层计数。
- vs `rows-codegen.md` 的 call/spill 形态 rows（`Forced Inlining for Hot Specialization Helpers`、`Register Pressure and Save/Restore Optimization`）：`powf` 是外部 libm 符号，非 translation-unit-local specialization helper，hot-helper-inlining row 的 TU-local 前提不成立；每轮 b spill/reload 由外部调用边界（fa1 caller-saved）引入，且 spill 指令 sample 为 0.00%，属于「数学 helper 主导」的同一机制症状 → 排除为独立根因（由 primary activation row 认领）。
- vs `Kernel Selection and Runtime Specialization`：caffe 该路径无 dispatch slot、无 fallback 选择证据，当前函数即直接实现 → 排除。
- vs `Floating-Point Semantic Lowering`：powf 是真实 IEEE pow 语义的外部库调用，非编译器为 rounding/NaN/FCSR 语义做的 lowering → 排除。

**(c) 双 Confidence 推导式**：
- `route: 当前代码来源=compiler-generated scalar（mkl_alternate.hpp:51 宏直证）+ hardware V（hwprobe）+ build v1p0（Tag_RISCV_arch）+ lane-independent transcendental + 数学 helper 主导（6/6 local samples 在调用循环区间）→ High`。
- `impact: 热点区间局部样本份额 6/6=100%（cpu-clock, local period），VLEN=256 已知，bound=compute-bound 已知；但采样元数据为 local period 且函数级 workload 贡献未知、样本数仅 6、Sampling IP precision 未知 → 缺采样语义项 → Medium（不封 Low：sample share 与 bound/VLEN 到位；不升 High：采样语义四条不全成立）`。

### 多命中仲裁小段
- **primary**：`RVV Elementwise Activation Kernels`（evidence mechanism 层 **L1**，operator semantic）。修复对象：将逐元素 scalar powf 调用循环改写为 RVV vector-math 管线（含 b 特殊值 dispatch）。
- **supporting**：`No vectorization`（同 L1 层）。解释同一机制的「缺少向量执行」方面；不进顶层计数、不单独排序。
- 归属按地址分账：6/6 local samples 全部落在同一 loop interval `[0x231bbc, 0x231bd8]`；无第二独立热点地址集。
- 因果消除测试：若上层改写为向量化 pow 管线（消除每轮 `powf@plt` 调用），b 的每轮 spill/reload 与 call setup sample 自然消失 → 上层 activation row 为 primary，codegen call/spill 形态不另立 root cause。
- 收益上界排序（入口条件 A）：仅 1 个顶层 finding，本 finding 的 evidence sample share 合计 = 100% 函数内局部样本份额；同层无需排序。companion 白名单（glibc semantic+integration）不适用：本 annotate 热点在 caffe 侧循环，不在 glibc 内部实现。

## Phase 4 — Root-cause blueprint / 根因蓝图：void vPowx<float>(int, float const*, float, float*)

**依据 pattern：`patterns/rvv_elementwise_activation_kernels.md`（row: RVV Elementwise Activation Kernels，primary）+ `patterns/no-vectorization.md`（supporting）。**

1. **Root cause**：`vPowx<float>` 的整条 hot loop 是标量逐元素 `powf` 调用循环。依据 `rvv_elementwise_activation_kernels.md` §Why this is slow：「逐元素 activation 的根因是 lane-independent 工作仍以标量分支/数学 helper 执行」，主要 leverage 是「满足数值合同的 vector-math pipeline」。本函数 6/6（100%）函数内局部 sample 落在 `[0x231bbc,0x231bd8]` 的 load→`powf@plt`→store 循环；每元素一次 libm PLT 调用（`auipc`+`jalr`，含调用约定引发的 b 每轮 spill/reload），vector unit 完全闲置（zero `v*`）。依据 `no-vectorization.md` §Why this is slow：`VLMAX = LMUL × VLEN / SEW`，本 loop 的并行元素处理为零，`flw`(50%)/`auipc`(33%)/`fsw`(17%) 即标量逐元素调用循环的开销分布。Build 侧虽然 `Tag_RISCV_arch` 含 `v1p0`，但 `powf` 为外部 libm 调用，无 `-ffast-math`+vector-libm 时 GCC 不会自动向量化 → 编译器保持标量循环。

2. **The fix / 修复方式**（与 `rvv_elementwise_activation_kernels.md` §4「Treat transcendental activations as vector-math pipelines」及 `no-vectorization.md` §The fix、`kernel-conventions.md` §2/§3 一致）：
   - 修复对象：`mkl_alternate.hpp` 的 `DEFINE_VSL_UNARY_FUNC_WITH_PARAM` 生成循环（`y[i] = pow(a[i], b)`），用带扩展宏保护的 RVV intrinsic kernel 替换标量循环，并在循环外对 `b` 做低成本特殊值 dispatch。
   - 修复前（当前标量形态）：
     ```cpp
     // mkl_alternate.hpp:51  for (int i = 0; i < n; ++i) { y[i] = pow(a[i], b); }
     ```
   - 修复后（RVV vector-math 管线示意形态；`b` 特殊值快速通道 + 一般通道）：
     ```cpp
     // 特殊值 dispatch（循环外一次标量比较，避免一般路径代价）
     if (b == 1.0f)      { for (int i=0;i<n;++i) y[i] = a[i]; }                    // identity
     else if (b == 2.0f) { for (int i=0;i<n;++i) y[i] = a[i] * a[i]; }             // 可向量化为 vfmul.vv
     else if (b == 0.5f) { for (int i=0;i<n;++i) y[i] = sqrtf(a[i]); }             // 可向量化为 vfsqrt（须校验语义，见风险）
     else {
       int i = 0;
       while (i < n) {
         size_t vl = __riscv_vsetvl_e32m4(n - i);          // scalable，硬件 VLEN=256 自动取满
         vfloat32m4_t x = __riscv_vle32_v_f32m4(a + i, vl);
         vfloat32m4_t r = /* pow(x, b) = exp2(b * log2(x))：调用项目经精度验证的 RVV 向量 exp2/log2 管线，
                             或项目已验收的向量化 libm vpowf 实现 */;
         __riscv_vse32_v_f32m4(y + i, r, vl);
         i += vl;
       }
     }
     ```
   - 适用前提：SEW=32；LMUL 按 `kernel-conventions.md` §2 live-vector budget 选择（VLEN=256、e32m1=8 lanes，m4=32 lanes 需确认无 spill）；长规则 `n` 用 fixed-VL main loop + runtime-VL tail（§3），短/控制流重时用单层 strip-mined `vsetvl(n-i)`；`#if defined(__riscv_v_intrinsic) && defined(__riscv_v)` 扩展保护，非 V 目标回退标量。
   - **不可破坏的 correctness contract**：`y[i] = pow(a[i], b)` 的 IEEE powf 语义（glibc `powf`）；`CHECK_GT(n,0); CHECK(a); CHECK(y)`（冷路径保留）；in-place alias（`y==a`）安全——chunk 内先整体 load 后整体 store，逐 lane 无写后读冲突；向量分解 `exp2(b*log2(x))` 必须满足项目数值合同（ULP/domain/NaN/±Inf/±0/subnormal，`vector-math-conventions.md` §Numeric contract），Caffe 训练/推理重现性不得被破坏。
   - **限制/风险**：① `powf` 无单一 RVV 指令，「exp2/log2 两段超越函数」与标量 powf 的误差特性不同，须与 reference 全 domain 扫描比对（`vector-math-conventions.md` §Verification）；② `b==0.5f` 快速通道用 `sqrtf` 与 `powf` 在 `-0.0`、负底数、NaN 上的语义不完全一致（如 `pow(-0.0,0.5)` 为域错误 NaN vs `sqrt(-0.0)=-0.0`），须按项目 reference 校验或仅对非负输入启用；③ small tensor crossover：`n` 很小（低于阈值）时标量可能更快，需 benchmark 定 crossover；④ 一般通道若采用 `exp2/log2` 分解，每个元素超越函数工作量增加，收益主要来自向量摊销与消除每元素调用——必须实机 A/B 验证；⑤ build 侧 `zvl128b` vs 硬件 VLEN 256：scalable `vsetvlmax` 可直接利用 256-bit，若坚持 fixed-VL 则建议以精确 `-march=rv64gcv_zvl256b` 重建；⑥ tail/mask lane 若只控制写回，inactive lane 仍可能执行超越函数并触发异常，需 control/dataflow 证明（`vector-math-conventions.md`）。
   - 修复后预期变化的 Profile signals：`[0x231bbc,0x231bd8]` 区间 `flw/auipc/jalr powf@plt/fsw` 的 sample 显著下降或消失；重新 annotate 出现 `vsetvli`、`vle32`、`vse32` 与向量数学指令序列；函数 cycles/element 改善（以 benchmark 为准）。

3. **Baseline facts 回填**：hardware ISA=SpacemiT X100 RVV 1.0（含 `zvbb/zvbc/zvfh/zvk*`）；build ISA=`rv64i…v1p0…zvl128b…`（含 V，fixed-VLEN 128 声明）；VLEN=256 bits（`vlenb=32`）；bound type=compute-bound（IPC≈2.30，L1D miss 0.281%、branch miss 0.160%）。

4. **收益上界**：入口条件 A——「当前 sampled event（cpu-clock, local period）下的函数内局部样本份额」= 100%（6/6，全部落在本 hot loop interval）。受 `baseline_gap: sampling metadata`（local period、函数级 workload 贡献未知）约束，**不是** workload 级 Amdahl 上界，不得外推为 cycle/workload 耗时份额。本函数为批内 rank 011/6，全 workload 中占比未知，排序按批内 rank 处理。

5. **三维路由判定**：
   - current source：compiler-generated scalar C++（`mkl_alternate.hpp:47-62` 宏 `DEFINE_VSL_UNARY_FUNC_WITH_PARAM` 实例化，`libcaffe.so.1.0.0` 内 symbol `0x231b74`）。
   - implementation existence/reachability：本源码树内 `vPowx/caffe_powx` 无任何 RVV/kernel 实现、无 CPU 侧 dispatch slot（MKL alternate 全部为标量宏）→ 目标实现缺失；修复路径为新增 intrinsic vector-math kernel（普通代码/intrinsic 载体，由函数级合同决定，非 `.S`）。
   - function-level policy：caffe 无 assembly-default/SIMD-first 策略，policy-backed missing `.S` 四证不成立 → 不进 missing-`.S` 分支；no-vectorization 仅作 supporting。
   - （Implementation-shape proof 仅限 policy-backed missing `.S`，本蓝图不适用。）

6. **Related PRs 小节**：
   - `patterns/rvv_elementwise_activation_kernels.md`：Related PRs：12 条 URL（opencv `524d8ae01c63`、`86241653a781`、`#27072`；oneDNN `aa23ab557391`、`d5ae44880903`；llama.cpp `#17227`、`#15057`；MNN `#4508`、`#4484`、`#4044`、`b7268aa`、`#4359`）。
   - `patterns/no-vectorization.md`：Related PRs：19 条 URL（opencv `#22179`、`#22520`、`#23980`、`#24058`、`#24132`、`#24166`、`#24301`、`#24325`、`#27160`、`#27119`、`#27097`、`#27007`、`#26958`、`#26865`、`b902a8e792e1`、`2c16f3b7d2b2`、`e06502a254f7`、`a2d784b6f53a`、`83104bed3209`）。

## Phase 5 — Verification forecast / 验证预测：void vPowx<float>(int, float const*, float, float*)

对 primary（及随附 supporting）的验证，按单假设归因：

**应消失/缩小**（锚定 Phase 3(a) 引用行）：
- `231bbc: flw fa0,0(s1)`（50.00%）、`231bc6: auipc ra,0xffeb2` + `231bca: jalr 1162(ra) # e4050 <powf@plt>`（33.33%）、`231bce: fsw fa0,0(s2)`（16.67%）所在 scalar 循环的 sample 应显著下降或消失；`231bc0/231bd2`（`fsw/flw fa1,-168(s0)` 每轮 b spill/reload）随每元素调用消失而消失。

**应出现**（锚定 `rvv_elementwise_activation_kernels.md` §Verification 与 `no-vectorization.md` §Verification）：
- 同一函数 re-annotate 出现 `vsetvli`、`vle32.v`、`vse32.v` 及向量数学（`vfmul`/`vfsqrt`/exp2-log2 管线）序列；scalar 指令不再主导 hot loop。
- 正确性合同：与标量 reference 对比，动态 `n` 覆盖 `n=0`、小 `n`、main-loop（fixed-VL step）整倍数及全部 tail length（1..step-1）；固定 `b` 特殊值（1.0/2.0/0.5）覆盖 `-0.0`、负底数、NaN/±Inf、subnormal；全 domain 最大 ULP/relative error 扫描（`vector-math-conventions.md` §Verification）；tail/mask lane 与 FRM/FCSR 状态核对。
- 性能判据：与 scalar reference 在短/中/长输入上 benchmark，报告 cycles/element 或吞吐改善；small-`n` crossover 确认 dispatch 阈值合理。
- 可达性：非 V 目标正确回退标量路径（扩展宏 gate）；真实 workload（含 bvlc_alexnet 中实际调用 `caffe_powx` 的路径）命中向量路径。
- 可选补充数据（仅用于提升 confidence，不阻塞）：`precise_ip`/Exact-IP 重采以支持 instruction-level 归因；global-period + 函数级 sample share 以获得 workload 级收益上界。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`void vPowx<float>(int, float const*, float, float*)` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline + 2 个 L0 gate + bound gate；gap 标签：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1 个（primary: RVV Elementwise Activation Kernels），supporting 1（No vectorization）；evidence 锚点 `50.00 : 231bbc: flw fa0,0(s1)`、`33.33 : 231bc6: auipc ra,0xffeb2`、`16.67 : 231bce: fsw fa0,0(s2)`；排除 6 条（arithmetic/normalization/no-vectorization 降 supporting/hot-helper-inlining/register-pressure/kernel-selection/FP-semantic-lowering）；推导式 2 条 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`rvv_elementwise_activation_kernels.md`（row: RVV Elementwise Activation Kernels；引用短语首词：`逐元素 activation 的根因是 lane-independent`、`满足数值合同的 vector-math pipeline`、`Treat transcendental activations as vector-math pipelines`）、`no-vectorization.md`（row: No vectorization；引用短语首词：`VLMAX = LMUL × VLEN / SEW`）；`The fix` 含 before/after、适用前提、correctness contract（IEEE powf + alias + ULP）、风险（b==0.5 语义、exp2/log2 误差、crossover、zvl128b vs 256、tail 异常）、预期 Profile 信号；missing `.S` 未进入；Related PRs：12+19 条 URL |
| 5 | 路径合规 | ✅ | 模式 A（profile-backed）；路径 L1 operator-semantic primary + supporting；class 列表 `rows-operator-rvv.md, rows-codegen.md`；`th.v*` 无、未全局停扫；按动态份额排序（唯一顶层 finding，函数内局部份额 100%） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧锚点 `231bbc: flw fa0,0(s1)` / `231bc6: auipc ra,0xffeb2` / `231bce: fsw fa0,0(s2)`；出现侧 `rvv_elementwise_activation_kernels.md §Verification` + `no-vectorization.md §Verification` |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成、无契约外分支；交付止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测 |

修正记录：无