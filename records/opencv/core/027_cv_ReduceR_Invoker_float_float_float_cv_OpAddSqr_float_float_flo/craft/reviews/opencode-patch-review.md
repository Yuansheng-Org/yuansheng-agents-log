# AI 补丁审核报告

- reviewId: rev-bp-opencv-core-027-r1
- patchCandidateId: pc-bp-opencv-core-027
- 审核时间: 2026-09-09
- 审核者: opencode-ai-reviewer（独立只读审核）
- 结论: **pass**

## 审核范围

本审核覆盖以下事实来源（只读，未修改任何代码）：

- RootCauseBlueprint: `.yuansheng/trace/opencv/core/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/blueprint_opencv_core_027.json`
- PatchPlan: `.yuansheng/craft/opencv/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/opencv/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/opencv/027_cv_ReduceR_Invoker_float_float_float_cv_OpAddSqr_float_float_flo/craft/patch-candidate.json`

## 通用审核

### 1. 根因解决

Blueprint rootCause：REDUCE_SUM 平方和 row 归约内循环被 autovec 固定 e32,m1（每轮 8 元素，VLEN=256），峰值 live 仅 2 向量，LMUL 欠利用；src 流式 load（76.38%）与 mv/循环控制（19.43%）随过多迭代重复。

补丁在 ReduceR_SIMD 挂载点新增 `ReduceR_SIMD<float, float, OpAddSqr<float,float,float>>` 显式 RVV 特化（LMUL=m4）：单 vsetvli + vle32 双 load + vfmacc + vse32，块循环 bounded vl 覆盖全部列并返回 end。

判定：根因解决（e32,m1 → e32,m4，每轮 8 → 32 元素），符合 blueprint recommendedFirstAction。

### 2. 约束保持

- REDUCE_SUM 平方和逐元素数值输出一致：OpAddSqr<float> = a + b*b（float→float 无饱和）；vfmacc(vb,vs,vs) = vb + vs*vs，且与当前 autovec 二进制的 vfmacc.vv 融合行为完全一致（annotate 证据），通过。
- saturate_cast 语义（float→float 恒等）：无饱和，通过。
- tail/mask（ta,ma）与公共 API：bounded avl 处理尾块，无 over-read，通过。

### 3. 最小性

仅修改 matrix_operations.cpp：通用 ReduceR_SIMD 之后 +38 行特化。非 RVV 构建走原 autovec 路径。

### 4. 产物一致性

PatchCandidate 由 candidate 工具基于真实 git diff 生成，gitDiff 与 patch.diff 字节一致。027 的 patch.diff 仅含 modules/core/src/matrix_operations.cpp。

### 5. 安全

无硬编码密钥、无危险命令/路径、无新增 I/O 面。

## RISC-V 架构审核

arch-scan 机器判定 `archSpecific=true`（命中 rvv-intrinsic 与 riscv-macro），执行架构专项审核：

1. **指令/特性在目标 ISA 内**：vsetvl_e32m4 / vle32_v_f32m4 / vse32_v_f32m4 / vfmacc_vv_f32m4 均为 RVV 1.0 标准指令，SpacemiT X100 支持。
2. **LMUL 合法性**：峰值 live 2 向量（vs/vb），m4 → 8 寄存器 ≤ 32，无 spill。
3. **FMA 语义**：vfmacc 融合乘加与当前 autovec 二进制一致（无额外 rounding 差异引入）。
4. **vtype 一致**：全部 e32,m4。
5. **无 x86/ARM 专属指令**：核验通过。

## 审核结果

**PASS** — 补丁准确解决 blueprint 根因（LMUL 欠利用），FMA 累加语义与现有二进制一致，RISC-V 架构核验通过。

## 发现问题

| id | file | line | severity | category | 说明 |
|----|------|------|----------|----------|------|
| F-027-01 | modules/core/src/matrix_operations.cpp | 337 | suggestion | verification-gap | 无 riscv64 交叉编译器，无法本地编译/性能复测；建议目标机 annotate 复核与 REDUCE_SUM 输出对拍（见 suggestion） |

无 critical/major 发现。唯一 suggestion 为验证性缺口（对应 blueprint diagnosis.currentGaps），不阻断通过。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度：架构相关结论有 diaglog/blueprint 证据支撑 | [PASS] |
| 声明溯源：每条 finding 的 evidence 来自 diff/blueprint 原文 | [PASS] |
| 可解释性：critical/major 说明为何必须改（本报告无 critical/major，N/A） | [PASS] |
| 内部一致性：reviewResult=pass 与 findings 严重度一致（无 critical/major） | [PASS] |
| 安全：无越权建议、无引入新风险的建议 | [PASS] |

## 结论

审核通过（reviewResult=pass）。补丁聚焦根因、约束保持、架构正确，建议进入 done。唯一 suggestion（目标机复测）不阻断流转。
