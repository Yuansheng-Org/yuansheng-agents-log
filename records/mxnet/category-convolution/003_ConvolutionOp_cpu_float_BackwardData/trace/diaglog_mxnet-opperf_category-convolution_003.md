Functions under analysis: [mxnet::op::ConvolutionOp<mshadow::cpu, float>::_BackwardData]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`003-mxnet：：op：：ConvolutionOp＜mshadow：：cpu, float＞：：_BackwardData(...)-f7b4d0feeed9-annotate.txt`，libmxnet.so，1411 samples，`cpu-clock`，`percent: local period`；hot loop body 完整覆盖）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-convolution.txt`，含 duration/cycles/instructions/branches/L1/LLC 计数）
- workload/binary/DSO/source context：已提供（mxnet-opperf category-convolution 基准；binary 为 `libmxnet.so`；源码 `src/operator/nn/im2col.h` `col2im_cpu` 与 `src/operator/nn/convolution-inl.h` `_BackwardData` 存在）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries.libmxnet.so-elf-A`，含 `v1p0`、`zve32f1p0`、`zve64d1p0`、`zvl128b1p0` 等）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（metadata `cpuinfo.isa`：`rv64imafdcv_zicbom_zicboz_..._zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_...`）
- `vlenb`：已提供（metadata `vector.vlen_bits=128`、`vlenb=16`）
- 采样元数据（event / percent type / scope / 窗口）：已提供（annotate header 原文 `cpu-clock (1411 samples, percent: local period)`，单次运行；函数 workload 级贡献未知）
- Sampling IP precision（`precise_ip` / Exact-IP / PMU skid）：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcv_..._zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_...`（RVV 1.0 `v` 暴露，含 `zve64d`、`zvfh`；SG2044/XuanTie C920v2） |
| Build ISA | 已提供：`libmxnet.so` `Tag_RISCV_arch` 含 `v1p0_zicsr2p0_zifencei2p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0`（build 已带 V 与 `zvl128b`） |
| Vector flavor | annotate cold 段出现 RVV 1.0 `v*` mnemonic（`vsetivli zero,2,e64,m1,ta,ma`、`vle64.v v1,(s3)`、`vid.v`、`vse8.v` 等，均 0.00%，位于 TShape 初始化与 libstdc++ `__to_chars_10_impl` 路径）→ RVV 1.0，无 `th.v*` |
| VLEN | 已提供：`vlenb=16` → VLEN=128 bits（SEW=32 时 m1=4 lanes） |
| Bound type | IPC=0.639；L1_dcache_load_miss_rate=0.628%（loads 基本命中 L1）；LLC_load_miss_rate=14.894%；branch_miss_rate=1.070% → 计算/issue 受限 + RMW（load-store）hazard 主导，非 memory-bandwidth 主导。注：derived `cache_miss_rate=100%` 是 `cache_misses/cache_references` 计数器别名伪影（misses 13.812e9 ≈ references 13.812e9），不作证据 |
| Sampling semantics | event=`cpu-clock`（时间可解释）；percent type=`local period`（annotate header 原文）；同一运行窗口；函数 workload 级贡献未知 → 百分比只表示本函数内局部样本份额，不得称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata` 的部分条件成立但 global-period 与 workload 贡献两条不成立） |
| Sampling IP precision | `precise_ip`/Exact-IP/skid 能力未知 → `baseline_gap: sampling IP precision`；单行高占比只锚定 basic block / loop interval，不做单指令 latency 归因 |

L0 baseline gate 1（hardware 有 `v` vs build 无 `v`）：hardware 与 build 均含 `v`（且均为 RVV 1.0）→ 无 mismatch，不冻结 vector route。
L0 baseline gate 2（`th.v*` flavor gate）：annotate 为 RVV 1.0 `v*`，非 `th.v*` → 无 `vector_flavor_mismatch`。
Bound-type gate：`baseline_gap: sampling IP precision`，但 bound 分类可判定（compute/issue-bound，L1 miss 0.628%）→ 不因 memory-bound 降级 RVV fix 的第一杠杆地位。

