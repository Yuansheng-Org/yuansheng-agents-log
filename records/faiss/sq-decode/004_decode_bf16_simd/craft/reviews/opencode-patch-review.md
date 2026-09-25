# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-decode-013`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-decode-013`
- 派生来源快照用例：`sq-decode/013_faiss_scalar_quantizer_QuantizerBF16_faiss_SIMDLevel_0_decode_vector`（候选 `candidate-14c3155f744bb993b35d48b3fcba4818.patch`）
- 变更文件：`faiss/utils/bf16.h`

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
- Vzext signedness/width must match _mm512_cvtepu16_epi32 semantics: an unsigned zero-extension (vzext.vf2) is required; a signed extension or an FP conversion would corrupt bit patterns
- Widening changes LMUL budget (e16 source m1 widens to e32 m~2); a misconfigured SEW/LMUL can exceed VLMAX or introduce per-iteration vsetvli overhead that eats the gain
- Toolchain/intrinsic availability and the header's x86-only include guard need careful #if handling to keep the header portable across ISAs
- Tail handling for n not divisible by VLMAX must keep the scalar reference exact
- GCC/LLVM might already scalar-unroll and outperform a naive first vector version, so a measured gate is required before replacing the existing path

控制措施：
- Gate the new block with #if defined(__riscv_vector) and otherwise keep the current scalar tail identical
- Use only vzext.vf2 + vsll.vi and validate output bit-for-bit against the scalar decode_bf16 reference before benchmarking
- Keep a fixed-VL main loop plus runtime-VL tail per kernel-conventions to keep vsetvli stable in the hot interval
- Run faiss scalar-quantizer/bf16 unit tests unchanged and compare against the AVX512 path logic on x86 if available

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Add an RVV fast path to decode_bf16_simd in faiss/utils/bf16.h, guarded by __riscv_vector, mirroring the existing __AVX512F__ widening sequence: per chunk load u16 with vle16.v, widen with unsigned vzext.vf2 (e32), shift left by 16 with vsll.vi, store with vse32.v, then fall through to the existing scalar tail for the remainder; leave the AVX512 path and scalar fallback intact.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
