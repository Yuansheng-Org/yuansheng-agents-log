# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-encode-008`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-encode-008`
- 派生来源快照用例：`sq-encode/008_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_encode_vector`（候选 `candidate-55cb7d9f5f59e4317afc256d874ed229.patch`）
- 变更文件：`faiss/impl/scalar_quantizer/quantizers.h`、`faiss/impl/scalar_quantizer/sq-rvv.cpp`

## 派生字段映射

- `reviewResult` ← `blueprint/machine-validation.json#status`
- `hallucinationCheck` ← `blueprint/semantic-validation.json#dimensions (five-dimension)`
- `archReview.notes` ← `blueprint.craft_actions.actions[0].proposal`
- `archReview.status` ← `blueprint.craft_actions.actions[0].readiness`
- `archReview.changeRiskLevel` ← `blueprint.craft_actions.actions[0].change_risk.level`
- `archReview.controls` ← `blueprint.craft_actions.actions[0].change_risk.controls`
- `findings[].detail` ← `blueprint.craft_actions.actions[0].change_risk.reasons`

## 上游机器校验（原件 `machine-validation.json`）

- `status = pass`
- checks = `blueprint_strict_json`, `blueprint_v1_schema`, `blueprint_cross_field_rules`, `claim_to_evidence_binding`, `evidence_size_and_sha256`

## 上游五维语义校验（原件 `semantic-validation.json`）

- **claim_traceability**：`pass` — claim_traceability passed deterministic assembly validation.
- **explainability**：`pass` — explainability passed deterministic assembly validation.
- **internal_consistency**：`pass` — internal_consistency passed deterministic assembly validation.
- **safety_guardrails**：`pass` — safety_guardrails passed deterministic assembly validation.
- **technical_accuracy**：`pass` — technical_accuracy passed deterministic assembly validation.

## 变更风险（来自 `blueprint.craft_actions`）

- 风险等级：`low`

风险理由：
- Zvfhmin vfncvt.f.f.w rounding/NaN canonicalization and denormal handling may differ from the ryg bit-trick (qNaN 0x7E00, Inf 0x7C00, denormals via FP32 denormals): mismatches would silently change stored codes unless repaired on the masked special lanes.
- Vectorization changes the encode's numerical path (FRM, FTZ, NaN payload) which affects downstream distance computations; any bit-pattern deviation changes index results, not just speed.
- A fixed 16-bit store stream requires vse16 alignment/layout identical to the ((uint16_t*)code) contract; an LMUL or SEW mismatch could corrupt adjacent code entries.
- If SIMDLevel<0> exists specifically to provide a scalar/reference path, unconditional vectorization could regress short-vector calls or duplicate maintenance.

控制措施：
- Gate the vector path behind a zvfhmin-capability check and keep the scalar ryg helper as the reference and fallback.
- Build a bit-exact conformance test comparing vector vs scalar ryg output across the full special-value corpus (NaN payloads, ±Inf, ±0, denormals, 65504/65520 boundaries, half-ties) before any perf comparison.
- Add a short-vector crossover (use scalar below a measured d threshold) and re-measure the 20-iteration QT_fp16 benchmark after landing.
- Verify one vsetvl per main-loop interval with no vtype switching inside; count header has zero vector spills.

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Replace the inlined scalar ryg encode_fp16 loop in QuantizerFP16<0>::encode_vector (quantizers.h:246) with a stable-vtype RVV narrowing fast path when zvfhmin is available: vsetvl e32, vle32.v load of the float block, a masked detect of special lanes (NaN / ±Inf / boundary), vfncvt.f.f.w (Zvfhmin f32->f16 narrowing, RNE through FRM) on the bulk lanes, a masked repair that writes the scalar ryg bit patterns (qNaN 0x7E00|sign, Inf 0x7C00|sign) only on special lanes, then vse16.v to the uint16_t code buffer; keep the untouched ryg encode_fp16 as the invariant scalar fallback and for the residual tail/SIMDLevel-0 path.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