## Phase 2 — Scope / 分析边界

- 函数清单与承诺声明一致：1 个函数（`_BackwardData`）。
- Hot loop 边界：内联 `col2im_cpu<float>`（`src/operator/nn/im2col.h:312-333`）的 output_col 内层循环，地址区间 `12b73a4–12b73d0`；外层 output_rows/kernel_col/kernel_row/channel 控制位于 `12b7302–12b73da`（合计约 1.3%）；函数其余（setup/CHECK/string 构造/销毁）均为 0.00%。
- 最高占比行原文（interval 锚点）：`37.70 :   12b73c6:        fsw     fa5,0(t2)`。
- annotate 覆盖完整（含整个 hot loop body）；`baseline_gap: sampling IP precision` → 根因按 loop-interval 机制归因，不按单指令 latency 归因。
- 采样窗口一致；函数级 workload 贡献未知 → 收益上界只表达为本函数局部样本份额。

## Phase 3 — Pattern scan / 模式扫描：mxnet::op::ConvolutionOp<mshadow::cpu, float>::_BackwardData

### Class selection trace（8 项）

| # | Class 文件 | 判定 | 触发观察 |
|---|---|---|---|
| 1 | `rows-operator-rvv.md` | include（第一级必选） | 当前代码来源为 compiler-generated scalar loop（GCC 内联 `col2im_cpu` 模板），热点是未向量化的空间卷积 scatter 语义循环 |
| 2 | `rows-codegen.md` | include | 独立信号：内层循环每轮 `addw t2,a3,a7; slli t2,t2,0x2; add t2,t2,s6` 重建 data_im 地址（4.89% 样本）→ 归纳变量强度削减 / 控制流形态候选；且存在现有 kernel/dispatch 判别需求（GEMM 已走 OpenBLAS） |
| 3 | `rows-vectorized-tuning.md` | exclude | 该类要求完整 annotate 的 hot main loop 已有 `v*`；本函数 hot interval 全 scalar，`v*` 仅出现在 cold 初始化/string 路径（0.00%） |
| 4 | `rows-string-memory.md` | exclude | 热点不是 copy/fill/sentinel-scan/compare/checksum 语义，而是浮点 scatter-accumulate |
| 5 | `rows-asm.md` | exclude | 当前代码不是手写 `.S`（无 .S provenance）；无 col2im 的 dispatch slot 与 assembly-default policy → policy-backed missing `.S` 四证不成立 |
| 6 | `rows-offload.md` | exclude | 目标核（C920v2）无矩阵引擎/AME/P-packed-SIMD；GEMM 已经 OpenBLAS `cblas_sgemm` 执行，且在本函数 annotate 内无样本 |
| 7 | `rows-crypto.md` | exclude | 非密码学原语 |
| 8 | `rows-runtime-os.md` | exclude | 非 kernel/RTOS/timer/CSR 热点 |

`Classes scanned: rows-operator-rvv.md, rows-codegen.md`

### Local performance pattern scan: `mxnet::op::ConvolutionOp<mshadow::cpu, float>::_BackwardData`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Spatial Convolution and Pooling Kernels（primary） | col2im 内层循环：固定邻域 gather-scatter、每元素 border 分支、overlap-sensitive RMW store、hot interval 内 zero `v*` | High | Medium | `patterns/rvv_spatial_convolution_and_pooling_kernels.md` |
| No vectorization（supporting） | 同一 hot main loop 全 scalar（`flw`/`fadd.s`/`fsw`）、interval 内 zero `v*`，hardware 与 build 均含 `v` | — | — | `patterns/no-vectorization.md` |
| Loop Induction Variable Strength Reduction（supporting） | 同一循环每轮以 `addw`+`slli`+`add` 重建 `data_im` 地址（12b73b2/12b73b6/12b73b8，4.89%），而访问为固定 stride 序列、可用指针递增表达 | — | — | `patterns/loop_induction_variable_strength_reduction.md` |

