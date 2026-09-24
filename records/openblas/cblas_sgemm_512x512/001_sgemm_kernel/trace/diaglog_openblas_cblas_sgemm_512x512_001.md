Functions under analysis: [sgemm_kernel]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-sgemm_kernel-annotate.txt`，4566 行，地址 0x12dd4–~0x15100 覆盖完整函数体，含 hot loop body 12f46–12f94；event=cpu-clock，182 samples，percent: local period）
- perf stat（bound/context）：已提供（`14-openblas-benchmark-riscv-cblas_sgemm_512x512.txt`，含 IPC / L1 miss / branch miss / GFLOPS）
- workload/binary/DSO/source context：已提供（`cblas_sgemm.goto` benchmark（OpenBLAS，commit 70c2410f17b60575f34dbb7eac88d16a9f485861），annotate 源行 2137–2235 显示 `__riscv_v*` intrinsic 生成代码；run 未产出 ELF（metadata warnings: No ELF executable binaries），无 debug symbols）
- readelf -A（build ISA）：缺失（详见 Phase 1）
- hardware ISA（/proc/cpuinfo 冻结快照）：已提供（`rv64imafdcvh_...zvbb_zvbc_zve32f_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_...`；含 `v`）
- vlenb：已提供（vlenb=32 → VLEN 256 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=cpu-clock；percent type=local period；同一运行窗口 430.8ms / 214 samples；函数 workload 贡献已知：perf_report sgemm_kernel self 85.05%）
- Sampling IP precision：缺失（cpu-clock 为软件定时器事件，无 precise_ip / Exact-IP / PMU skid 信息；详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `v`（RVV 1.0）暴露；`zve32f/zve64d/zvfh/zvfhmin`、`zvbb/zvbc/zvk*` 等存在；SpacemiT X100（mvendorid=0x710），out-of-order，VLEN 256（vlenb=32，冻结 metadata） |
| Build ISA | `baseline_gap: build ISA`；本 run 无 ELF 产物可执行 `readelf -A`。但 annotate 中实际执行代码全部为 RVV 1.0 `v*` mnemonic（`vle32.v`/`vfmul.vf`/`vfmacc.vf`/`vsetivli`），证明加载实现为含 `v` 的二进制 |
| Vector flavor | RVV 1.0 `v*`（`vsetivli zero,16,e32,m2,ta,ma`、`vle32.v`、`vfmacc.vf`）；zero `th.v*`；硬件支持 RVV 1.0 → 无 flavor mismatch |
| VLEN | 256 bits（vlenb=32）→ e32,m2 的 VLMAX = 16 elements；主循环 VL=16 == VLMAX，无 tail lanes |
| Bound type | compute-bound：IPC=1.035；L1_dcache_load_misses 3,907,033 / 537,044,032 = 0.728%；branch_misses 97,097 / 54,103,015 = 0.179%；cache_references/LLC NA。k-loop 每迭代 ~19.1 cycles、FMA datapath 占用 ~16 cycles（推导，见 Phase 4）→ FMA-pipe/issue bound，非 memory-bound |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=**local period**（非 global-period）；同一采样窗口；函数 workload 贡献已知（self 85.05%）→ annotate 百分比为函数内局部份额；可引用局部份额与 run-level self share，但**不得**称 workload 级 Amdahl 上界 → `baseline_gap: sampling metadata` |
| Sampling IP precision | `baseline_gap: sampling IP precision`；cpu-clock 软件事件、无 precise_ip/Exact-IP、skid 未知 → 单条指令占比只锚定 loop interval（12f46–12f94），不做 instruction-latency 归因 |

L0 baseline gate：hardware 有 `v`，执行代码全 `v*` → 无 hardware/build `v` mismatch 证据（build ISA 缺失但执行证明含 v）。无 `th.v*` → flavor gate 不触发。Bound-type gate：compute-bound 已确认（IPC ~1.0、L1 miss 0.728%）→ performance-impact confidence 不受 memory-bound 压制。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`[sgemm_kernel]`。

hot loop interval：内层 k-loop `12f46–12f94`（源行 2184–2194 `A00 = __riscv_vle32_v_f32m2(...)` → 8× `vfmacc_vf`），嵌套于 m-loop（M/16 块）`12ef4–1301e`，m-loop 再嵌套于 n-loop（N/8 块）`12dee–13202`。M=512、N=512 均为块大小整倍数，全部 tail/edge 段（M_TAIL/M_TAIL_ONE 等）0.00%（冷）。

trace anchor（最高行）：`19.78 :   12f78:  vfmacc.vf       v10,fa1,v2`（k-loop interval 内；次高 `17.58 : 12f88: vfmacc.vf v4,fa5,v2`、`14.29 : 12f80: vfmacc.vf v14,fa3,v2`）。annotate 覆盖完整（hot loop body 全覆盖）。`baseline_gap: sampling IP precision` → 单指令归因不允许，结论收敛到 interval-level mechanism（k-loop 体由 8× vfmacc.vf 的 FMA 突发 + 8× flw B 标量 load + 1× vle32.v A 向量 load + 2 addi + 1 bnez 组成）。

## Phase 3 — Pattern scan / 模式扫描：sgemm_kernel

Class selection trace（8 项）：
1. `rows-asm.md` — exclude — 当前代码来源为 intrinsic 生成（annotate 源行 `__riscv_vfmacc_vf_f32m2(...)`），非手写 `.S`；无目标 `.S` 缺失证据（RVV kernel 正在运行）。
2. `rows-operator-rvv.md` — include — FP32 matmul 微内核语义合同；hot interval 由 FP32 load + FMA + accumulator 链主导；Floating-Point Matmul row 明确接纳 already-vectorized 微内核形态。
3. `rows-string-memory.md` — exclude — 无 copy/fill/sentinel/compare/checksum 语义。
4. `rows-vectorized-tuning.md` — include — 已向量化（`v*`）且非 `.S`；评估 LMUL/unroll/operand-form/vector-state/inactive-lane/autovec-control。
5. `rows-codegen.md` — include — GEMM tile 的 cache-aware blocking、kernel-selection（Step 0c 维度 2）、register pressure、resource-aware scheduling 等 compiler 形态。
6. `rows-offload.md` — exclude — 冻结硬件快照（cpuinfo ISA）无任何矩阵引擎扩展位（无 IME/AME/P 证据）；weight-repack row 信号不存在（sgemm_itcopy+sgemm_oncopy 合计仅 5.14% 样本，kernel 内 A/B 为 packed 连续访问）。
7. `rows-crypto.md` — exclude — 非密码学 workload。
8. `rows-runtime-os.md` — exclude — 用户态 benchmark，无 kernel/RTOS 热点。

Classes scanned: `rows-operator-rvv.md`, `rows-vectorized-tuning.md`, `rows-codegen.md`

### Local performance pattern scan: `sgemm_kernel`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Floating-Point Matmul and GEMV Kernels（primary） | k-loop 12f46–12f94：8× `vfmacc.vf`(m2,16 lane) + 8× `flw`(B 标量) + 1× `vle32.v`(A 向量) 主导 ~93.4% 局部样本；FP32、无 Int8/requantization | High | Medium | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |

**三件套（primary finding）：**

**(a) 逐字 evidence 引用**（k-loop interval `12f46–12f94`；Sampling IP precision 不足 → 只作 interval 锚点）：
- `19.78 :   12f78:  vfmacc.vf       v10,fa1,v2`（源行 2190 `result67 = __riscv_vfmacc_vf_f32m2( result67, B3, A00, 16 );`）
- `17.58 :   12f88:  vfmacc.vf       v4,fa5,v2`（源行 2194 `resultEF = __riscv_vfmacc_vf_f32m2( resultEF, B7, A00, 16 );`）
- `14.29 :   12f80:  vfmacc.vf       v14,fa3,v2`（源行 2192 `resultAB = __riscv_vfmacc_vf_f32m2( resultAB, B5, A00, 16 );`）
- `4.95 :   12f66:  flw     fa5,28(t3)`（源行 2181 `FLOAT B7 = B[7];`）
- `0.00 :   12f46:  vle32.v v2,(t5)`（源行 2184 `A00 = __riscv_vle32_v_f32m2( A, 16 );`）
- `0.00 :   12ec2:  vsetivli        zero,16,e32,m2,ta,ma`（k-loop 外唯一 vsetvl 配置）
- 区间加总：flw 17.60% + vfmacc.vf 68.14% + addi 7.69% = **93.43%**（182 样本中 ~170 落在该 interval）

**(b) 互斥邻居排除**：
- `no-vectorization`：hot main loop 全 `v*`（`vle32.v`/`vfmacc.vf`）→ 排除。
- `rvv_register_group_utilization`：m2 是 16 行 m-block @VLEN256 的合法最小 LMUL（e32,m2 VLMAX=16 == VL=16；m1 使 FMA 指令数翻倍；m4@VL=16 每 vfmacc 只利用 16/32 lane、datapath 时间翻倍）；live set = 16 累加器 regs + 2 A00 = 18 regs ≤ 32、loop 内 zero spill → LMUL 在合法最优边界，排除。
- `rvv_register_budgeted_loop_unrolling`：gate 要求 backward branch / counter-update / 单一 accumulator 的 loop-carried recurrence 主导 — 实际 `bnez` 0.00%、`addi` 7.69%（含 skid）、8 条**独立** accumulator 链（无串行 recurrence 主导）→ 控制/递推不主导，不命中。
- `rvv_operand_form_selection`：`vfmacc.vf` 直接消费 `flw` 标量（rank-1 update 的自然 operand 形态），无 `vmv.v.x`/`vfmv.v.f`/load-constant-to-vector temporary → 排除。
- `rvv_vector_state_management`：hot interval 内 0 条 `vsetvl`/`vsetivli`（全函数 92 条 vsetivli 全部位于 0.00% 冷区/edge 段，配置在 12ec2 一次建立）→ 排除。
- `rvv_inactive_lane_policy`：全 `ta` + VL==VLMAX（无 tail/masked-off lane）→ 排除。
- `rvv_intrinsic_kernel_autovectorization_control`：无 compiler 插入冗余 vsetvl、无 kernel-budget 外 spill → 排除。
- `cache_aware_blocking_for_tiled_kernels`：L1 miss 0.728%；kernel 内 A/B 访问为 packed 连续偏移（`flw 0/4/.../28(t3)`、`vle32.v (t5)`），样本主导在 FMA 计算体而非 panel refill/load-stall 地址段 → 排除。
- `kernel_selection_and_runtime_specialization`：正确 RVV 微内核已被运行时分派采用并执行（v* 主导、self 85.05%）→ 排除。
- `weight_repacking_for_vectorized_risc_v_gemm`：repack 层已存在（sgemm_itcopy 3.27% + sgemm_oncopy 1.87%），开销小；kernel 消费已 pack 的 A/B → 排除。
- `resource_aware_instruction_scheduling`：row gate 要求 target-core PMU/latency-resource model 证据；`references/core-profiles.md` 仅覆盖 SpacemiT K1/X60（in-order），本 target X100 无微架构模型 → 缺 target-core 证据，不命中。
- operator 类其它 19 行：语义不匹配（无 elementwise/activation/normalization/reduction/strided/gather/packing/quantized/triangular/spatial/resampling/FFT/transform 合同；no-vectorization/precision-conversion 已单列排除）→ 整组排除。
- `rows-vectorized-tuning` 余下 1 行（maximal-LMUL byte）：非 byte processing → 排除。
- `rows-codegen` 余下 26 行：无 workaround/内联/control-flow/code-layout/addressing/GP/save-restore/FP-lowering/trap/atomic/spin/ISA-substitution/native-width/JIT/IV-strength 等 signal → 整组排除。

**(c) 双 Confidence 推导式**：
- route：intrinsic RVV FP32 GEMM 微内核 provenance + nested M/N/K 合同 + FP32 load/FMA 主导 + 无 Int8/requantization → **High**（provenance 与语义判据均直接，互斥排除完成）。
- impact：hot interval 局部样本份额 93.43%（~170/182）、kernel run-level self 85.05%、VLEN 已知（256）、bound type 已知（compute-bound）；但 `baseline_gap: sampling metadata`（local period）+ `baseline_gap: sampling IP precision` + `baseline_gap: build ISA` → **Medium**（缺项但 sample share 与采样窗口成立，故不为 Low；不得称 Amdahl 上界）。

**多命中仲裁小段**：唯一顶层命中（primary），无 companion（非 glibc 白名单）、无 supporting（被排除 rows 未通过自身 gate，不得降级为 supporting）。Evidence-mechanism layer：**L1**（vectorization / semantic dispatch — floating-point matmul/GEMV semantic row）。入口条件 A：该 finding 的 evidence sample share = k-loop interval 93.43%（局部）；Amdahl 上界不可用（sampling metadata gate），同层无其它 independent finding。

## Phase 4 — Root-cause blueprint / 根因蓝图：sgemm_kernel

命中 row：`RVV Floating-Point Matmul and GEMV Kernels`（rows-operator-rvv.md 行内判据）；已读 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md`。

