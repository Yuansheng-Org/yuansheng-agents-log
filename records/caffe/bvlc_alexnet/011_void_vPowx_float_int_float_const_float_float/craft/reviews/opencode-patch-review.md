# AI 补丁审核报告

- reviewId：pr-caffe-bvlc-alexnet-011-002
- patchCandidateId：pc-bp-caffe-bvlc-alexnet-011
- blueprintId：bp-caffe-bvlc-alexnet-011
- 审核对象：`.yuansheng/craft/caffe/011_void_vPowx_float_int_float_const_float_float/craft/patch.diff`
- 变更文件：`include/caffe/util/mkl_alternate.hpp`（+162/-1）
- 审核时间：2026-09-21T06:56:31.000Z
- 审核者：opencode independent read-only AI patch reviewer（只读，未修改任何源码或补丁产物）
- 轮次：第 2 轮（上一轮 reviewResult=needs-changes，本轮复核 F-001/F-002/F-005 修订）

## 审核范围

只读核验以下冻结产物：RootCauseBlueprint（`.yuansheng/trace/caffe/bvlc_alexnet/011_void_vPowx_float_int_float_const_float_float/blueprint_caffe_bvlc_alexnet_011.json`）、diaglog、`craft/patch-plan.json`、`craft/patch.diff`、`craft/patch-candidate.json`，以及当前 `include/caffe/util/mkl_alternate.hpp`、原始头文件（`git show HEAD:include/caffe/util/mkl_alternate.hpp`）与调用链（`src/caffe/util/math_functions.cpp`、`src/caffe/layers/lrn_layer.cpp`、`src/caffe/layers/power_layer.cpp`）。

审核维度：

1. 通用质量：是否解决蓝图根因（标量逐元素 libm `powf` 调用循环 → RVV 向量路径，含 bvlc_alexnet LRN 指数 b=-0.75）；是否保持 `blueprint.constraints.mustPreserve`（IEEE powf 语义与 Caffe 重现性、CHECK 守卫、公共 API、非 V 回退）；精度/域合同；精确快路径；in-place 别名安全；范围合理性；产物一致性；安全性。
2. RISC-V 架构专项（目标 SpacemiT X100 / K3，RVV 1.0，VLEN=256，vlenb=32，build Tag 含 v1p0/zvl128b/zve64d）。
3. 防幻觉锚点机器校验。

机器校验：`arch-scan` 返回 `archSpecific: true`（命中 `riscv-macro: '+#if defined(__riscv_v) && defined(__riscv_v_intrinsic)'`、`vlenb: '+  const size_t vlmax = __riscv_vsetvlmax_e32m4();'`），故执行 RISC-V 架构审核。`PatchCandidate.gitDiff` 与 `patch.diff` 逐字节相等（8398 bytes），`changedFiles` 为 `["include/caffe/util/mkl_alternate.hpp"]`，产物一致。

上一轮 findings 复核：

- F-001（major，vdPowx 形参被改为 `const double b`）：**已修复**。当前 diff 为 `+inline void vdPowx(\n+    const int n, const double* a, const float b, double* y) {`，与原始宏生成签名（`git show HEAD` 中 `const int n, const double* a, const float b, double* y`）一致；`caffe_powx<double>` 的 `const double b` 仍按原语义截断为 float。
- F-002（suggestion，exp 级数注释）：**已修复**。注释现为 `+  // exp(z) = 1 + z + z^2/2! + ... + z^10/10!`，与 Horner 首项 `1.0 / 3628800.0`（=1/10!）及 11 步累加一致。
- F-005（suggestion，编译期门）：**已修复**。门现为 `+#if defined(__riscv_v) && defined(__riscv_v_intrinsic)`。

## RISC-V 架构审核

- SEW/LMUL 与寄存器组：**通过**。一般路径 f32 `e32m2` → `vfwcvt_f_f_v_f64m4` 加宽为 f64 `e64m4` → 运算 → `vfncvt_f_f_w_f32m2` 收窄回 `e32m2`；加宽/收窄前后元素数一致（`e32m2 = 2*VLEN/32`，`e64m4 = 4*VLEN/64`），组宽匹配。精确快路径 `rvv_map_f32` 用 `e32m4`。
- vsetvl：**通过**。全为运行时动态（`vsetvlmax_e32m4`、`vsetvl_e32m2`、`vsetvl_e32m4`），硬件 VLEN=256 可被取满，不受 build `zvl128b` 固定下限约束。
- ISA/扩展前提：**通过**。`metadata.json` 的 `Tag_RISCV_arch` 含 `v1p0`、`zve32f`、`zve64d`、`zve64f`；f64 向量加宽/收窄所需 `zve64d` 存在。
- mask/tail 策略：**通过**。一般路径按 `vl = vsetvl_e32m2(n-i)` 动态分块，`rvv_map_f32` 主循环用 `vlmax`、尾块用运行时 VL；无 mask，仅写回有效 lane，无越界写。
- 标量↔向量边界：**通过**。仅 `b` 为标量广播（`vfmul_vf`/`vfmv_v_f`）；域检查为循环外一次标量预扫。
- 编译期门与回退：**通过**。`#if defined(__riscv_v) && defined(__riscv_v_intrinsic)`；门不成立时仅有通用模板与 `vsPowx`/`vdPowx` 标量包装，非 V 目标正确回退。
- 无 x86/ARM 指令：**通过**。全为 RVV intrinsic，无其他架构指令。
- 寄存器组溢出：implementer 上下文称无向量溢出；本只读审核无法独立复现反汇编，标记 **cannot verify**（见 F-001），不据此下结论。

