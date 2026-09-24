# AI 补丁审核报告

- reviewId: rv-bp-mxnet-opperf-category-reduction-003
- patchCandidateId: pc-bp-mxnet-opperf-category-reduction-003
- reviewer: opencode independent read-only reviewer
- reviewedAt: 2026-09-21T07:30:00.000Z

## 审核范围

- RootCauseBlueprint: `.yuansheng/trace/mxnet-opperf/category-reduction/003_003-void_mxnet_op_broadcast_seq_reduce_compute_mshadow_red_minim/blueprint_mxnet-opperf_category-reduction_003.json`
- diaglog: 同目录 `diaglog_*.md`
- PatchPlan / patch.diff / PatchCandidate: `craft/` 下同名产物（单文件 `src/operator/tensor/broadcast_reduce-inl.h`）
- arch-scan: `archSpecific: true`（规则 `riscv-macro`），执行 RISC-V 架构专项审核

## RISC-V 架构审核

1. **指令在目标 ISA 内**：使用 `vsetvl e32,m1`、`vle32.v`、`vlse32.v`、`vfredmin.vs`、`vmfeq`、`vmnot`、`vfirst`、`vfmv`。diaglog Phase 1 记录 build `Tag_RISCV_arch` 含 `v1p0`/`zvl128b`，blueprint `source.targetHardware` = SG2044/C920v2（RVV 1.0, VLEN=128）。均为 RVV 1.0 基础指令，在 ISA 内。
2. **vsetvl/vtype 一致性**：`__riscv_vsetvl_e32m1` 与 `vfloat32m1_t`/`vbool32_t`/`_f32m1` 系列匹配；按剩余 `len-off` 求 vl，tail 自然截断。
3. **tail/mask policy**：默认 `ta,ma`；mask 仅用于 NaN 检测与首个最小值定位，写侧无 mask 依赖。
4. **标量↔向量边界**：单寄存器组 m1，无重叠；stride==1 走 `vle32`，stride>1 走 `vlse32`。
5. **无 x86/ARM 指令混入**。

## 审核结果

**pass**

## 发现问题

| id | severity | category | 说明 |
|----|----------|----------|------|
| F1 | minor | 代码风格 | `std::is_same<OP, identity>::value` 为运行期分支而非 `if constexpr`；非命中实例化仍编译该分支。无正确性影响。 |
| F2 | suggestion | 性能 | `MinLength()` 每次调用 `__riscv_vsetvlmax_e32m1()`；可在编译期常量或缓存。 |
| F3 | suggestion | 覆盖范围 | 仅 `minimum` reducer 有向量实现，`maximum/nrm2/product` 仍走标量（本蓝图范围外，后续蓝图处理）。 |

无 critical/major。

## 证据与验证记录

- **正确性（独立复核，本轮 reviewer 亲自执行）**：将补丁中的 `mshadow_rvv_min_f32` 逐字提取，与 `mshadow::red::minimum::Reduce` 的标量语义（`if (!isnan(dst)) { if (!(dst <= src)) dst = src; }`）做差分测试。`riscv64-linux-gnu-g++ -O2 -march=rv64gcv -mabi=lp64d -static` 编译，`qemu-riscv64` 运行：**22,151 例，0 失败**（覆盖 len 0..40、stride 1..3、seed ∈ {+inf,-inf,1e30,+0,-0}、{±0,±inf,NaN} 三元素全组合、20000 例随机长序列）。NaN 传播与 ±0 first-wins 语义逐位一致。
- **语义推导**：标量 reducer 在 dst 非 NaN 时以 `!(dst <= src)` 更新，等价于“严格更小时替换、相等保留较早者”；向量路径取 chunk 内首个等于 chunk_min 的元素作为候选，再以 `candidate < best` 严格比较合并，与标量 first-wins 一致（含 ±0 同值不同符号情形）。
- **索引一致性**：`use_rvv` 仅在 `red_count == 1` 时启用；调用方 `M = rshape.Size()`，单非 1 维时 `M == rshape[red_dim]`，故 `big + j + k*rstride[red_dim]` 与标量 `dot(unravel(k,rshape), rstride)` 等价。
- **范围**：仅修改 `!use_omp` 分支并保留原标量循环作为 fallback；`use_omp` 路径、公共签名、`IndexOP`/`OP` 非 identity 情形不变。

## 幻觉自检

- [PASS] 技术精度：ISA 结论引用 diaglog build ISA 与 blueprint targetHardware；reducer 语义引用 `3rdparty/mshadow/mshadow/base.h` 原文。
- [PASS] 声明溯源：findings 的 evidence 均取自 patch.diff 新增行。
- [PASS] 可解释性：各 finding 说明影响与为何不阻断。
- [PASS] 内部一致性：reviewResult=pass，无 critical/major。
- [PASS] 安全：无越权建议。

## 结论

补丁针对 `seq_reduce_compute<minimum,...>` 内层 k 归约实现 RVV 分块归约，消除逐元素 div/rem 与分支链，并严格保持 NaN/±0 语义。经独立差分验证（22,151 例，0 失败）。剩余风险：`maximum/nrm2/product` 未覆盖（后续蓝图）；未在真实 SG2044 上做 workload 级测量。审核通过。
