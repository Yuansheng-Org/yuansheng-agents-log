# RISC-V Performance Root-Cause Report — mxnet cumsum_forward (category-misc, rank 003)

Functions under analysis: [`bool mxnet::op::mxnet_op::Kernel<mxnet::op::cumsum_forward, mshadow::cpu>::Launch<float*, float*, unsigned long, unsigned long> (._omp_fn.0)`，即 `cumsum_forward::Map` 的 hot scan loop]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 perf annotate：已提供（`003-bool mxnet：：op：：mxnet_op：：Kernel＜mxnet：：op：：cumsum_forward, mshadow：：cpu＞：：Launch＜float＊, float＊, unsigned long, unsigned long＞(ms-2c03fef87a31-annotate.txt`；`cpu-clock`，2086 samples，`percent: local period`；hot loop body 完整覆盖）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-misc.txt`；IPC 0.658，branch/cache/L1/LLC counters）
- workload/binary/DSO/source context：已提供（`libmxnet.so`；仓库 `apache/mxnet` commit `b84609d3fc73d20929c114eab95faaa56e6c5ede`；源码 `src/operator/numpy/np_cumsum-inl.h` `struct cumsum_forward::Map` 76–92 行、`src/operator/mxnet_op.h` `Kernel<cpu>::Launch` 1012–1032 行）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata snapshot `libmxnet.so-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`）
- hardware ISA（cpuinfo / hwprobe）：已提供（metadata cpuinfo isa = `rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`）
- `vlenb`：已提供（`vector.vlen_bits = 128`，vlenb = 16）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock`，freq=99；annotate percent=local period；函数样本 2086 与全运行样本 13578 来自同一 perf.data（`raw/metadata.txt`：`report_fallback=perf-script`）；函数级贡献可计算）
- Sampling IP precision：缺失（`baseline_gap: sampling IP precision`；precise_ip/Exact-IP/PMU skid 能力未提供）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：含 `v`（RVV 1.0）、`zve32f/zve64d/zve64f`、`zvfh/zvfhmin`、`zba/zbb/zbc/zbs`、`zfh/zfhmin`；无 `th.v*`（SG2044 / XuanTie C920v2，OoO，64 cores） |
| Build ISA | 已提供：libmxnet.so `Tag_RISCV_arch` 含 `v1p0`、`zve64d1p0`、`zvl128b1p0` —— 与 hardware 一致，无 hardware/build mismatch；无 IFUNC/multiversion 证据 |
| Vector flavor | RVV 1.0（build `v1p0` + hardware `v`）；annotate hot loop **zero `v*`、zero `th.v*`**（全 scalar）—— 这是 no-vectorization 信号，不是 flavor mismatch |
| VLEN | 128 bits（vlenb=16；`zvl128b`）；SEW=32 时 m1=4 lane / 寄存器组 |
| Bound type | IPC 0.658（较低）；`cache_references`≈`cache_misses`（5.3614e9 vs 5.3614e9）为 counter 异常/别名，不能解读为 100% miss；L1_dcache_load_miss_rate 2.57%、LLC_load_miss_rate 38.85%、branch_miss_rate 1.46%。热点 loop 内部呈 latency-bound：`flw` 56.38% 而 `fadd.s` 0.00%（串行前缀链在等 load），非带宽型 memory-bound 主导 |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；annotate percent=local period；函数样本 2086 / 全运行 13578（同一窗口）→ 函数级 workload 贡献 ≈ **15.4%**（近似，fallback report 属性） |
| Sampling IP precision | `baseline_gap: sampling IP precision`；单行占比只锚定 loop interval，不做 instruction-latency 归因 |

L0 baseline gate：hardware 有 `v` 且 build 有 `v` → **无 mismatch**；无 `th.v*` → flavor gate 不触发（硬件实际支持 RVV 1.0）。Bound-type gate：不判定为明确 memory-bound（cache counters 异常，热点为 latency-bound serial scan），本地 RVV 向量化 fix 不被 memory-bound 覆盖。

## Phase 2 — Scope / 分析边界

