Functions under analysis: [sgemv_n]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`/root/openblas-workspace-level2/14-openblas-benchmark_level2_rv64/sgemv_2048x2048/annotate/002-sgemv_n-annotate.txt`；17 samples，event=cpu-clock，percent=local period；覆盖 sgemv_n 全函数 0x4a0a–0x4c00，含 hot loop body）
- perf stat（可选 bound/context）：已提供（`14-openblas-benchmark-riscv-sgemv_2048x2048.txt`：IPC 0.672，L1_dcache_load_miss_rate 0.257%，branch_miss_rate 0.402%，benchmark_ms 2.211，gflops 3.794）
- workload/binary/DSO/source context：部分提供（metadata：OpenBLAS commit 70c2410f17b6，`taskset -c 0-7 env OPENBLAS_LOOPS=16 /workspace/source/benchmark/sgemv.goto 2048 2048 1`，threads=1；`binaries` 为空且 warning "No ELF executable binaries found" → `source_context_gap`：无法核对 .S provenance 与备选 kernel 存在性）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：缺失（详见 Phase 1）
- hardware ISA（/proc/cpuinfo / hwprobe）：已提供（metadata cpuinfo isa line，含 `v`、`zve32f/x`、`zve64d/f/x`、`zvfh/zvfhmin`、`zvbb`、`zvbc`、`zvk*`、`smaia` 等）
- `vlenb`：已提供（metadata：vlenb=32 → VLEN=256 bits，批次冻结 hardware profile）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=cpu-clock；percent type=local period；同一运行窗口；函数级 workload 贡献：sgemv_n self=13.08%，17/130，来自 raw/perf_report.txt）
- Sampling IP precision：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`（metadata frozen snapshot）→ 暴露标准 `v`（RVV 1.0）与 zve64d/zvfh/zvbb/zvbc/zvk* 等；无 xtheadvector |
| Build ISA | `baseline_gap: build ISA`（工作区无 ELF/readelf 数据；可选命令：对承载热点地址的 DSO 运行 `readelf -A <object>`）。注：annotate 显示 `vle32.v`/`vfmacc.vf`/`vse32.v`/`vsetvli`/`vlse32.v`/`vsse32.v` 标准 RVV 1.0 mnemonic → 运行中实现含 `v`，与硬件无 mismatch；但 object 级 Tag_RISCV_arch 未知 |
| Vector flavor | RVV 1.0 `v*`（annotate 全 `v*` mnemonic），无 flavor mismatch |
| VLEN | 256 bits（vlenb=32，批次冻结 hardware profile） |
| Bound type | perf stat：L1_dcache_load_miss_rate=0.257%、branch_miss_rate=0.402%、IPC=0.672 → 非 cache-miss-bound、非 branch-bound；热点 loop 归为 latency/overhead-bound（循环携带依赖链 + 每迭代控制开销）。memory-bound gate：不成立 → 本地 RVV/loop 调优杠杆有效，不冻结 |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=local period（local，非 global）→ 四条 Amdahl gate 不满足，收益只能表述为当前 event 的局部样本份额；`baseline_gap: sampling metadata`（如需 workload 级口径，重采命令：`perf annotate --stdio -l -s sgemv_n --percent-type=global-period`） |
| Sampling IP precision | `baseline_gap: sampling IP precision`（raw/metadata.txt：call_graph=fp、perf_freq=499、event=cpu-clock；无 precise_ip/Exact-IP 信息）→ 单指令归因不成立，最高行只锚定 basic block / loop interval |

L0 baseline gate 1（hardware 有 `v` vs build 无 `v`）：build ISA 无 readelf 数据（gap），但 annotate 本身显示 `v*` 指令 → 运行中实现含 V；无 mismatch 证据，不冻结。L0 baseline gate 2（`th.v*` flavor）：无 `th.v*`，不适用。Bound-type gate：非 cache-miss-bound → 不冻结 compute/loop 调优 route；VLEN 已知（256）。

## Phase 2 — Scope / 分析边界

函数清单：sgemv_n（batch-012-function-001，rank 002，overhead 13.08% self）。hot loop anchor：inc_y==1 双列路径的内层 i-loop（m 方向，e32/m8）interval **[0x4bc4, 0x4bee]**（back-edge 0x4bee → 0x4bc4）：

- 最高行：`47.06 :  4bd0:  vle32.v v16,(t4)`（a2_ptr 列加载 = 第二列）
- 次高行：`41.18 :  4bd4:  slli    t1,a4,0x2`（指针推进字节偏移 = vl*4）
- 其余：`5.88 :  4bcc:  vle32.v v24,(t3)`（第一列加载）、`5.88 :  4bec:  add     a2,a2,t1`（y_ptr 推进）

函数内 17/17 样本全部落在该 interval（100% local share）。Sampling IP precision 未知 → 只锚定 interval 级机制；单行占比不承担 instruction-latency/cost 证据，机制由反汇编 def-use/dataflow 交叉确认。annotate 覆盖完整（无 annotate_gap）。

## Phase 3 — Pattern scan / 模式扫描：sgemv_n

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — 当前代码为 compiler-generated（sgemv_n 是 OpenBLAS interface/gemv.c 的 CNAME，annotate 形态为 intrinsic 宏代码生成，非手写 `.S`）；无 DWARF/object mapping 证明 `.S` provenance；无 policy/existence 四证（`source_context_gap`）。
2. `rows-operator-rvv.md` — **include** — FP32 GEMV 语义合同明确（sgemv：`y[i] += alpha*sum_j a[i,j]*x[j]`，vfmacc.vf + vle32.v + 标量 broadcast），row 行内明确接纳 already-vectorized GEMV 失效形态（load / FMA / accumulator dependency 主导）。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/sentinel-scan/compare/checksum。
4. `rows-vectorized-tuning.md` — **include** — annotate 已有 `v*`（RVV 1.0）且当前代码来源非手写 `.S`；每迭代 vsetvli、单一 vector accumulator、LMUL 已达最大（m8）等信号。
5. `rows-codegen.md` — **include** — 每轮 `slli`+`add` 指针推进（loop-level IV signal）与 loop-control 占样（slli 41.18%）；考察 addressing-fusion / scheduling 形态。
6. `rows-offload.md` — **exclude** — 无 matrix-engine / packed-SIMD 证据；`smaia` 为 AI 加速器扩展，不是本 GEMM/GEMV 内核执行路径的矩阵引擎证据。
7. `rows-crypto.md` — **exclude** — 无 AES/SHA/SM/GHASH/CRC/GF(2^k) 原语。
8. `rows-runtime-os.md` — **exclude** — 用户态 OpenBLAS benchmark hot loop，无 timer/CSR/ISR/特权路径。

Classes scanned: `rows-operator-rvv.md`、`rows-vectorized-tuning.md`、`rows-codegen.md`

### Local performance pattern scan: `sgemv_n`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Floating-Point Matmul and GEMV Kernels（**primary**） | hot interval 4bc4–4bee；`47.06 : 4bd0: vle32.v v16,(t4)` + `41.18 : 4bd4: slli t1,a4,0x2`；FP32 load/FMA/accumulator-dependency 主导；无 Int8 合同 | High | Medium | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |
| RVV Vector-State Management（supporting） | 每迭代重复建立兼容 vtype/vl：`4bc4: vsetvli a4,a6,e32,m8,ta,ma`，a4 输出喂 slli/sub/adds 循环携带链 | — | — | `patterns/rvv_vector_state_management.md` |
| Register-Budgeted RVV Loop Unrolling（supporting） | 单一 y accumulator v8 WAR 串行 recurrence（vle(4bc8)→vfmacc(4be0)→vfmacc(4be4)→vse(4be8)）；LMUL 已为最大 m8 | — | — | `patterns/rvv_register_budgeted_loop_unrolling.md` |
| Loop Induction Variable Strength Reduction（supporting） | 每轮 index scaling：`41.18 : 4bd4: slli t1,a4,0x2` + `4bdc/4bde/4bec: add` 指针推进 | — | — | `patterns/loop_induction_variable_strength_reduction.md` |

**Primary 三件套（RVV Floating-Point Matmul and GEMV Kernels）：**

(a) 逐字 evidence 引用：
- `47.06 :  4bd0:  vle32.v v16,(t4)` — 第二列加载（a2_ptr），hot interval [4bc4,4bee] 内
- `41.18 :  4bd4:  slli    t1,a4,0x2` — 指针推进字节偏移，同 interval
- 同 interval 支撑行：`4bc8: vle32.v v8,(a2)`（y 加载）、`4be0: vfmacc.vf v8,fa5,v24`、`4be4: vfmacc.vf v8,fa4,v16`（0%）、`4be8: vse32.v v8,(a2)`、`4bee: bgtz a6,4bc4`
- def-use（反汇编数据流）：v16←4bd0 仅被 4be4 vfmacc 消费；v8 链 vle→vfmacc→vfmacc→vse→下一迭代 vle 形成 WAR 串行 recurrence；vsetvli 输出 a4 喂 slli(4bd4)/sub(4bd8) → add(4bdc/4bde/4bec) → 下一迭代地址

(b) 互斥邻居排除（判别性观察）：
- Quantized matmul：4be0/4be4 为纯 FP32 `vfmacc.vf`，无 Int8/zero-point/requantization → 排除
- Precision conversion：interval 内无 float↔int/FP16 转换序列 → 排除
- Layout/weight-repack：a 以 lda 列主直接流式访问（a_ptr/a2_ptr 每轮常数步长推进），无独立 packing pass → 排除
- Cache-aware blocking：L1_dcache_load_miss_rate=0.257%，无 tile-residency/复用拐点证据 → 排除
- No-vectorization：main loop 已向量化（vle32.v/vfmacc.vf/vse32.v）→ 排除
- Register-group utilization：m8 已是最大 LMUL，无 vlmul_ext/trunc、无 vector spill、3 group=24 寄存器 ≤32 → 非 LMUL/live-set 不匹配，按 row 行内判据转 unroll row → 不另立命中
- Operand-form：vfmacc.vf 直接消费标量 fa5/fa4（每 j 迭代一次 flw+fmul 计算 temp/temp2），无 vmv.v.f/vfmv.v.f 临时 → 排除
- Inactive-lane：vsetvli 用 ta（tail agnostic），无 tu/mu、无 vmclr.m → 排除
- Resource-aware scheduling：缺 target-core latency/resource model 与 PMU 交叉证据（core-profiles 表无 X100 行，metadata 仅注明 OoO）→ 不命中（非独立 gap，并入 sampling IP precision 约束）
- Load/store addressing-mode fusion：访存已用指针寄存器 base+0 折叠，无只服务单条 memory op 的 addi → 排除
- Kernel selection：sgemv_n 即执行中的 RVV 路径，无 fallback/dispatch 证据，无备选 kernel 证据（source_context_gap）→ 排除
- 其余 operator/vectorized-tuning/codegen rows（elementwise、complex、activation、normalization、reduction、strided、gather、packing、color、triangular、conv、resampling、FFT、separable-transform、autovec-control、maximal-LMUL、copy/fill、SWAR、kernel-selection、fusion、workaround-retirement、hot-helper、control-flow、code-layout、register-pressure、gp-relative、FP-semantic、trap-guard、algebraic、ALU-constant、precision-conversion、atomic、spin-wait、ISA-substitution、native-width、redundant-extension、runtime-dispatch、tail-call、zero-based-comparison、redundant-sync、JIT、crypto、offload、runtime-os）：无对应 signal 或 row 内互斥判据排除，整组排除。

(c) 双 Confidence 推导式：route: GEMV FP32 语义合同 + hot interval 直接 provenance（sgemv_n disassembly，无 `.S` 疑义）+ 上述互斥排除 → **High**；impact: sample share 有（函数内 100% interval，self 13.08%）、VLEN 有（256）、bound type 有（非 cache-bound）但 build ISA 缺（`baseline_gap: build ISA`）且采样语义仅 local（`baseline_gap: sampling metadata`）+ 采样点少（17）→ **Medium**。

**Supporting evidence 行（各一行 (a) + supporting because，两种 confidence 保持 —）：**

- RVV Vector-State Management：(a) `0.00 :  4bc4:  vsetvli a4,a6,e32,m8,ta,ma`（每迭代重建兼容 vtype，AVL=a6 递减；a4 被 4bd4 slli / 4bd8 sub 消费）; supporting because: 重复 vsetvli 是 source 级 runtime-VL strip-mined 循环结构（`for(i=m;i>0;i-=vl){ vl=VSETVL(i); ...}`）的产物，非 compiler emitter 状态失效；GEMV 层 fixed-VL main loop 修复后该重复建立自然消失，故为同一机制的次级命中。
- Register-Budgeted RVV Loop Unrolling：(a) interval 内 def-use：`4bc8: vle32.v v8,(a2)` → `4be0/4be4: vfmacc.vf v8,...` → `4be8: vse32.v v8,(a2)`，单一 v8 跨迭代 WAR recurrence，LMUL=m8 已达最大；合法 frontier 候选：m8×2col×2accum（4 group=32 寄存器）或 m4×4col×2accum（6 group=24 寄存器）; supporting because: 该串行 recurrence 正是 GEMV row 的 "accumulator dependency" signal；LMUL 已在合法边界，固定 LMUL 下的 ILP 修复（独立 accumulator + 列数扩展）属于 GEMV accumulator 结构修正的一部分。
- Loop Induction Variable Strength Reduction：(a) `41.18 :  4bd4:  slli    t1,a4,0x2` 每轮 index scaling（vl*4），随后 4bdc/4bde/4bec 三条 add 推进指针；countdown `4bd8: sub a6,a6,a4` + `4bee: bgtz a6` 已是零值倒数比较形态; supporting because: slli 每轮重建的唯一原因是 vl 为 runtime（来自每迭代 vsetvli）；GEMV fixed-VL 化后 slli 变 loop-invariant、指针推进可折叠为常数 `addi`，属同一修复链的循环级强度削减收尾。

**多候选仲裁小段：** 全部 finding 位于同一 hot interval [4bc4,4bee]、同一机制链（runtime-VL 循环结构 → 每迭代 vsetvli → slli/IV 链 → 单一 y accumulator WAR 串行）。因果消除测试：GEMV 层修复（fixed-VL main loop + 独立 accumulator + 提高每轮 y 往返覆盖列数）会同时消除 vsetvli 重复建立（vector-state signal）、slli/sub/adds（IV signal）与 v8 WAR 串行链（unroll signal）→ 按 arbitration L1 语义层（`rows-operator-rvv.md` 整体归 L1）认领 **primary = RVV Floating-Point Matmul and GEMV Kernels（L1）**；vector-state（L3）、register-budgeted unroll（L4）、IV strength reduction（L4）为 **supporting**（同 interval、同机制、无独立修复对象/验证方法分账），不计顶层命中数、不单独排序。同 interval 样本不可分账相加（同调用链）。入口模式 A 但 percent=local → 排序只按局部份额：primary 收益上界 = 函数内 100%（17/17）局部份额；sgemv_n self = 13.08%（17/130）of run。

## Phase 4 — Root-cause blueprint / 根因蓝图：sgemv_n

### Primary: RVV Floating-Point Matmul and GEMV Kernels（Phase 3 命中 row：rows-operator-rvv.md GEMV row）

1. **Root cause**：sgemv_n 内层 m-向量循环（4bc4–4bee）是 runtime-VL strip-mined GEMV kernel，其 y read-modify-write 链被单一 accumulator 串行化，且每迭代重建循环状态。机制要点（引用 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md`）：
   - §Why this is slow："FMA accumulator 依赖过长"——y 链 `vle(v8)→vfmacc(v8)→vfmacc(v8)→vse(v8)→下一轮 vle(v8)` 为 WAR 串行 recurrence，每轮仅 2 个 FMAC（2 列）摊还一次 y 全往返（y 流量 1024 次往返/call）；
   - "packing 与消费合同不匹配"在本例表现为 a 未打包、以 lda 列主直接流式访问，内层循环每迭代承担 vsetvli+slli 的标量重建成本；
   - "沿错误轴向量化"部分：沿 m（输出轴）向量化本身正确（每 lane 一个独立输出 y[i]），但内层以 runtime-VL 形式实现，把 vsetvli/整数延迟放在循环携带路径上（a4→slli t1,a4,0x2→add a2/t3/t4→sub a6→bgtz）；
   - col2 加载（4bd0，47.06%）是进入两条 FMAC 前最后一个 load，其延迟直接暴露在串行 y 链末端（Load-use）。
