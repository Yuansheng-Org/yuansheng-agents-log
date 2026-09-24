# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/openblas/ctrmm_512x512/004_ctrmm_kernel_LN/blueprint_openblas_ctrmm_512x512_004.json`（`blueprintId=bp-openblas-ctrmm_512x512-004`，`finalStatus=probable_root_cause`，`overallConfidence=0.7`）
- 补丁计划：`.yuansheng/craft/openblas/004_ctrmm_kernel_LN/craft/patch-plan.json`
- 候选补丁：`.yuansheng/craft/openblas/004_ctrmm_kernel_LN/craft/patch-candidate.json`
- 差异：`.yuansheng/craft/openblas/004_ctrmm_kernel_LN/craft/patch.diff`（单文件 `kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c`，5 个 hunk：4 组宏 `@@ -49/-57/-65/-73,6 @@` 与 main-pass k-loop `@@ -212,54 +220,38 @@`）
- 目标硬件：SpacemiT X100（rv64imafdcvh，RVV 1.0，VLEN=256），CORE=RISCV64_ZVL256B

独立只读审核：结论只来自上述落盘事实来源与真实仓库文件/真机 qemu 复现；未修改任何代码。

## 事实核对

1. **文件映射核对**：蓝图 `candidateFiles=["kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c"]`，`codeKnowledgeEvidence.relatedCodeLocation` 指向该文件 :215-262。实测 `KERNEL.RISCV64_ZVL256B:95` 的 `CTRMMKERNEL = ctrmm_kernel_$(CGEMM_UNROLL_M)x$(CGEMM_UNROLL_N)_zvl256b.c`，且 `Makefile.L3:1359+` 确认 ctrmm_kernel_LN 以 `-DTRMMKERNEL -DCOMPLEX -DLEFT -UTRANSA -UCONJ -DNN` 编译该文件。补丁落在真实热点文件。
2. **根因复核（6-op tmp 链）**：源行 215-262 每复数 MAC 为 `vfmul(A0i,Bxi) → VFMACC_RR/VFMACC_RI → vfadd(ACC,tmp)`，合计 16 vfmul + 16 FMA + 16 vfadd = 每轮 48 条向量 FP，与蓝图证据“6 op vs 最小 4 op”一致。
3. **折叠正确性推导**：目标多项式来自同文件标量 M&1 路径：
   - Δr = S0·Ar·Br + S1·Ai·Bi
   - Δi = S2·Ai·Br + S3·Ar·Bi
   折叠后 real 侧 `VFMACC_RR(ACC,Bi,Ai); VFMACC_RR(ACC,Br,Ar)`，在 VFMACC_RR=vfmsac 组给出 `Ar·Br−Ai·Bi`、vfmacc 组给出 `Ar·Br+Ai·Bi`，逐组匹配 S0=1 与 S1；imag 侧用 FOLD_OPA/FOLD_OPB（NN:vfmacc/vfmacc；NR:vfmacc/vfnmsac；RN:vfnmsac/vfmacc；RR:vfnmsac/vfnmsac），在 `vfmsac(x,u,v)=u·v−x`、`vfnmsac(x,u,v)=x−u·v` 语义下逐组给出 S2·Ai·Br+S3·Ar·Bi。
4. **每组 S 宏与 VFMACC 宏核对**：与 `cgemm_kernel_8x8_zvl256b.c` 的同名分组逐字节一致（S0..S3、VFMACC_RR/RI），故 001_cgemm_kernel_n 已验证的 FOLD 映射可直接复用。
5. **mustPreserve 保持**：k=0 初值块（tmp→ACC）、epilogue `alphar/alphai` 折叠（vfnmsac/vfmacc）、C 的 `vsse32` 交错存回与 `ldc` 步长、M&4/M&2/M&1 标量尾块、N&4 尾块均未触碰；`CNAME(...BLASLONG offset)` 签名与 `KERNEL.RISCV64_ZVL256B` 绑定未改。

## RISC-V 架构审核

架构检测：`arch-scan` 判定 `archSpecific: true`（命中 `rvv-intrinsic`/`riscv-macro`），`archReviewWarning` 为空，执行 RISC-V 专项审核。