#### Finding 1（primary）：RVV Spatial Convolution and Pooling Kernels

**(a) 逐字 evidence 引用**（均为内层 col2im 循环 interval `12b73a4–12b73d0`，对应源码 `im2col.h:321-327` 的 `for (output_col...) { if (is_a_ge_zero_and_a_lt_b(input_col, width)) { data_im[input_row * width + input_col] += *data_col; } data_col++; input_col += stride_w; }`）：

```
16.80 :   12b73ae:        bgeu    a3,s2,12b73ca     ; input_col ∈ [0, width) 边界分支
 4.89 :   12b73b2..b8:    addw t2,a3,a7 / slli t2,t2,0x2 / add t2,t2,s6   ; data_im 地址重建
13.96 :   12b73ba:        flw     fa5,0(t2)          ; load data_im（RMW 读）
12.12 :   12b73be:        flw     fa4,0(a5)          ; load data_col（连续读）
 7.02 :   12b73c2:        fadd.s  fa5,fa5,fa4
37.70 :   12b73c6:        fsw     fa5,0(t2)          ; store data_im（overlap-sensitive RMW 写）
 2.98 :   12b73d0:        bne     a5,s5,12b73ae     ; data_col 指针比较
```

hot interval 合计 `16.80+1.84+0.07+2.98+13.96+12.12+7.02+37.70+1.63+1.20+2.98 = 98.30%`（1411 样本中的约 1387 样本）。机制为 interval 级（Sampling IP precision 不足，不做单指令 latency 归因）。

**(b) 互斥邻居排除**：
- Strided Layout Transform row（`rvv_strided_layout_transform_kernels.md`）：本循环是带邻域重叠累加与 interior/border 分支的 scatter-RMW，不是纯固定 stride 搬运；该 row 行内互斥判据明确写"卷积固定邻域 → spatial-convolution row"。
- Indexed Gather row（`rvv_gather_indexed_memory_access.md`）：`input_col = -pad_w + kernel_col*dilation_w + output_col*stride_w` 与 `input_row` 完全由循环索引静态确定，无数据相关/运行期索引 → 不归 gather row。
- No vectorization row：更具体 semantic row（spatial）认领同一 hot loop；no-vectorization 只作 supporting。
- Floating-Point Matmul / Quantized Matmul rows：GEMM 经 `check_gemm`→`cblas_sgemm`（OpenBLAS）执行，其样本落在 `cblas_sgemm` 符号内，本函数 annotate 中 `12b6d92/12b6dd0` 调用点均为 0.00% → hot interval 不归 matmul rows。
- Matrix-engine im2col lowering（`matrix_engine_im2col_convolution_lowering.md`）：目标核无矩阵引擎 → 不适用。
- Cache-Aware Blocking row（`cache_aware_blocking_for_tiled_kernels.md`）：L1_dcache_load_miss_rate 0.628%、无 tile-residency/复用距离/尺寸拐点证据 → 排除（零命中 bound-type 路径）。
- Control-Flow Layout row：`bgeu` 为单条件分支、branch_miss_rate 仅 1.070%，非 branch-diamond/RAS 污染主导 → 排除。

**(c) 双 Confidence 推导式**：`route: 源码映射（im2col.h 内联）+ interval 全 scalar + 每元素边界分支 + overlap 写 + zero v* + hardware/build 均含 v → High`；`impact: hot interval 局部样本份额 98.3%（cpu-clock，local period）+ VLEN=128 + bound=compute/issue-bound → 局部收益方向明确；但 percent-type 非 global-period、函数 workload 贡献未知、sampling IP precision 缺失 → Medium（不得称 workload 级上界）`。

