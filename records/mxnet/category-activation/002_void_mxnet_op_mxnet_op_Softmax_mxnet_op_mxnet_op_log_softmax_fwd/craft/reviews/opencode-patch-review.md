# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-activation-002-r2`
- patchCandidateId: `pc-bp-mxnet-category-activation-002`
- reviewer: `opencode-cpp-reviewer`（独立只读审核）
- reviewedAt: `2026-09-21T05:19:32Z`
- 审核轮次: review_round=1（针对上一轮 critical F1 的修订复审）
- reviewResult: **pass**

## 审核范围

- 只读输入：
  - 蓝图 `.yuansheng/trace/mxnet/category-activation/002_void_mxnet_op_mxnet_op_Softmax_mxnet_op_mxnet_op_log_softmax_fwd/blueprint_mxnet_category-activation_002.json`
  - 同目录 `diaglog_mxnet_category-activation_002.md`、`metadata.json`
  - `<OUT>/craft/patch-plan.json`、`<OUT>/craft/patch.diff`、`<OUT>/craft/patch-candidate.json`
- 补丁改动文件（与 `patch-candidate.json.changedFiles` 一致）：
  - `CMakeLists.txt`（RISC-V `-march` 探测与追加）
  - `src/operator/nn/softmax-inl.h`（RVV log-softmax 快路径 + 向量 exp + FP16 窄化存储）
- 事实一致性：`patch-candidate.json.gitDiff` 与 `patch.diff` 逐字节相等（10518 bytes），候选与差异一致。
- 机器检查：
  - `arch-scan patch.diff` → `archSpecific=true`，命中 `riscv-macro`、`march-flag`（`-march=rv64gcv_zfhmin_zvfhmin`）。
  - 独立交叉编译验证：把 `softmax_rvv_detail` 代码块抽出，用 `clang++ --target=riscv64-unknown-linux-gnu -march=rv64gcv_zfhmin_zvfhmin -O3` 编译成功；`readelf` 确认 `Machine: RISC-V`，反汇编无 x86/ARM 指令。

## RISC-V 架构审核

`archSpecific=true`，执行 RISC-V 专项审核（仅依据 blueprint/diaglog/metadata 证据 + 交叉编译产物）。

1. ISA 成员资格（SG2044：RVV 1.0，VLEN=128，`zve32f/zve64f/zvl128b/zfh/zfhmin/zvfh/zvfhmin`）
   - 使用的 intrinsic 全部落在 base V（`zve32f`）或 `Zvfhmin`：`vle32/vse32/vfsub/vfmul/vfadd/vfmax/vfmin/vfredmax/vfredusum/vfcvt/vmfne/vmerge/vsll/vadd/vreinterpret/vfncvt/vse16`。
   - `vfncvt.f.f.v` 需要 `Zvfh/Zvfhmin`，由 `MXNET_RVV_F16_NARROW` 宏在 `__riscv_zvfh || __riscv_zvfhmin` 下开启；交叉编译 `-march=rv64gcv_zfhmin_zvfhmin` 生成 `vsetvli zero,zero,e16,mf2,ta,ma` + `vfncvt.f.f.w` + `vse16.v`，SEW/LMUL 比（f32m1→f16mf2）正确。
   - 无任何 x86/ARM 指令或宏。
2. vsetvl / LMUL / SEW
   - 统一使用 `vsetvl_e32m1`，与 `vfloat32m1_t / vbool32_t / vint32m1_t` 一致（SEW=32、LMUL=m1）；尾块用 `vsetvl_e32m1(M-j)` 运行时截断。
   - 反汇编确认主循环 `vsetvli ...,e32,m1,ta,ma`，窄化存储前 `vsetvli zero,zero,e16,mf2,ta,ma`。
3. tail / mask
   - 固定 VL 主循环 + 运行时 tail；reduction 采用 seed 式（标量 seed 跨块携带），无越界读；mask 仅用于 NaN 再注入（`vmfne.vv` + `vmerge.vvm`）。
