Functions under analysis: [void mxnet::op::mxnet_op::Softmax<mxnet::op::mxnet_op::log_softmax_fwd, false, double, float, mshadow::half::half_t, int, 2>(mshadow::Stream<mshadow::cpu>*, float*, mshadow::half::half_t*, int*, mshadow::Shape<2>, int, float) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`002-...Softmax<log_softmax_fwd, false, double, float, half_t, int, 2>...-annotate.txt`，libmxnet.so，含 3 个 hot loop：max reduction loop、exp-sum loop、normalize(log) loop；5753 samples，event=cpu-clock）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-activation.txt`，全局计数）
- workload/binary/DSO/source context：已提供（metadata JSON：mxnet-opperf @ commit b84609d3fc73d20929c114eab95faaa56e6c5ede；libmxnet.so ELF64 RISC-V）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries.libmxnet.so-elf-A`，见 Phase 1）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（metadata `cpuinfo.isa`，见 Phase 1）
- `vlenb`：已提供（vector.vlen_bits=128，vlenb=16）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=cpu-clock；`percent: local period`；函数 workload 贡献未知）
- Sampling IP precision：缺失（`precise_ip`/Exact-IP 未提供）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`（含 `v`、`zfh`、`zfhmin`；SOPHGO SG2044，XuanTie C920v2，out-of-order） |
| Build ISA | `Tag_RISCV_arch: "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0"`（含 `v1p0`/`zvl128b`，**无 `zfh`/`zfhmin`**） |
| Vector flavor | annotate 内无 `v*` 也无 `th.v*`（全 scalar）；build 有 `v` 而主循环零向量指令 → 未向量化信号；无 flavor mismatch |
| VLEN | 128 bits（vlenb=16；与 build `zvl128b1p0` 一致） |
| Bound type | 全局 IPC=0.562438；annotate 热点集中在 expf@plt（39.98%）与 log@plt（44.27%）调用点（合计 84.25%）及 max-loop compare-branch（10.01%）；本地 unit-stride 顺序访问 → compute/transcendental-call 主导。cache_references≈cache_misses 异常（22,879,522,533/22,879,524,850），cache_miss_rate=100% 不可信 |
| Sampling semantics | event=`cpu-clock`；percent type=`local period`；函数 workload 贡献未知 → 只能作为当前 event 的**局部样本份额**，不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `precise_ip`/Exact-IP 未提供 → `baseline_gap: sampling IP precision`；单行只锚定所属 basic block / loop interval |

L0 baseline gate 判定：
1. hardware 有 `v`，build 有 `v1p0` → 无 v-mismatch；build 具备 V 而主循环零 `v*` → 未向量化信号（由更具体 normalization row 认领）。
2. **hardware 有 `zfh`/`zfhmin`，build 无 `zfh`** → L0 类 baseline finding：FP16 输出走软件 float2half 位操作序列；修正方向 build flags 增补 zfhmin + 原生 `fcvt.h.s`（ISA-substitution row）。
3. `th.v*` gate：annotate 无 `th.v*`，不适用。
Bound-type gate：transcendental 调用主导，非 memory-bound → 本地 RVV normalization fix 的 impact 不被 memory 因子压制；受 Sampling IP precision 缺失约束。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`Softmax<log_softmax_fwd, false, double, float, mshadow::half::half_t, int, 2> [clone ._omp_fn.0]`（libmxnet.so @ 0x15b1188）。

三个 hot loop interval 与 trace anchor（最高占比行）：
1. **max reduction loop**：0x15b1286–0x15b12a0（`for j: mmax = max(mmax, in[base+j*sa])`）。anchor：`10.01 :   15b1296:  beqz  a5,15b129c`（`flt.s` 结果驱动条件分支），配套 `0.56 : 15b1294: add a0,a0,s9`。
2. **exp-sum loop**（temperature==1.0 快路径）：0x15b12b6–0x15b12d2。anchor：`39.98 :   15b12c2:  auipc  ra,0xfefee`（→ `jalr -194(ra) # expf@plt`，scalar libm expf）。
3. **normalize+store loop**（log_softmax map 循环）：0x15b133c–0x15b13b2。anchor：`44.27 :   15b1348:  auipc  ra,0xfeff6`（→ `jalr -840(ra) # log@plt`，scalar libm log）；`log_softmax_fwd::Map<float> = a - log(b)`（0x15b1340 `fmv.d fa0,fs4` 1.58% + 0x15b1356 `fsub.d`）；输出经 `fcvt.s.d` → 软件 float2half（0x15b1362 `and a3,s2,a4` 0.80% 起，至 `sh a5,0(s5)` 0.14%）。
另存在 temperature≠1.0 冷分支（0x15b1414–0x15b1540），样本 0.00%，不作为热点证据。

hot loop 覆盖完整；Sampling IP precision 未知 → 单指令样本只锚定区间。

## Phase 3 — Pattern scan / 模式扫描：Softmax<log_softmax_fwd, false, double, float, half_t, int, 2>

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude**：compiler-generated（OpenMP clone + libm PLT 调用），无 `.S`/DWARF 证据；无 policy/existence 四证。
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar 算子循环（Softmax/LogSoftmax 属 cross-lane normalization），第一级主归属 class。
3. `rows-string-memory.md` — **exclude**：非 string/memory 语义。
4. `rows-vectorized-tuning.md` — **exclude**：hot main loop 零 `v*`（全 scalar）。
5. `rows-codegen.md` — **include**：float→double→float 转换链（`fcvt.d.s`/`fcvt.s.d`）、软件 float2half（20+ 条位操作含 branch diamond）、scalar compare-branch 实现 max（`flt.s`+`beqz`+`fmv.s`）。
6. `rows-offload.md` — **exclude**：无矩阵引擎/packed-SIMD 信号。
7. `rows-crypto.md` — **exclude**：无密码学原语。
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel 热点。

Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: `Softmax<log_softmax_fwd, false, double, float, half_t, int, 2>`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Normalization Kernels（primary） | scalar 3-pass LogSoftmax（max→exp-sum→log-normalize）；expf@plt 39.98% + log@plt 44.27% = 84.25% 局部样本；max-loop compare-branch 10.01%；hardware V + build V 但主循环零 `v*` | High | Medium | `patterns/rvv_normalization_kernels.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（independent，zfhmin 分支） | 软件 float2half 序列（`and` 0.80%、`and` 0.38%、`srliw`/`sext.w` 0.09%、`addiw` 0.07%、`slliw` 0.19%、`srliw` 0.09%、`sh` 0.14% 等约 2%）；hardware 有 zfh/zfhmin、build 无 zfh | Medium | Low | `patterns/isa_extension_specific_instruction_substitution.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（supporting，fmax.s 分支） | 同一 max-reduction 机制的次级命中：`flt.s`+`beqz` 可用 base-F `fmax.s` 单指令替代 | —（supporting） | —（supporting） | `patterns/isa_extension_specific_instruction_substitution.md` |

### 顶层命中 row 三件套

**Finding 1（primary）：RVV Normalization Kernels**
- **(a) 逐字 evidence 引用**：
  - `39.98 :   15b12c2:        auipc   ra,0xfefee`（exp-sum loop interval 0x15b12b6–0x15b12d2 内 expf@plt 调用点）
  - `44.27 :   15b1348:        auipc   ra,0xfeff6`（normalize+store loop interval 0x15b133c–0x15b13b2 内 log@plt 调用点）
  - `10.01 :   15b1296:        beqz    a5,15b129c`（max loop interval 0x15b1286–0x15b12a0 内比较分支）
  - 合计 transcendental 调用点 84.25%，3-pass 流程约 99% 局部样本。
- **(b) 互斥邻居排除**：
  - `elementwise-activation` row：含跨 lane statistics（max、sum）与多遍扫描，非 lane-independent activation；行内互斥「Softmax/LayerNorm/RMSNorm → normalization row」；排除。
  - `extrema-reduction` / `widening-additive-reduction` row：max/sum loop 只是 LogSoftmax 阶段，完整流程含 log 变换；排除。
  - `precision-conversion` row：float2half 是输出 epilogue 子步骤，热点主导在 expf/log 调用点（84.25%）；排除独立命中。
  - `no-vectorization` row：行内互斥「operator semantic shape → 各自更具体 row」——被 normalization row 认领；排除独立认领。
  - `kernel-operation-fusion` row：3-pass 中间统计量只留寄存器（`fs4` sum、`fs0` mmax），无「中间结果写回再读入」内存往返；排除。
  - `eliminate-unnecessary-precision-conversions` row：`fcvt.d.s`/`fcvt.s.d` 包住 double 运算（sum 累加、`a - log(b)`），但 AType=double 是模板显式参数（数值设计合同），非「完全不必要的往返」；排除独立命中，保留为 correctness contract。
- **(c) 双 Confidence 推导式**：`route: scalar 3-pass LogSoftmax + hardware V(zve64f/zvl128b) + build V(v1p0) + 语义合同与热点地址分账清晰 → High；impact: 局部样本份额（expf+log 84.25%、3-pass ≈99%）但 percent=local period、workload 贡献未知、`baseline_gap: sampling IP precision` → Medium`。

**Finding 2（independent）：ISA Extension-Specific Instruction Substitution（zfhmin 分支）**
- **(a) 逐字 evidence 引用**：
  - `0.80 :   15b1362:        and     a3,s2,a4`（normalize+store loop 内软件 float2half 序列）
  - `0.14 :   15b13a8:        sh      a5,0(s5)`（float2half 结果写回 half 输出）
  - `10.01 :   15b1296:        beqz    a5,15b129c`（max loop compare-branch，`fmax.s` 可替代）
- **(b) 互斥邻居排除**：
  - `floating-point-semantic-lowering` row：float2half 为保持 rounding/NaN 语义的位操作，但行内互斥「纯 Zfa/Zfh native instruction substitution → ISA-substitution row」——hardware 有 zfhmin（`fcvt.h.s` 原生）；排除 semantic-lowering 独立命中。
  - `eliminate-unnecessary-precision-conversions` row：float2half 是 FP32→FP16 格式转换（输出类型合同），非 float32↔float64 往返；排除。
  - primary normalization finding：机制（FP16 转换指令选择 vs transcendental 调用）与修复对象可分账；向量化后输出仍为 half_t → independent。
  - `no-vectorization` row：ISA-substitution 不解释「未向量化」；排除。
- **(c) 双 Confidence 推导式**：`route: hardware zfh/zfhmin 确认 + build 缺 zfh（build-flags 修正前置）+ fmax.s 属 base F → Medium；impact: 局部样本份额 float2half 段约 2%（0.80+0.38+0.19+0.14+0.09×2+0.07 等）、max 分支链计入 primary → Low`。

**Finding 3（supporting，并入 Finding 1）：ISA Extension-Specific Instruction Substitution（fmax.s 分支）**
- (a) `10.01 : 15b1296: beqz a5,15b129c` + `0.56 : 15b1294: add a0,a0,s9`；supporting because: max reduction 的 scalar compare-branch 链是 normalization 第一阶段，向量化（`vfredmax`）或 `fmax.s` 修正消除同一机制；不另计顶层命中。
- (b)(c) 共享机制，confidences 记 `—`。

### 多命中仲裁小段
- **primary**：RVV Normalization Kernels（L1 layer）——未向量化 cross-lane 归一化 + scalar expf/log 调用。
- **supporting**：ISA-substitution（fmax.s 分支，L4）——上层向量化修复会让其 signal 消失，并入 supporting。
- **independent**：ISA-substitution（zfhmin `fcvt.h.s` 分支，L4）——evidence 落在 float2half 转换指令段（0x15b135e–0x15b13a8），与 expf/log 调用点不相交；机制/修复对象/验证预测可分账。
- 入口条件 A 局部份额排序：primary ≈99%（3-pass）＞ independent ≈2%；机制分账清晰，无叠加。

## Phase 4 — Root-cause blueprint / 根因蓝图：Softmax<log_softmax_fwd, false, double, float, half_t, int, 2>

### Finding 1（primary）：RVV Normalization Kernels（`patterns/rvv_normalization_kernels.md`；Phase 3 Finding 1 通过 gate）

1. **Root cause**：LogSoftmax（log_softmax_fwd，Map = `a - log(b)`）是 cross-lane normalization（max → exp(x-mmax) → sum → log 变换），hot path 完全未向量化：sum 阶段逐元素 scalar libm `expf@plt`（39.98%）、map 阶段逐元素 scalar libm `log@plt`（44.27%）、max 阶段 scalar compare-branch（10.01%）。机制句引用（依据 `patterns/rvv_normalization_kernels.md` §Why this is slow）：「归一化的根因是跨 lane reduction 造成依赖链，多阶段完整扫描造成额外流量，或 reciprocal/rsqrt/vector-math 不可达而退回 scalar helper」——本函数命中「vector-math 不可达退回 scalar helper」（expf/log）与「跨 lane reduction 依赖链」。
2. **The fix / 修复方式**（与该文件 §The fix §5/§6/§7 一致）：
   - RVV 1.0 向量 kernel（build 已含 `v1p0`/`zvl128b`），保持三阶段显式语义（§7）：
   - 第一阶段：`vle32` + `vfredmax.vs`（§5 seed 式分块归约），mmax 语义（NaN/Inf/全相等/空输入）按 reference 保留。
   - 第二阶段：`vle32` → `vfsub.vf`（减 mmax）→ 向量 exp（项目已验证 RVV exp；`references/vector-math-conventions.md`：exp 的 availability、特殊值、误差、mask-safe dataflow）→ `vfredusum.vs` 归约（§5：`vfredusum` 改变 FP 结合顺序；需有序语义时评估 `vfredosum`；AType=double 累加合同单独验证）。
   - 第三阶段：LogSoftmax 变换为 `x - mmax - log(sum)`——向量化 `log(sum)`（scalar 一次）+ 向量 `vfsub`（每元素减）；或向量 log 实现（项目已验证）逐元素 `vfsub`+`vflog`。§7 要求对 exp overflow/underflow、指数和为零/Inf/NaN、LogSoftmax consistency（`log_softmax = log(softmax)` 数值恒等式）单独验证。
   - **修复前**（scalar）：
     ```cpp
     float mmax = in[base];
     for (j=1; j<M; ++j) mmax = max(mmax, in[base+j*sa]);      // 10.01% 局部样本
     double sum = 0;
     for (j=0; j<M; ++j) sum += std::exp(in[base+j*sa] - mmax); // 39.98% 局部样本
     for (j=0; j<M; ++j) out[base+j*sa] = half_t(float(in[base+j*sa] - mmax - std::log(sum))); // 44.27% 局部样本
     ```
   - **修复后**（RVV，伪代码，LMUL 按 §2 预算）：
     ```cpp
     vfloat32m1_t v = vle32(in+base, vl);
     float mmax = vfmv_f_s(vfredmax(vseed, v, vl));            // 阶段 1
     v = vfsub_vf(vle32(...), mmax, vl);
     vfloat32m1_t e = rvv_exp(v, vl);                           // 项目已验证 RVV exp
     float sum = vfmv_f_s(vfredusum(vseed, e, vl));            // 阶段 2
     float lsum = (float)std::log(sum);                         // scalar log(sum)（1 次）
     v = vfsub_vf(v, lsum, vl);                                 // 阶段 3: x-mmax-log(sum)
     vse16(out+base, vfncvt_f_f_w(v, vl), vl);                  // 输出（zfhmin 见 Finding 2）
     ```
   - **适用前提**：build 已含 V；`M`（axis 长度）须大于向量化 crossover；短 axis 保留 scalar 或 `vl` 截断。
   - **不可破坏的 correctness contract**：稳定减 max；NaN/±Inf/全相等/全 -Inf/+Inf/指数和为零/溢出下溢；`count==0`；`vfredusum` FP 结合顺序（AType=double 合同）；LogSoftmax consistency（sum≈1 时 log 输出）；temperature 分支（`==1.0` 快路径 vs `/temperature` 慢路径，当前 0x15b12a4 分支）必须保留。
   - **限制/风险**：RVV exp/log 的 ULP 误差 vs libm；向量化提高 live register 压力（§2：LMUL×peak_live_vectors≤32）；fixed-VL main loop + runtime tail（`references/kernel-conventions.md` §3）。
   - **预期变化的 Profile signals**：`expf@plt`（`15b12c2`）与 `log@plt`（`15b1348`）调用点样本大幅下降/消失；`beqz`（`15b1296`）消失；出现 `vle32/vfredmax/vfredusum/v*exp/vse16`；cycles/row 或 IPC 改善。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv…zfh_zfhmin…（含 v, zve64f, zvl128b）；build ISA = rv64…v1p0…zvl128b（含 v，无 zfh）；VLEN=128；bound type = compute/transcendental 主导。
4. **收益上界**：当前 sampled event 下**局部样本份额**：expf 39.98% + log 44.27% = 84.25% 调用点；3-pass 流程约 99%。不构成 workload 级 Amdahl 上界（percent=local period、workload 贡献未知）。
5. **三维路由判定**：`current source` = compiler-generated scalar → intrinsic/编译器向量化路径；`implementation existence/reachability` = 无既有 RVV kernel/dispatch 证据，需新写 intrinsic kernel；`function-level policy` = 无独立 `.S` policy 证据，不走 missing-`.S` 分支。
6. **Implementation-shape proof**：不适用（非 missing `.S` 分支）。
7. **Related PRs**：Related PRs：17 条 URL —— oneDNN [36df719b3bc9](https://github.com/uxlfoundation/oneDNN/commit/36df719b3bc9001d994455c2899e42533b2c29ac)、[6dc3e2d84eec](https://github.com/uxlfoundation/oneDNN/commit/6dc3e2d84eec45eff0834e8985cc663db6fe07ee)、[3b90f9d0650f](https://github.com/uxlfoundation/oneDNN/commit/3b90f9d0650f4087d4a6fbcb3406dc0862f2ae25)、[#4809](https://github.com/uxlfoundation/oneDNN/pull/4809)、[#4622](https://github.com/uxlfoundation/oneDNN/pull/4622)、[#4453](https://github.com/uxlfoundation/oneDNN/pull/4453)、[#4480](https://github.com/uxlfoundation/oneDNN/pull/4480)、[0d8f4a9e702b](https://github.com/uxlfoundation/oneDNN/commit/0d8f4a9e702b70f6188f58a20247ae06dbd41ce5)、[7ad03324c2d1](https://github.com/uxlfoundation/oneDNN/commit/7ad03324c2d17480e40cc20114576584888373ce)、[3c8c37bab64f](https://github.com/uxlfoundation/oneDNN/commit/3c8c37bab64f67bbdebcdb76330db0dcb25b31af)、[c07b7f4e7776](https://github.com/uxlfoundation/oneDNN/commit/c07b7f4e777636cf527f49093c03f8cacf58b3de)、[#4734](https://github.com/uxlfoundation/oneDNN/pull/4734)、[#4491](https://github.com/uxlfoundation/oneDNN/pull/4491)、[25cd4a75095a](https://github.com/uxlfoundation/oneDNN/commit/25cd4a75095a2aebeb2f0140abadd466cd0b4e90)、[b8360ec0a1d5](https://github.com/uxlfoundation/oneDNN/commit/b8360ec0a1d566f039d4fe286349c7988a762c98)；MNN [#4508](https://github.com/alibaba/MNN/pull/4508)、[#4044](https://github.com/alibaba/MNN/pull/4044)、[b7268aa](https://github.com/alibaba/MNN/commit/b7268aa3dab754190ed7d86dc16b9bab02e73d12)。

### Finding 2（independent）：ISA Extension-Specific Instruction Substitution（`patterns/isa_extension_specific_instruction_substitution.md`；Phase 3 Finding 2 通过 gate）

1. **Root cause**：输出转换与 max 比较用多指令合成序列。机制句引用（依据 `patterns/isa_extension_specific_instruction_substitution.md` §Why this is slow）：「When the compiler does not exploit the specific extension available on the target CPU, it must fall back to synthesizing the operation from base integer or floating-point instructions」——软件 `float2half`（`mshadow::half::half_t::float2half`，sign 提取 `and`/`srliw`、`maxZ/minN/maxN/infN/nanN` branch diamond、rounding `snez/slliw/addw`、`srliw 0xd`+`or`，0x15b135e–0x15b13a8 与 0x15b14d4–0x15b1540 两处复制）可由 zfhmin 单条 `fcvt.h.s` 替代；max 的 `flt.s`+`beqz`+`fmv.s` 由 base-F `fmax.s` 替代（§5）。
2. **The fix / 修复方式**：
   - 前置（L0 类）：build ISA 增补 `zfhmin`（`-march=rv64gcv_zfhmin`）；扩展宏保护（`references/kernel-conventions.md` §Extension macro guards）：`#if defined(__riscv_zvfhmin) || defined(__riscv_zfhmin)` 启用原生路径，否则保留软件 fallback。
   - 修复前：`uint16_t float2half(float v) { /* ~20 条指令位操作 + 分支 */ }`。
   - 修复后：`#if defined(__riscv_zfhmin) || defined(__riscv_zvfhmin) _Float16 h = (_Float16)f; /* fcvt.h.s */ #else uint16_t h = float2half(f); #endif`。
   - max 分支：`flt.s`+`beqz`+`fmv.s` → `fmax.s`（base F，build 已有 f2p2）。
   - **适用前提**：hardware zfh/zfhmin 已确认；build flags 修正后 clean rebuild；fallback 保留。
   - **不可破坏的 correctness contract**：FP16 rounding（`MSHADOW_HALF_ROUND_TO_NEAREST` vs `fcvt.h.s` frm=RNE 对齐验证）；NaN/Inf/subnormal/±0（0x7c00/0x8000/0x7e00 边界）；fp16 溢出 → inf 语义对比。
   - **限制/风险**：依赖 build flags 修正；zfhmin 只含转换指令，本项目输出路径只需转换 → 足够。
   - **预期变化的 Profile signals**：0x15b135e–0x15b13a8（及 0x15b14d4–0x15b1540）位操作/branch 样本消失，出现 `fcvt.h.s`；max loop `beqz`（0x15b1296）被 `fmax.s` 替代后 branch-miss 下降。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv…zfh_zfhmin…；build ISA = rv64…v1p0（无 zfh，修正前置）；VLEN=128；bound type = compute 主导。
4. **收益上界**：局部样本份额 float2half 段约 2%（`15b1362` 0.80%、`15b136c` 0.38%、`15b139a` 0.19%、`15b13a8` 0.14%、`15b138a` 0.07%、`15b1380` 0.05%、`15b1370` 0.05%、`15b13a2` 0.09% 等）；不构成 workload 级上界。局部份额排序：primary 99% > independent 2%。
5. **三维路由判定**：`current source` = compiler-generated（模板代码 + 指令选择）；`implementation existence/reachability` = `fcvt.h.s`/`fmax.s` 硬件可达，build 修正后指令可达；`function-level policy` = 无独立 policy 约束。
6. **Implementation-shape proof**：不适用。
7. **Related PRs**：Related PRs：59 条 URL —— Go [3659b8756a2b](https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0)、[a6ecdf29e34d](https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c)、[#59488](https://github.com/golang/go/pull/59488)、[63ab68ddc5f1](https://github.com/golang/go/commit/63ab68ddc5f1307e552cf27ae7a6f0dfda2bb962)、[1951afc9193f](https://github.com/golang/go/commit/1951afc9193f8e197cb7dfaf6afed70ea02404cb)；OpenCV [a00818047ff5](https://github.com/opencv/opencv/commit/a00818047ff5671586c7b296cac6175b250f85d3)、[7e2c8cc9f4c4](https://github.com/opencv/opencv/commit/7e2c8cc9f4c49a3afeee084a2f01f7bac1d9a2dc)、[f0d29cd33c5d](https://github.com/opencv/opencv/commit/f0d29cd33c5d2b8f6c9c5c3177cbb3a359ee6b33)、[#21351](https://github.com/opencv/opencv/pull/21351)；OpenSSL [03ce37e11729](https://github.com/openssl/openssl/commit/03ce37e117)、[ca6286c382a7](https://github.com/openssl/openssl/commit/ca6286c382)、[48b6776678d7](https://github.com/openssl/openssl/commit/48b6776678)、[6136408e6abf](https://github.com/openssl/openssl/commit/6136408e6a)、[e4fd3fc379d7](https://github.com/openssl/openssl/commit/e4fd3fc379)、[80c664db430d](https://github.com/openssl/openssl/commit/80c664db43)、[08c8dd6b8ced](https://github.com/openssl/openssl/commit/08c8dd6b8cede3cdbe5b1866c1a7544e0fe7a378)、[49a3e7adc392](https://github.com/openssl/openssl/commit/49a3e7adc3)、[a41f9135f082](https://github.com/openssl/openssl/commit/a41f9135f0)、[4dbb537bd1ea](https://github.com/openssl/openssl/commit/4dbb537bd1)、[608cadfbdbdb](https://github.com/openssl/openssl/commit/608cadfbdb)、[b1b889d1b3fc](https://github.com/openssl/openssl/commit/b1b889d1b3)、[657d1927c68b](https://github.com/openssl/openssl/commit/657d1927c6)、[611685adc04a](https://github.com/openssl/openssl/commit/611685adc0)、[7ae2bc9df6e0](https://github.com/openssl/openssl/commit/7ae2bc9df6)；Linux Kernel RISC-V [e8620bd7e5e0](https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712)、[5ba15d419fab](https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49)、[cc2294d3f9c9](https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92)、[36e224168721](https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de)、[e11e367e9fe5](https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355)、[75ab93a244a5](https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb)、[c64086849110](https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68)；OpenJDK [6b89954c6534](https://github.com/openjdk/jdk/commit/6b89954c65342bc601633d24075dab4f4b248f4b)、[a7631ccf18e4](https://github.com/openjdk/jdk/commit/a7631ccf18e468d6ecba121865f7fed29cbf2186)、[#22752](https://github.com/openjdk/jdk/pull/22752)、[08d563ba1504](https://github.com/openjdk/jdk/commit/08d563ba15047020fd5f5fea80547e18898bbab2)、[b1a21b563e3a](https://github.com/openjdk/jdk/commit/b1a21b563e3ae13fa5c409a4f0c04686c3f5b34a)、[#22410](https://github.com/openjdk/jdk/pull/22410)、[9f582e56baee](https://github.com/openjdk/jdk/commit/9f582e56baee0e7f5af20da0f395cd935bf5a962)、[#24096](https://github.com/openjdk/jdk/pull/24096)、[1a4bbb0027ae](https://github.com/openjdk/jdk/commit/1a4bbb0027ae9e6df3b668454fa155861d531f72)、[2ed7ad4b5c7d](https://github.com/openjdk/jdk/commit/2ed7ad4b5c7d2344ae6571c186f8a2903770aa57)、[edfe28541a6e](https://github.com/openjdk/jdk/commit/edfe28541a6ed94357f873aa69778c7eba707cbb)、[b891bfa7e67c](https://github.com/openjdk/jdk/commit/b891bfa7e67c21478475642e2bfa2cdc65a3bffe)、[3d3b78203710](https://github.com/openjdk/jdk/commit/3d3b7820371058b40f2e694536c98aa3900abb5f)、[a7a09f69abc6](https://github.com/openjdk/jdk/commit/a7a09f69abc6c4730599d3de9067c2fde75c5172)、[bcc33d5ef3bd](https://github.com/openjdk/jdk/commit/bcc33d5ef3bdbfaee51c45014851c54028da03f1)、[#22386](https://github.com/openjdk/jdk/pull/22386)、[5866b16dbca3](https://github.com/openjdk/jdk/commit/5866b16dbca3f63770c8792d204dabdf49b59839)、[8cb9b479c529](https://github.com/openjdk/jdk/commit/8cb9b479c529c058aee50f83920db650b0c18045)、[#11921](https://github.com/openjdk/jdk/pull/11921)；llama.cpp [#17784](https://github.com/ggml-org/llama.cpp/pull/17784)；V8 [6f100865663f](https://github.com/v8/v8/commit/6f100865663fb99df2628144fc65977e55ab7e68)、[e62c1e307d20](https://github.com/v8/v8/commit/e62c1e307d207e1e56219c172929289a1b474530)、[200b5212ae47](https://github.com/v8/v8/commit/200b5212ae4752f6345dbcc6ecf23f651434bbf8)、[5136fb5200c1](https://github.com/v8/v8/commit/5136fb5200c1f3a33939419fed9de32e8d85bc1e)、[e02d2238f6ea](https://github.com/v8/v8/commit/e02d2238f6ea6f920a6ac31f888b4727bb0b0f80)、[94a3c420e2a4](https://github.com/v8/v8/commit/94a3c420e2a4f76d42e367e7d3b3f85ecd1b0a3f)、[1818e36d54d1](https://github.com/v8/v8/commit/1818e36d54d12540080dd58dbfb924155a920cba)、[5f433dd5024a](https://github.com/v8/v8/commit/5f433dd5024a566fa051317dd0c9c1d9921a9162)、[9bbbde26bb90](https://github.com/v8/v8/commit/9bbbde26bb90bd493afe33777d7b6be0c4f2afb4)、[758956654f28](https://github.com/v8/v8/commit/758956654f28861ba0d5e94f03fff6016b3a2998)、[32c5d22333c1](https://github.com/v8/v8/commit/32c5d22333c14a9d6bb645af3ea4d3fb28b541fe)、[89719bc239f4](https://github.com/v8/v8/commit/89719bc239f48735bbffe83e0803fd504b3c485a)、[223d7fb26b12](https://github.com/v8/v8/commit/223d7fb26b12145bdd335d51da2fa2ed1c58ce3a)、[b2852080c401](https://github.com/v8/v8/commit/b2852080c401cb0980f332918531597967de36c1)、[8b842dbb9f6d](https://github.com/v8/v8/commit/8b842dbb9f6d7ea03990d80dc945ef8c6c6ca144)、[e855c14cab2c](https://github.com/v8/v8/commit/e855c14cab2c6522a2a2d9b6be51e886fd3aae74)；QEMU [3de1fb712a07](https://github.com/qemu/qemu/commit/3de1fb712a072992d72bc99c2b70978132ee44d0)、[6ef584318238](https://github.com/qemu/qemu/commit/6ef5843182382f6a84995590ad91047b0f2bc1fa)；V8（bitmanip 系列）[8033cc56e2af](https://github.com/v8/v8/commit/8033cc56e2afb7b19214aa9a6f776a59fc509d2c)、[a654e27b50fe](https://github.com/v8/v8/commit/a654e27b50feb457a99ad27582dd555316eab587)、[21289aa92c80](https://github.com/v8/v8/commit/21289aa92c80a4161cc5f0d9915a0002a644d8b4)、[19a0f69f4c4b](https://github.com/v8/v8/commit/19a0f69f4c4b1257b567290412acda411568d565)；LLVM [#170824](https://github.com/llvm/llvm-project/pull/170824)、[4c1e1e05cb90](https://github.com/llvm/llvm-project/commit/4c1e1e05cb901a2ed9055e5d6ac6ce60b826a288)、[#92926](https://github.com/llvm/llvm-project/pull/92926)、[#152744](https://github.com/llvm/llvm-project/pull/152744)、[#122698](https://github.com/llvm/llvm-project/pull/122698)、[13e32a8a3c95](https://github.com/llvm/llvm-project/commit/13e32a8a3c95b23af51f081865db1bd259d269a4)、[787eeb8597fa](https://github.com/llvm/llvm-project/commit/787eeb8597fac22decb366a42176b11f52ec1bf0)、[c705b7b04dba](https://github.com/llvm/llvm-project/commit/c705b7b04dba467a67871a1bbb77907d0ed7fc19)。

## Phase 5 — Verification forecast / 验证预测：Softmax<log_softmax_fwd, false, double, float, half_t, int, 2>

**Finding 1（primary）——修复对象：3-pass scalar LogSoftmax → RVV 向量化（max/sum reduction + 向量 exp + log 变换）**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`39.98 : 15b12c2: auipc ra,0xfefee`（sum-loop expf 调用点）、`44.27 : 15b1348: auipc ra,0xfeff6`（map-loop log 调用点）、`10.01 : 15b1296: beqz a5,15b129c`（max-loop 条件分支）——重建后重新 annotate：这三行样本应大幅下降或消失；`flt.s`/`fcvt.d.s`/`fadd.d`/`fsub.d`/`fcvt.s.d` scalar 序列消失。
- 应出现侧（依据 `patterns/rvv_normalization_kernels.md` §Verification）：出现 vector reduction + transform（`vle32`、`vfredmax.vs`、`vfredusum.vs`、`vfsub.vf`、向量 exp/log、`vse16`）；cycles/row 或 throughput 改善；正确性合同：Softmax 覆盖全相等、极端 logits、NaN/Inf、sum≈1、LogSoftmax consistency 与稳定减 max；最后 reduction 的 `vl`/tail/ordered-unordered FP 语义与误差明确。
- 验证范围：动态 `M` 覆盖 0、小 M、main-loop 整倍数与全部 tail；短/中/长 axis benchmark；FP reordering 容差（`vfredusum` unordered vs double 累加 reference）说明。
- 升级采样：`perf annotate --stdio -l -s <func> --percent-type=global-period`；核对 `precise_ip`。

**Finding 2（independent）——修复对象：软件 float2half → zfhmin `fcvt.h.s`；max compare-branch → `fmax.s`**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`0.80 : 15b1362: and a3,s2,a4`、`0.14 : 15b13a8: sh a5,0(s5)`（float2half 序列 0x15b135e–0x15b13a8 及 0x15b14d4–0x15b1540）——转换为 `fcvt.h.s` 后消失；`10.01 : 15b1296: beqz`（max-loop branch）在 `fmax.s` 替换后消失。
- 应出现侧（依据 `patterns/isa_extension_specific_instruction_substitution.md` §Verification）：disassembly 确认 `fcvt.h.s`/`fmax.s`；无扩展 flag 编译时 fallback 重现；FP16 边界（0x7c00 inf、NaN 0x7e00、subnormal、±0、RNE 与 `MSHADOW_HALF_ROUND_TO_NEAREST` 对齐）；±0 与 NaN 传播按行内表验证。
- 升级采样：补 `perf stat -e branch-misses` 验证 max-loop branch 消除收益。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`Softmax<log_softmax_fwd, false, double, float, half_t, int, 2>` |
| 2 | Phase 1 输出要求 | ✅ | 7 行 baseline；`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ | Class selection trace 8 项（include: rows-operator-rvv.md, rows-codegen.md）；Classes scanned 同前；顶层 finding 2（primary normalization + independent ISA-substitution）+ supporting 1（fmax.s）；evidence 锚点 `15b12c2:39.98%`、`15b1348:44.27%`、`15b1296:10.01%`、`15b1362:0.80%`、`15b13a8:0.14%`；排除 9 条；推导式 2 条 |
| 4 | Phase 4 输出要求 | ✅ | 已读 pattern：`patterns/rvv_normalization_kernels.md`（引用「归一化的根因是跨 lane reduction…退回 scalar helper」；§5/§6/§7；fix before/after、correctness、风险、Profile signals；Related PRs：17 条 URL）、`patterns/isa_extension_specific_instruction_substitution.md`（引用「fall back to synthesizing the operation from base integer or floating‑point instructions」；§5；fix before/after、correctness、风险、Profile signals；Related PRs：59 条 URL） |
| 5 | 路径合规 | ✅ | 入口模式 A；primary(L1)+supporting(L4 fmax.s)+independent(L4 zfhmin)；局部份额排序 primary≈99% > independent≈2% |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`15b12c2`/`15b1348`/`15b1296`/`15b1362`/`15b13a8`；出现侧：`rvv_normalization_kernels.md §Verification`、`isa_extension_specific_instruction_substitution.md §Verification` |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；交付止于证据、蓝图、The fix、验证预测 |

修正记录：无