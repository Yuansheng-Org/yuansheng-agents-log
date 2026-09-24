**Functions under analysis: [zgemm_kernel_n]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-zgemm_kernel_n-annotate.txt`，141,891 bytes，944 samples，覆盖 hot k-loop body、C-tile epilogue 与全部 tail 路径）
- perf stat（bound/context）：已提供（共享 `14-openblas-benchmark-riscv-cblas_zgemm_512x512.txt`，单线程，IPC=0.984）
- workload/binary/DSO/source context：已提供（`cblas_zgemm.goto`；调用链 `main → cblas_zgemm → zgemm_nn → zgemm_kernel_n`（perf_report self 92.91%）；annotate 内嵌源码行显示 `__riscv_v*` RVV intrinsic 生成代码；test_output 确认 M=N=K=512，OPENBLAS_LOOPS=15，8314.50 MFlops / 1.937s）
- readelf -A（build ISA）：缺失（metadata warnings: "No ELF executable binaries were found for this run"，`binaries: {}`；详见 Phase 1）
- hardware ISA：已提供（metadata `cpuinfo.isa` = `rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`）
- `vlenb`：已提供（metadata `vector.vlenb = 32` → VLEN=256 bits）
- 采样元数据：已提供（event=`cpu-clock`，percent type=`local period`，scope=单函数 annotate，窗口=单次 2.04s 运行共 1016 samples；perf_freq=499Hz）
- Sampling IP precision：缺失（raw metadata 未记录 `precise_ip`/Exact-IP；详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：完整 RVV 1.0（`v`、`zve64d` 等），含 `zicbop`（prefetch hint）、`zicbom`/`zicboz`、`zfa`、`zvbb/zvbc/zvkb` 等（metadata cpuinfo.isa） |
| Build ISA | `baseline_gap: build ISA`（本 run 无 ELF binary，无法 readelf -A）；由 annotate 内容推断执行代码含标准 RVV 1.0 `v*` mnemonic（`vlse64`/`vfmul.vf`/`vfmsac.vf`/`vfadd.vv`/`vsetivli`），build 确实启用了 V；无 `th.v*` |
| Vector flavor | RVV 1.0 `v*` mnemonic（annotate 全量）；无 `th.v*` → 无 flavor mismatch |
| VLEN | 已提供：`vlenb=32` → VLEN=256 bits；`e64,m1` = 4 lanes/vector（全函数硬编码 `vsetivli zero,4,e64,m1,ta,ma`） |
| Bound type | 已提供：IPC=0.984（4.477e9 cycles / 4.407e9 instr），L1_dcache_load_miss_rate=1.345%，branch_miss_rate=1.327%，threads=1 → **issue-bandwidth-bound 计算内核**（k-loop）；C-tile epilogue 为 memory-latency-bound（见 Phase 3 Finding 2） |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=local period（annotate 内函数内局部份额）；同一运行窗口（1016 samples）；函数 workload 贡献已知（perf_report self=92.91%）→ 四项全部成立，可表述 workload 级收益上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（`precise_ip` 未记录；RISC-V cpu-clock 采样存在 skid）→ 单行占比只锚定 basic block / loop interval，不做单指令 latency/cost 归因 |

L0 baseline gate：hardware 有 `v`，build 推断含 `v`（annotate 直接证据），无 mismatch；无 `th.v*`；annotate 已提供，入口条件 A（profile-backed）。Bound-type gate：IPC≈1.0 且 FP 指令密度 48/66 → issue-bound；非 memory-bandwidth-bound（L1 miss 1.345% 低），但 epilogue 为 latency-bound。

## Phase 2 — Scope / 分析边界

函数清单：`zgemm_kernel_n`（唯一，本批次按 rank 001 分析）。hot loop interval：

- **k-loop body** `17b88–17c8c`（每 k-step 66 条指令 = 12 loads + 48 向量 FP + 6 scalar），占函数 samples ≈77%；
- **C-tile epilogue** `17c90–17de0`（16 vlse64 C-loads + 32 alpha-FMA + 16 vsse64 C-stores + 地址计算），占函数 samples ≈23%，但每 i-iteration 只执行 1 次（频率为 k-loop 的 1/512）→ 单次执行成本约为 k-step 的 ~150 倍（memory-latency-bound）。

Trace anchor（最高行）：`8.37 :  17ba0:  vlse64.v        v2,(t6),a4`（A 面板 A1i 跨步 gather，k-loop body）。Sampling IP precision 未确认 → 单行占比只锚定 k-loop body / epilogue 两个 interval，根因陈述保持 interval 级。

## Phase 3 — Pattern scan / 模式扫描：zgemm_kernel_n

### Class selection trace

1. `rows-asm.md` — **exclude**：annotate 源码行显示 `__riscv_vlse64_v_f64m1` 等 intrinsic 调用，当前代码是 compiler-generated（intrinsic 展开），非手写 `.S`；目标 `.S` 缺失问题不成立（RVV kernel 存在且正在执行，self=92.91%）。
2. `rows-operator-rvv.md` — **include**：intrinsic 生成 RVV 代码，热点由 complex-arithmetic 与 matmul microkernel 语义决定；行内明确接纳 already-vectorized 的 interleaved 二字段 strided 访问失效形态。
3. `rows-string-memory.md` — **exclude**：无 copy/fill/sentinel/compare/checksum 语义。
4. `rows-vectorized-tuning.md` — **include**（必选）：完整 annotate 已有 `v*` 且非 `.S`；扫描 LMUL/operand-form/inactive-lane/vector-state/unroll rows。
5. `rows-codegen.md` — **include**：C-tile epilogue 的 cache refill load-stall 证据、register pressure、IV strength reduction、addressing-mode、operation fusion rows。
6. `rows-offload.md` — **exclude**：无矩阵引擎（metadata 无 IME）、无 packed-SIMD/P、无跨 VLEN 可移植层证据；weight-repack 属于批次内独立函数 zgemm_oncopy（本批次另析）。
7. `rows-crypto.md` — **exclude**：非密码学 workload。
8. `rows-runtime-os.md` — **exclude**：用户态 benchmark，无 timer/ISR/CSR 热点。

`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md`（rows-asm.md、rows-string-memory.md、rows-offload.md、rows-crypto.md、rows-runtime-os.md 按 trace 排除）。

### Local performance pattern scan: zgemm_kernel_n

| # | Pattern | Evidence | Route conf. | Impact conf. | Detail file |
|---|---|---|---|---|---|
| 1 | RVV Complex Arithmetic Kernels（primary） | k-loop 每 k-step 16 vfmul + 16 vfmsac/vfmacc + 16 vfadd（48 向量 FP / 66 指令）；tmp→ACC 结构 + 16 条 `vfadd.vv`；A 面板固定二字段跨步访问 `vlse64`（stride 16B）；源行 `vfloat64m1_t tmp0r = __riscv_vfmul_vf_f64m1( A0i, B0i, gvl);` + `ACC0r = __riscv_vfadd( ACC0r, tmp0r, gvl);` | High | Medium | `patterns/rvv_complex_arithmetic_kernels.md` |
| 2 | Cache-Aware Blocking for Tiled Kernels（independent） | C-tile epilogue 17c90–17de0 ≈23% samples（16 行 8KB-stride C-load/store），单次执行 ≈150× k-step → cache-refill load-stall 段主导 | Medium | Medium | `patterns/cache_aware_blocking_for_tiled_kernels.md` |

#### Finding 1（primary）— RVV Complex Arithmetic Kernels

**(a) 逐字 evidence 引用**（k-loop body `17b88–17c8c`）：

- `8.37 :  17ba0:  vlse64.v        v2,(t6),a4`（A 面板 A1i 跨步 gather，stride=`a4=16` 字节 = `sizeof(FLOAT)*2`）
- `3.50 :  17bd4:  fld     fa1,16(a6)`（B 面板 B1r 标量广播加载）
- `3.18 :  17c00:  vfmsac.vf       v14,fa5,v1`（tmp2r −= B3r·A0r）
- `3.18 :  17c24:  vfmsac.vf       v12,ft0,v3`（tmp1r −= B0r·A1r）
- `2.01 :  17c78:  vfadd.vv        v24,v24,v5`、`1.80 :  17c04:  vfadd.vv        v31,v31,v5`、`1.06 :  17c84:  vfadd.vv        v18,v18,v15`、`1.06 :  17c6c:  vfadd.vv        v27,v27,v8`（每 k-step 16 条 `vfadd.vv` 中的 4 条，vfadd 合计 ≈7.95%）
- 源码行（annotate 内嵌）：`181   tmp0r = __riscv_vfmul_vf_f64m1( A0i, B0i, gvl);`、`197   ACC0r = __riscv_vfadd( ACC0r, tmp0r, gvl);`

k-loop 指令普查（`17b88–17c8c` 逐行计数）：12 loads（4× `vlse64` A + 8× `fld` B）+ 48 向量 FP（16 `vfmul` + 16 `vfmsac/vfmacc` + 16 `vfadd`）+ 6 scalar/loop = **66 指令/k-step**，IPC≈1.0 → issue-bound；FMA 密度仅 32/66=48%。A 侧 vlse64 合计 ≈12.2%（含地址生成 ≈15.3%），B 侧 fld 合计 ≈13.3%，vlse64+vfadd 是 k-loop 内最大两簇。

**(b) 互斥邻居排除**：
- 非 `RVV Contiguous Elementwise Arithmetic Kernels`：热点计算是复数乘加数据流（`Ar·Br−Ai·Bi` / `Ar·Bi+Ai·Br`），不是 lane-independent 单元素算术。
- 非 `RVV Strided Memory Access for Layout Transforms`：stride-16 的 `vlse64` 服务于复数算术（同一 traversal 内 r/i 分离载入后立即进入复数 MAC），不是无算术的固定 stride 搬运。
- 非 `RVV Floating-Point Matmul and GEMV Kernels`（matmul row 行内互斥：复数 matmul 的 blocking/packing/shape 主导才归它）：本热点 blocking/packing 健康——oncopy/itcopy 各自独立仅 1.28%/0.49%，kernel 内 packing 不重复；k-loop 主导的是复数 MAC 计算结构本身（48 FP ops/66 instr），归 complex row 认领。matmul row 的 microkernel-scheduling 信号与 complex primary 同区间同机制，按 arbitration 吸收为 supporting 性质（不另立顶层 finding）。
- 非 `No vectorization`：hot main loop 全量 `v*`。

**(c) 双 Confidence 推导式**：
- `route: intrinsic 生成 RVV + 源码复数 MAC 合同 + interleaved 二字段 strided 访问信号 + 成对 accumulator 修复适用 → High`
- `impact: k-loop 77% sample share + VLEN=256 + bound type 已知（IPC 0.984 issue-bound）+ 采样语义四项成立 → 但存在竞争 bottleneck（Finding 2 epilogue memory）→ Medium`

#### Finding 2（independent）— Cache-Aware Blocking for Tiled Kernels

**(a) 逐字 evidence 引用**（C-tile epilogue `17c90–17de0`）：

- `3.18 :  17d06:  vlse64.v        v9,(s5),a4`（C 面板 C3i 跨步 load）
- `2.75 :  17ca4:  add     s10,s1,a2`（C 行地址计算）
- `2.44 :  17cee:  vlse64.v        v3,(s1),a4`（C6i load）
- `1.59 :  17d02:  vlse64.v        v6,(a6),a4`、`1.38 :  17d0e:  vlse64.v        v5,(s3),a4`（C5r/C5i load）
- 该 interval 合计 ≈23% of 944 samples，但每 i-iteration 仅执行一次（每 i-iteration 内 k-loop 执行 511 次）→ 单次执行 ≈150× k-step 成本，属 cache-refill 暴露的 load-stall（C 行 stride = ldc×16B = 8KB，行间跨页，compulsory miss）。

**(b) 互斥邻居排除**：
- 非 matmul-row 计算体主导：epilogue 内 32 条 alpha-FMA 依附于 C-load 延迟（load→FMA→store 链），memory op + 地址段主导（17ca4 2.75%、16 vlse64 + 16 vsse64 占该段大头），非 FMA 吞吐饱和。
- 非 `RVV Layout and Channel Packing Kernels`：无标量重排/打包循环，是 C-tile 消费端 load/store。
- 非 `kernel_selection`：正确 kernel 已被采用（92.91% self），无 fallback 问题。
- 尺寸拐点 counter 未采集（perf stat 无 cache/memory refill 事件）→ 缺可选证据，route 维持 Medium。

**(c) 双 Confidence 推导式**：
- `route: annotate 直接显示 cache-refill 暴露的 load-stall 地址段（C-tile epilogue 23%），tile/复用顺序可确定（8×4 complex tile、C 行 8KB stride、每次复用前被驱逐）→ Medium（缺独立 refill counter/尺寸扫描）`
- `impact: epilogue 23% sample share + VLEN/bound 已知 → 但修复杠杆跨 driver blocking + kernel prefetch 两层，且缺少尺寸拐点 counter → Medium`

### 多候选仲裁小段

Finding 1 与 Finding 2 落在**不相交地址段**（k-loop body vs C-tile epilogue），机制可分账（issue 带宽 vs memory latency），修复对象不同（kernel 累加结构 vs C-tile 访存调度/blocking 参数）→ 均为顶层 `independent` finding。证据机制层次：Finding 1 属 L4 compute micro-structure（complex 算术累加结构），Finding 2 属 L2 data movement（C-tile refill latency）。A 模式按 evidence sample share 排序：Finding 1 上界 ≈0.77×(16/66)≈0.187（vfadd 消除对应指令份额约 0.08 + 指令数效应 0.187），Finding 2 上界 ≈0.23；二者样本不相交，联合上界 ≈0.417 函数级（≈0.39 workload 级），均按 Phase 1 采样语义四项成立表述为局部/工作负载级上界，受"修复不引入新瓶颈"约束。

Supporting evidence（归 Finding 1，不计顶层命中）：A 面板 strided gather（`vlse64` stride 16B，4 条/k-step ≈12.2%，A1i 单行 8.37%）与 B 广播 `fld`（8 条/k-step ≈13.3%）——同一 k-loop 机制内数据移动形态，fix 与验证并入 Finding 1。

### 排除表（逐 row，按 class 组）

- `rows-vectorized-tuning`：`RVV Register-Group Utilization / LMUL Sizing` — 排除：e64,m1=4 lanes 已占满 256-bit 向量；tile 为 8×4 complex = 16 向量 accumulator（r+i 各 8），m2 需 8×8 tile = 64 complex acc = 128 向量寄存器，超 32 上限，m1 已在合法 frontier；k-loop 无 vector spill。`RVV Operand-Form Selection` — 排除：B 用 `vf` 标量广播、A 用向量，是当前 lane 赋值（4 lanes=4 rows）下的正确 operand 形态；hot loop 无 `vmv.v.x`/`vfmv.v.f` temporary（`vfmv.s.f` 仅出现在 cold M&1 tail）。`RVV Inactive-Lane Policy` — 排除：hot 路径全用 `ta,ma`（`vsetivli zero,4,e64,m1,ta,ma`），无 `tu/mu`/mask-clear 问题。`RVV Vector-State Management` — 排除：steady-state k-loop（17b88–17c8c）内零 `vsetvl*`，仅 i-loop 边界重复 `vsetivli zero,4,e64,m1`（每 511 k-steps 一次，可摊薄）；M&1 tail 的每-k `vsetvli a5,a1,e8,mf8`（1828c/1895c/18d40）在本 workload 为 cold 路径（M=512，M&1=0）。`Register-Budgeted RVV Loop Unrolling` — 排除：16 条独立 ACC 链已有足够 ILP；unroll 不削减每 k-step 的 48 FP 指令，削减指令数是 FMA 折叠（Finding 1）而非 unroll。
- `rows-codegen`：`Register Pressure and Save/Restore Optimization` — 排除：i-loop 入口 9 条 stack store（17a9c–17abc）与 9 条 reload（17dec–17e1c）每 i-iteration 一次，摊到 511 k-steps 后 ≈0.00% samples。`Loop Induction Variable Strength Reduction` — 排除：k-loop 已用指针递增 + end-pointer 比较（`addi t5,t5,128` / `addi a6,a6,64` / `bne a0,a6`）。`Kernel Operation Fusion` — 排除：kernel 无独立全量后处理 pass（alpha 已融合进 C-tile epilogue）。`Load/Store Addressing-Mode Fusion` — 排除：k-loop 每步 3 条 A 地址 add（17b88/17b90/17b98）+ 1 条 B 指针 addi，属结构开销（占 ≈4.6%），非独立修复对象。`Resource-Aware Instruction Scheduling` — 排除（作为独立 row）：无目标核 latency/resource 模型证据；load 调度问题并入 Finding 1 的 load pre-issue 修复。
- `rows-operator-rvv` 其它 rows：complex/matmul 之外无 elementwise/normalization/reduction/layout 等合同。
- `rows-asm`（整组排除）：非 `.S` provenance；crypto 专项 memory-traffic row 因非 crypto、非 `.S` 排除，但其 load pre-issue / 常量驻留机制作为本地机制参考并入 Finding 1 的 fix（引 `riscv-assembly-kernel-performance-optimization.md` §1–§3）。

## Phase 4 — Root-cause blueprint / 根因蓝图：zgemm_kernel_n

### Finding 1（primary）— RVV Complex Arithmetic Kernels

命中 row：`rows-operator-rvv.md` → RVV Complex Arithmetic Kernels（Phase 3 Finding 1，route High / impact Medium）。

**1. Root cause**：k-loop 的复数 MAC 采用"tmp 寄存器 + 16 条 `vfadd.vv` 分步累加"结构，而不是 pattern 的成对 accumulator 直接 FMA 累加——pattern 独有内容：「一个复数乘加包含四个实数乘法与两个带符号加法」，其 fix 直接给出 `acc_r = rvv_fmacc(acc_r, ar, xr); acc_r = rvv_fnmsac(acc_r, ai, xi); acc_i = rvv_fmacc(acc_i, ar, xi); acc_i = rvv_fmacc(acc_i, ai, xr)`（依据 `patterns/rvv_complex_arithmetic_kernels.md` §The fix）。当前每 k-step 48 向量 FP（16 vfmul + 16 vfmsac/vfmacc + 16 vfadd）+ 12 loads + 6 scalar = 66 指令，IPC≈1.0 issue-bound，FMA 密度仅 48%：vfadd 16 条（≈8% 直接份额，指令数效应 ≈16/66=24% 的 k-loop 时间）与 16 个 tmp 向量（占寄存器、限制调度）是主要浪费。A 面板 interleaved 二字段 `vlse64`（stride 16B，4 条/k-step，≈12.2%）符合 pattern 的「interleaved complex 布局显示固定二字段 segmented/strided 访问」信号。

**2. The fix / 修复方式**（与 pattern §The fix 一致；VLEN-agnostic 结构，实施由 Craft 在 kernel/riscv64 侧完成）：

Before（每 k-step 当前结构）：
```c
// tmp 分步累加（当前）：16 vfmul + 16 vfmsac/vfmacc + 16 vfadd = 48 向量 FP
vfloat64m1_t tmp0r = __riscv_vfmul_vf_f64m1(A0i, B0i, gvl);      // tmp = Ai*Bi
tmp0r = __riscv_vfmsac_vf_f64m1(tmp0r, B0r, A0r, gvl);           // tmp -= Ar*Br
ACC0r = __riscv_vfadd_vf64m1(ACC0r, tmp0r, gvl);                 // ACC += tmp
```
After（成对 accumulator 直接折叠累加，32 条 signed-FMA，0 tmp、0 vfadd）：
```c
// 保持 VFMACC_RR(=vfmsac) 与 VFMACC_RI(=vfnmacc) 的既有符号约定，折叠进 ACC 目标：
ACC0r = __riscv_vfmacc_vf_f64m1(ACC0r, B0r, A0r, gvl);  // ACC_r += Ar*Br
ACC0r = __riscv_vfmsac_vf_f64m1(ACC0r, B0i, A0i, gvl);  // ACC_r -= Ai*Bi
ACC0i = __riscv_vfmacc_vf_f64m1(ACC0i, B0i, A0r, gvl);  // ACC_i += Ar*Bi
ACC0i = __riscv_vfmacc_vf_f64m1(ACC0i, B0r, A0i, gvl);  // ACC_i += Ai*Br
```
（实际符号按 kernel 的 S0..S3/共轭约定逐项核对，vfnmsac/vfnmacc 方向以现有宏定义为准；16 个 ACC 各 2 条 FMA，共 32 条。）

- **适用前提**：kernel 为 intrinsic 生成代码，32 个向量寄存器预算充足——折叠后 live set = 16 ACC + 4 A 向量 + 8 B 标量 ≈ 20 向量寄存器（< 32，无需 spill；依据 `references/kernel-conventions.md` §2 `LMUL × peak_live_vectors ≤ 32`）；VLEN≥256 已由 `vsetivli zero,4,e64,m1` 假设。
- **correctness contract**：必须保持 (1) 复数乘法/共轭符号约定（`VFMACC_RR`/`VFMACC_RI` 的 fmsac/vfnmacc 方向、S0..S3 常量）；(2) 浮点累加顺序变更——当前为"两乘积先入 tmp 再并入 ACC"，折叠后各乘积直接并入 ACC，rounding 不同；需按 OpenBLAS 数值误差合同（GEMM checksum/相对误差测试）验证；(3) B 广播、packing 合同不变；(4) 各 tail 路径（M&4/M&2/M&1、N&2/N&1）同步改写。
- **限制/风险**：每条 ACC 每 k-step 出现 2 条串行 FMA（依赖链 2×FMA latency），但 16 条独立 ACC 链提供 ILP；OoO 核（X100，metadata `out-of-order`）受益于指令数削减，需实测确认无回退；编译器若无法在保持依赖下调度 32 条 FMA，可显式按 A0/A1×B0..B3 分簇书写。
- **修复后预期 Profile signals**：k-loop 指令数 66→50（-24%），`vfadd.vv` 簇（17c04/17c08/17c18/17c54/17c58/17c5c/17c60/17c64/17c68/17c6c/17c70/17c74/17c78/17c7c/17c80/17c84）从 annotate 消失；k-loop interval 样本份额下降 ≈18.7%（函数级上界）；IPC 维持或上升；GFLOPS 从 8.3 向 issue-bound 新平衡点移动（FP-pipe 视角上限 ≈1.34×）。

配套（同 pattern fix 的数据移动维度）：在 `zgemm_oncopy`/`zgemm_itcopy`（本批次 rank 003/005）packing 时把 A/B 面板改为 re/im 分离（SoA）布局，kernel 侧 A 从 4 条 `vlse64`（stride 16B）降为 2 条 unit-stride `vle64`，并按 `riscv-assembly-kernel-performance-optimization.md` §2.0 将 A load 提前发射（load pre-issue）、与 FMA 链交错，隐藏 L2 延迟（A 面板每 i-iteration 流式 16KB，为 L1-missing 源，A1i 单行 8.37% 即 load-use 暴露）。

**3. Baseline facts 回填**：hardware ISA = RVV 1.0 全量（含 zicbop/zvbb/zvbc/zfa，metadata）；build ISA = `baseline_gap: build ISA`（annotate 证实执行代码为 RVV 1.0 `v*`）；VLEN = 256 bits（vlenb=32）；bound type = issue-bound（IPC 0.984），epilogue latency-bound；k-loop sample share ≈0.77（函数局部），函数 self=0.9291（workload）。

**4. 收益上界**：当前 sampled event（cpu-clock, local period, 同窗口）下 k-loop 局部份额 0.77；vfadd 消除的直接指令份额 ≈0.08，指令数效应上界 ≈0.77×(16/66)≈0.187 函数级（≈0.174 workload 级）；A 去交织/pre-issue 并入同一上界（A 侧份额 ≈0.12–0.15，不重复加总）。采样语义四项成立，可表述 workload 级上界；受"无新瓶颈"约束。

**5. 三维路由判定**：
- `current source`：compiler-generated（`__riscv_v*` intrinsic 展开，annotate 源码行证实），非 `.S`。
- `implementation existence/reachability`：RVV kernel 即执行实现（92.91% self），无 fallback/dispatch 问题。
- `function-level policy`：无独立 `.S` policy 证据；intrinsic kernel 是 OpenBLAS RISC-V zgemm 的现行载体；不进入 missing-`.S` 分支。

**6. Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。

**7. Related PRs**：`Related PRs：3 条 URL` — https://github.com/OpenMathLib/OpenBLAS/commit/d3bf5a5401e623e107a23fb70151c7102cbd14c7 、https://github.com/OpenMathLib/OpenBLAS/commit/18d7afe69daa196902cd68b63cc381aaafc9d26e 、https://github.com/OpenMathLib/OpenBLAS/commit/63cf4d01668f8f6c73a05039bc36785ba78b0940

### Finding 2（independent）— Cache-Aware Blocking for Tiled Kernels

命中 row：`rows-codegen.md` → Cache-Aware Blocking for Tiled Kernels（Phase 3 Finding 2，route Medium / impact Medium）。

**1. Root cause**：C-tile epilogue（每 i-iteration 一次）的 16 条 strided `vlse64` C-load + 16 条 `vsse64` C-store 行间距 8KB（ldc×16B），每次访问为 compulsory/L2-L3 以上 refill；pattern 独有内容：「分块的目的不是让 tile 越大越好，而是把下一次复用前仍需存活的数据限制在目标 cache 的可用容量内……算术强度高的内核也会退化为 refill/latency 瓶颈」（依据 `patterns/cache_aware_blocking_for_tiled_kernels.md` §Why this is slow）。epilogue 占函数 23% samples 而执行频率仅 k-loop 的 1/512 → 单次执行 ≈150× k-step 成本，C-tile 访存延迟（load→alpha-FMA→store 链串行化于 load latency）主导。

**2. The fix / 修复方式**（与 pattern §The fix 一致，分两层）：
- kernel 层（直接可改）：i-loop 入口或 k-loop 尾部对 8 个 C 行（8KB stride）发起 `zicbop` 软件预取（`prefetch.w` 提示指令，目标 ISA 已含 `zicbop`，无架构副作用，不需 runtime gate；机制同 `riscv-assembly-kernel-performance-optimization.md`「另一根轴：memory-bound stride 循环用软件预取」）；确保 16 条 C-load 在 alpha-FMA 链前全部发射以最大化 MLP。
- driver 层（zgemm_nn blocking 参数）：按实测 cache budget/并发份额验证 GEMM_*_N 等 blocking 常量（pattern fix 的 `cache_budget → choose_legal_tile` 结构），确认 C-tile 复用距离与 C 行跨页行为；本 workload（单 K-block，C 每元素每 rep 仅 touch 一次）C-traffic 为 compulsory，主要杠杆是延迟隐藏而非流量削减。
- **correctness contract**：C 读写顺序、alpha/beta 语义、alias 合同不变；prefetch 为 hint，无架构副作用。
- **风险**：prefetch 过量可能占用 load 带宽；blocking 改动需小尺寸 fallback。
- **预期 Profile signals**：epilogue interval（17c90–17de0）样本份额从 0.23 显著下降；`17d06`/`17cee` 等 C-load 行 stall 减少；refill counter（若可采）改善。

**3. Baseline facts 回填**：同 Finding 1（RVV 1.0、VLEN=256、issue-bound + epilogue latency-bound）。

**4. 收益上界**：epilogue 局部份额 ≈0.23（函数级，≈0.21 workload 级）；与 Finding 1 不相交地址段，联合上界 ≈0.417 函数级（≈0.39 workload 级）。

**5. 三维路由判定**：`current source` = intrinsic 生成代码；`implementation existence/reachability` = 采用中；`function-level policy` = 无 `.S` policy。

**6. Implementation-shape proof**：不适用。

**7. Related PRs**：`Related PRs：2 条 URL` — https://github.com/OpenMathLib/OpenBLAS/commit/269e1cd505b9bf531a76ffeb0cc7dc753eacb346 、https://github.com/OpenMathLib/OpenBLAS/commit/7c1839899e81829b096c62e73804d6859a0beed1

## Phase 5 — Verification forecast / 验证预测：zgemm_kernel_n

**Finding 1（primary，修复对象：k-loop 累加结构 + A 面板加载形态）**：
- 应消失/缩小：k-loop body 内 16 条 `vfadd.vv`（锚点：`17c04: vfadd.vv v31,v31,v5`、`17c78: vfadd.vv v24,v24,v5`、`17c6c: vfadd.vv v27,v27,v8`、`17c84: vfadd.vv v18,v18,v15` 等 16 条）从 annotate 消失；`17ba0: vlse64.v v2,(t6),a4`（8.37%）随 A 去交织/unit-stride 化份额显著下降；k-loop body（17b88–17c8c）样本份额自 ≈0.77 下降。
- 应出现（依据 `patterns/rvv_complex_arithmetic_kernels.md` §Verification）：real/imag 在同一个 vector traversal 内以成对 accumulator 完成（无 tmp+vfadd 分步）；重复 traversal、水平归约不增加；unit-stride/strided/二字段 segmented 与 register-reorder 形态按目标 VLEN/LMUL 合法性与无 spill 比较；conjugated/symmetric/transpose 与不同 stride/layout 对照 scalar reference；覆盖 NaN/Inf/subnormal/±0 并声明 FP 重排容差；非零虚部数据暴露共轭符号错误。
- 性能判据：重新 annotate 后 k-loop 指令数 66→50、GFLOPS 8.3 → 预期 +15~35%（issue-bound 上界 +24%），A/B 基准短/中/长 K 无退化；OoO 核验证吞吐与 issue 占用。

**Finding 2（independent，修复对象：C-tile 预取 + blocking 验证）**：
- 应消失/缩小：epilogue interval `17c90–17de0` 样本份额自 ≈0.23 下降；`17d06: vlse64.v v9,(s5),a4`（3.18%）、`17ca4: add s10,s1,a2`（2.75%）、`17cee: vlse64.v v3,(s1),a4`（2.44%）行份额下降。
- 应出现（依据 `patterns/cache_aware_blocking_for_tiled_kernels.md` §Verification）：固定频率/affinity/线程数扫描尺寸从 cache-resident 到超出 cache，原性能断崖或 packing/refill 占比缩小；annotate 重跑显示 refill 相关地址段 share 下降；单线程/共享 cache 多线程/未知拓扑/检测失败 fallback 分别验证；无 packed-buffer 溢出或小尺寸回退。

**支撑项（并入 Finding 1 验证）**：B 广播 `fld` 簇（`17bd4` 3.50% 等 8 条）随 issue-slot 释放而相对份额下降（不单独验证）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现（载荷：`1/1 组；zgemm_kernel_n`） | ✅（1/1 组；zgemm_kernel_n） |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline；gap 标签 = `baseline_gap: build ISA` + `baseline_gap: sampling IP precision`；含 Sampling IP precision 行） | ✅（7 行；gap: build ISA, sampling IP precision；含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md`；顶层 finding 2；锚点 = `8.37 : 17ba0 vlse64.v v2,(t6),a4`、`3.18 : 17d06 vlse64.v v9,(s5),a4`、`2.01 : 17c78 vfadd.vv v24,v24,v5`、`1.80 : 17c04 vfadd.vv v31,v31,v5`；supporting 1（A/B 加载形态）；排除条数 10+；推导式 2 组） | ✅（8 项 trace；2 顶层 finding；supporting 1；排除 10+；推导式 2） |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern = `patterns/rvv_complex_arithmetic_kernels.md` + `patterns/cache_aware_blocking_for_tiled_kernels.md` + `patterns/riscv-assembly-kernel-performance-optimization.md`（机制引用 §1–§3/§2.0）+ `references/kernel-conventions.md`（§2 预算）；命中 row 2；引用短语首词 = 成对 accumulator / 四个实数乘法 / 分块的目的；The fix 含 before/after、correctness、风险、Profile 信号；Related PRs：3+2 条 URL） | ✅（pattern 文件 2+1；The fix 完整；Related PRs 3+2） |
| 5 | 路径合规（载荷：模式 A profile-backed；8 项 trace 可解释；2 顶层 finding 均来自通过 gate 的 row；按动态份额排序 0.187 / 0.23；无 `th.v*` 停扫） | ✅（A 模式；Finding1 0.187、Finding2 0.23 排序） |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `17c04`/`17c78`/`17ba0`/`17d06`/`17ca4`/`17cee`；出现侧 `rvv_complex_arithmetic_kernels.md` §Verification + `cache_aware_blocking_for_tiled_kernels.md` §Verification） | ✅（双侧锚定逐 finding） |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付止于证据+蓝图+The fix+验证预测 | ✅ |

修正记录：无