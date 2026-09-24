Functions under analysis: [cgemm_kernel_n]（1 个）→ 本输出含 1 组 Phase 3–5

# RISC-V 性能诊断：OpenBLAS `cgemm_kernel_n`（cblas_cgemm_512x512，SpacemiT X100 / VLEN=256 / RVV 1.0）

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`001-cgemm_kernel_n-annotate.txt`，472 samples，event=cpu-clock，percent=local period，覆盖 0x179be–0x1a394 全部地址，含 hot loop body）
- perf stat（可选 bound/context）：已提供（`14-openblas-benchmark-riscv-cblas_cgemm_512x512.txt`，IPC 1.026、L1_dcache_load_miss_rate 1.806%、branch_miss_rate 0.255%）
- workload/binary/DSO/source context：已部分提供（annotate 内嵌源码行 `vfloat32m1_t A0r = __riscv_vlse32_v_f32m1(...)` 等，证明为 `__riscv_v*` intrinsic 编写的 C 代码，GCC 编译；无 DWARF 二进制文件）
- readelf -A（build ISA）：缺失（详见 Phase 1）
- hardware ISA（/proc/cpuinfo / hwprobe）：已提供（metadata snapshot：`rv64imafdcvh_zicbom_..._zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_...`）
- `vlenb`：已提供（vlenb=32 → VLEN=256 bits，metadata vector snapshot）
- 采样元数据（event / percent type / scope / 窗口）：已提供（`cpu-clock`，`local period`，单函数采样窗口 472 samples；函数级 workload 贡献未知——批次清单中本函数 rank 001，但全局样本份额未给出）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`（含 `v`、`zve64d`、`zvbb/zvbc`、`zvfh`；无 `th.v*`） |
| Build ISA | `baseline_gap: build ISA`——metadata `binaries: {}`，运行未产出 ELF，无法读取 `Tag_RISCV_arch`；可选命令：对承载热点地址的 ELF 运行 `readelf -A <object>`。annotate 中 `v*` mnemonics 表明该 object 实际以含 `v` 的 `-march` 构建 |
| Vector flavor | annotate 全部为标准 RVV 1.0 `v*` mnemonic（`vlse32.v`/`vfmul.vf`/`vfmacc.vf`/`vfadd.vv`），与硬件 RVV 1.0 一致，无 `th.v*`，无 flavor mismatch |
| VLEN | vlenb=32 → VLEN=256 bits；e32,m1 → 8 elements/register group（annotate `vsetivli zero,8,e32,m1,ta,ma` 证实） |
| Bound type | IPC≈1.03、L1 dcache miss rate≈1.81%、branch miss rate≈0.26% → **compute/issue-bound（非 memory-bound、非 branch-bound）**；L1 命中良好，热点以指令发射/执行资源为主 |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=**local period**（非 global-period）；单次运行同一窗口；函数级 workload 贡献未知（仅知 rank 001）→ 收益上界只能表述为「当前 sampled event 下的局部样本份额」，**禁止 Amdahl 上界**；`baseline_gap: sampling metadata`（缺 global-period 与全局份额） |
| Sampling IP precision | `precise_ip`/Exact-IP 能力未知 → `baseline_gap: sampling IP precision`；单行高占比只锚定 basic block / loop interval，单指令 latency 归因需区间聚合 + def-use 交叉证据 |

L0 baseline gate：
- hardware 有 `v`，annotate 显示 `v*` → build 实际含 `v`，无 hardware/build `v` mismatch；
- 无 `th.v*` → flavor gate 不触发；
- Bound-type gate：compute/issue-bound，本地 RVV/数据布局修复是合理第一杠杆，不因 memory-bound 下调 impact。

## Phase 2 — Scope / 分析边界
- 函数清单（与承诺声明一致）：`cgemm_kernel_n`（1 个）。
- 本函数为 OpenBLAS RISC-V CGEMM N 内核（`CNAME(BLASLONG M, BLASLONG N, BLASLONG K, FLOAT alphar, FLOAT alphai, FLOAT* A, FLOAT* B, FLOAT* C, BLASLONG ldc)`），8×8 复数 tile，`__riscv_v*` intrinsic 编写。
- hot loop 锚点：**main M×8 k-loop = 0x17c56–0x17d6a**（占本函数约 86% 样本，约 406/472）。最高占比行：
  - `23.09 :  17c5e:  vlse32.v        v1,(t2),a4`（A0i 跨步装载；所属 interval：k-loop 装载段 0x17c56–0x17c5e）
  - `2.75 :  17c62:  flw     ft2,4(a2)`、`2.33 :  17c66:  flw     ft3,0(a2)`（B 标量装载）
  - `3.81 :  17ca2:  vfmul.vf        v13,v2,ft0`、`3.39 :  17ca6:  vfmul.vf        v12,v1,fa1`（tmp 乘法）
  - `3.81 :  17cfe:  vfmacc.vf       v13,ft1,v1`、`2.97 :  17d0e:  vfmacc.vf       v9,fa1,v1`（tmp 累加）
  - `2.12 :  17d46:  vfadd.vv        v21,v21,v10`、`2.12 :  17d5e:  vfadd.vv        v15,v15,v4`（ACC 累加）
- 采样语义：`baseline_gap: sampling IP precision` → 23.09% 单行只锚定「A 跨步装载对（17c5a/17c5e）」所在 interval；interval 级根因由 def-use 交叉证据支持（两条 vlse32 的结果 v1/v2 供给后续全部 16 条 vfmul.vf，见 0x17c5e→0x17c72 等数据流）。
- 本 run M=N=512（M%8==0、N%8==0、N&4/N&2/N&1==0、M&4/M&2/M&1==0），因此 M/N 尾块（0x181c8+、0x183ec+、0x18616+、0x187f8+）与 N&4/N&2 段在本 workload 不执行，其样本≈0，不参与根因。

## Phase 3 — Pattern scan / 模式扫描：cgemm_kernel_n

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — 当前代码来源是 `__riscv_v*` intrinsic 编写的 C 代码（annotate 内嵌源码行 `vfloat32m1_t A0r = __riscv_vlse32_v_f32m1( &A[ai+0*gvl*2], sizeof(FLOAT)*2, gvl );` 为直接 provenance），非手写 `.S`；无 policy/existence 四证支持 missing-`.S` 分支。
2. `rows-operator-rvv.md` — **include** — intrinsic/template 生成代码，热点由算子语义决定：复数算术（interleaved r/i 数据流）、FP matmul microkernel、复数跨步数据搬运；需判定已向量化失效形态。
3. `rows-string-memory.md` — **exclude** — 无 copy/fill/sentinel/compare/checksum 语义。
4. `rows-vectorized-tuning.md` — **include** — 完整 annotate 已有 `v*` 且非 `.S`，修正对象候选为 RVV 配置/寄存器组/展开/operand 形态/vector-state。
5. `rows-codegen.md` — **include** — compiler 生成指令流：16 ACC+16 tmp 的寄存器编排（`vmv1r.v` 拷贝、prologue 栈保存）、load/store 形态、cache-aware blocking 层（packed A/B 消费方）。
6. `rows-offload.md` — **include** — 权重/操作数重排层：A/B 由 `cgemm_oncopy`/`cgemm_otcopy`（本批次独立热点 003/004）打包，其布局决策直接决定内核侧访存形态；portable-RVV 候选（`vsetivli zero,8` 硬编码）。
7. `rows-crypto.md` — **exclude** — 无密码学原语。
8. `rows-runtime-os.md` — **exclude** — 非 kernel/RTOS/timer/CSR 上下文。

Classes scanned: `rows-operator-rvv.md`、`rows-vectorized-tuning.md`、`rows-codegen.md`、`rows-offload.md`（全文件逐行评估）；`rows-asm.md`、`rows-string-memory.md`、`rows-crypto.md`、`rows-runtime-os.md` 按 class 一级判据整组排除（理由见上）。

### Local performance pattern scan: `cgemm_kernel_n`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Weight Repacking for Vectorized GEMM（independent，本函数主因之一） | k-loop 内 A 以 `vlse32`（stride=8B）成对 gather：`23.09 : 17c5e: vlse32.v v1,(t2),a4`；packed A 为 interleaved 复数 (r,i) 布局，source 行 `__riscv_vlse32_v_f32m1( &A[ai+0*gvl*2+1], sizeof(FLOAT)*2, gvl )` 证明 stride=2 floats | High | Medium（见推导） | `patterns/weight_repacking_for_vectorized_risc_v_gemm.md` |
| RVV Floating-Point Matmul and GEMV Kernels（independent，本函数主因之二） | k-loop 每轮 48 条向量 FP 指令（16 vfmul + 16 vfmacc/vfmsac + 16 vfadd）实现 64 复数 MAC；`3.81 : 17ca2: vfmul.vf v13,v2,ft0`、`3.81 : 17cfe: vfmacc.vf v13,ft1,v1`、`2.12 : 17d46: vfadd.vv v21,v21,v10`；无 Int8/requantization 合同 | High | Medium（见推导） | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |
| RVV Complex Arithmetic Kernels（supporting，为上述两 finding 的复数 r/i 合同提供直接累加结构） | source 语义确认 interleaved complex：A0r/A0i 两条 vlse32、ACC0r/ACC0i 成对累加、epilogue `vfmacc.vf`+`vfnmsac.vf`（alphar/alphai） | — | — | `patterns/rvv_complex_arithmetic_kernels.md` |

### Finding 1（primary-by-single-line，机制独立）：A 操作数跨步 gather（Weight Repacking）

**(a) 逐字 evidence 引用**（k-loop 装载段 0x17c56–0x17c5e）：
```
23.09 :  17c5e:  vlse32.v        v1,(t2),a4      ← A0i（虚部）跨步装载，stride=8B
 0.00 :  17c5a:  vlse32.v        v2,(a1),a4      ← A0r（实部）跨步装载，stride=8B
