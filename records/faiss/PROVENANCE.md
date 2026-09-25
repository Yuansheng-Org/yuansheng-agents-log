# `records/faiss` 归档溯源说明

本文件说明本次整理（2 级扁平 → mxnet 同构 3 级嵌套）中，**每个文件的来源与真实性等级**。

## 真实性等级

| 等级 | 含义 |
| --- | --- |
| **原件** | 上游真实存在的文件的逐字节拷贝（LF 归一）。可直接与来源核对。 |
| **派生** | 上游**不存在**该文件；由上游真实字段按固定映射生成。文件内含 `provenance` 块，正文顶部亦标注。 |
| **无来源** | 上游与快照均无对应物料，不生成（以缺失体现，不伪造）。 |

## 逐条溯源

### 001 — `sq-distance/001_DCTemplate_Quantizer8bitDirectSigned_RISCV_RVV/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea` 的候选 `candidate-42f3412bbfa105b82a1767fc307da3ad.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/001_DCTemplate_Quantizer8bitDirectSigned_RISCV_RVV.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/001_DCTemplate_Quantizer8bitDirectSigned_RISCV_RVV.md` | 原件 | 桌面提交包 `pr-des/1-PR-5602.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-distance/013_DCTemplate_Quantizer8bitDirectSigned_6_SimilarityIP_6_SL_0_query` 全目录 |

### 002 — `sq-encode/002_QuantizerFP16_RISCV_RVV_encode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector` 的候选 `candidate-55cb7d9f5f59e4317afc256d874ed229.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/002_QuantizerFP16_RISCV_RVV_encode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/002_QuantizerFP16_RISCV_RVV_encode_vector.md` | 原件 | 桌面提交包 `pr-des/2-PR-5603.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-encode/007_QuantizerFP16_SL_0_encode_vector` 全目录 |

### 003 — `sq-encode/003_Quantizer8bitDirect_RISCV_RVV_encode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-encode/011_faiss_scalar_quantizer_Quantizer8bitDirect_faiss_SIMDLevel_0_encode_vector` 的候选 `candidate-24ffabae714d0188f9e7d3e8b652377a.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-encode/011_faiss_scalar_quantizer_Quantizer8bitDirect_faiss_SIMDLevel_0_encode_vector/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-encode/011_faiss_scalar_quantizer_Quantizer8bitDirect_faiss_SIMDLevel_0_encode_vector/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-encode/011_faiss_scalar_quantizer_Quantizer8bitDirect_faiss_SIMDLevel_0_encode_vector/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/003_Quantizer8bitDirect_RISCV_RVV_encode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/003_Quantizer8bitDirect_RISCV_RVV_encode_vector.md` | 原件 | 桌面提交包 `pr-des/3-PR-5604.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-encode/010_Quantizer8bitDirect_SL_0_encode_vector` 全目录 |

### 004 — `sq-decode/004_decode_bf16_simd/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-decode/013_faiss_scalar_quantizer_QuantizerBF16_faiss_SIMDLevel_0_decode_vector` 的候选 `candidate-14c3155f744bb993b35d48b3fcba4818.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-decode/013_faiss_scalar_quantizer_QuantizerBF16_faiss_SIMDLevel_0_decode_vector/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-decode/013_faiss_scalar_quantizer_QuantizerBF16_faiss_SIMDLevel_0_decode_vector/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-decode/013_faiss_scalar_quantizer_QuantizerBF16_faiss_SIMDLevel_0_decode_vector/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/004_decode_bf16_simd.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/004_decode_bf16_simd.md` | 原件 | 桌面提交包 `pr-des/4-PR-5631.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-decode/012_QuantizerBF16_SL_0_decode_vector` 全目录 |

### 005 — `sq-decode/005_QuantizerTemplate_RISCV_RVV_decode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988` 的候选 `candidate-d5e4afe64193b60e461707799a1c7d25.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/005_QuantizerTemplate_RISCV_RVV_decode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/005_QuantizerTemplate_RISCV_RVV_decode_vector.md` | 原件 | 桌面提交包 `pr-des/6-rvv-sq-decode-codec-template.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-decode/005_QuantizerTemplate_Codec4bit_RISCV_RVV_Scaling_SL_0_decode_vector` 全目录 |

