# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-loss-002`
- patchCandidateId: `pc-bp-mxnet-category-loss-002`
- blueprintId: `bp-mxnet-category-loss-002`
- reviewer: `opencode-cpp-reviewer`
- reviewedAt: `2026-09-21T14:08:22Z`
- patch.diff: `.yuansheng/craft/mxnet/002_mxnet_warpctc_CpuCTC_float_log_softmax/craft/patch.diff`
- changed file: `3rdparty/ctc_include/detail/cpu_ctc.h`（单文件，+125 行）

## 审核范围

本次审核为独立只读审核，覆盖：

1. **结构门禁**：`patch-candidate.json` 通过 `yuansheng.patch_candidate.v1` 校验；`patch.diff` 与工作区真实 `git diff` 完全一致（忽略 `index` 行后逐字节相同）；`patch.diff` 仅含项目源码补丁，不含 `.yuansheng/` 协议产物。
2. **根因对齐**：补丁目标为 `mxnet_warpctc::CpuCTC<float>::log_softmax`，与 blueprint `hotspotEvidence[0].hotspotFunction`（`mxnet_warpctc::CpuCTC<float>::log_softmax(float const*, float*, int const*) [clone ._omp_fn.0]`）及 `recommendedFirstAction`（RVV 三阶段改写）一致。
3. **架构专项**：`node tools/yuansheng-craft-tools.js arch-scan patch.diff` 返回 `archSpecific: true`，命中 `riscv-macro`、`vlenb`、`rvv-intrinsic` 三条规则。
4. **语义合同**：逐项核对 `constraints.mustPreserve` 的 LogSoftmax 数值语义、特殊值行为与 `CpuCTC<float>` 接口不变性。
5. **幻觉锚点**：所有 finding 的 evidence 均引用 `patch.diff` 中真实 hunk 原文或 blueprint 具体字段。

审核对象为**未编译运行的静态源码补丁**；未执行构建/回归，故不声明运行时收益，仅声明结构正确性与语义推断，限制见下文。

## RISC-V 架构审核

- **架构相关性**：`arch-scan` → `archSpecific: true`。补丁新增 `#if defined(__riscv) && defined(__riscv_v)` 守卫与 `<riscv_vector.h>` 包含，使用 RVV 1.0 universal intrinsics，无 x86/ARM/其他架构指令（已全文检索确认无 `_mm`/`NEON`/`vaddq` 等）。
- **三阶段向量化**：目标函数的三阶段均已 RVV 化——
  - 阶段 1（行 max）：`vle32` + `vfredmax.vs`，标量 seed 分块归约，尾部用运行时 `vsetvl`；
  - 阶段 2（`denom = Σ exp(a-max)`）：`vle32` → `vfsub.vf(max)` → 向量 `exp_f32` → `vfredusum.vs`，标量 seed 分块归约；
  - 阶段 3（输出）：标量 `std::log(denom)` 仅调用一次，`act - max - log(denom)` 用 `vfsub.vf` + `vse32` 写回。
- **主循环处理**：fixed-VL 主循环（`i + vlmax <= n`，`vlmax = vsetvlmax_e32m4()`）处理整块，runtime-VL tail（`vsetvl_e32m4(n - i)`）处理余项，任意 `alphabet_size` 均有覆盖；`alphabet_size <= 0` 提前返回。
- **向量 exp 归约正确性（重点核对项）**：
  - `k = round(x·log2e)`（`vfcvt.x.f.v` + `vfcvt.f.x.v`）；`ln2` 拆分为 `ln2_hi = 0.693359375` 与 `ln2_lo = -2.1219444005469058277e-4`；
  - 范围归约 `r = x − k·ln2_hi − k·ln2_lo`，**两个 ln2 分量均由整数 k（`kf`）缩放后相减**，符合任务核对要求，`r` 收敛于 `[−ln2/2, ln2/2]`；
  - `exp(r)` 为 6 阶级 Horner 多项式（1/720…1/2…1，展开即 `1 + r + r²/2! + … + r⁶/6!`），`2^k` 经 IEEE-754 指数字段 `(k+127)<<23` 还原，无溢出（clamp 后 `k ∈ [−126, 127]`，`k+127 ∈ [1, 254]`）；
  - `x` 夹到 `[−87.33654475, 88.0]`，覆盖下溢尾与指数域安全。
- **分发约束**：`if constexpr (std::is_same<ProbT, float>::value)` 仅在 `ProbT == float` 时走向量路径并 `continue`，其余类型保持原标量三循环不变（`#include <type_traits>` 已补）。
- **标量 fallback 保留**：`#if` 之外的原标量实现完整保留，非 RISC-V 或未启用 V 的构建行为完全不变。
- **OMP 分片不变**：向量调用位于既有 `#pragma omp parallel for` 内层，未改变并行分片结构。
- **架构专项结论**：通过。无 x86/ARM 指令，RVV 使用形态正确，float 专属分发正确，标量回退完好。

## 审核结果

- **reviewResult: pass**
- **archReview.status: passed**
- 无 critical / major 问题；发现 4 条 minor/suggestion 级观察项（特殊值传播、frm 依赖、归约串行/多趟访存、exp 近似与 libm 的差异），均不阻断合并。
- 三阶段向量化、双 ln2 归约、单次 `log(denom)`、fixed-VL main + runtime-VL tail、float 专属 `if constexpr`、标量 fallback、无 x86/ARM 指令——全部核对通过。

