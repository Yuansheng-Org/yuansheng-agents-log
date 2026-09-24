# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/openblas/cblas_zgemm_512x512/001_zgemm_kernel_n/blueprint_openblas_cblas_zgemm_512x512_001.json`（`blueprintId=bp-openblas-cblas_zgemm_512x512-001`，`finalStatus=confirmed_root_cause`，`overallConfidence=0.8`，`recommendToCraft=yes`）
- 诊断日志：`.yuansheng/trace/openblas/cblas_zgemm_512x512/001_zgemm_kernel_n/diaglog_openblas_cblas_zgemm_512x512_001.md`
- 补丁计划：`.yuansheng/craft/openblas/001_zgemm_kernel_n/craft/patch-plan.json`
- 候选补丁：`.yuansheng/craft/openblas/001_zgemm_kernel_n/craft/patch-candidate.json`
- 差异：`.yuansheng/craft/openblas/001_zgemm_kernel_n/craft/patch.diff`（单文件 `kernel/riscv64/zgemm_kernel_8x4_zvl256b.c`，hunk `@@ -49,6 +49,8 @@`、`@@ -57,6 +59,8 @@`、`@@ -65,6 +69,8 @@`、`@@ -73,6 +79,8 @@`、`@@ -172,60 +180,57 @@`）
- 目标硬件：SpacemiT X100（K3，RVV 1.0，VLEN=256，OoO），CORE=RISCV64_ZVL256B

审核为独立只读审核：结论只来自上述落盘事实来源与本会话实际执行的交叉编译/仿真命令；未修改任何代码。

## 事实核对

1. **文件映射核对**：蓝图 `rootCause.candidateFiles` 记为 `kernel/riscv64/zgemm_kernel_n.c`，但 `diagnosis.currentGaps` 声明为“无 debug symbols、路径推断”。仓库实际 `kernel/riscv64/KERNEL.RISCV64_ZVL256B:153` 绑定 `ZGEMMKERNEL = zgemm_kernel_$(ZGEMM_UNROLL_M)x$(ZGEMM_UNROLL_N)_zvl256b.c`（本 CORE 8x4）；`001-zgemm_kernel_n-annotate.txt` 的源码行 175–228 与该文件（修改前）逐行吻合，热点 k-loop 位于 M/8 主 pass。补丁落在真实文件。
2. **根因证据核对**：annotate 显示 k-loop 每 k-step 含 16×`vfmul.vf`、16×`vfmacc/vfmsac.vf`、16×`vfadd.vv` 与 4×`vlse64.v`（`17ba0` A1i gather 单行 8.37%）。补丁后交叉编译 `-O3 -S` 计数：`vfadd.vv` 56→40、`vfmul.vf` 112→96、FMA 族 224→240、`vlse64.v` 104→100、`vlseg2e64` +2、向量 spill 0。修改的 M/8 主 loop 每 k-step 向量 FP 由 48 降到 32（16 vfmul+16 FMA+16 vfadd → 32 FMA），与蓝图 `codeKnowledgeEvidence`/`recommendedFirstAction` 一致。
3. **语义等价核对（关键）**：本审核独立构造 qemu-riscv64 交叉验证：将修改前（`git show HEAD:...`）与修改后文件分别以同一转置宏、`-DDOUBLE` 编译为独立目标，同一随机 packed A/B/C 输入调用，逐元素比较 C。**16 个转置宏（NN/NT/TN/TT/NR/NC/TR/TC/RN/RT/CN/CT/RR/RC/CR/CC）在 M∈{8,16,24,12,6,3}、K∈{1,2,5,7,3,9}（含 tail 形状）全部 bit-exact（diff=0）**。
4. **折叠正确性代数核对**：`vfmsac(x,u,v)=u*v−x` 且 `vfmacc(x,u,v)=u*v+x`（已用 qemu 实测确认：vfmacc→16、vfmsac→−4、vfnmacc→−16、vfnmsac→4，acc10/s2/v3）。故实部同一宏连用两次：`op(op(ACC,Bi,Ai),Br,Ar)=ACC+Br*ArAi*Bi`，严格等于原 `tmp=vfmul; tmp=VFMACC_RR(tmp); ACC+=tmp`。虚部按组取 FOLD_OPA/OPB 使 `ACC+σB*Br*Ai+σA*Ar*Bi` 严格等于原 `tmp=vfmul(Ar,Bi); tmp=VFMACC_RI(tmp,Br,Ai); ACC+=tmp`（σB/σA 由各组 VFMACC_RI 类型唯一决定）。特别地，本补丁**未沿用**既有 `001_cgemm_kernel_n` 补丁的折叠形式，因为该形式（`vfmacc(ACC,Bi,Ai)` 后 `VFMACC_RR(ACC,Br,Ar)`）在 VFMACC_RR=vfmsac 的分支会把 ACC 符号翻负；本审核用 qemu 实测确认该形式对 K>1 不等价（diff≠0），故本补丁改用经实测等价的形式。
5. **不变项核对**：diff 未触碰 tail 路径（M&4/M&2/M&1、N&2/N&1）、epilogue（alpha/beta/C stride）、A/B/C packing、`KERNEL.RISCV64_ZVL256B` 分发表；仅改 4 个转置宏块的宏定义与 M/8 主 pass k-loop。

## RISC-V 架构审核

架构检测：`arch-scan` 判定 `archSpecific: true`（命中 `riscv-macro`）。