#### Supporting evidence（计入 primary，不另起顶层）

- **No vectorization（supporting because: 同一机制）**：(a) `12b73ba: flw fa5,0(t2)` / `12b73c2: fadd.s fa5,fa5,fa4` / `12b73c6: fsw fa5,0(t2)` 为主，interval 内 zero 标准 `v*`；hardware `isa` 行含 `v`（`zve32f_zve64d`），build `Tag_RISCV_arch` 含 `v1p0_zvl128b1p0`；cold 段已有 RVV 1.0 `v*`（`12b686c vsetivli zero,2,e64,m1,ta,ma` 等）证明工具链能发向量指令。它只解释"向量单元未处理 hot main-loop 并行元素"这一载体事实。
- **Loop Induction Variable Strength Reduction（supporting because: 同一机制）**：(a) 每轮 `12b73b2: addw t2,a3,a7` / `12b73b6: slli t2,t2,0x2` / `12b73b8: add t2,t2,s6`（合计 4.89%）重建 data_im 地址，而 data_col 侧已用 `12b73ca: addi a5,a5,4` + 预计算 end（`s5`）做强度削减；data_im 侧在 stride_w=1 时为连续序列，本可用指针递增表达。它只解释同一循环内的地址重建冗余，随 primary 的循环重构一并消失。

#### 多命中仲裁小段

顶层 finding 计 1（primary），supporting 计 2。primary = Spatial Convolution row（L1，vectorization/semantic dispatch 层）。因果消除测试：把 col2im 内层循环重构为"预计算段边界 + interior 向量化"后，`bgeu` 边界分支与 `addw/slli/add` 地址重建都随之消失 → 上层 spatial 改写认领 root cause，no-vectorization（L1 同层载体事实）与 IV-SR（L4 compute micro-structure）均为 supporting，不单列修复对象。入口条件 A：仅 primary 计收益上界（98.3% 局部份额），supporting 不排序。独立 finding：无（函数内其余区间样本 ≈ 0）。

## Phase 4 — Root-cause blueprint / 根因蓝图：mxnet::op::ConvolutionOp<mshadow::cpu, float>::_BackwardData

**1. Root cause**：`_BackwardData` 非 1×1 分支先对每个 (n,g) 调用 OpenBLAS `cblas_sgemm`（`weight_3d[g]×out_grad_3d[g]→col_buffer_3d[g]`，样本落在 libopenblas 内、本函数 annotate 为 0%），随后执行 `col2im` scatter：内层 `output_col` 循环把 col_buffer 的连续元素累加回 `data_im[input_row*width+input_col]`。热点机制（依据 `patterns/rvv_spatial_convolution_and_pooling_kernels.md` §Why this is slow）：固定 stencil 反卷积/col2im 把 "interior/border 混在主循环"，每个元素重复 "interior/border 分支"（`12b73ae` 16.80%），并做 "overlap-sensitive store"（同一 `data_im` 元素被 kernel_h×kernel_w 个 tap 位置重复 RMW，`12b73c6` 37.70%）；hot interval 全 scalar（zero `v*`），每元素约 10 条指令（2 load + 1 fadd + 1 store + 1 边界分支 + 3 地址整数指令 + 2 循环控制），OoO 核上 RMW store→load 同地址依赖限制重叠。IPC 0.639、L1 miss 0.628% 支持"计算/issue 受限 + RMW hazard"而非内存带宽受限。

**2. The fix / 修复方式**（依据 pattern §2 "Vectorize the interior of depthwise convolution across output positions" 与 §5 "Separate interior and border paths"；载体为 compiler/intrinsic 代码，非 `.S`）：