函数清单：`batch-003-function-002`（rank 003）—— `bool mxnet::op::mxnet_op::Kernel<mxnet::op::cumsum_forward, mshadow::cpu>::Launch<float*, float*, unsigned long, unsigned long>(mshadow::Stream<mshadow::cpu>*, unsigned long, float*, float*, unsigned long, unsigned long) [clone ._omp_fn.0]`，其 hot 区间为内联的 `mxnet::op::cumsum_forward::Map<float, float>` 的 j-loop。

hot basic block / loop interval 边界：
- 内层 scan loop（j over `middle`）：`28da572`–`28da594`（源行 88–89：`lane_out[j * trailing] = lane_out[(j - 1) * trailing] + OType(lane_in[j * trailing])`）
- 外层 lane loop（i over 线程分片）：`28da550`–`28da594`

hot loop 最高行原文（trace anchor）：`56.38 :  28da57e:  flw fa4,0(a4)`（hot j-loop，stride 输入加载）

Sampling IP precision 未确认 → 该行只作为 loop interval 的 trace anchor，不承担单指令 latency 归因；区间聚合 + def-use（`fadd.s fa5,fa5,fa4` 的 loop-carried 链）交叉佐证。

annotate 覆盖完整（hot loop body 完整，无 `annotate_gap`）；入口模式 A（profile-backed）。

## Phase 3 — Pattern scan / 模式扫描：mxnet::op::cumsum_forward::Map（Kernel::Launch<float*,float*,…> ._omp_fn.0）

### Class selection trace

1. `rows-asm.md` — **exclude** — 当前代码来源是 compiler-generated（`np_cumsum-inl.h` C++ template `Map` 内联进 OMP worker `._omp_fn.0`），无 `.S` provenance；无 policy/existence 四证（mxnet CPU 算子无独立 `.S` 要求、无 dispatch slot/缺失实现证据）。
2. `rows-operator-rvv.md` — **include** — 第一级判据：compiler/intrinsic 生成代码，hot 是尚未向量化的算子语义循环（cumsum scan），hardware+build 均含 `v` 而 hot loop 全 scalar。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/sentinel/compare/checksum/back-reference 原语；是 FP 前缀累加 scan。
4. `rows-vectorized-tuning.md` — **exclude** — 完整 annotate 无任何 `v*`（全 scalar），该 class 修正对象要求已有 RVV 的配置/寄存器/展开问题。
5. `rows-codegen.md` — **include** — compiler-generated code；逐行核对指令形态/分派 row（addressing fusion、IV strength reduction、resource scheduling、register pressure 等）。
6. `rows-offload.md` — **exclude** — 无矩阵引擎（IME/AME）、packed-SIMD、权重重排层、portable-RVV、线程分块算术证据；target 为 C920v2（RVV 1.0）。
7. `rows-crypto.md` — **exclude** — 非 AES/SHA/SM/GHASH/CRC/GF(2^k) 原语。
8. `rows-runtime-os.md` — **exclude** — 热点在用户态 `libmxnet.so` 算子内，非 timer/ISR/特权路径。

Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: mxnet::op::cumsum_forward::Map

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| No vectorization（primary） | hot main loop 全 scalar（`flw`/`add`/`fadd.s`/`fsw`/`bne`），zero `v*`；hardware 有 `v`（RVV 1.0）且 build `Tag_RISCV_arch` 含 `v1p0`/`zvl128b1p0`；loop-carried `fadd.s` 串行链使 autovec 无法向量化 j-loop | High | Medium | `patterns/no-vectorization.md` |

**(a) 逐字 evidence 引用**（hot j-loop `28da572`–`28da594`，loop interval over `middle`）：
```
56.38 :  28da57e:  flw fa4,0(a4)      // lane_in[j*trailing] 的 stride 加载（最高占比行）
23.06 :  28da58e:  add a4,a4,a1       // 输入指针步进（stride = trailing*4）
14.05 :  28da594:  bne a3,a0,28da57a  // j-loop 回边分支
 4.75 :  28da582:  add a2,a5,a1       // 输出地址计算
 1.34 :  28da590:  fsw fa5,0(a2)      // 写回前缀
 0.38 :  28da57a:  flw fa5,0(a5)      // 重载上一轮刚写入的 accumulator（store→load 往返）
 0.00 :  28da588:  fadd.s fa5,fa5,fa4 // loop-carried 累加（前缀链）
```
整个函数区间（`28da4fe`–`28da5b0`）zero `v*`/`th.v*`。外层 lane loop `28da550`–`28da594` 与内层 j-loop 均全 scalar。归属：内层 j-loop 指令占比合计 ≈ 100%（0.38+56.38+4.75+23.06+1.34+14.05+0.05≈100.01%），证明 hot 区间即 scan 本体。