```
source 行原文：`vfloat32m1_t A0r = __riscv_vlse32_v_f32m1( &A[ai+0*gvl*2], sizeof(FLOAT)*2, gvl );`、`vfloat32m1_t A0i = __riscv_vlse32_v_f32m1( &A[ai+0*gvl*2+1], sizeof(FLOAT)*2, gvl );`——packed A 为 interleaved (r,i) 复数对，每 k 步两条 8 元素 gather。perf stat 佐证：L1 dcache miss 仅 1.81%，说明 23.09% 并非 cache miss，而是 **stride 8B gather 的 LSU 元素级代价/延迟**。同 interval 内 B 的 16 条标量 `flw`（0x17c62–0x17cee）合计约 16.3%（2.75/2.33/1.48/1.06/0.85/0.85/0.42/0.42/3.81/2.33），同属 load path。

**(b) 互斥邻居排除**：
- strided-layout row（`rvv_strided_layout_transform_kernels.md`）——行内判据要求 **scalar loop** 的固定 stride 访存主导；本函数访存已被向量化为 `vlse32`（gather），且所在合同是 GEMM microkernel 而非 layout transform，按 row 语义归 Weight-Repacking/FP-matmul 路由，不归 strided-layout；
- layout/channel-packing row——行内判据的 packing 算术（shift/mask/segment 重排）在本函数内不存在；打包发生在 `cgemm_oncopy`（独立函数）；
- indexed-gather row（`rvv_gather_indexed_memory_access.md`）——要求 data-dependent/LUT 索引；此处 stride 是编译期常量 2 floats，地址规律，非数据相关；
- register-group/LMUL row——本 interval 无 vector spill/reload、无 `vsetvl` churn、迭代/分支开销低（back-edge 1.48%），LMUL=1 与 8 行 tile 匹配，不归 register-group。

**(c) 双 Confidence 推导式**：
- `route: intrinsic C 源码 + source 行证实 interleaved complex A + annotate 直接显示 vlse32 跨步 gather + 互斥排除成立 → High`；
- `impact: 有 local sample share（17c5e=23.09%，load path 合计≈39.4%）+ VLEN 已知（256）+ bound type 已知（compute/issue-bound）→ Medium；缺 build ISA（baseline_gap: build ISA）且采样非 global-period → 不得 High`。

### Finding 2（independent，共主导）：k-loop FP 脚手架 48 ops/轮（FP Matmul microkernel 结构）

**(a) 逐字 evidence 引用**（k-loop 计算段 0x17c72–0x17d62）：
```
 3.81 :  17ca2:  vfmul.vf        v13,v2,ft0      ← tmp2i = A0r × B2i
 3.39 :  17ca6:  vfmul.vf        v12,v1,fa1      ← tmp3r = A0i × B3i
 3.81 :  17cfe:  vfmacc.vf       v13,ft1,v1      ← tmp2i += B2r × A0i
 2.97 :  17d0e:  vfmacc.vf       v9,fa1,v1       ← tmp0i(4) += B4r × A0i
 2.12 :  17d46:  vfadd.vv        v21,v21,v10     ← ACC4r += tmp0r(4)
 2.12 :  17d5e:  vfadd.vv        v15,v15,v4      ← ACC7r += tmp3r(7)
