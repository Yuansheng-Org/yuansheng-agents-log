**Functions under analysis: [zgemm_kernel_r]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-zgemm_kernel_r-annotate.txt`，140,341 bytes / 2,794 行；63 samples，event=`cpu-clock`，percent: local period；hot loop body 完整覆盖）
- perf stat（可选 bound/context）：已提供（`14-openblas-benchmark-riscv-zher2k_512x512.txt`；IPC=0.8875，L1-dcache-load-miss-rate=1.253%，branch-miss-rate=0.790%，threads=1，gflops=8.53）
- workload/binary/DSO/source context：部分提供（metadata.json：openblas @ `70c2410f17b6`，testcase=zher2k_512x512；`binaries: {}` → `source_context_gap`：无 ELF/DSO 映射，packing/driver 层调用链证据缺失）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：缺失（详见 Phase 1）
- hardware ISA（`/proc/cpuinfo` / hwprobe）：已提供（metadata cpuinfo：`rv64imafdcvh_...`，含 `v`、`zba/zbb/zbc/zbs`、`zfa`、`zvbb/zvbc`、`zve64d` 等；无矩阵引擎扩展）
- `vlenb`：已提供（metadata vector：`vlen_bits=256` / `vlenb=32`）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock`；percent type=local period；单次运行窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（无 `precise_ip` / Exact-IP / PMU skid 信息 → `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`（含 `v`，RVV 1.0；含 zba/zbb/zbc/zbs/zfa/zvbb/zvbc；无矩阵引擎扩展） |
| Build ISA | `baseline_gap: build ISA`（run 未收集 ELF → 无法核对 `Tag_RISCV_arch`；但 annotate 全为 RVV 1.0 `v*` mnemonic → kernel 显然以含 `v` 的 `-march` 构建，无 hardware-v/build-no-v 证据） |
| Vector flavor | RVV 1.0 `v*`（`vlse64.v`/`vfmacc.vf`/`vfmsac.vf`/`vfadd.vv`/`vsetivli`/`vlseg2e64.v`/`vlseg8e64.v`），无 `th.v*` → 无 flavor mismatch |
| VLEN | vlenb=32 → VLEN=256 bits；`e64,m1` → VLMAX=4 lanes；kernel 固定 `vsetivli zero,4,e64,m1,ta,ma`（gvl=4 = 满寄存器） |
| Bound type | IPC=0.8875（490,349,095 cycles / 435,191,502 instructions）；L1-dcache-load-miss-rate=1.253%（低）；branch-miss-rate=0.790%（低）→ **非 cache-miss-bound、非 branch-bound，为 issue/latency-bound（FP 与 LSU 指令吞吐主导）**；核频≈490M/0.226s≈2.17GHz → 实测≈3.9 flops/cycle ≈ 单条 256-bit FMA 管线峰值（8 flops/cycle）的一半 |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=local period（函数内局部份额）；同一运行窗口；函数级 workload 贡献未知 → 收益上界只能表述为「当前 sampled event 下的局部样本份额」，不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip/Exact-IP/skid 信息）→ 单行占比只锚定所属 loop interval，不做单指令 latency 归因 |

L0 baseline gate：hardware 有 `v` 且 kernel 指令为 `v*` → 无 hardware/build mismatch、无 `th.v*` flavor mismatch；annotate 完整覆盖 → 入口条件 A（profile-backed）。Bound-type gate：issue-bound 成立 → compute 侧 fix 的 performance-impact confidence 不被 memory-bound 抑制；`baseline_gap: sampling IP precision` 封顶 instruction-level attribution。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`zgemm_kernel_r`（1 个）——OpenBLAS zher2k 路径实际执行的复数 GEMM RVV microkernel（annotate 源码行 `__riscv_vlse64_v_f64m1(...)`、`vfloat64m1_t` 证明为 intrinsic C 生成代码，非手写 `.S`）。