2. **The fix / 修复方式**（依据 pattern §2 "增加合法独立 accumulator"、§3 N/K 轴比较，及 `references/kernel-conventions.md` §3 fixed-VL main loop/runtime-VL tail）：
   - 修复对象：OpenBLAS interface/gemv.c 的 RVV 路径（sgemv_n CNAME）内层 i-loop 与 j-loop 结构。
   - **步骤 1 — fixed-VL main loop**：进入循环前一次 `vl = vsetvlmax(e32,m8)`（=64），`nmain = m & ~(step-1)`（step=vl=64，2 的幂；m=2048=32×64 时无 tail）；main loop 用固定 vl=64，指针推进折叠为常数 `addi <ptr>,<ptr>,256`；m 尾部用 runtime-VL tail loop。消除每迭代 vsetvli、slli、sub 及三条 add 的标量指令与其依赖链（对应 41.18% slli 份额）。
   - **步骤 2 — 独立 accumulator + 提高列数**（pattern：不得固定 2/4/8，须枚举 frontier）：候选 A：m8×2列×2 独立 y-accumulator（4 个 m8 group=32 寄存器，无余量，需核对 allocator 不引入 spill）；候选 B：m4×4列×2 独立 y-accumulator（6 个 m4 group=24 寄存器，留 scratch）→ y 每轮往返覆盖 4 列（8 FMAC/往返 vs 现 2），y 流量减半，两条 FMAC 链独立并行，循环末一次 `vadd` 合并。
   - **步骤 3 — j-loop 摊还**：j-loop 由 `n>>1` 双列主循环扩展为 `n>>2` 四列主循环 + `n&3` tail（复用现有 `n&1` 路径结构）。
   - before/after 伪代码（结构示意，非可复制补丁）：
     ```
     before: for (i = m; i > 0; i -= vl) {
               vl = vsetvli(i);                      // e32,m8,ta,ma 每迭代
               vy  = vle32(y_ptr);  va = vle32(a_ptr);  va2 = vle32(a2_ptr);
               vy  = vfmacc(vy, t, va);  vy = vfmacc(vy, t2, va2);
               vse32(y_ptr, vy);
               y_ptr += vl*4; a_ptr += vl*4; a2_ptr += vl*4;   // slli 每迭代
             }
     after:  vl = vsetvlmax(e32,m8);  step = vl;  nmain = m & ~(step-1);
             for (i = 0; i < nmain; i += step) {
               vy  = vle32(y_ptr);  va = vle32(a0);  va2 = vle32(a1);
               vy  = vfmacc(vy, t0, va);  vy = vfmacc(vy, t1, va2);
               vse32(y_ptr, vy);
               y_ptr += 256; a0 += 256; a1 += 256;             // 常数 addi
             }
             // + runtime-VL tail loop（m - nmain 剩余）
             // 独立 accumulator 变体：vy0/vy1 双链 + 循环末 vadd 合并（m8×2col 或 m4×4col frontier 实机 A/B）
     ```
   - 适用前提：m 较大且可切 main/tail；X100 为 out-of-order（可隐藏部分跨迭代 load latency，但串行 y 链与 vsetvli 标量链无法隐藏）。
   - correctness contract：保持 `y[i] += alpha * sum_j A[i + j*lda] * x[j]` 的 read-modify-write 语义与 lda 列主布局；多 accumulator + 循环末合并改变 FP32 加法顺序 → 必须按 BLAS 误差容限验证（对照 reference 检查 FMA contraction、NaN/Inf/±0/subnormal 与允许误差，pattern §Verification）；`inc_y==1` 快速路径与 `inc_y!=1` 一般化路径（VLSEV/VSSEV，0x4a2e 起）各自合同保持；n 奇偶 tail、m 非 step 倍数 tail、n=0/m=0 边界覆盖。
   - 限制/风险：m8×2accum 无寄存器余量，allocator 可能引入 vector spill（须核对生成反汇编，风险时退回 m4×4col 或 1 accumulator）；4 列/往返增加 live group 与代码尺寸；短 m 下 fixed-VL 拆分与多列摊还收益低 → 保留 runtime-VL 小 m 路径或 crossover 阈值。
   - 预期 Profile signals：hot interval 内 vsetvli 由每迭代 1 次降为循环外 1 次；`slli`/`sub`/`add` 标量链消失（常数 addi）；4bd0 col2 load 与 4bd4 slli 的局部份额大幅下降；vfmacc/vle32.v 份额占比上升；cycles/FLOP 或 throughput 改善。