1. **指令 ISA 合规**：`vlseg2e64_v_f64m1x2`、`vget_v_f64m1x2_f64m1`、`vfmacc/vfmsac/vfnmsac`、`vsetvl_e64m1` 均属 RVV 1.0；`-march=rv64imafdcv_zvl256b` 覆盖。
2. **LMUL/SEW 一致性**：`vfloat64m1_t`、`sew=64`、`vl=gvl=4` 未变；`vlseg2e64 m1x2` 一次返回 2 个 m1 寄存器组，与原来 2 条 vlse64 的目标寄存器组合一致，无寄存器组重叠。
3. **寄存器预算/无 spill**：`-O3 -S` 确认修改后文件无向量 spill（`vse64.v ...,(sp)` 计数 0，与修改前一致）。折叠后消除了 16 个 tmp 向量寄存器需求。
4. **intrinsic 操作数位序**：审核确认所有 `VFMACC_RR/FOLD_OPA/FOLD_OPB` 调用把标量 B 值置于 rs1（第 2 参数）、向量 A 置于 vs2（第 3 参数），符合 `_vf` 形式签名（首版曾把向量误置 rs1 导致编译失败，已修正）。
5. **运行时 ISA 分发**：未修改分发表，静态绑定不变。
6. **无跨架构指令混入**：diff 仅含 RVV intrinsic、宏定义与 C 结构。

编译自检：`riscv64-linux-gnu-gcc -march=rv64imafdcv_zvl256b -mabi=lp64d -I/tmp/obcfg -I. -DBUILD_KERNEL -DRISCV64_ZVL256B -DDOUBLE -D<16 宏> -fsyntax-only kernel/riscv64/zgemm_kernel_8x4_zvl256b.c` 全部通过；并用 `-D<16 宏> -DCNAME=zk_new` 实际生成目标 + qemu 与 `zk_orig` 逐元素比对的等价性测试全部 bit-exact。

## 审核结果

`pass`

## 发现问题

- **minor / performance / kernel/riscv64/zgemm_kernel_8x4_zvl256b.c:202**：收益依赖 X100 的 FP64 向量 FMA 吞吐。hunk `@@ -172,60 +180,57 @@` 中新增 `+                ACC0r = VFMACC_RR( ACC0r, B0i, A0i, gvl);` 等折叠行。蓝图 `diagnosis.alternativeExplanations` 明示若 X100 FP64 向量 FMA 实际仅 2 lanes/cycle，则 8.3 GFLOPS 可能已近硬件峰值，收益上界需下调；`currentGaps` 亦无 readelf -A 与 PMU 证据。此为收益不确定性，非语义错误，故列 minor 而非 major。建议重建后在 X100 重跑 cblas_zgemm_512x512（OPENBLAS_LOOPS=15）与 perf annotate，核对 16 条 vfadd 消失、k-loop 份额自 77% 下降、GFLOPS 上升。
- **minor / verification / kernel/riscv64/zgemm_kernel_8x4_zvl256b.c:175**：折叠改变 FP rounding 顺序（蓝图 `constraints.mustPreserve` 已声明需通过 checksum/相对误差测试）。本审核在 qemu 仿真下 16 宏 × 6 形状全部 bit-exact，但仿真非 X100 实测，且未运行 OpenBLAS level3 zgemm 正确性套件与 GEMM checksum。建议后续在真机跑 level3 正确性测试与 checksum 对照。
- **suggestion / scope / kernel/riscv64/zgemm_kernel_8x4_zvl256b.c:194**：蓝图 `recommendedFirstAction` 还建议“对 A 做 load pre-issue”，并要求后续对 C-tile epilogue 加 `prefetch.w`。本补丁仅完成 MAC 折叠与 A gather 合并，未做显式 A 预取与 epilogue prefetch（epilogue 占样本 23%、执行频率仅 k-loop 的 1/512）。建议作为独立后续项评估。

## 幻觉自检

- [PASS] 技术精度：所有 ISA/寄存器/spill 结论引用本会话实际执行的 `-fsyntax-only`（16 宏）、`-O3 -S` 指令计数与 qemu 实测语义表；未凭知识断言 X100 微架构，收益不确定性显式降为 minor。
- [PASS] 声明溯源：各 finding 的 `file` 均在 patch.diff 中，`line` 202/175/194 均落在 hunk `@@ -172,60 +180,57 @@` 的旧行范围 172–232 内；evidence 引用 diff 原文与蓝图 `alternativeExplanations`/`currentGaps`/`mustPreserve`/`recommendedFirstAction`。
- [PASS] 可解释性：minor 说明“为什么必须真机复核”，suggestion 说明“为什么范围取舍（epilogue 频率低、A 预取独立）”。
- [PASS] 内部一致性：`reviewResult=pass`，findings 仅含 minor/suggestion；`patchCandidateId` 与 candidate 工具一致；折叠代数推导与 qemu 实测、`-O3 -S` 计数相互印证。
- [PASS] 安全：补丁不引入硬编码密钥、危险命令、越权行为或新攻击面；仅本地 kernel 指令改写。

## 结论

补丁聚焦 `kernel/riscv64/zgemm_kernel_8x4_zvl256b.c` 的 M/8 主 pass k-loop：把 16 个复数累加器的 tmp 物化式折叠为 32 条命中 ACC 的 signed-FMA 直接累加（每 k-step 向量 FP 48→32、FMA 密度 48%→100%），并把成对 interleaved A gather（4×vlse64）合并为 2×vlseg2e64。折叠算子按 4 个转置组分别定义，代数上严格等价于原 tmp 公式，并经 qemu 对 16 个转置宏 × 6 个 M/K 形状（含 tail）实测 bit-exact。特别说明：本补丁未沿用既有 cgemm 补丁的折叠形式（该形式在 vfmsac 分支翻转 ACC 符号，经实测对 K>1 不等价）。tail 路径、epilogue、packing 合同与分发表均未改动。剩余风险（真机收益、真机 checksum、A 预取/epilogue prefetch 未落地）已如实披露。审核通过。