hot loop interval 与 trace anchor（63 samples，cpu-clock，local period）：
- **main k-loop：0x72de–0x73e2**（每次 i-j tile 执行 K−1=511 次；含 4×`vlse64.v` A 载入、8×`fld` B 载入、48 条 vector FP、~6 条 int/branch；≈48/63 samples ≈ 76.2%）
  - 最高占比行：`6.35 : 72f6: vlse64.v v2,(t6),a4`（A1i 的 stride-16 gather 载入，4 samples）
  - 次高：`4.76 : 7336: vfmacc.vf v16,fa1,v1`、`4.76 : 7346: vfmul.vf v9,v3,fa5`、`4.76 : 7376/737a/737e/738a: vfmsac/vfmacc`（各 3 samples）
- **C update phase：0x73e6–0x7536**（每 i-iteration 执行一次；16×`vlse64.v` C 载入 + 16×`vsse64.v` C 写回 + 32 条 alpha-scaling 复数 FMA + ~20 条地址计算；≈15/63 samples ≈ 23.8%）
  - 最高占比行：`4.76 : 73fa: add s10,s1,a2`（C0i 载入地址链，3 samples）；C gather `3.17 : 7444/745c/7468: vlse64.v`（各 2 samples）
- annotate 覆盖完整：全部 63 个 sample 落在函数体内并覆盖两个 hot interval；tail 路径（M&4/M&2/M&1 于 0x75ba–0x8186、N&2/N&1 于 0x818a–0x850e）零 sample → 对 M=N=512 不执行（`512&7==0`、`512&3==0`、`512&1==0`）。
- Sampling IP precision 未知 → 所有根因归因收敛到 interval-level mechanism；单行占比（如 72f6 的 6.35%）仅作 trace anchor，不承担单指令 latency 结论。

## Phase 3 — Pattern scan / 模式扫描：zgemm_kernel_r

**Class selection trace（8 项）：**

1. `rows-asm.md` — exclude：当前代码来源为 intrinsic C（annotate 源码行 `vfloat64m1_t A0r = __riscv_vlse64_v_f64m1(...)`），无 `.S`/DWARF/object-mapping provenance；亦无 policy-backed missing `.S` 四证。
2. `rows-operator-rvv.md` — include：intrinsic 生成 RVV kernel，热点由复数算术 / FP matmul 算子语义决定；interleaved complex 固定二字段 strided 访问信号。
3. `rows-string-memory.md` — exclude：非 string/memory/compare/copy 函数。
4. `rows-vectorized-tuning.md` — include（第一级必选）：annotate 已全 `v*` 且当前代码来源不是 `.S`；需逐行评估 LMUL/register-group/operand-form/inactive-lane/vector-state/unroll 各 row。
5. `rows-codegen.md` — include：compiler-generated 指令形态；需评估 kernel-selection、cache-aware blocking、register pressure、addressing/induction、code-layout 等跨切 row。
6. `rows-offload.md` — include：目标为 SpacemiT X100 + GEMM workload → 需评估 IME 矩阵引擎卸载与 weight-repack 层（按 build-flag + 硬件检测把关）。
7. `rows-crypto.md` — exclude：无任何 crypto 原语信号。
8. `rows-runtime-os.md` — exclude：用户态 BLAS benchmark，无 timer/CSR/ISR/特权域热点。

`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md, rows-offload.md`

**Local performance pattern scan: `zgemm_kernel_r`**

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Complex Arithmetic Kernels（primary） | interleaved complex 布局（A/C 均为 [re,im] 对）用 4×`vlse64.v`(stride 16) 逐字段 de-interleave；k-loop 每轮 16 vfmul + 16 vfmacc/vfmsac + 16 vfadd 的复数 MAC 结构；C-phase 16+16 strided gather/scatter | High | Medium | `patterns/rvv_complex_arithmetic_kernels.md` |
| RVV Floating-Point Matmul and GEMV Kernels（supporting） | k-loop 每轮 16 条独立 `vfadd.vv` 的 accumulator-merge 结构 + microkernel scheduling/累加器形态 | — | — | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |

