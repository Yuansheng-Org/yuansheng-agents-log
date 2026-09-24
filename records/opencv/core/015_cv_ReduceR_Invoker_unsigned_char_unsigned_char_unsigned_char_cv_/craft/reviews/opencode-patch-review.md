# AI 补丁审核报告

- reviewId: rev-bp-opencv-core-015-r1
- patchCandidateId: pc-bp-opencv-core-015
- 审核时间: 2026-09-09
- 审核者: opencode-ai-reviewer（独立只读审核）
- 结论: **pass**

## 审核范围

本审核覆盖以下事实来源（只读，未修改任何代码）：

- RootCauseBlueprint: `.yuansheng/trace/opencv/core/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/blueprint_opencv_core_015.json`
- PatchPlan: `.yuansheng/craft/opencv/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/opencv/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/opencv/015_cv_ReduceR_Invoker_unsigned_char_unsigned_char_unsigned_char_cv_/craft/patch-candidate.json`

## 通用审核

### 1. 根因解决

Blueprint rootCause：ReduceR_Invoker<uchar, OpMin<uchar>> 行向 min 归约主循环全 scalar CV_MIN_8U 查表（zero v*），主循环约 99% 函数内样本。

补丁在 ReduceR_SIMD 挂载点（matrix_operations.cpp，实际代码位置；blueprint 标注 arithm.cpp 为符号源文件归属误差）新增 `ReduceR_SIMD<uchar, uchar, OpMin<uchar>>` 的 RVV 特化：vsetvli + vle8 双 load + vminu.vv + vse8 写回，块循环以 bounded vl 覆盖全部列并返回 end。

判定：根因解决（CV_MIN_8U 查表 → vminu 向量 min），符合 blueprint recommendedFirstAction。

### 2. 约束保持

- min 结果逐元素 bit 一致：CV_MIN_8U（precomp.hpp:111 `(a) - CV_FAST_CAST_8U((a) - (b))`）语义 = uchar 无符号 min，vminu.vv 逐位等价，无 FP 语义，通过。
- 跨行 running min 的 seed/累积语义：buf 初始化与逐行调用不变；vminu 覆盖全部列（含 tail）后返回 end，调用方 scalar 循环跳过，running-min 状态续接等价，通过。
- buf/dst 写出与 range.start!=0 列区间：特化从 start 开始处理、到 end 结束，dst 写出不变，通过。
- 外循环（行迭代）与 buf 初始化不变，通过。

### 3. 最小性

仅修改 matrix_operations.cpp：通用 ReduceR_SIMD 之后 +27 行特化。非 RVV 构建编译该特化（守卫排除），走原 no-op + 标量路径。

### 4. 产物一致性

PatchCandidate 由 candidate 工具基于真实 git diff 生成，gitDiff 与 patch.diff 字节一致。015 的 patch.diff 仅含 modules/core/src/matrix_operations.cpp。

### 5. 安全

无硬编码密钥、无危险命令/路径、无新增 I/O 面。

## RISC-V 架构审核

arch-scan 机器判定 `archSpecific=true`（命中 riscv-macro），执行架构专项审核：

1. **指令/特性在目标 ISA 内**：vsetvl_e8m1 / vle8_v_u8m1 / vminu_vv_u8m1 / vse8_v_u8m1 均为 RVV 1.0 基础指令；vminu 既有用法（norm.cpp:436、common.hpp:47）。SpacemiT X100（RVV 1.0, VLEN=256）支持。
2. **vtype 配置一致**：全部 e8,m1（VLEN=256 下 vlmax=32 lane），无错乱。
3. **语义精确性**：vminu.vv 对 uchar 计算 min(a,b)，与 CV_MIN_8U 查表逐位一致；无符号饱和减法不引入差值符号问题。
4. **tail 安全**：`vl = vsetvl_e8m1(end - i)` bounded avl，最后一块覆盖剩余列，无 over-read。
5. **无 x86/ARM 专属指令**：核验通过。

## 审核结果

**PASS** — 补丁准确解决 blueprint 根因（CV_MIN_8U 查表 → vminu 向量 min），逐位语义保持，RISC-V 架构核验通过。

## 发现问题

| id | file | line | severity | category | 说明 |
|----|------|------|----------|----------|------|
| F-015-01 | modules/core/src/matrix_operations.cpp | 337 | suggestion | verification-gap | 无 riscv64 交叉编译器，无法本地编译/性能复测；建议目标机 annotate 复核与边界对拍（见 suggestion） |

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