```
结构事实：每 k 步 = 16 `vfmul.vf`（tmp=A×B_i）+ 16 `vfmacc.vf`/`vfmsac.vf`（tmp+=A×B_r）+ 16 `vfadd.vv`（ACC+=tmp）= **48 条向量 FP 指令 / 64 复数 MAC**（64 输出复数 = 8 列 × 8 行，每复数 2 个 ACC）。FP path 合计样本 ≈45%（vfmul≈19.3% + macc/msac≈20.3% + vfadd≈5.5%）。IPC≈1.03 表明发射/执行资源接近饱和，指令数削减直接转化为收益。

**(b) 互斥邻居排除**：
- operand-form row——main loop 已直接用 `.vf` 形态（标量 FP 寄存器作 broadcast 源，如 `vfmacc.vf v3,ft3,v1`），未先 `vmv.v.x`/`vfmv.v.f` 物化 temporary；M&1 尾块中的 `vfmv.s.f v22,ft2`（0x18660）位于不执行的冷路径；
- register-group/unrolling row——无 spill、无 vsetvl churn、back-edge 仅 1.48%，LMUL 已达 8 行 tile 的匹配值；「48 ops」是模板结构问题而非 LMUL 选型问题；
- elementwise/activation/normalization/reduction 等语义 row——source 与 annotate 均为复数 matmul microkernel 合同（nested M/N/K、r/i 成对 ACC、epilogue alpha 缩放），按 complex-arith row 行内互斥「复数 matmul 的 blocking/packing/shape 主导 → floating-point-matmul row」路由；
- 量化 matmul row——无 Int8/zero-point/requantization 合同（FP32 全流程）。

**(c) 双 Confidence 推导式**：
- `route: source 行显示 tmp 物化式复数 MAC 模板（VFMACC_RI/VFMACC_RR 宏 + vfadd 归并）+ annotate 直接显示 48 ops/轮结构 + complex-arith row 显式路由到 FP-matmul → High`；
- `impact: FP path local share ≈45%（脚手架 vfmul+vfadd 可删 ≈24.8%）+ VLEN/bound 已知 → Medium；缺 build ISA（baseline_gap: build ISA）→ 不得 High`。

### 多命中仲裁小段（依据 triggers/arbitration.md）
- 两个顶层 finding 属**不同机制、可分离分账**：Finding 1 是 load/LSU 路径（A gather + B 标量装载 ≈39.4%，其中 gather 直接可删 ≈23.1%）；Finding 2 是 vector-FP 发射路径（≈45%，脚手架可删 ≈24.8%）。修复对象不同（`cgemm_oncopy` 打包布局 vs kernel 内层 FMA 结构）、验证预测不同（vle32 替换 vlse32 vs vfmul/vfadd 消失），按「evidence 落在可分离机制/修复对象/验证预测」判为 **independent**，并列输出。
- 因果消除测试：改写打包布局不会消除 vfmul/vfadd 脚手架；改写 FMA 结构不会消除 vlse32 gather——双向均不能互相吞并，故均保留顶层。
- 复数算术合同（r/i 数据流、`acc_r=+ar·xr−ai·xi / acc_i=+ar·xi+ai·xr` 累加形态、epilogue alpha 符号）由 `rvv_complex_arithmetic_kernels.md` 以 **supporting** 身份支撑两 finding 的机制解释（其 §The fix 提供 Finding 2 的直接累加结构），不计顶层命中数。
- Evidence-mechanism layer：两 finding 均按 L4（compute/codegen micro-structure）与 L2（data movement / packing）混合归属——Finding 1 主归 L2 data-movement（packing 合同），Finding 2 归 L4 micro-structure；无 L0/L1 冲突。
- 排序（入口 A，按 local sample share 加总）：Finding 1 单行最高（23.09%，load path 合计≈39.4%），Finding 2 path 合计更高（≈45%）；两者独立、同层并列，**不得简单相加为 Amdahl 上界**（采样语义缺 global-period，见 Phase 1）。

## Phase 4 — Root-cause blueprint / 根因蓝图：cgemm_kernel_n

### Finding 1（Weight Repacking；pattern：`patterns/weight_repacking_for_vectorized_risc_v_gemm.md`，对应 rows-offload.md 命中 row「已向量化 GEMM/GEMV 中，sample 主导在权重/激活的 gather……修复方向是把权重一次性预交织成向量 tile」）

**1. Root cause**
`cgemm_oncopy` 将 A 以 **interleaved (r,i) 复数对**写入 packed buffer，导致 `cgemm_kernel_n` 每个 k 步必须用两条 stride=8B 的 `vlse32` gather 装载 A0r/A0i。依 pattern 文件 §Why this is slow 第 1 条：「非交织布局强制 gather……向量单元空等数据」，X100 的 LSU 对 8 元素跨步 gather 按元素级分解处理，代价远高于 32B 连续装载；23.09% 单行样本（17c5e）与 L1 miss 仅 1.81% 的组合正是指向 gather 的 LSU 元素代价而非 cache miss。依 §Why 第 3 条「一次性重排摊薄到多次复用」：repack 层（oncopy）每次 GEMM 调用只跑一次、其输出被 j-loop（N/8 次）复用，把 gather 移出内层循环是标准做法；本内核缺失「r/i 分平面」打包变体，把布局成本留给了每 k 步的 gather。

**2. The fix / 修复方式**
修复对象：`cgemm_oncopy` 打包布局 + 本内核 A 装载形态（必须配对修改，pattern §Verification「打包/使用一致性：重排后的布局与 GEMM/GEMV kernel 的读取顺序严格匹配」）。
- Before（当前）：oncopy 按 (r,i) 交错写；kernel 每 k 步 `vlse32 v2,(a1),a4`（A0r）+ `vlse32 v1,(t2),a4`（A0i），stride=8B。
- After（目标）：oncopy 对每个 8 行块按 **split-plane** 写：先连续 8 个实部（32B），再连续 8 个虚部（32B）；kernel 改为两条 unit-stride `vle32`：
```c
// Before (kernel, per k step)
vfloat32m1_t A0r = __riscv_vlse32_v_f32m1(&A[ai], 2, gvl);   // stride 2 floats
vfloat32m1_t A0i = __riscv_vlse32_v_f32m1(&A[ai+1], 2, gvl);
// After (kernel, per k step, 假设 split-plane: Ar[ai], Ai[ai])
vfloat32m1_t A0r = __riscv_vle32_v_f32m1(&Ar[ai], gvl);      // unit-stride 32B
vfloat32m1_t A0i = __riscv_vle32_v_f32m1(&Ai[ai], gvl);
```
- 适用前提：修改必须同时覆盖本文件内所有消费 oncopy 布局的段（main M×8 loop 0x17b7e、M&4 段 0x181ea、M&2 段 0x1840e、N&4 段 0x17ff2、N&2 段 0x18876，以及 M&1 标量段 0x18650 的 `vlseg2e32` 访问）以及同一打包器输出方的其他 kernel 消费者；`ai=m_top*K*2` 与 k 步指针步长（当前 A 指针每 k 加 64B）需按新布局重算。
- correctness contract：复数 (r,i) 配对语义不变；C 仍为内存交错复数，epilogue 的 `vsse32`/`vlse32` C 访问（0x17d6e–0x17eb6）可保留，或改 `vsseg2e32`/`vlseg2e32` 减少指令数（冷路径，次要）；打包器输出布局与所有 kernel 读取顺序严格一致；对 M、N、K 的任意余数（tail）路径同样验证。
- 限制/风险：打包器改动影响面大（oncopy 是本批次独立热点 003，其自身成本会变化）；L1 友好度可能变化（split-plane 使实/虚部各占一条 cache line，互不共享）；浮点结果不变（纯布局搬运，无数值重排）。
- 修复后预期 Profile signals：k-loop 中 0x17c5a/0x17c5e 两条 `vlse32.v` 变为 `vle32.v`；17c5e 的 23.09% 份额消失/大幅下降；LSU 压力下降；指令数每 k 步不变（仍是 2 条 load）但 LSU 元素代价大减。

**3. Baseline facts 回填**：hardware ISA=rv64imafdcvh_…（含 v/zve64d，无 th.v*）；build ISA=`baseline_gap: build ISA`（无 ELF）；VLEN=256 bits（vlenb=32）；bound type=compute/issue-bound（IPC 1.03）。

**4. 收益上界**：当前 sampled event（cpu-clock，local period）下，A gather 对局部样本份额 ≈23.1%（17c5e=23.09% + 17c5a=0.00%；Sampling IP precision 不足，按 interval 计）；连同 B 标量装载在内 load path 合计 ≈39.4%，其中直接可删的 gather 份额 ≈23.1%（B 的 16 条 `flw` ≈16.3% 需要额外布局/结构改动，不在本 finding 直接范围内）。因 percent type=local period 且函数级全局贡献未知，**不构成 workload 级 Amdahl 上界**。

**5. 三维路由判定**：
- current source：`__riscv_v*` intrinsic C 代码（annotate source 行直接证明），非 `.S`；
- implementation existence/reachability：目标实现已存在且正在执行（本函数即活动内核）；repack 层存在（oncopy/otcopy）但布局变体缺失；
- function-level policy：OpenBLAS 允许 intrinsic 内核（无强制 `.S` 政策），无 policy 冲突。

**6. Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

**7. Related PRs**（pattern 本地表，仅列 URL，未联网补充）：
- Related PRs：5 条 URL（`https://github.com/ggml-org/llama.cpp/pull/19121`、`https://github.com/alibaba/MNN/pull/3813`、`https://github.com/alibaba/MNN/commit/6afcf99fedaec90b5b363ce4c78317800f291443`、`https://github.com/alibaba/MNN/pull/4426`、`https://github.com/alibaba/MNN/pull/4433`）。

