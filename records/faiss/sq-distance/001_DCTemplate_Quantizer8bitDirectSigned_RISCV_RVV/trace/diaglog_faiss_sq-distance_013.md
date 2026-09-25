# 诊断日志 — faiss / sq-distance / rank 013

Functions under analysis: [`DCTemplate<Quantizer8bitDirectSigned<6>,SimilarityIP<6>,(SL)0>::query_to_code`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`DCTemplate<Quantizer8bitDirectSigned<6>,SimilarityIP<6>,(SL)0>::query_to_code`，libfaiss.so，**2884 samples**，event=`cpu-clock`，percent: local period）
- perf stat：已提供（sq-distance，IPC=0.934，duration≈742s）
- workload/binary/source context：已提供（libfaiss.so；SQ 距离计算 benchmark）
- hardware ISA：已提供（K3 `rv64imafdcvh`，RVV 1.0）；vlenb=32
- 采样元数据：已提供；Sampling IP precision=`baseline_gap: sampling IP precision`

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`（含 `v`，RVV 1.0） |
| Build ISA | 含 RVV TU（sq-rvv.cpp），DC 内核已 RVV 向量化 |
| Vector flavor | RVV 1.0 `v*` |
| VLEN | 256 bits |
| Bound type | **latency-bound**（IPC=0.934，全用例最低） |
| Sampling semantics | event=cpu-clock；local period；单窗口 |
| Sampling IP precision | `baseline_gap: sampling IP precision` |

L0 baseline gate：无 build 缺 v；Bound-type gate：latency-bound，LMUL/归约结构为首要杠杆。

## Phase 2 — Scope / 分析边界
hot interval：57fcca: bnez a4 (80.51%), 57fca0: vsetvli e8mf4 (9.29%)。该函数为 SQ 距离计算（8bitDirectSigned codec）的 query_to_code/compute_distance 内核，sq-distance 主导热点。

## Phase 3 — Pattern scan / 模式扫描：DCTemplate<Quantizer8bitDirectSigned<6>,SimilarityIP<6>,
**Class selection trace（8 项）**：1. rows-asm exclude；2. rows-operator-rvv include；3. rows-string-memory exclude；4. rows-vectorized-tuning **include**（已 v*，LMUL/归约结构）；5. rows-codegen exclude；6. rows-offload exclude；7. rows-crypto exclude；8. rows-runtime-os exclude。

Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | 57fcca: bnez a4 (80.51%), 57fca0: vsetvli e8mf4 (9.29%)；vsetvli e32,m1 + 每迭代 vfredosum.vs 归约回标量再注入；IPC=0.934 | High | High | patterns/rvv_register_group_utilization.md |

| RVV Vector-State Management (supporting)（supporting） | 每迭代 2-3 次 vsetvli/vsetivli 配置 churn | — | — | patterns/rvv_vector_state_management.md |

**(a) 逐字 evidence 引用**：`58061a: vsetvli zero,a5,e32,m1,ta,ma`(4.93%)、`580612: vsetivli zero,1,e32,m1,ta,ma`、`580616: vfmv.s.f v3,fa0`、`58064c: vfredosum.vs v2,v1,v2`、`580650: vfmv.f.s fa0,v2`、`580654: bnez a4,580606`(88.24%)。

**(b) 互斥邻居排除**：- vs no-vectorization：v* 已存在 → 排除。- vs rvv_inactive_lane_policy：tail 非主导 → 排除。- vs memory-bound：L1 miss 率低、IPC 0.93 为 latency-bound → 排除。

**(c) 双 Confidence 推导式**：route：DC 已向量化 + 反汇编证明 LMUL=m1 与每迭代归约结构 + IPC 0.934 佐证 → High；impact：主导热点（2884/68,487≈4.2%）+ 每迭代归约串行链可消除 → High。

**仲裁**：primary=register-group/LMUL（含每迭代归约串行化）；supporting=vector-state（vsetvli churn）。

## Phase 4 — Root-cause blueprint / 根因蓝图：DCTemplate<Quantizer8bitDirectSigned<6>,SimilarityIP<6>,
1. **Root cause**：SQ 距离内核 query_to_code 已 RVV 向量化但 LMUL 选型欠佳：vsetvli e32,m1（VLEN=256 下仅 8 lane/迭代）且每迭代执行一次 vfredosum.vs 将累加器归约回标量再经 vsetivli 1 + vfmv.s.f 重新注入（fa0→v3→vfredosum→fa0 串行依赖链），叠加每次迭代 2-3 次 vsetvli 配置 churn；循环因此 latency-bound（perf stat IPC=0.934 佐证），热点样本 45-88% 落在循环回边分支。依据 patterns/rvv_register_group_utilization.md：'当前 LMUL 小于合法候选边界' + 归约结构导致串行依赖。
2. **The fix / 修复方式**：重构 sq-rvv.cpp 的 DC 内核：将标量累加器改为跨迭代存活的向量累加器（vfloat32mX acc，每迭代 vfmacc.vv，循环后一次 vfredusum.vs/vfredosum.vs 归约），消除每迭代的 vsetivli 1/vfmv.s.f/vfmv.f.s 与串行依赖；将 LMUL 提升到 m4/m8（枚举 m1/m2/m4/m8×unroll 候选，满足 LMUL*peak_live<=32 且无 spill），减少迭代与 vsetvli 次数。correctness：L2 保持逐分量平方累加语义，归约顺序变化需回归 sq-accuracy；IP 为 vfmacc 累加。Profile signals：循环回边样本显著下降、vfmacc.vv 占比上升、每迭代 vsetvli 减少。
3. **Baseline facts 回填**：hardware ISA=rv64imafdcvh；build ISA 含 RVV；VLEN=256；bound=latency（IPC 0.934）。
4. **收益上界**：当前 sampled event 局部份额 4.2%（主导热点）。
5. **三维路由判定**：current source=RVV intrinsic（sq-rvv.cpp DCTemplate）；implementation existence=已存在 RVV 内核（LMUL/归约结构待优化）；function-level policy=无 `.S` 合同。
6. **Implementation-shape proof**：不适用。
7. **Related PRs 小节**：`patterns/rvv_register_group_utilization.md`：13 条 URL（OpenCV/Linux/OpenBLAS/V8/vLLM）；`patterns/rvv_vector_state_management.md`：13 条 URL（V8/QEMU/LLVM）

## Phase 5 — Verification forecast / 验证预测：DCTemplate<Quantizer8bitDirectSigned<6>,SimilarityIP<6>,
- 消失/缩小侧：循环回边分支（如 `580654: bnez`）与 `vfmv.s.f/vfmv.f.s/vfredosum.vs` 每迭代样本应显著下降。
- 出现侧：`vfmacc.vv`（向量累加器）、更大的 `vsetvli ... e32,m4/m8`、循环后单次 `vfredusum.vs`。
- 回归：重跑 sq-distance，QT_*_iterations 与 sq-accuracy sql2_recons_error 基线不变；IPC 应从 0.934 提升。

## Phase 6 — Completion check / 完成自检
| # | 检查项 | 状态（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ 1/1 组 |
| 2 | Phase 1 输出要求 | ✅ 7 行 + Sampling IP precision gap；bound=latency |
| 3 | Phase 3 输出要求 | ✅ 8 项 Class selection trace；primary+supporting 已标注；evidence 锚点已列 |
| 4 | Phase 4 输出要求 | ✅ patterns 已读：rvv_register_group_utilization.md、rvv_vector_state_management.md；The fix 完整；Related PRs 已列 |
| 5 | 路径合规 | ✅ 模式 A；primary/supporting 分账明确 |
| 6 | Phase 5 两侧锚定 | ✅ |
| 7 | 契约边界合规 | ✅ |

修正记录：无