3. **Baseline facts 回填**：hardware ISA = rv64...v...（含 zve64d/zvfh/zvbb/zvbc/zvk*，frozen snapshot）；build ISA = `baseline_gap: build ISA`（无 readelf；annotate 显示 RVV 1.0 `v*`，build 至少含 `v`）；VLEN = 256 bits（vlenb=32）；bound type = latency/overhead-bound（L1 miss 0.257%、IPC 0.672，非 cache-miss-bound）。
4. **收益上界**：当前 sampled event（cpu-clock）下的**局部样本份额**：sgemv_n 内 100%（17/17）位于 hot interval 4bc4–4bee；sgemv_n self = 13.08%（17/130）of run。percent=local period 且 build ISA 缺失 → `baseline_gap: sampling metadata` + `baseline_gap: build ISA`，**禁止**称 workload 级 Amdahl 上界。kernel 级收益以 OpenBLAS benchmark 计时（当前 benchmark_ms 2.211ms/call、3.79 GFLOPS）重测为准；注意运行级 cpu-clock 样本被 random() 初始化噪声主导（65%），两种口径分离。
5. **三维路由判定**：
   - current source：compiler-generated（OpenBLAS interface/gemv.c RVV intrinsic 路径，CNAME sgemv_n；annotate 为 VSETVL/VLEV/VFMACCVF/VSEV 宏代码生成形态）；
   - implementation existence/reachability：执行中的即该 RVV 路径（annotate `v*` 指令）；无 dispatch/fallback/IFUNC 证据；无备选 riscv64 gemv 汇编 kernel 证据（`source_context_gap`）；
   - function-level policy：OpenBLAS gemv 为 C 级架构宏实现，无要求独立 `.S` 的 policy 证据 → 不走 missing-`.S` 分支；fix 针对现有 C 路径循环结构（源码级重构）。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs**（`patterns/rvv_floating_point_matmul_and_gemv_kernels.md` 本地表，按项目分组）：
   - OpenMathLib/OpenBLAS：7 条 URL（https://github.com/OpenMathLib/OpenBLAS/commit/0a967797a15617239523053633bf14be7895b25a、.../0acb60aab3c0134e879a68292904d8346dcd50ef、.../1cc377ef61d498b75c852aa4b9b042fe9422c347、.../2d82d144e2791e37d7a314237b638d85b156a2ec、.../809e1cba8f1f3f89972581e8b82f2ec52e51eadb、.../376d3a138faa0a0fe483a8fa8d4fa1ab0d395acf、.../2ae019161a85333a35018b517d4b34474a7694e9）
   - oneDNN：22 条 URL（d6f82a2d0d0db41e6daaf20fbb4fd352843aac64、3bac96b8bc1fc9c348c986f38f65285693943d2f、8b48a77091062ce78959ccb96e43a8ee4e97022d、8c52facbe61845d86062c76280b4bc515160c03e、b73fc3172d3e3230cf24ae29cbb6a07a09507a43、bd984d09dc5985a19fb427ac46d19d2cbd5558dd、d2a44b9b855706a0df33f9b6d4fb84f5420fdeaf、d6107ddb8be72041dade165a233c0de69f7a1387、fe04323ab0b4bba79ee60109fc391bb36052e43c、pull/4410、pull/4414、pull/4545、pull/4620、pull/4770、pull/4824、pull/4840、pull/4850、pull/4945、pull/5157、pull/5294、pull/5403、pull/5405）
   - llama.cpp：6 条 URL（pull/17318、pull/17448、pull/17314、pull/17161、pull/18199、pull/20627）
   - MNN：1 条 URL（pull/4426）
   - vLLM：1 条 URL（pull/44324）
   → `Related PRs：37 条 URL`（按 pattern 分组，组内 URL 去重）

