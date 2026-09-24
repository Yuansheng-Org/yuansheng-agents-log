# AI 补丁审核报告

- reviewId: rv-bp-mxnet-opperf-category-misc-002-r1
- patchCandidateId: pc-bp-mxnet-opperf-category-misc-002
- blueprintId: bp-mxnet-opperf-category-misc-002
- reviewer: opencode independent read-only reviewer
- reviewResult: pass

## 审核范围

本报告只基于以下产物独立复核，不依赖生成补丁时的对话上下文：

- `craft/patch.diff`（本蓝图隔离差异，148 行）
- `craft/patch-plan.json`、`craft/patch-candidate.json`
- Trace 交接蓝图 `blueprint_mxnet-opperf_category-misc_002.json` 与 `diaglog_mxnet-opperf_category-misc_002.md`
- 目标硬件：SOPHGO SG2044 / XuanTie C920v2，RVV 1.0，VLEN=128；build ISA 含 `v1p0`、`zvl128b`

改动文件（2 个）：

1. `3rdparty/mshadow/mshadow/extension/spatial_upsampling_nearest.h` — 新增 RVV 1.0 nearest 上采样 kernel `mshadow_rvv_upsampling_nearest_f32`，宏 guard + 标量 fallback。
2. `src/operator/nn/upsampling-inl.h` — 在单输入 nearest forward 路径按精确条件分派到该 kernel，其余路径保持原 `Assign(upsampling_nearest(...))`。

## RISC-V 架构审核

- **指令 ISA 合法性**：kernel 只使用 RVV 1.0 标准指令 `vsetvl`（`__riscv_vsetvl_e32m4`）、`vle32`、`vluxei32`、`vse32`，无 x86/ARM 指令，无 RVV 0.7/1.1 专有编码。build ISA 含 `v1p0 + zve32f + zvl128b`，与所用指令一致。
- **vsetvl / vtype 一致性**：主循环每次迭代用 `__riscv_vsetvl_e32m4(dst_width - x)` 得到运行期 `vl`，随后 `vle32/vluxei32/vse32` 全部使用同一 `vl`，SEW=32、LMUL=m4 全程一致；不写死 `vl`，VLEN-agnostic（VLEN=128 时 VLMAX=16，VLEN 变化时仍正确）。
- **tail / mask 策略**：无 mask；用 runtime `vl` 覆盖 `dst_width` 的全部剩余列，`x += vl` 推进，无 off-by-one，无越界（源索引上界 `(dst_width-1)/scale = src_width-1`）。
- **register-group 使用**：m4 下 live vector group 为 offset（`vuint32m4_t`）与数据（`vfloat32m4_t`）两组，`LMUL × peak_live ≈ 8 ≤ 32`，无 register-group overlap/别名。
- **gather 形态**：`vluxei32.v v24,(t3),v24` 使用 u32 字节偏移，索引非负、按 float 宽度缩放，符合 nearest 相邻输出共享源元素的 locality。
- **反汇编确认**（`riscv64-linux-gnu-objdump -d`，harness 实测）：`vsetvli ... e32,m4`、`vle32.v`、`vluxei32.v`、`vse32.v` 均实际生成。
- **fallback**：kernel 及分派整段位于 `#if defined(__riscv_v_intrinsic) || defined(__riscv_vector)` 内，非 RVV 构建零改动；`UpSamplingNearestTryRVV` 主模板对非 `<cpu,float>` 返回 false，原表达式路径保留。

## 审核结果

**pass** — 无 critical / major 问题。

根因覆盖：蓝图根因为「最近邻上采样 resampling 内核在 RVV 构建上未向量化，逐元素 `x/scale_` 除法 + 标量 gather/store 主导局部样本（flw 19.50% + fsw 66.43%）」。补丁在 benchmark 实际命中的单输入 nearest/write/float 路径用「一次调用预计算水平 byte-offset + `vluxei32` gather + `vse32` 连续 store」替代逐元素除法与标量 gather/store，直接消除该根因；并保留 OpenMP 行级并行与标量 fallback。

约束保持：