`archReview.status = passed`。

## 审核结果

`reviewResult = pass`。

- **根因修复正确**：`vPowx<float>` 由宏展开改为显式特化，安全域内走 `exp2(b*log2(x))` 的 f64 向量管线，消除逐元素 `powf@plt` 调用循环，命中蓝图 primary pattern（RVV Elementwise Activation Kernels）。bvlc_alexnet LRN 调用 `caffe_powx<Dtype>(scale_.count(), scale_data, -beta_, top_data)`（`src/caffe/layers/lrn_layer.cpp:150`），`scale_data = 1 + alpha/N^2·s >= 1`（正规格化）且 `b = -0.75`，落在安全域内 → 实际命中向量路径。
- **数值语义保持**：`rvv_powx_domain_ok` 要求 `b ∈ [-1,1]` 且逐元素指数域 `ef != 0 && ef != 0xFF` 且符号位为 0（全部正规格化浮点）；否则执行 `for (int i = 0; i < n; ++i) { y[i] = pow(a[i], b); }`，与原始标量 libm powf 逐位一致，NaN/±Inf/±0/次正规/负数语义不被触碰。
- **精确快路径受限正确**：`b==1.0f`（恒等拷贝）、`b==0.0f`（常量 1，且 `-0.0f` 亦命中）均在 `rvv_powx_domain_ok` 为真后才启用；域外输入回退标量，结果同样正确。
- **公共 API 保持**：`vsPowx`/`vdPowx`/通用 `vPowx<Dtype>` 的形参类型与顺序同原始宏展开（`vdPowx` 为 `const float b`）；`CHECK_GT(n,0); CHECK(a); CHECK(y)` 在通用模板与 float 特化中均保留。
- **别名安全**：`power_layer.cpp:39` 的 `caffe_powx(count, top_data, power_, top_data)` 与 LRN 的 `caffe_powx(..., scale_data, ..., top_data)` 均可能 a==y；`rvv_map_f32` 与 `rvv_powx_pos_chunk` 均在同一 chunk 内先整块 load 再整块 store，逐 lane 独立，无写后读冲突。
- **范围聚焦**：仅改动 `include/caffe/util/mkl_alternate.hpp`；未改其他 VSL 函数与 double 向量路径。
- **f64 管线逻辑自洽**：x=2^ei·m（m∈[1,2)）→ t=(m-1)/(m+1)∈[0,1/3) → `ln(m)=2t·(1+t²/3+…+t^16/17)` → `log2x=ei+ln(m)·log2e` → `yy=b·log2x` → `ki=round(yy)`、`f=yy-ki∈[-0.5,0.5]`、`z=f·ln2` → `exp(z)=Σ z^k/k! (k≤10)` → `res=exp(z)·2^ki`；截断项 z^11/11! ≈ 2e-13，远低于 f32 eps。

## 发现问题

本轮无 critical/major/minor 阻断项；以下为 suggestion（不影响结论）。

### F-001（suggestion，verification-limits）

精度声明 `1 ULP` 与「无向量寄存器溢出」为 implementer 实测结论，只读审核无法独立复现。

- 证据（patch.diff hunk2）：`+// up to ~100 ULP vs libm powf), while the f64 pipeline yields a result within`、`+// 1 ULP of libm powf on the target riscv64 libm (measured over the whole`、`+// normal positive domain for |b| <= 1, including the bvlc_alexnet LRN`；以及 `+  const size_t vlmax = __riscv_vsetvlmax_e32m4();` 所在的 `e64m4` 流水线。
- 说明：read-only 审核不能执行数值扫描或反汇编，标记 cannot verify。另注意输入域限定为正规格化，但输出可进入次正规（如 b=-1、x→f32 max 时结果 ≈2^-128）；次正规输出的相对误差合同未在注释中界定。
- 建议：以标量 libm powf 为 reference 做全正规格化域 ULP 扫描（含结果落次正规/下溢边界、n=0/小 n/main-loop 整倍数/各 tail 长度），并对目标 `-march` 反汇编确认无 vector spill；在注释中补充次正规输出行为。