### Supporting（不另立顶层蓝图，随 primary 验证预测合并）
- **RVV Vector-State Management**（Phase 3 命中 row：rows-vectorized-tuning.md vector-state row）：修复对象 = 消除每迭代重复 vsetvli（并入 primary 步骤 1）；依据 `patterns/rvv_vector_state_management.md` §The fix #2 "Reuse state only when VL and VTYPE both match"（fixed-VL 后 state 兼容可复用）与 §3 hoisting 到公共路径；`Related PRs：13 条 URL`（V8 c81ffb7a356d；QEMU d57dfe4b37ae、944b6dfd3d67、25669d275ce7、bd2c82283d21、81b9ef995a3b、949b6bcb2729、b8e1f32cda78；LLVM pull/148246、d0554ae4cf26、pull/118285、pull/123878、f59307bfdc01）。
- **Register-Budgeted RVV Loop Unrolling**（rows-vectorized-tuning.md unroll row）：修复对象 = 固定 LMUL 下独立 accumulator 数 + 列数（并入 primary 步骤 2）；依据 `patterns/rvv_register_budgeted_loop_unrolling.md` §The fix："每个 unrolled lane 必须有独立 accumulator"、"最后一次合并则是换取较短 main-loop recurrence 的固定成本"；`Related PRs：2 条 URL`（XNNPACK 0c7b565c2fc1、pull/10403）。
- **Loop Induction Variable Strength Reduction**（rows-codegen.md IV row）：修复对象 = 常数指针步长 + 预计算 trip count（并入 primary 步骤 1）；依据 `patterns/loop_induction_variable_strength_reduction.md` §The fix #1/#2（指针递增 + 预计算 end/边界）与 #4（main trip count 与 remainder 一次性拆出）；`Related PRs：3 条 URL`（linux 18be4ca5cb4e、OpenBLAS 477dd40f073c、d832ee50868a）。

