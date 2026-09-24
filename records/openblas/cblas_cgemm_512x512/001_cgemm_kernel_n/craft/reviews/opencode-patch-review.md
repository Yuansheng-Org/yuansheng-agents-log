# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/openblas/cblas_cgemm_512x512/001_cgemm_kernel_n/blueprint_openblas_cblas_cgemm_512x512_001.json`（`blueprintId=bp-openblas-cblas_cgemm_512x512-001`，`finalStatus=confirmed_root_cause`，`overallConfidence=0.82`）
- 诊断日志：`.yuansheng/trace/openblas/cblas_cgemm_512x512/001_cgemm_kernel_n/diaglog_openblas_cblas_cgemm_512x512_001.md`
- 补丁计划：`.yuansheng/craft/openblas/001_cgemm_kernel_n/craft/patch-plan.json`
- 候选补丁：`.yuansheng/craft/openblas/001_cgemm_kernel_n/craft/patch-candidate.json`
- 差异：`.yuansheng/craft/openblas/001_cgemm_kernel_n/craft/patch.diff`（单文件，5 个 hunk；热点 k-loop 为 `@@ -186,58 +194,45 @@`）
- 目标硬件：SpacemiT X100（RVV 1.0，VLEN=256，OoO），CORE=RISCV64_ZVL256B

审核为独立只读审核：结论只来自上述落盘事实来源，不依赖实现会话记忆；未修改任何代码。

## 事实核对

1. **文件映射核对**：蓝图 `rootCause.candidateFiles` 记为 `kernel/riscv64/cgemm_kernel_n.c`，且 `diagnosis.currentGaps` 已声明该路径为“按仓库布局推断、需在 commit 70c2410f 下确认”。实际仓库在 `70c2410f` 下 ZVL256B 的 `CGEMMKERNEL` 绑定为 `kernel/riscv64/cgemm_kernel_8x8_zvl256b.c`（`kernel/riscv64/KERNEL.RISCV64_ZVL256B:140`）。补丁准确落在真实热点文件上。
2. **根因 1（A gather）**：k-loop 中两条 `__riscv_vlse32_v_f32m1(..., sizeof(FLOAT)*2, gvl)`（stride=2 元素跨步 gather）合并为一条 `__riscv_vlseg2e32_v_f32m1x2` 段加载 + `__riscv_vget_v_f32m1x2_f32m1` 字段分离。因 packed A 为 interleaved `r0,i0,r1,i1,...`，field0/field1 与两处 `vlse32` 结果逐元素一致，属语义等价替换，命中蓝图证据“23.09% : 17c5e vlse32.v …（A0i 跨步 gather）”。
3. **根因 2（复数 MAC tmp 物化）**：折叠后每复数 MAC 由 6 op（vfmul+VFMACC_RR/RI+vfadd）降为 4 op（每 accumulator 两条带符号 FMA），每轮 48 → 32 条向量 FP，对应蓝图证据“k-loop 每轮 48 条向量 FP 指令 … 可压缩到 32 条”。
4. **vfmsac 语义核对（本轮关键修正）**：RVV `vfmsac(x,u,v)=u*v-x`（在 QEMU 实测确认：`vfmsac(acc=1,s=2,v=3)=5.0`），会将被减对象翻转为 accumulator，故“tmp 内一次 vfmsac 再累加”不能等价改写为“accumulator 上单次 vfmsac”。最终实现据此改为：
   - real：`ACCxr = VFMACC_RR(ACCxr, Bxi, A0i); ACCxr = VFMACC_RR(ACCxr, Bxr, A0r);`（vfmsac 二次复合退化为普通累加；vfmsac 组得 `+ArBr-AiBi`，vfmacc 组得 `+AiBi+ArBr`）。
   - imag：按组定义 `FOLD_OPA/FOLD_OPB`（NN 组 vfmacc/vfmacc，NR 组 vfmacc/vfnmsac，RN 组 vfnmsac/vfmacc，RR 组 vfnmsac/vfnmsac）。
5. **符号等价性实测**：以 `riscv64-linux-gnu-gcc + qemu-riscv64` 对**原始内核与补丁后内核**做差分（同一随机 A/B/C 输入，`-DNN`）：K=1/5、M/N∈{8,10,12,16,512}、含 M&2 尾块，**整数输入全部 bit-exact（maxabs=0）**；浮点输入 512³ 最大绝对差 6.5e-5，属累加顺序改变的正常舍入。另对 16 个转置宏逐一做独立配方差分，符号与量级全部一致（maxabs≈1e-6）。