4. dispatch 门控
   - 编译期 `#if defined(__riscv) && defined(__riscv_v)`；类型级 `if constexpr (rvv_log_softmax_fwd_eligible<...>)`；运行期 `length == nullptr && sa == 1 && temperature == DType(1.0)`，不满足则回落到原 scalar 路径，非 RISC-V 编译单元完全不受影响。
5. F1 修复复核（上一轮 critical）
   - 现状 `patch.diff` 第 89–91 行：
     - `+  vfloat32m1_t r =`
     - `+      __riscv_vfsub_vv_f32m1(xc, __riscv_vfmul_vf_f32m1(k, 0.6931457519531250f, vl), vl);`
     - `+  r = __riscv_vfsub_vv_f32m1(r, __riscv_vfmul_vf_f32m1(k, 1.428606765330187e-06f, vl), vl);`
   - `r = xc - k*ln2_hi - k*ln2_lo`，**两项都乘以整数 k**。反汇编确认：`vfmul.vf v14,v13,fa2`（k*ln2_hi）→ `vfsub.vv v11,v11,v14` → `vfmul.vf v13,v13,fa1`（k*ln2_lo）→ `vfsub.vv v11,v11,v13`，其中 `v13` 即 k。上一轮的 `vfsub.vf`（scalar ln2_hi 不乘 k）缺陷已消除。
   - 标量重放：`vexp(0)=1.0`、`vexp(-1)=0.3678795`（expf 0.3678794）、`vexp(-3)=0.0497871`、`vexp(-20)=2.0612e-9`；在 log-softmax 实际使用域 `d = x - max ≤ 0` 内最大相对误差约 `3e-8`（约 1 ulp f32）。
   - 结论：**F1 已修复**，不再构成 critical/major。
6. 构建参数
   - `CMakeLists.txt` 新增 `include(CheckCCompilerFlag)`，对 C/CXX 双编译器探测 `-march=rv64gcv_zfhmin_zvfhmin`，失败回落 `-march=rv64gcv`，仅 `CMAKE_SYSTEM_PROCESSOR MATCHES "riscv64"` 时生效；上一轮的 `SYSTEM_ARCHITECTURE`（非标准变量）已移除。

## 审核结果

- reviewResult: **pass**
- archReview.status: **passed**
- 根因覆盖：蓝图根因（未向量化 3-pass scalar LogSoftmax，scalar `expf`/`log` 调用点占 84.25%，scalar compare-branch max 10.01%；build 缺 `zfhmin` 导致软件 float2half）已被 `vfredmax` + 向量 exp + `vfredusum` + `vfsub.vf` 归一化 + `vfncvt.f.f.v` 窄化存储 + `-march` 增补覆盖。
- 约束保持（blueprint `constraints.mustPreserve`）：
  - 稳定减 max：保留（pass 1 向量 max，pass 2/3 均减 mmax）。
  - NaN/±Inf/全相等：全相等与 ±Inf 结构上保持；NaN 的 `vfredmax` 传播策略未能在本环境动态验证（见 F4，suggestion）。
  - FP16 RNE：`vfncvt.f.f.v` 采用动态舍入模式，默认 RNE，与 `MSHADOW_HALF_ROUND_TO_NEAREST` 一致，已在代码注释中披露运行时改 `frm` 的风险。
  - 公共 API/模板签名：未改动 `Softmax<OP, negate, AType, DType, OType, index_t, ndim>`。
  - temperature 分支：`temperature != 1.0` 回落原 scalar `/temperature` 路径，语义保留。
- 范围聚焦：仅 2 个文件，无重构/无关改动；`softmax_fwd`、`negate=true`、`length != nullptr`、CUDA 路径未触及。
- 安全：`patch.diff` 无 `system/popen/exec/fork`、无 shell 拼接、无硬编码密钥/凭据。

## 发现问题

