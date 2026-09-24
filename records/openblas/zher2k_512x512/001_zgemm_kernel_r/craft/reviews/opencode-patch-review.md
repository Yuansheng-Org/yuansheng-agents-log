# AI 补丁审核报告

## 审核范围

- **补丁候选**：`pc-bp-openblas-zher2k-512x512-001`
- **蓝图**：`bp-openblas-zher2k_512x512-001`（`.yuansheng/trace/openblas/zher2k_512x512/001_zgemm_kernel_r/`）
- **改动文件**：`kernel/riscv64/zgemm_kernel_8x4_zvl256b.c`（modify，+56/-52）
- **审核依据**：`patch.diff`、蓝图 `problem/rootCause/constraints.mustPreserve/diagnosis`、仓库真实文件 `kernel/riscv64/zgemm_kernel_8x4_zvl256b.c`、`kernel/Makefile.L3:1196-1206`（zgemm_kernel_r ← -DNC）、该批次已审核的 `001_cgemm_kernel_n`/`001_cgemm_kernel_r` 折叠范式与 `001_cgemm_kernel_r/craft/patch.diff`。独立只读审核，未修改候选补丁。

## RISC-V 架构审核

- **状态**：passed
- **指令集合规**：新增 `__riscv_vlseg2e64_v_f64m1x2`、`__riscv_vget_v_f64m1x2_f64m1` 与 `__riscv_vfmacc/vfmsac/vfnmsac`（经 VFMACC_RR/RI、FOLD_OPA/FOLD_OPB 宏）均在 RVV 1.0 内；SEW=64、LMUL=m1（gvl=4），最大 m1x2 复数对寄存器组在预算内。
- **折叠正确性（关键）**：逐组推导并机器验证 —— 原式为 `tmp_r=vfmul(Ai,Bi); tmp_r=VFMACC_RR(tmp_r,Br,Ar); ACC_r+=tmp_r`；折叠式 `ACCr=VFMACC_RR(ACCr,Bi,Ai); ACCr=VFMACC_RR(ACCr,Br,Ar)`。由 `vfmsac(x,u,v)=u*v-x` 与 `vfmsac(vfmsac(a,u,v),w,y)=a+w*y-u*v`（vfmacc/vfnmsac 同理）知折叠式在代数上等于 `ACCr + Br*Ar - Bi*Ai`，与原式逐 k 累加值一致；浮点乘法交换律保证 `Bi*Ai==Ai*Bi` 逐位相同。虚部 FOLD_OPA/FOLD_OPB 按转置组取 `vfmacc/vfnmsac`，覆盖 NN/NR/RN/RR 四组（见 patch.diff 四处宏组 hunk）。
- **segmented 载入等价性**：`vlseg2e64(&A[ai+o*gvl*2], gvl)` 对连续 `2*gvl` 个 double 解交织为 (re,im) 两向量，等价于原 `vlse64(...,+0,stride16)` 与 `vlse64(...,+1,stride16)` 两次 stride-16 gather；A 面板为连续交织布局，读取偏移与元素集合完全一致。
- **编译自检**：`riscv64-linux-gnu-gcc -march=rv64imafdcv_zvl256b -mabi=lp64d -I/tmp/obcfg -I. -DBUILD_KERNEL -DRISCV64_ZVL256B -DDOUBLE -DCOMPLEX -fsyntax-only kernel/riscv64/zgemm_kernel_8x4_zvl256b.c` 在 **16 个转置宏（NN/NT/TN/TT/NR/NC/TR/TC/RN/RT/CN/CT/RR/RC/CR/CC）** 下全部通过。
- **数值验证**：以 `qemu-riscv64 -cpu max,vlen=256` 对 HEAD 原始与补丁内核（各转置宏、独立 CNAME）做整数输入随机差分，16 宏 × 400 例（M,N∈[1,13]、K∈[1,3]、随机 ldc 与 alphar/alphai），输出 C 缓冲逐位比较：**全部 `fail=0 PASS`**。整数输入使乘积/累加精确可表示，故该差分严格验证了折叠式与原式的代数等价（含共轭符号规则）。
- **范围**：仅改主 pass（N/4）k=1..K-1 循环体；k=0 剥离块、M tail 与 N&2/N&1 tail 保持原实现，C update phase 与 ldc 语义不变。