### Finding 2（FP Matmul microkernel 结构；pattern：`patterns/rvv_floating_point_matmul_and_gemv_kernels.md`，对应 rows-operator-rvv.md 命中 row「已向量化 kernel 中 accumulator chain、N/K 轴选择、……或 microkernel scheduling 主导」；复数累加形态依据 supporting pattern `patterns/rvv_complex_arithmetic_kernels.md` §The fix）

**1. Root cause**
k-loop 采用「tmp 物化」式复数 MAC 模板：先 `vfmul.vf`（tmp=A×B_i）、再 `vfmacc.vf`/`vfmsac.vf`（tmp+=A×B_r）、最后 `vfadd.vv`（ACC+=tmp），每轮 48 条向量 FP 指令完成 64 复数 MAC（8 列 × 8 行 × 2 ACC）。依 FP-matmul pattern §Why「FMA accumulator 依赖过长」与 complex-arith pattern §The fix 的直接累加结构，正确做法是把 tmp 层消除、让 4 条 FMA 直接命中 ACC：`acc_r += ar·xr; acc_r −= ai·xi; acc_i += ar·xi; acc_i += ai·xr`，即每列 4 条 FMA、每轮 32 条——**48→32，FP 指令削减 33%**。依 kernel-conventions.md §1：ACC 上是 loop-carried 依赖，需多个独立 accumulator 隐藏 FMA 延迟；本内核已有 16 个独立 ACC（r/i × 8 列），直接累加后 ILP 充足。当前模板额外产生 16 `vfmul`+16 `vfadd`（≈24.8% 局部样本）与 tmp 寄存器周转（依赖 `vmv1r.v` 拷贝等编排），在 IPC≈1.03 的发射受限内核上直接转化为周期。

