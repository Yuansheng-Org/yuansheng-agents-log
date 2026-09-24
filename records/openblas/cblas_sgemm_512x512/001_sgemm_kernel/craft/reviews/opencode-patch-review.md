# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/openblas/cblas_sgemm_512x512/001_sgemm_kernel/blueprint_openblas_cblas_sgemm_512x512_001.json`（`blueprintId=bp-openblas-cblas_sgemm_512x512-001`，`finalStatus=confirmed_root_cause`，`overallConfidence=0.82`，`recommendToCraft=conditional`，`needsHumanReview=true`）
- 诊断日志：`.yuansheng/trace/openblas/cblas_sgemm_512x512/001_sgemm_kernel/diaglog_openblas_cblas_sgemm_512x512_001.md`
- 补丁计划：`.yuansheng/craft/openblas/001_sgemm_kernel/craft/patch-plan.json`
- 候选补丁：`.yuansheng/craft/openblas/001_sgemm_kernel/craft/patch-candidate.json`
- 差异：`.yuansheng/craft/openblas/001_sgemm_kernel/craft/patch.diff`（单文件 `kernel/riscv64/sgemm_kernel_16x8_zvl256b.c`，hunk `@@ -2170,7 +2170,57 @@`）
- 目标硬件：SpacemiT X100（K3，RVV 1.0，VLEN=256，OoO），CORE=RISCV64_ZVL256B

审核为独立只读审核：结论只来自上述落盘事实来源，不依赖实现会话记忆；未修改任何代码。

## 事实核对

1. **文件映射核对**：蓝图 `rootCause.candidateFiles` 记为 `kernel/riscv64/sgemm_kernel.c`，但 `diagnosis.currentGaps` 已声明该路径为“按 OpenBLAS commit 70c2410f 的 riscv64 intrinsic 微内核结构推断”。仓库实际 `kernel/riscv64/KERNEL.RISCV64_ZVL256B:98` 将 `SGEMMKERNEL` 绑定为 `sgemm_kernel_$(SGEMM_UNROLL_M)x$(SGEMM_UNROLL_N)_zvl256b.c`，`SGEMM_UNROLL_M=16`、`SGEMM_UNROLL_N=8` → `sgemm_kernel_16x8_zvl256b.c`。`001-sgemm_kernel-annotate.txt` 的源码行号 2159–2194 与该文件逐行吻合，确认补丁落在真实热点文件。
2. **根因证据核对**：annotate 热区间 `12f46`–`12f94`（k-loop）含 1×`vle32.v`、8×`flw`、8×`vfmacc.vf`、2×`addi`、1×`bnez`，与文件 2173–2194 行源码一一对应；局部样本 93.43%（flw 17.60% + vfmacc.vf 68.14% + addi 7.69%），与蓝图 `problem.evidence` 一致。
3. **语义等价核对**：修改前 k-step 序列为「k=0 由 2151–2171 初始化块处理，k=1..K-1 由 `for(--k)` 处理」，共 K 步；修改后为「初始化块（k=0）+ `for(k=K; k>2; k-=2)` 双步 + `for(;--k;)` 单步尾」，对 K=1..5 逐一枚举：K=1→0+0+0、K=2→0+0+1、K=3→0+1+0、K=4→0+1+1、K=5→0+2+0，总步数均等于 K-1，且每个累加器（result01..resultEF）内 k-step 的消耗顺序仍为升序，FP 累加链顺序不变。
4. **不变项核对**：diff 未触碰 `M_TAIL`/edge 路径、alpha/beta/C epilogue、`ldc` 语义、`vsetivli` 建立点、m2/8 累加器结构；B 仍以 8 个连续标量读入，packed 布局合同不变。

## RISC-V 架构审核

架构检测：`arch-scan` 判定 `archSpecific: true`（命中 `riscv-intrinsic`），故执行 RISC-V 专项审核。

1. **指令 ISA 合规**：`vle32.v`、`vfmacc.vf`、`vsetivli`、`flw` 均属 RVV 1.0 / RV64GC 基础；未引入新 intrinsic，仅复用文件既有 `__riscv_vle32_v_f32m2`/`__riscv_vfmacc_vf_f32m2`。
2. **LMUL/SEW 一致性**：`vfloat32m2_t`、`sew=32`、`vl=16`（`vsetivli zero,16,e32,m2`）均未变；新增 `A0/A1` 两条 m2 与 8 个 m2 累加器同时活跃 → 10×m2=20 个向量寄存器，在 32 寄存器预算内。`-O3 -S` 反汇编确认 `vse32.v ...,(sp)` 向量 spill 计数与原文件同为 1（该既有 spill 位于其他分支，非本补丁引入）。
3. **vsetvl/vtype 一致性**：紧循环内未新增 `vsetvl`/`vsetivli`（loop 外 12ec2 单次建立），LMUL/SEW 与数据流一致。
4. **寄存器预算/无 spill**：`-O3 -S` 展开体为 16×flw + 2×vle32 + 16×vfmacc + 4 条控制（约 38 条/2 k-step，对比原 40 条/2 k-step），未产生新向量 spill。
5. **运行时 ISA 分发**：未修改 `ifunc`/`hwprobe` 或 `KERNEL.RISCV64_ZVL256B` 分发表，静态绑定不变。
6. **无跨架构指令混入**：diff 仅含 RVV intrinsic、标量 load 与 C 结构。

