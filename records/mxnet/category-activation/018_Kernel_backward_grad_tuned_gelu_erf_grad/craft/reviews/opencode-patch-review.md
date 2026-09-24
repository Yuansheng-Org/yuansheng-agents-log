# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-activation-018`
- patchCandidateId: `pc-bp-mxnet-category-activation-018`
- 审核对象: `.yuansheng/craft/mxnet/018_Kernel_backward_grad_tuned_gelu_erf_grad/craft/patch.diff`
- 审核角色: opencode-cpp-reviewer（独立只读会话）
- 蓝图: `.yuansheng/trace/mxnet/category-activation/018_Kernel_backward_grad_tuned_gelu_erf_grad/blueprint_mxnet_category-activation_018.json`
- 审核时间: 2026-09-21T11:16:01Z

## 审核范围

本补丁改动 2 个文件，共 182 行新增、1 行删除：

| 文件 | 改动 |
|---|---|
| `src/operator/mshadow_op.h` | `erf_grad` 由 `MXNET_UNARY_MATH_OP` 宏改为显式 struct（通用模板保持 double 语义）+ `Map<float>` 显式特化（`2.0f` 系数）；新增受 `#if defined(__riscv) && defined(__riscv_v)` 保护的 `namespace gelu_erf_grad_rvv`（`exp_f32m1` 向量 exp、`gelu_erf_grad_block_f32m1`、`gelu_erf_grad_f32` 主循环+tail）；新增 `gelu_erf_grad_vec<xpu, DType>` 分派钩子（通用返回 false）及其 `<mshadow::cpu, float>` 特化 |
| `src/operator/leaky_relu-inl.h` | `case leakyrelu::kGELU_ERF` 在 `MXNET_ASSIGN_REQ_SWITCH` 之前尝试向量分派，仅 `kWriteTo`/`kWriteInplace` 且分派返回 true 时 `break`，否则原标量路径不变 |

审核覆盖：与蓝图根因的对应性、标量语义等价性、RVV 代码的编译与数值正确性、回退路径完整性、NaN/±Inf/±0/subnormal 行为、架构代码边界。

### 语义映射核对

- 标量链（blueprint `problem.summary` / diaglog Phase 2）：`out[i] = grad[i] · (output[i]/data[i] + 0.5f·data[i]·erf_grad(data[i]/√2)/√2)`，`erf_grad(t)=2/√π·exp(-t²)`。
- 分派调用 `Launch(s, N, gdata.dptr_, grad.dptr_, data.dptr_, output.dptr_)` 对应特化签名 `(out, g, a, b)`；块内 `va=data, vb=output, vg=grad`，`res = inner·vg`。参数顺序与标量 `Kernel<...backward_grad_tuned<gelu_erf_grad>...>::Launch(gdata, grad, data, output)` 一致。**通过**。
- 块内 `half_a=0.5f·a` → `term=half_a·eg` → `term/sqrt2` → `ba+term` → `·g`，运算顺序与标量表达式逐项一致。**通过**。

## RISC-V 架构审核

`arch-scan` 机器判定：`archSpecific: true`，命中规则 `riscv-macro`（`#if defined(__riscv) && defined(__riscv_v)`）与 `vlenb`（`__riscv_vsetvlmax_e32m1()`）。

- **架构代码边界**：全部 RVV 代码位于 `#if defined(__riscv) && defined(__riscv_v)` 之内（含 `#include <riscv_vector.h>`、`namespace gelu_erf_grad_rvv`、`gelu_erf_grad_vec<mshadow::cpu, float>` 特化）；非 RISC-V 或未启用 V 的目标只保留通用模板，`Launch` 返回 false 回退标量。**通过**。
- **intrinsic 编译验证**：将补丁中 `exp_f32m1`/`gelu_erf_grad_block_f32m1`/`gelu_erf_grad_f32` 逐字提取为独立 TU，使用环境内 `riscv64-linux-gnu-gcc`（GCC 15，`-march=rv64gcv -mabi=lp64d`）与 `clang-21 --target=riscv64-linux-gnu` 实编译，均 0 错误。所用 `__riscv_vfmv_v_f_f32m1`、`__riscv_vfcvt_x_f_v_i32m1`、`__riscv_vfcvt_f_x_v_f32m1`、`__riscv_vreinterpret_v_i32m1_f32m1`、`__riscv_vsll_vx_i32m1`、`__riscv_vfmerge`、`__riscv_vsetvlmax_e32m1` 等名称有效。**通过**。
- **向量化有效性**：`vle32`/`vfdiv`/`vfmul`/`vfadd`/`vfneg`/`vse32` + 自包含向量 exp，替换每元素 libm `expf@plt` 与 `2.0` double 往返；主循环 fixed-VL（`vsetvlmax`）+ runtime-VL tail，符合 RVV 惯例。**通过**。
- **无 x86/ARM 指令**：补丁仅含 RVV intrinsic 与标量 C++，未见 AVX/SSE/NEON/`__asm__`。**通过**。
- **无 VLEN 硬编码假设**：`vlmax` 由 `vsetvlmax_e32m1` 运行时探测，未假设 VLEN=128。**通过**。