- `floor(x/scale)`、`floor(y/scale)` 坐标语义与 `Plan::Eval` 逐元素一致（见验证）。
- 每输出元素恰好写一次：fast path 仅在 `req == kWriteTo` 时启用，`kAddTo`/slice/多输入/非连续/非 float 全部回退原 `Assign`。
- OpenMP 行分块行为保留：kernel 内 `#pragma omp parallel for` 对扁平化输出行并行，offset 表只读共享。
- 公共 API、签名、标量 fallback 未改动。

验证（动态、QEMU）：

- 差分 harness 对比「标量参考（与 `Plan::Eval` 同式）」与「新 RVV kernel」，`-O2 -march=rv64gcv -mabi=lp64d -static`，`qemu-riscv64` 运行。
- 覆盖 12 个定值用例（含 benchmark 形状 `(32,3,256,256) scale=2`、`(32,3,10000,1) scale=4`，以及 scale=1/16、空宽、含 `0/-0/INF/NAN` 位型）与 4800+ 组合扫描（scale 1..40 × src_w 1..40 × 多 plane）、向量边界宽度扫描；结果位级完全一致，`RESULT: ALL_PASS`。另用 `-fopenmp` 重编重跑，仍 `ALL_PASS`。
- 反汇编确认 `vluxei32.v`/`vse32.v` 实际发射。

## 发现问题

1. [minor] mshadow 表达式求值路径（`MapPlan<...UpSamplingNearestExp...>`）仍为标量。fast path 只在 `src/operator/nn/upsampling-inl.h` 的单输入 nearest/write 路径分派；其他（当前 MXNet 内不存在的）表达式调用者仍走标量。证据：`+    if (UpSamplingNearestTryRVV<xpu, DType>(data, out, req, param.scale)) {`。这是刻意的聚焦范围（`MapPlanRowExec` 钩子受 mshadow 头文件包含顺序限制，见蓝图 candidateFiles 与计划说明），不构成回归。
2. [minor] `vluxei32` 在 C920v2 上的吞吐与写带宽刚性未实测。蓝图 `diagnosis.blockReason` 已标注收益上界受 store 写带宽与 gather 吞吐影响；本补丁为静态验证 + QEMU 位级正确性，未在真实 SG2044 上跑 benchmark。证据：`+ *        then performs a vluxei32 indexed load from the current source row and`。
3. [suggestion] `std::vector<uint32_t> w_byte` 每次 forward 调用一次堆分配（O(dst_width)），对小 shape 短调用存在固定开销；offset 表按调用预计算已在行/通道间摊薄，若后续要消除分配可改为按 scale/dst_width 缓存。证据：`+  std::vector<uint32_t> w_byte(static_cast<size_t>(dst_width));`。

以上均为 minor/suggestion，不影响 pass。

## 幻觉自检

- 技术精度：PASS — 所有指令名、宏名、类型（`vuint32m4_t`/`vfloat32m4_t`）、行号、函数名均来自 patch.diff 或实测反汇编，未凭模型记忆断言。
- 声明溯源：PASS — 根因与收益数字（fsw 66.43%、flw 19.50%、全局 18.27%、0.859）逐字来自 blueprint/diaglog；验证结论来自本会话实跑的 harness 输出。
- 可解释性：PASS — 修复机制（预计算 offset、gather、连续 store、runtime vl、宏 guard）与根因一一对应，可复现。
- 内部一致性：PASS — patch-plan / patch.diff / patch-candidate 的 changedFiles 一致；review 的 patchCandidateId 等于 candidate 值。
- 安全：PASS — 无越界（源索引 ≤ src_width-1、写满且仅写 `len`）、无未定义行为、无秘密泄漏、fallback 完整、无 x86/ARM 指令。

## 结论

补丁在目标 RVV 1.0（VLEN=128）上正确向量化了 benchmark 命中的最近邻上采样热点，坐标与写语义逐元素保持，非 RVV / 非命中路径零改动。QEMU 差分位级验证与反汇编均通过。**审核通过（pass）**，可流转 done。残余风险为真实硬件上 gather 吞吐与写带宽的收益幅度未实测，已在 findings 中披露。