**2. The fix / 修复方式**
修复对象：`cgemm_kernel_n` 的 k-loop（0x17c56–0x17d6a）计算结构，由 3 条指令/复数-ACC 改为 2 条直接 FMA：
```c
// Before (per B column j, per k step): 3 条指令
tmp_r = vfmul(A_i, B_ji);   tmp_r = vfmsac(tmp_r, B_jr, A_r);   // 实部（符号见下）
ACC_r = vfadd(ACC_r, tmp_r);
tmp_i = vfmul(A_r, B_ji);   tmp_i = vfmacc(tmp_i, B_jr, A_i);
ACC_i = vfadd(ACC_i, tmp_i);
// After (per B column j, per k step): 2 条指令，直接命中 ACC
ACC_r = vfmacc(ACC_r, B_jr, A_r);
ACC_r = vfnmsac(ACC_r, B_ji, A_i);    // ACC_r = -B_ji*A_i + ACC_r
ACC_i = vfmacc(ACC_i, B_jr, A_i);
ACC_i = vfmacc(ACC_i, B_ji, A_r);
```
- 适用前提：16 个 ACC（r/i × 8 列）保留在 m1；每轮指令从 48 降为 32；peak live ≈ 16 ACC + 2 A + 0–2 tmp = 18–20 < 32（kernel-conventions §2，无 spill）。
- correctness contract：必须**逐符号核对**当前代码的复数约定——当前 tmp0r 实际持有 `A0i·B0i − A0r·B0r`（负实部，`VFMACC_RR`=vfmsac），tmp0i 持有 `A0r·B0i + A0i·B0r`（虚部），最终 epilogue 经 `vfmacc(alphar)`+`vfnmsac(alphai)`（0x17d9e/0x17da6 等）折叠符号；改写后必须保持 C_new = alpha·(A·B) 的最终语义不变，并保持 `K=0`（直接 beta·C 路径）、M/N tail 路径与 alpha=0/1 特例。FP 结合顺序会改变（当前先 macc 再 add；新结构 4 条 FMA 链在 ACC 上），需在项目既有误差合同内对照 reference 验证（complex GEMM 通常允许容差）；NaN/Inf/±0 行为随重排变化需声明。
- 限制/风险：改变舍入顺序，需 OpenBLAS 既有正确性测试（如 512×512 及奇数形状 tail）全绿；若编译器对 16 个 ACC 的长 live range 产生 spill，需降低 unroll（保留 8 列或改 8 行×4 列 tile），不可用 spill 换融合（complex-arith pattern §The fix 明文约束）；N-block 扩到 16 列（32 ACC）会超 32 寄存器预算，不作为本 finding 方案。
- 修复后预期 Profile signals：k-loop 中 0x17c72–0x17cc2 的 16 条 `vfmul.vf` 与 0x17cc6–0x17d62 的 16 条 `vfadd.vv` 全部消失；仅存 32 条 `vfmacc.vf`/`vfmsac.vf`/`vfnmsac.vf`；对应地址段的局部样本份额（vfmul≈19.3% + vfadd≈5.5% ≈24.8%）转移到 macc 段并整体缩小；指令数/轮 48→32，IPC 与 gflops 上升。