**(b) 互斥邻居排除**：
- **RVV Widening Additive Reduction**：cumsum 是 prefix-sum **scan**（每步写回前缀、保留全部中间结果），不是单值归约；pattern 文件 §When to apply 明列「存在跨元素递推或前缀依赖的扫描」不适用 → 排除。
- **RVV Strided Memory Access / Layout Transforms**（stride 固定、`add a4,a4,a1` 23.06%）：语义合同是 scan（每 lane 前缀依赖 + FP 累加），不是矩阵转置/规则列抽取/通道块搬运等布局变换；cross-lane 向量化会把 strided 访问变成 contiguous `vle32`，即 strided 形态是 scalar 逐 lane 实现的**症状**而非独立根因（因果消除测试）→ 排除。
- **RVV Contiguous Elementwise Arithmetic**：存在跨元素 loop-carried 依赖，非 lane-independent → 排除。
- **Load/Store Addressing-Mode Fusion**（rows-codegen.md）：`add a4,a4,a1`/`add a2,a5,a1` 是固定 stride 指针步进；RISC-V 标量寻址只有 reg+imm，无 reg+reg 可折叠 operand → 排除。
- **Loop Induction Variable Strength Reduction**（rows-codegen.md）：热循环已使用指针递增 + 分支比较（`add a4,a4,a1` + `bne`），已是强度削减形态；无 `slli+add` index scaling → 排除。
- **Resource-Aware Instruction Scheduling**：不能仅凭核名（OoO）断言 scheduler 模型错误；依赖链是 scan 算法固有 → 排除。
- **Register Pressure / Save-Restore**：hot loop 无 spill（除 accumulator store→load 往返外无 stack 流量）→ 排除。
- **Kernel Selection / Runtime ISA Dispatch**：源码中不存在被绕过的 RVV/vector kernel（`Kernel<cumsum_forward>::Launch` 直接走通用 template），无 dispatch 缺口 → 排除。
- 其余 operator/string/codegen row（elementwise activation/normalization/precision/extrema/arg-extrema/gather/packing/color/matmul/quantized/triangular/conv/pool/resample/FFT/fixed-separable）：语义合同均不匹配 scan 结构 → 整组排除。

**(c) 双 Confidence 推导式**：
- `route:` compiler-generated scalar loop（provenance：`np_cumsum-inl.h` template 内联进 `._omp_fn.0`）+ hardware `v` ∧ build `v1p0`/`zvl128b`（metadata 直接证据）+ hot loop zero `v*` + loop-carried 串行依赖解释 autovec 为何不生成 RVV + 上述互斥排除 → **High**
- `impact:` 函数 sample share ≈15.4%（2086/13578，同窗口）✓、VLEN=128 ✓、bound type 部分（IPC 0.658，cache counter 异常，loop 为 latency-bound）、`baseline_gap: sampling IP precision`、存在竞争瓶颈（串行前缀链、访存延迟、OpenMP 分片）→ **Medium**

### 多命中仲裁小段

仅一个顶层 finding（primary：No vectorization）。strided 访问（`add a4,a4,a1` 23.06%）与 accumulator store→load 往返（`fsw` 1.34% + `flw` 0.38%）是同一 scalar 实现机制下的**症状**，不满足各自独立 row gate，按 supporting 症状写入 primary 证据内，不另列顶层 finding、不计命中数、不单独验证。evidence mechanism layer：L1（vectorization / semantic dispatch）。证据 sample share 加总：hot j-loop ≈ 函数内 100%（2086 samples 全部落在该区间）。入口条件 A → 单一 finding，无同层 independent 排序问题。

## Phase 4 — Root-cause blueprint / 根因蓝图：mxnet::op::cumsum_forward::Map

**命中 row**：`rows-operator-rvv.md` 的 No vectorization row（唯一 gate 通过者）；leaf = `patterns/no-vectorization.md`。

