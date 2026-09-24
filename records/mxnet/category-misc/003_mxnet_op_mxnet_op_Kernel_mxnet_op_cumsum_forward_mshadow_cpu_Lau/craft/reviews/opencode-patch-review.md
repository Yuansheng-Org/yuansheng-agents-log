# AI 补丁审核报告

- reviewId: pr-bp-mxnet-opperf-category-misc-003
- patchCandidateId: pc-bp-mxnet-opperf-category-misc-003
- blueprintId: bp-mxnet-opperf-category-misc-003
- patch.diff: `.yuansheng/craft/mxnet-opperf/003_mxnet_op_mxnet_op_Kernel_mxnet_op_cumsum_forward_mshadow_cpu_Lau/craft/patch.diff`
- 变更文件: `src/operator/numpy/np_cumsum-inl.h`（唯一）
- 审核方式: 只读、仅基于 patch.diff + PatchCandidate + RootCauseBlueprint 与可复核命令

## 审核范围

本次审核对象是 `cumsum_forward` 前缀和 scan 的 RVV 跨 lane 批处理补丁。逐项核对：

1. 根因覆盖：Blueprint `rootCause.summary` 指出 hot loop 为纯标量串行 scan（`flw`/`add`/`fadd.s`/`bne`）。补丁新增 `cumsum_forward_rvv::Map`，用 `vle32/vfadd.vv/vse32` 批处理同一 `left` 组内连续的 lane，直接命中根因。
2. `constraints.mustPreserve` 逐条核对：
   - 每 lane 逐 j 的 FP 加法顺序：`acc = __riscv_vfadd_vv_f32m4(acc, x, vl)` 在 `for (j = 1; j < middle; ++j)` 内，每 lane 仍是严格 `acc_j = acc_{j-1} + x_j`，不引入 reassociation / 无 Hillis-Steele，bit-exact 成立。
   - OpenMP 分片结构：RVV 路径复用 `Kernel<cumsum_forward_rvv, xpu>::Launch`，即同一 `Kernel<cpu>::Launch` 的 `#pragma omp parallel for`；chunk 间写入区间不相交。
   - axis=None / trailing<VLEN 边界：`axis=None`（middle=Size, trailing=1）由 `trailing > 1` 判据回退原标量 `Map`；`trailing < VLMAX` 时 `right_end` 被 clamp 到 `trailing`，由 runtime `vsetvl` 处理尾块。
3. 范围合理性：仅改 1 个 candidateFile，无公共签名/语义变更，无格式噪音。
4. 产物一致性：patch.diff 与 patch-candidate.json 的 gitDiff 一致；candidate.changedFiles == diff 文件；arch-scan `archSpecific: true`。
5. 安全性：无系统调用/外部输入解析/权限改动。

## RISC-V 架构审核

- 目标 ISA：SG2044 / XuanTie C920v2，RVV 1.0，VLEN=128（`vlenb=16`，SEW=32 时 m4 VLMAX=16）。
- build ISA（metadata `Tag_RISCV_arch`）：`rv64i2p1…v1p0…zve32f1p0…zve64f1p0…zvl128b1p0`；hardware ISA 含 `v` 与 `zve32f/zve64f`。所用指令均属该子集，无 x86/ARM 指令。
- 指令合规：`vsetvlmax_e32m4` / `vsetvl_e32m4` / `vle32_v_f32m4` / `vse32_v_f32m4` / `vfadd_vv_f32m4` 均属 RVV 1.0；SEW=32 浮点要求 `zve32f`，build/hardware 均具备。
- vsetvl/vtype 一致性：`vlmax` 仅用于划分 chunk 与计算 `chunks_per_group`；每块内以 `__riscv_vsetvl_e32m4(right_end - i)` 动态求 `vl`，`vl <= vlmax`，`vle/vse/vfadd` 全部携带同一 `vl`，vtype 一致。
- tail/mask policy：不使用 mask，采用 runtime-VL tail（`vsetvl` 截断），尾块（`right_end - right < vlmax`）由 `while (i < right_end)` + 动态 `vl` 覆盖，无越界。
- register-group 重叠：m4 为 4 寄存器组；live 向量仅 `acc` 与 `x`（v24/v28 从反汇编可见），无重叠，未超出 RVV 32 寄存器预算（acc+x=8 个）。
- 目标 ISA 内证据：diff 内 `vfloat32m4_t`、`__riscv_vsetvl*_e32m4`、`__riscv_vle32_v_f32m4`、`__riscv_vfadd_vv_f32m4` 均为 RISC-V RVV intrinsic；`#if defined(__riscv_v_intrinsic) || defined(__riscv_vector)` 守卫。