**3. Baseline facts 回填**：同 Finding 1（hardware ISA 含 v；build ISA=`baseline_gap`；VLEN=256；bound=compute/issue-bound）。

**4. 收益上界**：FP path 局部样本份额 ≈45%（vfmul≈19.3%、macc/msac≈20.3%、vfadd≈5.5%）；其中脚手架（vfmul+vfadd）可直接删除的份额 ≈24.8%，替换 FMA 会承接部分份额。同 Finding 1，仅「当前 sampled event 下的局部样本份额」，不构成 Amdahl 上界（采样语义 gap）。

**5. 三维路由判定**：current source=intrinsic C；existence/reachability=内核已存在且执行；function-level policy=无政策冲突（同 Finding 1）。

**6. Implementation-shape proof**：不适用（非 missing `.S` 分支）。

**7. Related PRs**（pattern 本地表，仅列 URL）：
- Related PRs：21 条 URL（OpenBLAS `https://github.com/OpenMathLib/OpenBLAS/commit/0a967797a15617239523053633bf14be7895b25a`、`.../0acb60aab3c0134e879a68292904d8346dcd50ef`、`.../1cc377ef61d498b75c852aa4b9b042fe9422c347`、`.../2d82d144e2791e37d7a314237b638d85b156a2ec`、`.../809e1cba8f1f3f89972581e8b82f2ec52e51eadb`、`.../376d3a138faa0fe483a8fa8d4fa1ab0d395acf`、`.../2ae019161a85333a35018b517d4b34474a7694e9`；oneDNN `.../d6f82a2d0d0db41e6daaf20fbb4fd352843aac64`、`.../3bac96b8bc1fc9c348c986f38f65285693943d2f`、`.../8b48a77091062ce78959ccb96e43a8ee4e97022d`、`.../8c52facbe61845d86062c76280b4bc515160c03e`、`.../b73fc3172d3e3230cf24ae29cbb6a07a09507a43`、`.../bd984d09dc5985a19fb427ac46d19d2cbd5558dd`、`.../d2a44b9b855706a0df33f9b6d4fb84f5420fdeaf`、`.../d6107ddb8be72041dade165a233c0de69f7a1387`、`.../fe04323ab0b4bba79ee60109fc391bb36052e43c`、`.../3bac96b8bc1fc9c348c986f38f65285693943d2f`（去重后计一次）；llama.cpp `https://github.com/ggml-org/llama.cpp/pull/17318`、`.../17448`、`.../17314`、`.../17161`、`.../18199`、`.../20627`；MNN `https://github.com/alibaba/MNN/pull/4426`；vLLM `https://github.com/vllm-project/vllm/pull/44324`）。注：此计数按 pattern 文件本地表逐条保留，跨项目不去重。