**1. Root cause**：`no-vectorization.md` §Why this is slow 的关键机制句：*"vector unit 没有处理 hot main-loop 的并行元素"*，且 *"zero `v*` 只证明当前 loop 没有 RVV execution"*（依据 `patterns/no-vectorization.md` §Why this is slow）。具体机制：`cumsum_forward::Map`（`np_cumsum-inl.h:76-92`）对每个 lane 沿 `middle` 轴、stride=`trailing` 做串行前缀和 —— `lane_out[j*trailing] = lane_out[(j-1)*trailing] + lane_in[j*trailing]`，`fadd.s fa5,fa5,fa4`（`28da588`）构成贯穿整个 `middle` 的 loop-carried 依赖；`Kernel<cpu>::Launch`（`mxnet_op.h:1013-1032`）用 `#pragma omp parallel for` 按 lane 分片，每个 worker 逐 lane、逐 j 纯标量执行。硬件 VLEN=128 / SEW=32 时单个 m1 寄存器组即可容纳 4 个 lane；把 4–16 个 lane 的同一 scan 步打包进向量寄存器（`vle32`+`vfadd.vv`+`vse32`）后，串行 `fadd` 链被平摊到向量宽度，同时 `flw`/`fadd.s`/`fsw`/地址步进/分支的逐元素重复开销按向量宽度下降 —— 这就是 hardware `v` + build `v` 却完全没被利用的根因（autovec 无法向量化 loop-carried scan，需要结构改写）。

**2. The fix / 修复方式**（与 `no-vectorization.md` §The fix 一致：确认 build/hardware 后按语义改写，而非改 build flag；本函数 build 已含 `v1p0`，无需重建）：把逐 lane 的 `Map` 调用改为**跨 lane 批处理**的 RVV intrinsic 循环。修复前/后形态（SEW=32，VLEN=128，I/O 均为 float32，每 lane 内加法顺序不变）：

```cpp
// Before（np_cumsum-inl.h:83-90，per-lane scalar，annotate 28da57a-28da594）：
index_t left = i / trailing, right = i % trailing;
index_t offset = left * middle * trailing + right;
const IType* lane_in = in + offset;
OType* lane_out = out + offset;
lane_out[0] = OType(lane_in[0]);
for (index_t j = 1; j < middle; ++j) {
  lane_out[j * trailing] = lane_out[(j - 1) * trailing] + OType(lane_in[j * trailing]);
}

// After（跨 lane 批处理，VLEN-agnostic；在 Launch 的 lane 分片内按 vl 批处理）：
// 每批覆盖 [i, i+vl) 个连续 lane；同一 left 组内 consecutive right 在内存中连续，
// 故 scan 步 j 的输入 in[i + j*trailing .. i+j*trailing+vl-1] 是 contiguous vl 个元素。
while (i < end) {
  const size_t vl = __riscv_vsetvl_e32m1(end - i);          // tail: runtime vl
  const float* in0  = in + i;                                // lane 基址（right=i%trailing）
  const float* out0 = out + i;
  vfloat32m1_t acc = __riscv_vle32_v_f32m1(in0, vl);         // lane_out[0] = lane_in[0]
  __riscv_vse32_v_f32m1(out0, acc, vl);
  for (index_t j = 1; j < middle; ++j) {
    vfloat32m1_t x   = __riscv_vle32_v_f32m1(in0 + (size_t)j * trailing, vl);
    acc = __riscv_vfadd_vv_f32m1(acc, x, vl);                // 每 lane 内仍是串行前缀
    __riscv_vse32_v_f32m1(out0 + (size_t)j * trailing, acc, vl);
  }
  i += vl;
}
```

