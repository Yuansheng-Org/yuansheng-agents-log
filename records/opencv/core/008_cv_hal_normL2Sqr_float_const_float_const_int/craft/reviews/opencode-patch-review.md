# AI 补丁审核报告

- reviewId: rev-bp-opencv-core-008-r1
- patchCandidateId: pc-bp-opencv-core-008
- 审核时间: 2026-09-09
- 审核者: opencode-ai-reviewer（独立只读审核）
- 结论: **pass**

## 审核范围

本审核覆盖以下事实来源（只读，未修改任何代码）：

- RootCauseBlueprint: `.yuansheng/trace/opencv/core/008_cv_hal_normL2Sqr_float_const_float_const_int/blueprint_opencv_core_008.json`
- PatchPlan: `.yuansheng/craft/opencv/008_cv_hal_normL2Sqr_float_const_float_const_int/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/opencv/008_cv_hal_normL2Sqr_float_const_float_const_int/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/opencv/008_cv_hal_normL2Sqr_float_const_float_const_int/craft/patch-candidate.json`

## 通用审核

### 1. 根因解决

Blueprint rootCause：normL2Sqr_ 的样本约 70.6% 集中在 compiler 生成的 residual tail loop（2a8d58 bge 34.42%、2a8d9c 回边 29.67%），tail 每迭代（≤8 元素）支付回边分支、3 次 vsetvli/vsetivli 与 vfmv.s.f/vfredosum.vs/vfmv.f.s 串行归约链；4×vl 主循环健康。

补丁在 `d = v_reduce_sum(v_d0)` 之前插入 `#if defined(__riscv_v) && __riscv_v == 1000000` 守卫的 bounded-avl tail：avl=r（r=n-j<vl）vle32 双 load + vfsub + tail-undisturbed vfmacc 并入 v_d0，`j = n` 消耗残余，最终单次 v_reduce_sum 完成水平归约。回边分支消除、vsetvli 次数降至每 tail 一次、串行抽取链消除，直接命中 blueprint recommendedFirstAction。

### 2. 约束保持

对照 mustPreserve 逐条核验：

- 函数语义与数值输出：输入顺序、accumulator 初始（vx_setzero_f32）与最终状态（v_reduce_sum）不变；tail 并入向量累加器不改变逐 lane 乘积计算（t*t），通过。
- 浮点归约顺序：tail 与主循环部分和经同一次 vfredusum 树归约，与主循环 vfmacc 重排同类（blueprint 明确认可该容差类），通过。
- 不得 over-read：`__riscv_vle32_v_f32m1(a + j, r)` 使用 avl=r 有界形式，r<n-j<vl，不越界，通过。

### 3. 最小性

仅修改 normL2Sqr_ 一个函数，+18 行。主循环、v_reduce_sum 调用、标量回退循环均未改动。

### 4. 产物一致性

PatchCandidate 由 candidate 工具基于真实 git diff 生成，gitDiff 与 patch.diff 字节一致（candidate 工具强制校验）。008 的 patch.diff 仅含 modules/core/src/norm.dispatch.cpp。

### 5. 安全

无硬编码密钥、无危险命令/路径、无新增 I/O 面。

## RISC-V 架构审核

arch-scan 机器判定 `archSpecific=true`（命中 rvv-intrinsic 与 riscv-macro 规则），执行架构专项审核：

1. **指令/特性在目标 ISA 内**：`__riscv_vle32_v_f32m1`、`__riscv_vfsub_vv_f32m1`、`__riscv_vfmacc_vv_f32m1_tu` 均为 RVV 1.0 标准 intrinsic（_tu = tail-undisturbed 变体），SpacemiT X100（RVV 1.0, VLEN=256）支持。类型 v_float32=vfloat32m1_t（intrin_rvv_scalable.hpp:54）与 e32,m1 匹配。
2. **vtype 配置与数据流一致**：vle32/vfsub/vfmacc 全部 e32,m1，与 v_d0 的 vtype 一致，无错乱。
3. **tail/mask policy**：核心正确性论证——vfmacc 使用 `_tu`（tail-undisturbed）变体，lane 0..r-1 累加 t*t，lane r..vlmax-1 保留主循环部分和；最终 v_reduce_sum（vfredusum）归约全部 vlmax lane，语义与"主循环部分和 + tail 和"一致。若误用默认 TA 变体，tail lane 可能被置 1 导致归约污染——补丁正确选择了 _tu。
4. **标量↔向量边界**：j = n 精确消耗 tail，下方标量循环 `for(; j<n; j++)` 不执行；非 RVV 构建（__riscv_v 未定义）走原标量路径，行为不变。
5. **无 x86/ARM 专属指令**：核验通过。

## 审核结果

**PASS** — 补丁准确解决 blueprint 根因（residual tail 固定成本主导），保持全部 mustPreserve 约束，tail-undisturbed vfmacc 语义正确，RISC-V 架构核验通过。

## 发现问题

| id | file | line | severity | category | 说明 |
|----|------|------|----------|----------|------|
| F-008-01 | modules/core/src/norm.dispatch.cpp | 180 | suggestion | verification-gap | 无 riscv64 交叉编译器，无法本地编译/性能复测；建议目标机 annotate 复核与 n 边界对拍（见 suggestion） |

无 critical/major 发现。唯一 suggestion 为验证性缺口（对应 blueprint diagnosis.currentGaps 的 implementation_shape_gap 与长度分布未实测），不阻断通过。

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