## RISC-V 架构审核

架构检测：`arch-scan` 判定 `archSpecific: true`（命中 `riscv-macro`），`archReviewWarning` 为空，故执行 RISC-V 专项审核。

1. **指令 ISA 合规**：`vlseg2e32/vget/vfmacc/vfmsac/vfnmsac/vlse32/vsse32/vsetvl_e32m1` 均属 RVV 1.0 基础指令；`metadata.json` 记录目标含 `zve32f/zve64f`，`vlseg2e32` 在 `zve32f` 范围内。
2. **vsetvl/vtype 一致性**：未新增 `vsetvl`；`gvl` 仍由外层 `__riscv_vsetvl_e32m1(8)` 决定。`vlseg2e32` 与 FMA 同用 `e32,m1`，LMUL/SEW 与数据流一致。段加载一次取回 2 个 `m1` 寄存器组（A0r/A0i），与原两处 `vlse32` 目标寄存器一致，无重叠/越界。
3. **tail/mask policy**：未改 tail 策略；M/N 尾块路径未触碰。
4. **运行时 ISA 分发**：未改 `ifunc/hwprobe` 与 `KERNEL.RISCV64_ZVL256B` 静态绑定。
5. **无 x86/ARM 指令混入**：diff 仅含 RVV intrinsic 与 C 结构。
6. **编译验证**：`-march=rv64imafdcv_zvl256b -mabi=lp64d -DBUILD_KERNEL -DRISCV64_ZVL256B -fsyntax-only`，16 个转置宏（NN NT TN TT NR NC TR TC RN RT CN CT RR RC CR CC）全部通过。

## 审核结果

`pass`

## 发现问题

- **minor / performance / kernel/riscv64/cgemm_kernel_8x8_zvl256b.c:200**：`vlseg2e32` 收益依赖 X100 对 RVV 段加载的实现（可能被拆为多 uop）。蓝图 `diagnosis.alternativeExplanations` 亦提示 X100 对 stride 8B gather 的实现代价可能与假设不同。此属收益不确定性而非语义错误，故列 minor。建议在 X100 上以 `perf annotate` 复核 `17c5e`。
- **suggestion / scope / kernel/riscv64/cgemm_kernel_8x8_zvl256b.c:186**：未改动 `../generic/zgemm_ncopy_8.c` 的 interleaved 打包布局（蓝图 `recommendedFirstAction` 提及的 split-plane repack）。这是刻意范围取舍：`constraints.mustPreserve` 要求打包布局与所有消费 kernel（含 M/N 尾块）严格一致，改布局牵动多个尾块消费点，风险高于本补丁；本补丁用段加载在不改合同前提下削减跨步 gather。
- **suggestion / correctness / kernel/riscv64/cgemm_kernel_8x8_zvl256b.c:194**：real 侧“同算子二次复合”的推导依赖 `vfmsac(x,u,v)=u*v-x`；实现已在 QEMU 上对原始/补丁内核做整数 bit-exact 差分确认，但仍建议把该恒等式写入代码注释，避免后续维护者误改回单次 `vfmsac`。

## 幻觉自检

- [PASS] 技术精度：ISA 合规、LMUL/SEW、`vfmsac` 语义（QEMU 实测数值）、等价性结论均有 patch.diff/blueprint/metadata 证据；未凭模型知识断言硬件实现细节，段加载收益不确定性已降级为 minor。
- [PASS] 声明溯源：全部 finding 的 evidence 引用 patch.diff 真实 hunk 原文或蓝图具体字段（`constraints.mustPreserve`、`diagnosis.alternativeExplanations`、`diagnosis.currentGaps`），无编造锚点。
- [PASS] 可解释性：minor 说明为何需复核收益，suggestion 说明范围取舍依据与维护风险。
- [PASS] 内部一致性：`reviewResult=pass`，findings 仅含 minor/suggestion，无 critical/major。
- [PASS] 安全：不引入硬编码密钥、危险命令、越权行为或新攻击面。

## 结论

补丁聚焦 `cgemm_kernel_8x8_zvl256b.c` 主 8x8 pass 的 k-loop，以语义等价的段加载替换跨步 gather、以符合 `vfmsac` 语义的带符号 FMA 复合消除 tmp 物化与 vfadd，同时回应蓝图两主因。16 个转置宏语法编译通过；原始/补丁内核 QEMU 差分整数输入 bit-exact、浮点输入在轮机舍入内。剩余风险（段加载微架构收益、split-plane repack 未落地）已如实披露。审核通过。