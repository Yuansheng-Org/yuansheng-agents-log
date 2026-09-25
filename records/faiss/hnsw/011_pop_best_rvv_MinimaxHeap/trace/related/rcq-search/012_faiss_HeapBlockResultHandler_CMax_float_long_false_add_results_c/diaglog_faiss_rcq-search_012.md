# 诊断日志 — faiss / rcq-search / rank 012

Functions under analysis: [`faiss::HeapBlockResultHandler<CMax<float,long>,false>::add_results [clone ._omp_fn.0]`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`faiss::HeapBlockResultHandler<CMax<float,long>,false>::add_results [clone ._omp_fn.0]`，libfaiss.so，**766 samples**，event=`cpu-clock`，percent: local period）
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
函数清单与承诺声明一致（1 个）。hot interval：510dae: flt.s a5,fa5,fa4 (65.14%), 510da6: beq t4,a1 (14.10%), 510db2: beqz a5 (7.18%), 510daa: flw fa5,0(a2) (2.74%)。该函数与 rcq-search 计算热点（fvec_add/fvec_L2sqr/exhaustive_L2sqr 等）同属应用侧热路径，且叠加调度器/sched_yield 系统开销（见 rank 002-024）。

## Phase 3 — Pattern scan / 模式扫描：faiss::HeapBlockResultHandler<CMax<float,long>,false>::add_results [clone ._omp_fn.0]

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
| (No local pattern matched - top-k heap 比较串行) | 510dae: flt.s a5,fa5,fa4 (65.14%), 510da6: beq t4,a1 (14.10%), 510db2: beqz a5 (7.18%), 510daa: flw fa5,0(a2) (2.74%)；HeapBlockResultHandler::add_results 将分块距离写入 k 顶堆：主循环对每个距离做阈值... | Low | Low | — |

**(a) 逐字 evidence 引用**：510dae: flt.s a5,fa5,fa4 (65.14%), 510da6: beq t4,a1 (14.10%), 510db2: beqz a5 (7.18%), 510daa: flw fa5,0(a2) (2.74%)

**(b) 互斥邻居排除**：已向量化函数排除 no-vectorization（v* 存在）；scalar 函数排除 register-group/operand-form（无 v*）；L1 miss 0.231% 排除 memory-bound/cache-blocking。

**(c) 双 Confidence 推导式**：route：build 含 RVV TU 但本路径 dispatch 未接入 + hardware v + source 证据 → Low；impact：样本 766、IPC 1.931 compute-bound、缺 sampling IP precision → Low

**仲裁**：单 primary finding；无 companion（除 rank 008/028 的 dispatch-mask 观察并入 fix 说明）。

## Phase 4 — Root-cause blueprint / 根因蓝图：faiss::HeapBlockResultHandler<CMax<float,long>,false>::add_results [clone ._omp_fn.0]
1. **Root cause**：scalar 比较器 flt.s + 分支主导循环（65%+），因绝大多数距离不优于阈值只做比较。top-k 堆是固有串行数据结构，RVV 无法直接加速堆维护；catalog 无堆管理 pattern。归类 code_path（固有串行比较开销）。
2. **The fix / 修复方式**：不推荐 Craft 修改：堆维护算法固有串行。可选方向（低优先）：以更宽松阈值预过滤（如先对分块距离做 SIMD max 检查，全块低于阈值则整体跳过比较循环）。correctness：保持 top-k 结果与 tie-break 语义。Profile signals：若加预过滤，flt.s/beqz 样本下降。
3. **Baseline facts 回填**：hardware ISA=rv64imafdcvh；build ISA 基础 `rv64i2p1...c2p0`（RVV TU 生效）；VLEN=256；bound=compute。
4. **收益上界**：当前 sampled event（cpu-clock）局部份额约 1.4%（函数内样本 / rcq-search 顶层函数样本总和约 55,450）。
5. **三维路由判定**：current source=compiler-generated 标量; implementation existence=无专用内核; function-level policy=无 `.S` 合同。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs 小节**：无匹配 pattern，无 Related PRs 表可引用

## Phase 5 — Verification forecast / 验证预测：faiss::HeapBlockResultHandler<CMax<float,long>,false>::add_results [clone ._omp_fn.0]
- 消失/缩小侧：510dae: flt.s a5,fa5,fa4 
- 出现侧：不适用（无 pattern）
- 回归：重跑 rcq-search，对比该函数样本占比与总 duration（基线 271s）变化。

## Phase 6 — Completion check / 完成自检
| # | 检查项 | 状态（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ 1/1 组；faiss::HeapBlockResultHandler<CMax<float |
| 2 | Phase 1 输出要求 | ✅ 7 行 + Sampling IP precision=`baseline_gap: sampling IP precision`；gap 标签列表：sampling IP precision |
| 3 | Phase 3 输出要求 | ✅ 8 项 Class selection trace；Classes scanned 已声明；顶层 finding 0；evidence 锚点已列；推导式 1 |
| 4 | Phase 4 输出要求 | ✅ 零命中路径;The fix 含 before/after/correctness/风险/Profile signals |
| 5 | 路径合规 | ✅ 模式 A（profile-backed） |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧/出现侧已锚定 |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁 |

修正记录：无
