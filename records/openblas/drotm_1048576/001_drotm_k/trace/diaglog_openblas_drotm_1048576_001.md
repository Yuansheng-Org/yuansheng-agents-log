Functions under analysis: [drotm_k]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-drotm_k-annotate.txt`；event=`cpu-clock`，280 samples，percent type=**local period**；hot loop 完整覆盖，源码行与 `kernel/riscv64/rotm_rvv.c` 逐字一致）
- perf stat（bound/context）：已提供（`14-openblas-benchmark-riscv-drotm_1048576.txt`；IPC=0.1916、L1_dcache_load_miss_rate=4.813%、branch_miss_rate=0.449%；cache_references/LLC 为 NA）
- workload/binary/DSO/source context：已提供（OpenBLAS benchmark `drotm.goto`，n=1048576，incx=incy=1；源码 `kernel/riscv64/rotm_rvv.c` 在 workspace 检出中定位到，annotate 源码行号与其逐字吻合）
- readelf -A（`Tag_RISCV_arch`）：缺失（详见 Phase 1）
- hardware ISA（`/proc/cpuinfo`/hwprobe）：已提供（metadata cpuinfo：`rv64imafdcvh_..._zvbb_..._zve64d_..._zvl*`，含 `v`；SpacemiT X100）
- `vlenb`：已提供（`vlen_bits=256`，vlenb=32）
- 采样元数据（event/percent type/scope/窗口）：部分提供（event=cpu-clock、percent type=local period、单次运行同窗口；**函数 workload 级贡献未正式记录**，由采样率推算 drotm.goto≈280/288≈97% 运行时间）
- Sampling IP precision：缺失（无 precise_ip / Exact-IP 记录）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`v` 存在（RVV 1.0），含 `zvl256b` 体系（zve64d/zve64x 等）；无 `th.v*` |
| Build ISA | `baseline_gap: build ISA`（本 run 无 ELF binary，无法 readelf -A；间接证据：annotate 反汇编全为 RVV 1.0 `v*` mnemonic，且 KERNEL.RISCV64_ZVL256B 将 DROTMKERNEL 指向 `rotm_rvv.c` → build 含 `v`） |
| Vector flavor | RVV 1.0（`vsetvli e64,m8`、`vlse64.v`、`vse64.v`、`vfmacc.vf`、`vfmsac.vf`），无 flavor mismatch |
| VLEN | 已提供：256 bits（vlenb=32）；e64,m8 → VLMAX=32 doubles/block |
| Bound type | **memory-latency-bound**：IPC=0.19（workload 级）、L1 dcache load miss 4.8%、每 iteration ≈368 cycles（见 Phase 3），FMA 指令样本≈0 |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=local period（非 global-period）；同窗口；函数 workload 贡献未知 → 禁止 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip；单行样本只锚定 loop interval，不做指令级 latency 归因） |

L0 baseline gate：hardware 有 `v`，build 反汇编证明含 `v`（RVV 1.0）→ 无 hardware/build mismatch；无 `th.v*` → 无 flavor gate 阻塞；annotate 完整覆盖 hot loop → 入口条件 A（profile_backed）。
Bound-type gate：明确 memory-latency-bound → 对 compute-vectorization 类修复下调 performance-impact confidence；本函数命中 latency-hiding 类 pattern，route 不降，impact 封顶 Medium。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`drotm_k`（OpenBLAS DROTM modified Givens rotation kernel，double precision，Level-1）。

Hot loop 锚点（**全部 280 个局部样本都落在 dflag>0（源码 L30）向量主循环区间 0xf6e–0xfa4**）：

- `46.07 :  f7e: addi a7,a1,8`（两条 vlse64 之间的地址计算）
- `31.43 :  f86: mul a2,a5,a6`（`a2 = vl * stride`，块推进字节数）
- `10.36 :  fa0: vsse64.v v8,(a4),a6`
- `11.79 :  fa4: bgtz a0,f72`
- `0.36  :  f94: vfmsac.vf v8,fa5,v16`

区间结构（每 iteration 一个 m8 block=32 doubles，共 32768 次迭代）：`vsetvli`(f72) → `vlse64.v v16`(f7a, 读 dy) → `addi`(f7e) → `vlse64.v v8`(f82, 读 dx) → `mul/sub`(f86/f8a) → `vmv8r.v v24,v16`(f8c) → `vfmacc.vf v24`(f90) → `vfmsac.vf v8`(f94) → 指针 add（f98/f9a）→ `vsse64.v v24`(f9c) → `vsse64.v v8`(fa0) → `bgtz`(fa4)。

Sampling IP precision 未知 → 不做单指令归因；77.5% 样本集中在两条 load 周围的标量指令（f7e+f86）是 memory-latency stall 的 shadow，机制按 loop interval 归因。annotate 覆盖完整，无 tail-loop 样本（n=1048576=32768×32 恰好整除）。

## Phase 3 — Pattern scan / 模式扫描：drotm_k

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源是 intrinsic C（`rotm_rvv.c`），非手写 `.S`；OpenBLAS riscv64 Level-1 内核无"独立 `.S`"政策（policy 四证缺 policy/existence 两证）→ 不进入 missing-`.S` 分支
2. `rows-operator-rvv.md` — **include**：intrinsic 生成、已向量化的 elementwise 算术内核（含 strided 访存形态），需逐行评估
3. `rows-string-memory.md` — **exclude**：非 copy/fill/compare/checksum
4. `rows-vectorized-tuning.md` — **include（必选）**：完整 annotate 有 `v*` 且非手写 `.S`，修正对象是 LMUL/展开/寄存器组
5. `rows-codegen.md` — **include**：kernel-selection（rotm_rvv.c 已正确选中）、resource-aware scheduling、induction-variable、register pressure、cache-blocking
6. `rows-offload.md` — **exclude**：X100 有 IME 矩阵引擎但仅针对 GEMM/conv，Level-1 elementwise 无卸载路径；无 packing/portable 层证据
7. `rows-crypto.md` — **exclude**：无密码学原语
8. `rows-runtime-os.md` — **exclude**：用户态 BLAS benchmark，无 RTOS/kernel 路径

### Local performance pattern scan: `drotm_k`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Register-Budgeted RVV Loop Unrolling（primary） | intrinsic RVV main loop 一轮只处理一个 m8 block；load→FMA→store 全串行、寄存器组跨迭代复用导致零 overlap；backward branch/control 簇主导区间；LMUL=m8 已达合法上限 | High | Medium（memory-latency 竞争瓶颈） | `patterns/rvv_register_budgeted_loop_unrolling.md` |

**(a) 逐字 evidence 引用**（hot interval 0xf6e–0xfa4，dflag>0 主循环）：
- `46.07 :  f7e: addi a7,a1,8` — 位于 `f7a: vlse64.v v16,(a4),a6` 与 `f82: vlse64.v v8,(a7),a6` 两条 strided load 之间
- `31.43 :  f86: mul a2,a5,a6` — `a2=vl*stride`（stride=8 单步长），紧随第二条 load
- `11.79 :  fa4: bgtz a0,f72`、`10.36 :  fa0: vsse64.v v8,(a4),a6`、`0.36 : f94: vfmsac.vf v8,fa5,v16`
- perf stat：`IPC=0.1916`；`L1_dcache_load_miss_rate=4.813%`；578ms 内 1.26G cycles（≈2.18GHz）→ 32768 次迭代每迭代 ≈368 cycles，而迭代体仅 11 条指令
- 归属：该区间即唯一 hot basic block，280/280 局部样本全部在此；FMA（f90/f94）合计 ≈0.36% → compute 资源几乎空闲，样本 shadow 属于 strided load 的访存延迟

**(b) 互斥邻居排除**（该 row 行内互斥判据点名）：
- **RVV Register-Group Utilization**：当前 LMUL=m8 已是 e64 下合法架构上限（3 个 live m8 组=24 个 vector regs），不满足"LMUL 小于合法候选边界"；该 row 行内判据明确"若 LMUL 已达合法/实测最优边界且证据指向固定 LMUL 下的串行 recurrence/ILP，转 register-budgeted-loop-unrolling row" → 路由到本 row
- **RVV Vector-State Management**：每迭代仅 1 条 `vsetvli e64,m8,ta,ma`（f72），vtype 稳定、无 mask/state churn，vsetvli 样本 0.00% → 不命中
- **RVV Contiguous Elementwise / No vectorization**：主循环已向量化（`v*` 指令齐备、vfmacc/vfmsac 已正确 lowering），row 的 scalar/zero-`v*` gate 不满足 → 不命中（机制重合仅作 supporting 上下文，自身 row gate 不成立，不入顶层）
- **RVV Strided Memory Access**：row gate 要求 "scalar loop ... 热点仍以标量 load/store 为主"；本循环已用 `vlse64.v/vsse64.v` 向量化 → 不命中；"unit-stride（stride==8）数据用 strided load" 作为主修复内实现细节，不单独成为 matched finding
- **Kernel Selection**：`KERNEL.RISCV64_ZVL256B` 中 `DROTMKERNEL = rotm_rvv.c`，RVV 内核正是热路径 → 无 fallback 未选中问题
- **Resource-Aware Instruction Scheduling**：修复对象是循环结构（多 block 重叠），不是单 basic block 内 scheduler 重排；该 row 自身是 unroll row 的 exclusion（"目标是补齐 target-core latency/resource model 或在单个 basic block 内重排独立指令"）→ 不命中
- **Loop Induction Variable Strength Reduction**：循环已用指针递增（add a1,a1,a2）；`mul a2,a5,a6` 是 block 推进量（vl×stride），其样本是 load-stall shadow 而非 mul 自身成本；fixed-VL 化后该 mul 自然消失，不作为独立 finding
- **Cache-Aware Blocking**：单遍流式 2×8MB 工作集、无 tile 复用/尺寸拐点 → 不命中
- **Register Pressure / Save-Restore**：无 spill/reload（24/32 vector regs），无冗余 mv/context save → 不命中

**(c) 双 Confidence 推导式**：
- `route: rotm_rvv.c intrinsic provenance（源码树+annotate 逐字吻合） + LMUL=m8 达合法上限 + 固定 LMUL 下串行 load→FMA→store 链 + 无 spill 的 m4×2/m2×4 unroll 候选 + 区间聚合/数据流/PMU 三类证据 → High`
- `impact: 局部样本份额 100%（280/280） + VLEN=256 已知 + bound type=memory-latency（竞争瓶颈） + baseline_gap: sampling metadata（local period、workload 贡献未正式记录） + baseline_gap: sampling IP precision → Medium`

**多候选仲裁**：1 个顶层 matched finding（Register-Budgeted RVV Loop Unrolling，evidence-mechanism layer **L4** compute/codegen micro-structure）。无 independent/companion。`dynamic priority: 入口 A 单 finding，sample share 加总 = 1.0（280/280 局部）`。

## Phase 4 — Root-cause blueprint / 根因蓝图：drotm_k

对应 Phase 3 通过 gate 的 row：`rows-vectorized-tuning.md` → Register-Budgeted RVV Loop Unrolling。

**1. Root cause**：drotm_k 的 dflag>0 主循环虽然已向量化（e64,m8），但每 iteration 只处理一个 32-double block，且 `load(v16,v8) → FMA(v24,v8) → store(v24,v8)` 严格串行、寄存器组跨迭代复用：下一迭代的 `vlse64.v v16/v8` 必须等当前迭代 `vfmacc/vfmsac` 读完 v16/v8 并等 store 释放后才能发射，跨迭代零 overlap。每条 m8 strided load 256B 跨 4 个 cache line，L1 miss 率 4.8% 下暴露完整访存延迟，每迭代 ≈368 cycles（IPC 0.19），而 FMA 样本 ≈0.36% 证明向量计算单元几乎全程空闲。依据 `patterns/rvv_register_budgeted_loop_unrolling.md` §Why this is slow：「RVV 只有 32 个 vector registers；`LMUL * peak_live_vectors <= 32` 只是架构计数上限，不是充分条件」——m8 单组把 24/32 个 vector regs 全部占用，架构上不可能再放一个 block 的寄存器做双缓冲，这是零 overlap 的结构性根因。

**2. The fix / 修复方式**（与 `patterns/rvv_register_budgeted_loop_unrolling.md` §The fix 一致）：

先枚举合法 frontier `m1/m2/m4/m8 × unroll=1/2/4/8`，推荐 **m4 × unroll=2**（或 m2 × unroll=4），每迭代处理 2 个独立 block，寄存器预算：每 block 3 个 live m4 组（z、w→dy、dx）=12 regs，2 block=24 regs ≤ 32，无 spill；两 block 的 load 独立发射，block B 的 load 与 block A 的 FMA/store 重叠 → 隐藏访存延迟。

```c
// Before（现状，每迭代 1 个 m8 block，全串行）:
for (size_t vl; n > 0; n -= vl, dx += vl*i__2, dy += vl*i__2) {
    vl = VSETVL(n);                                  // 每迭代重算 vsetvli
    v_w = VLSEV_FLOAT(&dx[1], stride, vl);           // vlse64.m8 256B
    v_z  = VLSEV_FLOAT(&dy[1], stride, vl);          // vlse64.m8 256B
    v_dx = VFMACCVF_FLOAT(v_z,  dh11, v_w, vl);      // z + dh11*w
    v_dy = VFMSACVF_FLOAT(v_w, dh22, v_z,  vl);      // w - dh22*z
    VSSEV_FLOAT(&dx[1], stride, v_dx, vl);           // vsse64.m8
    VSSEV_FLOAT(&dy[1], stride, v_dy, vl);           // vsse64.m8
}
```

```c
// After（m4 × unroll 2；fixed-VL 主循环 + runtime-VL tail；unit-stride fast path）:
const size_t epr = __riscv_vsetvlmax_e64m4();        // VLMAX(m4)=16，hoist 出主循环
const size_t step = epr * 2;                         // 32 doubles/iteration
for (size_t i = 0; i + step <= n; i += step) {
    // block A（低 16 元素）与 block B（高 16 元素）load 均提前发射，与 FMA 重叠
    v_z0  = LD(&dy[i],        stride, epr);   // stride==8 时用 vle64.v
    v_w0  = LD(&dx[i],        stride, epr);
    v_z1  = LD(&dy[i + epr],  stride, epr);
    v_w1  = LD(&dx[i + epr],  stride, epr);
    v_dx0 = VFMACCVF(v_z0, dh11, v_w0, epr);
    v_dy0 = VFMSACVF(v_w0, dh22, v_z0,  epr);
    v_dx1 = VFMACCVF(v_z1, dh11, v_w1, epr);
    v_dy1 = VFMSACVF(v_w1, dh22, v_z1,  epr);
    ST(&dx[i],        stride, v_dx0, epr);
    ST(&dy[i],        stride, v_dy0, epr);
    ST(&dx[i + epr],  stride, v_dx1, epr);
    ST(&dy[i + epr],  stride, v_dy1, epr);
}
for (size_t vl; i < n; i += vl) { /* runtime-VL tail，覆盖 0..step-1 剩余 */ }
```

适用前提：incx==incy>0 路径（L30）；`stride==8`（unit-stride）时 LD/ST 用 `vle64.v/vse64.v` 激活硬件 prefetch/burst，非 unit-stride 保留 `vlse64.v/vsse64.v`（rotm_rvv.c 已分 L30/L70 路径，L70 路径按 stride_x/stride_y 处理）。
Correctness contract：BLAS DROTM 语义——每元素 `dx' = dy + dh11*dx`、`dy' = dx - dh22*dy` 逐 lane 独立，FMA 融合运算逐元素保持（不跨元素重排，NaN/±0/舍入逐元素不变）；unroll 不改变单元素结果；tail 必须精确覆盖剩余元素；dx/dy 为独立数组（无 alias 风险）；stride 符号处理保持现语义。
限制/风险：m4×2 live=24 regs 已接近上限，需反汇编确认无 spill；代码尺寸与 I-cache 压力；短输入（n<step）回归风险需 short-cutoff；X100 上 vlse vs vle 的吞吐与 misaligned vector 行为未 A/B 实测（`RISCV_HWPROBE_KEY_MISALIGNED_VECTOR_PERF` 未读）。
预期 Profile 信号：f7e/f86 的 stall-shadow 样本大幅坍缩；bgtz 份额减半；主循环内 `vsetvli` 与 `mul`（块推进）消失；同 workload IPC 从 0.19 显著上升、official_mflops（当前 1135.83）上升。

**3. Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（含 v/zvl256b）；build ISA=`baseline_gap: build ISA`（无 ELF，反汇编+ZVL256B profile 间接证明含 v）；VLEN=256；bound type=memory-latency-bound（IPC 0.19、L1 miss 4.8%）。

**4. 收益上界**：当前 sampled event（cpu-clock）下 drotm_k 局部样本份额 **100%（280/280 全部落在该 interval）**；因 percent type=local period 且 workload 贡献未正式记录（`baseline_gap: sampling metadata`），不声明 workload 级 Amdahl 上界。

**5. 三维路由判定**：`current source`=RVV intrinsic C（`kernel/riscv64/rotm_rvv.c`，annotate 源码行号与文件逐字吻合）；`implementation existence/reachability`=存在且可达（`KERNEL.RISCV64_ZVL256B` 注册 `DROTMKERNEL = rotm_rvv.c`，热路径即该内核，无 fallback 未选中）；`function-level policy`=OpenBLAS riscv64 Level-1 以 `.c` intrinsic kernel 为准（无强制独立 `.S` 政策）→ 不进入 missing-`.S` 分支。

**6. Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。

**7. Related PRs**：依据 `patterns/rvv_register_budgeted_loop_unrolling.md` §Related PRs — `Related PRs：2 条 URL`：https://github.com/google/XNNPACK/commit/0c7b565c2fc1debcb12a4dc8941a56c9f0ced6fe 、https://github.com/google/XNNPACK/pull/10403

## Phase 5 — Verification forecast / 验证预测：drotm_k

修复对象：`kernel/riscv64/rotm_rvv.c` 的 dflag>0（L30）主循环——m8 单 block 改 m4×unroll2（或 m2×unroll4）多独立组、fixed-VL 主循环 + runtime-VL tail、stride==8 时 vle64/vse64。

**应消失/缩小**（锚定 Phase 3(a) 引用行）：
- `f7e: addi a7,a1,8`（46.07%）与 `f86: mul a2,a5,a6`（31.43%）的 shadow 样本份额大幅坍缩（load 延迟被跨 block 重叠隐藏）
- `fa4: bgtz a0,f72`（11.79%）份额减半（迭代数减半）
- 主循环内 `f72: vsetvli a5,a0,e64,m8,ta,ma` 与 `f86: mul` 从主循环消失（fixed-VL hoist）
- `f7a/f82: vlse64.v`（stride==8 场景）替换为 `vle64.v`

**应出现**（锚定 `patterns/rvv_register_budgeted_loop_unrolling.md` §Verification）：
- 主循环每迭代出现 2 个（或 4 个）独立 load/FMA 链、多组独立 vector 寄存器；无 vector spill/reload
- 「每个输出对应的 backedge/control/address-update 减少」；per-iteration cycles/element 显著下降
- 正确性覆盖：n=0、n<step、step 整倍数（本 workload n=1048576=32768×32 恰为整倍数）、全部 tail 长度、非 unit-stride（incx≠1）路径回归、dx/dy 独立数组无 alias
- 数值语义：每元素 FMA 融合结果与 baseline 逐元素一致（NaN/±0/rounding）
- benchmark：同 workload IPC 上升（>0.19）、official_mflops 上升（>1135.83）；无短输入回归（需 short-cutoff 验证）

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：N 组 Phase 3–5 全部出现 | ✅ 1/1 组；`[drotm_k]` |
| 2 | Phase 1 输出要求满足 | ✅ 7 行 baseline 表；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行 |
| 3 | Phase 3 输出要求满足 | ✅ 8 项 Class selection trace（asm exclude / operator-rvv include / string-memory exclude / vectorized-tuning include / codegen include / offload exclude / crypto exclude / runtime-os exclude）；`Classes scanned:` rows-vectorized-tuning.md、rows-operator-rvv.md、rows-codegen.md；顶层 finding=1；evidence 锚点：`46.07 : f7e: addi a7,a1,8`、`31.43 : f86: mul a2,a5,a6`、`IPC=0.1916`；supporting=0；排除条数=11；推导式=1 |
| 4 | Phase 4 输出要求满足 | ✅ 已读 pattern：`patterns/rvv_register_budgeted_loop_unrolling.md`；命中 row：rows-vectorized-tuning row 4（Register-Budgeted RVV Loop Unrolling）；引用短语首词：`RVV 只有 32 个 vector registers`、`LMUL * peak_live_vectors <= 32`；The fix 含 before/after、correctness、风险、Profile 信号锚点；`Related PRs：2 条 URL` |
| 5 | 路径合规 | ✅ 模式 A（profile_backed）；主归属=rows-vectorized-tuning（intrinsic RVV，非 .S）；L4 机制层；单 finding 按局部样本份额排序；`th.v*` 无（RVV 1.0）未触发 flavor gate |
| 6 | Phase 5 两侧锚定 | ✅ 消失侧：f7e/f86/fa4/f72/f7a 引用行；出现侧：`patterns/rvv_register_budgeted_loop_unrolling.md` §Verification（backedge/control 减少、无 spill、cycles/element 改善） |
| 7 | 契约边界合规 | ✅ 无实施询问/代码修改/补丁生成；gap 走标注+可选命令；交付物止于 Profile 证据、根因蓝图、The fix、验证预测 |

修正记录：无