**（a）逐字 evidence 引用（Finding 1，primary，k-loop 0x72de–0x73e2）：**
- `6.35 :   72f6:   vlse64.v        v2,(t6),a4`（最高单行，A1i stride-16 gather；对应源码行 `178  A1i = __riscv_vlse64_v_f64m1( &A[ai+1*gvl*2+1], sizeof(FLOAT)*2, gvl );`）
- `1.59 :   72f2:   vlse64.v        v1,(t5),a4`（A0r gather；`175  A0r = __riscv_vlse64_v_f64m1( &A[ai+0*gvl*2], ...)`）
- `3.17 :   735a:   vfadd.vv        v31,v31,v5`（ACC 合并；k-loop 内共 16 条同型 `vfadd.vv`）
- `4.76 :   7336:   vfmacc.vf       v16,fa1,v1` 与 `4.76 :   737e:   vfmsac.vf       v11,ft0,v4`（复数 MAC 的带符号 FMA 对）
- k-loop 结构事实：每 k 迭代 = 4×vlse64（16 次元素访存）+ 8×fld + 48 条 vector FP（16 vfmul + 16 vfmacc/vfmsac + 16 vfadd）+ ~6 int/branch ≈ 66 条指令，产出 8 个复数 ACC × 4 lanes = 128 flops → **2.7 flops/vector-FP-instr**；直接配对 FMA 累加（`acc_r=fmacc(ar,br); acc_r=fnmsac(ai,bi); acc_i=fmacc(ar,bi); acc_i=fmacc(ai,br)`）只需 32 条 FMA/迭代。

**（b）互斥邻居排除（Finding 1）：**
- 非实数 elementwise row：主循环指令为复数符号结构（`72b4: vfmsac.vf v0,fa1,v2` = `A0r·B0i − A0i·B0r` 的虚部形态），不是同 lane 实数 add/sub/mul → 排除 `rvv_contiguous_elementwise_arithmetic_kernels.md`。
- 非纯 strided-layout row：A 的 `vlse64.v` 与复数 FMA 同处一个 k 迭代且立即被消费（`72f6` gather 结果被 `7326: vfmul.vf v12,v2,fa1` 使用），有完整复数算术 → 该 row 行内互斥「只有固定 stride 搬运、没有复数算术 → strided-layout row」不成立 → 排除。
- 非 widening reduction row：k-loop 内无跨 lane 加性归约；`vfredosum.vs` 仅出现在 M&1 tail（`7a5a/7a6e/7a78`）与 N&1 tail（`8110/811c/8120`），M=N=512 不执行 → 排除。
- 非 indexed gather row：A 的 `vlse64.v` 是固定 stride（16 bytes），不是数据相关索引 → 排除 gather row。

**（a）逐字 evidence 引用（Finding 2，independent，C-phase 0x73e6–0x7536）：**
- `4.76 :   73fa:   add     s10,s1,a2`（C 载入地址链；对应源码行 `235  vfloat64m1_t C0i = __riscv_vlse64_v_f64m1( &C[ci*2+1], sizeof(FLOAT)*2, gvl );`，sample 落在其地址生成上）
- `3.17 :   7444:   vlse64.v        v3,(s1),a4`、`3.17 :   7468:   vlse64.v        v1,(t6),a4`（C 行 strided 载入）
- `1.59 :   747c:   vsse64.v        v16,(s11),a4`（C 写回；`293  __riscv_vsse64_v_f64m1( &C[ci*2+0], sizeof(FLOAT)*2, C0r, gvl);`）
- C-phase 结构事实：每 i-iteration 执行 ~90 条（16 vlse64 + 16 vsse64 + 32 条 alpha 复数 FMA + ~20 条依赖 add/slli 链），约占该函数指令 0.3%，却占 **15/63 ≈ 23.8% samples** → 每条指令平均成本约为 k-loop 的 ~90 倍；C 的 32 条 strided 访存全部是 stride=16 bytes（每 8B 取一个 double，实/虚部分离）。

