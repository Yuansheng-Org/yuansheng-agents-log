# 诊断日志 — faiss / sq-encode / rank 007

Functions under analysis: [`QuantizerFP16<(SL)0>::encode_vector`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`QuantizerFP16<(SL)0>::encode_vector`，**2 sample(s)**，event=`cpu-clock`，percent: local period）
- perf stat：已提供（sq-encode，IPC=1.890，L1 miss 2.336%，duration≈0.309s）
- hardware ISA：已提供（K3 `rv64imafdcvh`）；vlenb=32
- 采样元数据：已提供；样本数极少

## Phase 1 — Baseline / 基线
7 行 baseline 照常（hardware ISA=rv64imafdcvh；build ISA 基础 `rv64i2p1...c2p0`；vector flavor=SQ 内核 scalar；VLEN=256；bound=compute（IPC 1.890）；Sampling semantics=local period 单窗口；Sampling IP precision=`baseline_gap: sampling IP precision`）。

## Phase 2 — Scope / 分析边界
hot interval：335eaa: sh a5,0(a2) (50.00%), 335e90: addw a5,a5,t5 (50.00%)。样本 2，统计置信度低。

## Phase 3 — Pattern scan / 模式扫描：QuantizerFP16<
**Class selection trace（8 项）**：1. rows-asm exclude；2. rows-operator-rvv include；3. rows-string-memory exclude；4. rows-vectorized-tuning exclude；5. rows-codegen exclude；6. rows-offload exclude；7. rows-crypto exclude；8. rows-runtime-os exclude。

Classes scanned: rows-operator-rvv.md

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| No vectorization | 335eaa: sh a5,0(a2) (50.00%), 335e90: addw a5,a5,t5 (50.00%)；样本 2 | Low | Low | patterns/no-vectorization.md |

**(a) 逐字引用**：335eaa: sh a5,0(a2) (50.00%), 335e90: addw a5,a5,t5 (50.00%)  **(b) 互斥排除**：无 v*。**(c) 推导**：route=SQ encode 标量 + sq-rvv.cpp tag 继承 NONE；impact=Low。

## Phase 4 — Root-cause blueprint / 根因蓝图：QuantizerFP16<
1. **Root cause**：标量量化器 encode_vector 为逐元素标量 encode 内核（表查询/量化计算 lbu/fdiv.s/flt.s + 位打包 slliw/sb），未向量化；sq-rvv.cpp 中 QuantizerTemplate<...,RISCV_RVV>/QuantizerFP16/BF16 仅 opaque tag（继承 NONE）。样本 2（0.309s encode 微基准）极少。
2. **The fix / 修复方式**：为 SQ encode 提供真实 RVV 内核（vle32 批量量化 + vse8 打包；4/6bit 打包可用 vcompress/移位合并）。correctness：保持量化表查找、round 语义与 packing 布局；回归 QT_*_iterations 与 sq-accuracy sql2_recons_error。Profile signals：vle32/vse8 出现，标量 lbu/sb 循环消失。
3. **Baseline facts 回填**：hardware ISA=rv64imafdcvh；build ISA=rv64i2p1...c2p0；VLEN=256；bound=compute。
4. **收益上界**：局部份额 2/约29 ≈ 6.9%（0.309s 微基准内）。
5. **三维路由判定**：current source=compiler-generated 标量 SQ encode；implementation existence=无真实 RVV SQ encode；function-level policy=无 `.S` 合同。
6. **Implementation-shape proof**：不适用。
7. **Related PRs 小节**：https://github.com/opencv/opencv/pull/22179; https://github.com/opencv/opencv/pull/22520; https://github.com/opencv/opencv/pull/23980; https://github.com/opencv/opencv/pull/24058; https://github.com/opencv/opencv/pull/24132; https://github.com/opencv/opencv/pull/24166; https://github.com/opencv/opencv/pull/24301; https://github.com/opencv/opencv/pull/24325; https://github.com/opencv/opencv/pull/27160; https://github.com/opencv/opencv/pull/27119; https://github.com/opencv/opencv/pull/27097; https://github.com/opencv/opencv/pull/27007; https://github.com/opencv/opencv/pull/26958; https://github.com/opencv/opencv/pull/26865; https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d; https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a; https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706; https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f; https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a

## Phase 5 — Verification forecast / 验证预测：QuantizerFP16<
- 消失侧：335eaa: sh a5,0(a2)；出现侧：vle32/vse8；回归：重跑 sq-encode，QT_*_iterations 基线不变。

## Phase 6 — Completion check / 完成自检
| # | 检查项 | 状态（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现 | ✅ 1/1 组 |
| 2 | Phase 1 输出要求 | ✅ 7 行 + Sampling IP precision gap |
| 3 | Phase 3 输出要求 | ✅ 8 项 Class selection trace；样本 2 已记录 |
| 4 | Phase 4 输出要求 | ✅ patterns/no-vectorization.md |
| 5 | 路径合规 | ✅ 模式 A（样本极少，impact Low） |
| 6 | Phase 5 两侧锚定 | ✅ |
| 7 | 契约边界合规 | ✅ |

修正记录：无