## Phase 5 — Verification forecast / 验证预测：sgemv_n

入口模式 A，按收益上界顺序验证（唯一 primary，supporting 随 primary 预测，不单独排序）：

- **Primary（GEMV）— 消失/缩小侧**（锚定 Phase 3(a) 引用行）：
  - `47.06 :  4bd0:  vle32.v v16,(t4)` 与 `41.18 :  4bd4:  slli    t1,a4,0x2` 的局部份额应大幅下降；
  - hot interval 内每迭代 `4bc4: vsetvli a4,a6,e32,m8,ta,ma` 应移出循环（仅循环外 1 次）；`4bd4 slli`、`4bd8 sub`、`4bdc/4bde/4bec add` 应被常数 `addi <ptr>,<ptr>,256` 取代。
- **出现侧**（`patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Verification + `references/kernel-conventions.md` §3）：
  - 出现预期 vfmacc*/vector load 占比上升，cycles/FLOP 或 throughput 改善（成功判据）；
  - fixed-VL main loop 反汇编不含 per-iteration vsetvli；新增独立 accumulator 的循环末 `vadd` 合并指令出现；
  - 无新 vector spill/reload（m8×2accum 候选须核对，风险时回退 m4×4col）；
  - 失败判据：conversion/packing 成新瓶颈、spill 增加、短 shape crossover 退化、数值误差超出合同。
- **正确性**：对照 reference 检查 FP32 累加精度、FMA contraction、NaN/Inf/±0/subnormal 与允许误差（reassociation 容限）；覆盖 m/n=0、1、VLEN 边界（VLMAX=64）、step 整倍数（2048=32×64）、n 奇偶 tail、m tail，及 `inc_x/inc_y≠1` 一般化路径回归。
- **Benchmark**：同机同输入 `taskset -c 0-7 env OPENBLAS_LOOPS=16 /workspace/source/benchmark/sgemv.goto 2048 2048 1` 重测 benchmark_ms / GFLOPS（当前 2.211ms / 3.79 GFLOPS）；代表性短/中/长 m、n 验证无 crossover 退化。
- **Supporting 各自锚点并入上述**：unroll 按合法 frontier 实机 A/B（m8×2col×2acc 与 m4×4col×2acc），确认 backedge/控制指令下降、无 spill；vector-state 验证 hot interval 内 vsetvli 计数=1；IV 验证 `slli` 指令从 hot interval 消失。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 全部出现（载荷：1/1 组；[sgemv_n]） | ✅ |
| 2 | Phase 1 输出要求满足：7 行 baseline 表 + 2 个 L0 gate + bound-type gate；结论项含 Sampling IP precision 行；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`（载荷：7 行；3 个 gap 标签 + Sampling IP precision） | ✅ |
| 3 | Phase 3 输出要求满足：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md、rows-vectorized-tuning.md、rows-codegen.md；顶层 finding=1（primary GEMV）；每个 finding 有 evidence 锚点（`47.06 : 4bd0: vle32.v v16,(t4)`、`41.18 : 4bd4: slli t1,a4,0x2`、`0.00 : 4bc4: vsetvli a4,a6,e32,m8,ta,ma`）；supporting=3；互斥排除≥18 条（含整组排除）；推导式=1 条（route High / impact Medium）（载荷：8 trace；3 class；1 finding；3 supporting；≥18 排除；1 推导式） | ✅ |
| 4 | Phase 4 输出要求满足：已读 pattern 4 个（rvv_floating_point_matmul_and_gemv_kernels.md、rvv_vector_state_management.md、rvv_register_budgeted_loop_unrolling.md、loop_induction_variable_strength_reduction.md）对应命中 row（GEMV/vector-state/unroll/IV）；引用短语：§Why this is slow "FMA accumulator 依赖过长"、§The fix #2 "Reuse state only when VL and VTYPE both match"、§The fix "最后一次合并则是换取较短 main-loop recurrence 的固定成本"、§The fix #1/#2 "指针递增 + 预计算 end"；The fix 含 before/after、correctness、风险、Profile signals 锚点；Related PRs：GEMV 37 条 URL、vector-state 13 条、unroll 2 条、IV 3 条（载荷：4 pattern；4 row；引用短语 4 组；before/after+contract+risk+signal；37+13+2+3 条 URL） | ✅ |
| 5 | 路径合规：8 项 class 可解释扫描集；多命中仲裁 primary/supporting（因果消除，L1 语义层认领）；每 blueprint leaf 来自通过 gate 的 row；入口模式 A 但 percent=local → 收益上界为局部份额并标 `dynamic priority` 受限；`th.v*` 未全局停扫（载荷：模式 A；路径 operator→vectorized-tuning→codegen；class 3 文件） | ✅ |
| 6 | Phase 5 两侧锚定：消失侧 `4bd0: vle32.v v16,(t4)`、`4bd4: slli t1,a4,0x2`、`4bc4: vsetvli a4,a6,e32,m8,ta,ma`；出现侧 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Verification（载荷：3 个消失侧锚点 + §Verification） | ✅ |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成或契约外实施分支；无向用户追问（无 object-clarification 例外触发）；交付物止于 Profile 证据、根因蓝图、完整 The fix 和验证预测（载荷：Phase 0–6 全结构，无实施分支） | ✅ |

修正记录：无