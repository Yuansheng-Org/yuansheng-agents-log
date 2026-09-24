# AI 补丁审核报告

- reviewId: rev-bp-opencv-core-026-r1
- patchCandidateId: pc-bp-opencv-core-026
- 审核时间: 2026-09-09
- 审核者: opencode-ai-reviewer（独立只读审核）
- 结论: **pass**

## 审核范围

本审核覆盖以下事实来源（只读，未修改任何代码）：

- RootCauseBlueprint: `.yuansheng/trace/opencv/core/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/blueprint_opencv_core_026.json`
- PatchPlan: `.yuansheng/craft/opencv/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/opencv/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/opencv/026_cv_cpu_baseline_cvt32f16s_unsigned_char_const_unsigned_long_unsi/craft/patch-candidate.json`

## 通用审核

### 1. 根因解决

Blueprint rootCause：convert.simd.hpp:156-157 的 `#if` 条件在 GCC 构建下排除 CV_SIMD_SCALABLE 向量路径（issues/26936 临时规避），cvt32f16s 热循环退化为逐元素 lrintf@plt + saturate_cast 标量转换（zero v*）。

补丁将该排除**收窄**：向量路径恢复为 `#if (CV_SIMD || CV_SIMD_SCALABLE)` + 编译期常量 `useSimd` 判定——GCC+RVV 下仅 `float→short`（DEF_CVT_FUNC 32f16s）实例启用向量路径，其余转换对维持原 #26936 规避。对应 blueprint recommendedFirstAction 的「收窄/解除排除」。

### 2. 约束保持

- float32→int16 的 cvRound/lrintf rounding 语义：由 universal intrinsic 的 v_store_pair_as（v_float32→short 饱和转换路径，vfcvt 按默认 FRM=RNE）承载，与 lrintf 默认语义一致，通过（目标机验证见 F-026-01）。
- saturate_cast<short> 饱和边界：universal intrinsic 的饱和窄化路径，通过。
- +0/-0、denormal、overflow/underflow、tie：依赖向量路径实现，目标机对拍（F-026-01）。
- 函数公共 API 与 CV_32F→CV_16S 结果：cvt_ 模板结构不变，通过。

### 3. 最小性（关键论证）

补丁**不解除** cvt_64f（line 180）与第三处（line 199）的 GCC 排除，也不启用其他 cvt_ 实例——仅 (float,short) 单一实例在 GCC+RVV 下进入向量路径。useSimd 为编译期常量（std::is_same），无运行时分支开销。回归面 = 恰好一个转换函数。

### 4. 产物一致性

PatchCandidate 由 candidate 工具基于真实 git diff 生成，gitDiff 与 patch.diff 字节一致。026 的 patch.diff 仅含 modules/core/src/convert.simd.hpp。

### 5. 安全

无硬编码密钥、无危险命令/路径、无新增 I/O 面。

## RISC-V 架构审核

arch-scan 机器判定 `archSpecific=true`（命中 riscv-macro），执行架构专项审核：

1. **指令/特性在目标 ISA 内**：补丁本身不新增 intrinsic，而是恢复 universal intrinsic 向量路径（其 RVV 实现 vle32/vfcvt.x.f.v/vnclip/vse16 等均为 RVV 1.0 标准指令，SpacemiT X100 支持）。
2. **vtype 一致性**：由 universal intrinsic 层保证（v_float32 → v_int16 饱和转换），无显式 vtype 操作。
3. **收窄正确性**：GCC+RVV 下 `useSimd = (std::is_same<_Ts,float> && std::is_same<_Td,short>)` 仅对 32f16s 为 true；clang/x86 行为不变。
4. **无新增 x86/ARM 指令**：核验通过。

## 审核结果

**PASS** — 补丁准确解决 blueprint 根因（GCC 下向量路径被 #26936 排除），以最小回归面（单实例）恢复 cvt32f16s 向量化，RISC-V 架构核验通过。

## 发现问题

| id | file | line | severity | category | 说明 |
|----|------|------|----------|----------|------|
| F-026-01 | modules/core/src/convert.simd.hpp | 156 | suggestion | verification-gap | 无 riscv64 交叉编译器，无法本地复现 issues/26936 缺陷范围；建议目标机重跑 test_core + perf core 并对拍 CV_32F→CV_16S 输出（见 suggestion） |

无 critical/major 发现。唯一 suggestion 为验证性缺口（对应 blueprint diagnosis.currentGaps：issues/26936 未本地复现），不阻断通过。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度：架构相关结论有 diaglog/blueprint 证据支撑 | [PASS] |
| 声明溯源：每条 finding 的 evidence 来自 diff/blueprint 原文 | [PASS] |
| 可解释性：critical/major 说明为何必须改（本报告无 critical/major，N/A） | [PASS] |
| 内部一致性：reviewResult=pass 与 findings 严重度一致（无 critical/major） | [PASS] |
| 安全：无越权建议、无引入新风险的建议 | [PASS] |

## 结论

审核通过（reviewResult=pass）。补丁聚焦根因、回归面最小、架构正确，建议进入 done。唯一 suggestion（目标机复测与 #26936 复核）不阻断流转。
