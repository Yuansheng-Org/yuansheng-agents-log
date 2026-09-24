Functions under analysis: [void mxnet::op::mxnet_op::Softmax<mxnet::op::mxnet_op::softmax_fwd, false, double, float, mshadow::half::half_t, int, 2>(mshadow::Stream<mshadow::cpu>*, float*, mshadow::half::half_t*, int*, mshadow::Shape<2>, int, float) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`001-...Softmax<softmax_fwd, false, double, float, half_t, int, 2>...-annotate.txt`，libmxnet.so，含 3 个 hot loop：max reduction loop、exp-sum loop、normalize+store loop；5813 samples，event=cpu-clock）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-activation.txt`，全局计数）
- workload/binary/DSO/source context：已提供（metadata JSON：mxnet-opperf @ commit b84609d3fc73d20929c114eab95faaa56e6c5ede；libmxnet.so ELF64 RISC-V，Tag_RISCV_arch 见 Phase 1）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries.libmxnet.so-elf-A`，见 Phase 1）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（metadata `cpuinfo.isa`，见 Phase 1）
- `vlenb`：已提供（vector.vlen_bits=128，vlenb=16）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=cpu-clock；header 标注 `percent: local period`；同一窗口/函数 workload 贡献未知 → 局部份额语义成立，workload 级 Amdahl 上界不成立，详见 Phase 1）
- Sampling IP precision：缺失（`precise_ip`/Exact-IP 未提供；详见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`（含 `v`、`zfh`、`zfhmin`、`zfa`、`zba/zbb/zbs`；SOPHGO SG2044，XuanTie C920v2，out-of-order） |
| Build ISA | `Tag_RISCV_arch: "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0"`（含 `v1p0`/`zvl128b`，**无 `zfh`/`zfhmin`**） |
| Vector flavor | annotate 内无任何 `v*` 也无 `th.v*`（全 scalar）；build 有 `v` 而主循环零向量指令 → 未向量化信号；无 flavor mismatch（build 即 RVV 1.0 目标） |
| VLEN | 128 bits（vlenb=16；与 build `zvl128b1p0` 一致） |
| Bound type | 全局 IPC=0.562；annotate 热点几乎全部集中在 expf@plt 调用点（83.21%）与 max-loop 比较分支（9.51%），本地访问为 unit-stride 顺序（`flw 0(a3)`/`add s10,s10,s9`），未见长依赖 cache 段 → compute/transcendental-call 主导。注意 perf stat 中 `cache_references`≈`cache_misses`（22,879,522,533 / 22,879,524,850）数值异常，cache_miss_rate=100% 不可信，bound 判断以 annotate 指令形态为主 |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=`local period`（函数内局部份额）；采样窗口/函数 workload 贡献未知 → 百分比只能作为当前 event 的**局部样本份额**，不得解释为 workload 级 Amdahl 上界 |
| Sampling IP precision | `precise_ip`/Exact-IP/skid 能力未提供 → `baseline_gap: sampling IP precision`；单行高占比只锚定所属 basic block / loop interval，不做 instruction-latency 归因 |

L0 baseline gate 判定：
1. **hardware 有 `v`，build 也有 `v1p0`** → 不存在 "hardware 有 v 而 build 无 v" 的 L0 mismatch；但 build 具备 V 能力而主循环零 `v*` 指令，属**未向量化**信号（由更具体的 normalization row 认领，不单独命中 no-vectorization）。
2. **hardware 有 `zfh`/`zfhmin`，build 无 `zfh`** → L0 类 baseline finding（类似 v gate 的扩展）：FP16 转换只能走软件 `float2half` 位操作序列；修正方向为 build flags 增补 zfhmin + 原生 `fcvt.h.s` 指令替换（对应 ISA-substitution row）。
3. `th.v*` gate：annotate 无 `th.v*`，不适用。
Bound-type gate：热点由 transcendental 库调用与 scalar 归约主导，非 memory-bound → 本地 RVV normalization fix 的 impact 判断不被 memory 因子压制；performance-impact 仍受 Sampling IP precision 缺失约束。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`Softmax<softmax_fwd, false, double, float, mshadow::half::half_t, int, 2> [clone ._omp_fn.0]`（libmxnet.so @ 0x18de944）。

三个 hot loop interval 与 trace anchor（最高占比行）：
1. **max reduction loop**：0x18dea40–0x18dea5a（`for j: mmax = max(mmax, in[base+j*sa])`）。anchor：`9.51 :   18dea50:  beqz  a5,18dea56`（`flt.s` 比较结果驱动的条件分支），配套 `0.52 : 18dea4e: add a0,a0,s9`（指针递增）、`0.41 : 18dea44: fmv.s fa5,fs0`。
2. **exp-sum loop**（temperature==1.0 快路径）：0x18dea70–0x18dea8c。anchor：`43.14 :   18dea7c:  auipc  ra,0xfecc0`（→ `jalr 1924(ra) # expf@plt`，scalar libm expf 调用点）。
3. **normalize+store loop**（map 循环）：0x18deaf6–0x18deb68。anchor：`40.07 :   18deafe:  auipc  ra,0xfecc0`（→ `jalr 1794(ra) # expf@plt`）；输出经 `fdiv.d` → `fcvt.s.d` → 软件 `float2half` 序列（0x18deb14 `fmv.x.w` 2.31% 起，至 `sh a5,0(s5)`）。
另存在 temperature≠1.0 冷分支（0x18debc8–0x18def0），样本 0.00%，不作为热点证据。

hot loop 覆盖完整（含 3 个 pass 的循环体）；Sampling IP precision 未知 → 单指令样本只锚定区间。

## Phase 3 — Pattern scan / 模式扫描：Softmax<softmax_fwd, false, double, float, half_t, int, 2>

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude**：当前代码来源为 compiler-generated（OpenMP clone + libm PLT 调用；annotate 无 `.S`/DWARF 证据）；无 dispatch slot / scalar fallback / policy 四证，不走 missing-`.S` 分支。
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar 算子循环（Softmax 属 cross-lane normalization），第一级主归属 class。
3. `rows-string-memory.md` — **exclude**：非 string/memory/compare/checksum 语义。
4. `rows-vectorized-tuning.md` — **exclude**：hot main loop 零 `v*` 指令（全 scalar），该 class 要求已有 `v*` 的 RVV 配置修正对象。
5. `rows-codegen.md` — **include**：出现 float→double→float 转换链（`fcvt.d.s`/`fcvt.s.d`）、软件 FP16 转换（20+ 条位操作含 branch diamond）、scalar compare-branch 实现 max（`flt.s`+`beqz`+`fmv.s`）；需逐行评估 precision-conversion / semantic-lowering / ISA-substitution / fusion / register-pressure 等行。
6. `rows-offload.md` — **exclude**：无矩阵引擎/packed-SIMD/权重重排信号。
7. `rows-crypto.md` — **exclude**：无密码学原语热点。
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel/timer/CSR 热点。

Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: `Softmax<softmax_fwd, false, double, float, half_t, int, 2>`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Normalization Kernels（primary） | scalar 3-pass Softmax（max→exp-sum→normalize）；expf@plt 调用点 83.21% 局部样本；max-loop compare-branch 9.51%；hardware V + build V 但主循环零 `v*` | High | Medium | `patterns/rvv_normalization_kernels.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（independent，zfhmin/fmax 分支） | 软件 float2half 序列 ~3.8% 局部样本（`fmv.x.w` 2.31% 等 20+ 指令/元素）；max 用 `flt.s`+`beqz`+`fmv.s` 合成；hardware 有 zfh/zfhmin、build 无 zfh | Medium | Low | `patterns/isa_extension_specific_instruction_substitution.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（supporting，fmax.s 分支） | 同一 max-reduction 机制的次级命中：`flt.s`+`beqz` 可用 base-F `fmax.s` 单指令替代 | —（supporting） | —（supporting） | `patterns/isa_extension_specific_instruction_substitution.md` |

### 顶层命中 row 三件套

**Finding 1（primary）：RVV Normalization Kernels**
- **(a) 逐字 evidence 引用**（含占比与地址，所属 basic block / loop interval）：
  - `43.14 :   18dea7c:        auipc   ra,0xfecc0`（sum loop interval 0x18dea70–0x18dea8c 内的 expf@plt 调用点）
  - `40.07 :   18deafe:        auipc   ra,0xfecc0`（normalize+store loop interval 0x18deaf6–0x18deb68 内的 expf@plt 调用点）
  - `9.51 :   18dea50:        beqz    a5,18dea56`（max loop interval 0x18dea40–0x18dea5a 内的比较分支）
  - 合计 expf 调用点 83.21%，3-pass 流程约 99% 局部样本。
- **(b) 互斥邻居排除**：
  - `elementwise-activation` row：本函数含跨 lane statistics（max、sum reduction）与多遍扫描，非 lane-independent activation；排除（row 行内互斥原文：「Softmax/LayerNorm/RMSNorm 或统计归约+transform → normalization row」）。
  - `extrema-reduction` row：max loop 只是 softmax 第一阶段，完整流程含 sum+normalize；排除。
  - `widening-additive-reduction` row：sum loop 是 softmax 第二阶段（`sum += exp(in-mmax)`），非 standalone reduction；排除。
  - `precision-conversion` row：float2half 是输出 epilogue 子步骤，热点主导在 expf（83.21%）而非转换；排除独立命中。
  - `no-vectorization` row：行内互斥「operator semantic shape → 各自更具体 row」——softmax 已被更具体的 normalization row 认领；排除独立认领（保留为 supporting 语义）。
  - `kernel-operation-fusion` row：3-pass 的中间统计量只留寄存器（`fs3` 中 sum、`fs0` 中 mmax），**没有**「中间结果先写回再完整读入」的内存往返；排除。
  - `eliminate-unnecessary-precision-conversions` row：`fcvt.d.s`/`fcvt.s.d` 往返包住 double 运算，但 AType=double 是模板显式参数（`Softmax<..., double, float, half_t, ...>`）——double accumulator 是数值设计合同，非「完全不必要的往返」；排除独立命中，作为 correctness contract 保留在 fix 中。
- **(c) 双 Confidence 推导式**：`route: scalar 3-pass softmax + hardware V(zve64f/zvl128b) + build V(v1p0) + 语义合同（max→exp→sum→normalize）与热点地址分账清晰 → High；impact: 有局部样本份额（expf 83.21%、3-pass ≈99%）但 percent=local period（非 global-period）、函数 workload 贡献未知、`baseline_gap: sampling IP precision` → Medium`。

**Finding 2（independent）：ISA Extension-Specific Instruction Substitution（zfhmin/fmax）**
- **(a) 逐字 evidence 引用**：
  - `2.31 :   18deb14:        fmv.x.w a4,fa5`（normalize+store loop 内软件 float2half 序列起点）
  - `0.43 :   18deb5e:        sh      a5,0(s5)`（float2half 结果写回 half 输出）
  - `9.51 :   18dea50:        beqz    a5,18dea56`（max loop 的 compare-branch，F 扩展 `fmax.s` 可替代）
- **(b) 互斥邻居排除**：
  - `floating-point-semantic-lowering` row：float2half 确为保持 rounding/NaN 语义的位操作序列，但行内互斥原文「纯 Zfa/Zfh native instruction substitution → ISA-substitution row」——hardware 有 zfhmin（`fcvt.h.s` 原生），归 ISA-substitution；排除 semantic-lowering 独立命中。
  - `eliminate-unnecessary-precision-conversions` row：float2half 是 FP32→FP16 格式转换（输出类型合同），非 float32↔float64 往返；排除。
  - primary normalization finding：ISA-substitution 的修复对象（FP16 转换指令选择、max 指令选择）与向量化 normalization 机制可分账——即使 3-pass 向量化，输出仍为 half_t，转换路径独立存在；判为 independent。
  - `no-vectorization` row：ISA-substitution 不解释「未向量化」；排除。
- **(c) 双 Confidence 推导式**：`route: hardware zfh/zfhmin 确认（metadata cpuinfo.isa）+ build 缺 zfh（Tag_RISCV_arch 无 zfh）→ 修正依赖 build-flags 前置 + fmax.s 属 base F（build 有 f2p2）→ Medium；impact: 局部样本份额 float2half 段约 3.8%（fmv.x.w 2.31% + sh 0.43% + 分支/移位若干）、max 分支链 9.51% 中 branch 部分随 supporting 计入 primary → Low`。

**Finding 3（supporting，并入 Finding 1）：ISA Extension-Specific Instruction Substitution（fmax.s 分支）**
- (a) `9.51 : 18dea50: beqz a5,18dea56` + `0.52 : 18dea4e: add a0,a0,s9`（max loop）；supporting because: max reduction 的 scalar compare-branch 链是 normalization 第一阶段的一部分，向量化（`vfredmax`）或 `fmax.s` 修正都消除同一机制；不另计顶层命中。
- (b)(c) 与 primary 共享机制说明，confidences 记 `—`。

### 多命中仲裁小段
- **primary**：RVV Normalization Kernels —— 一条因果链的顶层 root-cause row（未向量化 cross-lane 归一化 + scalar expf 调用）。Layer：L1（vectorization / semantic dispatch）。
- **supporting**：ISA-substitution（fmax.s 分支）—— 解释 primary 同一机制（max reduction 慢）的次级命中，上层向量化修复会让其 signal 消失 → 并入 supporting（符合 arbitration「上层修复若会让下层 signal 消失，下层并入 supporting」）。Layer：L4。
- **independent**：ISA-substitution（zfhmin `fcvt.h.s` 分支）—— evidence 落在 float2half 转换指令段（0x18deb14–0x18deb5e），与 expf 调用点不相交；机制（FP16 输出转换指令选择 vs transcendental 调用）、修复对象（build flags + 指令替换 vs 向量化 exp/reduction）、验证预测均可分离；向量化后输出转换路径仍独立存在 → independent。Layer：L4。
- 入口条件 A：按局部样本份额排序 —— primary ≈99%（3-pass）＞ independent ≈3.8%；同一调用链不得简单相加，此处机制分账清晰，无叠加。

## Phase 4 — Root-cause blueprint / 根因蓝图：Softmax<softmax_fwd, false, double, float, half_t, int, 2>

### Finding 1（primary）：RVV Normalization Kernels（`patterns/rvv_normalization_kernels.md`；Phase 3 Finding 1 通过 gate）

1. **Root cause**：Softmax 是典型 cross-lane normalization（max → exp(x-mmax) → sum → normalize 三阶段），但 hot path 完全未向量化：每个元素一次 scalar libm `expf@plt` 调用（83.21% 局部样本）、max/sum 用 scalar reduction + 条件分支（max-loop 9.51%）。机制句引用（依据 `patterns/rvv_normalization_kernels.md` §Why this is slow）：「归一化的根因是跨 lane reduction 造成依赖链，多阶段完整扫描造成额外流量，或 reciprocal/rsqrt/vector-math 不可达而退回 scalar helper」——本函数同时命中「vector-math 不可达退回 scalar helper」（expf）与「跨 lane reduction 依赖链」（max/sum loop 每轮 `flt.s`→`beqz`、`fadd.d` 串行累加）。
2. **The fix / 修复方式**（与该文件 §The fix §5/§6/§7 一致）：
   - 将 3-pass scalar 循环改写为 RVV 1.0 向量 kernel（build 已含 `v1p0`/`zvl128b`，硬件 zve64f 支持 fp32/fp64 vector 运算），保持三阶段显式语义（§7「Softmax 应至少分为：第一阶段 max reduction → 第二阶段 exp(x-max) 并归约指数和 → 第三阶段 reciprocal/除法缩放」）：
   - 第一阶段：`vle32` + `vfredmax.vs`（按 §5「每个数据块独立执行 reduction，并将当前标量累计值作为 seed」），NaN/Inf/全相等/空输入语义按 reference 保留。
   - 第二阶段：`vle32` → `vfsub.vf`（减 mmax）→ 向量 exp（调用项目真实存在且通过验证的 RVV exp 实现，见 `references/vector-math-conventions.md`：exp 的 availability、特殊值、误差和 mask-safe dataflow；不得凭空假设存在）→ `vfredusum.vs` 归约指数和（§5：`vfredusum` 改变 FP 加法结合顺序；需要有序语义时评估 `vfredosum`；本项目 AType=double 累加若需 bit-exact 或误差合同必须单独验证）。
   - 第三阶段：对保存的指数值做 vector-scalar reciprocal/除法缩放（§7「使用 reciprocal 或除法对保存的指数值进行缩放」，reciprocal estimate 精度需验证）→ 输出转换（见 Finding 2）。
   - **修复前**（scalar，per-element）：
     ```cpp
     // 伪代码：当前 scalar 形态
     float mmax = in[base];
     for (j=1; j<M; ++j) mmax = max(mmax, in[base+j*sa]);
     double sum = 0;
     for (j=0; j<M; ++j) sum += std::exp(in[base+j*sa] - mmax);   // 43.14% 局部样本
     for (j=0; j<M; ++j) out[base+j*sa] = half_t(softmax_fwd::Map<double>(in[base+j*sa]-mmax, sum)); // 40.07% 局部样本
     ```
   - **修复后**（RVV，伪代码，LMUL 按 §2 预算）：
     ```cpp
     // 伪代码：RVV 形态（SEW=32, LMUL 候选 m1/m2/m4，峰值 live set 约束 LMUL*peak<=32）
     vfloat32m1_t v = vle32(in+base, vl);
     float mmax = vfmv_f_s(vfredmax(vseed, v, vl));            // 阶段 1
     v = vfsub_vf(vle32(...), mmax, vl);
     vfloat32m1_t e = rvv_exp(v, vl);                           // 项目已验证 RVV exp
     float sum = vfmv_f_s(vfredusum(vseed, e, vl));            // 阶段 2
     v = vfdiv_vf(e, sum, vl);                                  // 阶段 3（或 vfrec7+refine）
     vse16(out+base, vfncvt_f_f_w(v, vl), vl);                  // 输出（zfhmin 见 Finding 2）
     ```
   - **适用前提**：build 已含 V（无需改 ISA 基线即可编译 intrinsic）；`M`（axis 长度）须大于向量化 crossover，避免小 axis 退化（短 axis 保留 scalar 或 `vl` 截断）。
   - **不可破坏的 correctness contract**：稳定减 max（mmax 语义）；NaN/±Inf/全相等/全 -Inf/+Inf/指数和为零/溢出下溢（§7 单独处理清单）；`count==0`；`vfredusum` 改变 FP 结合顺序（本项目 `MSHADOW` softmax 的 AType=double 累加合同——若要求 bit-exact 输出，须先证明误差合同允许 unordered reduction，否则用有序路径或保留 double 累加）；Softmax 输出和容差；temperature 参数分支（`==1.0` 快路径 vs `/temperature` 慢路径）必须保留（当前 0x18dea5e 的分支）。
   - **限制/风险**：RVV exp 的 ULP 误差 vs libm `expf`；向量化提高 live register 压力（§2：LMUL×peak_live_vectors≤32，mask v0、widening EMUL 计入）；fixed-VL main loop + runtime tail（`references/kernel-conventions.md` §3，UNROLL_FACTOR ∈ {1,2,4,8}）。
   - **预期变化的 Profile signals**：`expf@plt` 调用点（`18dea7c`、`18deafe`）样本大幅下降/消失；`beqz`（`18dea50`）消失；出现 `vle32/vfredmax/vfredusum/v*exp/vse16`；cycles/row 或 IPC 改善。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv…zfh_zfhmin…（含 v, zve64f, zvl128b）；build ISA = rv64…v1p0…zvl128b（含 v，无 zfh）；VLEN=128（vlenb=16）；bound type = compute/transcendental-call 主导（非 memory-bound）。
4. **收益上界**：当前 sampled event（cpu-clock）下的**局部样本份额**：expf 调用点合计 83.21%（`18dea7c` 43.14% + `18deafe` 40.07%）；3-pass normalization 流程合计约 99%。不构成 workload 级 Amdahl 上界（percent=local period、函数 workload 贡献未知 → `baseline_gap: sampling metadata` 对应约束）。入口模式 A，可按局部份额排序。
5. **三维路由判定**：`current source` = compiler-generated scalar（OpenMP clone，无 `.S` provenance）→ intrinsic/编译器向量化路径为修正对象；`implementation existence/reachability` = 无既有 RVV softmax kernel / dispatch slot 证据（无 kernel-selection 信号），需新写 intrinsic kernel 或依赖 autovec（autovec 不可达时 intrinsic 为 carrier）；`function-level policy` = 无独立 `.S` policy 证据（mxnet 无 assembly-default 合同），不走 missing-`.S` 分支。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。
7. **Related PRs**：Related PRs：17 条 URL —— oneDNN [36df719b3bc9](https://github.com/uxlfoundation/oneDNN/commit/36df719b3bc9001d994455c2899e42533b2c29ac)、[6dc3e2d84eec](https://github.com/uxlfoundation/oneDNN/commit/6dc3e2d84eec45eff0834e8985cc663db6fe07ee)、[3b90f9d0650f](https://github.com/uxlfoundation/oneDNN/commit/3b90f9d0650f4087d4a6fbcb3406dc0862f2ae25)、[#4809](https://github.com/uxlfoundation/oneDNN/pull/4809)、[#4622](https://github.com/uxlfoundation/oneDNN/pull/4622)、[#4453](https://github.com/uxlfoundation/oneDNN/pull/4453)、[#4480](https://github.com/uxlfoundation/oneDNN/pull/4480)、[0d8f4a9e702b](https://github.com/uxlfoundation/oneDNN/commit/0d8f4a9e702b70f6188f58a20247ae06dbd41ce5)、[7ad03324c2d1](https://github.com/uxlfoundation/oneDNN/commit/7ad03324c2d17480e40cc20114576584888373ce)、[3c8c37bab64f](https://github.com/uxlfoundation/oneDNN/commit/3c8c37bab64f67bbdebcdb76330db0dcb25b31af)、[c07b7f4e7776](https://github.com/uxlfoundation/oneDNN/commit/c07b7f4e777636cf527f49093c03f8cacf58b3de)、[#4734](https://github.com/uxlfoundation/oneDNN/pull/4734)、[#4491](https://github.com/uxlfoundation/oneDNN/pull/4491)、[25cd4a75095a](https://github.com/uxlfoundation/oneDNN/commit/25cd4a75095a2aebeb2f0140abadd466cd0b4e90)、[b8360ec0a1d5](https://github.com/uxlfoundation/oneDNN/commit/b8360ec0a1d566f039d4fe286349c7988a762c98)；MNN [#4508](https://github.com/alibaba/MNN/pull/4508)、[#4044](https://github.com/alibaba/MNN/pull/4044)、[b7268aa](https://github.com/alibaba/MNN/commit/b7268aa3dab754190ed7d86dc16b9bab02e73d12)。

### Finding 2（independent）：ISA Extension-Specific Instruction Substitution（`patterns/isa_extension_specific_instruction_substitution.md`；Phase 3 Finding 2 通过 gate）

1. **Root cause**：输出转换与 max 比较用了多指令合成序列，而目标硬件具备原生指令。机制句引用（依据 `patterns/isa_extension_specific_instruction_substitution.md` §Why this is slow）：「When the compiler does not exploit the specific extension available on the target CPU, it must fall back to synthesizing the operation from base integer or floating-point instructions」——① 软件 `float2half`（`mshadow::half::half_t::float2half`，约 20 条指令/元素：sign 提取 `and`/`srliw`、`maxZ/minN/maxN/infN/nanN` 四级 branch diamond、rounding `sne/slli/addw`、`srliw 0xd`+`or`，0x18deae6–0x18debbe 与 0x18dec52–0x18decf0 两处复制）可由 zfhmin 单条 `fcvt.h.s` 替代（§8 表格「FP32→FP16」行；zfhmin 提供 `fcvt.h.s`/`fcvt.s.h`）；② max loop 的 `flt.s`+`beqz`+`fmv.s` 由 base-F `fmax.s` 替代（§5「Hardware Float Min/Max (F/D Extensions)」：`fmin.s`/`fmax.s` 消除 branch，RISC-V 语义与 ±0/NaN 规则需按行内表验证）。
2. **The fix / 修复方式**：
   - 前置（L0 类修正）：build ISA 增补 `zfhmin`（`-march=..._zfhmin` 或 `-march=rv64gcv_zfhmin`；hardware 已支持 zfh/zfhmin），使编译器/项目可生成 `fcvt.h.s`。扩展宏保护（`references/kernel-conventions.md` §Extension macro guards）：`#if defined(__riscv_zvfhmin) || defined(__riscv_zfhmin)` 才启用原生路径，否则保留软件 fallback（`__riscv_zfhmin` 宏 gate，与 `kernel-conventions.md` 要求一致——「不要只依赖 `__riscv_v_intrinsic`」）。
   - 修复前（软件 float2half 简化形态，0x18deae6 段）：
     ```cpp
     // Before: 软件 FP32→FP16（sign/exp/mantissa 位操作 + 分支 + rounding）
     uint16_t float2half(float v) { /* ~20 条指令, maxZ/minN/maxN/infN/nanN 分支 */ }
     ```
   - 修复后（zfhmin 原生）：
     ```cpp
     // After: zfhmin 单指令转换（frm=RNE 与 mshadow round-to-nearest 对齐需验证）
     #if defined(__riscv_zfhmin) || defined(__riscv_zvfhmin)
       _Float16 h = (_Float16)f;            // -> fcvt.h.s
     #else
       uint16_t h = float2half(f);          // fallback 保留
     #endif
     ```
   - max 分支：`flt.s`+`beqz`+`fmv.s` → `fmax.s`（F 扩展，build 已有 f2p2，无新依赖）。
   - **适用前提**：硬件 zfh/zfhmin 已确认（metadata cpuinfo.isa）；build flags 修正后 clean rebuild；fallback 路径必须保留（可移植性）。
   - **不可破坏的 correctness contract**：FP16 rounding 模式（mshadow `MSHADOW_HALF_ROUND_TO_NEAREST` vs `fcvt.h.s` 的 frm——默认 RNE 需验证一致）；NaN/Inf/subnormal/±0 语义（0x7c00/0x8000/0x7e00 边界与 pattern §Verification 边界表一致）；fp16 溢出行为（fcvt.h.s 对超范围输入产生 inf，与软件路径 `infN` 分支语义对比）。
   - **限制/风险**：依赖 build flags 修正（否则无法 emit fcvt.h.s）；zfhmin 只含转换指令（不含 half 算术），本项目输出路径恰好只需要转换 → 足够。
   - **预期变化的 Profile signals**：0x18deb14–0x18deb5e（及 0x18dec52–0x18decf0）的位操作/branch 样本消失，出现 `fcvt.h.s`；max loop 的 `beqz`（0x18dea50）被 `fmax.s` 替代后 branch-miss 下降。
3. **Baseline facts 回填**：hardware ISA = rv64imafdcv…zfh_zfhmin…；build ISA = rv64…v1p0（**无 zfh**，为修正前置）；VLEN=128；bound type = compute 主导。
4. **收益上界**：当前 sampled event 下局部样本份额：float2half 转换段约 3.8%（`18deb14` 2.31% + `18deb20` 0.40% + `18deb5e` 0.43% + 其余 <0.5% 指令）；不构成 workload 级上界。入口模式 A 局部份额排序：primary 99% > independent 3.8%。
5. **三维路由判定**：`current source` = compiler-generated（float2half 模板代码 + 编译器指令选择）；`implementation existence/reachability` = 原生 `fcvt.h.s`/`fmax.s` 在目标硬件可达（zfhmin/f 扩展），build flags 修正后指令可达；`function-level policy` = 无独立 policy 约束。
6. **Implementation-shape proof**：不适用（非 missing `.S` 分支）。
7. **Related PRs**：Related PRs：59 条 URL（按 pattern 表去重后逐条）—— Go [3659b8756a2b](https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0)、[a6ecdf29e34d](https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c)、[#59488](https://github.com/golang/go/pull/59488)、[63ab68ddc5f1](https://github.com/golang/go/commit/63ab68ddc5f1307e552cf27ae7a6f0dfda2bb962)、[1951afc9193f](https://github.com/golang/go/commit/1951afc9193f8e197cb7dfaf6afed70ea02404cb)；OpenCV [a00818047ff5](https://github.com/opencv/opencv/commit/a00818047ff5671586c7b296cac6175b250f85d3)、[7e2c8cc9f4c4](https://github.com/opencv/opencv/commit/7e2c8cc9f4c49a3afeee084a2f01f7bac1d9a2dc)、[f0d29cd33c5d](https://github.com/opencv/opencv/commit/f0d29cd33c5d2b8f6c9c5c3177cbb3a359ee6b33)、[#21351](https://github.com/opencv/opencv/pull/21351)；OpenSSL [03ce37e11729](https://github.com/openssl/openssl/commit/03ce37e117)、[ca6286c382a7](https://github.com/openssl/openssl/commit/ca6286c382)、[48b6776678d7](https://github.com/openssl/openssl/commit/48b6776678)、[6136408e6abf](https://github.com/openssl/openssl/commit/6136408e6a)、[e4fd3fc379d7](https://github.com/openssl/openssl/commit/e4fd3fc379)、[80c664db430d](https://github.com/openssl/openssl/commit/80c664db43)、[08c8dd6b8ced](https://github.com/openssl/openssl/commit/08c8dd6b8cede3cdbe5b1866c1a7544e0fe7a378)、[49a3e7adc392](https://github.com/openssl/openssl/commit/49a3e7adc3)、[a41f9135f082](https://github.com/openssl/openssl/commit/a41f9135f0)、[4dbb537bd1ea](https://github.com/openssl/openssl/commit/4dbb537bd1)、[608cadfbdbdb](https://github.com/openssl/openssl/commit/608cadfbdb)、[b1b889d1b3fc](https://github.com/openssl/openssl/commit/b1b889d1b3)、[657d1927c68b](https://github.com/openssl/openssl/commit/657d1927c6)、[611685adc04a](https://github.com/openssl/openssl/commit/611685adc0)、[7ae2bc9df6e0](https://github.com/openssl/openssl/commit/7ae2bc9df6)；Linux Kernel RISC-V [e8620bd7e5e0](https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712)、[5ba15d419fab](https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49)、[cc2294d3f9c9](https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92)、[36e224168721](https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de)、[e11e367e9fe5](https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355)、[75ab93a244a5](https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb)、[c64086849110](https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68)；OpenJDK [6b89954c6534](https://github.com/openjdk/jdk/commit/6b89954c65342bc601633d24075dab4f4b248f4b)、[a7631ccf18e4](https://github.com/openjdk/jdk/commit/a7631ccf18e468d6ecba121865f7fed29cbf2186)、[#22752](https://github.com/openjdk/jdk/pull/22752)、[08d563ba1504](https://github.com/openjdk/jdk/commit/08d563ba15047020fd5f5fea80547e18898bbab2)、[b1a21b563e3a](https://github.com/openjdk/jdk/commit/b1a21b563e3ae13fa5c409a4f0c04686c3f5b34a)、[#22410](https://github.com/openjdk/jdk/pull/22410)、[9f582e56baee](https://github.com/openjdk/jdk/commit/9f582e56baee0e7f5af20da0f395cd935bf5a962)、[#24096](https://github.com/openjdk/jdk/pull/24096)、[1a4bbb0027ae](https://github.com/openjdk/jdk/commit/1a4bbb0027ae9e6df3b668454fa155861d531f72)、[2ed7ad4b5c7d](https://github.com/openjdk/jdk/commit/2ed7ad4b5c7d2344ae6571c186f8a2903770aa57)、[edfe28541a6e](https://github.com/openjdk/jdk/commit/edfe28541a6ed94357f873aa69778c7eba707cbb)、[b891bfa7e67c](https://github.com/openjdk/jdk/commit/b891bfa7e67c21478475642e2bfa2cdc65a3bffe)、[3d3b78203710](https://github.com/openjdk/jdk/commit/3d3b7820371058b40f2e694536c98aa3900abb5f)、[a7a09f69abc6](https://github.com/openjdk/jdk/commit/a7a09f69abc6c4730599d3de9067c2fde75c5172)、[bcc33d5ef3bd](https://github.com/openjdk/jdk/commit/bcc33d5ef3bdbfaee51c45014851c54028da03f1)、[#22386](https://github.com/openjdk/jdk/pull/22386)、[5866b16dbca3](https://github.com/openjdk/jdk/commit/5866b16dbca3f63770c8792d204dabdf49b59839)、[8cb9b479c529](https://github.com/openjdk/jdk/commit/8cb9b479c529c058aee50f83920db650b0c18045)、[#11921](https://github.com/openjdk/jdk/pull/11921)；llama.cpp [#17784](https://github.com/ggml-org/llama.cpp/pull/17784)；V8 [6f100865663f](https://github.com/v8/v8/commit/6f100865663fb99df2628144fc65977e55ab7e68)、[e62c1e307d20](https://github.com/v8/v8/commit/e62c1e307d207e1e56219c172929289a1b474530)、[200b5212ae47](https://github.com/v8/v8/commit/200b5212ae4752f6345dbcc6ecf23f651434bbf8)、[5136fb5200c1](https://github.com/v8/v8/commit/5136fb5200c1f3a33939419fed9de32e8d85bc1e)、[e02d2238f6ea](https://github.com/v8/v8/commit/e02d2238f6ea6f920a6ac31f888b4727bb0b0f80)、[94a3c420e2a4](https://github.com/v8/v8/commit/94a3c420e2a4f76d42e367e7d3b3f85ecd1b0a3f)、[1818e36d54d1](https://github.com/v8/v8/commit/1818e36d54d12540080dd58dbfb924155a920cba)、[5f433dd5024a](https://github.com/v8/v8/commit/5f433dd5024a566fa051317dd0c9c1d9921a9162)、[9bbbde26bb90](https://github.com/v8/v8/commit/9bbbde26bb90bd493afe33777d7b6be0c4f2afb4)、[758956654f28](https://github.com/v8/v8/commit/758956654f28861ba0d5e94f03fff6016b3a2998)、[32c5d22333c1](https://github.com/v8/v8/commit/32c5d22333c14a9d6bb645af3ea4d3fb28b541fe)、[89719bc239f4](https://github.com/v8/v8/commit/89719bc239f48735bbffe83e0803fd504b3c485a)、[223d7fb26b12](https://github.com/v8/v8/commit/223d7fb26b12145bdd335d51da2fa2ed1c58ce3a)、[b2852080c401](https://github.com/v8/v8/commit/b2852080c401cb0980f332918531597967de36c1)、[8b842dbb9f6d](https://github.com/v8/v8/commit/8b842dbb9f6d7ea03990d80dc945ef8c6c6ca144)、[e855c14cab2c](https://github.com/v8/v8/commit/e855c14cab2c6522a2a2d9b6be51e886fd3aae74)；QEMU [3de1fb712a07](https://github.com/qemu/qemu/commit/3de1fb712a072992d72bc99c2b70978132ee44d0)、[6ef584318238](https://github.com/qemu/qemu/commit/6ef5843182382f6a84995590ad91047b0f2bc1fa)；V8（bitmanip 系列）[8033cc56e2af](https://github.com/v8/v8/commit/8033cc56e2afb7b19214aa9a6f776a59fc509d2c)、[a654e27b50fe](https://github.com/v8/v8/commit/a654e27b50feb457a99ad27582dd555316eab587)、[21289aa92c80](https://github.com/v8/v8/commit/21289aa92c80a4161cc5f0d9915a0002a644d8b4)、[19a0f69f4c4b](https://github.com/v8/v8/commit/19a0f69f4c4b1257b567290412acda411568d565)；LLVM [#170824](https://github.com/llvm/llvm-project/pull/170824)、[4c1e1e05cb90](https://github.com/llvm/llvm-project/commit/4c1e1e05cb901a2ed9055e5d6ac6ce60b826a288)、[#92926](https://github.com/llvm/llvm-project/pull/92926)、[#152744](https://github.com/llvm/llvm-project/pull/152744)、[#122698](https://github.com/llvm/llvm-project/pull/122698)、[13e32a8a3c95](https://github.com/llvm/llvm-project/commit/13e32a8a3c95b23af51f081865db1bd259d269a4)、[787eeb8597fa](https://github.com/llvm/llvm-project/commit/787eeb8597fac22decb366a42176b11f52ec1bf0)、[c705b7b04dba](https://github.com/llvm/llvm-project/commit/c705b7b04dba467a67871a1bbb77907d0ed7fc19)。

## Phase 5 — Verification forecast / 验证预测：Softmax<softmax_fwd, false, double, float, half_t, int, 2>

**Finding 1（primary）——修复对象：3-pass scalar softmax → RVV 向量化（max/sum reduction + 向量 exp + 除法缩放）**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`43.14 : 18dea7c: auipc ra,0xfecc0`（sum-loop expf 调用点）、`40.07 : 18deafe: auipc ra,0xfecc0`（map-loop expf 调用点）、`9.51 : 18dea50: beqz a5,18dea56`（max-loop 条件分支）——同一函数重建后重新 annotate：这三行样本应大幅下降或消失；`flt.s`/`fcvt.d.s`/`fadd.d`/`fdiv.d`/`fcvt.s.d` scalar 序列消失。
- 应出现侧（依据 `patterns/rvv_normalization_kernels.md` §Verification）：出现 vector reduction + transform（`vle32`、`vfredmax.vs`、`vfredusum.vs`、`vfsub.vf`、向量 exp、`vfdiv.vf`/reciprocal、`vse16`）；cycles/row 或 throughput 改善；正确性合同：Softmax 覆盖全相等、极端 logits、NaN/Inf、sum≈1、LogSoftmax consistency 与稳定减 max；最后 reduction 的 `vl`/tail/ordered-unordered FP 语义与 reciprocal 误差明确；若融合 passes 证明中间值无外部观察者。
- 验证范围：动态 `M`（axis 长度）覆盖 0、小 M、fixed-VL main-loop 整倍数与全部 tail（1..step-1）；短/中/长 axis benchmark；FP reordering 容差（`vfredusum` unordered vs double 累加 reference）在表述为可接受前说明。
- 升级采样：`perf annotate --stdio -l -s <func> --percent-type=global-period` 补 global share；核对 `precise_ip`。

**Finding 2（independent）——修复对象：软件 float2half → zfhmin `fcvt.h.s`；max compare-branch → `fmax.s`**
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`2.31 : 18deb14: fmv.x.w a4,fa5`、`0.43 : 18deb5e: sh a5,0(s5)`（float2half 序列 0x18deb14–0x18deb5e 及 0x18dec52–0x18decf0）——转换为 `fcvt.h.s` 后消失；`9.51 : 18dea50: beqz`（max-loop branch）在 `fmax.s` 替换后消失。
- 应出现侧（依据 `patterns/isa_extension_specific_instruction_substitution.md` §Verification）：disassembly 确认 `fcvt.h.s`/`fmax.s`；无扩展 flag 编译时 fallback 序列重现；FP16 边界测试（0x7c00 inf、NaN 0x7e00、subnormal、±0、RNE rounding 与 `MSHADOW_HALF_ROUND_TO_NEAREST` 对齐）；±0（−0.0<+0.0）与 NaN 传播（quiet NaN→非 NaN operand）按行内表验证。
- 升级采样：补 `perf stat -e branch-misses` 验证 max-loop branch 消除收益。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`Softmax<softmax_fwd, false, double, float, half_t, int, 2>` |
| 2 | Phase 1 输出要求 | ✅ | 7 行 baseline 全出；`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`（局部份额约束）；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ | Class selection trace 8 项（include: rows-operator-rvv.md, rows-codegen.md；exclude: rows-asm/string-memory/vectorized-tuning/offload/crypto/runtime-os）；Classes scanned: rows-operator-rvv.md, rows-codegen.md；顶层 finding 2（primary normalization + independent ISA-substitution）+ supporting 1（fmax.s）；evidence 锚点 `18dea7c:43.14%`、`18deafe:40.07%`、`18dea50:9.51%`、`18deb14:2.31%`、`18deb5e:0.43%`；排除 9 条；推导式 2 条 |
| 4 | Phase 4 输出要求 | ✅ | 已读 pattern：`patterns/rvv_normalization_kernels.md`（命中 row: RVV Normalization Kernels；引用「归一化的根因是跨 lane reduction…退回 scalar helper」；§5/§6/§7；fix before/after、correctness、风险、Profile signals；Related PRs：17 条 URL）、`patterns/isa_extension_specific_instruction_substitution.md`（命中 row: RISC-V ISA Extension-Specific Instruction Substitution；引用「fall back to synthesizing the operation from base integer or floating‑point instructions」；§5/§8；fix before/after、correctness、风险、Profile signals；Related PRs：59 条 URL） |
| 5 | 路径合规 | ✅ | 入口模式 A（profile-backed）；8 项 trace 可解释扫描集；primary(L1 normalization)+supporting(L4 fmax.s)+independent(L4 zfhmin)；每 leaf 来自通过 gate 的 row；局部份额排序 primary≈99% > independent≈3.8% |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`18dea7c`/`18deafe`/`18dea50`/`18deb14`/`18deb5e`；出现侧：`rvv_normalization_kernels.md §Verification`、`isa_extension_specific_instruction_substitution.md §Verification` |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；无向用户追问；交付止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 |

修正记录：无