结论：RISC-V 架构专项审核 passed。

## 审核结果

pass。无 critical/major findings；存在 3 条 minor/suggestion 级覆盖与调优观察，不影响根因修复的正确性与约束保持。

## 发现问题

- F-001（minor，覆盖范围）：`axis=None`（`trailing == 1`）与单 lane 形状走原标量路径，不获得向量加速。这是 bit-exact 约束（不得沿 scan 轴 reassociate）下的必要回退，非缺陷，但意味着 `(10000,1)` 类输入无收益。
  证据（patch.diff 原文）：`+          std::is_same<OType, float>::value && trailing > 1) {`
- F-002（minor，类型覆盖）：仅 `float32` I/O 走 RVV；`float16/float64/int*` 回退标量。属技术必要（仅实现 e32m4/f32 路径），已正确回退。
  证据（patch.diff 原文）：`+      if (std::is_same<xpu, cpu>::value && std::is_same<IType, float>::value &&`
- F-003（suggestion，调优）：chunk 粒度绑定 e32m4 的 VLMAX，`trailing` 较小时 chunk 数偏少（如 trailing=100 → 7 chunks），可能不足以填满多核；后续可按 `trailing`/线程数在 m1/m2/m4 间自适应。当前实现不影响正确性。
  证据（patch.diff 原文）：`+        const index_t chunks_per_group = static_cast<index_t>((trailing + vlmax - 1) / vlmax);`

## 幻觉自检

- 技术精度：PASS — 所有指令/内在函数名与实际补丁一致；未虚构性能数字，收益以 Blueprint 实测份额为准。
- 声明溯源：PASS — 每条 finding 的 evidence 均为 patch.diff 字面行；ISA/VLEN 取值来自 metadata.json 与 build_isa.txt。
- 可解释性：PASS — 从标量 scan 到跨 lane 批处理的映射给出了每 lane 加法顺序不变的显式论证。
- 内部一致性：PASS — patch.diff、patch-candidate.json、patch-plan.json 的路径与范围一致。
- 安全：PASS — 无越界写：`right_end <= trailing`、`vl = min(vlmax, right_end-i)`，chunk 区间互斥。

## 验证证据（非审核产物，供溯源）

- 差分测试 `/tmp/opencode/cumsum_diff.cc`：原始标量 kernel 与补丁 `cumsum_forward_rvv::Map` 逐字复制，`-O2 -march=rv64gcv -mabi=lp64d -static` 编译，`qemu-riscv64 -cpu rv64,v=true,vext_spec=v1.0,vlen=128` 运行：`cases=2305 fails=0`（含 in-place 别名、inf/-inf/NaN/-0.0/denormal、middle/trailing/num_left 全组合）。
- `riscv64-linux-gnu-objdump -d` 确认生成 `vsetvli … e32,m4`、`vle32.v`、`vse32.v`、`vfadd.vv`。
- 说明：完整 MXNet 头文件 `-fsyntax-only` 因 3rdparty 子模块（dmlc-core/nnvm）未检出而不可行；以逐字 kernel 差分 + intrinsic 生成指令反汇编替代。

## 结论

补丁正确定位并缓解 Blueprint 根因（cumsum 前缀和 scan 未向量化），保持每 lane 加法顺序 bit-exact，保留 OpenMP 分片与标量回退，RISC-V 架构合规。判定：**pass**。
