# AI 补丁审核报告（派生件）

> **本文件不是 Agent Debug 的 `opencode` 原件。**
> 上游（`yuansheng-patches` 快照与 `yuansheng-agent-debug`）没有为这条记录产出
> `opencode-patch-review`。本文件由快照中**真实存在**的机器/语义校验件与蓝图
> `craft_actions` 派生而来，仅用于补齐 mxnet 的 12 文件归档形态。
> 同目录另存有两个上游原件，可直接核对。

## 审核范围

- 审核对象：`pc-bp-faiss-sq-encode-011`（PatchCandidate）及其 `craft/patch.diff`
- 对应蓝图：`bp-faiss-sq-encode-011`
- 派生来源快照用例：`sq-encode/011_faiss_scalar_quantizer_Quantizer8bitDirect_faiss_SIMDLevel_0_encode_vector`（候选 `candidate-24ffabae714d0188f9e7d3e8b652377a.patch`）
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
- Special-value/out-of-range semantics: scalar fcvt.wu.s rtz saturates negative and >2^32-1 operands to 0 / 0xFFFFFFFF before byte truncation; a vector narrowing path (e.g. saturating vnclipu or unverified vnsrl) can change output bytes unless every lane matches the scalar byte output exactly.
- Short-vector overhead: the 8bit_direct quantizer encode runs per vector with a known small dimension in the sq-encode bench; vsetvl/dispatch and tail costs can erase or invert the gain for small d, so an unconditional vector path may regress.
- Store shape: the loop writes 1 byte per 4 bytes read, so after vectorization the residual bound may shift to L1 store/port throughput rather than ALU conversion; benefits must be verified on the target microarchitecture, not assumed from the lane count.

控制措施：
- Gate the RVV path on actual V availability consistent with the object's Tag_RISCV_arch v1p0/zve64d, with the scalar path as identical fallback.
- Benchmark across the realized d distribution with the existing sq-encode benchmark (benchmark_QT_8bit_direct_*) and retain a scalar path/tail below the measured crossover.
- Verify lane-exact byte output against the scalar reference before enabling the RVV path in production.

## 拟定改动（`blueprint.craft_actions.actions[0].proposal`）

Add an RVV precision-conversion fast path to Quantizer8bitDirect::encode_vector (or make the existing V-capable build actually emit vector code for this template): a dynamic-vsetvl strip-mined loop that unit-stride loads x with vle32, converts f32->u32 with vfcvt.rtz.xu.f.v (same round-toward-zero semantics as the current scalar fcvt.wu.s rtz), truncates to 8-bit via a narrowing shift vnsrl.wi shift 0 so the stored byte equals the (uint8_t)x[i] truncation of the conversion result, and stores with vse8; keep the scalar loop as the fallback and unchanged codec contract, with scalar tail handling for partial vl.

## 只读边界

本派生件未修改任何源码、蓝图、PatchPlan、`patch.diff` 或 PatchCandidate。
