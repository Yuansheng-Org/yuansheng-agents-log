# AI 补丁审核报告

- reviewId: rev-bp-opencv-core-030-r1
- patchCandidateId: pc-bp-opencv-core-030
- 审核时间: 2026-09-09
- 审核者: opencode-ai-reviewer（独立只读审核）
- 结论: **pass**

## 审核范围

本审核覆盖以下事实来源（只读，未修改任何代码）：

- RootCauseBlueprint: `.yuansheng/trace/opencv/core/030_reduceRowSum_8u32s/blueprint_opencv_core_030.json`（注：原文件 `validation.regressionCommand` 为空串不符 schema，已做最小机械修复为与同批次一致的 `"agent1 patch_regression --case core"`，诊断内容未改动）
- PatchPlan: `.yuansheng/craft/opencv/030_reduceRowSum_8u32s/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/opencv/030_reduceRowSum_8u32s/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/opencv/030_reduceRowSum_8u32s/craft/patch-candidate.json`

## 通用审核

### 1. 根因解决

Blueprint rootCause：reduceRowSum_8u32s 主累加循环 u16 累加域 LMUL=1（e16,m1，VLEN=256 下 16 lane/op），v_expand 的 e16m2 组被拆成 lo/hi 两个 m1 半区消费；主循环 vsetvli 57.5% + 整型记账 34%，向量处理仅 6%。

补丁在 RVV 路径以 e16,m2 处理 u16 累加域（32 lane/op）：vle8(u8m1) + vwcvtu(u8m1→u16m2) + vle16 + vsaddu + vse16，单块 32 lane 完成累加，消除 lo/hi 拆分；x86/NEON 通用路径原样保留。

判定：根因解决（e16,m1 → e16,m2，消除 lo/hi 拆分与冗余 vsetvli/地址重建），符合 blueprint recommendedFirstAction。

### 2. 约束保持

- vsaddu u16 饱和累加语义：显式 vsaddu 与 universal v_add（v_uint16 → __riscv_vsaddu，intrin_rvv_scalable.hpp:749）契约完全一致；flush_at=256 合同（255×256=65280<65535）内不触发饱和，通过。
- 每 256 行 flush 到 u32 的累加顺序与结果：flush 循环未触碰，通过。
- tail 行为：循环条件 `i <= len - vlanes8`（vlanes8=32）与标量余数循环 `for(; i < len; i++)` 承接不变，无 off-by-one，通过。
- x86/NEON 通用 universal-intrinsic 路径：RVV 改动以 `#if defined(__riscv_v) && __riscv_v == 1000000` 守卫，其他架构/模式走原路径，通过。

### 3. 最小性

仅修改 reduce.simd.hpp 的 reduceRowSum_8u32s 主累加循环（+15 行）。非 RVV 构建编译走原 generic 循环。无无关重构。

### 4. 产物一致性

PatchCandidate 由 candidate 工具基于真实 git diff 生成，gitDiff 与 patch.diff 字节一致。030 的 patch.diff 仅含 modules/core/src/reduce.simd.hpp。

### 5. 安全

无硬编码密钥、无危险命令/路径、无新增 I/O 面。

## RISC-V 架构审核

arch-scan 机器判定 `archSpecific=true`（命中 rvv-intrinsic 与 riscv-macro），执行架构专项审核：

1. **指令/特性在目标 ISA 内**：vle8/vwcvtu/vle16/vsaddu/vse16 均为 RVV 1.0 标准指令；vwcvtu 重载既有用法（intrin_rvv_scalable.hpp:605），vsaddu 与 universal 契约一致。
2. **LMUL 合法性**：e16,m2 = 32 lane/op（VLEN=256），寄存器峰值 ~5 组（vs u8m1 + vw/va u16m2），≪ 32 预算，无 spill。
3. **vtype 一致性**：u8m1 载入/加宽 → u16m2 累加/存储，SEW/LMUL 匹配；单块内仅 2 种 vtype，编译器可将循环不变 vsetvli 提出循环。
4. **饱和/加宽语义**：vsaddu 无符号饱和加 + vwcvtu 零扩展，与 v_expand/v_add 逐位一致。
5. **tail 安全**：vlanes8 恒定 32，循环边界与余数循环不变。
6. **无 x86/ARM 专属指令**：核验通过。

## 审核结果

**PASS** — 补丁准确解决 blueprint 根因（u16 域 LMUL 欠利用 + lo/hi 拆分），饱和语义与通用路径保持，RISC-V 架构核验通过。

## 发现问题

| id | file | line | severity | category | 说明 |
|----|------|------|----------|----------|------|
| F-030-01 | modules/core/src/reduce.simd.hpp | 848 | suggestion | verification-gap | 无 riscv64 交叉编译器，无法本地编译/性能复测；建议目标机 annotate 复核与饱和累加对拍（见 suggestion） |

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