- **段拆分**：对每个固定 (kernel_row, kernel_col)，`input_col` 为 `output_col` 的线性函数（`stride_w` 斜率）。先计算该段有效区间 `[lo, hi]`：`input_col>=0` 与 `input_col<width` 分别对应 `output_col` 下界/上界，把 `output_col` 循环拆为 left-boundary / interior / right-boundary 三段。interior 段内 `bgeu` 边界分支整体消除（对应 pattern §5 "对 interior 使用无边界判断的 RVV fast path"），left/right 边界保留标量 fallback 或带合法地址保证的 mask。
- **interior 向量化（RVV）**：data_col 连续 → `vle32.v`；data_im 侧在 `stride_w==1 && dilation_w==1` 时也为连续行段 → `vle32.v` + `vfadd.vv` + `vse32.v`（RMW 保持）；`stride_w>1` 时用固定 stride 直读 `vlse32.v`（依据 §1 "用直接 `vlse*` 替换"），禁止用 `vcompress`/mask/temporary-buffer 往返。LMUL 选型按 `kernel-conventions.md` §2 live-register budget（`LMUL*peak_live<=32`；e32m4 时 2 load + 1 acc 约 3 个 live group，m8 需核对 spill），§3 fixed-VL main loop + runtime-VL tail（VLEN=128：e32m4=16 lanes/vector 为候选，不硬编码 lane count）。
- **correctness contract**：每元素 `data_im[idx] += col[k]` 在向量 lane 间相互独立，per-element FP 加法语义逐元素不变（无跨 tap 重结合 → 无 FP 顺序改变）；padding 偏移不得因 interior fast path 省略（pattern §2 "padding 偏移不能因为采用 interior fast path 而省略"）；border 不得用越界指针构造 + mask 掩盖（pattern §2 "padding 区域不能通过构造越界指针后再依赖 mask 避免访问"），用独立 border scalar path 或预填充输入；`kNullOp`/`kAddTo` 的 `std::fill` 清零语义保持。
- **限制/风险**：小 shape 下向量化 overhead（`vsetvl`、段头尾标量）可能超过收益，需为小 `output_w` 保留 scalar 路径；RMW overlap 仍会命中同一 cache line（L1 内 store→load forward），但指令数按 4–16× 下降是主要收益；channel tail 与 group 循环保持外层不变。
- **预期 Profile signals**：interior 段 `bgeu`（12b73ae）与 `addw/slli/add`（12b73b2–b8）份额显著下降，`flw/fadd.s/fsw`（12b73ba/c2/c6）替换为 `vsetvli/vle32/vse32/vfadd.vv`（stride>1 时为 `vlse32`）；scalar 份额仅保留在 boundary/tail 段。

**3. Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zve64d_zvfh...`（V、RVV 1.0）；build ISA=`libmxnet.so` `rv64i2p1_..._v1p0_..._zve32f1p0_zve64d1p0_..._zvl128b1p0`；VLEN=128 bits（`vlenb=16`）；bound type=compute/issue-bound（IPC 0.639、L1 miss 0.628%、branch miss 1.070%）。

**4. 收益上界**：当前 sampled event（`cpu-clock`，`local period`）下本函数内局部样本份额 **98.3%**（hot interval 1387/1411）。percent type 非 global-period、函数 workload 级贡献未知 → 不称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata` 部分成立）。

**5. 三维路由判定**：
- current source：compiler-generated C++ 模板代码（`im2col.h` `col2im_cpu` 被 GCC 内联进 `convolution-inl.h` `_BackwardData`；annotate 源码行交错 + 无 `.S` provenance）。
- implementation existence/reachability：MXNet CPU 路径无 RVV col2im kernel、无 dispatch slot/注册表，仅 scalar 模板路径可达；GEMM 部分已由 OpenBLAS 承接（样本不在本函数）。
- function-level policy：MXNet 对 col2im 无独立 `.S`/assembly-default policy，policy-backed missing `.S` 的 dispatch-slot/scalar-fallback/kernel-absence/policy 四证不成立（缺 slot 与 policy 两证）→ 修复载体为 compiler/intrinsic 代码，不入 missing `.S` 分支，不输出 Implementation-shape proof。