**（b）互斥邻居排除（Finding 2）：**
- 非纯 strided-layout row：C-phase 含 32 条 alpha-scaling 复数 FMA（`7402: vfmacc.vf v16,fa0,v31` + `741a: vfnmsac.vf v16,fa4,v0`），有复数算术 → 排除。
- 非 kernel-selection / cache-blocking row：C-phase 位于正确执行中的 RVV kernel 内部（63 samples 全部在其体内），属 epilogue 数据搬运，与 driver 层 blocking/kernel 选择无关；无 tile-residency/尺寸拐点证据 → 排除 `rows-codegen.md` 中该两行。
- 非 register-pressure row：C-phase 与 k-loop 均无 vector spill/reload/stack slot（k-loop 完整使用 v0–v31 但无溢出访存）→ 排除。
- 非 vector-state / inactive-lane row：C-phase 与 k-loop 均无 `vsetvli`（vtype 稳定于 `e64,m1,ta,ma`）；`vsetvli/vsetivli` 切换（`e8,mf8`↔`e64,m1`）只存在于不执行的 tail 路径 → 排除 `rows-vectorized-tuning.md` 对应两行。

**（c）双 Confidence 推导式：**
- Finding 1：`route: interleaved-complex source 语义 + k-loop 指令逐字证据（72f6/7336/735a）+ 4 项互斥排除 → High；impact: 局部 sample share 48/63 可用、VLEN/bound 已知，但缺 sampling IP precision 与 workload 级贡献 → Medium`
- Finding 2：`route: C 的 interleaved 布局 + 32 条 strided 访存 + 地址链逐字证据（73fa/7444/747c）直接可见 → High；impact: 15/63 局部份额 + VLEN/bound 已知，缺 IP precision/workload 级 → Medium`
- Supporting（fp-matmul）：`route/impact 均 —（supporting 不另计）`

**多命中仲裁小段：**
- **Finding 1 = primary**（k-loop 0x72de–0x73e2；机制 = interleaved-complex 布局的 strided de-interleave + tmp/vfadd 累加结构导致的 FP/LSU 指令通胀；L1 semantic 类别 + L4 compute 细节）。
- **fp-matmul = supporting**（同一地址段、同一机制的 accumulator-chain/microkernel-scheduling 维度：每 k 16 条独立 `vfadd.vv` 与 16 条 `vfmul.vf` 是 packing/累加器结构的直接后果；已通过自身 row gate：sample 主导在 FP load/FMA/accumulator dependency）。
- **Finding 2 = independent**（不同地址段 0x73e6–0x7536、不同执行频率（每 i-iteration 一次 vs 每 k 一次）、可分离修复对象（C 段 gather/scatter + 地址生成 vs k-loop 的 A/B 载入与累加结构）；机制上共享同一 pattern leaf 的 interleaved-complex 访问形态，按地址与修复对象分账；L2 data movement）。
- 收益上界（入口条件 A，按局部样本份额排序）：Finding 1 ≈ 48/63（76.2%）→ Finding 2 ≈ 15/63（23.8%）。仅「当前 sampled event 下的局部样本份额」，非 workload 级（采样语义四条中 percent-type=local、workload 贡献未知不成立）。
- 上层修复的因果消除：若 A/B/C 段访存改为 segmented/unit-stride，C-phase 的链式地址 add 与 k-loop 的 gather 地址 add 作为症状随之消失；vector-state 切换本来就不存在，故无 L3 归属。

