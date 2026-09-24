Functions under analysis: [cpu_attention::AttentionMainLoop<cpu_attention::AttentionImpl<(cpu_attention::ISA)5, float, 64l, float> >::operator()(cpu_attention::AttentionInput const*) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`002-cpu_attention：：AttentionMainLoop＜...ISA)5, float, 64l, float＞...-annotate.txt`，3783 行，覆盖 hot loop body 全部三段：QK^T GEMM、P·V GEMM、reduce_splits 与 epilogue；header 标注 `cpu-clock (74456 samples, percent: local period)`）
- perf stat（可选 bound/context）：已提供（`13-vllm-cpu-validation-riscv-vllm-bench-throughput.txt`；含 duration_time、cpu_cycle、instruction、L1_dcache_loads/misses、LLC_loads/misses、branches 等）
- workload/binary/DSO/source context：已提供（vllm `v0.24.0` commit `ee0da84ab9e04ac7610e28580af62c365e898389`，`vllm_C.so-elf-A/-elf-h` ELF RISC-V DYN RVC double-float ABI；源码 `csrc/cpu/cpu_attn_rvv.hpp` `gemm_micro_rvv_fma_Mx8_Ku4` / `TileGemmRVV` / `AttentionImpl<ISA::RVV>`，ISA 枚举 `csrc/cpu/cpu_attn_impl.hpp` `enum class ISA { AMX, VEC, VEC16, NEON, VXE, RVV, VSX }`，ISA=5 即 `ISA::RVV`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zfhmin1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvfh1p0_zvfhmin1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`）
- hardware ISA（/proc/cpuinfo 或 riscv_hwprobe）：已提供（metadata cpuinfo：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`）
- `vlenb`：已提供（metadata vector：`RVV 1.0`, `vlen_bits=128`, `vlenb=16`）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（annotate header 标注 `cpu-clock (74456 samples, percent: local period)`；函数级 workload 贡献未知；`precise_ip`/skid 能力未知 → `baseline_gap: sampling metadata`（workload 级贡献项））
- Sampling IP precision：缺失（`precise_ip` 与 PMU skid 能力未提供 → `baseline_gap: sampling IP precision`；本报告所有单行占比仅锚定所属 basic block / loop interval）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：SG2044 / XuanTie C920v2（T-Head，mvendorid=0x5b7），`rv64imafdcv_...` 含 `v`（RVV 1.0）、`zvl128b` 隐含、`zicbom/zicboz/zicbop`（zicbop 未见，zicbom/zicboz 有）、`zihintpause`、`zba/zbb/zbc/zbs`、`zawrs`、`zfa/zfh/zvfh*`；硬件支持 RVV 1.0 |
| Build ISA | 已提供：`vllm_C.so` `Tag_RISCV_arch = rv64i2p1_..._v1p0_..._zvl128b1p0...`；含 `v`（RVV 1.0）与 `zvl128b`；与 hardware 匹配（无 L0 mismatch） |
| Vector flavor | annotate 为 RVV 1.0 标准 `v*` mnemonic（`vfmacc.vf`、`vle32.v`、`vse32.v`、`vsetivli`、`vmv2r.v`），无 `th.v*`；无 flavor mismatch |
| VLEN | 已提供：`vlenb=16` → VLEN=128 bits；SEW=32 时 m1=4 lanes，LMUL_256=m2=8×FP32 |
| Bound type | `perf stat`：L1_dcache_load_miss_rate=22.204%，LLC_load_miss_rate=39.681%，cache_miss_rate=100.000%（cache_references≈cache_misses），branch_miss_rate=0.204%；IPC≈0.381（306.4e12 instr / 804.3e12 cycles）；workload 明确 memory/cache-bound |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=local period；同一运行窗口（单次 run）；该函数占整个 workload 的贡献未知 → 收益上界只能表述为「当前 sampled event 下函数内局部样本份额」，不得称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`） |
| Sampling IP precision | 未知（`precise_ip`/Exact-IP/PMU skid 能力未提供 → `baseline_gap: sampling IP precision`）；单指令占比只锚定 basic block / loop interval |

L0 baseline gate：#1 hardware 有 `v` 且 build 有 `v`（`v1p0` + `zvl128b`）→ 无 mismatch；#2 annotate 为 RVV 1.0 `v*`，非 `th.v*`，硬件支持 RVV 1.0 → flavor gate 通过，标准 RVV 依赖的 route 不冻结。Bound-type gate：profile 为 memory/cache-bound → 本地 compute-vectorization fix 的 performance-impact confidence 下调；cache-aware-blocking row 按行内判据单独评估（见 Phase 3）。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个函数）。hot loop 锚点（覆盖完整）：
- **QK^T GEMM interval**（`paged_attention` QK 阶段，gemm_micro_rvv_fma_Mx8_Ku4<8,float>，K=block 内 token 组）最高行：`10.74 :   254340: flw     fa2,0(t1)`（喂给 `vfmacc.vf` 的 A-slice 标量 load；地址区间约 0x2533a8–0x2543d6）
- **P·V GEMM interval**（`paged_attention` PV 阶段，gemm_micro_rvv_fma_Mx8_Ku4<4,float>，K=token_num）最高行：`10.54 :   254b6a: flw     fa2,0(t6)`（地址区间约 0x254b3e–0x254c0e）
- **reduce_splits spin interval**：`0.03 :   253392: jal     1bf3c0 <sched_yield@plt>` + `0.02 :   25339a: beqz    a4,253392`（轮询 flag 的 tight poll）
- 其它次要热行：`0.57 :   253cbe: addi    t4,t4,64`（final_output 写回循环）、`0.37 :   253b42: addi    a4,a4,64`（partial_output）、`0.30 :   2533f2: flw     fa3,256(a7)`（M=2 micro-kernel）、copy_q_heads_tile 段（0.10–0.22）。

Sampling IP precision 不足（`baseline_gap: sampling IP precision`）：所有单行占比只锚定 loop interval，不承担单条指令 latency/cost 归因；根因以区间聚合 + def-use/source/PMU 交叉证据支撑。

## Phase 3 — Pattern scan / 模式扫描：cpu_attention::AttentionMainLoop<cpu_attention::AttentionImpl<(cpu_attention::ISA)5, float, 64l, float> >::operator()(cpu_attention::AttentionInput const*)

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — 当前代码来源为 compiler/intrinsic-generated（`csrc/cpu/cpu_attn_rvv.hpp` RVVI() intrinsic 模板，disassembly 无 `.S` 标记；无手写 assembly provenance，无缺失 `.S` 的 dispatch-slot/四证证据）
2. `rows-operator-rvv.md` — **include** — 热点是 intrinsic 生成的**已向量化** FP32 GEMM microkernel（QK^T 与 P·V 双段），浮点 matmul 语义合同明确（`vfmacc.vf` + FP32 load），row 明确接纳 already-vectorized 的 matmul 失效形态
3. `rows-string-memory.md` — **exclude** — 无 copy/fill/sentinel/compare/checksum/glibc 接线信号；热点为浮点计算
4. `rows-vectorized-tuning.md` — **include** — 完整 annotate 已有 `v*` 且非手写 `.S`；逐行评估 operand-form / register-group / unroll / vector-state / inactive-lane / autovec / maximal-LMUL 候选
5. `rows-codegen.md` — **include** — compiler-generated；需评估 kernel-selection、cache-aware-blocking、spin-wait backoff、resource-aware scheduling、register pressure 等行；memory-bound 信号触发 cache-blocking 行检查
6. `rows-offload.md` — **exclude** — 目标无矩阵引擎（无 AME/IME/P 扩展证据），无独立权重重排层（KV cache 无法跨调用摊薄 repack），无跨 VLEN 可移植缺失证据（build 与硬件均 zvl128b）
7. `rows-crypto.md` — **exclude** — 非密码学原语
8. `rows-runtime-os.md` — **exclude** — 用户态 vLLM 推理 kernel，无 timer/ISR/privileged CSR 信号

Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md

### Local performance pattern scan: cpu_attention::AttentionMainLoop<...ISA)5, float, 64l, float...>

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Floating-Point Matmul and GEMV Kernels（primary） | QK^T/P·V microkernel 的 FP32 load + `vfmacc.vf` + 寄存器复用形态主导热点区间；无量化合同 | High | Medium（memory-bound 竞争 + sampling metadata 缺 workload 份额） | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |
| Bounded Spin-Wait Backoff（independent, 次要） | reduce_splits 内轮询 flag 的 tight poll 循环（lbu→beqz→sched_yield，无退避），0.03%+0.02% 局部份额 | High（该地址区间 gate 成立） | Low（sample 占比极小） | `patterns/bounded_spin_wait_backoff.md` |

**Finding 1（primary）— RVV Floating-Point Matmul and GEMV Kernels**

**(a) 逐字 evidence 引用**（hot interval 归属：QK^T GEMM interval，地址约 0x2533a8–0x2543d6，`gemm_micro_rvv_fma_Mx8_Ku4<8,float>` K-loop body）：
```
10.74 :   254340: flw     fa2,0(t1)
 7.66 :   254356: vfmacc.vf       v8,fa2,v14
 8.67 :   25437c: vfmacc.vf       v8,fa2,v10
10.30 :   2543a2: vfmacc.vf       v8,fa2,v14
 0.11 :   25435e: vfmacc.vf       v4,fa4,v14
 0.12 :   2543aa: vfmacc.vf       v4,fa4,v14
```
（第二处 PV GEMM interval，地址约 0x254b3e–0x254c0e，`gemm_micro_rvv_fma_Mx8_Ku4<4,float>`）：
```
10.54 :   254b6a: flw     fa2,0(t6)
 7.28 :   254b82: vfmacc.vf       v2,fa2,v12
 9.06 :   254baa: vfmacc.vf       v2,fa2,v10
10.44 :   254bd2: vfmacc.vf       v2,fa2,v12
```
（A-slice 相邻标量 load 行，同 interval）：`0.12 : 25438c: flw fa2,8(t1)`、`0.11 : 2543ba: flw fa4,524(t1)`、`0.10 : 2543c4: vfmacc.vf v8,fa2,v10`、`0.09 : 254bb2: vfmacc.vf v6,fa4,v10`、`0.09 : 254bf6: vfmacc.vf v2,fa2,v10`；B-slice 向量 load：`0.07 : 254378: vle32.v v14,(t4)`、`0.08 : 254ba6: vle32.v v12,(s1)`。
区间级机制（Sampling IP precision 不足，不逐指令归因）：A-slice（QK 阶段=paged K-cache K 行切片；PV 阶段=prob 行）元素逐标量 `flw` 载入并广播给 `vfmacc.vf`，B-slice（QK 阶段=Q tile；PV 阶段=V-cache 行）用 `vle32.v` 整向量载入；K-loop 每步 32 次标量 load + 16 次 vfmacc.vf（M=8）。

**(b) 互斥邻居排除**
- `rvv_quantized_matmul_and_requantization_kernels`（rows-operator-rvv 行 26）：无 Int8/zero-point/widening-MAC/requantization 合同；`kv_cache_t=float`、全部指令为 FP32 `vfmacc.vf`/`flw`/`vle32.v`（引用行 254356/254340）→ 排除。
- `cache_aware_blocking_for_tiled_kernels`（rows-codegen 行 14）：该行要求 sample 主导在 packing/copy、packed-panel load 或 refill 暴露的 load-stall **地址段**，且 source 能确定 tile 复用顺序并有尺寸拐点证据。本 profile 的 sample 主导在**已正确到达的 RVV microkernel 计算体**（vfmacc.vf 与 A/B-slice load 属同一 k-loop 计算区间，非独立 packing/refill 段）；调度器已按可用 L2（`get_available_l2_size()` + `calcu_default_tile_size`/`calcu_tile_size_with_constant_q`）计算 tile size，无「工作集跨 cache 份额」的重复性能拐点数据；perf stat 缺 cache refill 随尺寸扫描的拐点证据 → 该行不命中（bound-type 零命中路径按「cache-aware-blocking 证据不满足」处理，第一杠杆仍在计算体数据访问形态）。
- `weight_repacking_for_vectorized_risc_v_gemm`（rows-offload）：KV cache 由 `reshape_and_cache` 动态增量写入、block 布局是 attention 语义合同，不存在跨调用可摊薄的一次性 repack 层；sample 不在独立 repack 地址段 → 排除。
- `rvv_register_group_utilization`（rows-vectorized-tuning 行 17）：LMUL_256（VLEN=128 下 m2）是源码模板常量且已达该 kernel 的合法边界（M=8 时 8 acc×m2 + B 载入，32 个向量寄存器预算见 kernel-conventions §2）；disassembly 无 `vlmul_ext/trunc`、无向量 spill/reload；hot interval 无 vsetvli churn（vsetivli 提升在外层）→ 排除（LMUL/live-set 不是直接根因）。
- `rvv_operand_form_selection`（行 15）：`vfmacc.vf` 是 A 标量广播 × B 向量的**原生 RVV operand form**（源码注释明确「RVV has no lane-indexed FMA; we load A elements as scalars and use vfmacc_vf」）；无临时 `vmv.v.x`/`vfmv.v.f` 仅被单条 op 消费的形态；同一 broadcast 被 8 个 accumulator 复用属健康形态 → 排除。
- `rvv_register_budgeted_loop_unrolling`（行 16）：K-loop 无 loop-carried accumulator recurrence（每 k 步 16 个独立 vfmacc 累加 8 个独立 acc），已 4×K-unroll；无「单 accumulator 串行 recurrence 主导」证据 → 排除。
- `resource_aware_instruction_scheduling`（rows-codegen 行 25）：缺 target-core latency/resource model 与依赖/barrier 交叉证据（`baseline_gap: sampling IP precision` 且无 PMU 调度证据），且 workload 明确 memory/cache-bound（load-use stall 由 cache refill 主导而非调度器排错）→ 排除（机制并入 primary 的 load-use/cache 解释）。
- `kernel_selection_and_runtime_specialization`（行 13）：ISA dispatch（`cpu_attn.cpp` 运行时选中 `ISA::RVV`）已到达正确实现，hot sample 全部落在 RVV kernel 计算体，无 fallback/generic 热点 → 排除。

**(c) 双 Confidence 推导式**
- route: 已向量化 FP32 GEMM microkernel + 浮点语义合同（vfmacc.vf，无量化）+ 地址直接归属 QK/PV k-loop + 互斥排除（量化/repack/register-group/operand-form/unroll/scheduling/kernel-selection）→ **High**
- impact: sample share 成立（QK/PV 双 interval 合计约 60–75% 局部份额）但 bound type=memory/cache-bound 为竞争瓶颈、VLEN 已知（128）但 sampling metadata 缺 workload 级份额 → **Medium**

**Finding 2（independent, 次要）— Bounded Spin-Wait Backoff**

**(a) 逐字 evidence 引用**（interval 归属：`reduce_splits` flag 轮询，地址约 0x253384–0x25339c）：
```
0.03 :   253392: jal     1bf3c0 <sched_yield@plt>
0.02 :   25339a: beqz    a4,253392
0.00 :   253396: lbu     a4,0(s1)
```
（另有 reduce_splits 的 `fence r,rw` 后重读 flag 结构；guard_counter spin 见 0x254ee2–0x254efc 约 0.1–0.2%）

**(b) 互斥邻居排除**
- `redundant_synchronization_elimination_in_uncontended_contexts`（方向相反）：此处存在真实并发写者——partial-output 线程通过 `sb 1,0(flag)`（0x253b62）推进 flag，同步必要，不能整体删除 → 排除。
- `hardware_atomic_operations_for_synchronization`：flag 是 `volatile bool` 普通 load（`lbu`）+ 独立 `fence`，非 atomic helper 未 lowering 问题 → 排除。
- `isa_extension_specific_instruction_substitution`：缺的不仅是 `pause` hint——循环每次重读同一 cache line 并调用 `sched_yield` 系统调用（延迟与干扰问题），属整段轮询频率/退避 progression → 排除（归本行）。

**(c) 双 Confidence 推导式**
- route: 轮询 load+backward branch 紧密循环、producer 真实推进、无有界/递增退避（每次直接 sched_yield，sched_yield 语义上「让出」但无退避增长）→ **High**（仅限该地址区间）
- impact: 局部样本份额极小（约 0.05–0.3% 合计，含 guard_counter spin 约 0.1–0.2%），`baseline_gap: sampling IP precision` → **Low**

### 多候选仲裁小段
- **primary**：`rvv_floating_point_matmul_and_gemv_kernels`（L1/L2 层：浮点 matmul 语义 + microkernel 数据访问形态）。QK^T 与 P·V 两个 GEMM interval 是**同一 kernel 模板（gemm_micro_rvv_fma_Mx8_Ku4）的两个调用点**，机制相同（A-slice 标量 gather load → vfmacc.vf 广播），sample 分账于两个不相交地址段，合并为同一 primary 的区间证据。
- **supporting**：无（register-group/operand-form 等已按互斥排除，非 gate 成立的低层命中）。
- **independent**：`bounded_spin_wait_backoff`（地址不相交 0x253392，机制=同步等待策略，修复对象与验证方法独立）。
- **未命中且按 L2/L0 归因排除**：cache-aware-blocking（L2）证据不满足（无 tile-residency 拐点、sample 属计算体）；kernel-selection（L0）不成立（正确实现已到达）。
- 收益上界排序（入口条件 A）：Finding 1（primary，局部份额约 60–75%）> Finding 2（independent，约 0.1–0.3%）。两者机制与地址分账，非同一调用链，分别验证。

## Phase 4 — Root-cause blueprint / 根因蓝图：cpu_attention::AttentionMainLoop<cpu_attention::AttentionImpl<(cpu_attention::ISA)5, float, 64l, float> >::operator()(cpu_attention::AttentionInput const*)

### Finding 1（primary）— RVV Floating-Point Matmul and GEMV Kernels（对应 Phase 3 Finding 1，row `rows-operator-rvv.md` 行 25）

1. **Root cause**：FP32 GEMM microkernel 的 A-slice 访问形态与 cache 布局不匹配。微内核把 **A 矩阵元素逐标量 gather**（QK 阶段 A=paged K-cache 的 K 行切片，`lda=32 floats`，即每行 1KB 步距；PV 阶段 A=prob tile 行，`lda=32`）载入后经 `vfmacc.vf` 广播乘加，B-slice 用 `vle32.v` 向量载入（QK 阶段 B=Q tile 行 `ldb=8`；PV 阶段 B=V-cache 行）。依据 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Why this is slow：「浮点 microkernel 沿错误轴向量化、FMA accumulator 依赖过长、packing 与消费合同不匹配」是主要根因；`Related PRs` 表中 vLLM 自己的 WNA16 RVV micro-GEMM PR（#44324）即属于本 pattern 的 FP load/accumulator 修复方向。结合 `kernel-conventions.md` §1（accumulator loop-carried dependency 需多 accumulator + unroll 隐藏 latency）与 §2（m8 FMA 需 accumulator 与当前输入两个 group）解释：本 kernel 已用 8 个独立 accumulator + 4×K-unroll 解除 loop-carried 依赖，但 A-slice 每 k 步 8 行 × 每行 1KB 步距的**标量 load-use 链**（flw→vfmacc.vf 立即消费，间隔仅 2 条指令）成为新的 dominant stall：32 次 `flw` 各命中 paged block 内不同 cache line（32-token block × head_dim 64 float 交错的 layout），L1 miss 时 load-use 延迟无法被 OoO 完全隐藏；`perf stat` 佐证 L1_dcache_load_miss_rate=22.204%、LLC_load_miss_rate=39.681%。
2. **The fix / 修复方式**：
   - 纠正对象：`gemm_micro_rvv_fma_Mx8_Ku4` 的 A-slice 载入方式（QK 与 PV 两处调用共用）与 KV-slice 的 tile 驻留策略。
   - 修复前（现状）：
     ```cpp
     // A: [M x K] row-major, lda=32 (paged KV / prob tile), B: [K x 8]
     for (; k + 3 < K; k += 4) {
       fixed_fp32x8_t b = load_row8_B_as_f32<kv_cache_t>(B + (k+0) * ldb);
       acc0 = RVVI(__riscv_vfmacc_vf_f32, LMUL_256)(acc0, *(a0 + k + 0), b, 8); // 每行标量
       acc1 = RVVI(__riscv_vfmacc_vf_f32, LMUL_256)(acc1, *(a1 + k + 0), b, 8);
       // ... 8 行 × 4 k 步：32 flw + 16 vfmacc.vf
     }
     ```
   - 修复后（方向 A — A-slice 向量化载入，保持 M=8/lda 不变）：对每行 4 连续元素用 `vle32` 一次载入（VLEN=128 下 4×e32=m1，m2 可载 8 元素），再以 `vfmacc.vv`（向量×向量）与 B-slice 向量累加；将标量 load-use 链改为向量 load 链，load 指令数降 4×，且使 A 元素按 cache line 连续读取：
     ```cpp
     for (; k + 3 < K; k += 4) {
       auto b0 = load_row8_B_as_f32<kv_cache_t>(B + (k+0) * ldb);   // vle32.v (8)
       auto a0v = RVVI(__riscv_vle32_v_f32, LMUL_128)(a0 + k, 4);    // 每行 4 元素向量
       acc0 = RVVI(__riscv_vfmacc_vv_f32, LMUL_256)(acc0, a0v /*broadcast组合*/, b0, 8);
       // ...
     }
     ```
     若保留 `vfmacc.vf`（改 layout 不可行时），则应把 A 标量 load 提前 1–2 个 k 步预取（software pipelining），切断 flw→vfmacc 紧邻依赖。
   - 方向 B — K-slice 驻留与布局：`AttentionImpl<ISA::RVV>::k_cache_token_group_stride` 返回 `BlockSizeAlignment=32` 而非实际 block_size；decode-only 批（q_token_num=1）下 K 行切片（head_dim×block_size=64×32 float=8KB/行）应在 tile 复用窗口内显式驻留 L1/L2（结合 `calcu_default_tile_size` 已按 `get_available_l2_size()` 算 tile），并为 K/V block 指针加 `zicbop` 软件 prefetch（CPU 支持 zicbom/zicboz，需确认 zicbop 是否暴露；不可用则用 `__builtin_prefetch`）。这是 cache 布局层的配合改动，不改变 paged KV 合同。
   - 适用前提：head_dim=64、block_size=32、VLEN=128、FP32（`kv_cache_t=float`）；`c10::Half`/`BFloat16` 路径（`load_row8_B_as_f32` 特化）同样受益但需按 `zvfh/zvfbfmin` gate 调整。
   - 不可破坏的 correctness contract：paged KV block 布局（`reshape_and_cache` 的 strided-store 布局与 `k/v_cache_*_stride` 常量）、causal/sliding-window/softcap/alibi 掩码语义、split-reduction 的 flag/fence 协议、FP32 累加顺序语义（若改 `vfmacc.vf`→`vfmacc.vv` 仅改变 load 形态，累加顺序不变；若软件流水改变 FMA 顺序，须按 `kernel-conventions.md` §1 验证 FP reassociation 误差）。
   - 限制/风险：VLEN=256 的 `zvl256b` 目标映射不同（m1）；`vfmacc.vv` 需额外向量寄存器（VMV 预算：M=8 时 8×m2 acc + B 载入 + A 载入需 ≤32 regs，按 `kernel-conventions.md` §2 逐 instruction 核算）；K-slice 驻留改动影响 `AttentionScheduler` 的 tile-size 计算边界。
   - 预期 Profile signals：`flw` A-slice 行（254340/254366/25438c/2543b2 及 254b6a/254b92/254bba/254be2）占比显著下降；`vfmacc.vf` 行（254356/25437c/2543a2 等）随之下降或转为 `vfmacc.vv`；L1_dcache_load_miss_rate（22.2%）与 LLC_load_miss_rate（39.7%）下降；microkernel interval 总样本份额下降。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zvl128b...`（RVV 1.0）；build ISA=`rv64i2p1_..._v1p0_..._zvl128b1p0`；VLEN=128（vlenb=16）；bound type=memory/cache-bound（L1 miss 22.2%、LLC miss 39.7%、IPC≈0.381）。
4. **收益上界**：当前 sampled event（cpu-clock）下函数内局部样本份额合计约 **60–75%**（QK interval 约 32–38% + PV interval 约 30–37%，含 vfmacc.vf/flw/vle32.v 的 k-loop 计算体）。仅表述为局部份额；`baseline_gap: sampling metadata`（函数级 workload 贡献未知）→ 不得称 workload 级 Amdahl 上界。
5. **三维路由判定**：
   - current source：compiler-generated（vLLM C++ RVVI() intrinsic 模板，`csrc/cpu/cpu_attn_rvv.hpp`，非 `.S`，非 JIT）
   - implementation existence/reachability：存在且可达——`cpu_attn.cpp` 运行时 dispatch `ISA::RVV` 命中本实现；disassembly 无 fallback/多版本分派热点（调用链 `AttentionMainLoop → AttentionImpl<ISA::RVV>::execute_attention → paged_attention<TileGemmRVV> → gemm_macro_rvv_fma_Mx8_Ku4 → gemm_micro...`）
   - function-level policy：该函数是 vLLM CPU attention 的 OpenMP 并行主循环（`#pragma omp parallel for schedule(static,1)`），热点路径全部落于已选中的 RVV kernel；无独立 `.S` policy/存在性四证需求（不走 missing-assembly 分支）
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。
7. **Related PRs 小节**：
   - 按 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Related PRs 表（节选同项目与核心同类实现）：
   - `Related PRs：17 条 URL`：https://github.com/vllm-project/vllm/pull/44324 ；https://github.com/OpenMathLib/OpenBLAS/commit/0a967797a15617239523053633bf14be7895b25a ；https://github.com/OpenMathLib/OpenBLAS/commit/0acb60aab3c0134e879a68292904d8346dcd50ef ；https://github.com/OpenMathLib/OpenBLAS/commit/1cc377ef61d498b75c852aa4b9b042fe9422c347 ；https://github.com/OpenMathLib/OpenBLAS/commit/2d82d144e2791e37d7a314237b638d85b156a2ec ；https://github.com/OpenMathLib/OpenBLAS/commit/809e1cba8f1f3f89972581e8b82f2ec52e51eadb ；https://github.com/OpenMathLib/OpenBLAS/commit/376d3a138faa0a0fe483a8fa8d4fa1ab0d395acf ；https://github.com/OpenMathLib/OpenBLAS/commit/2ae019161a85333a35018b517d4b34474a7694e9 ；https://github.com/uxlfoundation/oneDNN/commit/d6f82a2d0d0db41e6daaf20fbb4fd352843aac64 ；https://github.com/ggml-org/llama.cpp/pull/17318 ；https://github.com/ggml-org/llama.cpp/pull/17448 ；https://github.com/ggml-org/llama.cpp/pull/17314 ；https://github.com/ggml-org/llama.cpp/pull/18199 ；https://github.com/uxlfoundation/oneDNN/commit/3bac96b8bc1fc9c348c986f38f65285693943d2f ；https://github.com/uxlfoundation/oneDNN/commit/8b48a77091062ce78959ccb96e43a8ee4e97022d ；https://github.com/uxlfoundation/oneDNN/commit/d6107ddb8be72041dade165a233c0de69f7a1387 ；https://github.com/uxlfoundation/oneDNN/pull/4410

### Finding 2（independent, 次要）— Bounded Spin-Wait Backoff（对应 Phase 3 Finding 2，row `rows-codegen.md` 行 30）

1. **Root cause**：`reduce_splits` 的 producer/consumer flag 轮询无退避策略：`while (!flags[split_idx]) { sched_yield(); }`（disassembly `lbu→beqz→jal sched_yield`，每次等待都让出并立即重读同一 cache line）。依据 `patterns/bounded_spin_wait_backoff.md` §Why this is slow：「无间隔轮询会持续占用前端、load pipeline 与共享 cache line」；每次迭代直接调用 `sched_yield` 又引入 syscall 延迟，64 线程共享 L2 时轮询放大 coherence 流量。
2. **The fix / 修复方式**：
   - 纠正对象：`reduce_splits` 的 flag 等待循环（`csrc/cpu/cpu_attn_impl.hpp` reduce_splits；两处 flag 轮询：split flag 与 guard_counter）。
   - 修复前：`while (!flags[split_idx]) { sched_yield(); }`
   - 修复后（保留 acquire 语义）：
     ```c
     unsigned delay = 1;
     while (atomic_load_explicit(&flags[split_idx], memory_order_acquire) != true) {
       for (unsigned i = 0; i < delay; ++i)
         __riscv_pause();          // Zihintpause；硬件支持（cpuinfo 含 zihintpause）
       delay = min(delay * 2, MAX_SPIN_DELAY);   // 有界递增退避
       if (delay >= YIELD_THRESHOLD)
         sched_yield();            // 项目既有让出路径
     }
     ```
   - 适用前提：同步必要（真实并发写者推进 flag）；内存序与退出条件保持不变。
   - 不可破坏的 correctness contract：`atomic_thread_fence`（`fence rw,w` 写侧 / `fence r,rw` 读侧）与 flag store-release 语义不得删除或弱化。
   - 限制/风险：阈值/退避步长须实测；`sched_yield` 在负载低时仍可能是正确选择；短等待时避免 syscall 开销。
   - 预期 Profile signals：`253392: jal sched_yield` 与 `253396/25339a` 轮询行占比下降；guard_counter 轮询（254ef2/254ef6）同步下降。
3. **Baseline facts 回填**：hardware ISA 含 `zihintpause` 与 `zawrs`；VLEN=128；bound type=memory-bound（同 Phase 1）。
4. **收益上界**：局部样本份额约 0.1–0.3%（split flag 轮询 + guard_counter spin），修复后预计仅回收该份额的一部分（等待本身耗时由生产者进度决定，本修复只减轮询干扰与 syscall 延迟）。
5. **三维路由判定**：current source=compiler-generated（`sched_yield` 经 `__gthrw_`/`__gthread_yield` 内联）；existence/reachability=退出条件由 `partial_output` 的 flag store 推进，可达且必要；function-level policy=用户态 OpenMP 并行代码，无 `.S` 合同。
6. **Implementation-shape proof**：不适用。
7. **Related PRs 小节**：按 `patterns/bounded_spin_wait_backoff.md` §Related PRs 表：`Related PRs：1 条 URL` — https://github.com/OpenMathLib/OpenBLAS/commit/0b3db03d4b41b86bb26170e4f4e36785ced9d947

## Phase 5 — Verification forecast / 验证预测：cpu_attention::AttentionMainLoop<...ISA)5, float, 64l, float...>

**Finding 1（primary，按收益上界排序第 1）**
- 修复对象：`gemm_micro_rvv_fma_Mx8_Ku4` A-slice 载入形态 + KV-slice tile 驻留/prefetch。
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`254340: flw fa2,0(t1)`（10.74%）、`254356/25437c/2543a2: vfmacc.vf`（7.66/8.67/10.30%）、`254b6a: flw fa2,0(t6)`（10.54%）、`254b82/254baa/254bd2: vfmacc.vf`（7.28/9.06/10.44%）——这些行的样本份额应显著下降或指令被 `vfmacc.vv` 替换。
- 应出现侧（锚定 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Verification）：预期 FP kernel 可达、scalar load/loop-control share 下降、出现预期 `vfmacc*`/vector load、cycles/FLOP 或 throughput 改善；**失败信号**：转换/packing 成为新瓶颈、register spill 增加、短 shape crossover 退化、数值误差超出合同；并对照 reference 覆盖 M/N/K=0/1/VLEN 边界/main-loop 整倍数与三维 tail，验证 FP32 累加精度、FMA contraction、NaN/Inf/±0/subnormal。
- 额外验证：同函数重跑 annotate，确认 A-slice `flw` 地址段（0x254340 起）样本份额下降；对比 `perf stat` L1_dcache_load_miss_rate（基线 22.204%）与 LLC_load_miss_rate（基线 39.681%）；对 decode-only 批（q_token_num=1）与 prefill 批分别 benchmark，确认 tile 驻留改动不使 prefill 退化。

**Finding 2（independent，按收益上界排序第 2）**
- 修复对象：`reduce_splits` flag 轮询与 guard_counter spin 的退避策略。
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`253392: jal 1bf3c0 <sched_yield@plt>`（0.03%）、`25339a: beqz a4,253392`（0.02%）、guard_counter 轮询 `254ef2/254ef6`。
- 应出现侧（锚定 `patterns/bounded_spin_wait_backoff.md` §Verification）：轮询 load/branch sample share 与每次等待的 load 次数下降；内存序/可见性/进展性/退出条件未变；对比低争用 latency、尾延迟、吞吐与 producer/holder 进展；有/无 `Zihintpause` 与单核/多核环境验证 fallback。

（两 finding 机制与地址分账，分别验证，不做合并验证替代。）

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | 1/1 组；`cpu_attention::AttentionMainLoop<...ISA)5, float, 64l, float...>::operator()` |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 2 个 L0 gate + bound-type gate） | ✅ | 7 行结论齐全；gap 标签：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`（含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`（include：rows-operator-rvv、rows-vectorized-tuning、rows-codegen；exclude：rows-asm/string-memory/offload/crypto/runtime-os）；`Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md`；顶层 finding 2（primary 1 + independent 1）；evidence 锚点 `254340: flw fa2,0(t1)`、`254356: vfmacc.vf v8,fa2,v14`、`254b6a: flw fa2,0(t6)`、`254b82: vfmacc.vf v2,fa2,v12`、`253392: jal sched_yield`；supporting 0；排除 8 条（量化/cache-blocking/weight-repack/register-group/operand-form/unroll/resource-scheduling/kernel-selection + spin 互斥 3 条）；推导式 2 条 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`rvv_floating_point_matmul_and_gemv_kernels.md`（§Why this is slow「packing 与消费合同不匹配」；§The fix 2/3 节）、`bounded_spin_wait_backoff.md`（§Why this is slow「无间隔轮询会持续占用前端、load pipeline 与共享 cache line」；§The fix 伪代码）；`The fix` 含 before/after 伪代码、correctness contract、风险、预期 Profile signals；Related PRs：17+1 条 URL |
| 5 | 路径合规：trace 可解释扫描集；多命中仲裁合规；入口模式 A 按动态份额排序 | ✅ | 模式 A（profile_backed）；路径 rows-operator-rvv → FP-matmul（primary，L1/L2）+ rows-codegen → spin-wait（independent，L4）；QK/PV 同 kernel 模板合并为同一 primary 区间证据；dynamic share 排序：Finding1（60–75%）> Finding2（0.1–0.3%） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：Phase 3(a) 引用行（254340/254356/254b6a/254b82/253392…）；出现侧：两 pattern 文件 §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无源码/补丁修改、无契约外实施分支；除 object-clarification 例外外无追问；交付物止于 Profile 证据、根因蓝图、完整 `The fix`、验证预测 |

修正记录：无