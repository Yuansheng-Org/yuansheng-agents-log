# AI 补丁审核报告

- reviewId: rv-bp-mxnet-opperf-category-convolution-002
- patchCandidateId: pc-bp-mxnet-opperf-category-convolution-002
- reviewer: opencode independent read-only reviewer
- reviewedAt: 2026-09-21T06:10:00.000Z

## 审核范围

- RootCauseBlueprint: `.yuansheng/trace/mxnet-opperf/category-convolution/002_void_mxnet_op_im2col_float_mshadow_Stream_mshadow_cpu_float_cons/blueprint_mxnet-opperf_category-convolution_002.json`
- diaglog: 同目录 `diaglog_mxnet-opperf_category-convolution_002.md`
- PatchPlan: `craft/patch-plan.json`
- patch.diff: `craft/patch.diff`（单文件 `src/operator/nn/im2col.h`）
- PatchCandidate: `craft/patch-candidate.json`
- arch-scan: `archSpecific: true`（命中规则 `riscv-macro`），因此执行 RISC-V 架构专项审核

## RISC-V 架构审核

1. **指令/特性在目标 ISA 内**：blueprint `source.targetHardware` = `SOPHGO SG2044 / XuanTie C920v2 (RVV 1.0, VLEN=128)`；diaglog Phase 1 记录 build `Tag_RISCV_arch` 含 `v1p0`、`zve32f`、`zve64d`、`zvl128b`。补丁使用的 `vsetvli`、`vle32.v`、`vlse32.v`、`vse32.v`（SEW=32, LMUL=1）均属 RVV 1.0 基础向量指令，在硬件与 build ISA 内。结论：通过（有 diaglog/blueprint 证据）。
2. **vsetvl/vtype 配置与数据流一致**：`__riscv_vsetvl_e32m1(n-i)` 与 `vfloat32m1_t` / `_f32m1` 系列（SEW=32、LMUL=1）严格匹配；每轮以剩余长度 avl 求 vl，tail 由 vl 自然截断，无状态错乱。
3. **tail/mask policy**：使用默认 `ta,ma` 全尾策略，整向量写 `data_col` 连续区间，未依赖 mask；未构造越界指针（补丁注释与实现一致，border 单独写 0）。
4. **标量↔向量边界与寄存器组**：单向量寄存器组 m1，无重叠；stride_w==1 与 stride_w>1 分支使用不同 load intrinsic，写侧统一 `vse32.v`。
5. **运行时 ISA 分发**：采用编译期 `__riscv_v_intrinsic || __riscv_vector` 条件编译，而非 ifunc/hwprobe 运行时分发；非 RISC-V 或未开启向量扩展的构建走标量 fallback，路径自洽。
6. **未混入 x86/ARM 专属指令**：diff 中无任何非 RISC-V intrinsic。

## 审核结果

**pass**

## 发现问题

| id | severity | category | 文件 | 说明 |
|----|----------|----------|------|------|
| F1 | minor | code-style | src/operator/nn/im2col.h | `if (std::is_same<DType, float>::value)` 为运行期分支（非 `if constexpr`），非 float 实例化仍会编译 RVV 分支；建议改 `if constexpr`（C++17）或按类型特化，避免死代码。 |
| F2 | suggestion | portability | src/operator/nn/im2col.h | 使用非限定 `ptrdiff_t`；建议改为 `std::ptrdiff_t` 以保证跨标准库可移植。 |
| F3 | suggestion | performance | src/operator/nn/im2col.h | interior 循环每轮调用一次 `vsetvl`；对长 run 可先取一次 `vlmax` 并对齐块复用，减少 vsetvli 开销（不构成正确性问题）。 |

以上均为 minor/suggestion，不构成 critical/major，不影响结论。

## 证据与验证记录

- 正确性（动态）：将补丁后的 `im2col_cpu` 与 `HEAD` 标量实现做差分测试，`riscv64-linux-gnu-g++ -O2 -march=rv64gcv -mabi=lp64d -static` 编译后经 `qemu-riscv64` 运行，覆盖 269,031 组参数组合（channels/height/width/kernel/pad/stride/dilation 及 output_w 边界、整行越界、tail 长度），输出 **bit-identical，0 失败**。
- 指令生成：反汇编确认新增路径实际产生 `vsetvli e32,m1`、`vle32.v`、`vlse32.v`、`vse32.v`，与 diaglog Phase 5 预期信号一致。
- 约束保持（静态）：公共 `im2col()`/`im2col_cpu()` 签名与 `mshadow::Stream<cpu>` 契约未变；`data_col` 布局与标量路径一致；padding 写 0 语义保留；`data_im`/`data_col` 不 alias 的前提未变。
- 范围：仅 `src/operator/nn/im2col.h`，无无关重构、无格式噪音。

### 锚点说明

- F1 evidence: `+  if (std::is_same<DType, float>::value) {`
- F2 evidence: `+      const ptrdiff_t byte_stride = static_cast<ptrdiff_t>(stride) * static_cast<ptrdiff_t>(sizeof(float));`
- F3 evidence: `+        size_t vl = __riscv_vsetvl_e32m1(static_cast<size_t>(n - i));`

## 幻觉自检

- [PASS] 技术精度：架构结论均引用 diaglog 的 `Tag_RISCV_arch`（含 `v1p0`/`zvl128b`）与 blueprint `source.targetHardware`，未凭模型知识断言硬件能力。
- [PASS] 声明溯源：每条 finding 的 evidence 均取自 patch.diff 新增行原文。
- [PASS] 可解释性：各 finding 说明“为什么必须改/为何只是建议”，且明确不影响 pass。
- [PASS] 内部一致性：reviewResult=pass，findings 中无 critical/major。
- [PASS] 安全：无越权建议、无引入新风险的建议。

## 结论

补丁聚焦解决 blueprint `rootCause` 指向的 im2col 抽取循环未向量化根因，interior/border 显式分离并接入 RVV 连续/固定步长路径；已通过差分动态验证与架构专项审核。剩余风险：qemu 验证不等价于 C920v2 实测微架构行为（vlse32 吞吐、vsetvli 开销），且未运行完整 mxnet 回归；这些为验证条件限制，不影响补丁正确性结论。审核通过。