**6. Related PRs 小节**（`patterns/rvv_spatial_convolution_and_pooling_kernels.md`）：
Related PRs：17 条 URL — oneDNN [#5506](https://github.com/uxlfoundation/oneDNN/pull/5506)、[8208c6a731ed](https://github.com/uxlfoundation/oneDNN/commit/8208c6a731edff86af37b1ede3208775cd46aaef)、[4f915a57ce2f](https://github.com/uxlfoundation/oneDNN/commit/4f915a57ce2fd600b9ca04bf5a312c365a4b1830)、[563c5fcf8a](https://github.com/uxlfoundation/oneDNN/commit/563c5fcfcf8ad591b43d3dc7e0bc9292102eeba0)、[56a75b311f2e](https://github.com/uxlfoundation/oneDNN/commit/56a75b311f2e9fdb176b3ca30a007c7fcdc641de)、[a4c4855a9f4c](https://github.com/uxlfoundation/oneDNN/commit/a4c4855a9f4c478177e7fc799f64ac1457553ad7)、[#4323](https://github.com/uxlfoundation/oneDNN/pull/4323)、[ce942914cb9d](https://github.com/uxlfoundation/oneDNN/commit/ce942914cb9d51c84ee0fc7f3ae253d6cd0121f6)、[dd1dff668fe9](https://github.com/uxlfoundation/oneDNN/commit/dd1dff668fe9194d9a955333461e299d19aefb1c)、[#4735](https://github.com/uxlfoundation/oneDNN/pull/4735)、[#5345](https://github.com/uxlfoundation/oneDNN/pull/5345)；MNN [#4042](https://github.com/alibaba/MNN/pull/4042)、[672c586](https://github.com/alibaba/MNN/commit/672c5862392393c171f1513bf7994d3b95e2a6a1)、[#4359](https://github.com/alibaba/MNN/pull/4359)；OpenCV [5be158a2b6ed](https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5)。

supporting pattern 的 Related PRs（仅随 primary 引用，不单列修复）：`patterns/no-vectorization.md` Related PRs：19 条 URL — OpenCV [#22179](https://github.com/opencv/opencv/pull/22179)、[#22520](https://github.com/opencv/opencv/pull/22520)、[#23980](https://github.com/opencv/opencv/pull/23980)、[#24058](https://github.com/opencv/opencv/pull/24058)、[#24132](https://github.com/opencv/opencv/pull/24132)、[#24166](https://github.com/opencv/opencv/pull/24166)、[#24301](https://github.com/opencv/opencv/pull/24301)、[#24325](https://github.com/opencv/opencv/pull/24325)、[#27160](https://github.com/opencv/opencv/pull/27160)、[#27119](https://github.com/opencv/opencv/pull/27119)、[#27097](https://github.com/opencv/opencv/pull/27097)、[#27007](https://github.com/opencv/opencv/pull/27007)、[#26958](https://github.com/opencv/opencv/pull/26958)、[#26865](https://github.com/opencv/opencv/pull/26865)、[b902a8e792e1](https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d)、[2c16f3b7d2b2](https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a)、[e06502a254f7](https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706)、[a2d784b6f53a](https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f)、[83104bed3209](https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a)。`patterns/loop_induction_variable_strength_reduction.md` Related PRs：3 条 URL — Linux [18be4ca5cb4e](https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d)、OpenBLAS [477dd40f073c](https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df)、OpenBLAS [d832ee50868a](https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec)。

## Phase 5 — Verification forecast / 验证预测：mxnet::op::ConvolutionOp<mshadow::cpu, float>::_BackwardData

按收益上界顺序（仅 primary 有上界；supporting 跟随 primary 验证）：

**primary（Spatial Convolution/col2im RVV 重构）**：
- 应消失/缩小：interior 段 `12b73ae: bgeu a3,s2,12b73ca`（16.80%）、`12b73ba: flw fa5,0(t2)`、`12b73be: flw fa4,0(a5)`、`12b73c2: fadd.s fa5,fa5,fa4`、`12b73c6: fsw fa5,0(t2)`、`12b73b2/12b73b6/12b73b8: addw/slli/add` 的 sample share 显著下降；scalar 指令只留在边界/tail 段。
- 应出现（依据 `patterns/rvv_spatial_convolution_and_pooling_kernels.md` §Verification）：interior 反汇编出现 `vsetvli`/`vle32.v`/`vfadd.vv`/`vse32.v`（`stride_w>1` 时 `vlse32.v`），fixed-VL main loop 内无每轮 `vsetvli`（`kernel-conventions.md` §3）；重复 tap 的 scalar load、border 分支与 scalar 邻域归约份额下降；cycles/output 改善；stride=1/dilation=1 主路径正确性对照覆盖 kernel/pad/stride/dilation、NCHW channel tail、interior 与四边 border，FP per-element 结果与 reference 逐元素一致。
- 失败判据：border/overlap 结果错误、LMUL 过高导致 spill、cache traffic 反而增加、小 shape 退化、或实现未进入预期路径（`kernel-conventions.md` §Verification checklist）。
- 另需说明：`baseline_gap: sampling IP precision` → 复采时先确认 `precise_ip`/Exact-IP，再评估单指令级归因。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ 1/1 组；`mxnet::op::ConvolutionOp<mshadow::cpu, float>::_BackwardData` |
| 2 | Phase 1 输出要求满足 | ✅ 7 行 baseline（Hardware ISA / Build ISA / Vector flavor / VLEN / Bound type / Sampling semantics / Sampling IP precision）＋ 2 个 L0 gate ＋ bound-type gate；gap 标签：`baseline_gap: sampling IP precision`；`Sampling IP precision` 行已输出 |
| 3 | Phase 3 输出要求满足 | ✅ 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding=1（primary，evidence 锚点 `37.70 : 12b73c6: fsw fa5,0(t2)`、`16.80 : 12b73ae: bgeu a3,s2,12b73ca`、`98.30%` interval 合计）；supporting=2（No vectorization、Loop IV Strength Reduction）；排除条数≥6（strided/gather/matmul×2/matrix-engine/cache-blocking/control-flow）；推导式=1 组（route High / impact Medium） |
| 4 | Phase 4 输出要求满足 | ✅ 已读 pattern：`rvv_spatial_convolution_and_pooling_kernels.md`（命中 row：RVV Spatial Convolution and Pooling Kernels；引用短语首词：`interior/border 混在主循环`、`overlap-sensitive store`、§2 `Vectorize the interior`、§5 `Separate interior and border paths`、`vlse*` 替换、`padding 区域不能通过构造越界指针`；`The fix` 含 before/after、correctness contract、风险、预期 Profile signals）；supporting 已读 `no-vectorization.md`、`loop_induction_variable_strength_reduction.md`；missing `.S` 四证不成立（缺 dispatch slot 与 policy）→ 无 Implementation-shape proof；Related PRs：spatial=17、no-vectorization=19、IV-SR=3 |
| 5 | 路径合规 | ✅ 入口模式 A（profile-backed）；primary 按动态份额排序（98.3% 局部），supporting 不排序；每个 blueprint leaf 来自通过 gate 的 row（1 primary + 2 supporting）；`th.v*` 无 mismatch 未停扫；class 列表=2 |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧对 Phase 3 引用行（`12b73ae`/`12b73ba`/`12b73be`/`12b73c2`/`12b73c6`/`12b73b2`/`12b73b6`/`12b73b8`）；出现侧标注 `patterns/rvv_spatial_convolution_and_pooling_kernels.md` §Verification 与 `kernel-conventions.md` §3/§Verification |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁生成；无向用户追问（无 object-clarification 需要）；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测 |

修正记录：无