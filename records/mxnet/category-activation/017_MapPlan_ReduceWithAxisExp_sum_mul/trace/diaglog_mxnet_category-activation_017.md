Functions under analysis: [void mshadow::MapPlan<mshadow::sv::saveto, mshadow::Tensor<mshadow::cpu, 2, float>, 2, float, mshadow::expr::ReduceWithAxisExp<mshadow::red::sum, mshadow::expr::BinaryMapExp<mshadow::op::mul, mshadow::Tensor<mshadow::cpu, 3, float>, mshadow::Tensor<mshadow::cpu, 3, float>, float, 1>, float, 3, false, 2>>(mshadow::TRValue<mshadow::Tensor<mshadow::cpu, 2, float>, mshadow::cpu, 2, float>*, mshadow::expr::Plan<...> const&) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（rank 017 `MapPlan<sv::saveto, Tensor<cpu,2,float>, 2, float, ReduceWithAxisExp<red::sum, BinaryMapExp<mul, Tensor<cpu,3,float>, Tensor<cpu,3,float>, float, 1>, float, 3, false, 2>> [clone ._omp_fn.0]`，140 samples，event=`cpu-clock`，`percent: local period`；覆盖完整函数体，含 inner k-reduction loop）
- perf stat（可选 bound/context）：已提供（整 testcase；IPC=0.562、LLC_load_miss_rate=43.99%、L1_dcache_load_miss_rate=5.16%、branch_miss_rate=2.72%）
- workload/binary/DSO/source context：已提供（`libmxnet.so`，mxnet master commit b84609d3fc73d20929c114eab95faaa56e6c5ede；mshadow `expr/expression.h` 的 MapPlan/ReduceWithAxisExp 表达式求值器 —— 双输入 3D→2D 按轴 sum-of-products 归约；当前实现是 compiler-generated C++ 模板）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（`libmxnet.so-elf-A`：`rv64i2p1_..._v1p0_..._zvl128b1p0`，含 `v`、`zvl128b`）
- hardware ISA：已提供（`rv64imafdcv_..._zve64d_zvfh...`，含 `v`）
- `vlenb`：已提供（16 bytes → VLEN=128 bits）
- 采样元数据：已提供（event=`cpu-clock`，`percent: local period` 非 global-period；workload 贡献未知）→ Phase 1 采样语义 gate
- Sampling IP precision：缺失（无 precise_ip/Exact-IP 标记）→ `baseline_gap: sampling IP precision`

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zve64d_zvfh...`：暴露标准 `v`（RVV 1.0），VLEN=128 |
| Build ISA | `Tag_RISCV_arch` = `rv64i2p1_..._v1p0_..._zvl128b1p0`：含 `v`、`zvl128b`；无 IFUNC/multiversion |
| Vector flavor | 全函数 zero `v*`/`th.v*`（全 scalar）→ 无 flavor mismatch |
| VLEN | `vlenb`=16 → VLEN=128 bits；SEW=32 时 VLMAX(m1)=4 lanes |
| Bound type | 内层 k 归约 loop 由 loop-carried volatile accumulator 往返（flw/fadd.s/fsw 栈槽 -84(s0)）、每元素 div/rem 索引解码与串行 FP 链主导（latency-bound）；全局 IPC=0.562、LLC miss 43.99% 为竞争性约束 |
| Sampling semantics | event=`cpu-clock`；percent type=`local period`（非 global-period）→ 只有函数内局部份额；禁止 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision` → 单行只锚定 loop interval |

L0 baseline gate：hardware 含 `v`、build 含 `v` → 无 mismatch。`th.v*` gate：不适用。Bound-type gate：latency-bound，向量化修复的 impact 受 FP 顺序语义与调用上下文约束。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：仅 `MapPlan<sv::saveto, Tensor<cpu,2,float>, 2, float, ReduceWithAxisExp<red::sum, BinaryMapExp<mul, Tensor<cpu,3,float>, Tensor<cpu,3,float>, float, 1>, float, 3, false, 2>> [._omp_fn.0]`。

语义：`dst[y][x] = Σ_{k=0..size_-1} src1[z] * src2[z]`（`z = (x*size_+k)*trailing_+y`，`src 内部再按 last_ 解码为二维索引 z/last_、z%last_`）—— 双输入 3D tensor 沿 axis 的 sum-of-products（点积式）归约，结果 2D。外层 y-loop（shape[0]，OpenMP 分片）+ x-loop（shape[1]）+ 内层 k-loop（size_，归约）。

hot loop 边界与 trace anchor（inner k-reduction loop）：
- **Loop interval（11bf510 – 11bf548）**：`11bf510: div a5,a3,a1`（z/last_ 索引解码）、`11bf518: addi a2,a2,1`（50.71%）、`11bf51a: rem s7,a3,a1`（z%last_）、`11bf52e: flw fa5,0(a4)`（3.57%，src1 load）、`11bf532: add a5,a5,s7`（8.57%，src2 地址）、`11bf538: flw fa3,0(a5)`（0.71%，src2 load）、`11bf53c: fmul.s fa5,fa5,fa3`（4.29%）、`11bf540: fadd.s fa5,fa5,fa4`（4.29%）、`11bf544: fsw fa5,-84(s0)`（27.86%，volatile accumulator 回写栈槽）、`11bf548: bne a2,a6,11bf510`（back-edge）。
- 最高占比行：`50.71 :  11bf518:  addi    a2,a2,1`；interval 样本合计 = 50.71+27.86+8.57+3.57+4.29+4.29+0.71 ≈ **100.0%**（140 样本 local period）。
- 关键机制：`mshadow::red::sum::Reduce<float>(float volatile&, float)`（源码 `dst += src;`）—— volatile accumulator 强制每迭代 `flw fa4,-84(s0)`/`fadd.s`/`fsw fa5,-84(s0)` 栈内存往返（过早回写），叠加每元素 div/rem 索引解码（11bf510/11bf51a）与串行 FP 链 → latency-bound。

Sampling IP precision 不足 → 结论收敛到 interval-level mechanism。annotate 覆盖完整。入口模式 A（profile_backed）。

## Phase 3 — Pattern scan / 模式扫描：void mshadow::MapPlan<sv::saveto, Tensor<cpu,2,float>, 2, float, ReduceWithAxisExp<red::sum, BinaryMapExp<mul, ...>>>

### Class selection trace（8 项）

1. `rows-asm.md` — exclude：compiler-generated C++ 模板（OpenMP clone），无手写 `.S`/policy 四证。
2. `rows-operator-rvv.md` — include：compiler-generated scalar loop + 明确归约语义（sum-of-products 按轴归约，点积式加性累加）。
3. `rows-string-memory.md` — exclude：非 string/memory/copy/compare 语义。
4. `rows-vectorized-tuning.md` — exclude：全函数 zero `v*`。
5. `rows-codegen.md` — include：compiler-generated 指令形态需逐 row 排除（volatile accumulator 往返、div/rem 索引解码、循环控制）。
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD 信号。
7. `rows-crypto.md` — exclude：无密码原语。
8. `rows-runtime-os.md` — exclude：无 timer/ISR/CSR 信号。

### Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: `MapPlan<ReduceWithAxisExp<sum, mul>>`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Widening Additive Reduction Kernels（primary） | sum-of-products 按轴归约（点积式累加）：标量 accumulator 逐元素累加，且 volatile 合同强制过早回写（`27.86 : 11bf544: fsw fa5,-84(s0)` + `0.00 : 11bf514: flw fa4,-84(s0)`），loop 控制与索引算术主导（`50.71 : 11bf518: addi`、`8.57 : 11bf532: add`、每元素 `div`/`rem` 11bf510/11bf51a）；zero `v*` | High | Low | `patterns/rvv_widening_reduction_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`；hardware+build 均含 `v` | — | — | `patterns/no-vectorization.md` |

**顶层 finding 三件套（primary — RVV Widening Additive Reduction Kernels）：**

(a) **逐字 evidence 引用**（inner k-loop interval 11bf510–11bf548）：
- `50.71 :  11bf518:  addi    a2,a2,1` —— k-loop 循环计数（interval 主导行）；
- `27.86 :  11bf544:  fsw     fa5,-84(s0)` 与 `0.00 :  11bf514:  flw     fa4,-84(s0)` —— `red::sum::Reduce(float volatile&)` 的栈槽 accumulator 逐迭代回写/重载（过早回写）；
- `4.29 :  11bf540:  fadd.s  fa5,fa5,fa4` —— 标量加性归约；
- `3.57 :  11bf52e:  flw     fa5,0(a4)` 与 `0.71 :  11bf538:  flw     fa3,0(a5)` —— src1/src2 逐元素 load；
- `0.00 :  11bf510:  div     a5,a3,a1` 与 `0.00 :  11bf51a:  rem     s7,a3,a1` —— 每元素索引解码（z/last_、z%last_）。

语义链（source：mshadow `expr/expression.h` `ReduceWithAxisExp::Eval`）：`out[y*stride+x] = Σ_k src1[z/last_]*src2[z%last_...]`，`z=(x*size_+k)*trailing_+y`；`red::sum::Reduce<float>(float volatile&, float)` 为 volatile 累加（源码 `dst += src;`）。

(b) **互斥邻居排除**：
- normalization row：排除 —— 行内互斥 "Softmax/LayerNorm/RMSNorm 完整算子 → normalization row"；本函数内无 exp/max/softmax 阶段（loop 只有 fmul.s/fadd.s + 索引算术），是通用 mshadow 表达式基础设施的独立轴归约，非归一化完整算子；
- floating-point-matmul/GEMV row：排除 —— 无 matmul microkernel 形态（无 K/M/N 分块、无 FMA 连续流、无 packing/edge kernel）；evidence 是逐元素 div/rem 索引访存 + volatile 标量累加，属独立归约（该 row 明确互斥 "点积式加性累加（不含 matmul microkernel 结构的独立归约）→ widening-reduction row"）；
- elementwise rows：排除 —— 存在跨元素归约（fadd 累加到共享 accumulator）；
- precision-conversion / eliminate-unnecessary-precision-conversions：排除 —— 全 float 数据流，无 float64 往返或 narrowing 主导；
- kernel-selection row：排除 —— mxnet/mshadow 无该表达式的既有 RVV kernel/dispatch slot 证据。

(c) **双 Confidence 推导式**：
- route：compiler-generated provenance（annotate 源码行映射）+ source 确认 sum-of-products 轴归约语义（点积式加性累加，符合该 row 的 signal）+ hardware `v`/build `v` 下全 scalar + 互斥排除 → High；
- impact：函数内局部份额成立（interval ≈100%），但采样语义四条不全（`local period` 非 global-period）、workload 贡献未知（该表达式是通用基础设施，调用方场景未确认）、FP 顺序合同（volatile 严格顺序 vs vfredusum unordered）与调用上下文待验证、无 ARM 对照 → Low（只能讨论方向，不估算收益幅度）。

**多候选仲裁小段：** primary = RVV Widening Additive Reduction Kernels；supporting = No vectorization（同一 hot loop、同一向量化载体机制）。evidence-mechanism layer：L1（vectorization / semantic dispatch）。supporting 不计顶层 finding 数。顶层 finding 局部样本份额加总 ≈ **100.0%**（140 样本 local period，函数内；非 workload 级）。

**零命中路径不适用。** 其余 rows 无对应 signal（见 class trace 与 (b) 排除；codegen rows 如 IV-strength-reduction 因 div/rem 为语义索引解码、样本 0.00% 且非主导，不单列顶层 finding，作为 fix 内附注）。

## Phase 4 — Root-cause blueprint / 根因蓝图：void mshadow::MapPlan<ReduceWithAxisExp<sum, mul>>

**纳入蓝图的 pattern 与对应 row**：primary row「RVV Widening Additive Reduction Kernels」（`rows-operator-rvv.md`，已过 gate）；supporting row「No vectorization」。

1. **Root cause**：sum-of-products 按轴归约以全标量执行：① `red::sum::Reduce` 的 volatile accumulator 合同（`float volatile&`）强制每迭代把部分和写回栈槽再重载（`11bf544: fsw fa5,-84(s0)` 27.86% + `11bf514: flw fa4,-84(s0)`），是"过早回写"（premature writeback）形态；② 每元素执行 `div`/`rem` 索引解码（11bf510/11bf51a）与多次 mul/add 地址算术（11bf520-11bf536），k-loop 循环控制（`11bf518: addi` 50.71%）承担主要样本 —— 整条链 latency-bound；③ 全函数 zero `v*`，RVV 归约完全未参与。依据 `patterns/rvv_widening_reduction_kernels.md` §Why this is slow：逐元素归约的循环携带 accumulator 依赖与过早回写（"先把向量 lane 写回内存再求和会增加 load/store 与 cache 流量；RVV reduction 可以在寄存器内合并多个部分和"）。
2. **The fix / 修复方式**（与 `patterns/rvv_widening_reduction_kernels.md` §2/§3/§7/§9 一致；修复对象 = mshadow `ReduceWithAxisExp<sum, mul>` 的标量求值路径，普通代码/intrinsic 向量化）：

```cpp
// Before（当前标量形态，mshadow expr::ReduceWithAxisExp::Eval + red::sum::Reduce(volatile&)）
for (k = 0; k < size_; ++k) {
  z = (x*size_ + k)*trailing_ + y;        // 每元素 div/rem 解码
  res += src1[z/last_] * src2[z/last_ 的偏移...];   // volatile 栈槽往返：flw/fadd/fsw
}

// After（RVV 转换形态示意；fixed-VL main + runtime-VL tail）
// 1) 索引解码强度削减：z 每 k 递增 trailing_ → 预计算两路基址，内层用指针/stride 递增，
//    每 chunk 基址 + offset*trailing_，消除每元素 div/rem（trailing_==1 时为 unit-stride 连续）
// 2) 归约向量化（pattern §2/§3）：
float res = 0.0f; size_t k = 0;
vfloat32m2_t vacc0 = __riscv_vfmv_v_f_f32m2(0.0f, vlmax);   // 多独立 accumulator 缩短依赖链（§7）
vfloat32m2_t vacc1 = __riscv_vfmv_v_f_f32m2(0.0f, vlmax);
while (k < size_) {
  const size_t vl = __riscv_vsetvl_e32m2(size_ - k);
  vfloat32m2_t va = __riscv_vle32_v_f32m2(src1 + off1(k), vl);   // stride=trailing_（trailing_==1 连续）
  vfloat32m2_t vb = __riscv_vle32_v_f32m2(src2 + off2(k), vl);
  vacc0 = __riscv_vfmacc_vv_f32m2(vacc0, va, vb, vl);            // 寄存器内部分和，无过早回写
  k += vl;
}
vfloat32m2_t vsum = __riscv_vfadd_vv_f32m2(vacc0, vacc1, vl);
res = __riscv_vfmv_f_s_f32m2_f32(__riscv_vfredusum_vs_f32m2_f32m1(vsum, seed, vl));  // 或 vfredosum（见下）
// 3) saver：saveto::Save(out[y*stride+x], res)
```

前置/适用前提：
- FP 顺序合同：volatile 标量 reference 是严格顺序加法；`vfredusum`（unordered）改变结合顺序，须按项目容差验证，或使用 `vfredosum`（ordered）保持顺序语义（pattern §5/§9）；
- 每块独立归约 + 标量 seed 的安全结构（pattern §3：不要用最后一次循环的 vl 做最终归约；不要跨动态 vl 长期维护 tail-undisturbed 向量 accumulator）；
- 多独立 accumulator（vacc0/vacc1）改变 FP 顺序，同样需容差验证（kernel-conventions §1：多个 accumulator 会改变加法顺序）；
- `size_==0` 空输入 → out=0（现有代码 11bf562 路径）保持；`trailing_`/`last_` 的 stride 语义必须保持（saveto 输出索引不变）；
- widening 选择：本实例输入/输出均 float（无 widening 需求），但 pattern 的 widening 讨论对泛型 DType 仍适用（如 fp64 输入需 SEW=64 或 vfwredusum）；本轮按 float SEW=32。

Correctness contract 不可破坏项：`out[y*stride+x] = Σ_k src1[z/last_] * src2[...]` 的索引语义（z 序列、trailing_/last_ 布局）、`size_==0` 置零路径、saveto 直写语义、FP 加法顺序的 reference 容差、OpenMP 分片边界（y-loop）。

限制/风险：`trailing_>1` 的 strided 访存（vle32 stride 形态）可能降低收益 —— 常见 trailing_==1 场景为 unit-stride；FP unordered 归约的容差验证是前置；该表达式是 mshadow 通用基础设施，修改会影响所有消费该表达式的算子（需全量回归）；调用方（category-activation 内具体算子）未确认。

修复后预期 Profile signals：`11bf518: addi`（50.71%）、`11bf544: fsw fa5,-84(s0)`（27.86%）、`11bf532: add`（8.57%）、`11bf52e/11bf538: flw` 与每元素 div/rem（11bf510/11bf51a）份额消失；出现 `vsetvli`/`vle32`/`vfmacc`/`vfredusum`（或 `vfredosum`）/`vse32`；函数局部 cycles 改善（方向性）。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcv_..._zve64d_zvfh...`（RVV 1.0，C920v2/SG2044，OoO）；build ISA = `rv64i2p1_..._v1p0_..._zvl128b1p0`（含 `v`、`zvl128b`）；VLEN = 128 bits；bound type = 函数内 latency-bound（volatile 往返 + div/rem + 串行 FP 链）。
4. **收益上界**：当前 sampled event（cpu-clock）下的函数内局部样本份额 ≈ **100.0%**（140 样本 local period）；`baseline_gap: sampling metadata` → 禁止 workload 级 Amdahl 上界。
5. **三维路由判定**：`current source` = compiler-generated C++ 模板（mshadow 表达式求值器，OpenMP clone）；`implementation existence/reachability` = mshadow/mxnet 无该归约的既有 RVV kernel/dispatch slot → 不走 missing `.S` 分支；`function-level policy` = 无要求独立 `.S` 的官方 policy（通用表达式基础设施）→ 修复载体为普通代码/intrinsic 向量化。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。shape 要点：size_（归约长度）为运行期动态值 → `LMUL_min(W): N/A`，按 live set/unroll/吞吐选 LMUL（SEW=32, VLEN=128 → m1=4 lanes；双 accumulator + 2 输入 live ≈ 4-6 vectors → m1–m2 稳妥，m4 需 spill 检查）；tail 用 runtime-VL；`implementation_shape_gap: 无`。
7. **Related PRs 小节**：
   - `patterns/rvv_widening_reduction_kernels.md`：Related PRs：13 条 URL（OpenCV `#27096`、`33d632f85e4c`、`#26624`；oneDNN `#5361`、`a95f0060cfcb`；OpenJDK `72297d22d19e`、`08a2f841ec78`、`2c1e4c381615`、`#20910`、`134b63f0e8c4`、`#16629`、`1aebab780c5b`、`1b6281d98cf0`；OpenBLAS `c37509c213a3`、`3918d8504e77`；MNN `#4433`）。
   - `patterns/no-vectorization.md`（supporting）：Related PRs：16 条 URL（OpenCV `#22179`、`#22520`、`#23980`、`#24058`、`#24132`、`#24166`、`#24301`、`#24325`、`#27160`、`#27119`、`#27097`、`#27007`、`#26958`、`#26865`、`b902a8e792e1`、`2c16f3b7d2b2`、`e06502a254f7`、`a2d784b6f53a`、`83104bed3209`）。

## Phase 5 — Verification forecast / 验证预测：void mshadow::MapPlan<ReduceWithAxisExp<sum, mul>>

（primary = RVV Widening Additive Reduction Kernels；supporting 不单独验证）

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`11bf518: addi a2,a2,1`（50.71%）、`11bf544: fsw fa5,-84(s0)`（27.86%）、`11bf532: add a5,a5,s7`（8.57%）、`11bf52e: flw fa5,0(a4)`（3.57%）、`11bf53c: fmul.s`（4.29%）、`11bf540: fadd.s`（4.29%）与每元素 div/rem（11bf510/11bf51a）应消失或大幅缩小。
- **应出现侧**（锚定 `patterns/rvv_widening_reduction_kernels.md` §Verification）：annotate 出现 `vsetvli`、`vle32`、`vfmacc`、`vfredusum`/`vfredosum`、`vse32`；"标量 load、累加、乘加、指针更新和循环分支" 份额下降。
- **正确性/数值合同**：FP 顺序验证（vfredusum unordered vs volatile 严格顺序 reference 的逐位/容差，必要时改 vfredosum）；多 accumulator 顺序变化；`size_==0`/单元素/VLMAX 整倍数/全 tail 长度；strided（trailing_>1）与 unit-stride（trailing_==1）两形态；index 解码等价性（z 序列一致）。
- **收益顺序**：单一顶层 primary；幅度待补采。
- **补采升级**：至少需 (1) `precise_ip`/Exact-IP 确认；(2) `--percent-type=global-period` 重采量化 global 份额；(3) 确认该表达式的调用算子（category-activation 内归属）并对其做短/中/长归约长度 benchmark。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`MapPlan<ReduceWithAxisExp<sum, mul>> [._omp_fn.0]` |
| 2 | Phase 1 输出要求 | ✅ | 7 行 baseline；gap：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（primary=widening-reduction）+ supporting 1（no-vectorization）；锚点 `50.71 : 11bf518: addi`、`27.86 : 11bf544: fsw fa5,-84(s0)`、`0.00 : 11bf510: div`、`4.29 : 11bf540: fadd.s`；supporting 1；排除条数 ≥6；推导式 2 组 |
| 4 | Phase 4 输出要求 | ✅ | 已读 pattern：`patterns/rvv_widening_reduction_kernels.md`（命中 row=Widening Additive Reduction；引用 `Use widening partial sums`/`Reduce partial sums in registers`/`Minimize horizontal reductions`）、`patterns/no-vectorization.md`（supporting；`zero v*`）；The fix before/after/correctness/风险/Profile signals 齐备；`Related PRs：13 条 URL`（widening-reduction）+ `Related PRs：16 条 URL`（no-vectorization supporting） |
| 5 | 路径合规 | ✅ | 模式 A；primary=widening-reduction（L1）→ supporting=no-vectorization；leaf 均来自已过 gate 的 row；动态份额仅函数内局部（100.0%）并显式 `baseline_gap: sampling metadata` |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 `11bf518: addi`/`11bf544: fsw fa5,-84(s0)`/`11bf52e: flw`；出现侧 `rvv_widening_reduction_kernels.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；无追问；交付止于证据、根因蓝图、完整 The fix、验证预测 |

修正记录：无