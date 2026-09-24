Functions under analysis: [mshadow::MapPlan<mshadow::sv::saveto, mshadow::Tensor<mshadow::cpu, 4, float>, 4, float, mshadow::expr::UpSamplingNearestExp<mshadow::Tensor<mshadow::cpu, 4, float>, float, 4>>(mshadow::TRValue<...>*, mshadow::expr::Plan<...> const&) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（batch-003-function-001，rank 002，`cpu-clock (2446 samples, percent: local period)`，含完整 hot loop body）
- perf stat（可选 bound/context）：已提供（category-misc 全量 perf stat：IPC 0.658、L1_dcache_load_miss_rate 2.570%、LLC_load_miss_rate 38.853%、branch_miss_rate 1.461%、cache_misses≈cache_references）
- workload/binary/DSO/source context：已提供（libmxnet.so；源码 3rdparty/mshadow/mshadow/extension/spatial_upsampling_nearest.h + src/operator/nn/upsampling.cc、upsampling-inl.h；perf report 全局份额 18.27%，13386 samples）
- readelf -A（build ISA）：已提供（metadata binaries.libmxnet.so-elf-A：Tag_RISCV_arch 含 v1p0、zvl128b 等）
- hardware ISA（cpuinfo/hwprobe）：已提供（metadata cpuinfo isa：rv64imafdcv_... 含 v；T-Head C920v2 / SOPHGO SG2044，OoO）
- vlenb：已提供（metadata vector snapshot：vlenb=16 → VLEN=128 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=cpu-clock；annotate percent=local period；同一运行窗口；perf report 全局 13386 samples 中本函数 18.27%）
- Sampling IP precision：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`（metadata cpuinfo；含 `v` = RVV 1.0；C920v2/SG2044，out-of-order） |
| Build ISA | libmxnet.so Tag_RISCV_arch = `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`（metadata binaries.libmxnet.so-elf-A；含 v1p0 与 zvl128b，与 hardware v 一致） |
| Vector flavor | annotate 全程零 `v*` 零 `th.v*`（全 scalar）；无 flavor mismatch；标准 RVV 依赖 route 冻结可用 |
| VLEN | 128 bits（vlenb=16，metadata 冻结快照） |
| Bound type | IPC 0.658；L1_dcache_load_miss_rate 2.570%（低）、LLC_load_miss_rate 38.853%（高）；cache_misses≈cache_references（5,361,446,610 vs 5,361,444,278；报告出的 100% miss rate 属计数伪影，不作单点解释）；函数内局部样本 86% 落在 flw/fsw → store 主导的数据搬运形态，memory 分量强 |
| Sampling semantics | event=cpu-clock（可解释为时间）；annotate percent=local period；同一运行窗口；函数级全局份额已知（perf report 13386 samples 中本函数 18.27%）→ 函数级 Amdahl 表述可用 18.27%，函数内收益仍按局部份额表述 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（cpu-clock 未声明 precise_ip/Exact-IP，PMU skid 能力未知；单指令不承担 latency/cycle 归因，只锚定 basic block / loop interval） |

L0 baseline gate：hardware 有 `v`、build 有 `v` → 无 hardware/build mismatch；annotate 无 `th.v*` → 无 flavor gate 阻塞，标准 RVV route 冻结可用。Bound-type gate：memory 分量强（store 主导 + LLC miss 高）→ 本地 compute-vectorization fix 的 performance-impact confidence 下调，但路由明确的 RVV anti-pattern 仍照常报告。

## Phase 2 — Scope / 分析边界
- 函数清单：`mshadow::MapPlan<sv::saveto, Tensor<cpu,4,float>, 4, float, UpSamplingNearestExp<Tensor<cpu,4,float>, float, 4>>(TRValue<...>*, Plan<...> const&) [clone ._omp_fn.0]`（rank 002；annotate 2446 samples；perf report 全局 18.27%）。
- hot loop 边界：outer loop（输出行 y）`1a1733e–1a1737a`；inner loop（输出列 x）`1a1735e–1a17374`。
- 最高行 trace anchor：`66.43 :   1a17370: fsw fa5,-4(a3)`（saveto store，inner loop 尾部）；次高 `19.50 :   1a1736c: flw fa5,0(a5)`（src gather load）。
- Sampling IP precision 不足（Phase 1 `baseline_gap: sampling IP precision`）→ 单行只锚定 interval，不做 instruction-latency 归因。
- annotate 覆盖完整（hot loop body 全量呈现，非 annotate_incomplete）。

## Phase 3 — Pattern scan / 模式扫描：mshadow::MapPlan<sv::saveto, Tensor<cpu,4,float>, 4, float, UpSamplingNearestExp<...>> [clone ._omp_fn.0]

### Class selection trace
1. `rows-asm.md` — exclude：当前代码来源是 compiler-generated（GCC omp clone 标量循环，`[clone ._omp_fn.0]` 后缀；无 `.S` provenance/DWARF/object-mapping 证据）；无 dispatch-slot / 独立 `.S` policy 信号，policy/existence 四证不成立。
2. `rows-operator-rvv.md` — include（第一级必选）：compiler-generated scalar loop，hot loop 全 scalar 且 zero `v*`，hardware 与 build 均含 `v`；算子语义 = 最近邻空间上采样（resampling 合同）。
3. `rows-string-memory.md` — exclude：非 libc/string/mem 函数（无 memcpy/fill/scan/compare/sentinel/back-reference 合同）。
4. `rows-vectorized-tuning.md` — exclude：该组要求 hot loop 已有 `v*`；本函数 zero `v*`。
5. `rows-codegen.md` — include（补充）：compiler-generated 指令形态观察——inner loop 每元素 `div`（x/scale_）+ `slli`+`add` 重建 src 地址，dst 侧已用指针递增。
6. `rows-offload.md` — exclude：无矩阵引擎 / packed-SIMD / 权重重排信号；纯标量数据搬运 + 坐标映射。
7. `rows-crypto.md` — exclude：无密码学原语。
8. `rows-runtime-os.md` — exclude：用户态库代码，无 timer/CSR/特权域信号。

Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`

### Local performance pattern scan: `mshadow::MapPlan<...UpSamplingNearestExp...>`
| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Resampling Kernels（primary） | inner loop 逐元素坐标变换 + scalar gather+store：`66.43 : 1a17370: fsw fa5,-4(a3)`、`19.50 : 1a1736c: flw fa5,0(a5)`、`1a17360: div a5,a4,a1`（w=x/scale_，每元素一次）、`1a1733e: rem a5,a7,t3`（y=i%new_height_）、`1a17348: div a5,a5,a1`（h=y/scale_）；hot loop 零 `v*` | High | Medium | `patterns/rvv_resampling_kernels.md` |
| No vectorization（supporting） | 同一 main loop（`1a1735e–1a17374`）全 scalar、zero 标准 `v*` 与 `th.v*`；hardware `v` + build v1p0 | — | — | `patterns/no-vectorization.md` |
| Loop Induction Variable Strength Reduction（supporting） | inner loop 每轮 `div a5,a4,a1` + `slli a5,a5,0x2` + `add a5,a5,a6` 重建 src 地址；dst 侧已指针递增（`addi a3,a3,4` + `fsw fa5,-4(a3)`） | — | — | `patterns/loop_induction_variable_strength_reduction.md` |

#### 顶层 finding：RVV Resampling Kernels（primary）
**(a) 逐字 evidence 引用**（来源：rank 002 annotate，`percent: local period`，2446 samples）：
- `66.43 :   1a17370: fsw fa5,-4(a3)`（saveto store；inner loop `1a1735e–1a17374` 尾部）
- `19.50 :   1a1736c: flw fa5,0(a5)`（src gather load；inner loop）
- `1a17360: div a5,a4,a1`（w = x / scale_；inner loop 每元素一次）
- `1a1733e: rem a5,a7,t3`（y = i % new_height_；outer loop `1a1733e–1a1737a`）；`1a17348: div a5,a5,a1`（h = y / scale_）
- 所属 interval：outer loop `1a1733e–1a1737a` + inner loop `1a1735e–1a17374`。flw+fsw 局部份额合计 ≈ 85.9%；坐标/地址/控制行合计 ≈ 13.7%。

**(b) 互斥邻居排除**：
- RVV Indexed Gather for Table Lookup and Data-Dependent Access（rows-operator-rvv.md gather row）：行内互斥判据「带 interpolation 坐标/权重合同 → resampling row」。本函数是空间上采样算子（源码 spatial_upsampling_nearest.h `Plan::Eval`：`w=x/scale_`、`h=y/scale_`、`c*src_height_+h`），有明确运行期坐标合同，不是「无插值语义的任意 LUT/data-dependent gather」；resampling pattern §Shared conventions 明确「只有重采样认领运行期坐标/index/weight 驱动的 gather」→ gather row 不命中。
- RVV Spatial Convolution and Pooling：行内互斥「运行期坐标/index/weight 与 indexed gather → resampling row」；本函数无固定 stencil tap，坐标由运行期 scale 驱动 → 不命中。
- RVV Strided Memory Access for Layout Transforms：行内互斥「运行期坐标/权重的重采样 → resampling row」；本函数索引非固定 stride（由 x/scale_ 映射）→ 不命中。
- Elementwise / Complex / Activation / Normalization / Precision / Extrema / Arg-Extrema / Reduction / Matmul / Quantized / Triangular / Color / Layout / FFT / Separable-Transform 等其余 operator rows：语义不符（非 lane-independent 纯算术、无跨 lane 统计、无累加/MAC、无固定 W×W 变换）→ 整组排除。

**(c) 双 Confidence 推导式**：
- route：compiler-generated scalar provenance + 源码坐标合同（UpSamplingNearestExp::Plan::Eval）+ hardware `v` + build v1p0（hot loop zero `v*`）→ **High**
- impact：局部样本份额明确（flw+fsw ≈ 85.9%；perf report 全局 18.27%），VLEN=128 已知；但存在 store 写流量刚性（memory-bound 竞争 bottleneck）且 `baseline_gap: sampling IP precision` → **Medium**

#### Supporting evidence（挂在 primary 下）
- (a) No vectorization：同一 main loop 地址区间（`1a1735e–1a17374`）零 `v*`/`th.v*`，全 scalar flw/fsw。supporting because: 与 primary 同一 hot loop、同一机制（未向量化的标量数据搬运），修复对象相同。
- (a) Loop Induction Variable Strength Reduction：inner loop 每元素 `div a5,a4,a1`（x/scale_）与 `slli a5,a5,0x2; add a5,a5,a6`（src 地址每轮重建），dst 侧已指针化。supporting because: 标量路径的坐标/地址低效形态，向量化改写（primary fix）会自然消除；单独修复不足以改变数据搬运本质。

#### 多命中仲裁
primary = RVV Resampling Kernels（L1 vectorization / semantic dispatch 层，认领 root cause）；no-vectorization（L1）与 loop-IV-SR（L4）作为 supporting 并入同一机制。evidence 完全落在同一 hot interval（`1a1733e–1a1737a` + `1a1735e–1a17374`），机制不可分账，不做 independent/companion。收益上界：入口条件 A，仅对顶层 primary 加总 evidence sample share = 0.859（flw 0.1950 + fsw 0.6643，local period 局部份额）；supporting 不计入顶层 finding 数。

## Phase 4 — Root-cause blueprint / 根因蓝图：mshadow::MapPlan<sv::saveto, Tensor<cpu,4,float>, 4, float, UpSamplingNearestExp<...>> [clone ._omp_fn.0]
（纳入蓝图的 pattern：`rvv_resampling_kernels.md`（primary，对应 rows-operator-rvv.md「RVV Resampling Kernels」row，Phase 3 通过 gate）；`no-vectorization.md` 与 `loop_induction_variable_strength_reduction.md` 仅作 supporting 参考，不另立顶层 leaf。）

1. **Root cause**：
   - 依据 `patterns/rvv_resampling_kernels.md` §Why this is slow（"重采样的主要根因是逐输出重复坐标计算、数据相关地址导致 scalar gather、bilinear/cubic 权重与边界处理割裂成多遍"）与 §When to apply 正向 signal（"每个输出 lane 有运行期 source index/weight；annotate 显示 offset generation、`vluxei*` 候选或逐 lane scalar gather"）。
   - 机制：该函数把 NCHW 输入按运行期 scale_ 做最近邻上采样，每个输出元素从 `src[(c*src_height_+h)*src_stride_ + w]`（w=x/scale_、h=y/scale_）gather 后写回。GCC 生成全标量双循环：inner loop 每元素执行一次 `x/scale_` 整数除法（`1a17360: div a5,a4,a1`）、`slli+add` 地址重建（`1a1735a`、`1a1736a`）、一次 gather load（`1a1736c: flw`，19.50%）与一次 store（`1a17370: fsw`，66.43%）。fsw+flw ≈ 86% 的局部样本表明热点由标量 load/store 数据搬运构成。
   - 结论：在 RVV 1.0（VLEN=128，build 含 v1p0+zvl128b）上，该 resampling 算子循环完全没有向量化（zero `v*`）；运行期 scale_ 除法与逐元素地址重建把可预计算/可向量化的坐标与索引开销逐元素摊开，且每个源元素被相邻 scale_ 个输出重复 gather 加载。

2. **The fix / 修复方式**（依据 `rvv_resampling_kernels.md` §6 "Precompute resampling indices and weights"（"预计算只有在其成本能够被多个通道、行或调用复用时才有收益"）与 §7 "Vectorize bilinear resampling with indexed loads" 的 indexed-load 骨架（去掉插值权重/FMA，适配 nearest 退化形态），以及 `kernel-conventions.md` §3 fixed-VL main loop + runtime-VL tail、§2 LMUL 预算）：
   - 修复对象：mshadow UpSamplingNearest 表达式求值路径（`spatial_upsampling_nearest.h` 的 `Plan<UpSamplingNearestExp>::Eval` 对应的 MapPlan forward kernel），新增 RVV intrinsic 实现，保留标量 reference fallback。
   - 修复前（当前生成代码形态）：
     ```
     // Plan<UpSamplingNearestExp>::Eval + MapPlan 展开的标量双循环
     for (i in 行分块) {                                  // outer：输出行
       y = i % new_height_; c = i / new_height_; h = y / scale_;   // 每行一次 rem/div
       for (x = 0; x < ow; ++x) {                        // inner：输出列
         w = x / scale_;                                  // 每元素一次 div（1a17360）
         dst[i*dst_stride_ + x] = src[(c*src_height_+h)*src_stride_ + w];  // 一次 gather flw + 一次 fsw
       }
     }
     ```
   - 修复后（RVV intrinsic 形态，nearest 无插值权重，VLEN-agnostic）：
     ```
     // ① 预计算水平 byte-offset 数组（一次调用内跨所有输出行/通道复用；§6 复用条件成立：
     //    各行/通道共享同一水平坐标合同）：
     //    wByte[x] = (x / scale_) * sizeof(float)，x ∈ [0, ow)；仅在预计算成本可跨行/通道摊薄时启用
     for (i in 行分块) {
       c = i / new_height_; h = (i % new_height_) / scale_;   // 垂直复制由外层天然实现
       src_row = src + (c*src_height_+h)*src_stride_;
       dst_row = dst + i*dst_stride_;
       // ② fixed-VL main loop + runtime-VL tail（kernel-conventions §3）
       epr = __riscv_vsetvlmax_e32m2();                      // SEW=32，LMUL 候选 m1/m2（live：offset + data 两个 vector group，§2 预算内）
       for (x = 0; x + epr <= ow; x += epr) {
         off = __riscv_vle32_v_u32m2(wByte + x, epr);        // 预计算 byte offsets
         v   = __riscv_vluxei32_v_f32m2(src_row, off, epr);  // indexed gather
         __riscv_vse32_v_f32m2(dst_row + x, v, epr);         // unit-stride store
       }
       for (vl = __riscv_vsetvl_e32m2(ow - x); vl > 0;
            x += vl, vl = __riscv_vsetvl_e32m2(ow - x)) {    // runtime tail
         /* 同主体，vl 运行期 */
       }
     }
     ```
   - 适用前提：输出宽度 ow ≥ 1；scale_ 为运行期任意正整数；输出尺寸恰好为 scale_ 整数倍（`UpSamplingShape` 已 CHECK）；无 border/padding 合同。
   - 不可破坏的 correctness contract：floor(x/scale_)、floor(y/scale_) 坐标定义与 `Plan::Eval` 逐元素一致；index×sizeof(float) 不截断 byte offset（u32 index 需确认 ow 上限，大图须升 index width）；每输出元素恰好写一次（saveto / req=write 语义）；tail 覆盖 ow 全部剩余列；OpenMP 行分块下各行独立，wByte 只读共享安全；`__riscv_v_intrinsic` 宏 guard，无 RVV 时走标量 fallback。
   - 限制/风险：`vluxei32` 在 C920v2 上的吞吐未实测（gather 依赖 cache locality——nearest 相邻 scale_ 个输出共享同一源元素，命中同 cache line 概率高，locality 良好）；预计算 wByte 的额外流量在短宽/小 shape 调用下可能不摊薄（需与即时计算 A/B）；输出写带宽为刚性（scale=2 时输出约 4× 输入，200MB 级），向量化主要消除坐标/地址/控制指令与重复 gather load，不代表写带宽需求下降。
   - 修复后预期 Profile signals：hot interval 出现 `vsetvli`/`vle32`/`vluxei32`/`vse32`；scalar `flw`/`fsw` 与每元素 `div a5,a4,a1` 的局部份额显著下降；store 指令数按 vl 摊销（每元素 store 指令数降至 ~1/vl）。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcv_...`（含 `v`，RVV 1.0，C920v2/SG2044 OoO）；build ISA = libmxnet.so `v1p0 + zvl128b`；VLEN = 128 bits（vlenb=16）；bound type = 混合 bound，store/写带宽分量强（IPC 0.658、LLC_load_miss_rate 38.853%）。

4. **收益上界**：入口条件 A。primary finding 的 evidence sample share 加总 = **0.859**（flw 0.1950 + fsw 0.6643，local period 局部份额）。表述为「当前 cpu-clock sampled event 下的局部样本份额」；函数级全局贡献 18.27%（perf report 13386 samples）。store 写流量为刚性，实际可消除部分主要位于坐标/地址/控制指令（局部约 13.7%）与可摊薄的重复 gather load；因 `baseline_gap: sampling IP precision`，不做单指令 cycle 归因，收益上界取证据份额、不取可实现收益。

5. **三维路由判定**：
   - current source：compiler-generated（GCC omp clone 标量循环；mshadow 表达式模板 + `MapPlan` 的 `#pragma omp parallel for`）。来源证据：annotate 无 `.S` provenance、`[clone ._omp_fn.0]` 后缀、libmxnet.so DSO 地址映射。
   - implementation existence / reachability：无现存 RVV kernel、无 dispatch slot（mshadow MapPlan 直接生成标量循环，无 kernel 选择层）→ 不存在「kernel 未选中」问题，`kernel-selection` row 不适用。
   - function-level policy：mshadow / mxnet 该 operator 无独立 `.S` 政策（policy/existence 四证不成立）→ 不进入 missing-assembly 分支；函数级贡献 = 单次 forward 调用。

6. **Implementation-shape proof**：不适用（未进入 policy-backed missing `.S` 分支）。shape 要点已在 §The fix 注明：动态 N（ow 运行期确定）→ 不写固定 vl，按 kernel-conventions §3 fixed-VL main + runtime tail；LMUL 候选 m1/m2（live = byte-offset vector + gathered data vector 两个 register group，`LMUL × peak_live_vectors ≤ 32`），最终以目标反汇编的 live interval 与 A/B benchmark 为准，不硬编码。

7. **Related PRs 小节**（按 pattern 分组；同一 pattern 内按 URL 去重）：
   - `rvv_resampling_kernels.md`：Related PRs：2 条 URL
     - https://github.com/alibaba/MNN/pull/4053
     - https://github.com/alibaba/MNN/commit/f4fcff3436a9d95c5367699b1c53d4eb1cbc3b7d
   - `no-vectorization.md`（supporting）：Related PRs：15 条 URL
     - https://github.com/opencv/opencv/pull/22179
     - https://github.com/opencv/opencv/pull/22520
     - https://github.com/opencv/opencv/pull/23980
     - https://github.com/opencv/opencv/pull/24058
     - https://github.com/opencv/opencv/pull/24132
     - https://github.com/opencv/opencv/pull/24166
     - https://github.com/opencv/opencv/pull/24301
     - https://github.com/opencv/opencv/pull/24325
     - https://github.com/opencv/opencv/pull/27160
     - https://github.com/opencv/opencv/pull/27119
     - https://github.com/opencv/opencv/pull/27097
     - https://github.com/opencv/opencv/pull/27007
     - https://github.com/opencv/opencv/pull/26958
     - https://github.com/opencv/opencv/pull/26865
     - https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d
     - https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a
     - https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706
     - https://github.com/opencv/opencv/commit/a2d784b6f53a
     - https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a
   - `loop_induction_variable_strength_reduction.md`（supporting）：Related PRs：3 条 URL
     - https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d
     - https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df
     - https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec

## Phase 5 — Verification forecast / 验证预测：mshadow::MapPlan<sv::saveto, Tensor<cpu,4,float>, 4, float, UpSamplingNearestExp<...>> [clone ._omp_fn.0]
- 应消失/缩小侧（锚定 Phase 3(a) 逐字引用行）：
  - `1a17370: fsw fa5,-4(a3)`（66.43%）与 `1a1736c: flw fa5,0(a5)`（19.50%）的局部份额显著下降（被 `vse32` / `vluxei32` 替代）；
  - `1a17360: div a5,a4,a1`（每元素 x/scale_）与 `1a1733e: rem a5,a7,t3`、`1a17348: div a5,a5,a1` 消失或移出 hot interval；
  - `1a1735a: slli a3,a3,0x2`、`1a1735e: addi a3,a3,4` 等逐元素地址算术的局部份额下降。
- 应出现侧（pattern §Verification）：
  - `rvv_resampling_kernels.md` §Verification：hot interval 出现预期 indexed vector loads（`vluxei*`）与 `vse32`；坐标/floor 规则逐元素一致（覆盖 floor 坐标、尾列、奇数宽度、大图 byte-offset 不截断、plane/channel 合同）；cycles/output 改善；若 indexed load 在目标核更慢、index vector 导致 spill、预计算流量超过节省或小尺寸退化则判定失败。
  - `no-vectorization.md` §Verification：同一函数 annotate 出现 `vsetvli`/`vle*`/`vse*`；scalar 指令不再主导 hot loop；与 scalar reference 对比覆盖 empty、小长度、vl 整倍数与全部 tail 长度。
  - `loop_induction_variable_strength_reduction.md` §Verification（supporting，跟随 primary）：循环体 `slli`/`add`（index scaling）与 limit reload 减少；遍历结果逐元素等价、无 off-by-one。
- 收益顺序：入口条件 A，仅一个顶层 primary（0.859 局部证据份额），无排序问题。
- 整体回归：重跑 category-misc benchmark，观察 `opperf_UpSampling_input_*_avg_time_forward_UpSampling_us`（当前 35.46 / 19.85 / 未提供第三个）与函数全局份额 18.27% 的下降。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅（1/1 组；函数名 `mshadow::MapPlan<...UpSamplingNearestExp...> [clone ._omp_fn.0]`） |
| 2 | Phase 1 输出要求：7 行 baseline 表 + 两个 L0 gate + bound-type gate + Sampling IP precision 行 | ✅（7 行全含；gap 标签 `baseline_gap: sampling IP precision`；无 hardware/build mismatch；无 th.v* 停扫） |
| 3 | Phase 3 输出要求：8 项 Class selection trace + Classes scanned + 三件套 + 仲裁 | ✅（Classes scanned: `rows-operator-rvv.md`、`rows-codegen.md`；顶层 finding 1（RVV Resampling Kernels，route High / impact Medium）；supporting 2（no-vectorization、loop-IV-SR）；排除：gather/spatial-conv/strided/其余 operator rows 整组 + codegen rows 整组；evidence 锚点 `1a17370: fsw fa5,-4(a3)`、`1a1736c: flw fa5,0(a5)`、`1a17360: div a5,a4,a1`；推导式：route High / impact Medium） |
| 4 | Phase 4 输出要求：逐字段七项 + pattern 独有内容引用 + Related PRs | ✅（已读 pattern：`rvv_resampling_kernels.md` §Why this is slow / §6 / §7 / §Verification、`no-vectorization.md` §The fix / §Verification、`loop_induction_variable_strength_reduction.md` §The fix / §Verification；before/after 伪代码、correctness contract、限制/风险、Profile signals 锚点齐全；Related PRs：resampling 2 条、no-vectorization 15 条、loop-IV-SR 3 条 URL） |
| 5 | 路径合规：模式 A profile-backed；L1 primary + L4 supporting；evidence 单一 hot interval 分账；无 th.v* 全局停扫 | ✅（模式 A；路径 = rows-operator-rvv.md（primary）→ rvv_resampling_kernels.md；rows-codegen.md（supporting）→ loop_induction_variable_strength_reduction.md；blueprint leaf 均来自通过 gate 的 row） |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3 引用行、出现侧标注 pattern §Verification | ✅（消失侧：`1a17370: fsw fa5,-4(a3)`、`1a1736c: flw fa5,0(a5)`、`1a17360: div a5,a4,a1`、`1a1733e: rem a5,a7,t3`；出现侧：rvv_resampling_kernels.md §Verification、no-vectorization.md §Verification、loop_induction_variable_strength_reduction.md §Verification） |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成、无契约外分支 | ✅（交付止于 Profile 证据、根因蓝图、完整 The fix 与验证预测） |

修正记录：无