# AI 补丁审核报告

## 审核范围
- 软件 / 测试用例：openblas / sger_2048x2048
- 热点函数：`saxpy_k`（rank 002）
- 根因蓝图：`.yuansheng/trace/openblas/sger_2048x2048/002_saxpy_k/blueprint_openblas_sger_2048x2048_002.json`
- 补丁计划：`.yuansheng/craft/openblas/002_saxpy_k/craft/patch-plan.json`
- 补丁差异：`.yuansheng/craft/openblas/002_saxpy_k/craft/patch.diff`
- 候选补丁：`.yuansheng/craft/openblas/002_saxpy_k/craft/patch-candidate.json`（`pc-bp-openblas-sger-2048x2048-002`）
- 变更文件：`kernel/riscv64/axpy_vector.c`（`SAXPYKERNEL`）
- 审核类型：独立只读，基于落盘事实来源；未修改源码

## RISC-V 架构审核
机器判定 `arch-scan`：`archSpecific: false`（仅一行 `#define LMUL m4`），跳过架构专项审核，`archReview.status = "not-applicable"`，仅做通用质量审核。

## 审核结果
**pass**（通用审核通过；架构专项 not-applicable；0 critical / 0 major）

## 发现问题
- `F1`（suggestion，性能/验证）：`kernel/riscv64/axpy_vector.c` — ZVL256B 分支 LMUL 由 m2 提升为 m4（e32 路径每向量 16 floats），主循环 unroll=2 峰值 live ≈ 4×m4 = 16 reg ≤ 32，无 spill；实际收益与 m8 候选取舍须 X100 A/B 实测。证据：diff `-#       define LMUL m2` / `+#       define LMUL m4` 与 blueprint `recommendedFirstAction`。
- `F2`（suggestion，范围）：改动仅一行，其余分支与 tail 结构未变。

**根因解决核对**：diff 将 `SAXPYKERNEL = axpy_vector.c` 的 `RISCV64_ZVL256B` 分支 LMUL 从 m2 提升到合法 m4 frontier，减少迭代数、标量地址算术与循环控制开销（蓝图诊断样本中约 45.45% 为标量地址算术）；与 blueprint `recommendedFirstAction`（"将 e32 路径 LMUL 由 m2 提升为 m4/m8 候选"）一致。**已解决**。

**约束保持核对**：
- 数值输出：AXPY 逐元素 `y[i] += da*x[i]`，LMUL 变化只改变每次迭代覆盖元素数与循环次数，**逐位一致**。
- 公共 API / fallback：接口、`n<=0`/`da==0` 早退、各 inc 分支与 tail 逻辑未改。
- 编译：`TARGET=RISCV64_ZVL256B` 完整构建成功，无新增 error/warning。

**产物一致性核对**：`changedFiles` 与 diff 一致；`gitDiff` 与 `patch.diff` 字节一致。

**安全核对**：无硬编码密钥、无危险命令/路径。

## 幻觉自检
- 技术精度：`[PASS]` — LMUL/寄存器预算结论有 diff 与 blueprint 证据。
- 声明溯源：`[PASS]` — findings 的 evidence 引用 diff 原文与 blueprint 字段。
- 可解释性：`[PASS]` — 无 critical/major；suggestion 说明性质。
- 内部一致性：`[PASS]` — `reviewResult: pass` 与 findings 严重度一致。
- 安全：`[PASS]` — 无越权建议。

## 结论
补丁按蓝图将 `saxpy_k` 所在 `axpy_vector.c` 的 ZVL256B 分支 LMUL 提升至合法 m4，逐元素运算不变（逐位一致），编译通过，范围最小聚焦。**审核通过（pass）**。遗留项：X100 实机收益与 m8 候选取舍待 A/B 实测。
