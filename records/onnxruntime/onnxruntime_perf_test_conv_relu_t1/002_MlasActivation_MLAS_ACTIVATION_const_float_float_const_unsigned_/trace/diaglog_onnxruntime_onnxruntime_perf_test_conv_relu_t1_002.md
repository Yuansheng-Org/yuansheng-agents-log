Functions under analysis: [MlasActivation(MLAS_ACTIVATION const*, float*, float const*, unsigned long, unsigned long, unsigned long)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`002-...MlasActivation(...)-annotate.txt`，`libonnxruntime.so.1.30.0`，event=`cpu-clock`，191 samples，percent: local period，hot loop 覆盖完整）
- perf stat（可选 bound/context）：已提供（`perf_stat_onnxruntime_perf_test_conv_relu_t1.txt`：cycles=24470762914、instructions=54401011884、task-clock=11129.14ms、IPC≈2.22）
- workload/binary/DSO/source context：已提供（onnxruntime commit `d4d792f545760ab292c5e68b2bb547d89306a7f0`，`onnxruntime/core/mlas/lib/activate.cpp` 源码在本仓库可读，热点 object=`libonnxruntime.so.1.30.0`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（`libonnxruntime.so`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`，含 `v1p0`）
- hardware ISA（metadata / cpuinfo）：已提供（SpacemiT X100，`rv64imafdcvh_..._zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_...`，含 `v`，RVV 1.0）
- `vlenb`：已提供（metadata：`vlen_bits=256`，`vlenb=32`）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=`cpu-clock`；percent type=`local period`；同一运行窗口；函数占整个 workload 的贡献未知 → `baseline_gap: sampling metadata`，详见 Phase 1）
- Sampling IP precision：缺失（`precise_ip`/Exact-IP 未知 → `baseline_gap: sampling IP precision`，详见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_...`：RVV 1.0 `v` 存在，含 `zvbb/zvbc/zve32f/zve32x/zve64d/zve64f/zve64x/zvfh/zvfhmin`；vendor=SpacemiT，core=spacemit-x100，OoO（来源：metadata cpuinfo） |
| Build ISA | `libonnxruntime.so` Tag_RISCV_arch 含 `v1p0`、`zvl128b1p0`；无 IFUNC/multiversion 证据（MlasActivation 走编译期 jump table dispatch） |
| Vector flavor | annotate 全为标准 RVV 1.0 `v*` mnemonic（`vle32.v/vsetivli/vmflt.vv/vmerge.vim/vse32.v` 等），无 `th.v*` → 与硬件 RVV 1.0 一致，无 flavor mismatch |
| VLEN | `vlenb=32` → VLEN=256 bits；e32,m1 时 VLMAX=8 lanes；e32,m2/m4/m8 时 VLMAX=16/32/64 lanes |
| Bound type | IPC=54401011884/24470762914=2.22（system-level）；CPUs utilized 0.993；无 cache/memory counter → 无法细分 cache/memory-bound → `baseline_gap: bound type`（缺 cache/memory counters）；现有证据倾向 compute/throughput 取向而非明显 latency-bound |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=`local period`（非 global-period）；同一窗口；函数 workload 贡献未知 → `baseline_gap: sampling metadata`；百分比只表达函数内局部样本份额，禁止 workload 级 Amdahl |
| Sampling IP precision | 未知（无 `precise_ip`/Exact-IP 信息）→ `baseline_gap: sampling IP precision`；单行占比只锚定 basic block / loop interval，不承担 instruction-latency 根因 |

L0 baseline gate 判定：
- hardware 有 `v` ∧ build 有 `v`（zvl128b min）→ **无 hardware/build mismatch**。
- 无 `th.v*` → vector flavor gate 无约束，不冻结任何 route。
- Bound-type gate：无 cache/memory counter，bound type 未完全确定，但 IP 精度与采样语义缺项已使 performance-impact confidence 封顶（Low）。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个）。191 samples 全部归属 switch 分派后的 `MlasActivationKernel<(MLAS_ACTIVATION_KIND)1,false>`（Relu、无 bias）路径。

hot loop 锚点（`MlasActivationKernel<(MLAS_ACTIVATION_KIND)1,false>` 的 `do {} while (n >= 4)` 主循环，地址区间 d98c7a–d98c92）：
```
91.10 :   d98c82: addi    a3,a3,-4
 7.85 :   d98c90: addi    a4,a4,16
 0.52 :   d98c88: vmerge.vim      v1,v1,0,v0
 0.00 :   d98c7e: vle32.v v1,(a4)
```
该 interval 合计 189/191 = 98.95% 的函数内样本；其余 0.52%（d9859e prologue `addi sp,sp,-64`）+ 0.52%（d98c88 已计入）为冷路径。annotate 覆盖完整（含 hot loop body、tail loop 与全部 dispatch 分支）。

区间结构（每轮固定处理 4 个 float）：
- `d98c72: vsetivli zero,4,e32,m1,ta,ma`（循环外 hoist，固定 vl=4）
- `d98c7e: vle32.v v1,(a4)` → `d98c82: addi a3,a3,-4`（n-=4）→ `d98c84: vmflt.vv v0,v1,v3` → `d98c88: vmerge.vim v1,v1,0,v0` → `d98c8c: vse32.v v1,(a4)` → `d98c90: addi a4,a4,16`（buffer+=4）→ `d98c92: bltu a2,a3,d98c7e`

Sampling IP precision 未知：`addi a3,a3,-4` 的 91.10% 只锚定该 loop interval，不解释为该条 addi 的 latency/cost；根因按 interval 级机制（固定 4-lane 向量化 + 每元素循环开销）归因。

## Phase 3 — Pattern scan / 模式扫描：MlasActivation(MLAS_ACTIVATION const*, float*, float const*, unsigned long, unsigned long, unsigned long)

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — 当前代码来源是 compiler-generated C++ template（`activate.cpp` MlasActivationKernel），非手写 `.S`；无 dispatch slot/policy 要求独立 `.S` 的信号。
2. `rows-operator-rvv.md` — **include** — compiler/template 生成代码，热点语义合同是 elementwise activation（Relu）。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/sentinel/compare/checksum/string 语义。
4. `rows-vectorized-tuning.md` — **include** — annotate 已有标准 `v*`（非 `.S`），修正对象是 RVV lane 宽度/LMUL/循环结构。
5. `rows-codegen.md` — **include** — compiler-generated 指令形态：hot interval 由 loop induction（addi/bltu）样本主导，需扫 induction/codegen rows。
6. `rows-offload.md` — **exclude** — 无矩阵引擎/packed-SIMD/权重重排/并行分块信号。
7. `rows-crypto.md` — **exclude** — 无密码学原语信号。
8. `rows-runtime-os.md` — **exclude** — 用户态 MLAS 库，无 timer/CSR/ISR 热点。

Classes scanned: `rows-operator-rvv.md`、`rows-vectorized-tuning.md`、`rows-codegen.md`

### Local performance pattern scan: `MlasActivation(...)`
| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | hot loop 已向量化但 `vsetivli zero,4,e32,m1` 固定 vl=4（VLEN=256 下 VLMAX(e32,m1)=8，仅用半宽）；peak live=2 vector group+mask，m2/m4/m8 合法无 spill；98.95% 样本落在 loop induction `addi` 上 | High | Low | `patterns/rvv_register_group_utilization.md` |

**(a) 逐字 evidence 引用**（所属 basic block：d98c7a–d98c92，`MlasActivationKernel<(MLAS_ACTIVATION_KIND)1,false>` 主循环）：
```
91.10 :   d98c82: addi    a3,a3,-4
 7.85 :   d98c90: addi    a4,a4,16
 0.00 :   d98c7e: vle32.v v1,(a4)
 0.52 :   d98c88: vmerge.vim      v1,v1,0,v0
```
循环外 hoist 行：`d98c72: vsetivli zero,4,e32,m1,ta,ma`（每轮固定 4 lanes = 16 bytes）。tail 对照行（同一 TU 内编译器已能生成 runtime-VL 形态）：`d98c4a: vsetvli zero,a4,e32,m1,ta,ma`。

**(b) 互斥邻居排除**：
- Register-Budgeted RVV Loop Unrolling（rows-vectorized-tuning 行内互斥）：该 row 明文规定「若缩短 live range 后更大 LMUL 合法、无 spill，并直接减少 iteration/control overhead，则由 register-group row 认领、unroll 延后」。本例 peak live = v1(data)+v3(zero)+v0(mask)=2 group+mask，m2/m4/m8 均满足 `LMUL*peak_live<=32`（2×8=16+mask<32）且 hot loop 内无 spill/reload → 归 register-group，unroll 不命中。判别观察：hot loop 内无任何 `vsetvl`/`vsetivli`（已 hoist 到 d98c72），无 vector spill 指令。
- No vectorization / RVV Elementwise Activation Kernels（rows-operator-rvv）：该两 row 要求 scalar hot path（zero `v*`）；主循环存在 `d98c7e: vle32.v`、`d98c88: vmerge.vim`、`d98c8c: vse32.v` 等标准 `v*` → 不命中。
- RVV Vector-State Management：hot loop 内无重复建立兼容 `vl`/`vtype`（d98c72 单次 hoist，tail loop 独立 vsetvli）→ 不命中。
- RVV Inactive-Lane Policy：主循环 vtype 为 `ta,ma`，无 `tu/mu`、无冗余 mask clear → 不命中。
- RVV Operand-Form Selection：零向量 v3 在循环外 `vmv.v.i v3,0`（d98c76）一次，被 `vmflt.vv` 与 `vmerge.vvm` 两条 op 跨迭代复用——row 行内注明「同一 broadcast 被多条 op 复用时仅作为低置信候选」；且 merge 已用 immediate 0（`vmerge.vim`）→ 不命中，仅记作 symptom。
- Loop Induction Variable Strength Reduction（rows-codegen）：循环已用指针递增（`d98c90: addi a4,a4,16`）+ 独立计数器递减（`d98c82: addi a3,a3,-4`），无每轮 `slli+add` index scaling → 不命中。
- 其余 codegen rows（addressing-mode fusion / code layout / register pressure / FP lowering / scheduling / algebraic simplification 等）：无对应判别性证据（无独立地址生成序列、无 spill/reload、无分支 diamond、无常数池/跳板主导）→ 不命中。

**(c) 双 Confidence 推导式**：
- `route: 已向量化非-.S 模板代码（provenance=activate.cpp 源码直证）+ vsetivli zero,4 固定 4-lane + source 证明 MLAS_FLOAT32X4=vector_size(16) 固定类型 + 行内互斥排除完整 → High`
- `impact: 函数内局部 sample share 98.95% 已知，但采样语义缺 global-period、Sampling IP precision 未知、bound type 缺 cache/memory 细分 → Low`

多命中仲裁小段：唯一顶层 finding（`RVV Register-Group Utilization and LMUL Sizing`），无 companion / independent。Evidence mechanism layer：L3（vector/runtime configuration——LMUL/register-group 选型）。supporting：无；排除条数：8 类 class 中 5 个 exclude + 3 个 include 内逐 row 排除；推导式：1 条。

## Phase 4 — Root-cause blueprint / 根因蓝图：MlasActivation(MLAS_ACTIVATION const*, float*, float const*, unsigned long, unsigned long, unsigned long)

命中 row：`rows-vectorized-tuning.md` → `RVV Register-Group Utilization and LMUL Sizing`（Phase 3 通过 route gate）。

1. **Root cause**：
   `MlasActivationKernel`（`onnxruntime/core/mlas/lib/activate.cpp:282-355`）对输出矩阵按固定 128-bit 块处理：`MLAS_FLOAT32X4` 在 RISC-V 构建下定义为 `float __attribute__((vector_size(16)))`（`mlasi.h:2204`），即**编译期固定 4-float（16 字节）的不可伸缩向量类型**。GCC 对它的所有 lowering 都用 `vsetivli zero,4,e32,m1`（d98c72），无论硬件 VLEN 多大——在 VLEN=256（`vlenb=32`）的 spacemit-x100 上 e32,m1 的 VLMAX 是 8 lanes，实际只用 **4/8 lane = 半宽**。每轮固定成本（2×`addi` + 1×`bltu` + 每元素 1 load/1 store/2 vector ALU）按 4 个 float 摊销，loop-induction 样本占比 98.95%，正是 pattern 文件「Underutilized register-group frontier」与「Loop/config overhead dominates mid-size inputs」（`patterns/rvv_register_group_utilization.md` §Why this is slow 第 1、2 条）描述的形态：当前 LMUL/lane 数「小于合法候选边界」，每轮有效元素数偏低，循环开销按更多迭代重复发生。同一 TU 的 tail loop 已被编译器用 runtime-VL（d98c4a `vsetvli zero,a4,e32,m1,ta,ma`）生成，证明 runtime-VL lowering 在该 TU 可用——固定 4-lane 不是 toolchain 能力限制，而是源码固定宽度类型的直接映射。

2. **The fix / 修复方式**（依据 `patterns/rvv_register_group_utilization.md` §The fix 第 2 条「Raise LMUL when the budget allows」与第 5 条「Preserve autovectorization as the fallback」）：
   **纠正对象**：`MlasActivationKernel` 主循环的固定 4-lane 向量化形态（`activate.cpp` 中 `if (n >= 4) do {...MLAS_FLOAT32X4...} while`），将其改为 runtime-VL 循环。
   **修复前（现有固定宽度抽象）**：
   ```cpp
   if (n >= 4) {
       do {
           MLAS_FLOAT32X4 Vector = BiasAddition.Add(MlasLoadFloat32x4(buffer));  // vsetivli zero,4,e32,m1 + vle32.v
           MlasStoreFloat32x4(buffer, ActivationFunction.Activate(Vector));      // vmflt/vmerge + vse32.v
           buffer += 4;
           n -= 4;
       } while (n >= 4);
   }
   ```
   **修复后（RVV runtime-VL 路径，示意形态）**：
   ```cpp
   #if defined(__riscv_v)   /* MLAS RVV 路径；非 RVV 构建保留原路径 */
   size_t n = N;
   float* buffer = Buffer;
   while (n > 0) {
       size_t vl = __riscv_vsetvl_e32m4(n);        /* 或按 budget 选 m1/m2/m8；每轮覆盖 ≥8 lanes */
       vfloat32m4_t v = __riscv_vle32_v_f32m4(buffer, vl);
       v = ActivationFunction.Activate(v, vl);      /* Relu 等 lane-independent；Relu 可直接 vfmax.vf(v, 0.0f, vl) */
       __riscv_vse32_v_f32m4(buffer, v, vl);
       buffer += vl;
       n -= vl;
   }
   #else
   /* 现有固定 4-lane 路径（其他 arch fallback 不动） */
   #endif
   ```
   **适用前提**：本函数激活语义全部 lane-independent（Relu/LeakyRelu/Clip/HardSigmoid 等均为逐元素合同，无跨 lane 归约）；live budget 核算：peak live = 2 个 vector group（data+zero 常数）+ mask v0，`LMUL × peak_live_vectors ≤ 32`（`kernel-conventions.md` §2）在 m1/m2/m4/m8 全部满足；无命名 target policy 时保留 m1 fallback——即使只升到 m1 runtime-vl（8 lanes），已是当前 4-lane 的 2 倍带宽利用率。
   **Correctness contract（不可破坏）**：in-place 读改写顺序不变；runtime `vl` strip-mining 与原 `do-while(4)+tail` 对同一输入产生逐元素一致的输出；float 语义不变（如 Relu：`vfmax(0.0f, NaN)=0.0f` 与现有 `MlasMaximumFloat32x4` blend 语义一致）；非 RVV 构建路径字节级不变。
   **限制/风险**：本 kernel 为流式读改写（每元素 1 load+1 store），若在特定输入上转为 memory-bandwidth-bound，收益上界受带宽约束，LMUL 加大不线性加速；更高 LMUL 需实测确认无 vector spill/调度退化（本 kernel peak live 极小，风险低）；改动必须 `#if defined(__riscv_v)` 保护，避免破坏 NEON/SSE/LSX 等其他架构的固定宽度路径。
   **修复后预期变化的 Profile signals**：hot interval 内出现 runtime `vsetvli`（vl≥8）；`vsetivli zero,4,e32,m1` 固定 4-lane 主循环消失；`d98c82 addi a3,a3,-4` / `d98c90 addi a4,a4,16` 的每元素样本密度约减半；指令数/元素从约 1.75（7 insns/4 floats）降至约 0.88（7 insns/8 floats，m1 满宽）。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcvh_...`（RVV 1.0，VLEN=256，spacemit-x100 OoO）；build ISA=`libonnxruntime.so` = `rv64...v1p0...zvl128b...`（含 v，无 mismatch）；VLEN=256 bits（vlenb=32）；bound type=`baseline_gap: bound type`（仅 IPC=2.22，无 cache/memory counter）。

4. **收益上界**：当前 sampled event（`cpu-clock`）下函数内局部样本份额 98.95%（189/191，该 finding 的 evidence sample share 加总）。`baseline_gap: sampling metadata`（percent-type=local period、函数 workload 贡献未知）→ 不得表述为 workload 级 cycle/时间份额或 Amdahl 上界。

5. **三维路由判定**：
   - current source：compiler-generated C++ template 代码（`activate.cpp`，非 `.S`，非 JIT）——由源码直证。
   - implementation existence/reachability：不存在独立 RVV kernel/`.S`；执行路径即该编译器生成路径（jump-table dispatch 直达，样本全部落在其中）——reachable，无需 dispatch 修复。
   - function-level policy：MLAS 使用可移植 C++ template + 固定宽度 `MLAS_FLOAT32X4` 抽象，无要求独立 `.S` 载体的官方 policy；不进入 policy-backed missing `.S` 分支。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs**：
   `RVV Register-Group Utilization and LMUL Sizing` 的 Related PRs（N=13）：
   - https://github.com/opencv/opencv/pull/26318
   - https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5
   - https://github.com/opencv/opencv/pull/25586
   - https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c
   - https://github.com/torvalds/linux/commit/a894e8ed09c6c7fa239711819db83b8c050eb7b0
   - https://github.com/torvalds/linux/commit/c2a658d419246108c9bf065ec347355de5ba8a05
   - https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e
   - https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870
   - https://github.com/OpenMathLib/OpenBLAS/commit/d69be17b6ff7eea5371b03a199db9c112aa6dc4b
   - https://github.com/OpenMathLib/OpenBLAS/commit/4a12cf53ec116c06e5d74073b54a3bca6046cb17
   - https://github.com/OpenMathLib/OpenBLAS/commit/240695862984d4de845f1c42821a883946932df7
   - https://github.com/v8/v8/commit/3844339936068c529170dcb4f2aa160654d25943
   - https://github.com/vllm-project/vllm/pull/47538

## Phase 5 — Verification forecast / 验证预测：MlasActivation(MLAS_ACTIVATION const*, float*, float const*, unsigned long, unsigned long, unsigned long)

每条 primary finding 的修复对象与验证预测（唯一顶层 finding）：
- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：
  - `d98c82: addi a3,a3,-4`（91.10%）与 `d98c90: addi a4,a4,16`（7.85%）的每元素样本密度应约减半（interval 总份额不高于当前 98.95% 的按元素归一化值，且随 N 增大的每元素开销曲线变平）；
  - `d98c72: vsetivli zero,4,e32,m1,ta,ma` 固定 4-lane 主循环应消失，替换为 runtime `vsetvli`。
- **应出现侧**（锚定 `patterns/rvv_register_group_utilization.md` §Verification）：
  - 主循环出现所选 LMUL 的 runtime `vsetvli`，且候选 frontier `m1/m2/m4/m8 × unroll=1/2/4/8` 逐个记录合法性、live interval、group 对齐、EMUL、mask、spill（§Verification「候选 frontier 验证」）；
  - `objdump`/`perf annotate` 确认无因跨宽度不匹配插入的多余 `vlmul_ext`/`vlmul_trunc`（§Verification「指令验证」）；
  - 不同 VLEN/编译器上无 vector spill 回归（§Verification「register spill 验证」）；
  - 对相同输入逐元素输出一致（§Verification「正确性对照」），覆盖 N=0、小 N、main-loop 整倍数与全部 tail（runtime vl 自然覆盖）；
  - 短、中、长输入 benchmark（§Verification 末条「测试资产说明」按本仓库测试体系执行）。
- **补充说明**：Sampling IP precision 未知，消失侧按 interval 级验证（再跑同一函数 annotate，观察固定 4-lane 区间整体缩小），不要求单指令级归因。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现 | ✅ | `1/1 组；MlasActivation(MLAS_ACTIVATION const*, float*, float const*, unsigned long, unsigned long, unsigned long)` |
| 2 | Phase 1 输出要求满足（7 行 baseline + 2 个 L0 gate + bound-type gate + Sampling IP precision 行） | ✅ | 7 行结论；gap 标签：`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行 |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace；Classes scanned；顶层 finding 数；evidence 锚点；supporting/排除/推导式计数） | ✅ | 8 项 trace：asm=exclude、operator=include、string-memory=exclude、vectorized-tuning=include、codegen=include、offload=exclude、crypto=exclude、runtime-os=exclude；`Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md`；顶层 finding=1；evidence 锚点=`d98c82: addi a3,a3,-4`(91.10)、`d98c90: addi a4,a4,16`(7.85)、`d98c7e: vle32.v v1,(a4)`、`d98c88: vmerge.vim v1,v1,0,v0`；supporting=0；排除条数=8 邻居/rows；推导式=1 条 |
| 4 | Phase 4 输出要求满足（已读 pattern 文件名 + 命中 row + 引用短语 + The fix 的 before/after、correctness、风险、Profile signals 锚点 + Related PRs） | ✅ | 已读 `patterns/rvv_register_group_utilization.md`；命中 row=`RVV Register-Group Utilization and LMUL Sizing`；引用短语=`Underutilized register-group frontier`、`Loop/config overhead dominates mid-size inputs`、`Raise LMUL when the budget allows`、`Preserve autovectorization as the fallback`、`LMUL * peak_live_vectors <= 32`；fix 锚点=`vsetivli zero,4,e32,m1`→runtime `vsetvli`、`d98c82/d98c90` 每元素样本密度、insns/elem 1.75→0.88；Related PRs=13 条 URL |
| 5 | 路径合规：8 项 trace 可解释扫描集；零/多命中、关系与 evidence-mechanism layer 合规；每个 blueprint leaf 来自已通过 gate 的 row；入口模式 A 按动态份额排序；`th.v*` 未全局停扫 | ✅ | 模式 A（profile_backed）；单顶层 finding 按 98.95% 局部份额排序；leaf=`rvv_register_group_utilization.md` 来自通过 gate 的 row；L3 layer；`th.v*` 不存在故无 flavor 冻结；class 列表见项 3 |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3 引用行、出现侧标注 pattern 文件 §Verification | ✅ | 消失侧=`d98c82: addi a3,a3,-4`、`d98c90: addi a4,a4,16`、`d98c72: vsetivli zero,4,e32,m1,ta,ma`；出现侧=`patterns/rvv_register_group_utilization.md` §Verification（候选 frontier / 指令验证 / register spill 验证 / 正确性对照） |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成；无向用户追问；交付物止于证据+蓝图+The fix+验证预测 | ✅ | 全流程仅诊断输出；The fix 为蓝图形态伪代码，未编辑/未生成补丁 |

修正记录：无