本轮无 critical / major。以下为 minor / suggestion（不影响 pass）。

### F1 — minor：块内 f32 无序归约与 AType=double 累加契约的偏差

- file: `src/operator/nn/softmax-inl.h`
- evidence: `+      const vfloat32m1_t blk = __riscv_vfredusum_vs_f32m1_f32m1(zero, e, cur);` / `+      sum += static_cast<double>(__riscv_vfmv_f_s_f32m1_f32(blk));`
- 说明：标量参考在 `AType=double` 下逐元素 `std::exp` 后累加；快路径改为“块内 f32 无序 `vfredusum` + 块间 double 累加”。VLEN=128 时每块至多 4 lane，偏差很小（实测域内 ~1 ulp），但结果依赖 lane 顺序、且与参考不完全同序。
- suggestion：在回归中显式校验 `log_softmax == log(softmax)` 恒等式与 `sum≈1` 的容差；如需强可复现，可评估 `vfredosum` 或块间补偿求和。

### F2 — minor：eligibility trait 未约束 AType

- file: `src/operator/nn/softmax-inl.h`
- evidence: `+struct rvv_log_softmax_fwd_eligible {` / `+      std::is_same<OP, log_softmax_fwd>::value && !negate &&` / `+      std::is_same<DType, float>::value &&`
- 说明：trait 只检查 `OP/negate/DType/OType`，未检查 `AType`。当 `MXNET_SAFE_ACCUMULATION=0`（`safe_acc=false`，实例化为 `AType=float`）时，快路径仍按 double 累加并 `std::log(double)`，与配置的 float 累加参考结果产生差异（更精确，但改变了该配置下的数值行为）。
- suggestion：将 `std::is_same<AType, double>::value` 纳入 trait，或在 `AType=float` 时明确记录该精度差异并补回归。

### F3 — suggestion：exp 多项式截断精度说明

- file: `src/operator/nn/softmax-inl.h`
- evidence: `+  // Q(r) = 1 + r/2 + r^2/6 + r^3/24 + r^4/120 + r^5/720 (Horner).` / `+  const vfloat32m1_t er = __riscv_vfadd_vf_f32m1(__riscv_vfmul_vv_f32m1(r, q, vl), 1.0f, vl);`
- 说明：注释中 Q 为 5 次（准确）；`er = 1 + r*Q(r)` 使 exp 实际到 6 次，截断项为 `r^7/5040`（|r|≤ln2/2 时约 `1.2e-7`），实测最大相对误差约 `1.4e-7`，实际使用域 `d≤0` 内约 `3e-8`。对 FP16 输出远低于分辨率。
- suggestion：对 `OType=float` 实例化，如需 ~1 ulp 可考虑 minimax/Cephes 系数；对当前 FP16 目标可维持现状，但建议在注释中写明截断项为 `r^7/5040`。

### F4 — suggestion：`vfredmax` 的 NaN 传播策略未验证

- file: `src/operator/nn/softmax-inl.h`
- evidence: `+      seed                  = __riscv_vfredmax_vs_f32m1_f32m1(seed, x, cur);`
- 说明：标量参考 `if (mmax < val)` 仅当 NaN 位于行首时才传播 NaN；向量 max 归约的 NaN 策略与标量链不同，且本环境无 RVV 硬件，无法从 blueprint/diaglog 证据判定（无法验证）。
- suggestion：补行级用例：NaN 位于非首元素、全 `-Inf`、混合 `±Inf`、全相等，逐 lane 与 scalar 参考对比后再长期启用快路径。

### F5 — suggestion：未执行真实 RISC-V CMake configure