## 审核结果

**pass**。补丁直接作用于蓝图指认的 zgemm_kernel_r k-loop，将 stride-16 A gather 改为 vlseg2e64 复数对载入、将 tmp/vfadd 三段式 MAC 折叠为直接配对 FMA，减少 FP/LSU 指令数；逐组折叠定义与已审核 cgemm 补丁一致，满足 `constraints.mustPreserve` 的共轭符号与 ldc/交织布局合同。无 critical/major 问题。

## 发现问题

1. **[minor / verification]** 未在 SpacemiT X100 上复测收益：本环境仅有 `qemu-riscv64` 的功能逐位等价证据，无法验证 k-loop 指令数下降带来的 IPC/GFLOPS 变化与样本份额。证据：patch.diff hunk `@@ -172,60 +180,56 @@` 中新增 `+                    vfloat64m1x2_t Aseg0 = __riscv_vlseg2e64_v_f64m1x2( &A[ai+0*gvl*2], gvl );`；蓝图 `diagnosis.currentGaps`（无 ELF、无 ARM 对比）。建议：X100 上重跑 zher2k_512x512 并 perf annotate，确认 vlse64.v gather 与 vfadd 序列消失、k-loop 样本下降。
2. **[suggestion / scope]** C-phase（0x73e6–0x7536，蓝图 23.8% samples）的 stride-16 gather/scatter 未在本补丁处理（仅 k-loop）。证据：patch.diff 未触及 `ci=n_top*ldc+m_top` 之后的 C 载入/写回序列；蓝图 `diagnosis.recommendedFirstAction` 同时建议改 C-phase 为 vlseg2e64/vsseg2e64。建议：如需进一步按蓝图推进，可在后续单独补丁中把 C-phase 的 16×vlse64+16×vsse64 改为 segmented 载入/存储并归纳地址，需单独 A/B 验证。
3. **[suggestion / maintainability]** 折叠式与原式在一般浮点输入下舍入次序不同（代数等价、非逐位等价）；本测试用整数输入证明代数等价。证据：patch.diff 新增 FOLD_OPA/FOLD_OPB 宏组 hunk 与折叠注释块。建议：若下游依赖与参考实现逐位对齐，需在文档/回归中明确容差语义（与该批次 cgemm 折叠补丁保持一致）。

## 幻觉自检

- **技术精度**：PASS —— vfmsac/vfmacc/vfnmsac 的折叠恒等式、vlseg2e64 解交织语义、8 累加器与 A0/A1×B0..B3 映射均逐条核对。
- **声明溯源**：PASS —— 引用来自 patch.diff 原文、Makefile 规则与蓝图字段，未凭模型知识补全。
- **可解释性**：PASS —— 折叠推导、等价性论证、验证命令与未覆盖的 C-phase 完整披露。
- **内部一致性**：PASS —— PatchPlan 目标、改动文件与 patch.diff 一致。
- **安全**：PASS —— 载入/累加均在 gvl 限定范围内，无越界；无硬编码敏感信息。

## 结论

补丁为针对 zher2k 复数 GEMM 微内核 k-loop 的局部等价优化，把 A 的 stride-16 gather 换为 vlseg2e64 复数对载入、把 tmp 三段式 MAC 折叠为直接配对 FMA，逐组 FOLD 定义与该批次 cgemm 补丁一致；`patch.diff` 仅含单一 kernel 源文件改动，16 转置宏编译自检与 qemu 逐位差分（16×400 例）均通过。审核通过（pass）；剩余为硬件收益、C-phase 范围与舍入容差的不确定性，作为 minor/suggestion 记录，不阻断交付。