1. **Root cause**：本函数是 OpenBLAS riscv64 的 FP32 GEMM 微内核，已按模式 §2 的正确方向实现——沿 M 轴向量化（A 的 16 行载入 m2 vector）、8 个独立 N 列累加器、K 轴循环、fixed-VL=16==VLMAX(e32,m2)、loop 外单次 vsetivli、无 spill。剩余瓶颈是模式文件独有内容所述 microkernel 形态的 issue 开销：依据 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md §Why this is slow`，根因即 "FMA accumulator 依赖过长" 的对偶——"增加合法独立 accumulator、减少重复转换并保持明确的 FP 累加语义"（同文件 §Why this is slow 关键句）与 §The fix 之 "增加输出行 accumulator 数量可以提高复用，但会增加寄存器压力。不得固定使用两个、四个或八个输出行"（§2）。具体机制（interval-level，标注 skid 限制）：每 k-iteration 20 条指令（8 flw + 8 vfmacc.vf + 1 vle32.v + 2 addi + 1 bnez）服务 128 个 FMA；X100 为 OoO、VLEN 256，若 FP datapath 为 256-bit f32（8 FMA/cycle），FMA floor = 16 cycles/k-iter，实测 ~19.1 cycles/k-iter（推导：0.8505×943.15M cycles / 40×64×32×512 k-iters）→ FMA datapath 占用 ~84%、flops/cycle ~13.4/16；12 条非 FMA 指令（含 8 条 B 标量 flw、loop 控制）未能完全隐藏在 FMA 突发之后。非 FMA issue 开销是剩余 ~16% 差距的载体。

2. **The fix / 修复方式**（方向与命中 pattern §The fix 一致，具体化到本 kernel）：

   **候选 A（推荐首选，风险最小）——K-loop 2× unroll + A 双缓冲**，保持 16 行 m-block / m2 / 8 累加器结构不变：
   ```
   before（当前，每 k-iter 20 条指令）:
     12f46: vle32.v v2,(t5)          ; A00 = A[k]
     flw ×8 (t3+0..28)               ; B[0..7]
     vfmacc.vf v6/v18/v12/v10/v8/v14/v16/v4, B[0..7], v2   ; ×8 独立累加器
     addi t5,t5,64 ; addi t3,t3,32 ; addi t6,t6,-1 ; bnez t6,12f46

   after（2× unroll，每 2 k-iter 38 条指令）:
     vle32.v v2,(t5)                 ; A00 = A[k]
   loop:
     flw ×8 (t3)                     ; B[0..7]  @k
     vfmacc.vf ×8, B[0..7], v2       ; 用 A[k] 计算
     vle32.v v4,64(t5)               ; 预取 A[k+1]（双缓冲）
     flw ×8 (t3+32)                  ; B[0..7]  @k+1
     vfmacc.vf ×8, B[0..7], v4       ; 用 A[k+1] 计算
     addi t5,t5,128 ; addi t3,t3,64 ; addi t6,t6,-2 ; bnez t6,loop
   ```
   前提：`__riscv_vle32_v_f32m2(A+16,16)` 预取与 8 条 `__riscv_vfmacc_vf_f32m2` 之间的 live range 合法（新增 1 个 m2 group → live set 20 regs ≤ 32，无 spill，需 disassembly 复核 group 对齐）；K 为偶数时 2× unroll 无 tail（K=512 ✓，K 奇数时 1 次标量 epilogue 或 strip-mining）。correctness：每列 K 累加顺序不变（同一列的 A[k],A[k+1] 顺序一致），FMA 收缩与 FP 舍入合同不变。风险/限制：收益来自隐藏 A-load 延迟 + 削减 addi/bnez 控制指令（每 256 FMA 40→38 条），预计有限（~5–10%）。

   **候选 B（收益更大，改动更大）——32 行 m-block + m4 + 4 列/趟共享 A**（合法 frontier 枚举见 `references/kernel-conventions.md §2`：m1/m2/m4/m8 × unroll=1/2/4/8）：
   ```
   vsetivli zero,32,e32,m4,ta,ma     ; VL=32 == VLMAX(e32,m4)@VLEN256（无浪费 lane）
   每 k-iter: 1× vle32.v v2 (m4, 32 行 A)
             4× flw (B[0..3]) ; 4× vfmacc.vf acc0..3 (m4)   ; 4 列
             （n-loop 每 4 列一趟，同 A 不重读；或第二趟复用 v2）
   live set: 4 acc × m4(4regs) = 16 + A m4 = 4 → 20 regs ≤ 32
   ```
   每 128 FMA 指令数 20 → 12（每 FMA 0.156 → 0.094，−40%），FMA floor 仍 16 cycles/k-iter；再叠加 2× K-unroll 时每 256 FMA 22 条（live set 24 regs）。前提：m-loop 改 M/32、N 按 4 列 strip、C epilogue 改 4× m4 load/alpha-FMA/store（注意 epilogue 瞬时寄存器压力，防 spill）；M 非 32 倍数走现有 edge kernel。correctness：每列 K 累加顺序不变、packed A/B 布局与 ldc/alpha/beta 合同不变。风险/限制：需重排 n-loop 块结构并 A/B benchmark；不得默认固定 4 或 8 个输出列（pattern 明确 "不得固定使用两个、四个或八个输出行"），必须按实机数据定。

   修复后预期 Profile signals：k-loop 每迭代 cycles 从 ~19.1 向 16-cycle FMA floor 收敛；flw/addi/bnez 行局部样本显著缩小；vfmacc.vf 仍保持主导（FMA 计算体不变）；loop 内不出现新增 vsetvli、不出现 vector spill。

3. **Baseline facts 回填**：hardware ISA = rv64imafdcvh_...（含 `v`、zve64d、zvfh 等，SpacemiT X100 OoO）；build ISA = `baseline_gap: build ISA`（执行代码证明含 v）；VLEN = 256 bits；bound type = compute-bound（IPC 1.035 / L1 miss 0.728%）。

4. **收益上界**：入口条件 A →「当前 sampled event 下的局部样本份额」= k-loop interval 93.43%（182 样本中 ~170）；kernel run-level self 85.05%（perf_report 同一窗口）。`baseline_gap: sampling metadata`（local period）→ 不得表述为 workload 级 Amdahl 上界。

5. **三维路由判定**：
   - `current source`：compiler-generated（intrinsic 展开）代码 —— annotate 源行 2184–2194 的 `__riscv_vle32_v_f32m2`/`__riscv_vfmacc_vf_f32m2` + disassembly `v*`（非 `.S`）。
   - `implementation existence/reachability`：RVV 微内核存在且被采用（perf_report self 85.05%；annotate 全 v*）→ kernel-selection 排除。
   - `function-level policy`：OpenBLAS RISC-V backend 以 intrinsic kernel 为当前实现；policy-backed missing `.S` 四证不成立（kernel 已向量化）→ 该分支不适用。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs**：`patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Related PRs 表，共 37 条 URL（按项目分组；同一 pattern 内 URL 已去重）：
   - OpenMathLib/OpenBLAS（7）：https://github.com/OpenMathLib/OpenBLAS/commit/0a967797a15617239523053633bf14be7895b25a 、https://github.com/OpenMathLib/OpenBLAS/commit/0acb60aab3c0134e879a68292904d8346dcd50ef 、https://github.com/OpenMathLib/OpenBLAS/commit/1cc377ef61d498b75c852aa4b9b042fe9422c347 、https://github.com/OpenMathLib/OpenBLAS/commit/2d82d144e2791e37d7a314237b638d85b156a2ec 、https://github.com/OpenMathLib/OpenBLAS/commit/809e1cba8f1f3f89972581e8b82f2ec52e51eadb 、https://github.com/OpenMathLib/OpenBLAS/commit/376d3a138faa0a0fe483a8fa8d4fa1ab0d395acf 、https://github.com/OpenMathLib/OpenBLAS/commit/2ae019161a85333a35018b517d4b34474a7694e9
   - oneDNN（9 commits + 13 PRs）：https://github.com/uxlfoundation/oneDNN/commit/d6f82a2d0d0db41e6daaf20fbb4fd352843aac64 、https://github.com/uxlfoundation/oneDNN/commit/3bac96b8bc1fc9c348c986f38f65285693943d2f 、https://github.com/uxlfoundation/oneDNN/commit/8b48a77091062ce78959ccb96e43a8ee4e97022d 、https://github.com/uxlfoundation/oneDNN/commit/8c52facbe61845d86062c76280b4bc515160c03e 、https://github.com/uxlfoundation/oneDNN/commit/b73fc3172d3e3230cf24ae29cbb6a07a09507a43 、https://github.com/uxlfoundation/oneDNN/commit/bd984d09dc5985a19fb427ac46d19d2cbd5558dd 、https://github.com/uxlfoundation/oneDNN/commit/d2a44b9b855706a0df33f9b6d4fb84f5420fdeaf 、https://github.com/uxlfoundation/oneDNN/commit/d6107ddb8be72041dade165a233c0de69f7a1387 、https://github.com/uxlfoundation/oneDNN/commit/fe04323ab0b4bba79ee60109fc391bb36052e43c 、https://github.com/uxlfoundation/oneDNN/pull/4410 、https://github.com/uxlfoundation/oneDNN/pull/4414 、https://github.com/uxlfoundation/oneDNN/pull/4545 、https://github.com/uxlfoundation/oneDNN/pull/4620 、https://github.com/uxlfoundation/oneDNN/pull/4770 、https://github.com/uxlfoundation/oneDNN/pull/4824 、https://github.com/uxlfoundation/oneDNN/pull/4840 、https://github.com/uxlfoundation/oneDNN/pull/4850 、https://github.com/uxlfoundation/oneDNN/pull/4945 、https://github.com/uxlfoundation/oneDNN/pull/5157 、https://github.com/uxlfoundation/oneDNN/pull/5294 、https://github.com/uxlfoundation/oneDNN/pull/5403 、https://github.com/uxlfoundation/oneDNN/pull/5405
   - llama.cpp（6）：https://github.com/ggml-org/llama.cpp/pull/17318 、https://github.com/ggml-org/llama.cpp/pull/17448 、https://github.com/ggml-org/llama.cpp/pull/17314 、https://github.com/ggml-org/llama.cpp/pull/17161 、https://github.com/ggml-org/llama.cpp/pull/18199 、https://github.com/ggml-org/llama.cpp/pull/20627
   - MNN（1）：https://github.com/alibaba/MNN/pull/4426
   - vLLM（1）：https://github.com/vllm-project/vllm/pull/44324