- file: `CMakeLists.txt`
- evidence: `+    check_c_compiler_flag("-march=rv64gcv_zfhmin_zvfhmin" MXNET_RISCV_ZFHMIN_C_SUPPORTED)` / `+      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=rv64gcv_zfhmin_zvfhmin")`
- 说明：本审核环境未安装 `cmake`，无法执行真实 configure；已静态复核 if/else/endif 与变量展开，且用同一 `-march` 成功交叉编译了 RVV kernel。探测块对 C/CXX 双端探测、回落逻辑完整（已修复上一轮 F5）。
- suggestion：在 RISC-V 工具链上执行一次真实 CMake configure，确认探测命中并把 `-march` 追加到 C/CXX flags；随后跑 `category-activation` 回归并重新 perf annotate。

### F6 — suggestion：缺少小 M 交叉阈值

- file: `src/operator/nn/softmax-inl.h`
- evidence: `+    if (length == nullptr && sa == 1 && temperature == DType(1.0)) {` / `+        softmax_rvv_detail::log_softmax_fwd_row<OType>(in + base, out + base, M);`
- 说明：对很短 axis（如 M=1..3）快路径仍执行三遍向量扫描与归约，存在固定开销；diaglog 的「适用前提」也提到短 axis 可保留 scalar/crossover。
- suggestion：可选地加入小 M 阈值回落 scalar，或经基准确认短 axis 收益后维持现状。

### F7 — suggestion：candidate 的 `archReviewWarning` 为空

- file: `CMakeLists.txt`
- evidence: `arch-scan patch.diff` 返回 `archSpecific=true`（`riscv-macro`、`march-flag`），但 `patch-candidate.json.archReviewWarning` 为空字符串。
- suggestion：确认 candidate 工具按 arch-scan 结果自动写入该字段；若为工具缺口请修复工具，避免后续产物缺少架构告警。

## 幻觉自检

- 所有结论均可溯源到只读输入：蓝图 `problem/rootCause/constraints/validation` 字段、`diaglog` 的 Phase 1/3/4/5 证据锚点、`patch.diff` 原文，以及本次独立交叉编译产物（`readelf`/反汇编）。
- 未凭模型知识断言硬件行为：`vfredmax` 的 NaN 策略明确标注为「无法验证」，仅作 suggestion。
- 机器校验：`arch-scan`、`validate patch-review`、`review-validate` 均通过；findings 的 file 与 evidence 均取自真实 diff 文本。
- 五维自检：
  - 技术精度: [PASS]（F1 修复经反汇编与标量重放双重确认；多项式截断项按 `r^7/5040` 正确表述）
  - 声明溯源: [PASS]（每条 evidence 引用 patch.diff 真实行/blueprint 字段）
  - 可解释性: [PASS]（根因→改动→约束保持→残留风险链条完整）
  - 内部一致性: [PASS]（candidate.gitDiff 与 patch.diff 逐字节一致；结论与 findings 严重度一致）
  - 安全: [PASS]（无危险命令、无密钥、无越界/UB 新增）

## 结论

- **reviewResult = pass**；**archReview.status = passed**；无 critical/major。
- 上一轮 critical **F1 已修复**：`patch.diff` 第 90–91 行现为
  `+      __riscv_vfsub_vv_f32m1(xc, __riscv_vfmul_vf_f32m1(k, 0.6931457519531250f, vl), vl);`
  `+  r = __riscv_vfsub_vv_f32m1(r, __riscv_vfmul_vf_f32m1(k, 1.428606765330187e-06f, vl), vl);`
  即 `r = xc - k*ln2_hi - k*ln2_lo`，两项均乘 k；反汇编 `vfmul.vf ...fa2` + `vfsub.vv` + `vfmul.vf ...fa1` + `vfsub.vv` 佐证。
- 残留风险（均为 minor/suggestion）：块内 f32 无序归约与 AType 契约偏差、trait 未约束 AType、`vfredmax` NaN 策略与多项式截断精度未动态验证、未跑真实 CMake configure 与 `category-activation` 回归、小 M 无阈值、candidate `archReviewWarning` 为空。
- 建议进入 `done`；回归验证（`agent1 patch_regression --case category-activation` 与重新 perf annotate）作为后续动态确认项。