**未命中 rows 汇总（逐 row 评估过的行）：** rows-vectorized-tuning 的 operand-form（k-loop 直接用 `fld` scalar 作 `.vf` 操作数，无 `vmv.v.x/vfmv.v.f` 临时 → 不命中）、register-budgeted-unrolling（单条 back-edge `73e2: bne` 零 sample，无 recurrence/ILP 证据 → 不命中）、inactive-lane（全 `ta,ma`、vl=4=VLMAX → 不命中）、vector-state（k-loop 零 vsetvli → 不命中）、autovec-control（无 compiler 插入冗余 vsetvli/spill 证据 → 不命中）、maximal-LMUL（非 byte 路径 → 不命中）；rows-codegen 的 kernel-selection（RVV kernel 正在执行，无 fallback/dispatch 误路由证据；`source_context_gap`：driver 层是否应有独立 her2k kernel 无法从本 annotate 判定 → 不命中）、cache-aware-blocking（无 tile-residency/尺寸拐点证据 → 不命中）、register-pressure（热区间零 spill → 不命中）、addressing-mode-fusion / induction-variable（C-phase 的 add 链是 strided 访问的地址生成症状，非独立可折叠对象 → 不命中）、其余 22 行无对应 signal → 不命中；rows-offload 的 weight-repack（无 pack 层证据且 gather 未主导计算体 → 不命中）、IME matrix-engine（cpuinfo 无矩阵引擎 ISA 扩展、无 build-flag/API 证据 → gate 不成立，不命中）、portable-RVV（单目标部署，无跨 VLEN 证据 → 不命中）、parallel-GEMM（threads=1 → 不命中）、packed-SIMD（ISA 无 P/DSP → 不命中）。

## Phase 4 — Root-cause blueprint / 根因蓝图：zgemm_kernel_r

### Finding 1（primary）— k-loop：interleaved-complex de-interleave + tmp/累加器拆分导致 FP/LSU 指令通胀

1. **Root cause**：`zgemm_kernel_r` 是 OpenBLAS zher2k 路径的复数 GEMM RVV microkernel（intrinsic C）。A、C 均为 interleaved complex（`[re,im]` 对），kernel 用 4×`vlse64.v`（stride=16B）逐字段 gather 分离实/虚部、用 8 条 `fld` 逐标量取 B，复数 MAC 采用「tmp=vfmul → tmp=vfmacc/vfmsac → ACC=vfadd」三段式结构。每 k 迭代 66 条指令只产出 128 flops（48 条 vector FP = 2.7 flops/instr；理论配对 FMA 结构只需 32 条/迭代，且 16 条 `vfadd.vv` 为纯合并开销）。依据 `patterns/rvv_complex_arithmetic_kernels.md`：§Why this is slow「interleaved 布局若逐标量解包会浪费向量带宽」；§When to apply「interleaved complex 布局显示固定二字段 segmented/strided 访问」；§The fix「real/imag 在同一 strip-mined traversal 中共同存活，使用独立 accumulator 隐藏依赖」。硬编码 `vsetivli zero,4,e64,m1`（gvl=4）使 VLEN=256 下每条 FP 指令只覆盖 4 lanes。
2. **The fix**（与该 pattern §The fix 一致，纠正对象 = A/B 载入形态与累加结构）：
   - A 侧：用 segmented `vlseg2e64.v` 一次载入复数对（或把 A 面板按实/虚部分离打包成两条 unit-stride 流），替换 4×stride-16 gather（每 k 元素访存 16→8 次）；
   - B 侧：B 的 8 个连续 double 用 2×`vle64.v` unit-stride 载入（或按 target 验证保持 scalar），消除 8 条独立 `fld` 的 LSU 占用；
   - 累加结构：改为对 ACC 直接配对 FMA，删除 tmp 寄存器与 16 条 `vfadd.vv`：
     ```c
     // Before（每复数 MAC 6 条）：tmp = vfmul(Ai,Bi); tmp = vfmacc(Br,Ar,tmp); ACC = vfadd(ACC,tmp); ...
     // After（每复数 MAC 4 条，直接入 ACC）：
     acc_r = __riscv_vfmacc_vf_f64m1(acc_r, Br, Ar, gvl);   // +Ar·Br
     acc_r = __riscv_vfnmsac_vf_f64m1(acc_r, Bi, Ai, gvl);  // −Ai·Bi
     acc_i = __riscv_vfmacc_vf_f64m1(acc_i, Bi, Ar, gvl);   // +Ar·Bi
     acc_i = __riscv_vfmacc_vf_f64m1(acc_i, Br, Ai, gvl);   // +Ai·Br
     ```
   - LMUL 候选：在 `LMUL × peak_live_vectors ≤ 32`（`references/kernel-conventions.md` §2）预算内评估 `e64m2`（8 lanes、4 个复数 ACC 的 tile 重构，FP 指令/迭代可再减半），枚举 `m1/m2 × unroll=1/2` 且验证无 spill；更大 LMUL 不预设为必然更快。
   - 适用前提与 correctness contract：必须保持 zher2k 的共轭符号规则（Hermitian 路径点名被共轭的 operand 及 `±Ai·Br` 符号）、NaN/Inf/±0/subnormal 语义、复数 MAC 的 FMA contraction 行为；FP 累加顺序变化需声明容差；`gvl` 建议由 `vlenb` 推导而非硬编码 4（跨 VLEN 可移植性）。
   - 限制/风险：segmented load 在部分微架构上拆分代价高（需 target A/B）；m2 重构若 live-range 预算算错会 spill；packing 层改动需同步 driver（超出本 annotate 可验证范围）。
   - 预期变化的 Profile signals：k-loop `vfadd.vv` 由 16 条/迭代降至 0；`vlse64.v` gather 消失或减半；k-loop vector FP 指令 48→32（或 m2 下 →24）；`72f6` 类行占比消失。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh+zba/zbb/zbc/zbs/zfa/zvbb/zvbc`；build ISA=`baseline_gap: build ISA`；VLEN=256 bits；bound type=issue/latency-bound（IPC 0.8875）。
4. **收益上界**：k-loop 局部样本份额 ≈ 48/63（76.2%），表述为「当前 sampled event（cpu-clock）下的局部样本份额」；因 percent-type=local 且 workload 级贡献未知，不得称为 workload 级 Amdahl 上界。
5. **三维路由判定**：current source = intrinsic C 生成的 RVV kernel（annotate 源码行直接证据）；implementation existence/reachability = kernel 正在执行且被 workload 采用（63 samples 全部位于其体内）；function-level policy = 无要求独立 `.S` 的官方 policy 证据（OpenBLAS RISC-V kernel 政策未在本 run 中提供）。
6. Related PRs：`patterns/rvv_complex_arithmetic_kernels.md` 3 条：`https://github.com/OpenMathLib/OpenBLAS/commit/d3bf5a5401e623e107a23fb70151c7102cbd14c7`、`https://github.com/OpenMathLib/OpenBLAS/commit/18d7afe69daa196902cd68b63cc381aaafc9d26e`、`https://github.com/OpenMathLib/OpenBLAS/commit/63cf4d01668f8f6c73a05039bc36785ba78b0940`（supporting 组见下）。

