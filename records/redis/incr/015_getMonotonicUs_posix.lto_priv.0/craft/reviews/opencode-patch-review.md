# AI 补丁审核报告

## 审核范围

- 蓝图：`bp-redis-incr-015`（`getMonotonicUs_posix.lto_priv.0`，incr 批次 rank 015 热点）
- PatchPlan：`pp-bp-redis-incr-015`
- PatchCandidate：`pc-bp-redis-incr-015`
- 变更文件：`src/Makefile`
- 根因类型：`config_mismatch`（`USE_PROCESSOR_CLOCK` 宏未定义，RISC-V mtime 快速路径未编译，dispatch 落入 POSIX 回退）

## 审核结果

- `reviewResult`: pass
- 发现问题：critical 0 / major 0 / minor 0 / suggestion 1

## 根因解决

补丁准确解决 blueprint 指出的根因。blueprint `rootCause.summary` 为：`USE_PROCESSOR_CLOCK` 宏未定义，使 `monotonic.c` 中 RISC-V mtime 快速路径（`csrr time`）未编译、`monotonicInit` dispatch 落入 `getMonotonicUs_posix`，每事件循环迭代多次的时钟读取每次付一次完整 `clock_gettime` syscall。

补丁在 `src/Makefile` 为 riscv64 追加 `-DUSE_PROCESSOR_CLOCK`，使 `monotonic.c:141` 的 `#if defined(USE_PROCESSOR_CLOCK) && defined(__riscv) && defined(__linux__)` 成立，RISC-V mtime 路径（`monotonic.c:141-194`）被编译，`monotonicInit`（`monotonic.c:230-232`）选中 `monotonicInit_riscv`，从而消除 syscall 回退。

## 约束保持

对照 `constraints.mustPreserve` 逐项核验：

- **monotime 语义（微秒、单调不回退）**：`getMonotonicUs_riscv` 返回 `read_mtime() / mono_ticksPerMicrosecond`，保持微秒单调语义，未改变。
- **mono_ticksPerMicrosecond 校准 >0 否则回退**：`monotonicInit_riscv`（`monotonic.c:184-193`）已有 `if (mono_ticksPerMicrosecond == 0) return;` 回退逻辑，未被破坏。
- **mtime 单调性由 Zicntr 保证**：补丁未改动 `read_mtime` 的 `csrr time` 序列，仍由 Zicntr 架构保证单调性。
- **x86 构建行为一致**：补丁仅在 `findstring riscv,$(uname_M)` 条件分支下追加 `-DUSE_PROCESSOR_CLOCK`，x86 构建（`uname_M=x86_64`）不命中该分支，TSC 路径行为与现状一致。

## RISC-V 架构审核

- **ISA 内特性**：启用路径依赖的 `csrr time` 由 Zicntr 支持。blueprint `problem.evidence` 明确记录 "hardware isa 含 zicntr（用户态 time CSR）；build 含 zicsr2p0（csrr 可用）"，证据充分，未凭模型知识断言。
- **无向量边界问题**：不涉及 `vsetvl`/`vtype`/LMUL/SEW/tail/mask。
- **无运行时分发问题**：为编译期宏门控，非 ifunc/hwprobe 运行时分发。
- **未混入异架构指令**：`-DUSE_PROCESSOR_CLOCK` 为通用宏，仅 riscv64 分支追加，不影响 x86/ARM。
- **架构结论溯源**：`csrr time` 可用性结论均来自 blueprint evidence（zicntr / zicsr2p0），未臆造。

## 发现问题

- `suggestion` `sugg-monotonic-dt-fallback`：blueprint 建议为 `get_timebase_frequency` 增加 riscv_hwprobe/auxv 兜底以应对 DT 文件缺失，本补丁未实现。此为有意的最小化取舍：RISC-V `riscv_hwprobe` 无 timebase-frequency key，hwprobe/auxv 兜底在技术上不可行；且 `monotonicInit_riscv` 已具备 `mono_ticksPerMicrosecond==0` 时回退 POSIX 的安全语义（满足 mustPreserve）。不构成必须修改的问题。

## 幻觉自检

- **技术精度**：`[PASS]` `csrr time`/Zicntr 可用性结论均有 blueprint evidence（zicntr、zicsr2p0）支撑。
- **声明溯源**：`[PASS]` suggestion finding 的 evidence 引用 blueprint `recommendedFirstAction` 字段原文，可溯源。
- **可解释性**：`[PASS]` 无 critical/major finding；suggestion 已说明为何不做 hwprobe 兜底。
- **内部一致性**：`[PASS]` reviewResult=pass，findings 仅含 suggestion，无 critical/major，一致。
- **安全**：`[PASS]` 无越权建议、无引入新风险。

## 结论

补丁准确解决根因、严格遵守 `mustPreserve`、变更最小（仅 Makefile 一处），通过 RISC-V 架构专项审核与防幻觉自检，判定 **pass**。