编译自检：`riscv64-linux-gnu-gcc -march=rv64imafdcv_zvl256b -mabi=lp64d -I/tmp/obcfg -I. -DBUILD_KERNEL -DRISCV64_ZVL256B -D<NN|NT|TN|TT|NR|NC|TR|TC|RN|RT|CN|CT|RR|RC|CR|CC> -fsyntax-only kernel/riscv64/sgemm_kernel_16x8_zvl256b.c`，16 个转置宏全部通过。

## 审核结果

`pass`

## 发现问题

- **minor / performance / kernel/riscv64/sgemm_kernel_16x8_zvl256b.c:2173**：收益依赖 X100 的 FMA 发射带宽与 OoO 重叠。hunk `@@ -2170,7 +2170,57 @@` 中新增 `+            for (k = K; k > 2; k -= 2) {`、`+                vfloat32m2_t A0 = __riscv_vle32_v_f32m2( A, 16 );`、`+                vfloat32m2_t A1 = __riscv_vle32_v_f32m2( A + 16, 16 );`。蓝图 `diagnosis.alternativeExplanations` 明示候选 A 收益可能仅 ~5%，且 `currentGaps` 无 X100 PMU/latency 模型；蓝图 `problem` 中 ~84% datapath 占用为推断值。此为收益不确定性，非语义错误，故列 minor 而非 major。建议重建后在 X100 上重跑 `cblas_sgemm.goto 512 512 1` 并采集 perf stat cycles/instructions 与 annotate，核对 k-loop 每迭代由 ~19.1 降至 ~16–17 cycles、flw/addi/bnez 局部样本变小、loop 内无新增 vsetvli/spill。
- **minor / verification / kernel/riscv64/sgemm_kernel_16x8_zvl256b.c:2174**：补丁只覆盖 M/16 主 pass，未覆盖 M 非 16 倍数（含 `M_TAIL_ONE` 的 M<=2/3/<=8 路径，如 1960 行同类 `for(--k)`）。蓝图 `constraints.mustPreserve` 要求 M 非 16 倍数与 N 非 8 倍数 edge/tail 正确性。本补丁刻意不改 tail（避免扩大风险面），但因此主/尾路径指令形态不一致；无 X100 硬件时无法运行 M%16≠0 的数值回归，已如实披露。建议后续实机对 M%16 与 N%8 edge shapes 做回归与 FP 数值对照。
- **suggestion / scope / kernel/riscv64/sgemm_kernel_16x8_zvl256b.c:2170**：蓝图候选 B（32 行 m-block + m4 + 4 列/趟共享 A）可把每 FMA 的指令数降得更多，但需重构 epilogue 并处理 M 非 32 倍数，风险高于本补丁。本补丁采用候选 A 作最小改动，候选 B 建议作为独立后续项在实机 A/B 后再评估。

## 幻觉自检

- [PASS] 技术精度：ISA/LMUL/寄存器预算/无新增 spill 等结论均引用本会话实际执行的 `-fsyntax-only`（16 宏）与 `-O3 -S`（spill 计数对照）命令结果、`KERNEL.RISCV64_ZVL256B:98` 与 annotate 原文；未凭知识断言 X100 微架构细节，收益不确定性显式降级为 minor。
- [PASS] 声明溯源：全部 finding 的 `file` 均在 patch.diff 中，`line` 分别为 2173/2174/2170，均落在 hunk `@@ -2170,7 +2170,57 @@` 的旧行范围 2170–2177 内；evidence 引用 diff 原文与蓝图具体字段（`diagnosis.currentGaps`、`diagnosis.alternativeExplanations`、`constraints.mustPreserve`、`problem.evidence`）。
- [PASS] 可解释性：minor 说明“为什么必须实机复核”（收益依赖发射带宽、蓝图置信区间宽），suggestion 说明“为什么范围取舍”（候选 B 需重构 epilogue，风险更高）。
- [PASS] 内部一致性：`reviewResult=pass`，findings 仅含 minor/suggestion，无 critical/major；`patchCandidateId` 与 candidate 工具产物一致；K=1..5 步数枚举与源码一致。
- [PASS] 安全：补丁不引入硬编码密钥、危险命令、越权行为或新攻击面；仅本地 kernel 调度改写。

## 结论

补丁聚焦 `kernel/riscv64/sgemm_kernel_16x8_zvl256b.c` 的 M/16 主 pass k-loop，以 2× K-unroll + A 双缓冲削减单位 FMA 的循环控制指令并暴露 A load 延迟，直接回应蓝图根因（每 k-iter 12 条非 FMA 指令未隐藏）。每个累加器的 K 累加顺序与 FP 语义不变；m2/16 行/8 累加器结构、packed 布局合同、epilogue 与 edge/tail 路径均未改动。16 个转置宏语法编译通过，`-O3 -S` 确认无新增向量 spill。剩余风险（实机收益仅 ~5% 上界、M/N 尾块未回归、候选 B 未落地）已如实披露。审核通过。