**Supporting evidence（fp-matmul，仅随 Finding 1 输出）**：`3.17 : 735a: vfadd.vv v31,v31,v5` + k-loop 每轮 16 条 `vfadd.vv` / 16 条 `vfmul.vf`；supporting because: 同一 k-loop 机制中的 accumulator-chain 与 microkernel scheduling 维度（packing/累加器结构决定 vfadd 与 vfmul 数量），无独立修复对象。Related PRs：`patterns/rvv_floating_point_matmul_and_gemv_kernels.md` 37 条（OpenBLAS 7：0a967797a156、0acb60aab3c0、1cc377ef61d4、2d82d144e279、809e1cba8f1f、376d3a138faa、2ae019161a85；oneDNN 22：d6f82a2d0d0d、3bac96b、8b48a77、8c52fac、b73fc31、bd984d0、d2a44b9、d6107dd、fe04323、#4410、#4414、#4545、#4620、#4770、#4824、#4840、#4850、#4945、#5157、#5294、#5403、#5405；llama.cpp 6：#17318、#17448、#17314、#17161、#18199、#20627；MNN 1：#4426；vLLM 1：#44324）。

### Finding 2（independent）— C update phase：C 段 strided gather/scatter + 串行地址链

1. **Root cause**：C update phase（0x73e6–0x7536，每 i-iteration 一次）对行主序 interleaved complex C 执行 16×`vlse64.v`（stride 16B 逐字段取实/虚部）+ 16×`vsse64.v` 写回 + ~20 条依赖 add/slli 地址链（`73ee ld → 73fa add → …`）。该段仅占函数指令 ~0.3%，却占 **15/63 ≈ 23.8% samples**（每条指令平均成本约为 k-loop 的 ~90 倍）——stride-16 的 gather/scatter 每条分解为 4 次元素访存且地址逐行独立，缓存行利用减半，store 无法合并；8 条输出行的基址每次重复用链式 add 重建。依据 `patterns/rvv_complex_arithmetic_kernels.md` §When to apply「interleaved complex 布局显示固定二字段 segmented/strided 访问」、§Why「interleaved 布局若逐标量解包会浪费向量带宽」。
2. **The fix**（纠正对象 = C 段访存形态与地址生成）：
   - C 载入/写回改用 segmented `vlseg2e64.v` / `vsseg2e64.v` 一次搬运 `[re,im]` 对（32 条 strided 访存 → 16 条 segment 访存；C 的 8 行 × 4 列复数对在行内连续，segment 形态语义合法）：
     ```c
     // Before：Cr = vlse64(&C[ci*2+0], 16, gvl); Ci = vlse64(&C[ci*2+1], 16, gvl);
     // After：vlseg2e64.v (vCr, vCi), (&C[ci*2]), gvl;   // 一次载入 (re,im) 对
     //        vsseg2e64.v (&C[ci*2]), (vCr, vCi), gvl;   // 一次写回 (re,im) 对
     ```
   - 行基址改为归纳更新（每 i-iteration 基址 `+= 8*ldc`，预计算 8 行指针数组），删除 ~20 条链式 add/slli；
   - 若 target 上 segment 指令昂贵，评估「连续对载入 + 寄存器内 de-interleave」替代路径（register reorder）。
   - 适用前提与 correctness contract：`ldc` 行 stride、`[re,im]` 顺序、alpha/beta 语义与 C 的 Hermitian 三角写回合同不变；必须覆盖 ldc≠N 的 padding 行；位模式保持（NaN payload/signed zero）。
   - 限制/风险：segment 指令在部分实现上与 strided 同价（需 A/B 验证）；寄存器内 de-interleave 增加 shuffle 指令。
   - 预期变化的 Profile signals：C-phase `vlse64/vsse64` 计数 32→16；`73fa` 类地址 add 消失；C-phase 样本占比（当前 23.8%）显著下降。