### 006 — `sq-decode/006_QuantizerFP16_BF16_8bitDirect_RISCV_RVV_decode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-accuracy/012_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_decode_vector` 的候选 `candidate-dfdcb87a45ab33f88f9ed861109f9804.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-accuracy/012_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_decode_vector/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-accuracy/012_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_decode_vector/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-accuracy/012_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_decode_vector/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/006_QuantizerFP16_BF16_8bitDirect_RISCV_RVV_decode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/006_QuantizerFP16_BF16_8bitDirect_RISCV_RVV_decode_vector.md` | 原件 | 桌面提交包 `pr-des/7-rvv-sq-decode-raw-codecs.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-decode/013_QuantizerFP16_SL_0_decode_vector` 全目录 |

### 007 — `sq-encode/007_QuantizerTemplate_RISCV_RVV_encode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-accuracy/001_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec8bit_faiss_SIMDLevel_6_0550d977` 的候选 `candidate-409a6b766b3ff48db24b1ec327cd24cb.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-accuracy/001_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec8bit_faiss_SIMDLevel_6_0550d977/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-accuracy/001_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec8bit_faiss_SIMDLevel_6_0550d977/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-accuracy/001_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec8bit_faiss_SIMDLevel_6_0550d977/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/007_QuantizerTemplate_RISCV_RVV_encode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/007_QuantizerTemplate_RISCV_RVV_encode_vector.md` | 原件 | 桌面提交包 `pr-des/8-rvv-sq-encode-codec-template.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-encode/013_QuantizerTemplate_Codec8bit_RISCV_RVV_Scaling0_SL_0_encode_vecto` 全目录 |

### 008 — `sq-encode/008_QuantizerFP16_RISCV_RVV_encode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/patch.diff` | 原件 | 原 `records/faiss` 记录；与快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector` 的候选 `candidate-eaaa78e3e6246d2d1c16e4588b544318.patch` LF 归一同字节 |
| `craft/patch-plan.json` | 派生 | ← 快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector/blueprint/blueprint.json` 的 `summary.observed_anomaly` + `craft_actions.actions[0].proposal` / `.conditions[]`；`changes[].filePath` ← 真实 `patch.diff` 的 `diff --git` 头 |
| `craft/patch-candidate.json` | 派生 | `gitDiff` ← 真实 `craft/patch.diff` 正文；`changedFiles` ← 其 `diff --git` 头 |
| `craft/reviews/semantic-validation.json` | 原件 | 快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector/blueprint/semantic-validation.json` |
| `craft/reviews/machine-validation.json` | 原件 | 快照 `sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector/blueprint/machine-validation.json` |
| `craft/reviews/opencode-patch-review.json` | 派生 | `reviewResult` ← machine-validation#status；`hallucinationCheck` ← semantic-validation#dimensions；`archReview`/`findings` ← `craft_actions.actions[0]` |
| `craft/reviews/opencode-patch-review.md` | 派生 | 同上（人类可读版） |
| `reviews/008_QuantizerFP16_RISCV_RVV_encode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/008_QuantizerFP16_RISCV_RVV_encode_vector.md` | 原件 | 桌面提交包 `pr-des/9-rvv-sq-encode-raw-codecs.md` |
| `trace/*-annotate.txt` 等 5 件 | 原件 | Agent Debug Trace `trace/faiss/sq-encode/007_QuantizerFP16_SL_0_encode_vector` 全目录 |

### 009 — `sq-encode/009_QuantizerLloydMax_RISCV_RVV_encode_vector/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/NOTE.md` | 原件 | 原 `records/faiss` 记录（说明为何无优化前补丁） |
| `reviews/009_QuantizerLloydMax_RISCV_RVV_encode_vector.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/009_QuantizerLloydMax_RISCV_RVV_encode_vector.md` | 原件 | 桌面提交包 `pr-des/10-rvv-sq-encode-lloydmax.md` |
| `trace/` | 无来源 | 106 个 trace 用例中无同 demangled 符号命中；未生成 |

### 010 — `sq-decode/010_QuantizerTemplate_QuantizerFP16_RISCV_RVV_decode_vector_increment/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/NOTE.md` | 原件 | 原 `records/faiss` 记录（说明为何无优化前补丁） |
| `reviews/010_QuantizerTemplate_QuantizerFP16_RISCV_RVV_decode_vector_increment.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/010_QuantizerTemplate_QuantizerFP16_RISCV_RVV_decode_vector_increment.md` | 无来源 | 桌面 `pr-des/` 无对应文件；未生成 |
| `trace/` | 无来源 | 106 个 trace 用例中无同 demangled 符号命中；未生成 |