## Phase 5 — Verification forecast / 验证预测：sgemm_kernel

修复对象：sgemm_kernel 的 k-loop 微内核调度（候选 A：K 2× unroll + A 双缓冲；候选 B：32 行/m4 tile 重构）。

- **应消失/缩小**（锚定 Phase 3(a) 引用行）：`4.95 : 12f66: flw fa5,28(t3)`、`3.85 : 12f62: flw fa4,24(t3)`、`7.69 : 12f8c: addi t5,t5,64` 及 `12f94: bnez t6,12f46`（当前 0.00%，2× unroll 后迭代数减半）的局部样本应显著缩小；k-loop 每迭代 cycle 从 ~19.1 向 FMA floor（16 cycles/k-iter，候选 B 配 m4 后同样 16）收敛；`12f78/12f80/12f88: vfmacc.vf` 保持主导（FMA 计算体不变）。
- **应出现**（锚定 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md §Verification`）：正确 FP kernel 可达、scalar FMA/loop-control share 下降、预期 `vfmacc*`/vector load 仍在、cycles/FLOP 或 throughput 改善；失败判据：转换/packing 成为新瓶颈、register spill 增加、短 shape crossover 退化、edge kernel 仍回退 scalar、数值误差超出合同。候选 B 时 disassembly 应出现 `vsetivli zero,32,e32,m4,ta,ma`（VL=32==VLMAX）且 loop 内无新增 vsetvl/spill。
- 验证方法：目标 `-march`（含 `v`，VLEN 256）+ 相同优化级别重建；对 M=512/N=512/K=512（main-loop 整倍数）与 M%16/N%8 edge shapes 分别跑 `perf stat -e cycles,instructions` 与 `perf annotate`；FP 累加顺序/误差对照 reference（每列求和顺序不变）。入口条件 A：按收益上界顺序验证（本 finding 唯一）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status（Anchor 载荷） |
|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现 | ✅（1/1 组；`sgemm_kernel`） |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + L0 gate + bound-type gate + Sampling IP precision） | ✅（7 行结论项；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；已含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足 | ✅（8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md, rows-vectorized-tuning.md, rows-codegen.md`；顶层 finding=1（RVV Floating-Point Matmul and GEMV Kernels）；evidence 锚点：`19.78 : 12f78: vfmacc.vf v10,fa1,v2` / `17.58 : 12f88: vfmacc.vf v4,fa5,v2` / `14.29 : 12f80: vfmacc.vf v14,fa3,v2` / `4.95 : 12f66: flw fa5,28(t3)`；supporting=0；排除条数：class 级 5（asm/string-memory/offload/crypto/runtime-os）+ 行级 operator-rvv 19 + vectorized-tuning 6 + codegen 26；推导式=1（route High / impact Medium）） |
| 4 | Phase 4 输出要求满足 | ✅（已读 pattern：`patterns/rvv_floating_point_matmul_and_gemv_kernels.md`；命中 row：RVV Floating-Point Matmul and GEMV Kernels；引用短语首词：§Why this is slow "主要根因是浮点 microkernel 沿错误轴向量化..."、§2 "增加输出行 accumulator 数量可以提高复用，但会增加寄存器压力。不得固定使用两个、四个或八个输出行"；The fix 含 before/after 伪代码（候选 A/B）、correctness、风险、预期 Profile signals；非 missing-.S 分支；Related PRs：37 条 URL） |
| 5 | 路径合规（8 项 trace 可解释扫描集；唯一 primary、无 companion/supporting；L1 层归属；入口模式 A 按动态份额排序） | ✅（模式 A：profile-backed；路径：`rows-operator-rvv.md` → `patterns/rvv_floating_point_matmul_and_gemv_kernels.md`；class 列表见 #3） |
| 6 | Phase 5 两侧锚定 | ✅（消失侧：`12f66: flw fa5,28(t3)`、`12f62: flw fa4,24(t3)`、`12f8c: addi t5,t5,64`、`12f94: bnez t6,12f46`；出现侧：`patterns/rvv_floating_point_matmul_and_gemv_kernels.md §Verification`） |
| 7 | 契约边界合规（无实施询问/代码修改/补丁生成；无向用户追问；止于证据+蓝图+The fix+验证预测） | ✅（本输出仅诊断交付物） |

修正记录：无