3. **Baseline facts 回填**：同 Finding 1（HW ISA / build gap / VLEN=256 / issue-bound）。
4. **收益上界**：C-phase 局部样本份额 ≈ 15/63（23.8%），同上限定为局部份额。
5. **三维路由判定**：同 Finding 1（intrinsic C 生成、kernel 在执行、无 `.S` policy 证据）。
6. Related PRs：`patterns/rvv_complex_arithmetic_kernels.md` 3 条（同 Finding 1 的 pattern 分组，URL 去重后相同）。

## Phase 5 — Verification forecast / 验证预测：zgemm_kernel_r

- **Finding 1（primary）**：修复对象 = k-loop 的 A/B 载入形态 + 累加结构（LMUL/segment 方案按候选）。
  - 应消失/缩小侧（锚定 Phase 3 引用行）：`6.35 : 72f6: vlse64.v v2,(t6),a4`、`1.59 : 72f2: vlse64.v v1,(t5),a4` 与 8 条 `fld` 行占比下降或消失；`3.17 : 735a: vfadd.vv v31,v31,v5` 及 k-loop 内其余 `vfadd.vv` 序列（735e/736e/73aa–73da）消失。
  - 应出现侧（依据 `patterns/rvv_complex_arithmetic_kernels.md` §Verification「disassembly/annotate 应显示 real/imag 在同一 vector loop 中完成，重复 traversal、scalar extract 或水平归约次数下降；比较 unit-stride、strided、二字段 segmented 与 register reorder，检查 LMUL/NFIELD 合法性与 spill」）：k-loop 出现 `vlseg2e64.v`（或两条 unit-stride `vle64.v`）与直接入 ACC 的 `vfmacc/vfnmsac` 对；per-k vector FP 指令数 48→32（m2 候选下 →24）；`vsetvli/vsetivli` 的 LMUL 显示候选值；无新增 spill/reload。
  - 数值正确性：对照 OpenBLAS 标量/参考实现覆盖普通、共轭（Hermitian）路径（构造非零虚部数据暴露共轭符号错误）、transpose、M/N 非 8/4 倍数 tail、K 边界；声明 FP 累加重排容差。
