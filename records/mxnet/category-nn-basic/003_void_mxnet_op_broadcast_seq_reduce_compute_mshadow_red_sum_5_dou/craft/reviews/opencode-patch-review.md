# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-nn-basic-003-r1`
- patchCandidateId: `pc-bp-mxnet-category-nn-basic-003`
- patchPlanId: `pp-bp-mxnet-category-nn-basic-003`
- blueprintId: `bp-mxnet-category-nn-basic-003`
- 审核人: `opencode-comet-reviewer`（独立只读会话）
- 审核日期: 2026-09-21
- 被审对象: `.yuansheng/craft/mxnet/003_void_mxnet_op_broadcast_seq_reduce_compute_mshadow_red_sum_5_dou/craft/patch.diff`
- patch.diff sha256(前16位): `6f64e6db32f2b089`
- 审核轮次: 1（`review_round: 0` → 首轮）

## 审核范围

本轮审核覆盖四个层面：

1. **通用质量**：补丁是否聚焦已诊断根因、是否引入未声明的行为变化、是否可读可维护、是否满足 `PatchPlan.changes` 的范围约定。
2. **约束保持**：逐条核对蓝图 `constraints.mustPreserve`（见下「约束保持逐条核对」）。
3. **RISC-V 架构专项**：由 `arch-scan` 机器判定。
4. **防幻觉锚点**：所有 finding 的 `file` / `line` / `evidence` 必须能在 `patch.diff` 的 hunk 原文或蓝图字段中逐字锚定，禁止凭模型知识推断。

被审补丁的事实来源为 `git diff` 输出（`PatchCandidate.gitDiff` 与 `patch.diff` 字节一致，已校验 `True`）。改动范围为单文件单 hunk：

```
diff --git a/src/operator/tensor/broadcast_reduce-inl.h b/src/operator/tensor/broadcast_reduce-inl.h
@@ -361,13 +361,27 @@ MSHADOW_XINLINE void seq_reduce_assign(const index_t idx,
 1 file changed, 16 insertions(+), 2 deletions(-)
```

`review-validate` 按 hunk 头的**旧文件**行号区间校验锚点，即 `@@ -361,13 +361,27 @@` 对应可锚定区间 `361..373`；本报告所有 finding 的 `line` 均落在该区间内。

## RISC-V 架构审核

`arch-scan` 机器判定：`{"archSpecific": false, "matches": []}`。补丁为纯 C++ 源码（模板内的整数索引运算与循环控制），**不含**汇编、inline-asm、RVV intrinsic 或 ISA 特性开关。按 Comet 规则跳过架构专项审核，仅做通用质量审核；`PatchCandidate.archReviewWarning` 已由工具自动写入对应警告。`archReview.status = not-applicable`。

## 审核结果

`reviewResult: pass`（4 条 finding 全为 `suggestion`，无 `critical` / `major`）。

### 约束保持逐条核对（对照蓝图 `constraints.mustPreserve`）