### F-002（suggestion，numeric-contract）

`vfcvt_x_f_v_i64m4` 与 `vfncvt_f_f_w_f32m2` 依赖动态舍入模式（FRM，默认 RNE）。

- 证据（patch.diff hunk2）：`+  const vint64m4_t ki = __riscv_vfcvt_x_f_v_i64m4(yy, vl);`、`+  __riscv_vse32_v_f32m2(y + i, __riscv_vfncvt_f_f_w_f32m2(res, vl), vl);`。blueprint `diagnosis` 的 `recommendedVerification` 将「FRM/FCSR 状态核对」列为验证项。
- 建议：在注释显式声明该路径假定 FRM=round-to-nearest-even，或在入口校验 frm，非默认舍入模式下回退标量 libm powf；验证覆盖 FRM≠RNE。

### F-003（suggestion，performance）

域检查为完整标量预扫，向量路径随后再次读取整个 `a`；`b==1`/`b==0` 精确快路径也被迫先扫描。

- 证据（patch.diff hunk2）：`+  for (int i = 0; i < n; ++i) {\n+    uint32_t u;\n+    std::memcpy(&u, &a[i], sizeof(u));`，以及 `+  if (rvv_powx_domain_ok(a, n, b)) {\n+    if (b == 1.0f) {`。
- 建议：`b==1`（恒等拷贝）与 `b==0`（常量填充）对所有输入均精确成立，可不做逐元素域扫描直接执行；一般路径可考虑将域判定融合进向量分块或按块提前退出，避免对大数组重复读取。

### F-004（suggestion，maintainability）

`DEFINE_VSL_UNARY_FUNC_WITH_PARAM` 宏在移除 Powx 展开后已无任何使用点。

- 证据（patch.diff hunk2）：删除行 `-DEFINE_VSL_UNARY_FUNC_WITH_PARAM(Powx, y[i] = pow(a[i], b))`；仓库内该宏仅剩定义处 `#define DEFINE_VSL_UNARY_FUNC_WITH_PARAM(name, operation) \` 与说明注释。
- 建议：删除该未使用宏，或保留并注明其已无实例化点，避免后续误用。

### F-005（suggestion，robustness）

`rvv_map_f32` 主循环边界 `i + static_cast<int>(vlmax) <= n` 以有符号 int 求和。

- 证据（patch.diff hunk2）：`+  for (; i + static_cast<int>(vlmax) <= n; i += static_cast<int>(vlmax)) {`；对照被替换的原始标量循环 `for (int i = 0; i < n; ++i)`（无加法溢出路径）。
- 建议：改为 `n - i >= static_cast<int>(vlmax)` 或使用 `size_t` 计数，规避 `n` 接近 `INT_MAX` 时 `i + vlmax` 的有符号溢出（理论边界，正常 Caffe blob 尺寸不触发）。

## 幻觉自检

| 维度 | 结果 | 依据 |
|---|---|---|
| 证据锚点真实性 | PASS | 所有 findings 的 evidence 均逐字引用 `patch.diff` hunk 原文或 blueprint/patch-plan 具体字段；无凭模型知识推断的锚点。 |
| 无臆造事实 | PASS | 无法独立验证项（ULP 实测、无 vector spill）均标注 cannot verify 并降为 suggestion，未写成既成事实。 |
| 约束边界合规 | PASS | 对照 `blueprint.constraints.mustPreserve` 逐项核验：IEEE powf 语义（域外逐位回退）、CHECK 守卫、公共 API（vsPowx/vdPowx/模板签名与原宏一致）、in-place 别名、非 V 回退均成立。 |
| RISC-V 事实准确 | PASS | ISA/VLEN/LMUL/扩展均以 `metadata.json` 的 `Tag_RISCV_arch`（v1p0/zve32f/zve64d）与 blueprint/diaglog（VLEN=256、vlenb=32）为准，未外推。 |
| 逻辑一致性 | PASS | 数值域/回退/快路径/别名声明与 diff 逐行一致；f64 管线推导自洽；F-001/F-002/F-005 修订经原文比对确认。 |

## 结论

`reviewResult = pass`，`archReview.status = passed`。补丁在 `include/caffe/util/mkl_alternate.hpp` 内为 `vPowx<float>` 增加受 `__riscv_v && __riscv_v_intrinsic` 保护的 RVV 1.0 f64 向量管线，安全域外逐位回退标量 libm powf，公共 API、CHECK 守卫、in-place 别名与非 V 回退均保持；上一轮 major（F-001）及两条 suggestion 已修复。本轮 5 条均为 suggestion，不阻断交付。
