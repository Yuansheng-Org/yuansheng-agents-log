# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-accuracy-001`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-accuracy-001`
- 派生来源快照用例：`sq-accuracy/001_faiss_scalar_quantizer_QuantizerTemplate_faiss_scalar_quantizer_Codec8bit_faiss_SIMDLevel_6_0550d977`（候选 `candidate-409a6b766b3ff48db24b1ec327cd24cb.patch`）
- 变更文件：`faiss/impl/scalar_quantizer/quantizers.h`

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
- RVV elementwise FP ops must reproduce the scalar per-lane roundings: replacing fdiv.s then fmul.s by a single precomputed reciprocal multiply can change low-order bits and recons error
- vfmin/vfmax NaN and signed-zero semantics differ from the branchy scalar clamp; the reference clamp must win on special values
- vdiff==0 must map every lane to 0 exactly; a per-lane test inside the vector body would be wrong
- dynamic-tail and vsetvl/v set up add overhead that can regress very short vectors (small d)
- reachability risk: adding intrinsics to the header/cpp without keeping the NONE fallback compiled can leave non-RVV platforms broken or the realized path unreachable
- fusing *255 into the scale constant changes error vs the reference (fdiv then mul), so it must be validated even if faster

控制措施：
- Gate the RVV path with the same compile/runtime SIMD gate used by the rest of faiss (COMPILE_SIMD_RISCV_RVV / SIMDConfig) and keep the scalar NONE base for comparison and fallback
- Choose clamp instructions to exactly match the scalar compare semantics, or use vmerge from vmflt/vmfgt masks; verify NaN and ±0 behavior on hardware/simulator
- Add an explicit lane-independent shortcut for vdiff==0 written once before the strip-mine loop
- Benchmark d sweep (e.g. 4..512) and pick LMUL and a short-input threshold on the target core; do not hard-code a lane count
- Compile-check the RVV path and re-verify the ELF attributes include v; confirm the symbol instantiates the new path via disassembly
- A/B validate error by comparing vector vs scalar encode bit-for-bit before switching

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Implement a real RVV encode_vector for the QT_8bit (Codec8bit<RISCV_RVV>) QuantizerTemplate in sq-rvv.cpp instead of inheriting the scalar NONE base: precompute scalar invariants (vmin, vdiff, and 255/vdiff reciprocal, plus the vdiff==0 shortcut) before the loop, then strip-mine over d with dynamic __riscv_vsetvl, vle32 unit-stride loads of x, vfsub.vf vmin, vfmul.vf (or vfdiv.vv) scale, two-sided clamp via vfmin/vfmax (or compare+mask-merge to match the reference's NaN/signed-zero behavior), vfcvt.rtz.xu.f.v and vfncvt narrowing to uint8, then vse8 stores; keep the existing scalar implementation for non-RVV builds and for a short-input tail.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
