# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-distance-013`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-distance-013`
- 派生来源快照用例：`sq-distance/013_faiss_scalar_quantizer_DCTemplate_faiss_scalar_quantizer_Quantizer8bitDirectSigned_faiss_SI_7a2d1cea`（候选 `candidate-42f3412bbfa105b82a1767fc307da3ad.patch`）
- 变更文件：`faiss/impl/scalar_quantizer/sq-rvv.cpp`

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
- Larger blocks change vfredosum association unless floating-point ordering contract permits; QT_8bit_direct_signed metric tolerance must be checked against the scalar reference
- Larger LMUL may spill if the allocator's live set grows beyond LMUL*peak_live_vectors<=32 (query float group, byte group, widened group, accumulators); m8 must not be assumed
- Autovec heuristics may not honor LMUL hints or may emit incompatible EMUL for the vsext.vf4 (8->32) widening step, forcing an intrinsic rewrite
- Longer bodies could increase frontend/I-cache pressure on the target core

控制措施：
- Enumerate the legal m1/m2/m4/m8 x unroll=1/2/4/8 frontier from objdump/live-range analysis and admit only spill-free candidates (inspect for vmv/stack spill-reload)
- Keep vsetvli a5,a4 min-on-remaining AVL so d not a multiple of VLMAX remains correct
- A/B within bench_scalar_quantizer_distance sq-distance comparing CPU time and IPC over the workload's d distribution

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Raise LMUL/register groups for the auto-vectorized compute_distance loop from e8/mf4 (4 code bytes per iteration on VLEN-128) to the legal spill-free frontier (e8/m1..m8 with matching e32 widening EMUL) via a targeted loop pragma/attribute or intrinsic rewrite of faiss/impl/scalar_quantizer/distance_computers.h:34-39, amortizing per-iteration vsetvl/reduction/branch overhead over 4-16x more elements and keeping VLEN-agnostic strip mining for the tail.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
