# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-accuracy-012`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-accuracy-012`
- 派生来源快照用例：`sq-accuracy/012_faiss_scalar_quantizer_QuantizerFP16_faiss_SIMDLevel_0_decode_vector`（候选 `candidate-dfdcb87a45ab33f88f9ed861109f9804.patch`）
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

- 风险等级：`medium`

风险理由：
- Widening conversion doubles the destination register group (e16m1 -> e32m2); with VLEN=128 the chosen LMUL must keep the e32 destination within the register file and not force spills.
- vfwcvt.f.f.v NaN quieting/payload rules may differ from decode_fp16's bit-preserving infnan path (o + 0x70000000), which would change reconstruction bits and sql2_recons_error even when numerically close.
- Reconstruction feeds faiss distance computations; any bit-level drift changes sq-accuracy results, so the vector path must be validated exhaustively before enabling.
- Gating on __riscv_v alone is wrong: vfwcvt.f.f.v requires zvfhmin; improper gating yields a compile error or illegal instruction.
- vsetvl and pipeline fill overhead wins only when d is not tiny; a threshold or dispatch must avoid regressing short vectors.

控制措施：
- Guard the vector path with __riscv_v && __riscv_zvfhmin and keep the current scalar decode_fp16 loop as the untouched fallback, following the extension-macro-guard rule.
- Choose source LMUL so VLMAX(e16) covers a practical batch (e.g. e16m1 = 8 elems on zvl128b) and the e32 destination register group fits, with a stable vtype (one vsetvl per iteration, no reconfig inside the loop).
- Follow the kernel-conventions widening rule for source/destination LMUL budgets.
- Gate enabling on the exhaustive 0x0000..0xFFFF bit-pattern comparison passing against decode_fp16.

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Add an RVV fast path to QuantizerFP16 decode_vector gated on __riscv_v && __riscv_zvfhmin: per iteration vsetvl_e16m* on remaining count, vle16 load of FP16 codes, vfwcvt.f.f.v widening FP16->FP32 (native IEEE handling of normals, denormals, ±0, ±Inf, NaN), vse32 store, with tail handling for d % VL and the existing scalar decode_fp16 loop retained as the fallback; add a masked/narrow repair only if bit-exact NaN-payload/sNaN parity with decode_fp16 is required.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
