# AI 补丁审核报告

- 审核对象：`PatchCandidate pc-bp-mxnet-opperf-category-convolution-003`
- 对应蓝图：`bp-mxnet-opperf-category-convolution-003`（`.yuansheng/trace/mxnet-opperf/category-convolution/003_ConvolutionOp_cpu_float_BackwardData/`）
- 补丁计划：`pp-bp-mxnet-opperf-category-convolution-003`
- 审核模式：独立只读审核（仅基于磁盘产物：`craft/patch.diff`、`craft/patch-candidate.json`、`craft/patch-plan.json`、蓝图、diaglog）
- 审核时间：2026-09-21T06:32:00Z

## 审核范围

本补丁仅改动一个文件：`src/operator/nn/im2col.h`（`changedFiles` = `["src/operator/nn/im2col.h"]`）。差异含两个 hunk：

1. `@@ -129,6 +129,53 @@`：新增内联模板 `col2im_accumulate_strided_row<DType>(src, dst, stride, n)`，实现 `dst[i*stride] += src[i]` 的 RMW 累加，RVV 分支下按 `stride==1` 走 unit-stride `vle32/vse32`，否则走 strided `vlse32/vsse32`，并保留标量回退。
2. `@@ -394,20 +441,43 @@`：重写 `col2im_cpu` 的 `output_col` 内层循环，按 `kernel_col` 预计算合法区间 `[o_start, o_end]`，对 `interior` 段调用上述 helper，`data_col`/`input_row` 推进逻辑保持不变。

审核未依赖写补丁时的记忆；对每条结论均回到 `patch.diff` 原文与蓝图字段取证。`git rev-parse HEAD` = `b84609d3fc73d20929c114eab95faaa56e6c5ede`，与蓝图 `source.commitHash` 一致。

## RISC-V 架构审核

`arch-scan` 结果：`archSpecific: true`（命中 `riscv-macro`），因此执行专项审核。

- **目标 ISA 一致性**：补丁使用的内建函数为 `__riscv_vsetvl_e32m4`、`__riscv_vle32_v_f32m4`、`__riscv_vse32_v_f32m4`、`__riscv_vlse32_v_f32m4`、`__riscv_vsse32_v_f32m4`、`__riscv_vfadd_vv_f32m4`，属 RVV 1.0 / Zve32f 的 e32 定点/FP32 向量指令。证据：diaglog Phase 1「Build ISA」原文含 `v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0`，硬件 ISA 含 `rv64imafdcv_..._zve32f_zve32x_zve64d_...`；蓝图 `source.targetHardware` = `SOPHGO SG2044 / XuanTie C920v2`。均在目标 ISA 范围内，未使用 x86/ARM 指令或非 RVV 1.0 的 `th.v*` 变体。
- **vsetvl/vtype 一致性**：所有 load/store/算术均使用 `e32,m4`，与 `__riscv_vsetvl_e32m4` 设置的 SEW/LMUL 一致（SEW=32，LMUL=4）。未硬编码 `vl`，`vl = min(AVL, VLMAX)`，AVL = `n - i`，因此对 VLEN=128（VLMAX m4 = 16）与任意 VLEN 均正确。
- **tail/mask 策略**：未使用 mask（无 `_m`/`vmerge`），以运行期 `vl` 精确覆盖主循环与 tail；`ta,ma` 默认尾策略下只 store `vl` 个元素，tail 元素不写内存，语义安全。无越界尾写。
- **寄存器组/重叠**：m4 为 4 寄存器组，主循环同时活跃 `vc`、`vd` 与 `vfadd` 结果（可复用），峰值 live ≈ 8/32 寄存器，无 spill 风险；跨 lane 地址在 `stride >= 1` 时两两不同，组内无重叠别名。
- **strided 语义正确性**：`stride>1` 路径使用 `vlse32`/`vsse32` 并传入字节步长 `stride*sizeof(float)`。审核中曾独立验证 GCC 内建 `vse32` 为 unit-stride、`vsse32` 才是 strided store，当前 diff 使用的是 `__riscv_vsse32_v_f32m4`（正确），已通过反汇编确认生成 `vlse32.v`/`vsse32.v`。
- **scalar 回退守卫**：RVV 代码位于 `#if MXNET_IM2COL_USE_RVV` 且 `std::is_same<DType,float>::value` 分支内，非 RISC-V / 无 V 构建走标量循环，行为不变。

