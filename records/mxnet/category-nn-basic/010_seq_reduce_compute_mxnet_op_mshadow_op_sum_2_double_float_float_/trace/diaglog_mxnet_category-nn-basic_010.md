Functions under analysis: [`mxnet::op::broadcast::seq_reduce_compute<mxnet::op::mshadow_op::sum, 2, double, float, float, mxnet::op::mshadow_op::square, mxnet::op::mshadow_op::set_index_no_op<double, long> >(...) [clone ._omp_fn.0]`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`010-...square...-annotate.txt`，200 行，6798 samples，含 hot loop body `13de80e`–`13de85c`）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-nn-basic.txt`，testcase 级，非函数级）
- workload/binary/DSO/source context：已提供（`libmxnet.so`，DWARF 行号可映射至 `src/operator/mshadow_op.h:1823-1830` 与 `src/operator/tensor/broadcast_reduce-inl.h:364-371`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries` → `rv64i2p1_...v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0`）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（SG2044 profile：RVV 1.0，含 `zve32f/zve64f/zve64d/zvfh/zfa/zicond`）
- `vlenb`：已提供（profile：vlenb=16 → VLEN=128）
- 采样元数据（event / percent type / scope / 窗口）：已提供（`cpu-clock`，`percent: local period`，单次运行窗口）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0；`v` 存在；另含 `zve32f`/`zve64f`/`zve64d`（`vfwcvt.f.f.v` f32→f64 可用）、`zvfh`、`zfa`、`zicond`（出处：sg2044 hw profile） |
| Build ISA | `Tag_RISCV_arch = rv64i2p1_..._v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0`（出处：metadata `binaries`，libmxnet.so） |
| Vector flavor | annotate 内**零** `v*` 与零 `th.v*`（全 scalar），无 flavor mismatch |
| VLEN | VLEN=128 bits（vlenb=16，sg2044 profile） |
| Bound type | 函数级 bound type 缺失 → `baseline_gap: bound type`；testcase 级 IPC=0.567926、L1_dcache_load_miss_rate=0.974%、LLC_load_miss_rate=22.822%、branch_miss_rate=0.960%（出处：共享 perf_stat）。可选命令 `perf stat -e cycles,instructions,l1d_cache_refill -- <microbench>` |
| Sampling semantics | event=`cpu-clock`（可解释为时间 ✓）；percent type=**local period**（✗ 非 global-period）；同一运行窗口 ✓；函数级 workload 贡献未知 ✗ → 四项不齐，`baseline_gap: sampling metadata`，收益只能表述为「当前 sampled event 下的局部样本份额」 |
| Sampling IP precision | `precise_ip`/Exact-IP 能力未知 → `baseline_gap: sampling IP precision`；最高行只锚定 loop interval，不承担单指令 latency/cost 根因 |

L0 baseline gate ①：hardware 有 `v` 且 build 有 `v`（`v1p0` + `zvl128b1p0`）→ 无 mismatch，无最高优先级 baseline finding。L0 baseline gate ②：annotate 内无 `th.v*` → 不触发 flavor gate。Bound-type gate：`baseline_gap: bound type` + `baseline_gap: sampling metadata` → 本轮所有命中的 performance-impact confidence 因此封顶于 Medium。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个函数，`seq_reduce_compute<...,square,...>` 的 OpenMP outline clone）。

hot loop 边界：`13de80e` → `13de85c`（回边 `bne a0,a4,13de80e`），即 `broadcast_reduce-inl.h:364-371` 的 `for (size_t k = 0; k < M; ++k)` 串行归约体。循环外为 OpenMP 分块 setup（`13de754`–`13de7ce`）与收尾 `assign`（`13de860`–`13de874`）。

trace anchor（最高行原文）：
```
22.62 :   13de816:        fld     fa3,-80(s0)
21.87 :   13de854:        fsd     fa5,-72(s0)
```
Sampling IP precision 不允许单指令归因 → 上述两行只锚定 hot loop interval 的累加器访存机制，不声明单条 `fld`/`fsd` 的 cycle 成本。

## Phase 3 — Pattern scan / 模式扫描：seq_reduce_compute<sum, 2, double, float, float, square, set_index_no_op> [._omp_fn.0]

### Class selection trace（8 项）
1. `include — rows-operator-rvv.md`：compiler-generated scalar loop，对连续序列做加性归约（平方和），hot interval 零 `v*`，符合「尚未向量化的算子语义循环」。
2. `include — rows-codegen.md`：hot loop 每轮做 index scaling（`div`/`rem`/`mul`/`add`/`slli` 重建地址），构成独立的循环级归纳变量信号；且 `fcvt`/volatile 强制访存属指令形态问题。
3. `exclude — rows-asm.md`：symbol 为 C++ 模板实例化的 OpenMP outline clone（`._omp_fn.0`），无手写 `.S` provenance；无 policy/existence 四证。
4. `exclude — rows-string-memory.md`：hot interval 无 copy/fill/sentinel/two-input compare/checksum/back-reference 语义。
5. `exclude — rows-vectorized-tuning.md`：hot interval 零 `vsetvli`/`vle`/`vf*`，不是已向量化 RVV 循环。
6. `exclude — rows-offload.md`：SG2044/C920v2 无矩阵引擎或 P/DSP packed-SIMD 资源被本循环使用；无权重重排/多线程 GEMM 分块语义。
7. `exclude — rows-crypto.md`：evidence 未点名任何 AES/SHA/SM3/SM4/GHASH/CRC/GF(2^k) 原语。
8. `exclude — rows-runtime-os.md`：hot interval 在用户态算子代码，无 timer/ISR/CSR/PMP 证据。

`Classes scanned:` `rows-operator-rvv.md`（20 rows 逐行评估）、`rows-codegen.md`（27 rows 逐行评估）。

### Scan 表

| # | Row | Class | 关系 | 局部样本份额 |
|---|---|---|---|---|
| F1 | RVV Widening Additive Reduction Kernels | rows-operator-rvv | **primary** | 94.88% |
| F2 | Loop Induction Variable Strength Reduction | rows-codegen | **independent** | ≈4.65% |
| — | RVV Contiguous Elementwise Arithmetic Kernels | rows-operator-rvv | excluded（互斥） | — |
| — | RVV Normalization Kernels | rows-operator-rvv | excluded（互斥） | — |
| — | No vectorization (autovec gap / RVV kernel not built) | rows-operator-rvv | excluded（被更具体 row 认领） | — |
| — | Eliminate Unnecessary Float32/Float64 Conversions | rows-codegen | excluded（非冗余转换） | — |

#### F1 — RVV Widening Additive Reduction Kernels（primary）

**(a) 逐字 evidence 引用**（loop interval `13de80e`–`13de85c`）：
```
22.62 :   13de816:        fld     fa3,-80(s0)
21.87 :   13de854:        fsd     fa5,-72(s0)
11.43 :   13de840:        fcvt.d.s        fa5,fa5
11.30 :   13de848:        fadd.d  fa3,fa5,fa3
9.86 :   13de844:        fsub.d  fa5,fa5,fa2
9.03 :   13de850:        fsub.d  fa5,fa4,fa5
4.90 :   13de83c:        fmul.s  fa5,fa5,fa5
3.13 :   13de812:        fld     fa2,-72(s0)
0.74 :   13de84c:        fsub.d  fa4,fa3,fa4
```
DWARF 归属确认这是 `mshadow_op::sum::Reduce<double, double>(double volatile&, double, double volatile&)`（`src/operator/mshadow_op.h:1823-1830`）的 Kahan 补偿链，被 `broadcast_reduce-inl.h:364-371` 的 `for (size_t k = 0; k < M; ++k)` 逐元素调用。

**(b) 互斥邻居排除**
- **非 RVV Contiguous Elementwise Arithmetic Kernels**：该 row 行内互斥判据写明「跨元素 accumulator → 对应 reduction row」；本条 evidence 的核心正是跨元素 accumulator（`fld fa3,-80(s0)`/`fsd fa5,-72(s0)` 与 `fadd.d` 构成 loop-carried 链），且输出不是 lane-wise 映射，故归 reduction row。
- **非 RVV Normalization Kernels**：该 row 认领 Softmax/LayerNorm/RMSNorm 等「统计归约与逐元素变换耦合」的完整算子；本函数是纯统计子核（只算平方和并 `assign` 回 `small[idx]`，`broadcast_reduce-inl.h:390`），不含归一化除法，故不归它。
- **非 No vectorization**：Step 0d 规定只有不符合更具体 semantic row 时才进入该 row；本 evidence 已被 widening-reduction 语义 row 精确认领。
- **非 Eliminate Unnecessary Float32/Float64 Conversions**：`13de840: fcvt.d.s fa5,fa5` 的转换不是冗余——它是 `AType=double` 累加器对 `DType=float` 输入的语义要求的 widening（模板实参 `seq_reduce_compute<sum, 2, double, float, float, square, ...>` 已固定该语义）。

**(c) 双 Confidence 推导式**
`route: 直接 provenance（DWARF 映射到 mshadow_op.h:1823-1830 + broadcast_reduce-inl.h:364-371）+ 语义合同（加性归约 / 平方和）+ 互斥排除（3 项均以具体指令观察判别）→ High`
`impact: 缺 global-period 采样语义、缺函数级 bound type → Medium（sample share 94.88% 与 VLEN=128 已到位）`

#### F2 — Loop Induction Variable Strength Reduction（independent）

**(a) 逐字 evidence 引用**（同一 loop interval，整数侧）：
```
0.07 :   13de80e:        div     a5,a4,a1
0.03 :   13de81e:        rem     a5,a5,t1
1.24 :   13de82c:        mul     a3,a3,a6
1.79 :   13de830:        add     a5,a5,a2
0.57 :   13de832:        add     a5,a5,a3
0.29 :   13de7fc:        mul     a2,a2,s1
```
即 `broadcast_reduce-inl.h:365-366` 的 `coord = unravel(k, rshape); OP::Map(big[j + dot(coord, rstride)])` 每轮重建地址（1×`div` + 2×`rem` + 2×`mul` + `add`/`slli`）。对照观察：循环不变量 `13de7d4: div a5,t4,s2`（0.00%）已被编译器提到循环外，而依赖 `k` 的 `13de80e: div` 留在循环内——编译器已做完它能做的强度削减。

**(b) 互斥邻居排除**
- **非 Algebraic Simplification for Instruction Elimination**：该 row 只认领 peephole 级单指令代数恒等式；本条是循环级归纳变量演化（每轮 `div`/`rem` 重建 `unravel` 坐标），非恒等式化简。
- **非 Zero-Based Comparison for Register Pressure Reduction**：本 evidence 的比较对象 `bne a0,a4` 未呈现 zero-register 比较形态，问题在地址重建而非比较对象。
- **非 RVV Indexed Gather for Table Lookup**：`13de838: flw fa5,0(a5)` 的地址由归纳变量线性推导，不是数据相关索引 gather。

**(c) 双 Confidence 推导式**
`route: signal 成立（每轮 index scaling 逐字可见），但该 row 的 route gate 要求「实际访问是连续、固定 stride」，而 rshape/rstride 是运行期 Shape 实参（从 OpenMP capture 结构 `s6` 载入：`13de790-13de7b8`），annotate 无法证明 stride 固定 → 间接来源，route 降为 Medium`
`impact: 缺 global-period 采样语义 + 份额仅 4.65% + 该整数链当前被 FP 依赖链掩盖（94.88% 的 FP 序列主导同一 interval）→ Low`

### 多候选仲裁小段
F1 与 F2 的 evidence 可分账：F1 由 `fld`/`fsd`/`fadd.d`/`fsub.d`/`fcvt.d.s`/`fmul.s` 构成（94.88%），F2 由 `div`/`rem`/`mul`/`add` 构成（≈4.65%），指令类别与修复对象互不重叠 → **primary (F1) / independent (F2)**。

L0–L4 归属（按本次 evidence mechanism）：F1 属 **L1（循环携带依赖 + 逐元素标量归约 + 内存往返）**；F2 属 **L2（循环级地址重建开销）**。入口条件 A 排序：F1（94.88%）> F2（≈4.65%）。

**关键交互说明（避免误判优先级）**：F2 的整数地址链当前**不在关键路径上**——它被 F1 的串行 FP 依赖链（`fadd.d`/`fsub.d` 各 3–4 cycle 延迟，每元素 4 级串行）掩盖。因此不得把 F2 当作主要杠杆；F1 修复后 F2 才可能暴露为次级瓶颈。

## Phase 4 — Root-cause blueprint / 根因蓝图：seq_reduce_compute<sum, 2, double, float, float, square, set_index_no_op> [._omp_fn.0]

### F1（primary）— RVV Widening Additive Reduction Kernels

**Root cause**：`mshadow_op::sum::Reduce` 的**三个形参全部 `volatile` 限定**（`mshadow_op.h:1818`、`1823-1825`），迫使累加器 `val` 与补偿项 `residual` 每轮从栈往返；配合 `broadcast_reduce-inl.h:364` 的逐元素标量 Kahan 链，形成贯穿整个 reduce 长度的串行依赖。依据 `patterns/rvv_widening_reduction_kernels.md §Why this is slow`：①「本轮结果依赖上一轮，形成贯穿完整序列的串行链」（loop-carried accumulator dependency）；②「过早回写」——该文件明确要求「在寄存器内保留部分和并用 RVV reduction 合并」，而当前实现把部分和每轮写回内存。份额构成：volatile 栈往返 `fld fa3`(22.62%) + `fsd fa5`(21.87%) + `fld fa2`(3.13%) = **47.62%**；补偿算术 4 条 `fsub.d`/`fadd.d`(9.86+11.30+0.74+9.03) = **30.93%**；语义 widening `fcvt.d.s` = **11.43%**；f32 平方 `fmul.s` = **4.90%**。

**The fix / 修复方式**（与 `patterns/rvv_widening_reduction_kernels.md §The fix` 一致；以下为代表性代码形态，非可直接套用的补丁）：

修复前（现状）：
```cpp
// src/operator/mshadow_op.h:1823-1830 —— volatile 把累加器钉在栈上
template <typename AType, typename DType>
MSHADOW_XINLINE static void Reduce(volatile AType& dst,
                                   volatile DType src,
                                   volatile DType& residual) {
  DType y  = src - residual;      // fsub.d  fa5,fa5,fa2
  DType t  = dst + y;             // fadd.d  fa3,fa5,fa3
  residual = (t - dst) - y;       // fsub.d  x2
  dst      = t;                   // fsd 每轮写回栈
}
// src/operator/tensor/broadcast_reduce-inl.h:364-371
for (size_t k = 0; k < M; ++k) {
  coord      = mxnet_op::unravel(k, rshape);
  AType temp = OP::Map(big[j + mxnet_op::dot(coord, rstride)]);
  Reducer::Reduce(val, temp, residual);
}
```

修复后（RVV 形态，分两步、按风险递增实施）：
```cpp
// 步骤 1（无语义变更）：累加器去 volatile，保持寄存器驻留
//   → 直接消除 47.62% 的 fld/fsd 栈往返，Kahan 序列逐条不变
MSHADOW_XINLINE static void Reduce(AType& dst, DType src, AType& residual) {
  DType y  = src - residual;
  DType t  = dst + y;
  residual = (t - dst) - y;
  dst      = t;
}

// 步骤 2：分块向量化，保持稳定算法语义（不换成朴素 sum(x*x)）
//   顺序必须与现状一致：先在 f32 平方，再 widening 到 f64 累加
vfloat64m4_t vacc  = __riscv_vfmv_v_f_f64m4(0.0, vlmax);   // 主部分和
vfloat64m4_t vcomp = __riscv_vfmv_v_f_f64m4(0.0, vlmax);   // 补偿项
while (k < M) {
  const size_t vl = __riscv_vsetvl_e32m2(M - k);           // 动态 vl，不写死 lane
  vfloat32m2_t x  = __riscv_vle32_v_f32m2(big + base + k, vl);   // unit-stride
  vfloat32m2_t sq = __riscv_vfmul_vv_f32m2(x, x, vl);      // 保持 f32 平方语义
  vfloat64m4_t d  = __riscv_vfwcvt_f_f_v_f64m4(sq, vl);    // 语义要求的 widening
  /* 维持 Kahan/Neumaier 补偿更新：vcomp 与 vacc 均为向量寄存器 */
  vacc = __riscv_vfadd_vv_f64m4(vacc, d, vl);
  k += vl;
}
// 末尾只在寄存器内做一次有序水平归约
vfloat64m1_t r = __riscv_vfredosum_vs_f64m4_f64m1(vacc, seed, vlmax);
```

适用前提与 correctness contract（不可破坏）：
1. **f32 平方语义必须保留**：现状 `13de83c: fmul.s` 先于 `13de840: fcvt.d.s`，即 `sqr<float>` 在 **f32 精度**下平方后才 widening。若改为「先 widening 再在 f64 平方」，`|x| > ~1.8e19` 的溢出行为与 reference 不同 → 语义变更，禁止。
2. **稳定算法不得削弱**：`patterns/rvv_widening_reduction_kernels.md §9` 明确要求「NRM2 一类 scaled sum-of-squares 算法必须继续保持 scale/ssq 更新、Inf/NaN 和 overflow/underflow 防护，不能用朴素 sum(x*x) 换性能」。若项目要求与标量 reference 位一致，必须保留补偿结构或用 **ordered** `vfredosum`，不得默认 `vfredusum`（unordered 改变加法结合顺序）。
3. **空输入与 tail**：`M==0` 时须返回 `SetInitValue` 的 0（`mshadow_op.h:1874-1883`），不得读到未初始化寄存器；tail 用运行时 `vl`。
4. **最终归约范围**：跨多个数据块的长期 accumulator 不得用最后一次循环的 `vl` 做最终归约（pattern §3）。
5. 限制/风险：`vfwcvt.f.f.v` 需 `ELEN≥64`——本目标有 `zve64d` ✓；f32m2→f64m4 的 EMUL 关系需按 destination 预算（`references/kernel-conventions.md §2`）；LMUL 与多 accumulator 数量属调优项，不是根因。

修复后预期 Profile signals：`13de816: fld fa3,-80(s0)` 与 `13de854: fsd fa5,-72(s0)` 消失；`13de83c`–`13de854` 的 9 条标量 FP 序列折叠为向量体。

**Baseline facts 回填**：hardware ISA = RVV 1.0 + `zve32f/zve64f/zve64d`（SG2044/C920v2）；build ISA = `..._v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0`；VLEN = 128 bits；bound type = `baseline_gap: bound type`（函数级缺失）。

**收益上界**：**94.88%**（当前 `cpu-clock`/local-period 采样下的函数内局部样本份额）。因 `baseline_gap: sampling metadata`（percent type=local period，函数级 workload 贡献未知），**不得**表述为 workload 级 Amdahl 上界。

**三维路由判定**：
- `current source`：compiler-generated scalar code（C++ 模板实例化 + OpenMP outline clone，DWARF 映射到 `mshadow_op.h:1823-1830` 与 `broadcast_reduce-inl.h:364-371`）→ 非手写 `.S`。
- `implementation existence/reachability`：无 RVV 实现存在；本 DSO 已按含 `v` 的 ISA 构建（`zvl128b1p0`），无 dispatch gate 问题 → route 不冻结。
- `function-level policy`：无任何 policy 要求该归约由独立 `.S` 承载；不走 missing-`.S` 分支。

**Related PRs（RVV Widening Additive Reduction Kernels）**：16 条
https://github.com/opencv/opencv/pull/27096 ・ https://github.com/opencv/opencv/commit/33d632f85e4c1cc4d70bc7210c2f126fa8a4b0bb ・ https://github.com/opencv/opencv/pull/26624 ・ https://github.com/uxlfoundation/oneDNN/pull/5361 ・ https://github.com/uxlfoundation/oneDNN/commit/a95f0060cfcb75aeee7b937f18dbb2ff32064f50 ・ https://github.com/openjdk/jdk/commit/72297d22d19e34ff26bd34644dc087a1dec9527e ・ https://github.com/openjdk/jdk/commit/08a2f841ec78a10f8d6d54b2ac3a92e89f765f14 ・ https://github.com/openjdk/jdk/commit/2c1e4c381615ce52276f4bf331a1e7a845af4b6e ・ https://github.com/openjdk/jdk/pull/20910 ・ https://github.com/openjdk/jdk/commit/134b63f0e8c4093f7ad0a528d6996898ab881d5c ・ https://github.com/openjdk/jdk/pull/16629 ・ https://github.com/openjdk/jdk/commit/1aebab780c5b84a85b6f10884d05bb29bae3c3bf ・ https://github.com/openjdk/jdk/commit/1b6281d98cf0e7c5435c563bfedd6f07b79bfa62 ・ https://github.com/OpenMathLib/OpenBLAS/commit/c37509c213a34a8cae449ededd7bc7064675ecc4 ・ https://github.com/OpenMathLib/OpenBLAS/commit/3918d8504e7720d94221025ae6078a2459ccb104 ・ https://github.com/alibaba/MNN/pull/4433

### F2（independent）— Loop Induction Variable Strength Reduction

**Root cause**：`broadcast_reduce-inl.h:365-366` 在每轮迭代重建 `unravel(k, rshape)` 坐标并 `dot(coord, rstride)` 求偏移，编译为每元素 1×`div` + 2×`rem` + 2×`mul` + `add`/`slli`（`13de80e`/`13de81e`/`13de82c`/`13de830`/`13de832`/`13de7fc`）。依据 `patterns/loop_induction_variable_strength_reduction.md §Why this is slow`：「index-based loop 每轮把 `i` 乘 stride 再加 base，增加 integer pipeline 压力」；该文件 §4 指出在 OoO 核（本文参考目标 SG2044/C920v2 即乱序）上更短的 dependency chain 才让访存更易重叠。

**The fix / 修复方式**：把依赖 `k` 的 `unravel`/`dot` 索引链替换为**预计算 stride + 递增偏移**（stride 固定时）或**分轴递增计数器 + 条件回绕**（stride 不固定但可分轴时），循环内只保留 `ptr += stride` 与预计算 end 比较。修复前/后示意：
```cpp
// before: 每轮 div/rem/mul/add 重建地址
coord      = mxnet_op::unravel(k, rshape);
AType temp = OP::Map(big[j + mxnet_op::dot(coord, rstride)]);

// after（stride 固定时）：预计算步长与终止指针
const index_t step = mxnet_op::dot(unravel(1, rshape), rstride); // 已证明等价
const float* p   = big + j;
const float* end = p + M * step;
for (; p != end; p += step) { AType temp = OP::Map(*p); Reducer::Reduce(val, temp, residual); }
```
correctness contract：仅当 reduce 轴的 stride 在循环内不变（或各轴 stride 固定、可用增量计数器精确回绕）时等价；`M` 的边界与 `rshape[i]==1` 的退化维必须逐一证明，否则禁止替换。限制/风险：当前该链不在关键路径上，单独修复收益有限。预期 Profile signals：`13de80e: div`、`13de81e/13de822: rem`、`13de82c: mul`、`13de7fc: mul` 从循环体消失，回边比较改为指针/end 比较。

**Baseline facts 回填**：同 F1（hardware/build ISA 含 `v`；VLEN=128；bound type = `baseline_gap: bound type`）。

**收益上界**：**≈4.65%**（local-period 局部样本份额；`baseline_gap: sampling metadata`，不得称 workload 级上界）。

**三维路由判定**：`current source` = compiler-generated scalar；`implementation existence/reachability` = N/A（不依赖 kernel 存在性）；`function-level policy` = 无。

**Related PRs（Loop Induction Variable Strength Reduction）**：3 条
https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d ・ https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df ・ https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec

## Phase 5 — Verification forecast / 验证预测：seq_reduce_compute<sum, 2, double, float, float, square, set_index_no_op> [._omp_fn.0]

**F1（primary，先验证）**
- 应消失/缩小：`22.62 : 13de816: fld fa3,-80(s0)` 与 `21.87 : 13de854: fsd fa5,-72(s0)`（累加器栈往返）；`11.30 : 13de848: fadd.d` / `9.86 : 13de844: fsub.d` / `9.03 : 13de850: fsub.d` / `0.74 : 13de84c: fsub.d` 四条标量补偿链折叠进向量体。
- 应出现（依据 `patterns/rvv_widening_reduction_kernels.md §Verification`）：`vsetvli`、`vle32.v`（unit-stride 载入）、`vfmul.vv`（f32 平方，保持原精度语义）、`vfwcvt.f.f.v`（f32→f64 widening）、`vfadd.vv`/`vfmacc.vv`（f64 累加）、`vfredosum.vs` 或 `vfredusum.vs`（水平归约，要求确定性时取 ordered）。
- 数值对照：与标量 reference 比较平方和（含最大长度、极值输入、全零输入）；若采用 unordered 归约，须确认最后一位差异在项目容差内。

**F2（independent）**
- 应消失/缩小：`0.07 : 13de80e: div a5,a4,a1`、`0.03 : 13de81e: rem a5,a5,t1`、`1.24 : 13de82c: mul a3,a3,a6`、`0.29 : 13de7fc: mul a2,a2,s1` 从循环体移除。
- 应出现（依据 `patterns/loop_induction_variable_strength_reduction.md §Verification`）：循环体内仅剩 `add`/`flw` 与指针-上界比较（`bne p,end`），不再有 `div`/`rem`/`mul` 地址重建。

**验证顺序**：按收益上界 F1（94.88%）→ F2（≈4.65%）。F2 单独验证时需注意其收益可能因仍受 F1 依赖链掩盖而不可见。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`seq_reduce_compute<sum, 2, double, float, float, square, set_index_no_op> [._omp_fn.0]` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行结论 + gap 标签：`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行；两个 L0 gate 判定 + bound-type gate 结论 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 2 个（F1/F2）；F1 evidence 锚点 `22.62 : 13de816: fld fa3,-80(s0)`、`21.87 : 13de854: fsd fa5,-72(s0)`，F2 锚点 `1.79 : 13de830: add a5,a5,a2`、`1.24 : 13de82c: mul a3,a3,a6`；supporting 0；排除 4 条；推导式 2 条 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 `patterns/rvv_widening_reduction_kernels.md`（row: RVV Widening Additive Reduction Kernels；引用首词「本轮结果依赖上一轮」「过早回写」「NRM2 一类 scaled sum-of-squares」）+ `patterns/loop_induction_variable_strength_reduction.md`（row: Loop Induction Variable Strength Reduction；引用首词「index-based loop 每轮把」）；`The fix` 含 before/after、correctness（f32 平方语义、ordered/unordered、M==0/tail、最终归约范围）、风险、预期 Profile signals；Related PRs 16 条 + 3 条 |
| 5 | 路径合规 | ✅ | 模式 A（profile_backed）；路径 = 8 项 trace 可解释扫描集（include 2 / exclude 6）；primary/independent 关系与 L1/L2 归属合规；每个 leaf 均来自通过 gate 的 row；按动态份额排序 F1>F2；`th.v*` 未出现，未全局停扫 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 `13de816: fld fa3,-80(s0)`、`13de854: fsd fa5,-72(s0)`、`13de80e: div a5,a4,a1` 对上 Phase 3(a)；出现侧标注 `patterns/rvv_widening_reduction_kernels.md §Verification` 与 `patterns/loop_induction_variable_strength_reduction.md §Verification` |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；无契约外追问；交付物止于 Profile 证据 / 根因蓝图 / 完整 `The fix` / 验证预测 |

修正记录：无
