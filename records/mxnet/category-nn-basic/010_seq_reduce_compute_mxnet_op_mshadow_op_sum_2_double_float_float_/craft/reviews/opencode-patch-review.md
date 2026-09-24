# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-nn-basic-010-r1`
- patchCandidateId: `pc-bp-mxnet-category-nn-basic-010`
- reviewer: `opencode-comet-reviewer`（独立只读审核）
- reviewedAt: 2026-09-21
- 审核依据：`.yuansheng/craft/.comet.yaml` 记录的 `root_cause_blueprint`、同目录 `diaglog_mxnet_category-nn-basic_010.md`、`craft/patch-plan.json`、`craft/patch.diff`、`craft/patch-candidate.json`

## 审核范围

- 蓝图：`bp-mxnet-category-nn-basic-010`（`recommendToCraft: yes`，`finalStatus: probable_root_cause`，`overallConfidence: 0.85`）
- 热点函数：`mxnet::op::broadcast::seq_reduce_compute<mshadow_op::sum, 2, double, float, float, square, set_index_no_op<double,long>> [._omp_fn.0]`
- 补丁范围：`src/operator/mshadow_op.h`（+12 / −6，单文件）
- 本轮为**通用质量审核**（架构专项按机器判定跳过，见下节）

## RISC-V 架构审核

- 机器判定：`arch-scan craft/patch.diff` → `{"archSpecific": false, "matches": []}`
- 按规则**跳过** RISC-V 架构专项审核（无 `vsetvl`/`vtype`/LMUL/SEW/tail-mask/ISA 分发可审）。`PatchCandidate` 首字段 `archReviewWarning` 已由工具自动写入对应警告。
- `archReview.status = not-applicable`。

## 审核结果

**`reviewResult: pass`**（无 critical / major finding）

| # | 审核项 | 结论 | 依据 |
|---|---|---|---|
| 1 | 根因解决 | 通过（主因机制解决） | 补丁命中蓝图 primary finding 所指的「`volatile` 迫使双精度累加器 `val` 与补偿项 `residual` 每轮栈往返（占 47.62% 局部样本）」 |
| 2 | 约束保持 | 通过 | 见下方逐条核对 |
| 3 | 最小性 | 通过 | 单文件 +12/−6；FP 运算序列逐字未改；无无关重构 |
| 4 | 产物一致性 | 通过 | `patch-candidate.json` 的 `gitDiff` 与 `patch.diff` 逐字节一致；`changedFiles = ["src/operator/mshadow_op.h"]` 与 diff 一致 |
| 5 | 安全 | 通过 | 无硬编码密钥、无危险命令/路径 |

### 约束保持逐条核对（对照蓝图 `constraints.mustPreserve`）

| mustPreserve 条目 | 结论 | 依据 |
|---|---|---|
| 归约数值语义：必须保持「先在 f32 精度下平方，再 widening 到 f64 累加」的顺序 | 通过 | 补丁未触及 `sqr`/`OP::Map` 与 `fcvt.d.s` 的语义；`Reduce` 内 FP 序列 `y=src-residual; t=dst+y; residual=(t-dst)-y; dst=t;` 在 diff 中逐字未变 |
| 稳定算法语义：补偿求和结构不得替换为朴素 `sum(x*x)` | 通过 | 补偿项 `residual` 的更新表达式逐字保留；补丁仅移除形参的 `volatile` 限定，未改变算法结构 |
| 函数签名与公共 API 不变 | 通过 | 模板参数序列、函数名、返回类型均未变；仅移除参数的 cv 限定（兼容性见 F3） |
| 空输入与 tail 语义不变 | 通过 | 补丁未触及 `SetInitValue`、`Finalize` 与循环边界；`M==0` 仍返回初值 0 |
| 长期 accumulator 不得用最后一次 `vl` 做最终归约 | 通过（N/A） | 补丁未引入任何向量归约 |

### 本补丁的验证证据（供追溯）

1. **实例化编译测试**：把改动后的 `mshadow_op::sum` 结构体逐字抽取为自包含 TU，用真实 C++ 编译器（g++ 15.3.0）以与 `src/operator/tensor/broadcast_reduce-inl.h:337/370/386` 相同的**非 volatile 实参形态**实例化 `Reduce`（2/3 参，double 与 float）、`Merge`（2/4 参）、`Finalize`、`SetInitValue`、`PartialGrad` → 编译通过（exit=0），仅报出 `Finalize` 空函数体的 unused-parameter 既有警告。
2. **数值等价性**：把 volatile 原版与非 volatile 新版并排执行，覆盖 20,001 组输入（每组 1–512 元素，含 NaN/±Inf/±0/subnormal/3.4e38/1.8e19 极值）→ **0 处不一致（BIT-IDENTICAL）**。
3. **调用点核查**：全仓搜索确认**无任何调用点**把 `volatile` 左值传给 `mshadow_op::sum::Reduce/Merge`；`src/common/cuda/rtc/reducer-inl.h` 中的同名 volatile 重载属 CUDA 设备端**另一个结构体**，与本次改动无关。

## 发现问题

| id | severity | file | category | 摘要 |
|---|---|---|---|---|
| F1 | suggestion | `src/operator/mshadow_op.h` | 根因覆盖完整性 | 蓝图步骤②（分块 RVV 归约）未覆盖；仓库零 RVV 基础设施 |
| F2 | suggestion | `src/operator/mshadow_op.h` | 一致性 | `Finalize` 与 4 参 `Merge` 仍保留 `volatile`（按最小性刻意未改） |
| F3 | suggestion | `src/operator/mshadow_op.h` | 可移植性 | 位一致性论证依赖 `FLT_EVAL_METHOD == 0` |

三条均为 `suggestion`：无证据支撑升级为 critical/major；据此 `reviewResult` 与 findings 严重度一致。

## 幻觉自检

- [PASS] **技术精度**：本报告不含 RISC-V 架构结论（`archSpecific: false` 由机器判定）；编译与数值等价性结论均以真实编译器与可执行测试为据。
- [PASS] **声明溯源**：F1/F2/F3 的 `evidence` 分别引用 `patch.diff` hunk 原文与蓝图具体字段（`recommendedFirstAction` 的步骤②文本、`diagnosis.metricEvidence` 的 47.62%、`constraints.mustPreserve` 条目），未引用模型知识。
- [PASS] **可解释性**：三条 finding 均给出「为什么」与可执行建议。
- [PASS] **内部一致性**：`reviewResult: pass` 且最高严重度为 `suggestion`。
- [PASS] **安全**：无越权建议，无新风险引入。

## 结论

补丁精确命中蓝图 primary finding 的根因：`mshadow_op::sum::Reduce` 的 `volatile` 形参迫使双精度累加器与补偿项每轮经栈往返，移除后二者保持寄存器驻留，直接针对 47.62% 的 `fld/fsd` 栈往返份额。变更最小（单文件 +12/−6）、FP 运算序列逐字未变、数值经 20,001 组含极值输入的逐位比对确认等价、实例化编译通过、无 volatile 左值调用点。产物一致，安全项通过。

蓝图步骤②（分块 RVV 归约）未覆盖；经复核 mxnet 仓库不存在 RVV 向量数学管线（`__riscv_v` / `riscv_vector.h` / `MSHADOW_USE_RVV` / `vsetvl` 均为 0 文件，仅 `MSHADOW_USE_SSE` 存在），实现它属新增 RVV 后端而非最小修复，已作为 F1 记录并建议独立立项。

**审核通过（`pass`）**，建议进入终态 `done`；三条 suggestion 不阻断流转。