### 排除项（逐 row，供复核）
- `rvv_register_group_utilization.md` / `rvv_register_budgeted_loop_unrolling.md`（rows-vectorized-tuning）：hot interval 无 vector spill/reload、无 vsetvl churn、back-edge 仅 1.48%，LMUL=m1 与 8 行 tile 匹配，loop-control 非主导 → 排除；
- `rvv_operand_form_selection.md`：main loop 已直接使用 `.vf` broadcast 形态（`vfmacc.vf v3,ft3,v1`），无冗余 `vmv.v.*` temporary；`vfmv.s.f` 仅出现在不执行的 M&1 冷路径 → 排除；
- `rvv_vector_state_management.md` / `rvv_inactive_lane_policy.md`：k-loop 内零 `vsetvl*`（vtype 在 loop 外一次建立，`vsetivli zero,8,e32,m1,ta,ma`），且用 `ta,ma` 非 `tu/mu` → 排除；
- `rvv_intrinsic_kernel_autovectorization_control.md`：无 compiler 插入冗余 vsetvl/spill 的 autovec 扰动证据（GCC 输出干净）→ 排除；
- `kernel_selection_and_runtime_specialization.md`：正确内核已被选中并执行（本函数即热点）→ 排除；
- `cache_aware_blocking_for_tiled_kernels.md`：无 tile-residency/尺寸扫描/复用拐点证据；打包成本由 oncopy/otcopy 独立承载 → 排除；
- `portable_rvv_across_vector_length_and_vendor_extensions.md`：`vsetivli zero,8` 硬编码 VLEN≥256，但本部署矩阵单一（SpacemiT X100, VLEN=256），无独立 portability 证据 → 排除（作为 Finding 1/2 蓝图中的风险提示）；
- `spacemit_ime_matrix_engine_acceleration.md` / `attached_matrix_register_file_gemm_offload.md`：无矩阵引擎卸载证据（ISA 含 `smaia`，但无 IME/矩阵指令出现于 annotate）→ 排除；
- `register_pressure_and_save_restore.md`：hot loop 无 spill/reload；prologue 12×`sd`+12×`fsd` 位于 0.00% 冷段 → 排除；
- `resource_aware_instruction_scheduling.md`：缺 X100 调度模型/PMU 依赖证据，无法成立 route gate → 排除；
- `load_store_addressing_mode_fusion.md`：k-loop 地址基址在 loop 外计算，每轮仅 `addi a2,a2,64`/`addi a1,a1,64` 增量，无可折叠地址序列 → 排除；
- `no-vectorization.md`：main loop 已充分向量化（`v*` 主导）→ 排除。

