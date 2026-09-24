# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/openblas/cher2k_512x512/001_cgemm_kernel_r/blueprint_openblas_cher2k_512x512_001.json`（`blueprintId=bp-openblas-cher2k_512x512-001`，`finalStatus=confirmed_root_cause`，`overallConfidence=0.82`，`recommendToCraft=yes`）
- 诊断日志：`.yuansheng/trace/openblas/cher2k_512x512/001_cgemm_kernel_r/diaglog_openblas_cher2k_512x512_001.md`
- 补丁计划：`.yuansheng/craft/openblas/001_cgemm_kernel_r/craft/patch-plan.json`
- 候选补丁：`.yuansheng/craft/openblas/001_cgemm_kernel_r/craft/patch-candidate.json`
- 差异：`.yuansheng/craft/openblas/001_cgemm_kernel_r/craft/patch.diff`（单文件 `kernel/riscv64/cgemm_kernel_8x8_zvl256b.c`，hunk `@@ -49,6 +49,8 @@`、`@@ -57,6 +59,8 @@`、`@@ -65,6 +69,8 @@`、`@@ -73,6 +79,8 @@`、`@@ -186,58 +194,53 @@`）
- 目标硬件：SpacemiT X100（K3，RVV 1.0，VLEN=256，OoO），CORE=RISCV64_ZVL256B

审核为独立只读审核：结论只来自上述落盘事实来源与本会话实际执行的交叉编译/仿真命令；未修改任何代码。

## 事实核对

1. **文件映射核对**：蓝图 `rootCause.candidateFiles` 记为 `cgemm_kernel_r` 且 `diagnosis.currentGaps` 声明源文件路径未解析。仓库 `kernel/Makefile.L3:1154-1158` 显示 `cgemm_kernel_r` 由 `$(CGEMMKERNEL)` 以 `-UDOUBLE -DCOMPLEX -DNC` 编译；ZVL256B 的 `CGEMMKERNEL`（`KERNEL.RISCV64_ZVL256B:140`）为 `cgemm_kernel_$(CGEMM_UNROLL_M)x$(CGEMM_UNROLL_N)_zvl256b.c`，`param.h` RISCV64_ZVL256B 段 `CGEMM_DEFAULT_UNROLL_M/N=8/8` → `kernel/riscv64/cgemm_kernel_8x8_zvl256b.c`。`001-cgemm_kernel_r-annotate.txt` 的内嵌源码行 114/121–152 与该文件逐行吻合，确认补丁落在真实热点文件。
2. **根因证据核对**：annotate 热区间 k-loop（`0x7354`–`0x7468`）占函数内样本 90.3%，最高单行 12.90% 为 `vlse32.v v1,(t2),a4`（A0i stride-8B gather），另有 16 条 B 标量 flw/步。补丁后交叉编译 `-O3 -S`（NC 配置）计数：`vfadd.vv` 90→74、`vfmul.vf` 180→164、FMA 族 360→376、`vlse32.v` 138→136、`vlseg2e32` 4→5、向量 spill 3→3（无新增）。主 k-loop 每 k-step 向量 FP 由 48 降到 32，与蓝图 `recommendedFirstAction` 一致。
3. **语义等价核对（关键）**：本审核独立构造 qemu-riscv64 差分测试——将修改前（`git show HEAD:...`）与修改后文件分别以同一转置宏 `-DCOMPLEX` 编译为独立目标，同一随机 packed A/B/C 输入调用并逐元素比较 C：
   - **整数取值输入（算术精确、无舍入）：16 个转置宏 × {M,N ∈ 8/16/24/12/6/3} 全部 bit-exact（diff=0）**，证明折叠在语义上与原子式严格等价（含 M&4/M&2/M&1 与 N&4/N&2/N&1 tail 形状）。
   - 随机浮点输入（NC）：31064 个输出单元中 1627 个末位不同，**maxabs=1.79e-7、max|C|=1.45**，即误差处于 FP32 舍入量级；这是折叠改变 rounding 顺序的预期结果（蓝图 `constraints.mustPreserve` 明确允许并要求以 checksum/相对误差校验）。
4. **折叠正确性代数核对**：`vfmsac(x,u,v)=u*v−x`、`vfmacc(x,u,v)=u*v+x`（实测：vfmacc→16、vfmsac→−4、vfnmacc→−16、vfnmsac→4，acc10/s2/v3）。实部同一宏连用两次严格等于 ACC+Br*Ar+s*Ai*Bi；虚部按组取 FOLD_OPA/OPB 严格等于原 `tmp=vfmul(Ar,Bi); tmp=VFMACC_RI(tmp,Br,Ai); ACC+=tmp`。本补丁**未沿用**既有 `001_cgemm_kernel_n` 补丁的折叠形式（该形式在 vfmsac 分支翻转 ACC 符号，经 qemu 实测对 K>1 不等价），而采用经实测等价的形式；注意本任务对象 `cgemm_kernel_r` 编译为 NC（VFMACC_RR=vfmacc）时旧形式恰好等价，但本文件亦用于其它转置变体，故采用全组正确的形式。
5. **不变项核对**：diff 未触碰 tail（M&4/M&2/M&1、N&4/N&2/N&1）与 epilogue（alpha/beta/C stride）结构、packing 布局、分发表；仅改 4 个转置宏块与主 M/8 k-loop。

## RISC-V 架构审核

架构检测：`arch-scan` 判定 `archSpecific: true`（命中 `riscv-macro`、`march-flag`）。