| 蓝图 mustPreserve 条目 | 核对结论 |
|---|---|
| `offset(k) == dot(unravel<5>(k, rshape), rstride)` 必须对全部 k ∈ [0, M) 逐点等价，含各维进位顺序（从最低维开始）与 k=0 时 coord 全 0 的初值 | **满足**。k=0 时 `coord` 显式清零、`offset = 0`，与原 `unravel(0, rshape) = 全 0`、`dot = 0` 一致；进位自 `i = ndim - 1`（最低维）向高维传播，与原 `unravel` 的 `for (i = ndim-1, j = idx; i >= 0; --i)` 分解顺序一致。穷举验证见下。 |
| 边界与退化输入：M == 0 时循环零次执行；rshape[i] == 0 不得引入除零；j + offset 不得越出 big 合法范围；offset 与回绕补偿量不得溢出 | **满足**。补丁**完全消除了除法与取余**（原 `unravel` 每维一次 `tmp = j / shape[i]`），故除零风险归零；M == 0 时 `for (size_t k = 0; k < M; ++k)` 零次执行，与原实现一致；`offset` 的取值域与原 `dot(coord, rstride)` 完全相同（同一组 `coord` 与 `rstride` 的线性组合），越界与溢出性质不变。 |
| 必须保留 `mshadow::red::sum` 的补偿求和递推与 `isinf(t) → residual = 0` 守卫，不得退化为朴素 vfredusum 求和 | **满足**。`Reducer::Reduce(val, temp, residual)` 调用点、实参、调用次数（每 k 一次）与顺序完全未变。 |
| 跨 lane 归并必须使用 `red::sum::Merge` 的同一 Neumaier 公式（含 isinf(t1) 分支） | **不适用/未触及**。补丁未改动 `use_omp` 分支的 `Reducer::Merge` 归并路径（`broadcast_reduce-inl.h:386` 附近）。 |
| 向量化会改变加法结合顺序…要求确定性时不得默认 unordered 归约等价 | **满足**。补丁未向量化归约，加法结合顺序逐元素保持不变 → 结果与打补丁前**位级一致**（bit-identical），优于蓝图的最低要求。 |
| `vfwcvt` 加宽与最终 `OType(val)` 收窄舍入须与标量 `fcvt.d.s`/`fcvt.s.d` 一致 | **不适用**。未引入任何浮点转换；`OType(val)` 收窄点未改动。 |
| k/offset 必须按实际 vl 推进…不得改变 `seq_reduce_compute` 对 idx 的 OMP 并行与 N >= thread_count 分派 | **满足**。补丁只改 `seq_reduce_assign` 的 `!use_omp` 内层循环体；`seq_reduce_compute`（`:437-459`）的 `N >= thread_count` 分派、`#pragma omp parallel for`、`seq_reduce_assign` 的调用签名与 `use_omp` 实参均未改动。 |
| 不得改变共享 `sum::Reduce` / `red::sum::Reduce` 的 volatile 形参在其他 reducer 路径上的既有语义 | **满足**。补丁未触及 `3rdparty/mshadow/mshadow/base.h` 与 `src/operator/mshadow_op.h`。 |
| 非 RVV 环境与非满足 short_cutoff 的输入必须保留标量 fallback | **满足**。补丁为纯标量代码，不含任何 ISA 条件编译；原有 fallback 结构未改动。 |

### 本补丁的验证证据（供追溯）

**A. 索引映射等价性（穷举 + 随机）**

补丁后源码的进位逻辑由脚本**机械提取**（`src/operator/tensor/broadcast_reduce-inl.h` 第 368–385 行，1-indexed），仅替换副作用语句（`OP::Map` → 记录 `j + offset`；`IndexOP::Op` / `Reducer::Reduce` → 空语句），**未手抄**任何算术表达式。参照实现为 `src/operator/mxnet_op.h:679-699` 的 `unravel` + `dot` 逐字语义。

编译 `gcc -O2` 运行，结果：

```
NDIM=1 NV=5 cases=30770     mismatches=0 -> EQUIVALENT
NDIM=2 NV=5 cases=151187    mismatches=0 -> EQUIVALENT
NDIM=3 NV=5 cases=782114    mismatches=0 -> EQUIVALENT
NDIM=4 NV=5 cases=4172270   mismatches=0 -> EQUIVALENT
NDIM=5 NV=4 cases=20571883  mismatches=0 -> EQUIVALENT
NDIM=6 NV=4 cases=107954325 mismatches=0 -> EQUIVALENT
```

覆盖面：`rshape` 各维穷举 `{1,2,3,5,7}`（NDIM≥5 用 `{1,2,3,5}`）；步长同时覆盖**行主序连续**（`rstride[i] = rstride[i+1]*rshape[i+1]`）与**非连续**（每维再乘 `i+2`，模拟大张量交错）两种方案；`j` 取 `{0, 1, 12345, -7}`；另含 2000 组随机 `rshape ∈ [1,9]^ndim` + 随机步长缩放，以及退化输入 `M == 0`、`M == 1`。合计 1.2 亿+ 组 `(k, offset)` 逐点比对，0 不一致。

