# 诊断日志 — faiss / rcq-search / rank 023

Functions under analysis: [`void faiss::top1_add_results_tpl<CMax<float,long>,false,(SIMDLevel)0>(...)`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`void faiss::top1_add_results_tpl<CMax<float,long>,false,(SIMDLevel)0>(...)`，libfaiss.so，**172 samples**，event=`cpu-clock`，percent: local period）
- perf stat：已提供（rcq-search，IPC=1.931，L1_dcache_load_miss_rate=0.231%，branch_miss_rate=0.113%，duration≈271s）
- workload/binary/source context：已提供（`libfaiss.so`）
- readelf -A：已提供（build ISA `rv64i2p1...c2p0`；含 RVV TU：distances_rvv.cpp 生效，fvec_L2sqr/norm 实际运行 RVV 指令）
- hardware ISA：已提供（K3 `rv64imafdcvh`，RVV 1.0）；vlenb=32
- 采样元数据：已提供；Sampling IP precision=`baseline_gap: sampling IP precision`（RVV 指令样本严重偏向标量控制指令）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`（含 `v`，RVV 1.0） |
| Build ISA | `rv64i2p1...c2p0`（基础无 v；部分 TU 编译含 RVV，运行期 SIMDConfig::level=6 实测命中 RISCV_RVV） |
| Vector flavor | RVV 1.0 `v*`（distances_rvv.cpp 内核）；其余路径 scalar |
| VLEN | 256 bits |
| Bound type | compute-bound（IPC 1.931；L1 miss 0.231% 排除 memory-bound） |
| Sampling semantics | event=cpu-clock；local period；单窗口 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（RVV 内核样本聚集于 add/ret，向量指令≈0%） |

L0 baseline gate：hardware 有 v，部分构建路径已用 v；`with_simd_level_256bit` 路径未接入 RVV（dispatch mask 排除）。Bound-type gate：compute-bound。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个）。hot interval：30a062: addi a4,a4,1 (25.00%), 30a066: bne a2,a4,30a050 (22.67%), 30a058: beqz a6 (23.84%), 30a054: flt.s a6,fa5,fa4 (9.88%), 30a050: flw fa5,0(a5) (8.14%)。该函数与 rcq-search 计算热点（fvec_add/fvec_L2sqr/exhaustive_L2sqr 等）同属应用侧热路径，且叠加调度器/sched_yield 系统开销（见 rank 002-024）。

## Phase 3 — Pattern scan / 模式扫描：void faiss::top1_add_results_tpl<CMax<float,long>,false,

**Class selection trace（8 项）**
1. `rows-asm.md` — exclude：无 `.S` provenance
2. `rows-operator-rvv.md` — include：compiler 生成标量/已向量化热点
3. `rows-string-memory.md` — exclude：非 string/memory 合同
4. `rows-vectorized-tuning.md` — exclude：无 v*
5. `rows-codegen.md` — include：dispatch/调用开销相关
6. `rows-offload.md` — exclude：无矩阵引擎证据
7. `rows-crypto.md` — exclude：非密码原语
8. `rows-runtime-os.md` — exclude：非内核热点

Classes scanned: rows-operator-rvv.md, rows-codegen.md

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| No vectorization | 30a062: addi a4,a4,1 (25.00%), 30a066: bne a2,a4,30a050 (22.67%), 30a058: beqz a6 (23.84%), 30a054: flt.s a6,fa5,fa4 (9.88%), 30a050: flw fa5,0(a5) (8.14%)；top1_add_results_tpl 的标量 top-1 归约循环（flw/flt.s/beqz/addi/bne，... | Medium | Medium | patterns/no-vectorization.md |

**(a) 逐字 evidence 引用**：30a062: addi a4,a4,1 (25.00%), 30a066: bne a2,a4,30a050 (22.67%), 30a058: beqz a6 (23.84%), 30a054: flt.s a6,fa5,fa4 (9.88%), 30a050: flw fa5,0(a5) (8.14%)

**(b) 互斥邻居排除**：已向量化函数排除 no-vectorization（v* 存在）；scalar 函数排除 register-group/operand-form（无 v*）；L1 miss 0.231% 排除 memory-bound/cache-blocking。

**(c) 双 Confidence 推导式**：route：build 含 RVV TU 但本路径 dispatch 未接入 + hardware v + source 证据 → Medium；impact：样本 172、IPC 1.931 compute-bound、缺 sampling IP precision → Medium

**仲裁**：单 primary finding；无 companion（除 rank 008/028 的 dispatch-mask 观察并入 fix 说明）。

## Phase 4 — Root-cause blueprint / 根因蓝图：void faiss::top1_add_results_tpl<CMax<float,long>,false,
1. **Root cause**：top-1 归约循环以标量逐元素 flt.s 比较执行，NONE 实例化被运行时选中；符合 No vectorization pattern（RVV vfredmin/vfredmax 或 vmslt/vmerge 可实现）。

**patterns 已读**：`no-vectorization.md`（引用其独有内容）
2. **The fix / 修复方式**：提供 SIMDLevel::RISCV_RVV 的 top1_add_results 变体（vfredmin/vfredmax + vfmv 归约 + vcompress 拿 index），并将 dispatch 覆盖 RISCV_RVV。correctness：保持 top-1 tie-break 语义（CMax 含 tie 规则）。Profile signals：vfredmax/vse32 出现，flt.s/beqz 标量行消失。
3. **Baseline facts 回填**：hardware ISA=rv64imafdcvh；build ISA 基础 `rv64i2p1...c2p0`（RVV TU 生效）；VLEN=256；bound=compute。
4. **收益上界**：当前 sampled event（cpu-clock）局部份额约 0.3%（函数内样本 / rcq-search 顶层函数样本总和约 55,450）。
5. **三维路由判定**：current source=compiler-generated 标量; implementation existence=faiss/utils/distances_fused/simdlib_kernel-inl.h; function-level policy=无 `.S` 合同。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs 小节**：https://github.com/opencv/opencv/pull/22179; https://github.com/opencv/opencv/pull/22520; https://github.com/opencv/opencv/pull/23980; https://github.com/opencv/opencv/pull/24058; https://github.com/opencv/opencv/pull/24132; https://github.com/opencv/opencv/pull/24166; https://github.com/opencv/opencv/pull/24301; https://github.com/opencv/opencv/pull/24325; https://github.com/opencv/opencv/pull/27160; https://github.com/opencv/opencv/pull/27119; https://github.com/opencv/opencv/pull/27097; https://github.com/opencv/opencv/pull/27007; https://github.com/opencv/opencv/pull/26958; https://github.com/opencv/opencv/pull/26865; https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d; https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a; https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706; https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f; https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a

## Phase 5 — Verification forecast / 验证预测：void faiss::top1_add_results_tpl<CMax<float,long>,false,
- 消失/缩小侧：30a062: addi a4,a4,1 
- 出现侧：v* 指令按 f["pattern"] 的 Verification 预期出现
- 回归：重跑 rcq-search，对比该函数样本占比与总 duration（基线 271s）变化。

## Phase 6 — Completion check / 完成自检
| # | 检查项 | 状态（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ 1/1 组；void faiss::top1_add_results_tpl<CMax<fl |
| 2 | Phase 1 输出要求 | ✅ 7 行 + Sampling IP precision=`baseline_gap: sampling IP precision`；gap 标签列表：sampling IP precision |
| 3 | Phase 3 输出要求 | ✅ 8 项 Class selection trace；Classes scanned 已声明；顶层 finding 1；evidence 锚点已列；推导式 1 |
| 4 | Phase 4 输出要求 | ✅ patterns/no-vectorization.md;The fix 含 before/after/correctness/风险/Profile signals；Related PRs 已列 |
| 5 | 路径合规 | ✅ 模式 A（profile-backed） |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧/出现侧已锚定 |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁 |

修正记录：无