### 数值与回退专项

- **向量 exp 范围归约**：`k = round_to_nearest(x·log2e)`，`r = x − k·ln2_hi − k·ln2_lo`，**两个 ln2 项均由同一整数 `k`（`kfl`）缩放**（patch.diff 中 `kfl` 同时用于 `ln2_hi` 与 `ln2_lo` 两次 `vfmul`），`|r| ≤ ln2/2`；Cephes 6 阶多项式 + 整数位构造的精确 `2^k` 缩放。系数与 Cephes `expf` 一致。**通过**。
- **float 字面量吸收**：`erf_grad::Map<float>` 返回 `2.0f / math::sqrt(PI) * math::exp(...)`；`PI` 为 `const float`，`math::sqrt(float)` 返回 float，故整条 float 数据流不再提升到 double，消除 `fcvt.d.s/fmul.d/fcvt.s.d`。与旧式 double 中间精度差异 ≤ 约 1 ULP，在 float 容差内。**通过**。
- **标量语义保持**：`erf_grad` 通用模板与宏展开逐字等价（`DType(2.0 / math::sqrt(PI) * math::exp(-(a * a)))`），double/其他 dtype 结果不变；`gelu_erf_grad`、`SQRT_2`、`PI` 均未改。**通过**。
- **分派只影响连续 float 启动**：`DType=float, xpu=cpu, RVV` 且 `kWriteTo/kWriteInplace` 才走向量路径；其他 dtype/req/target（含 `kAddTo/kNullOp`、GPU、非 RVV）通用模板返回 false，原 `MXNET_ASSIGN_REQ_SWITCH` 路径不变。**通过**。
- **NaN/±Inf/±0**：NaN 由 `vfmerge` 显式回填 NaN 且多项式本身传播 NaN；`-inf` 经 `x ≤ -87.33654475f` flush 为 0（与 expf(-inf)=+0 一致）；`a==0` 时 `t=0`、`x=-0`、`exp=1`，`b/a` 由 `vfdiv` 保持 IEEE ±Inf/NaN；向量路径中 `x=-t²≤0`，正向 overflow 分支不可达。**通过**。

## 审核结果

**reviewResult: pass**

无 critical / major 问题；向量化目标（消除每元素 `expf@plt` 与 float64 往返、启用 RVV 执行单元）在代码层面成立，标量回退完整。以下为 minor / suggestion 级改进项，不阻断交付。

## 发现问题

| id | 文件 | 级别 | 类别 | 摘要 |
|---|---|---|---|---|
| F-018-01 | src/operator/leaky_relu-inl.h | minor | performance | 向量快路径为纯串行循环，绕过了原 `LaunchTuned` 的 OpenMP 并行，可能在多核目标上造成回归 |
| F-018-02 | src/operator/mshadow_op.h | suggestion | numerical | 向量 exp 将 `x ≤ -87.33654475f` 一律 flush 为 0，未复现 libm 的 subnormal 输出 |
| F-018-03 | src/operator/leaky_relu-inl.h | suggestion | performance | 快路径无 N 阈值，短 tensor 可能被向量设置/tail 开销主导 |

详见 `opencode-patch-review.json` 的 `findings`。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 技术精度 | [PASS] | 逐条核对 patch.diff：参数顺序、运算顺序、Cephes 系数、范围归约（两 ln2 项同乘 k）、`2.0f` 常量吸收、保护宏边界均属实；intrinsic 经 GCC 15 / Clang 21 实编译验证 |
| 声明溯源 | [PASS] | 所有 finding 的 `evidence` 均引用 patch.diff 新增 hunk 原文或 blueprint 具体字段；无凭模型知识虚构的锚点 |
| 可解释性 | [PASS] | 从蓝图根因（libm expf + double 往返 + zero v*）到补丁改动（向量 exp / float 特化 / 分派）链条完整可复核 |
| 内部一致性 | [PASS] | blueprint / patch-plan / patch-candidate / patch.diff 的 id、文件路径、变更范围一致；candidate `archReviewWarning` 为空与 arch-scan `archSpecific:true` 相符 |
| 安全 | [PASS] | 无硬编码密钥、无未保护的特权操作；RVV 代码受架构宏保护，不影响其他平台；无 x86/ARM 专属指令 |

## 结论

补丁在语义、数值与回退设计上与蓝图根因对齐，RVV 代码边界正确且可编译，标量路径保持等价，未发现 critical/major 问题。**审核通过（pass）**，建议将 F-018-01（OpenMP 并行性）作为后续性能跟进项优先处理，F-018-02/F-018-03 作为数值与短张量基准的补充验证项。
