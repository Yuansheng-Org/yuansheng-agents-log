# AI 补丁审核报告

## 审核范围

- Blueprint：`.yuansheng/trace/pytorch/ao_sparsifier/001_mul_kernel_vectorized_loop/blueprint_pytorch_ao_sparsifier_001.json`
- Diaglog：`.yuansheng/trace/pytorch/ao_sparsifier/001_mul_kernel_vectorized_loop/diaglog_pytorch_ao_sparsifier_001.md`
- PatchPlan：`.yuansheng/craft/pytorch/001_mul_kernel_vectorized_loop/craft/patch-plan.json`
- PatchCandidate：`.yuansheng/craft/pytorch/001_mul_kernel_vectorized_loop/craft/patch-candidate.json`
- patch.diff：`.yuansheng/craft/pytorch/001_mul_kernel_vectorized_loop/craft/patch.diff`
- 变更文件：`aten/src/ATen/cpu/vec/rvv/vec_float.h`（1 文件，+2/-1）

## RISC-V 架构审核

arch-scan 判定 `archSpecific=true`，命中规则 `riscv-macro`：

```text
+    vfloat32m2_t v = __riscv_vle32_v_f32m2(this->values, VFLOAT32_VL);
```

专项检查结论：

1. **vtype / VL 正确性**：`Vectorized<float>` 的类型合同为 LMUL=m2、SEW=32，`VFLOAT32_VL = CONFIG_VLMAX_BITS/32`，与 blueprint 中 `vsetivli zero,8,e32,m2` 自洽。`__riscv_vle32_v_f32m2` 与 `__riscv_vse32_v_f32m2` 均携带 `VFLOAT32_VL` / `count` 作为 `vl`，未引入 `vsetvl` 类型不匹配或 LMUL 混用。
2. **寄存器组 / 数据承载**：`store()` 由 `std::memcpy` 的 e8/m1 字节搬运循环改为一次 `vle32.v` + 一次 `vse32.v`，消除了 blueprint 证据中 `38fc228-38fc23a` 的 `vsetvli a5,a4,e8,m1` + `vle8.v/vse8.v` 降级循环；未改变 `values` 内存数组表示。
3. **语义等价**：原 `std::memcpy(ptr, values, count*sizeof(float))` 与 `vse32.v(ptr, v, count)` 均写出前 `count` 个 float 元素（`vse32` 的 `vl` 截断至 VLMAX）。当 `count > size()` 时 `memcpy` 本就越界读取 `values` 数组，因此不存在由本次改动引入的越界回归。
4. **对齐**：`vle32.v` 从 `this->values` 读取；该模式在本文件 `isnan()`/`abs()`/`operator==` 等处已广泛使用（同一 `values` 成员），本次改动未新增对齐风险，也未触碰 `RVV_SUPPORT_UNALIGN` 分支。
5. **ABI 稳定性**：未改动 `Vectorized<float>` 的成员类型与公共接口，仅改动函数体，符合 blueprint `constraints.mustPreserve` 对逐元素语义、`size()`=8、广播 `S>0`、尾块 `basic_loop` 与非对齐 `loadu` 的保持要求。

## 审核结果

- reviewResult：`pass`
- 是否消除根因直接证据：是。blueprint 最高占比行 `53.85 : 38fc238: add a3,a3,a5`（32-byte `std::memcpy` 降级循环）对应的 `store()` 实现被替换为单条 RVV `vse32.v`，Phase 5 预测的「字节搬运循环消失」侧已由代码直接兑现。
- 是否引入数值/API/语义变更：否。逐元素数值结果、dtype、公共 API、`values` 表示均未变。

## 发现问题

| id | 文件 | 行 | 严重度 | 类别 | 说明 |
|---|---|---|---|---|---|
| F1 | aten/src/ATen/cpu/vec/rvv/vec_float.h | 155 | suggestion | 性能残差 | 本次改动消除 `store` 的 e8/m1 字节搬运，但 `Vectorized<float>::values` 仍为内存数组，`vectorized_loop` 中其余栈 spill/reload（`ld a5,-560(s0)` 等）与 `dereference_vec_impl` out-of-line 调用仍在；彻底寄存器化属跨 RVV vec 层 ABI 改动，blueprint 亦标注需人工复核。 |

F1 只给出建议，不构成 critical/major：其指向的残差并非本次 blueprint 最高占比证据行（53.85% memcpy 循环）本身，且彻底修复需更大范围改动，超出「最小、语义保持」的补丁边界。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 文件锚点真实性 | PASS | `aten/src/ATen/cpu/vec/rvv/vec_float.h` 真实出现在 patch.diff 的 `diff --git`/`+++ b/` 头。 |
| 行号锚点真实性 | PASS | F1 行号 155 落在 hunk `@@ -152,7 +152,8 @@` 的 `-` 侧区间 [152,159] 内。 |
| 证据引用真实性 | PASS | 引用的 `std::memcpy(ptr, this->values, count * sizeof(float));`、`__riscv_vle32_v_f32m2`、`__riscv_vse32_v_f32m2` 均为 patch.diff 原文；blueprint 行号 `38fc238`/`38fc228` 均取自 diaglog。 |
| 结论可溯源性 | PASS | pass 结论绑定 blueprint `recommendedFirstAction` 与 `constraints.mustPreserve`，未外推工作负载级 Amdahl 收益。 |
| 模型知识越界 | PASS | 未引入 patch.diff/blueprint 之外的具体性能数字；RVV 语义仅作类型合同级判定。 |

## 结论

补丁以最小、语义保持的方式落在 blueprint `candidateFiles` 与 `recommendedFirstAction` 指定的 `aten/src/ATen/cpu/vec/rvv/vec_float.h::store()`，直接消除最高占比证据（32-byte `std::memcpy` 降级循环），未触碰 ABI/数值/公共接口，无 critical/major 问题。审核通过（pass）。F1 作为后续深化的建议保留。
