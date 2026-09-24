# AI 补丁审核报告

- reviewId: rev-bp-opencv-core-002-r1
- patchCandidateId: pc-bp-opencv-core-002
- 审核时间: 2026-09-09
- 审核者: opencode-ai-reviewer（独立只读审核）
- 结论: **pass**

## 审核范围

本审核覆盖以下事实来源（只读，未修改任何代码）：

- RootCauseBlueprint: `.yuansheng/trace/opencv/core/002_cv_rvv_hal_core_dft/blueprint_opencv_core_002.json`（含 diagnosis 质量信号与 diaglog 证据链）
- PatchPlan: `.yuansheng/craft/opencv/002_cv_rvv_hal_core_dft/craft/patch-plan.json`
- patch.diff: `.yuansheng/craft/opencv/002_cv_rvv_hal_core_dft/craft/patch.diff`
- PatchCandidate: `.yuansheng/craft/opencv/002_cv_rvv_hal_core_dft/craft/patch-candidate.json`

## 通用审核

### 1. 根因解决

Blueprint rootCause 指出：f32 DFT 固定映射到 RVV_F32MF2（VLEN=256 下每 op 仅 4 lane，m1 为 8 lane），蝶形/重排循环迭代数与回边/vsetvli 开销翻倍。补丁将 `rvv<float>` 基类从 `RVV_F32MF2` 提升到 `RVV_F32M1`，segment intrinsic（vlseg/vlsseg/vsseg）同步改为 f32m1 变体，使每向量 op lane 数从 4 提升到 8，直接消除 LMUL 选型过保守这一根因（对应 Register-Group Utilization pattern，benefitUpperbound=0.1724）。判定：根因解决，且保持逐 lane FP 运算语义不变。

### 2. 约束保持

对照 blueprint `constraints.mustPreserve` 逐条核验：

- dft 的 nf/factors/scale/isInverse/noPermute 语义与输出布局：未触碰算法控制流，仅替换 intrinsic LMUL 变体，通过。
- 复数蝶形逐 lane 独立运算的数值结果：LMUL 改变不改变逐 lane 独立 FP 运算顺序与结果，通过。
- vloxei32 index 位宽语义（index=字节偏移/元素大小）：TabType/TabTypeF 未动，index 乘 sizeof(T)*2 逻辑未动，通过。
- RVV 1.0 ISA 与 CV_HAL_RVV_1P0_ENABLED 构建门：未改动，通过。

### 3. 最小性

仅修改 `hal/riscv-rvv/src/core/dxt.cpp` 单文件，共 4 处 hunk：rvv<float> 基类与 3 个 segment intrinsic、两个 IdxV 类型别名、dft() 内 IdxV 分派、odd-radix index 向量 setvlmax/vid 改用 IdxV。无无关重构、无格式噪音。

### 4. 产物一致性

PatchCandidate 由 `candidate` 工具基于真实 `git diff` 生成，gitDiff 与 patch.diff 字节一致（candidate 工具强制校验）。

### 5. 安全

无硬编码密钥、无危险命令/路径、无新增 I/O 面。

## RISC-V 架构审核

arch-scan 机器判定 `archSpecific=true`（命中 riscv-macro 规则，`__riscv_vlseg2e32_v_f32m1x2`），执行架构专项审核：

1. **指令/特性在目标 ISA 内**：全部新增 intrinsic 均为 RVV 1.0 标准 intrinsic；目标硬件 SpacemiT X100（RVV 1.0, VLEN=256, ISA 见 metadata.json）支持。同型 intrinsic 在代码库既有用法（warp.cpp:287 `__riscv_vlseg2e32_v_f32m1x2`、norm.cpp:123 `__riscv_vsetvlmax_e32m1()`、dotprod.cpp:25 `__riscv_vid_v_u32m1`），构成编译可行性证据。
2. **vsetvl/vtype 配置与数据流一致**：odd-radix 路径的 index 向量改为按 T 分派（IdxV）：float→RVV_U32M1（e32,m1），与新的 rvv<float>::setvl 一致；double→RVV_U32MF2（e32,mf2），保持原语义。vtype 不混乱。
3. **tail/mask policy、标量↔向量边界、寄存器组重叠**：未引入 masked op；vl 全部来自 setvl/setvlmax 显式推导；vlseg2e32 x2 为相邻寄存器组操作，无重叠风险。
4. **运行时 ISA 分发**：无改动。
5. **无 x86/ARM 专属指令**：核验通过。

关键架构论证（index LMUL 规则）：`LMUL_index = LMUL_data × EEW_index / SEW_data`。f32m1 数据 → 1×32/32 = m1（u32m1），f64m1 数据 → 1×32/64 = mf2（u32mf2）。这正是 IdxV 分派必须存在的原因——若统一改为 e32m1，double 分支的 vloxei32 重载签名（f64m1 数据要求 vuint32mf2_t index）将不匹配，导致编译失败。补丁正确处理了该约束。

## 审核结果

**PASS** — 补丁准确解决 blueprint 根因（f32 LMUL 选型过保守），保持全部 mustPreserve 约束，最小且一致，RISC-V 架构核验通过。

## 发现问题

| id | file | line | severity | category | 说明 |
|----|------|------|----------|----------|------|
| F-002-01 | hal/riscv-rvv/src/core/dxt.cpp | 469 | suggestion | verification-gap | 本环境无 riscv64 交叉编译器，无法本地编译/性能复测；建议目标机验证（见 suggestion） |

无 critical/major 发现。唯一 suggestion 为验证性缺口（对应 blueprint diagnosis.currentGaps 中已标注的 implementation_shape_gap），不阻断通过。

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
