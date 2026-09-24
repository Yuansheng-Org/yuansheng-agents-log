# AI 补丁审核报告

- **审核对象**: bp-postgresql-select_only_clients_16-024 (PinBuffer)
- **审核人**: AI Reviewer (comet-review)
- **日期**: 2026-09-15

## 审核范围

通用质量审核 + RISC-V 架构专项审核（arch-scan 判定 archSpecific=true，含 `__riscv` 宏判定）

## RISC-V 架构审核

1. **指令/特性是否在目标硬件 ISA 内** — PASS
   - 目标硬件（blueprint.source.targetHardware）：XuanTie C920v2 (SOPHGO SG2044)，RV64
   - blueprint `problem.evidence` 与 `rootCause` 明确：RV64 对齐 8 字节 load 单拷贝原子性为 RISC-V ISA 保证（ld 指令），`lr.d/sc.d` 为过度实现
   - 交叉编译器实测：RV64 下 `pg_atomic_read_u64_impl` 编译为单条 `ld a0,0(a0)` + `ret`，正是期望的快速路径
2. **vsetvl/vtype 配置与数据流一致** — N/A（无向量指令）
3. **tail/mask policy、标量↔向量边界** — N/A
4. **静态检查运行时 ISA 分发路径自洽** — PASS
   - 编译期 `#if defined(__riscv) && __riscv_xlen == 64` 宏判定；RV32（__riscv_xlen=32）时条件不成立，保持 generic.h CAS-with-0 回退，符合 blueprint constraints
5. **未混入 x86/ARM 专属指令** — PASS

**架构结论证据来源**：blueprint problem.evidence（pg_atomic_read_u64 被编译为 lr.d.aqrl + sc.d.rl）与 rootCause（RV64 单拷贝原子性）一致；编译器实测佐证。

## 审核结果: PASS

## 通用审核清单

### 1. 根因解决 — PASS

Blueprint 根因：RISC-V 未定义 `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY`（atomics.h 无 `__riscv` arch 分支、无 arch-riscv.h），导致 `pg_atomic_read_u64()` 走 generic.h 的 CAS-with-0 回退（`lr.d.aqrl` + `sc.d.rl` 循环）而非单条 `ld`。

补丁措施：
- `atomics.h`：arch 头选择分支增加 `#elif defined(__riscv)` → `#include "port/atomics/arch-riscv.h"`
- 新建 `arch-riscv.h`：`__riscv_xlen == 64` 时定义 `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY`，使 generic.h 310-317 的 `return ptr->value;` 快速路径生效

与 blueprint `recommendedFirstAction` 完全一致。交叉编译器实测确认 `ld` 快速路径生成。

### 2. 约束保持 — PASS

对照 blueprint `constraints.mustPreserve`：

| 约束 | 结论 |
|------|------|
| pg_atomic_read_u64 的『无屏障语义』合约与单拷贝原子性 | PASS：RV64 对齐 8 字节 load 本身单拷贝原子（RISC-V ISA 保证），`return ptr->value` 无屏障语义 |
| PinBuffer 的 refcount/usagecount 语义与 BM_VALID 返回值 | PASS：仅底层读原语 lowering 变化，调用点语义不变 |
| pg_atomic_compare_exchange_u64 的 seq_cst CAS 路径 | PASS：未修改 CAS 实现，refcount CAS loop 仍 seq_cst |
| PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY 仅在 __riscv_xlen==64 定义 | PASS：`#if __riscv_xlen == 64` 编译期守卫，RV32 保持回退 |

### 3. 范围合理性 — PASS

变更文件 2 个：`atomics.h`（arch 分支挂接）+ 新建 `arch-riscv.h`（单拷贝原子性声明）。聚焦、最小。

### 4. 产物一致性 — PASS

PatchCandidate.gitDiff 与 patch.diff 字节一致。

### 5. 安全 — PASS

无硬编码密钥、无危险命令/路径。

## 发现问题

无 critical/major finding。以下为建议（suggestion 级）：

- **S-1**（suggestion，`arch-riscv.h`）：本修复只覆盖 `pg_atomic_read_u64` 的读回退；`pg_atomic_write_u64` 等其他 64 位操作若仍有性能敏感调用点，可后续评估类似 lowering（不在本蓝图范围，不扩大补丁）。
- **S-2**（suggestion，`atomics.h`）：目标机重建后 perf annotate 确认 PinBuffer 内 `lr.d.aqrl`/`sc.d.rl zero` 消失、出现单条 `ld`；并跑 make check 与 buffer/lmgr 并发回归。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度：架构相关结论有 diaglog/blueprint 证据支撑 | [PASS] |
| 声明溯源：每条 finding 的 evidence 来自 diff/blueprint 原文 | [PASS] |
| 可解释性：critical/major 说明"为什么必须改" | [PASS]（无 critical/major，N/A） |
| 内部一致性：reviewResult 与 findings 严重度一致 | [PASS]（pass ↔ 无 critical/major） |
| 安全：无越权建议、无引入新风险的建议 | [PASS] |

## 结论

补丁准确解决 PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY 缺失的 arch_gap 根因，RV64 单拷贝原子性由 ISA 保证且经交叉编译器实测（单条 ld），RV32 回退保留，约束全部保持，范围聚焦。RISC-V 架构专项审核通过。整体审核通过（pass）。