### 011 — `hnsw/011_pop_best_rvv_MinimaxHeap/`

| 文件 | 等级 | 来源 |
| --- | --- | --- |
| `craft/NOTE.md` | 原件 | 原 `records/faiss` 记录（说明为何无优化前补丁） |
| `reviews/011_pop_best_rvv_MinimaxHeap.patch` | 原件 | 原 `records/faiss` 的 `review/patch-opt.diff` |
| `reviews/011_pop_best_rvv_MinimaxHeap.md` | 原件 | 桌面提交包 `pr-des/patch.hnsw-popmin-on-main.md` |
| `trace/` | 无来源 | 106 个 trace 用例中无同 demangled 符号命中；未生成 |

## 未纳入本次整理的上游物料

### 1. 桌面提交包 `5-rvv-fvec-pq-distance-dispatch.patch`（9007 B）

按本次范围决定（**只重组现有 11 条**），该补丁**不新设为记录**。其情况记录如下：

- 目标为 `fvec_add` / `fvec_sub` / `compute_PQ_dis_tables_dsub2` 的 RVV 内核与 dispatch 修复，
  触及 `ProductQuantizer.cpp`、`simd_dispatch.h`、`distances_dispatch.h`、`distances_rvv.cpp`。
- 全部 22 个 `records/faiss` 候选文件中**无同源**；桌面包之外无对应记录（序号 001–011 连续无空洞）。
- 若日后归档，`trace/faiss/rcq-search/008`（`fvec_add`）、`pq-dis-tables-dsub2/001`
  （`compute_PQ_dis_tables_dsub2`）有同源 trace 用例，`craft/faiss` 亦有同名目录。
- 其 PR 描述为 `pr-des/5-rvv-fvec-pq-distance-dispatch.md`，已随本次整理保留在桌面侧，未入库。

### 2. `yuansheng-agent-debug` 的 `craft/faiss`（20 目录 × 5 文件）

**未纳入**，且**不可作为本 11 条的 craft 证据**：

- 其 20 个 `blueprintId` 全部落在 `sq-distance`(17) / `rcq-search`(2) / `pq-dis-tables-dsub2`(1)，
  即 `query_to_code` / `fvec_*` / `Clustering` 家族；本 11 条记录的目标是 `encode_vector` / `decode_vector`，
  **覆盖 0/11**。
- 20/20 目录的 `patch-plan.json#changes[].filePath` 与该目录自身 `patch-candidate.json#changedFiles`
  **互相矛盾**（plan 称改 1 个文件、candidate 实改 4 个）。
- 但 `patch.diff` 与该目录自己的 `gitDiff` **20/20 LF 归一同字节** ⇒ 不是"正文被覆盖"，
  而是 `patch-plan.json` 本身不可信。
- 结论：以该目录任一字段作为 craft 证据都会出错，故整棵树不用于本 11 条。

## 两条记录共用同一 trace / 快照用例

| 记录 | 快照用例 | trace 用例 |
| --- | --- | --- |
| 002 / 008 | `sq-encode/008` | `sq-encode/007` |

002 与 008 的 `craft/patch.diff` 分别等于该快照用例的**两个不同候选**
（`candidate-55cb7d9f…` / `candidate-eaaa78e3…`），因此两条记录共享同一个 trace 用例，
`trace/` 内容相同、编号均为 trace 侧的 `007`。**不得据「序号」断言 11 条与 11 个用例一一对应。**

## 复现方式

```bash
# 快照：29 case / 48 candidate
#   E:/snap/faiss-1f93154314afbef210f0ebebeab840da22f9ec7d/be8a0c3a4aed4276acf9c4a638466869/
# trace：106 case     E:/t/trace/faiss/
# 原记录：11 条       E:/rf/records/faiss/
# 桌面包：11 补丁+PR描述  C:/Users/LYD/Desktop/faiss-patch/
```

配对键（**不得按序号**）：

- 记录 ↔ 桌面包：`craft/patch.diff` 与 `reviews/*.patch` 的 LF 归一逐字节同一性。
- 记录 ↔ 快照候选：`craft/patch.diff` 与 `candidate-*.patch` 的 LF 归一同字节。
- 记录 ↔ trace 用例：从快照 `blueprint/evidence/annotate.txt` 提取完整 demangled 符号，
  与 106 个 trace 用例的 annotate 做**全串相等**。