适用前提与 correctness contract：
- 每 lane 内 j 的加法顺序与 scalar reference 完全一致（`acc = acc + x` 逐 j 顺序执行）→ FP 结果 bit-exact，不引入 reassociation / `vfred*` 顺序问题。
- boundary 处理：当 `right + vl > trailing`（lane 块跨 left 组边界）时 contiguous 假设断裂（组间跳 `middle*trailing`）；当 `trailing >= VLMAX(SEW,LMUL)` 时块永不出界；出界块按组内剩余 lane clamp 或转 scalar/掩码回退。
- 退化情形：`axis=None`（`middle = out.Size()`、`trailing=1`、N=1 lane）时跨 lane 批处理退化为单 lane，需换沿 scan 轴的并行 scan 算法（如 Hillis-Steele `vadd`+`vslide` 两遍法）—— 这是**不同向量化轴**，需单独 shape 判定。
- 寄存器/LMUL：live set 为 `acc`+`x` 两向量 → m1/m2/m4 均在 `kernel-conventions.md` §2 预算内（m4 用 8 个向量寄存器）；`vsetvl` 移出 j-loop（fixed vtype）后 j-loop 内无 per-iteration vsetvl。
- OpenMP 分片结构保留（每个 worker 批处理其 lane 区间），并行度不丢失。

限制/风险：`middle` 很小时每批的 `vsetvl` + 循环开销可能吞噬收益（需 crossover benchmark）；stride/trailing 与 VLEN 关系未知导致边界分支；真实 benchmark 输入 shape（`opperf_cumsum_input_0/1/2` = 4,194,304 元素）未公开，收益以实测 shape 为准。

修复后预期 Profile signals：hot j-loop 中出现 `vsetvli`/`vle32`/`vfadd.vv`/`vse32`；`flw fa4,0(a4)`、`add a4,a4,a1`、`bne` 的标量份额大幅下降；accumulator 的 store→load 往返（`fsw`+`flw fa5`）消失（acc 常驻向量寄存器）。

**3. Baseline facts 回填**：hardware ISA = `rv64imafdcv_…_zve64d…_zvfhmin…`（有 `v`，RVV 1.0，OoO C920v2）；build ISA = libmxnet.so `Tag_RISCV_arch = rv64i2p1…v1p0…zvl128b1p0`（有 `v`）；VLEN = 128 bits；bound type = 非明确 memory-bound（IPC 0.658；cache counters 异常；hot loop latency-bound serial scan）。

**4. 收益上界**：当前 sampled event（cpu-clock）下的函数级局部样本份额 **≈15.4%**（2086/13578，同一运行窗口）。采样语义四条全部成立（event 可解释为时间、全局份额来自同窗口、函数贡献已知），可给**条件性 workload 级 Amdahl 上界**：`saving ≈ 15.4% × (1 − 1/S)`，其中 `S` 为 hot loop 加速比（VLEN=128/SEW=32 时 lane 平摊理论上限 m1=4×、m4=16×，受访存带宽、串行链 latency 与 OpenMP 分片限制，须实测）。注：全运行样本来自 fallback report（perf script 合成），份额为近似值。

**5. 三维路由判定**：
- `current source`：compiler-generated（C++ template `cumsum_forward::Map` 内联进 `._omp_fn.0`，无 `.S`）；证据 = annotate 标量指令形态 + 源码行号对应。
- `implementation existence/reachability`：无既有 RVV/vector kernel（`Kernel<cumsum_forward>::Launch` 直接执行通用 template 路径），无 dispatch 缺口。
- `function-level policy`：mxnet CPU 算子无独立 `.S` 载体的官方 policy 证据 → 不进入 missing `.S` 分支；修复载体为 intrinsic/普通 C++ 结构改写。

**6. Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。对应 shape 要点已并入 §The fix：work domain = 跨 lane 扫描（向量化轴 = lane，SEW=32）；`W` = 动态 lane 数 N，`LMUL_min(W): N/A`，按 peak live set（acc+x = 2 向量）选 m1/m2/m4；tail = runtime-VL lane 尾 + trailing 边界块；`axis=None` 单 lane 退化沿 scan 轴需并行 scan 算法。

