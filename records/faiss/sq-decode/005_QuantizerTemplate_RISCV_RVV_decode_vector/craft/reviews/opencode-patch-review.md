# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-decode-002`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-decode-002`
- 派生来源快照用例：`sq-decode/002_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec4bit_faiss_SIMDLevel_6_9652b988`（候选 `candidate-d5e4afe64193b60e461707799a1c7d25.patch`）
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
- Replacing the exact /15.0f with a reciprocal multiply changes the last-bit FP rounding of reconstructed components, which can propagate into downstream distance values
- Interleaving low/high nibbles back into component order is the classic even/odd lane-ordering bug site
- A VLEN-agnostic tail is mandatory; an off-by-one on packed byte loads can read past code_size
- A new duplicate decode path increases maintenance surface across NONE/AVX2/AVX512/RVV variants

控制措施：
- Keep the scalar NONE-base decode_vector untouched so non-RVV and fallback builds produce the original bit pattern
- Gate the new override with the same #ifdef COMPILE_SIMD_RISCV_RVV used by existing sq-rvv.cpp kernels
- Add a unit-style comparison harness that decodes random codes through RVV and scalar paths and reports max abs/rel diff, and treat any nibble-order bug as a hard failure
- Never assume a fixed VLEN or a d multiple of VLEN; derive vl from remaining count each iteration

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Add a real VLEN-agnostic RVV decode_vector override to the QuantizerTemplate RISCV_RVV specializations in faiss/impl/scalar_quantizer/sq-rvv.cpp (NON_UNIFORM, and UNIFORM for consistency), mirroring the AVX2/AVX512 Codec4bit decode_8/16_components approach without any fixed width: strip-mined loop that loads vl/2 packed code bytes (vle8), unpacks low/high 4-bit nibbles (vand 0x0F / vsrl 4 or even-odd deinterleave), converts to float (vzext + vfwcvt.f.xu or vwf), applies the affine decode (0.5f*inv15 offset and inv15 scale via precomputed scalars), and fuses vmin[i]+vdiff[i]*x with vfmacc against contiguous vmin/vdiff vectors, ending with vse32 for the even/odd interleaved component order; keep fixed-VL main loop plus runtime-VL tail per kernel conventions and leave the scalar NONE base as the fallback for non-RVV builds.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