## 审核结果

**pass**

补丁聚焦根因（`col2im_cpu` interior 标量 gather-scatter 未向量化），符合蓝图 `diagnosis.recommendedFirstAction`（按 `(kernel_row, kernel_col)` 预计算段边界、interior 向量化、宏守卫保留 scalar 回退）。所有 `constraints.mustPreserve` 均满足。

动态正确性验证（独立于写补丁过程重新执行）：

- 差分测试 harness 对比 HEAD 原始 `col2im_cpu` 与补丁后 `col2im_cpu`，位精确比较 `data_im`：
  - `riscv64-linux-gnu-g++ -O2 -march=rv64gcv`（RVV 路径）+ `qemu-riscv64`：`cases=1040577 fails=0`
  - `-march=rv64gc`（无 V，scalar 回退）+ `qemu-riscv64`：`cases=1040577 fails=0`
  - 宿主 x86 `g++ -O2`（非 RISC-V scalar 回退）：`cases=1040577 fails=0`
  - 覆盖 `kWriteTo`/`kAddTo`、`channels∈[1,4]`、`stride∈[1,4]`、`dilation∈[1,3]`、`pad∈[0,3]`、kernel∈[1,7]、小/大 shape 与四边 padding。
- 反汇编确认 `col2im_accumulate_strided_row` 生成 `vsetvli e32,m4` / `vle32.v` / `vfadd.vv` / `vse32.v`（stride=1）与 `vlse32.v` / `vsse32.v`（stride>1）。

## 发现问题

- **F1（suggestion / code-quality）**：`col2im_accumulate_strided_row` 使用 `if (std::is_same<DType, float>::value)` 运行期常量分支而非 `if constexpr`；对非 `float` 的 `DType`，RVV 分支体仍会被实例化（死代码，但可编译，因 `reinterpret_cast` 对任意对象指针合法）。证据：`patch.diff` 原文 `+  if (std::is_same<DType, float>::value) {`。建议：可改为 `if constexpr` 消除死代码；但同文件既有 `im2col_copy_strided_row` 使用相同惯用法，为一致性也可保留。不影响正确性。
- **F2（suggestion / verification）**：本环境无法在 SG2044/C920v2 实机复采 `perf`，故仅完成功能等价与指令生成验证，未实测 `_BackwardData` 耗时下降。证据：蓝图 `diagnosis.recommendedVerification` 字段原文「在 SG2044/C920v2 上重新运行 mxnet-opperf category-convolution 基准并对 _BackwardData perf record/annotate」。建议：合入后在目标机运行 `agent1 patch_regression --case category-convolution` 并复采确认 interior 段 `bgeu`/`fsw` 份额下降、出现 `vle32/vfadd.vv/vse32`。此为环境限制下的残余风险，非补丁缺陷。

无 critical / major 问题。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 技术精度 | PASS | ISA 判定（RVV 1.0 / Zve32f / VLEN=128）、`vsse32` vs `vse32` 区分、区间边界推导均与 diaglog/反汇编/差分测试实测一致，无凭模型知识编造。 |
| 声明溯源 | PASS | 所有结论锚定 `patch.diff` 原文行或蓝图字段（`targetHardware`、`commitHash`、`recommendedVerification`）；测试数字来自本会话真实执行的 qemu/x86 输出。 |
| 可解释性 | PASS | 根因、修复方向、段拆分公式、`data_col` 推进不变量、FP 累加语义保持均可逐步解释。 |
| 内部一致性 | PASS | `patchCandidateId`/`patchPlanId` 与 candidate/plan 一致；`changedFiles` 与 diff 文件一致；审核结果 pass 且无 critical/major。 |
| 安全 | PASS | 无越界指针构造（padding 仅推进 `data_col`）、无越界尾写（运行期 `vl`）、无硬编码 `vl`、无 secrets、无公共 API/签名改动。 |

## 结论

补丁正确、聚焦、可解释，RISC-V 架构专项审核通过，动态差分验证位精确通过（RVV / 标量 / x86 三条路径共 1,040,577 用例 0 失败）。`reviewResult = pass`。残余风险仅为 F2 所述：目标机性能复采需在 SG2044 实机完成。
