Functions under analysis: [mxnet_warpctc::CpuCTC<float>::log_softmax(float const*, float*, int const*) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`002-...log_softmax...-annotate.txt`，cpu-clock 2443 samples，percent: local period，覆盖全部 hot loop 与 setup/epilogue）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-loss.txt`）
- workload/binary/DSO/source context：已提供（libmxnet.so，DYN ELF64 RISC-V，RVC + double-float ABI；annotate 内嵌 CpuCTC 模板源码行，source context 充分）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（libmxnet.so Tag_RISCV_arch 含 `v1p0`、`zve64d1p0`、`zvl128b1p0` 等）
- hardware ISA（cpuinfo）：已提供（rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt）
- `vlenb`：已提供（VLEN=128 bits / vlenb=16，来自批次冻结 metadata）
- 采样元数据（event / percent type / scope / 窗口）：部分提供 — event=cpu-clock，percent=local period，单次运行窗口；该函数占整个 workload 的贡献未知
- Sampling IP precision（precise_ip / Exact-IP / skid）：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `v` 存在（RVV 1.0，XuanTie C920v2 / SOPHGO SG2044），并含 zba/zbb/zbc/zbs/zfa/zfh/zvfh 等；OoO core |
| Build ISA | libmxnet.so `Tag_RISCV_arch` 含 `v1p0 ... zve32f1p0 zve64d1p0 zve64f1p0 zve64x1p0 zvl128b1p0`；build 具备 RVV，hot loop 却为全 scalar |
| Vector flavor | annotate 内无 `v*` 也无 `th.v*`（全 scalar），无 flavor mismatch；build 与 hardware 均为 RVV 1.0 |
| VLEN | 128 bits（vlenb=16，来自冻结 metadata snapshot） |
| Bound type | IPC=0.6316；L1_dcache_load_miss_rate=0.259%、LLC_load_miss_rate=42.913%（LLC 事件疑似语义混杂，cache_miss_rate 100% 视为计数伪影）；hot loop 按指令形态属 compute/latency-bound（标量 FP 比较/归约 + libm 调用），非 memory-bound |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=local period（**非 global**）；同一运行窗口；函数级 workload 贡献未知 → 收益上界只能表述为局部样本份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip / Exact-IP 记录，C920v2 PMU skid 能力未知）→ 单行高占比只能锚定所属 basic block / loop interval，不得作 instruction-latency 归因 |

L0 baseline gate：hardware 有 `v`（cpuinfo isa 含 `v`）且 build 含 `v`（Tag_RISCV_arch 含 `v1p0`+`zvl128b1p0`）→ **非 build/hardware mismatch**；hot loop 零 `v*` 属 autovec gap。无 `th.v*`，无 flavor 冻结。Bound-type gate：非 memory-bound，RVV rewrite 不被 gate 降级；impact 仍受 sampling semantics（local period）与函数级贡献未知约束。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（单函数）。hot interval 划分与 trace anchors：
- **max 归约环** `12c54b0–12c54c8`：最高行 `29.02 :   12c54be: beqz a3,12c54c4`（max-loop interval 锚点；IP precision 不足，不归因单指令 latency）；同区间 `6.92 : 12c54ba: flt.s a3,fa5,fa4`、`8.02 : 12c54b4: fmv.s fs1,fa5`、`10.72 : 12c54c4: fmv.s fa5,fs1`
- **denom 环** `12c5450–12c5466`：`4.09 : 12c545a: auipc ra,0xff2da` + `0.16 : 12c545e: jalr -602(ra) # 59f200 <expf@plt>`（逐元素 expf 调用点）；`11.05 : 12c5466: bne s11,s1,12c5450`
- **log 输出环** `12c546c–12c548c`：`11.95 : 12c5488: fsw fs0,-4(s2)`；`7.86 : 12c548c: bne s10,s1,12c546c`；`0.16 : 12c547e: jalr 646(ra) # 5a4700 <logf@plt>`
- setup/epilogue（12c534a–12c5448、12c54ce–12c54d2）仅占约 0.1%，不作证据主体。

annotate 覆盖完整，含全部三个计算环。采样语义：仅局部份额，禁止 workload 级 Amdahl 上界。

## Phase 3 — Pattern scan / 模式扫描：mxnet_warpctc::CpuCTC<float>::log_softmax
### Class selection trace（8 项）
1. `rows-asm.md` — exclude：当前代码来源为 compiler-generated（_omp_fn.0 clone + @plt 调用形态），无 `.S`/DWARF 直接 provenance；无 mxnet warpctc assembly-default policy 证据，policy-backed missing `.S` 四证不全。
2. `rows-operator-rvv.md` — include：compiler-generated scalar loop，语义明确为 LogSoftmax（max → exp-sum → log 三阶段），命中 normalization row；同时核对 elementwise-activation / extrema-reduction / widening-reduction 行内互斥。
3. `rows-string-memory.md` — exclude：非 copy/fill/scan/compare/checksum/back-reference/string 原语。
4. `rows-vectorized-tuning.md` — exclude：hot loop 零 `v*`，无 RVV 配置/寄存器/展开可调。
5. `rows-codegen.md` — include：compiler-generated；核对 control-flow（beqz 高占比）、register-pressure（fmv.s 挪移）、ISA-substitution（F 扩展 fmax.s）、FP-semantic-lowering、kernel-selection、fusion 等跨行。
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD/权重重排/GEMM tile 证据。
7. `rows-crypto.md` — exclude：无密码学原语。
8. `rows-runtime-os.md` — exclude：用户态 MXNet 算子，非 RTOS/kernel/timer/CSR 热点。

### Classes scanned: rows-operator-rvv.md, rows-codegen.md

### Local performance pattern scan: `mxnet_warpctc::CpuCTC<float>::log_softmax`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Normalization Kernels（primary） | LogSoftmax 三阶段：标量分支式 max 归约环（flt.s+beqz+fmv.s，≈58.8% 局部样本）、逐元素 `expf@plt` denom 环、逐元素 `logf@plt`+`fsw` 输出环；全环零 `v*`；硬件 RVV 1.0 / VLEN=128 / build 含 `v1p0` | High | Medium（sampling semantics 非 global + 函数级贡献未知） | `patterns/rvv_normalization_kernels.md` |
| No vectorization（supporting） | 同一 hot main loop 全 scalar、zero `v*`；build 已含 `v1p0`+`zvl128b1p0`，hardware 暴露 `v`；只解释缺向量执行载体，不决定贡献主体 | — | — | `patterns/no-vectorization.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（supporting，fmax.s 分支） | max 环 `flt.s a3,fa5,fa4` + `beqz` + 冗余 `fmv.s fs1,fa5`/`fmv.s fa5,fs1` 条件更新序列；build 含 `f2p2`（F 扩展提供 `fmax.s`/`fmin.s`）；逐元素 3 指令+2 挪移 可换 1 条 branchless `fmax.s` | — | — | `patterns/isa_extension_specific_instruction_substitution.md` |

**顶层 finding 1（primary）— RVV Normalization Kernels 三件套：**
- (a) 逐字 evidence 引用：
  - `29.02 :   12c54be:        beqz    a3,12c54c4`（max 归约环 12c54b0–12c54c8，interval 锚点）
  - `6.92 :   12c54ba:        flt.s   a3,fa5,fa4`；`8.02 : 12c54b4: fmv.s fs1,fa5`；`10.72 : 12c54c4: fmv.s fa5,fs1`
  - `4.09 :   12c545a:        auipc   ra,0xff2da` → `0.16 : 12c545e: jalr -602(ra) # 59f200 <expf@plt>`（denom 环）
  - `11.95 :  12c5488:        fsw     fs0,-4(s2)`（log 输出环 store）；`0.16 : 12c547e: jalr 646(ra) # 5a4700 <logf@plt>`
- (b) 互斥邻居排除：
  - elementwise-activation row：排除 — exp/log 并非 standalone lane-independent activation；其前后被跨 lane max 归约（12c54ba/12c54be）与跨 lane denom 累加（12c5462 `fadd.s fs2,fs2,fa0`，loop-carried 归约）包夹，是完整 LogSoftmax 的 transform 阶段。
  - extrema-reduction row：排除 — max 环虽为 value-only 归约，但它是 LogSoftmax 完整流程的第一阶段；行内互斥明示 "normalization 完整流程 → normalization row"。
  - widening-reduction row：排除 — denom 累加是 Softmax sum 阶段（非 standalone additive reduction），且 FP32 无 widening 语义。
  - no-vectorization row：自身 gate 成立但为 generic fallback，行内互斥 "operator semantic shape → 各自更具体 row" → 本 case 归一化语义行先认领，no-vectorization 降为 supporting。
- (c) 双 Confidence 推导式：`route: compiler-generated 标量 + 源码语义 LogSoftmax 三阶段 + 硬件 V 与 build v1p0/zvl128b 直接 provenance + 行内互斥排除 → High；impact: 局部 sample share ≈99%（三环加总）但 sampling semantics 非 global、函数级 workload 贡献未知、expf/logf 的 libm 内部样本不在此函数内 → Medium`

**Supporting（no-vectorization）三件套：**
- (a) `0.04 : 12c5450: flw fa0,0(s11)` / `0.25 : 12c5470: fmv.s fa0,fs2` 等 —— 三环全部为 scalar FP load/compute/store，零 `v*`（类内全文检索无 vsetvli/vle/vse/vfred*）。
- supporting because: 与 primary 同一机制（缺少向量执行载体）；build 已含 V 而 autovectorizer 未覆盖这些循环（expf/logf 调用 + 分支式 max + std::max 引用语义），解释"为什么没有 RVV 指令"，不决定贡献主体。
- (a) `8.02 : 12c54b4: fmv.s fs1,fa5` / `10.72 : 12c54c4: fmv.s fa5,fs1` / `29.02 : 12c54be: beqz a3,12c54c4` — max 环每元素执行 2 条挪移 + 1 分支 + 1 flt.s 的条件更新序列，F 扩展原生 `fmax.s` 可替代（build 含 f2p2）。
- supporting because: 与 primary 同一 max-loop interval 上的 codegen 微结构；RVV 归一化重写后该标量序列整体消失（causal elimination），故并入 primary 不另立顶层。

**多候选仲裁小段：** primary = RVV Normalization Kernels（L1 vectorization/semantic dispatch，evidence 主导三个 hot interval ≈99% 局部样本）；两个 supporting（no-vectorization、fmax.s 替换）与其同机制、被因果消除，不计顶层命中；无 companion / independent。`dynamic priority unavailable` 不适用（入口条件 A 有局部份额）——本 finding 局部 sample share 加总 ≈ 58.8% + 15.6% + 24.6% + 0.25%（denom init）≈ 99.2%；expf/logf 的 libm 内样本不在本函数 symbol 下，实际收益份额更高（不可量化）。

## Phase 4 — Root-cause blueprint / 根因蓝图：mxnet_warpctc::CpuCTC<float>::log_softmax
（依据 `patterns/rvv_normalization_kernels.md`，matched row = rows-operator-rvv.md normalization row；supporting 依据 `patterns/no-vectorization.md` 与 `patterns/isa_extension_specific_instruction_substitution.md`）

1. **Root cause**：`CpuCTC<float>::log_softmax` 对每行（alphabet_size_ 个元素）执行 LogSoftmax，生成三个全标量 pass：① 分支式 max 归约（`flt.s`+`beqz`+冗余 `fmv.s` 双寄存器挪移），② 逐元素 `expf@plt` 的 denom 归约，③ 逐元素 `logf@plt` 加 `fsw` 的输出 pass。依据 `patterns/rvv_normalization_kernels.md` §Why this is slow："归一化的根因是跨 lane reduction 造成依赖链，多阶段完整扫描造成额外流量，或 reciprocal/rsqrt/vector-math 不可达而退回 scalar helper"——max/denom 两个 loop-carried 标量归约链 + 每元素一次 libm 调用（`expf`/`logf`@plt）即该句的直接实例；§7 要求 "Softmax 应至少分为三个阶段：max → exp(x-max) 保存并归约指数和 → reciprocal/除法缩放"，当前实现正是未向量化的标量三阶段，跨 lane 归约逐元素串行。硬件（RVV 1.0 / VLEN=128）与 build（v1p0+zvl128b1p0）均已具备向量能力，但三个环零 `v*`（no-vectorization supporting：autovec 因 expf/logf 调用、分支式 max、std::max 引用语义未覆盖）。IP precision 不足，不将 29.02% 的 `beqz` 单行归因为分支延迟，只作 max-loop interval 证据。
2. **The fix / 修复方式**（与 pattern §5/§6/§7 一致，作为蓝图非实施）：
   - **max 阶段**：每 chunk 向量加载 `vle32`，`vfredmax`（带标量 seed 的安全分块归约，§5 结构）得行内 max。
   - **exp+sum 阶段**：`vle32` → `vfsub.vf`(max) → 向量 exp（**必须调用项目真实存在且通过验证的 RVV exp 实现**——RVV 无单指令 exp；无可用向量 exp 时该阶段退回 scalar helper 并在误差合同内评估，见 `vector-math-conventions.md`）→ `vfredusum` 分块归约 denom（§7 结构，保存中间 exp 值）。
   - **输出阶段**：复用保存的 exp 中间值或再加载 `act`，`act - max - log(denom)`，`vse32` 写 `log_probs`（LogSoftmax 输出 pass；`logf` 仅对 scalar denom 调用一次，不再逐元素）。
   - 伪代码（before/after）：
     ```cpp
     // Before（当前 scalar 三环）：
     //   max = scalar loop over r: flt.s+beqz+fmv.s (12c54b0–12c54c8)
     //   denom = scalar loop: denom += expf(act[r]-max) (12c5450–12c5466)
     //   log_probs[r] = act[r] - max - logf(denom) (12c546c–12c548c)
     // After（RVV 形态，示意）：
     //   vfloat32m4_t maxv = vfredmax(vle32(act), -inf seed);
     //   float max = extract(maxv);
     //   per-chunk: expv = vexp(vfsub(vle32(act+off), max));  // 项目验证的向量 exp
     //              denom += extract(vfredusum(expv, denom_seed));
     //   per-chunk: vse32(log_probs+off, vfsub(vfsub(vle32(act+off), max), logf(denom)));
     ```
   - 适用前提：`-march` 已含 V（当前 build 已满足）、每行长度 ≥ VLMAX 时收益明显（VLEN=128 → e32m4 = 16 lanes/iter）、CTC alphabet_size 足够大。
   - correctness contract：vfredusum/vfredmax 改变 FP 结合顺序（pattern §5 "vfredusum 会改变浮点加法结合顺序"，有序语义需 vfredosum）；覆盖全相等、全 -Inf、+Inf、NaN、denom=0/Inf/NaN、exp overflow/underflow、LogSoftmax 稳定减 max（§7 清单）；`std::max` 的 NaN 引用语义若改用 `fmaxf`/`fmax.s` 需验证 NaN 首参病态路径（scalar supporting 单独注意）。
   - 限制/风险：RVV 无原生 exp/log —— 向量 exp 必须新实现或复用已验证实现（`vector-math-conventions.md`：不得仅凭 hardware V 推断 vector-math 可用）；短行（alphabet 小）可能低于向量化 crossover；FRM/FCSR 状态与 tail 处理按 `kernel-conventions.md` §3 fixed-VL main loop + runtime-VL tail；OoO core（C920v2）上 reduction 依赖链仍需多 accumulator 隐藏延迟（`kernel-conventions.md` §1）。
   - 修复后预期 Profile signals：三环中 `flt.s/beqz/fmv.s`（max）、`expf@plt`（denom）、`fsw`/`logf@plt`（输出）样本显著下降；出现 `vsetvli`/`vle32`/`vfredmax`/`vfredusum`/`vse32`；`expf`/`logf` symbol 下（libm）样本下降。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv_...（RVV 1.0）；build ISA = libmxnet.so `v1p0`+`zvl128b1p0`（含 V）；VLEN = 128 bits；bound type = compute/latency-bound（IPC 0.6316、L1 miss 0.259%）；sampling = cpu-clock local period（非 global）。
4. **收益上界**：当前 sampled event（cpu-clock）下局部样本份额 ≈ 99.2%（max 环 58.8% + denom 环 15.6% + 输出环 24.6% + init 0.25%）；非 workload 级 Amdahl（Phase 1 采样语义四条未全成立，`baseline_gap: sampling metadata` 已标）。expf/logf libm 内部样本不在本函数 symbol 下，实际收益上界被低估。
5. **三维路由判定**：`current source` = compiler-generated scalar（_omp_fn.0 OMP clone，@plt 形态，无 `.S` provenance）；`implementation existence/reachability` = 无现成 RVV kernel/dispatch slot 证据，修正对象为模板函数代码生成（intrinsic/编译器引导），非 kernel-selection；`function-level policy` = 无 assembly-default policy 证据 → 不走 missing `.S` 分支，The fix 落在 intrinsic/代码改写层。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。
7. **Related PRs 小节**（依据 `patterns/rvv_normalization_kernels.md` §Related PRs）：
   - RVV normalization kernel optimization（oneDNN）：Related PRs：3 条 URL — https://github.com/uxlfoundation/oneDNN/commit/36df719b3bc9001d994455c2899e42533b2c29ac、https://github.com/uxlfoundation/oneDNN/commit/6dc3e2d84eec45eff0834e8985cc663db6fe07ee、https://github.com/uxlfoundation/oneDNN/commit/3b90f9d0650f4087d4a6fbcb3406dc0862f2ae25
   - RVV normalization follow-up（oneDNN）：Related PRs：4 条 URL — https://github.com/uxlfoundation/oneDNN/pull/4809、https://github.com/uxlfoundation/oneDNN/pull/4622、https://github.com/uxlfoundation/oneDNN/pull/4453、https://github.com/uxlfoundation/oneDNN/pull/4480
   - RVV normalization follow-up（oneDNN）：Related PRs：1 条 URL — https://github.com/uxlfoundation/oneDNN/commit/0d8f4a9e702b70f6188f58a20247ae06dbd41ce5
   - RVV Softmax kernel optimization（oneDNN）：Related PRs：3 条 URL — https://github.com/uxlfoundation/oneDNN/commit/7ad03324c2d17480e40cc20114576584888373ce、https://github.com/uxlfoundation/oneDNN/commit/3c8c37bab64f67bbdebcdb76330db0dcb25b31af、https://github.com/uxlfoundation/oneDNN/commit/c07b7f4e777636cf527f49093c03f8cacf58b3de
   - RVV Softmax follow-up（oneDNN）：Related PRs：4 条 URL — https://github.com/uxlfoundation/oneDNN/pull/4734、https://github.com/uxlfoundation/oneDNN/pull/4491、https://github.com/uxlfoundation/oneDNN/commit/25cd4a75095a2aebeb2f0140abadd466cd0b4e90、https://github.com/uxlfoundation/oneDNN/commit/b8360ec0a1d566f039d4fe286349c7988a762c98
   - Softmax normalization（MNN）：Related PRs：3 条 URL — https://github.com/alibaba/MNN/pull/4508、https://github.com/alibaba/MNN/pull/4044、https://github.com/alibaba/MNN/commit/b7268aa3dab754190ed7d86dc16b9bab02e73d12
   - No vectorization（支持性，scalable RVV universal-intrinsic backend 上游范例）：Related PRs：15 条 URL — https://github.com/opencv/opencv/pull/22179、https://github.com/opencv/opencv/pull/22520、https://github.com/opencv/opencv/pull/23980、https://github.com/opencv/opencv/pull/24058、https://github.com/opencv/opencv/pull/24132、https://github.com/opencv/opencv/pull/24166、https://github.com/opencv/opencv/pull/24301、https://github.com/opencv/opencv/pull/24325、https://github.com/opencv/opencv/pull/27160、https://github.com/opencv/opencv/pull/27119、https://github.com/opencv/opencv/pull/27097、https://github.com/opencv/opencv/pull/27007、https://github.com/opencv/opencv/pull/26958、https://github.com/opencv/opencv/pull/26865、https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d

## Phase 5 — Verification forecast / 验证预测：mxnet_warpctc::CpuCTC<float>::log_softmax
（修复对象 = primary：RVV 化三阶段 LogSoftmax；supporting 跟随，不单独验证）
- 应消失/缩小（锚定 Phase 3(a) 引用行）：max 环 `12c54ba: flt.s`/`12c54be: beqz`/`12c54b4`+`12c54c4: fmv.s` 序列样本大幅下降；denom 环 `12c545a: auipc`→`12c545e: jalr expf@plt` 调用点消失（改为向量 exp 或单次标量）；输出环 `12c5488: fsw`/`12c547e: jalr logf@plt` 逐元素调用点消失。
- 应出现（依据 `patterns/rvv_normalization_kernels.md` §Verification）：`vsetvli`、`vle32`、`vfredmax`、`vfredusum`、向量 exp/`vse32` 指令进入 hot loop；cycles/row 或吞吐改善；数值合同通过（全相等、极端 logits、NaN/Inf、sum≈1、LogSoftmax consistency、稳定减 max）。
- 验证方法（profile-flow §Phase 5）：目标 `-march`（含 V）同优化级重建 → 对 log_softmax 重新 annotate 核对上述指令出现/消失；对 CTC 典型短/中/长 alphabet_size 与 minibatch 做 benchmark；FP reordering 与 exp 误差合同（`vector-math-conventions.md`：最大 ULP/relative error 扫描、特殊值、FRM/FCSR、tail/mask）先于表述可接受。
- 升级到更强结论所需数据（已具备多数）：函数级 workload 贡献（global-period 重采或全程序热点排序）→ 才能从局部份额升级到 Amdahl 上界；`perf evlist -v`/`perf report --header-only` 核对 `precise_ip` → 才允许 instruction-latency 级归因。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | `1/1 组；mxnet_warpctc::CpuCTC<float>::log_softmax` |
| 2 | Phase 1 输出要求 | ✅ | 7 行 baseline 齐（Hardware/Build/Flavor/VLEN/Bound/Sampling/IP）；gap 标签：`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ | 8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding=1（RVV Normalization）；evidence 锚点 `29.02 : 12c54be: beqz a3,12c54c4`、`4.09 : 12c545a: auipc`、`11.95 : 12c5488: fsw`；supporting=2（no-vectorization、fmax.s）；排除=3（elementwise/extrema/widening）；推导式=route High / impact Medium |
| 4 | Phase 4 输出要求 | ✅ | 已读 pattern：`rvv_normalization_kernels.md`（§Why this is slow 引文、§5 vfredusum、§7 三阶段）、`no-vectorization.md`（autovec gap）、`isa_extension_specific_instruction_substitution.md`（fmax.s）；The fix 含 before/after、correctness（vfredusum 顺序/NaN/Inf/denom）、风险（向量 exp 缺失、短行 crossover）、Profile signals；Related PRs：normalization 15 条 + supporting 15 条 URL |
| 5 | 路径合规 | ✅ | 模式 A（profile-backed）；8 项 trace 全覆盖；L0 无 mismatch；leaf 均来自通过 gate 的 row；impact 未用单行占比升级为 instruction-latency（IP precision gap） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 `12c54be: beqz`/`12c545e: jalr expf@plt`/`12c5488: fsw`；出现侧 `rvv_normalization_kernels.md §Verification`（vfredmax/vfredusum/vse32） |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；无用户追问；交付止于证据、蓝图、The fix、验证预测 |

修正记录：无