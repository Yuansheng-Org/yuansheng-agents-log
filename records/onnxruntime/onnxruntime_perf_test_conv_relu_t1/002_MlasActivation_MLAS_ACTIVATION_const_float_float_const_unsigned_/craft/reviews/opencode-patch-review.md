# AI 补丁审核报告

## 审核范围

- **blueprint**: `.yuansheng/trace/onnxruntime/onnxruntime_perf_test_conv_relu_t1/002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_/blueprint_onnxruntime_onnxruntime_perf_test_conv_relu_t1_002.json`
- **patch-plan**: `.yuansheng/craft/onnxruntime/002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_/craft/patch-plan.json`
- **patch.diff**: `.yuansheng/craft/onnxruntime/002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_/craft/patch.diff`
- **patch-candidate**: `.yuansheng/craft/onnxruntime/002_MlasActivation_MLAS_ACTIVATION_const_float_float_const_unsigned_/craft/patch-candidate.json`
- 目标硬件：SpacemiT X100 (K3)，RVV 1.0 VLEN=256；热点函数 `MlasActivation`，rank 002

## 根因摘要（来自 blueprint）

MlasActivationKernel 主循环被编译为固定 `vsetivli zero,4,e32,m1`（每轮仅 4 个 float），在 VLEN=256 上只利用 4/8 lane 半宽，98.95% 函数内样本落在 loop induction 指令上。recommendedFirstAction：为 RVV 构建增加 runtime-VL 路径（`__riscv_vsetvl` m1 满宽 VLMAX=8 lanes），Relu 等 lane-independent 激活可直接单指令形式；非 RVV 路径保持不变。

## RISC-V 架构审核

- `arch-scan` 判定 `archSpecific: true`，执行架构专项审核。
- 补丁为 `MlasActivationKernel` 主循环新增 RVV runtime-VL 路径（`#if defined(MLAS_TARGET_RISCV64) && defined(MLAS_USE_RVV) && defined(__riscv_v)`），并新增 `MLAS_BIAS_ADDITION::AddRvv` 与各 `MLAS_ACTIVATION_FUNCTION::ActivateRvv`：
  1. **指令/特性在目标硬件 ISA 内**：目标硬件 SpacemiT X100 支持 RVV 1.0（`__riscv_v` 在 `-march=rv64gcv` 下启用）；补丁使用标准 RVV intrinsic（`vsetvl_e32m1`/`vle32`/`vse32`/`vfadd`/`vfmul`/`vfmacc`/`vmerge`/`vmfgt`），VLEN=256 时 e32m1 VLMAX=8 lanes，与 blueprint 建议一致。✅
  2. **vsetvl/vtype 一致性**：`vl` 由 `__riscv_vsetvl_e32m1(n)` 每轮动态获取并在循环末尾更新，装载/存储/计算全部使用同一 `vl`；`while (n >= vl && vl != 0)` 避免 n=0 时 `vl=0` 的死循环。✅
  3. **NaN 语义保持**：各 `ActivateRvv` 用比较选择（`vmfgt`+`vmerge`）复现通用 fallback 的 NaN round-trip 语义（ReLU max(0,NaN)、LeakyReLU NaN→alpha*x 分支、Clip/HardSigmoid/HardSwish NaN 透传），与 `constraints.mustPreserve` 的"激活输出逐元素数值一致（含 NaN 语义）"一致。✅
  4. **非 RVV 路径不变**：RVV 路径整体包在 `#if ... && defined(__riscv_v)` 内，非 RVV 构建（NEON/SSE/LSX）仍走原固定 4-float 路径；cmake 仅在 `HAS_RISCV64_RVV` 通过时为 `activate.cpp` 追加 `-march=rv64gcv -mabi=lp64d`，不影响其他平台。✅
  5. **未混入 x86/ARM 专属指令**：全部为 RVV intrinsic。✅

## 通用审核清单

1. **根因解决**：主循环由固定 4-lane 块改为 runtime-VL（VLEN=256 时 8 lanes/轮），直接消除"固定 vsetivli zero,4,e32,m1 半宽利用"根因，loop induction 开销减半。✅
2. **约束保持**：公共 API 签名未变；逐元素数值与 NaN 语义保持一致；非 RVV 路径原样保留。✅
3. **最小性**：仅新增 RVV 条件编译分支与方法，无无关重构。✅
4. **产物一致性**：PatchCandidate.gitDiff 与 patch.diff 一致；changedFiles 仅含 activate.cpp 与 cmake。✅
5. **安全**：无硬编码密钥、无危险命令/路径。✅

## 审核结果

**reviewResult: pass**，findings 为空。

## 发现问题

无。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度 | [PASS] RVV 语义结论均基于 blueprint 证据（vsetivli zero,4,e32,m1、98.95% induction 样本）与补丁原文 |
| 声明溯源 | [PASS] 无 findings，全部结论来自 diff/blueprint 原文 |
| 可解释性 | [PASS] 无 critical/major 需要解释 |
| 内部一致性 | [PASS] reviewResult=pass 与 findings 空一致 |
| 安全 | [PASS] 未引入越权建议或新风险 |

## 结论

补丁准确、最小、架构正确（runtime-VL 化 + NaN 语义保持），通过全部审核维度，流转至终态 done。
