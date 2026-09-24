# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-activation-019`
- patchCandidateId: `pc-bp-mxnet-category-activation-019`
- blueprintId: `bp-mxnet-category-activation-019`
- reviewer: `opencode-cpp-reviewer`（独立只读会话）
- reviewedAt: `2026-09-21T12:31:47Z`

## 审核范围

- 目标软件：mxnet（commit `b84609d3fc73d20929c114eab95faaa56e6c5ede`，testcase `category-activation`）
- RootCauseBlueprint：`.yuansheng/trace/mxnet/category-activation/019_Kernel_backward_grad_tuned_relu_grad/blueprint_mxnet_category-activation_019.json`
- PatchPlan：`pp-bp-mxnet-category-activation-019`
- PatchCandidate：`pc-bp-mxnet-category-activation-019`
- 审核对象：`craft/patch.diff`，仅含两处源码改动
  - `src/operator/mshadow_op.h`：新增 `#include <riscv_vector.h>` 守卫块 + `namespace relu_grad_rvv`（`relu_grad_block_f32m1` / `relu_grad_apply_range`）
  - `src/operator/nn/activation-inl.h`：新增 `Kernel<op_with_req<backward_grad_tuned<mshadow_op::relu_grad>, req>, mshadow::cpu>` 的 RVV 偏特化（float32 向量路径 + 变参标量回退）
- 未改动：`relu_grad::Map` 标量语义、`ActivationForward`/`ActivationBackward`、请求分发 switch、其它激活算子
- 审核方式：源码语义比对 + 蓝图契约核对 + 交叉编译/反汇编/qemu 动态验证（见下）

## RISC-V 架构审核

- `arch-scan` 机器判定：`archSpecific = true`，命中规则
  - `riscv-macro`：`+#if defined(__riscv) && defined(__riscv_v)`
  - `vlenb`：`+  const size_t vlmax = __riscv_vsetvlmax_e32m1();`
- 架构守卫：所有 RVV 代码均在 `#if defined(__riscv) && defined(__riscv_v)` 内，非 RISC-V / 无 V 扩展构建不编译该路径；标量 `relu_grad::Map` 与既有 `Kernel<OP, cpu>` 主模板保持可用。
- 指令集合规：仅使用 RVV 1.0 基础浮点向量指令（`vle32`/`vse32`/`vmfgt.vf`/`vmfne.vv`/`vmerge.vvm`/`vfmul.vv`/`vfadd.vv`/`vsetvl`/`vsetvlmax`），属 V + Zve32f 范畴，与目标硬件 `rv64imafdcv_..._zve64d_zvfh` 及 build ISA `rv64i2p1_..._v1p0_zvl128b1p0` 兼容；未引入 x86/ARM 指令或宏。
- 无标量回退破坏：其它 dtype（`half_t`/`bf16_t`/`double`/整型等）由变参 `Launch` 回退到原 `op_type::Map` + OpenMP 逻辑。
- 编译验证（clang 21，`--target=riscv64-linux-gnu -march=rv64gcv_zvl128b -mabi=lp64d`）：逐字抽取的补丁代码块编译通过并链接。
- 反汇编验证：生成体包含 `vsetvli`、`vle32.v`、`vmfgt.vf`、`vmerge.vvm`、`vmfne.vv`、`vfmul.vv`、`vse32.v`；逐元素 compare/branch 已被 mask/select 取代，仅保留循环控制分支。
- 动态语义验证（qemu-riscv64 `-cpu max`）：以标量 reference（`isnan(a)?a:(a>0?1:0)` 再 `lhs*m`）逐位比对，覆盖 `n=0..11`（命中 fixed-VL 主循环 + runtime-VL tail 边界）与 `n=40`，输入含 ±0、±1、±Inf、qNaN/带 payload NaN、sNaN、subnormal、FLT_MAX，并分别验证 write 与 kAddTo 模式 —— **全部逐位一致**。
- 结论：`status = passed`。

## 审核结果