## Phase 5 — Verification forecast / 验证预测：cgemm_kernel_n

按收益上界顺序逐项验证（每 finding 独立验证，先分别后组合；supporting 不单独验证）：

**Finding 1（Weight Repacking / A 布局）**——修复对象：`cgemm_oncopy` 打包布局 + 本内核 A 装载。
- 应消失/缩小：k-loop 锚点行 `23.09 :  17c5e:  vlse32.v v1,(t2),a4` 与 `0.00 :  17c5a:  vlse32.v v2,(a1),a4` 被 `vle32.v` 取代，该地址段局部样本份额从 ≈23.1% 显著下降；
- 应出现：annotate 中出现 unit-stride `vle32.v` 装载（新地址），A 指针步进改为 32B/32B 分平面；
- 额外验证：`cgemm_oncopy` 自身（rank 003）annotate 重测，确认 split-plane 写入成本可接受；打包器-内核布局严格匹配（pattern §Verification「打包/使用一致性」）；512×512 及 M/N 余数形状（513、511、4、2、1 尾）正确性测试全绿；无 RVV 回退路径正确性不变；
- 升级到 workload 级结论所需最小补采：`perf record` 全局采样 + `perf annotate --percent-type=global-period`（获取函数级/事件级全局份额），以及 `readelf -A` 确认 build ISA。

**Finding 2（FP Matmul microkernel 结构 / 直接累加）**——修复对象：`cgemm_kernel_n` k-loop FMA 结构。
- 应消失/缩小：k-loop 锚点行 `3.81 :  17ca2:  vfmul.vf v13,v2,ft0`、`3.39 :  17ca6:  vfmul.vf v12,v1,fa1`、`2.12 :  17d46:  vfadd.vv v21,v21,v10`、`2.12 :  17d5e:  vfadd.vv v15,v15,v4` 所代表的 16 条 `vfmul.vf` 与 16 条 `vfadd.vv` 全部消失（对应地址段份额 ≈24.8%）；
- 应出现：k-loop 中 32 条直接命中 ACC 的 `vfmacc.vf`/`vfmsac.vf`/`vfnmsac.vf`（无 tmp 中间寄存器），每轮向量 FP 指令 48→32；
- 额外验证：与 scalar/reference 对照（含 conjugated、transpose、alpha=0/1、K=0 路径）在项目 FP 容差合同内一致；NaN/Inf/±0 声明；M/N 尾块（M&4/2/1、N&4/2/1）正确性；perf stat 重测预期 instructions/cycle 下降、IPC 上升、gflops（当前 16.57）提升；
- 升级所需最小补采：同 Finding 1（global-period + build ISA）。

**组合验证**：两修复分别验证后再组合 A/B benchmark（短/中/长 K，512×512 及余数形状），确认无回退且收益可加（指令数与 LSU 压力分别下降）。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现（载荷：1/1 组；cgemm_kernel_n） | ✅（1/1 组；`Functions under analysis: [cgemm_kernel_n]`） |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 表 + 2 个 L0 gate + bound gate + Sampling IP precision 行；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`） | ✅（7 行齐全；含 Sampling IP precision 行 `baseline_gap: sampling IP precision`） |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md、rows-offload.md；顶层 finding=2（independent）；evidence 锚点=17c5e/17c5a/17c62/17ca2/17cfe/17d0e/17d46/17d5e；supporting=1（complex-arith）；排除条数=15 条逐 row + 4 类整组 exclude；推导式=2 组双 confidence） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern 文件=weight_repacking_for_vectorized_risc_v_gemm.md、rvv_floating_point_matmul_and_gemv_kernels.md、rvv_complex_arithmetic_kernels.md + 对应命中 row + 引用短语首词=「非交织布局强制 gather」/「FMA accumulator 依赖过长」/「acc_r = rvv_fmacc」；The fix 含 before/after、correctness、风险、Profile 信号锚点；Related PRs：weight-repack 5 条 URL、FP-matmul 21 条 URL、complex-arith 3 条 URL） | ✅ |
| 5 | 路径合规（载荷：模式 A（profile_backed）；路径=intrinsic→operator/offload/codegen；class 列表=4 include + 4 exclude；两 finding 按机制/地址可分账 → independent；收益按 local share 排序、无 Amdahl；th.v* 不适用未停扫） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧=17c5e/17c5a、17ca2/17ca6/17d46/17d5e；出现侧=weight_repacking §Verification 打包一致性、rvv_floating_point_matmul §Verification 性能预测） | ✅ |
| 7 | 契约边界合规（载荷：无实施询问、无代码修改、无补丁生成；交付止于证据+蓝图+The fix+验证预测；无向用户提问） | ✅ |

修正记录：无