**7. Related PRs**：`patterns/no-vectorization.md` 本地表，19 条 URL：
Related PRs：19 条 URL — OpenCV [#22179](https://github.com/opencv/opencv/pull/22179)、[#22520](https://github.com/opencv/opencv/pull/22520)、[#23980](https://github.com/opencv/opencv/pull/23980)、[#24058](https://github.com/opencv/opencv/pull/24058)、[#24132](https://github.com/opencv/opencv/pull/24132)、[#24166](https://github.com/opencv/opencv/pull/24166)、[#24301](https://github.com/opencv/opencv/pull/24301)、[#24325](https://github.com/opencv/opencv/pull/24325)、[#27160](https://github.com/opencv/opencv/pull/27160)、[#27119](https://github.com/opencv/opencv/pull/27119)、[#27097](https://github.com/opencv/opencv/pull/27097)、[#27007](https://github.com/opencv/opencv/pull/27007)、[#26958](https://github.com/opencv/opencv/pull/26958)、[#26865](https://github.com/opencv/opencv/pull/26865)、commit [b902a8e792e1](https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d)、[2c16f3b7d2b2](https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a)、[e06502a254f7](https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706)、[a2d784b6f53a](https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f)、[83104bed3209](https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a)

## Phase 5 — Verification forecast / 验证预测：mxnet::op::cumsum_forward::Map

**应消失/缩小**（锚定 Phase 3(a) 引用行）：
- `56.38 :  28da57e:  flw fa4,0(a4)`（stride 输入加载）→ 被 `vle32` 替代，份额消失/大幅下降
- `23.06 :  28da58e:  add a4,a4,a1`（stride 指针步进）→ 地址算术并入向量访存形态
- `14.05 :  28da594:  bne a3,a0,28da57a`（j-loop 标量回边）→ 迭代数按向量宽度下降
- `0.00 :  28da588:  fadd.s fa5,fa5,fa4` 与 `fsw fa5,0(a2)`+`flw fa5,0(a5)`（accumulator store→load 往返）→ 由 `vfadd.vv` + acc 常驻向量寄存器取代

**应出现**（锚定 `patterns/no-vectorization.md` §Verification）：
- 同一函数的 annotate 出现 `vsetvli`、`vle32`、`vse32`、`vfadd.vv` 等 RVV 指令；scalar 指令不再主导 hot loop
- 与 scalar reference 逐位对照：FP 结果 bit-exact（每 lane 内加法顺序不变）
- 覆盖形状：空输入、小 `middle`、`trailing < VLMAX`（边界块）、`trailing >= VLMAX`、lane 尾块、`axis=None` 单 lane 退化
- 代表输入 benchmark：报告 cycles/element 或 throughput；对照 m1/m2/m4 与 crossover
- 真实 workload annotate 确认 RVV 路径被采用（而非仅微基准）

入口条件 A：按收益上界顺序验证（单一 primary finding，无排序问题）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅ `1/1 组；Kernel<cumsum_forward, mshadow::cpu>::Launch<float*,float*,unsigned long,unsigned long> (._omp_fn.0)` |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + L0 gates + bound-type gate；含 Sampling IP precision） | ✅ `7 行 + 2 L0 gate + bound gate；gap 标签：[baseline_gap: sampling IP precision]` |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace；Classes scanned；三件套；仲裁小段） | ✅ `Classes scanned: [rows-operator-rvv.md, rows-codegen.md]；顶层 finding 数=1（No vectorization，High/Medium）；evidence 锚点：[28da57e flw fa4,0(a4) 56.38%、28da58e add a4,a4,a1 23.06%、28da594 bne a3,a0,28da57a 14.05%、28da588 fadd.s fa5,fa5,fa4 0.00%]；supporting 数=0（2 症状归入 primary）；排除条数=8 class trace + 9 条邻居/row 排除；推导式条数=1` |
| 4 | Phase 4 输出要求满足 | ✅ `已读 pattern: [patterns/no-vectorization.md]；命中 row: No vectorization（rows-operator-rvv.md）；引用短语首词：[vector unit 没有处理 hot main-loop 的并行元素]；The fix before/after/correctness/风险/Profile signals 齐备；Related PRs：19 条 URL` |
| 5 | 路径合规：模式 A + 单 primary（L1 layer）+ 8 项 trace 扫描集；非 `.S` 分支 | ✅ `模式=profile_backed；路径=no-vectorization（非 missing .S）；class=[rows-operator-rvv.md, rows-codegen.md]；th.v* 未触发` |
| 6 | Phase 5 两侧锚定 | ✅ `消失侧=Phase 3(a) 引用行 4 条；出现侧=no-vectorization.md §Verification（vsetvli/vle32/vse32/vfadd.vv）` |
| 7 | 契约边界合规：无实施询问/代码修改/补丁生成；无向用户追问 | ✅ `交付物止于 Profile 证据、根因蓝图、完整 The fix、验证预测` |

修正记录：无