- `reviewResult = pass`
- 语义等价性（对照蓝图 `mustPreserve`）：
  - NaN→a（含 payload）：`vmfne(x,x)` 生成 NaN mask，`vmerge(m, x, nan)` 复制原 x 位，再参与 `vfmul`；与标量 `IsNan(a)->a` 一致，动态验证逐位通过。
  - a>0→1 / else 0：`vmfgt(x,0.0f)` + `vmerge(0.0f,1.0f,gt)`；`-0.0f > 0` 为 false → 0，与标量一致。
  - signed-zero / 乘法符号位：`vfmul(g,m)` 与标量 `lhs * relu_grad(rhs)` 操作数顺序一致，动态验证覆盖 ±0 逐位通过。
  - 乘法顺序：向量 `vfmul(g, m)` = `lhs * m`，与 `backward_grad<GRAD_OP>::Map` 的 `a * GRAD_OP::Map(...)` 一致。
  - kAddTo：`accumulate=(req==kAddTo)` → `vfadd(res, out)`；kNullOp 早退不写，与 `KERNEL_ASSIGN` 一致（`OpReqType` 仅 kNullOp/kWriteTo/kWriteInplace/kAddTo，无 kAddInplace）。
  - OpenMP：`omp_threads>=2 && N>=4096` 时按 chunk 划分 `[0,N)` 并行，否则串行；分片边界无重叠/遗漏（已核验 chunk 取整覆盖）。
  - tail 处理：fixed-VL 主循环 + `vsetvl_e32m1(n-i)` runtime-VL tail，无越界读写。
- 回退与分派：重载解析已验证 —— `float*` 签名选中向量非模板重载，`double*` 等选中变参标量回退（`op_type::Operation` = `backward_grad_tuned<relu_grad>`，`tuned_op::UseOMP` 可解析）。
- 补丁范围：`patch.diff` 仅含项目源码，无 `.yuansheng/` 协议产物混入。

## 发现问题

| id | file | severity | category | 说明 |
|---|---|---|---|---|
| F1 | src/operator/nn/activation-inl.h | minor | 性能调优一致性 | OpenMP 阈值硬编码 `N >= 4096`，偏离原 `tuned_op::UseOMP` 决策 |
| F2 | src/operator/nn/activation-inl.h | suggestion | 代码组织/ODR | 偏特化置于 activation-inl.h，非主模板头文件 |
| F3 | src/operator/mshadow_op.h | suggestion | 数值边界 | kAddTo 加法操作数顺序与标量 `out += val` 相反 |

无 critical / major 问题。

## 幻觉自检

- [PASS] 技术精度：所有结论均由可复现证据支撑 —— `arch-scan` 输出、clang 交叉编译、`objdump` 反汇编、qemu-riscv64 逐位比对（write/kAddTo × n=0..11/40 × 特殊值集）；未凭模型记忆断言。
- [PASS] 声明溯源：语义契约逐条对应蓝图 `constraints.mustPreserve` 与源码 `mshadow_op.h:1541-1547`（`relu_grad::Map`）、`mxnet_op.h:805-808`（`backward_grad::Map`）、`mxnet_op.h:599-614`（`KERNEL_ASSIGN`）；findings 均引用 patch.diff 新增行原文。
- [PASS] 可解释性：mask/select 推导链（vmfgt → vmerge(0/1) → vmfne → vmerge(m,x) → vfmul）与标量三分支一一对应，已说明 NaN/±0/±Inf/subnormal 的行为。
- [PASS] 内部一致性：`reviewResult=pass` 与 findings 全为 minor/suggestion 一致；`archReview.status=passed` 与 `archSpecific=true` 及 RVV 指令使用一致。
- [PASS] 安全：改动不涉及密钥、输入边界、外部 IO、SQL/命令执行或权限；RVV 代码受架构守卫保护；无越界访问（主循环+tail 已论证并经 n=0..11 验证）。

## 结论

补丁忠实实现了 PatchPlan：在 RVV 1.0 硬件上以 branch-free 的 compare/mask/select 路径替换 `relu_grad` 逐元素标量 compare/branch 形态，并严格保持标量分段语义（NaN→a 含 payload、a>0→1、else 0、signed-zero、乘法顺序）、kAddTo/kNullOp 语义、OpenMP 并行与其它 dtype 标量回退；无 x86/ARM 指令；动态逐位验证通过。未发现 critical/major 缺陷，判定 **pass**。剩余 F1–F3 为 minor/suggestion 级别改进项，不阻断交付。

> 环境说明：动态验证使用 qemu-riscv64 `-cpu max`；`-cpu rv64,v=true` 因本机 clang `-march=rv64gcv` 展开出向量密码学扩展（zvbb/zvkb/zvkt）而报 illegal instruction，属测试用 CPU 模型特性配置差异，与补丁代码（仅基础 RVV 浮点指令）无关。真实 SG2044 硬件上的性能收益仍需按蓝图 `recommendedVerification` 复采确认。