1. **指令 ISA 合规**：`vsetvl_e32m1`、`vlse32/vsse32_v_f32m1`、`vfmul_vf/vfadd/vfmacc/vfmsac/vfnmsac/vfnmacc` 均属 RVV 1.0；`m1`/`e32` 与既有数据流一致，未新增 vtype。
2. **LMUL/寄存器预算**：折叠消除 tmp 物化的同时并不增加活跃向量数（ACC×16 + A×2 = 18，原 tmp 链峰值更高），寄存器压力下降，符合蓝图“32/32 预算耗尽”问题方向。
3. **tail/vsetvl 与尾块**：仅改 main-pass k-loop 内部算子，未改 `gvl` 设定、tail 策略或尾块路径。
4. **未改架构分发/绑定**：`KERNEL.RISCV64_ZVL256B`、common_riscv64.h 的 RISCV_RVV 包装均未触碰；无 x86/ARM 指令混入。
5. **编译验证**：按真实构建宏 `-march=rv64imafdcv_zvl256b -mabi=lp64d -I/tmp/obcfg -I. -DBUILD_KERNEL -DRISCV64_ZVL256B -DTRMMKERNEL -DCOMPLEX` 对 16 个转置宏（NN NT TN TT NR NC TR TC RN RT CN CT RR RC CR CC）逐个 `-fsyntax-only`，全部通过。
6. **动态差分（qemu-riscv64，`-cpu max,vlen=256`）**：将 HEAD 原始文件与补丁文件分别编成 `orig`/`new` 目标，输入小整数（−4..4）：
   - 方向 A（`-DLEFT -DTRANSA -UCONJ`，对应 LT 内核）：16 宏 × M∈{8,16,24,40} × K∈{2,4,9,16} × offset∈{0..3}，共 **1792 例**；
   - 方向 B（`-DLEFT -UTRANSA -UCONJ`，触发 `BACKWARDS`，对应 LT/RT 内核）：16 宏 × M∈{8,16,24} × K∈{9,16,32} × offset∈{0..3}；
   全部用例 orig-vs-new **数值完全一致，无非零差值**；仅少数元素存在 `0.0`/`-0.0` 符号差异（reassociation 的正常结果，`0.0 == -0.0`）。覆盖 M/8 主 pass、M&4/M&2/M&1 与 N&4 尾块、以及 k-loop 边界。
   - 注：qemu 默认 VLEN=128 会使 e64/m1 初始化向量只搬运 2 个元素，故差分统一使用 `-cpu max,vlen=256` 以匹配构建的 zvl256b（VLEN=256）。

## 审核结果

`pass`

## 发现问题

- **minor / performance / kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c:220**：折叠收益依赖 X100 的 FMA 发射/延迟；蓝图 `alternativeExplanations` 提示若该 kernel 实为 latency-bound，real 侧每 ACC 每 k 变为 2 条串行 FMA 可能部分抵消 issue 下降。属收益不确定性，非语义错误。
- **suggestion / correctness / kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c:49**：FOLD 映射依赖 `vfmsac/vfnmsac/vfnmacc` 的精确 RVV 语义；已在 qemu 上以整数输入差分确认（数值零差），建议保留注释，避免维护者误改算子符号。
- **suggestion / scope / kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c:212**：仅折叠 main-pass（M/8、N/8）k-loop；M&4/M&2/M&1 与 N&4 尾块仍为原 6-op tmp 链。这是最小范围取舍（热点在 main pass），尾块指令膨胀可作为独立后续项。
- **suggestion / correctness / kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c:220**：整数输入下存在 ±0 符号差异（非零差值 0 个）；对下游 BLAS 语义无影响，但如需逐位一致的自检，应在比较时把 ±0 视为相等。

## 幻觉自检

- [PASS] 技术精度：ISA 合规、LMUL/预算、折叠推导（含 vfmsac/vfnmsac/vfnmacc 语义）均有 patch.diff、真实源码行与 qemu 实测（1792+ 例）证据；收益不确定性已降级为 minor。
- [PASS] 声明溯源：findings 的 evidence 引用 patch.diff 真实 hunk 与蓝图具体字段（`alternativeExplanations`、`candidateFiles`）及真实仓库行号/编译命令，无编造锚点。
- [PASS] 可解释性：每条 finding 说明收益不确定来源、维护约束与范围取舍依据。
- [PASS] 内部一致性：`reviewResult=pass`，findings 仅 minor/suggestion，无 critical/major；`patchCandidateId` 与候选产物一致。
- [PASS] 安全：不引入硬编码密钥、危险命令、越权行为或新攻击面。

## 结论

补丁聚焦 `ctrmm_kernel_8x8_zvl256b.c` 的 main-pass k-loop，将每复数乘积的 6-op tmp 物化链折叠为命中成对 accumulator 的 4-op 带符号 FMA，按转置分组以 FOLD_OPA/FOLD_OPB 保证 imag 符号正确，real 侧用同算子二次复合。16 个转置宏语法编译通过；qemu-riscv64（VLEN=256）在非 BACKWARDS 与 BACKWARDS 两种方向下对 16 宏做整数差分，全部数值一致（仅 ±0 符号差异）。剩余风险（X100 上 FMA latency/发射的实测收益、尾块未折叠）已如实披露。审核通过。