## 发现问题

| id | 级别 | 类别 | 位置 | 摘要 |
|---|---|---|---|---|
| F1 | minor | correctness / fp-special-values | `3rdparty/ctc_include/detail/cpu_ctc.h` | NaN/±Inf 传播语义相对标量路径发生改变 |
| F2 | suggestion | numerical-robustness | `3rdparty/ctc_include/detail/cpu_ctc.h` | `k` 依赖动态舍入模式（frm），非 RNE 时多项式误差上界增大 |
| F3 | suggestion | performance | `3rdparty/ctc_include/detail/cpu_ctc.h` | 归约 seed 循环携带、阶段 2/3 重复访存，长行难以完全隐藏依赖链 |
| F4 | suggestion | numerical-accuracy | `3rdparty/ctc_include/detail/cpu_ctc.h` | 向量 exp 为多项式近似，与 libm `expf` 非逐位一致 |

详细说明：

- **F1（minor）**：`exp_f32` 中 `vfmax_vf(x, -87.33654475f)` 与 `vfredmax` 遵循 maxNum 语义，NaN lane 会被替换为 `-87.33654`（而非标量 `expf(NaN)=NaN`）。标量路径下任一 NaN 经 `denom += expf(NaN)` 使整行输出均为 NaN；向量路径下仅该 NaN 元素输出为 NaN，其余元素有限。`mustPreserve` 明示需保持 NaN/±Inf/±0/denom=0 行为，diaglog Phase 4 亦将 NaN/Inf 列入 correctness contract。属边界语义差异，非主流程缺陷，标记 minor。
- **F2（suggestion）**：`vfcvt_x_f_v_i32m4` 未显式指定舍入模式，使用动态 frm。默认 RNE 下 `r ∈ [−ln2/2, ln2/2]`、误差 ~1e-7；若调用方遗留非 RNE 的 frm，`r` 可扩至约 `ln2`，6 阶多项式相对误差升至 ~1e-5 量级。
- **F3（suggestion）**：跨 chunk 归约以单一标量 seed 循环携带，C920v2（OoO）上长行可能无法充分隐藏归约依赖链；阶段 2/3 各自重新 `vle32` 加载 activations，未复用 exp 中间值。属性能提升空间，非正确性问题。
- **F4（suggestion）**：向量 exp 相对误差约 1e-7（补丁注释自述，实测最大绝对误差 ~4e-6），与 libm `expf` 非逐位一致；`vfredusum` 亦改变求和结合顺序。建议在数值回归（LogSoftmax consistency）中持续约束漂移。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 技术精度 | [PASS] | 三阶段归约、双 ln2 缩放、`(k+127)<<23`、Horner 系数与 tail 处理均逐条对照 `patch.diff` 原文核实，无凭记忆推断的断言；RVV intrinsic 名称与语义在 1.0 语义下自洽。 |
| 声明溯源 | [PASS] | 每条 finding 的 evidence 均引自 `patch.diff` 真实新增行或 blueprint 字段（`constraints.mustPreserve`、`hotspotEvidence`），无外部不可溯源声明。 |
| 可解释性 | [PASS] | 每条 finding 给出触发条件、影响范围与可执行建议；对未运行构建/回归的边界做了显式披露。 |
| 内部一致性 | [PASS] | reviewResult=pass 与 findings 全部为 minor/suggestion 一致；archReview.status=passed 与 arch-scan 结果及逐项核对一致；无自相矛盾结论。 |
| 安全 | [PASS] | 补丁无硬编码密钥、无外部输入信任、无未定义行为引入（除已披露的 NaN 边界）；未改变内存边界（`alphabet_size` 守卫 + tail 覆盖），无越界读写。 |

## 结论

补丁在 `CpuCTC<float>::log_softmax` 上按 RVV 1.0 正确实现了三阶段 LogSoftmax 向量化：行 max（`vfredmax` 分块）、`exp` 和归一（`vfredusum` 分块）、输出（单次标量 `log(denom)` + `vse32`）；范围归约对 `ln2_hi`/`ln2_lo` **两个分量均按 k 缩放**，fixed-VL 主循环 + runtime-VL tail 覆盖任意行长，`if constexpr` 仅对 `float` 分发，其他类型与标量回退保持原样，且无 x86/ARM 指令。根因（RVV Normalization Kernels，三环全标量、零 `v*`）被直接针对，与 blueprint `recommendedFirstAction` 对齐。

无 genuine critical/major 问题，**审核通过（pass）**。F1–F4 为 minor/suggestion 级改进建议，可在后续迭代处理，不阻断当前补丁。建议后续以真实构建 + `category-loss` 回归 + `log_softmax` 重新 annotate 验证 `vsetvli/vle32/vfredmax/vfredusum/vse32` 出现及 `flt.s/beqz/expf@plt/logf@plt/fsw` 样本下降，并补充 NaN/±Inf 特殊值数值回归以闭合 F1。

**限制披露**：本次为静态只读审核，未执行编译、未运行测试、未采集性能数据；对运行时收益率与 bit 级数值等价性不作保证，相关结论以数值回归与 profile 复测为准。
