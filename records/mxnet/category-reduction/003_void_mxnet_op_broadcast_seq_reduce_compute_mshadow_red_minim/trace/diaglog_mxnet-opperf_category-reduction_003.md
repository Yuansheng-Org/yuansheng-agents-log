Functions under analysis: [void mxnet::op::broadcast::seq_reduce_compute<mshadow::red::minimum, 2, float, float, float, mxnet::op::mshadow_op::identity, mxnet::op::mshadow_op::set_index_no_op<float, long> >(...) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 `perf annotate`：已提供（`003-void mxnet：：op：：broadcast：：seq_reduce_compute＜mshadow：：red：：minimum, 2, float, float, float, mxnet：：op：：mshadow_op：：identity, mxnet：-43843a1fd551-annotate.txt`；event=`cpu-clock`，1294 samples，`percent: local period`，hot loop body 完整覆盖）
- `perf stat`（bound/context 证据）：已提供（`11-mxnet-opperf-benchmark-riscv-category-reduction.txt`；IPC=0.484，L1_dcache_load_miss_rate=2.991%，LLC_load_miss_rate=36.729%，branch_miss_rate=0.487%）
- workload/binary/DSO/source context：已提供（mxnet-opperf，热点承载 object 为 `libmxnet.so`，commit `b84609d3fc73d20929c114eab95faaa56e6c5ede`（master），函数为 C++ 模板 `seq_reduce_compute` 的 OpenMP clone）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `libmxnet.so-elf-A`，见 Phase 1）
- hardware ISA（`/proc/cpuinfo` / hwprobe）：已提供（metadata cpuinfo，T-Head C920v2 / SOPHGO SG2044，见 Phase 1）
- `vlenb`：已提供（`vlen_bits=128`，`vlenb=16`）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=`cpu-clock`（可解释为时间）、percent type=`local period`、单 annotate 窗口）；函数级 workload 贡献未知（该函数为 category-reduction 批次 rank 003 热点，但绝对占比未知）→ 见 Phase 1 `baseline_gap: sampling metadata`（workload 级）
- Sampling IP precision：缺失（无 `precise_ip` / Exact-IP 记录）→ 见 Phase 1 `baseline_gap: sampling IP precision`

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`（metadata cpuinfo）。含 `v`（RVV 1.0）、`zve64d`、`zfa`、`zbb`/`zba`/`zbs`、`zvfh`；OoO superscalar |
| Build ISA | libmxnet.so `Tag_RISCV_arch`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`。build 含 `v`/`zve64d`/`zvl128b`，**不含** hardware 已具备的 `zfa`/`zbb`/`zba`/`zbs` |
| Vector flavor | RVV 1.0 标准 `v*` mnemonic（build/hardware 均 RVV 1.0）；本函数 annotate 中 **0 条 `v*`/`th.v*` 指令**（全 scalar），无 flavor mismatch |
| VLEN | 128 bits（`vlenb=16`，metadata `vlen_bits=128`）；SEW=32 时 m1=4 lanes |
| Bound type | 函数内：compute/latency-bound（loop-carried min accumulator + 分支主导；load 仅 1.62%）。workload 全局：IPC=0.484、L1 命中率高（miss 2.99%）、LLC miss 36.7%、branch miss 0.49%——不足以把本函数判为 memory-bound |
| Sampling semantics | event=`cpu-clock`（时间可解释）；percent type=`local period`（非 global-period）；单窗口；函数 workload 级贡献未知 → 只能给出「当前 event 下的函数内局部样本份额」，**不得**称 workload 级 Amdahl 上界。标 `baseline_gap: sampling metadata`（workload 级贡献） |
| Sampling IP precision | 未知（无 `precise_ip`/Exact-IP 记录）→ 标 `baseline_gap: sampling IP precision`；单行高占比只能锚定 basic block / loop interval，不做单指令 latency 归因 |

L0 baseline gate：hardware 有 `v` 且 build 有 `v`（`v1p0`+`zve64d`+`zvl128b`）→ **无 hardware/build RVV mismatch**。annotate 无 `th.v*` → flavor gate 不触发。Bound-type gate：函数内为 compute/latency-bound（非 memory-bound），本地 compute-vectorization fix 的 performance-impact confidence 不被 memory-bound 降级；但 `baseline_gap: sampling IP precision` 使单指令归因受限。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数）。hot loop 锚点：**内层归约循环 interval `0x1ca1604–0x1ca163e`**（对单个输出元素 `idx` 的 `k = 0..M-1` 归约；外层为 OMP 分块，样本占比 0.00%）。该 interval 占据本函数 ~100% 的 1294 个 local samples。

最高占比行（interval 内 trace anchor）：
- `82.92 :   1ca1638:        bnez    a5,1ca163e <...+0xf8>` — `fle.s` 之后的「dst<=src 则跳过更新」条件分支（min 归约 compare/conditional-update 链）
- ` 5.87 :   1ca162e:        beqz    a1,1ca163e <...+0xf8>` — NaN guard（`1ca160c feq.s` + `beqz`）
- ` 3.55 :   1ca163e:        bne     a0,a4,1ca1604 <...+0xbe>` — 循环回边
- 地址/索引计算 interval：`0.31 : 1ca1604 div`、`0.70 : 1ca1610 rem`、`1.39 : 1ca1622 add`、`1.62 : 1ca162a flw fa5,0(a5)`（合计约 6.96%）
- volatile 累加器往返：`0.00 : 1ca1608 flw fa5,-68(s0)` / `0.00 : 1ca163a fsw fa5,-68(s0)`（每迭代经栈槽重读重写）

Sampling IP precision 未确认 → 以上行只作 interval 归属锚点，不承担单指令 latency 归因。annotate 覆盖完整（非 `annotate_incomplete`）。

## Phase 3 — Pattern scan / 模式扫描：void mxnet::op::broadcast::seq_reduce_compute<mshadow::red::minimum, 2, ...> [clone ._omp_fn.0]

### Class selection trace

| Class 文件 | include/exclude — 触发观察 |
|---|---|
| rows-asm.md | exclude — 当前代码来源为 compiler-generated（C++ 模板 + OpenMP `[clone ._omp_fn.0]`），无 `.S` source/DWARF/object-mapping 证据；亦无 missing-`.S` 的 policy/existence 证据 |
| rows-operator-rvv.md | include — compiler-generated scalar loop，算子语义为 min 归约；第一级必选 class |
| rows-string-memory.md | exclude — 非 copy/fill/sentinel/compare/checksum 循环 |
| rows-vectorized-tuning.md | exclude — 完整 annotate 中 0 条 `v*`；本 class 要求已有 `v*` |
| rows-codegen.md | include — compiler-generated 指令形态：内层循环每迭代执行 `div/rem` 索引分解与 `slli+add` 地址合成；评估 ISA-substitution 与 LIVSR 两 row |
| rows-offload.md | exclude — 无矩阵引擎/packed-SIMD/权重重排/跨 VLEN 可移植证据，无 GEMM/tile 结构 |
| rows-crypto.md | exclude — 非密码学原语 |
| rows-runtime-os.md | exclude — 非 RTOS/kernel 侧 timer/ISR/CSR 热点 |

`Classes scanned: rows-operator-rvv.md, rows-codegen.md`

### Local performance pattern scan: `seq_reduce_compute<minimum,2,...> [._omp_fn.0]`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Extrema Reduction Kernels（primary） | scalar min 归约：仅维护 best value（IndexOP=`set_index_no_op`）、loop-carried accumulator 存栈槽 `-68(s0)`、compare/conditional-update 链（`82.92% : 1ca1638 bnez` + `5.87% : 1ca162e beqz` + `0.39% : 1ca1634 fle.s`）+ 循环回边 `3.55% : 1ca163e bne`；零 `v*` | High | Medium | `patterns/rvv_extrema_reduction_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar（仅 `flw`/`fle.s`/`fsw`/整数分支）、zero `v*`；只解释缺少向量执行，不决定贡献载体 | — | — | `patterns/no-vectorization.md` |
| ISA Extension-Specific Instruction Substitution（supporting，Zfa `fminm.s`） | min 更新用 compare+branch 合成（`1ca1630 flw`→`1ca1634 fle.s`→`1ca1638 bnez`），hardware 有 `zfa`（NaN-propagating `fminm.s` 单指令可替换），build 缺 `zfa` | — | — | `patterns/isa_extension_specific_instruction_substitution.md` |

**Primary finding 三件套：**

**(a) 逐字 evidence 引用**（所属 interval：内层归约循环 `0x1ca1604–0x1ca163e`）：
- `82.92 :   1ca1638:        bnez    a5,1ca163e <...+0xf8>` — `fle.s` 后的条件跳过更新分支（dst<=src 时保持当前 min），归约 compare/conditional-update 链的核心
- ` 5.87 :   1ca162e:        beqz    a1,1ca163e <...+0xf8>` — `feq.s fa5,fa5` 后的 NaN guard
- ` 3.55 :   1ca163e:        bne     a0,a4,1ca1604 <...+0xbe>` — 循环回边
- ` 0.00 :   1ca1608:        flw     fa5,-68(s0)` 与 ` 0.00 :   1ca163a:        fsw     fa5,-68(s0)` — loop-carried accumulator 经 volatile 栈槽每迭代往返
- 地址/索引计算子 interval：` 0.31 : 1ca1604: div a5,a4,a2`、` 0.70 : 1ca1610: rem a5,a5,t1`、` 1.39 : 1ca1622: add a5,a5,a3`、` 1.62 : 1ca162a: flw fa5,0(a5)`（合计约 6.96%）
- 符号证据：`mshadow::red::minimum::Reduce<float>(float volatile&, float)`（annotate 行 85、119）证明 volatile 限定累加器；`set_index_no_op<float,long>` 证明归约不维护 index

**(b) 互斥邻居排除**：
- Arg-extrema selection：排除 — IndexOP 为 `set_index_no_op<float,long>`（annotate symbol），只维护 value，不维护 index，无 tie-break 合同
- Widening additive reduction：排除 — 更新路径为 `fle.s`+`bnez`+`fsw` 的条件覆盖，无任何 add/MAC 累加指令（无 `fadd.s`/`fmadd.s`）
- Normalization：排除 — 循环内无多遍统计/exp/除法（仅归约 compare 与地址数学）
- Elementwise：排除 — 存在跨元素 loop-carried 依赖（`-68(s0)` 栈槽），非 lane-independent
- Strided layout / Indexed gather：排除 — 主导样本在 compare/update 簇（82.92+5.87+3.55≈92.3%），地址计算仅 ~5.3%；且访问由 shape 确定性生成，非 data-dependent index
- No vectorization：排除为 primary — 被更具体的 extrema-reduction row 认领（行内互斥判据），仅作 supporting

**(c) 双 Confidence 推导式**：
- `route: 符号与 annotate provenance（C++ 模板 OMP clone，非 .S）+ 语义合同（value-only min、IndexOP no-op、NaN-propagating Reduce）+ 样本主导在 compare/conditional-update/loop-carried accumulator/循环分支 → High`
- `impact: 热点 interval 样本份额已知（~99% 的 1294 local samples 在归约循环；compare/update 簇 ~89.5%）+ VLEN=128 已知 + 函数内 compute/latency-bound；但 percent=local period 无法给 workload 级上界（baseline_gap: sampling metadata）+ precise_ip 未知（baseline_gap: sampling IP precision）→ Medium`

**Supporting evidence 行：**

- No vectorization（supporting because: 同一 main loop、同一机制——V-capable hardware+build 上向量执行单元未参与；只解释缺失的向量执行载体，不决定贡献）— (a) main loop interval `0x1ca1604–0x1ca163e` 内全部为 scalar `flw/fle.s/fsw/bnez/bne` 与整数 `div/rem/mul/add`，零 `v*`；hardware `v` 与 build `v1p0` 均确认
- ISA Extension-Specific Instruction Substitution（supporting because: 同一 interval、同一机制——min 更新实现；`fminm.s`（Zfa，NaN-propagating）单指令可替换 compare+branch 链；该信号被 primary 向量化修复因果消除）— (a) `0.31 : 1ca1630 flw fa4,-68(s0)` + `0.39 : 1ca1634 fle.s a5,fa4,fa5` + `82.92 : 1ca1638 bnez a5,...`；hardware `isa` 含 `zfa`，build `Tag_RISCV_arch` 不含 `zfa`

**多候选仲裁小段**：evidence 与机制不可分账——三候选均指向同一归约循环 `0x1ca1604–0x1ca163e` 的同一 min 归约实现机制。Extrema reduction（L1 semantic dispatch）为唯一 primary；no-vectorization（L1）与 ISA-substitution（L4 compute/codegen micro-structure）为 supporting（上层向量化修复会使其 signal 消失，按因果消除测试并入）。无 companion（glibc 白名单不适用）。无 independent。入口条件 A：primary 的 evidence sample share 加总 ≈ compare/update 簇 89.49% + 地址/load 子 interval 6.96%（fix 重构整个循环）≈ **循环内 ~100% 的 1294 函数内 local samples**；supporting 不单独排序。

排除记录：`Kernel Selection`（无既有 RVV kernel 未选中的证据）、`Loop Induction Variable Strength Reduction`（地址并非 k 的仿射函数——`div/rem` 分解破坏仿射性，「指针递增替代 index scaling」前提不成立；该子 interval 由向量化修复整体消除，不独立命中）、`Cache-Aware Blocking`（无 tile/复用距离证据）、`Control-Flow/Code-Layout`（无分支菱形/RAS 信号）、`Register Pressure`（无 spill/reload 证据，仅 s3/s4/s5 保存）、`FP Semantic Lowering`（`feq.s`/`fle.s` 为归约语义所需，非 lowering 冗余）。

## Phase 4 — Root-cause blueprint / 根因蓝图：void mxnet::op::broadcast::seq_reduce_compute<mshadow::red::minimum, 2, ...> [clone ._omp_fn.0]

**1. Root cause**：编译器将 mxnet broadcast-reduce（`mshadow::red::minimum`，value-only，IndexOP no-op）的内层归约生成为**全 scalar 的 loop-carried min 循环**：每输出元素在 `k=0..M-1` 上逐元素执行「volatile 栈槽累加器往返（`1ca1608/1ca1630 flw -68(s0)`、`1ca163a fsw -68(s0)`）+ NaN guard（`1ca160c feq.s`+`1ca162e beqz`）+ 条件更新（`1ca1634 fle.s`+`82.92% 的 1ca1638 bnez`）+ 回边（`1ca163e bne`）」，并每迭代用 `div/rem/mul/add/slli`（`1ca1604`–`1ca1628`）重建 `dot(unravel(k,rshape),rstride)` 索引。对照 `patterns/rvv_extrema_reduction_kernels.md` §Why this is slow：「loop-carried scalar best-value accumulator 串行化 compare/update，并让每个输入元素承担分支与循环控制」——本函数 82.92% 的样本正好落在该串行化的条件分支上；`Reduce<float>(float volatile&,float)` 的 **volatile 限定累加器**迫使每迭代内存往返，是编译器无法向量化的直接障碍（§The fix 亦要求「如果标量 reference 的 NaN 或 signed zero 语义与 RVV reduction 不一致，应使用额外分类、比较、mask 或标量特殊路径保持原语义」——本函数标量语义为 **NaN 传播型 min**（`if(!isnan(dst)) if(!(dst<=src)) dst=src`），与 RVV `vfredmin`（IEEE minNum，忽略 NaN）不一致，修复必须处理）。同一机制支撑 `no-vectorization.md`（V-capable hardware+build 上零 `v*`）与 `isa_extension_specific_instruction_substitution.md` §5（compare+branch 合成 min，硬件有 `zfa` 却未用 `fminm.s`）。

**2. The fix / 修复方式**：将内层归约改写为 **RVV 分块 min 归约**（intrinsic，`__riscv_v_intrinsic` + runtime `hwprobe`/`__riscv_v` 编译期 guard 保护），结构遵循 pattern §The fix 的「per-chunk 归约 + 当前全局标量作为 seed」形态与 `kernel-conventions.md` §3（fixed-VL main loop + runtime-VL tail）、§2（LMUL 按 live-vector budget）。

Before（当前生成形态，每迭代）：
```asm
1ca1604: div  a5,a4,a2      ; k / rshape[1]（每迭代多周期除法）
1ca1608: flw  fa5,-68(s0)   ; 重读 volatile 累加器
1ca160c: feq.s a1,fa5,fa5   ; NaN guard
1ca1610: rem  a5,a5,t1      ; k % rshape[0]
1ca1614: rem  s6,a4,a2      ; k % rshape[1]
1ca1622: add  a5,a5,a3      ; + broadcast offset
1ca1628: add  a5,a5,t3      ; + big 基址
1ca162a: flw  fa5,0(a5)     ; load big[idx]
1ca162e: beqz a1,...        ; dst 为 NaN 则跳过
1ca1630: flw  fa4,-68(s0)
1ca1634: fle.s a5,fa4,fa5   ; dst <= src
1ca1638: bnez a5,...        ; 保持 dst（82.92% 样本在此）
1ca163a: fsw  fa5,-68(s0)   ; 条件更新
```

After（RVV intrinsic 形态）：
```cpp
// 每输出元素归约；big 访问按 rstride 确定 stride（unit-stride 用 vle32，固定 stride 用 vlse32）
float best = limits::PosInfValue<float>();          // seed = +inf（SetInitValue 语义）
const int epr = __riscv_vsetvlmax_e32m1();          // VLEN=128, SEW=32 → 4 lanes；live set 允许时可选 m2/m4（kernel-conventions §2）
for (int off = 0; off < M; ) {
  int vl = __riscv_vsetvl_e32m1(M - off);           // main loop 用 epr，tail 用 runtime vl（§3）
  vfloat32m1_t chunk = __riscv_vle32_v_f32m1(big + idx + off, vl);   // 或 __riscv_vlse32_v_f32m1(big+idx, stride, vl)
  // NaN 传播契约：任一元素为 NaN → 结果 NaN（标量 reference 语义）
  vbool32_t non_nan = __riscv_vmfeq_vv_f32m1_b32(chunk, chunk, vl);  // 对 NaN lane 为 false
  if (__riscv_vfirst_m_b32(__riscv_vmnot_m_b32(non_nan, vl), vl) >= 0) { best = NAN; break; }
  vfloat32m1_t seed = __riscv_vfmv_v_f_f32m1(best, vl);
  vfloat32m1_t red  = __riscv_vfredmin_vs_f32m1_f32m1(chunk, seed, vl);  // minNum（对 NaN lane 忽略，已由 NaN 检测兜底）
  best = __riscv_vfmv_f_s_f32m1_f32(red);
  off += vl;
}
// 结果写回 small[idx]（addto 分支保持现状）
```
辅助方向（supporting，scalar 兜底/tail 路径）：用 Zfa `fminm.s`（NaN-propagating minimum）单指令替换 `fle.s+bnez` 分支链（硬件已含 `zfa`），须以精确 `-march`（含 `zfa`）重建或 `__riscv_zfa` guard；RVV `vfmin`/`vfredmin` 为 minNum（NaN 忽略），**不能**直接替代 NaN 传播语义。

适用前提：M>0 的归约主体（`M==0` 路径保持现有 `1ca15c8 beqz` 分支）；访问 stride 固定（unit-stride `vle32` 或固定 stride `vlse32`）；工具链 `-march` 含 `v`（当前 build 已含 `v1p0`/`zvl128b`）。不可破坏的 correctness contract：①NaN 传播（任一元素 NaN → 结果 NaN）；②±0 语义——标量 `!(dst<=src)` 在 ±0 相等时保留先值（first-wins），而 `vfredmin`/minNum 对 (+0.0,-0.0) 返回 -0.0，必须按 pattern §Verification「明确 FP NaN、±0、ordered/unordered 与 seed 语义」显式对齐（接受差异需在测试中记录，或加 ±0 显式处理）；③seed=+inf；④tail 必须用与有效元素相符的 `vl`，不得用尾块 `vl` 解释长块（pattern §The fix）；⑤累加器改用局部非 volatile 寄存器（volatile 往返仅在 API 需要外部可观察者时保留）。限制/风险：M 较小（<VLEN lanes）时 setup/`vsetvl` 开销占比升高（core-profiles：C920v2 OoO、VLEN=128，per-iteration overhead 更可见）；FP 归约无 bit-exact 合同时须验证数值等价（min 结合律成立，但 NaN/±0 规则不同）。修复后预期 Profile signals：`1ca1638/1ca162e/1ca163e` 的 scalar 分支份额消失或大幅缩小，出现 `vsetvli`/`vle32`/`vlse32`/`vfredmin.vs`，`div/rem` 索引链消失，cycles/element 改善。

**3. Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zfa_zbb_zba_zbs_zve64d_...`（SG2044/C920v2，OoO）；build ISA=`rv64i2p1_..._v1p0_..._zve64d_..._zvl128b`（含 `v`，不含 `zfa`/`zbb`）；VLEN=128 bits（vlenb=16）；bound type=函数内 compute/latency-bound（loop-carried dependency；workload 级 IPC 0.484，本函数非 memory-bound）。

**4. 收益上界**：当前 sampled event（cpu-clock）下的函数内局部样本份额——primary fix 重构整个归约循环（compare/update 簇 89.49% + 地址/load 子 interval 6.96% + 回边 3.55% ≈ 100% 的 1294 函数内 local samples）；因 percent=local period、workload 级贡献未知，**不构成 workload 级 Amdahl 上界**（`baseline_gap: sampling metadata`）。supporting 不单独计上界。

**5. 三维路由判定**：
- current source：compiler-generated scalar 代码（C++ 模板 `seq_reduce_compute` 的 OMP clone，annotate 符号后缀 `[clone ._omp_fn.0]`；无 `.S` provenance）
- implementation existence/reachability：无现成 RVV kernel 存在（整函数零 `v*`，无 dispatch/registration 证据）→ 修复 = 在现有模板 kernel 内向量化（intrinsic 特化），非修复 dispatch
- function-level policy：mxnet 使用 compiler-generated `MSHADOW_XINLINE` 模板 kernel，无 assembly-default policy 证据 → **policy-backed missing `.S` 分支不适用**（policy/existence 四证不齐），fix 载体为普通/intrinsic 代码

**6. Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

**7. Related PRs 小节**：
- `patterns/rvv_extrema_reduction_kernels.md`：Related PRs：5 条 URL（oneDNN #5361、oneDNN a95f0060cfcb、MNN #4036、MNN 09c339c、MNN #4433）
- `patterns/no-vectorization.md`：Related PRs：14 条 URL（OpenCV #22179、#22520、#23980、#24058、#24132、#24166、#24301、#24325、#27160、#27119、#27097、#27007、#26958、#26865、b902a8e792e1、2c16f3b7d2b2、e06502a254f7、a2d784b6f53a、83104bed3209）
- `patterns/isa_extension_specific_instruction_substitution.md`：Related PRs：多项目（Go 3659b8756a2b、a6ecdf29e34d、#59488、63ab68ddc5f1、1951afc9193f；OpenCV a00818047ff5、7e2c8cc9f4c4、f0d29cd33c5d、#21351；OpenSSL 03ce37e11729、ca6286c382a7、48b6776678d7、6136408e6abf、e4fd3fc379d7、80c664db430d、08c8dd6b8ced、49a3e7adc392、a41f9135f082、4dbb537bd1ea、608cadfbdbdb、b1b889d1b3fc、657d1927c6b、611685adc0a、7ae2bc9df6e0；Linux e8620bd7e5e0、5ba15d419fab、cc2294d3f9c9、36e224168721、e11e367e9fe5、75ab93a244a5、c64086849110；OpenJDK 6b89954c6534、a7631ccf18e4、#22752、08d563ba1504、b1a21b563e3a、#22410、9f582e56baee、#24096、1a4bbb0027ae、2ed7ad4b5c7d、edfe28541a6e、b891bfa7e67c、3d3b78203710、a7a09f69abc6、bcc33d5ef3bd、#22386、5866b16dbca3、8cb9b479c529、#11921；llama.cpp #17784；V8 6f100865663f、e62c1e307d20、200b5212ae47、5136fb5200c1、e02d2238f6ea、94a3c420e2a4、1818e36d54d1、5f433dd5024a、9bbbde26bb90、758956654f28、32c5d22333c1、89719bc239f4、223d7fb26b12、b2852080c401、8b842dbb9f6d、e855c14cab2c、8033cc56e2af、a654e27b50fe、21289aa92c80、19a0f69f4c4b；QEMU 3de1fb712a07、6ef584318238；LLVM #170824、4c1e1e05cb90、#92926、#152744、#122698、13e32a8a3c95、787eeb8597fa、c705b7b04dba）

## Phase 5 — Verification forecast / 验证预测：void mxnet::op::broadcast::seq_reduce_compute<mshadow::red::minimum, 2, ...> [clone ._omp_fn.0]

- **Primary（extrema reduction）**：
  - 应消失/缩小：Phase 3(a) 引用行——`1ca1638 bnez`（82.92%）、`1ca162e beqz`（5.87%）、`1ca163e bne`（3.55%）、`1ca1604 div`/`1ca1610 rem`/`1ca1614 rem` 索引链、`1ca1608/1ca1630 flw -68(s0)` 与 `1ca163a fsw -68(s0)` volatile 往返
  - 应出现：pattern §Verification——「scalar compare/branch share 下降，出现对应 `vredmin*`/`vredmax*` 或 FP reduction，cycles/element 改善」；本函数出现 `vsetvli`、`vle32.v`/`vlse32.v`、`vfredmin.vs`（+NaN 检测 mask 路径）
  - 正确性边界：empty/one element、全相等、极值在首尾与 VLEN/tail 边界；NaN（任一元素 → NaN 结果）；±0（与标量 first-wins 对齐或显式记录差异）；seed=+inf；tail `vl` 正确
- **Supporting（no-vectorization）**：随 primary 验证——同一函数 annotate 出现 RVV 指令（pattern §Verification：「annotate 现在包含 RVV instructions」），scalar 指令不再主导 hot loop；与 scalar reference 对比覆盖小长度、main-loop 整倍数与全部 tail 长度
- **Supporting（Zfa `fminm.s`）**：scalar 兜底/tail 路径（若实现）出现 `fminm.s`（pattern §Verification：float min/max NaN propagation、±0、Inf、denormal 边界测试）；精确 `-march` 含 `zfa` 后重跑 annotate 验证分支链消失
- 性能验证：对同一函数重跑 annotate；对代表性短/中/长 `M` 与多输出规模做 benchmark（cycles/element）；C920v2 为 OoO，需实测确认 loop-carried 依赖消除收益不被 setup 抵消（core-profiles 提示 VLEN=128 时 per-iteration overhead 更可见）

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（载荷：`1/1 组；void mxnet::op::broadcast::seq_reduce_compute<mshadow::red::minimum, 2, ...>`） | ✅ 1/1 组；seq_reduce_compute<minimum,2,...> |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline；gap 标签 `baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） | ✅ 7 行；sampling metadata、sampling IP precision |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（primary）+ 2 supporting；primary evidence 锚点 `82.92 : 1ca1638 bnez`、`5.87 : 1ca162e beqz`、`3.55 : 1ca163e bne`、`1ca1604 div`、`1ca162a flw`、`1ca1608/1ca163a volatile`；supporting 2 行；排除 6 条；推导式 route High / impact Medium） | ✅ 8 项 trace；rows-operator-rvv.md, rows-codegen.md；1 primary + 2 supporting；锚点齐 |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern `rvv_extrema_reduction_kernels.md`（row: Extrema Reduction；引用「loop-carried scalar best-value accumulator 串行化 compare/update」+「NaN 或 signed zero 语义...额外分类、比较、mask 或标量特殊路径」）、`no-vectorization.md`、`isa_extension_specific_instruction_substitution.md`（row: ISA Extension-Specific Instruction Substitution；引用 §5 Hardware Float Min/Max）；`The fix` before/after、correctness、风险、Profile signals 齐；missing `.S` N/A；各 pattern Related PRs 条数 5/14/multi） | ✅ 3 个 pattern；The fix 完整；Related PRs 已列 |
| 5 | 路径合规：8 项 trace 可解释扫描集；primary/supporting 仲裁与 L1/L4 归属合规；每个 leaf 来自通过 gate 的 row；入口模式 A 按动态份额排序；`th.v*` 未全局停扫（载荷：模式 A + 路径 rows-operator-rvv/rows-codegen + class 列表） | ✅ A；rows-operator-rvv.md, rows-codegen.md |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3(a) 引用行（`1ca1638`、`1ca162e`、`1ca163e`、`1ca1604`、`1ca1608/1ca163a`）；出现侧标注 pattern §Verification（载荷：见 Phase 5 各条） | ✅ 锚点对齐 |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成、无契约外实施分支；无向用户追问；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测（载荷：本输出 Phase 0–6 结构） | ✅ 无越界 |

修正记录：无