**B. 未做的验证（如实声明）**

- 本环境**无法编译 mxnet**：`3rdparty/dmlc-core`、`dlpack`、`onednn` 子模块为空（0 文件），`g++ -fsyntax-only` 连头文件都不可达；且**无 RISC-V 工具链**，无 RVV 运行环境。因此**未做**端到端编译、未做 `perf annotate` 回归、未做数值对拍。这是本补丁验证的已知缺口（见 F4）。
- 补丁的收益（索引区间 ≈81.3% 局部样本份额）**未经实测确认**，只作方向性判断；`benefitUpperbound` 为 `cpu-clock` / `local period` 下的函数内局部份额，非 workload 级 Amdahl 上界。

## 发现问题

| id | 位置 | 严重度 | 类别 | 摘要 |
|---|---|---|---|---|
| F1 | `src/operator/tensor/broadcast_reduce-inl.h:372` | suggestion | 根因覆盖完整性 | 蓝图步骤②（RVV 加宽归约）未实施，且无载体可挂载 |
| F2 | `src/operator/tensor/broadcast_reduce-inl.h:371` | suggestion | 范围一致性 | `use_omp == true` 分支（`seq_reduce_assign_block`）仍保留 `unravel`/`dot` |
| F3 | `src/operator/tensor/broadcast_reduce-inl.h:368` | suggestion | 验证可追溯性 | 首轮 NDIM=2 报 1 mismatch 系验证脚手架越界写导致的假阳性 |
| F4 | `src/operator/tensor/broadcast_reduce-inl.h:364` | suggestion | 验证完备性 | 无端到端编译/数值对拍验证，建议补回归用例 |

详见同目录 `opencode-patch-review.json` 的 `findings[].evidence` / `suggestion`。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 技术精度 | PASS | 补丁为纯整数索引运算，无浮点/指令级声明；报告未声称任何 RVV 指令已落地；`arch-scan` 判定已由工具复核。 |
| 声明溯源 | PASS | 所有 finding 的 `file` 与 `line` 均可锚定 `patch.diff` 的 hunk（新行号区间 361..387）；蓝图引文逐字取自 `bp-mxnet-category-nn-basic-003` 的 `diagnosis.recommendedFirstAction` 与 `constraints.mustPreserve`。 |
| 可解释性 | PASS | 「索引强度削减」的机制、进位公式与等价性证明路径可复现（提取脚本 + 对照实现 + 穷举结果均已记录）。 |
| 内部一致性 | PASS | `reviewResult: pass` 与 findings 严重度一致（全 `suggestion`，无 `critical`/`major`）；`archReview.status: not-applicable` 与 `arch-scan` 输出一致；`patchCandidateId` / `patchPlanId` / `blueprintId` 与产物文件一致。 |
| 安全 | PASS | 未修改函数语义；未引入除零（反而消除除法）；未改动 OMP 结构、归约算法、volatile 语义或任何公共接口；未触碰 `.yuansheng/` 之外的非目标文件。 |

## 结论

**通过（pass）**。补丁严格实施了蓝图 `recommendedFirstAction` 的步骤①，改动聚焦单文件单 hunk，逐条满足 `constraints.mustPreserve`，并以 1.2 亿+ 组穷举/随机比对证明 `offset(k) == dot(unravel(k, rshape), rstride)` 在 `ndim ∈ [1,6]` 下逐点等价，且因未改变加法结合顺序而保证结果位级一致。

四条 finding 均为 `suggestion`：F1/F2 记录了本轮刻意未覆盖的范围（RVV 步骤②、`use_omp == true` 分支），F3/F4 记录了验证脚手架的假阳性溯源与端到端验证缺口。均不构成阻断。