1. **指令 ISA 合规**：`vlseg2e32_v_f32m1x2`、`vget_v_f32m1x2_f32m1`、`vfmacc/vfmsac/vfnmsac`、`vsetvl_e32m1` 均属 RVV 1.0；`-march=rv64imafdcv_zvl256b` 覆盖。
2. **LMUL/SEW 一致性**：`vfloat32m1_t`、`sew=32`、`gvl=8` 未变；`vlseg2e32 m1x2` 一次返回 2 个 m1 寄存器组，与原来 2 条 vlse32 的目标寄存器组一致。
3. **寄存器预算/无 spill**：`-O3 -S` 确认修改前后向量 spill 计数均为 3（无新增），折叠消除 16 个 tmp 向量寄存器需求。
4. **intrinsic 操作数位序**：所有 `VFMACC_RR/FOLD_OPA/FOLD_OPB` 调用标量 B 置 rs1、向量 A 置 vs2，符合 `_vf` 形式签名。
5. **运行时 ISA 分发**：未修改分发表，`cgemm_kernel_r` 仍按 `-DNC` 生成。
6. **无跨架构指令混入**：diff 仅含 RVV intrinsic、宏定义与 C 结构。

编译自检：`riscv64-linux-gnu-gcc -march=rv64imafdcv_zvl256b -mabi=lp64d -I/tmp/obcfg -I. -DBUILD_KERNEL -DRISCV64_ZVL256B -DCOMPLEX -D<16 宏> -fsyntax-only kernel/riscv64/cgemm_kernel_8x8_zvl256b.c` 全部通过；并以 `-D<16 宏>` 实际生成目标 + qemu 与原始文件逐元素比对（整数输入 bit-exact）通过。

## 审核结果

`pass`

## 发现问题

- **minor / performance / kernel/riscv64/cgemm_kernel_8x8_zvl256b.c:200**：收益依赖 X100 对 vlseg2e32 与 FMA 发射的实现。hunk `@@ -186,58 +194,53 @@` 中新增 `+                ACC0r = VFMACC_RR( ACC0r, B0i, A0i, gvl);` 等折叠行与 `+                    vfloat32m1x2_t Aseg0 = __riscv_vlseg2e32_v_f32m1x2( &A[ai+0*gvl*2], gvl );`。蓝图 `diagnosis.alternativeExplanations` 指出 vlse32 在 X100 的每元素代价无公开资料、若被优化为合并访问则收益低于 48.4% 上界，且 benchmark 中 random() 另占约 48% 样本。此为收益不确定性，非语义错误，故列 minor 而非 major。建议重建后重跑 cher2k_512x512 与 perf annotate，核对 0x735c vlse32 与 16 条 flw 行样本骤降、出现 vlseg2e32。
- **minor / verification / kernel/riscv64/cgemm_kernel_8x8_zvl256b.c:187**：折叠改变 FP rounding 顺序，随机输入下 maxabs≈1.8e-7（FP32 量级）。本审核在 qemu 以整数输入证明语义等价、随机输入量化舍入误差，但仿真非 X100 实测，且未跑 OpenBLAS cher2k/level3 检验。建议真机跑 cher2k_512x512 与复数 Hermitian 回归（含非零虚部、共轭、alpha 缩放）验证误差合同。
- **suggestion / scope / kernel/riscv64/cgemm_kernel_8x8_zvl256b.c:194**：蓝图 `problem`/`recommendedFirstAction` 还要求把 C 更新 epilogue 的 16×vlse32/vsse32 换成 8×vlseg2e32/vsseg2e32（占函数样本 9.7%，单条成本约 k-loop 均值 12 倍）。本补丁仅覆盖 k-loop（占 90.3%）；epilogue 改动涉及更多 tail 复制点，建议作为独立后续项评估。

## 幻觉自检

- [PASS] 技术精度：ISA/LMUL/spill/量化误差结论均引用本会话实际执行的 `-fsyntax-only`（16 宏）、`-O3 -S` 指令计数、qemu 整数与随机差分命令；未凭知识断言 X100 微架构，收益不确定性显式降为 minor。
- [PASS] 声明溯源：findings 的 `file` 为 patch.diff 中唯一文件，`line` 187/194/200 均落在 hunk `@@ -186,58 +194,53 @@` 的旧行范围 186–244 内；evidence 引用 diff 原文与蓝图 `alternativeExplanations`/`recommendedFirstAction`/`problem`。
- [PASS] 可解释性：minor 说明“为什么必须真机复核”，suggestion 说明“为什么 epilogue 独立”。
- [PASS] 内部一致性：`reviewResult=pass`，findings 仅含 minor/suggestion；`patchCandidateId` 与 candidate 工具一致；折叠代数、整数 bit-exact 与 fp 量化误差三种证据相互印证。
- [PASS] 安全：补丁不引入硬编码密钥、危险命令、越权行为或新攻击面；仅本地 kernel 指令改写。

## 结论

补丁聚焦 `kernel/riscv64/cgemm_kernel_8x8_zvl256b.c`（`cgemm_kernel_r` = 该文件 `-DNC` 编译）的主 M/8 k-loop：把 16 个复数累加器的 tmp 物化式折叠为 32 条 signed-FMA 直接累加，并把 A 的成对 stride-8B vlse32 gather 合并为 1 条 vlseg2e32。折叠算子按 4 个转置组定义，代数严格等价，并经 qemu 对 16 个转置宏 × 多形状（含 tail）的整数输入实测 bit-exact。tail、epilogue、packing 合同与分发表均未改动。剩余风险（真机收益、真机 checksum、C epilogue 未落地）已如实披露。审核通过。