- **Finding 2（independent）**：修复对象 = C-phase 的 segment 访存 + 行基址归纳。
  - 应消失/缩小侧（锚定 Phase 3 引用行）：`4.76 : 73fa: add s10,s1,a2`、`3.17 : 7444: vlse64.v v3,(s1),a4`、`3.17 : 7468: vlse64.v v1,(t6),a4`、`1.59 : 747c: vsse64.v v16,(s11),a4` 占比下降或消失。
  - 应出现侧：C-phase 出现 `vsseg2e64.v`/`vlseg2e64.v`（或寄存器 de-interleave 路径）；C-phase 总样本占比从 ~24% 显著下降；stride=16 访存指令数 32→16。
  - 分别验证后再组合验证（单假设归因）；入口条件 A 下按收益上界顺序（Finding 1 → Finding 2）逐项验证。
- **补充采集（升级证据）**：`perf record -e cycles --call-graph` 同 testcase 重采以确认 workload 级贡献与 `precise_ip`/Exact-IP；对修正后 kernel 重跑 `perf annotate --stdio -l -s zgemm_kernel_r --percent-type=global-period` 核对上述锚点。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：N 组 Phase 3–5 全部出现 | ✅ `1/1 组；zgemm_kernel_r` |
| 2 | Phase 1 输出要求满足 | ✅ 7 行 baseline 表齐（Hardware ISA/Build ISA/Vector flavor/VLEN/Bound type/Sampling semantics/Sampling IP precision）；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`；`Sampling IP precision` 行已单列 |
| 3 | Phase 3 输出要求满足 | ✅ 8 项 Class selection trace（include：rows-operator-rvv/rows-vectorized-tuning/rows-codegen/rows-offload；exclude：rows-asm/rows-string-memory/rows-crypto/rows-runtime-os）；`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md, rows-offload.md`；顶层 finding 2 个（primary=Complex Arithmetic；independent=C-phase），supporting 1 个（fp-matmul）；evidence 锚点：`6.35 : 72f6: vlse64.v v2,(t6),a4`、`3.17 : 735a: vfadd.vv v31,v31,v5`、`4.76 : 73fa: add s10,s1,a2`、`1.59 : 747c: vsse64.v v16,(s11),a4`；排除 4+4 条；推导式 2 组 |
| 4 | Phase 4 输出要求满足 | ✅ 已读 pattern：`rvv_complex_arithmetic_kernels.md`（命中 rows-operator-rvv 复数 row；引用「interleaved 布局若逐标量解包会浪费向量带宽」「real/imag 在同一 strip-mined traversal 中共同存活」；The fix 含 before/after、correctness、risk、Profile signals 锚点）、`rvv_floating_point_matmul_and_gemv_kernels.md`（supporting row；accumulator-chain 锚点 `3.17 : 735a`）；Related PRs：complex 3 条、fp-matmul 37 条 URL |
| 5 | 路径合规 | ✅ 模式 A（profile_backed）+ 8 项 class 扫描集 + primary/supporting/independent 仲裁 + L1/L4（Finding 1）与 L2（Finding 2）归属 + 收益上界按动态份额排序（48/63 → 15/63）；无 `th.v*` 停扫 |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧：`6.35 : 72f6`、`3.17 : 735a`、`4.76 : 73fa`、`3.17 : 7444`、`3.17 : 7468`、`1.59 : 747c`；出现侧：`rvv_complex_arithmetic_kernels.md` §Verification |
| 7 | 契约边界合规 | ✅ 无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 The fix 与验证预测；无向用户追问 |

修正记录：无