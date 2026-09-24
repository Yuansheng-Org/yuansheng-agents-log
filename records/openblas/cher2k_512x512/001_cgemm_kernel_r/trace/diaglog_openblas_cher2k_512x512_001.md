Functions under analysis: [cgemm_kernel_r]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 perf annotate：已提供（`001-cgemm_kernel_r-annotate.txt`，5229 行，event=`cpu-clock`，31 samples，`percent: local period`；覆盖 prologue、8×8 主循环稳态 k-loop、N=8/4/2/1 尾段、C 更新 epilogue 与 return，完整无 gap）
- perf stat（bound/context）：已提供（`14-openblas-benchmark-riscv-cher2k_512x512.txt`：IPC=0.897337、L1D load miss rate=1.574%、branch miss rate=0.542%、GFLOPS=16.65068、duration_time=149,729,876ns、cpu_cycle=325,009,523、instruction=291,643,031、threads=1）
- workload/binary/source context：部分提供（DSO=`cher2k.goto`（BLAS benchmark 二进制）；annotate 内嵌 kernel C 源码行（`__riscv_v*` intrinsic，行 80–327 可见）；无 debug symbols / object mapping → `source_context_gap`（kernel 源文件路径未能从 annotate 解析，metadata `binaries` 为空）
- readelf -A（build ISA）：缺失（详见 Phase 1；批次与工作区均未提供 build_isa 文件）
- hardware ISA：已提供（metadata `cpuinfo.isa`：`rv64imafdcvh_zicbom_..._zve64d_zve64f_zvbb_zvbc_zfa_zfh_...`）
- vlenb：已提供（metadata：vlenb=32 → VLEN=256 bits）
- 采样元数据：部分提供（event=`cpu-clock`；annotate 头 `percent: local period`；同批 `raw/perf_report_no_children.txt` 给出函数级全局占比 42.47%（31/73 samples）→ 事件可解释为时间、同一窗口、函数级贡献已知；annotate 行内百分比为 local）
- Sampling IP precision：缺失（无 `precise_ip`/Exact-IP 信息，见 Phase 1）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：RVV 1.0（`v`）+ `zve64d`/`zve64f`/`zvbb`/`zvbc`/`zfa`/`zfh` 等；执行模型 out-of-order（metadata `instruction scheduling method`） |
| Build ISA | `baseline_gap: build ISA`（无 readelf -A 文件）；annotate 逐字显示 `vsetivli zero,8,e32,m1,ta,ma`、`vlseg2e32.v`、`vlseg8e32.v`、`vcompress.vm` 等 RVV 1.0 专属 mnemonic → 执行 binary 明确以 RVV 1.0 构建 |
| Vector flavor | RVV 1.0（`v*` mnemonic），与硬件一致 → 无 flavor mismatch |
| VLEN | 256 bits（vlenb=32）；e32,m1 = 8 lanes，恰好覆盖 kernel 固定 `gvl=8` 的 tile 宽度（`0x7222: vsetivli zero,8,e32,m1,ta,ma`） |
| Bound type | IPC 0.897（中低）；L1D load miss rate 1.574%（低）→ 非 cache-capacity/DRAM bound；branch miss 0.542%（低）→ 非 branch bound；k-loop 呈 issue/LSU 事务数受限（strided gather 把 1 条向量访存摊为 8 个元素访问） |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；annotate 内 percent=local period；同一运行窗口（annotate 与 report 同源 `raw/perf.data`）；函数级 workload 贡献已知（42.47%，31/73）→ 函数级收益可表述；行内百分比仅作局部份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 precise_ip/Exact-IP 信息）→ 单条指令高占比只锚定 hot interval，不做 instruction-latency/cost 归因 |
L0 baseline gates：hardware 有 `v`，执行 binary 为 `v*` mnemonic → 无 hardware/build mismatch；无 `th.v*` → vector-flavor gate 通过。Bound-type gate：perf stat counting 数据齐全，通过。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个函数）。hot loop 锚点：
- 主区间：8×8 主循环的稳态 k-loop `0x7354–0x7468`（28/31 = 90.3% 函数内样本）
- 最高行：`12.90 :  735c: vlse32.v v1,(t2),a4`（A0i strided gather，stride=8B，属于 k-loop）
- C 更新 epilogue 锚点：`6.45 :  74c8: vlse32.v v1,(a2),a4`、`3.23 :  74d0: vlse32.v v10,(t1),a4`（每 tile 执行一次；指令占比 ~0.8%，却承载 3/31 = 9.7% 样本）
- Sampling IP precision 未确认 → 全部归因收敛到 interval 级机制，不把单行占比升格为单指令 latency/cost。

## Phase 3 — Pattern scan / 模式扫描：cgemm_kernel_r
### Class selection trace（8 项）
1. `rows-vectorized-tuning.md` — include — 当前代码来源为 compiler-generated `__riscv_v*` intrinsic kernel（annotate 内嵌 intrinsic 源码行 + `v*` 指令，无 `.S` provenance），Step 0d 必选
2. `rows-operator-rvv.md` — include — annotate 内嵌源码行确认 interleaved complex 数据流（`vlse32(..., sizeof(FLOAT)*2, gvl)`、`VFMACC_RR/RI` 复数乘加、C 的 `[ci*2+0/1]` re/im 访问）
3. `rows-codegen.md` — include — cross-cutting 行需逐项排除（cache-aware blocking、register pressure、vector-state、kernel selection、induction variable、operand-form 等）
4. `rows-asm.md` — exclude — 无 source/DWARF/object mapping 证明 `.S`；代码来自 intrinsic 编译，非手写汇编；亦无 missing-`.S` 政策四证需求（kernel 已存在且已向量化）
5. `rows-string-memory.md` — exclude — 非 string/memory/copy/compare workload
6. `rows-crypto.md` — exclude — 无 AES/SHA/SM4 等 crypto primitive
7. `rows-offload.md` — exclude — 冻结 hardware ISA 中无矩阵引擎扩展（无 smme/xtheadmatrix 等），无 matrix-engine 卸载信号
8. `rows-runtime-os.md` — exclude — 用户态 benchmark kernel，非 RTOS/timer/privileged 路径

Classes scanned: `rows-vectorized-tuning.md`（6 行）、`rows-operator-rvv.md`（complex 相关行 + 互斥行）、`rows-codegen.md`（cross-cutting 行）

### Local performance pattern scan: `cgemm_kernel_r`
| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Complex Arithmetic Kernels（primary） | k-loop 内 interleaved complex 固定二字段 strided 访问 + 复数乘加主导（见三件套） | High | Medium | `patterns/rvv_complex_arithmetic_kernels.md` |
| RVV Floating-Point Matmul and GEMV Kernels（supporting） | 同一 k-loop 的 FP32 load/FMA 结构；packed complex microkernel 形态 | — | — | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |

#### (a) 逐字 evidence 引用（primary）
- `12.90 :  735c: vlse32.v v1,(t2),a4` — k-loop（0x7354–0x7468）内 A0i strided gather，函数内最高单行；对应源码行 `190  A0i = __riscv_vlse32_v_f32m1( &A[ai+0*gvl*2+1], sizeof(FLOAT)*2, gvl );`（byte-stride=8 = 1 个 complex element，`0x721a: li a4,8`）
- `3.23 :  7360: flw ft2,4(a2)`、`3.23 :  7364: flw ft3,0(a2)`、`6.45 :  7378: flw fa1,28(a2)`、`3.23 :  7384: flw fa5,52(a2)`、`3.23 :  7390: flw fs0,60(a2)`、`3.23 :  73d4: flw ft2,8(a2)` — 每 k-step 16 条 `flw`（B0..B7 的 re/im 标量）中的 6 条
- `6.45 :  73bc: vfmul.vf v6,v1,fa5`、`6.45 :  73c0: vfmul.vf v5,v2,fa5`、`6.45 :  73fc: vfmsac.vf v13,ft1,v1`、`6.45 :  7414: vfmsac.vf v7,fa2,v1`、`6.45 :  7420: vfmacc.vf v4,fa5,v2` — 复数乘加链（源码 `201 tmp0r = VFMACC_RR( tmp0r, B0r, A0r, gvl );`、`202 tmp0i = VFMACC_RI( tmp0i, B0r, A0i, gvl );`、`209 ACC0r = __riscv_vfadd( ACC0r, tmp0r, gvl );`）
- `6.45 :  74c8: vlse32.v v1,(a2),a4`、`3.23 :  74d0: vlse32.v v10,(t1),a4` — C 更新 epilogue 的 strided gather（源码 `246 vfloat32m1_t C0r = __riscv_vlse32_v_f32m1( &C[ci*2+0], sizeof(FLOAT)*2, gvl );`、`327 __riscv_vsse32_v_f32m1( &C[ci*2+1], sizeof(FLOAT)*2, C7i, gvl);`），每 tile 一次、指令占比 ~0.8% 却承载 9.7% 样本
- interval 对照：稳态 k-loop 内无任何 `vsetvl*`（vtype 由区间外 `0.00 :  7222: vsetivli zero,8,e32,m1,ta,ma` 一次性建立）；loop control 仅 `0.00 :  7428/7464: addi a2,a2,64 / addi a1,a1,64` + `0.00 :  7468: bne a0,a2,7354`（3/69 条指令，1/31 样本）

k-loop 每步（稳态 0x7354–0x7468，约 69 条指令）构成：2×`vlse32.v`（A0r/A0i，各 8 个 stride-8 元素访问）+ 16×`flw`（B 标量）+ 16×`vfmul.vf` + 16×`vfmacc/vfmsac.vf` + 16×`vfadd.vv` + 3×loop control。LSU 访问事务 ≈ 2×8+16 = 32，对照每步 64 个复数 MAC（384 flops）。

supporting because: 同一 k-loop 的访存/FMA 结构是 interleaved complex 布局合同的直接表达——packed panel 以 `(re,im)` 交织存储迫使 A 用 8-byte-stride gather、B 用 16 条标量 load；消除交织布局后该 load 形态即消失（同机制；其 §2/§3 的轴选择与 operand 投递讨论并入 primary 的 The fix）。

#### (b) 互斥邻居排除
- **Strided-layout row（`rvv_strided_layout_transform_kernels.md`）排除**：该行互斥判据要求"只有固定 stride 搬运、没有复数算术"；本 kernel 的 k-loop 是完整复数乘加（`vfmacc/vfmsac` 成对、共轭/alpha 符号处理），stride-8 访问是复数算术的数据投递形态，归属 complex-arithmetic 行
- **Gather-indexed row（`rvv_gather_indexed_memory_access.md`）排除**：stride 固定为常量 8（`0x721a: li a4,8`），非 data-dependent 索引（非 LUT/idx[i] 形态）
- **No-vectorization row 排除**：hot main loop 全 `v*`，硬件与 binary 均 RVV 1.0
- **Register-group/LMUL row（`rvv_register_group_utilization.md`）排除**：峰值 live set ≈ 16 ACC + 2 A + ~4 tmp ≈ 22 个 m1 组；按 `references/kernel-conventions.md` §2（`LMUL*peak_live_vectors ≤ 32`），m2 需 ≥ 44 个寄存器 → 不合法；m1 即给定 tile 结构的合法边界，无 mismatch；k-loop 内无 vector spill/reload（相关行全部 0.00%）
- **Vector-state row 排除**：稳态 k-loop 无任何 `vsetvl*`；`vsetvli`（e8,mf4 等）只出现在冷/尾段（0x71dc、0x7202、0x7d4e、0x833c、0x8702 等，均 0.00%）
- **Register-budgeted-unroll row 排除**：16 个独立 accumulator（ACC0..7r/ACC0..7i）已提供充足 ILP；loop-control 仅 3/69 条、1/31 样本，不构成"backward branch/counter 或单 accumulator 串行 recurrence 主导"
- **Operand-form row 排除**：B 标量经 `flw` 直接由 `.vf` 指令消费（`vfmul.vf/vfmsac.vf` 的 rs1 为 FP reg），正是期望的 broadcast operand-form；hot loop 内无 `vmv.v.x/vfmv.v.f` temporary
- **Cache-aware-blocking row 排除**：L1D load miss rate 仅 1.574%；无 tile-residency/尺寸拐点证据；k-loop 代价集中在 gather 的访问事务数而非 cache miss 容量
- **Register-pressure/save-restore row 排除**：528B frame、12×s + 12×fs 的 save/restore、epilogue 的 `vmv1r.v` 全部 0.00%（冷区间）
- **Kernel-selection row 排除**：正确 kernel（cgemm_kernel_r）已被选中并执行（42.47% 样本落入其中）
- **Autovec-perturbation row（`rvv_intrinsic_kernel_autovectorization_control.md`）排除**：annotate 中 intrinsic 源码行与 `v*` 指令 1:1 对应，无 compiler 插入的冗余 `vsetvli`、无 kernel budget 外 vector spill

#### (c) 双 Confidence 推导式
- route: annotate 内嵌源码行确认 interleaved complex 数据流 + 固定二字段 strided 访问 + 复数 FMA 链 + 硬件/binary 均 RVV 1.0（直接 provenance，语义互斥排除完整）→ **High**
- impact: VLEN 已知（256）、bound type 已知（issue/LSU 事务受限、L1D miss 低）、函数级 share 已知（42.47%）、采样语义四条件基本满足；但 `baseline_gap: sampling IP precision` + `baseline_gap: build ISA`（缺 readelf）+ C 更新 lane 映射未从源码逐行确认 → **Medium**

### 多命中仲裁小段
- primary：RVV Complex Arithmetic Kernels（L1 semantic；evidence = k-loop 访存行 12/31 + C 更新 strided 行 3/31 = **15/31 ≈ 48.4% 局部份额**；k-loop 区间整体 90.3% 局部）
- supporting：RVV Floating-Point Matmul and GEMV Kernels（同 interval、同机制族；因果消除测试：消除 interleaved 布局后 load 主导信号即消失 → 上层布局/数据流变更认领 primary；其 §2/§3 独有内容并入 primary 的 The fix）
- 样本份额排序（入口条件 A，动态份额可用）：primary 直接证据行 48.4% 局部 → k-loop 区间内主导机制；C 更新 strided 行 9.7% 局部同属该机制族
- L 层次归属：stride-8 gather 数据投递 → L2（data movement）；复数 FMA 链结构本身健康 → 不另列 L4 finding

## Phase 4 — Root-cause blueprint / 根因蓝图：cgemm_kernel_r
（纳入蓝图的 primary row：RVV Complex Arithmetic Kernels（Phase 3 已过 gate）；supporting：RVV Floating-Point Matmul and GEMV Kernels）

1. **Root cause**：k-loop（0x7354–0x7468）把 complex packed panel 以 `(re,im)` 交织（AoS）布局喂给复数乘加链，迫使：(a) A0r/A0i 用 8-byte-stride `vlse32` gather——函数内最高单行 `12.90 :  735c: vlse32.v v1,(t2),a4`；(b) B 的 re/im 用 16 条 `flw` 标量 load/步。每个 k-step 的 LSU 访问事务 ≈ 32（2×8 + 16），而有效计算仅 48 条向量 FP 指令（64 个复数 MAC = 384 flops）。C 更新 epilogue 同样以 stride-8 `vlse32/vsse32`（16 load + 16 store/tile）访问 `C[ci*2+0/1]`，~0.8% 指令占比承载 9.7% 样本，进一步证实 strided 访问在该核上的高成本。bound 判断：L1D miss rate 1.574%（低）→ 瓶颈不是 cache 容量而是 LSU 事务数/issue 带宽。依据 `patterns/rvv_complex_arithmetic_kernels.md` §Why this is slow："interleaved 布局若逐标量解包会浪费向量带宽，反之盲目使用高 NFIELD segmented 指令也可能增加目标相关的执行代价和 register-group 压力"；§The fix 框架："让 real/imag 在同一 strip-mined traversal 中共同存活，使用独立 accumulator 隐藏依赖"。
2. **The fix / 修复方式**：
   - 方案 A（结构性，推荐）：在 packing 层（对应 OpenBLAS 复数 pack/copy 路径，如 cgemm_otcopy/ntcopy 等价物）把 A/B panel 改为 split-complex（SoA）布局——A_re/A_im 各自连续、B_re/B_im 各自连续；kernel k-loop 改用 2×`vle32`（A_re、A_im）+ 2×`vle32`（B_re、B_im），标量操作数经 `vfmv.f.s` 提取后继续 `.vf` FMA（FMA 结构与顺序不变），每步 LSU 事务从 ~32 降至 ~6–8。前置条件：packing 与 kernel 消费顺序严格一致；pack 成本不变（搬运字节数相同，可摊薄）。
   - 方案 B（最小侵入，不改 packing）：A 的 2×`vlse32` 换 1×`vlseg2e32.v`（8 对 complex → A_r/A_i 两个 m1 寄存器，单位步长访问 64B）；C 的 16×`vlse32/vsse32` 换 8×`vlseg2e32.v`/`vsseg2e32.v`；B 保持 `flw` 或同样 `vlseg2e32` + 提取。注意 pattern 对 NFIELD 执行代价的警告，需实机 A/B 验证。
   - 修复前后伪代码：
     ```
     // Before（interleaved complex panel，每 k-step）：
     A0r = vlse32(&A[ai+0], stride=8, gvl);   // 8 元素 gather
     A0i = vlse32(&A[ai+1], stride=8, gvl);
     B0r..B7r, B0i..B7i = 16×flw;
     tmp0i = vfmul.vf(A0r, B0i); tmp0r = vfmul.vf(A0i, B0i); ... // 16 vfmul
     tmp0r = vfmacc.vf(tmp0r, B0r, A0r); tmp0i = vfmsac.vf(tmp0i, B0r, A0i); ... // 16 FMA
     ACC0r = vfadd.vv(ACC0r, tmp0r); ... // 16 vfadd
     // After（split-complex panel，每 k-step；FMA 结构不变）：
     A0r = vle32(&A_re[k*gvl]);  A0i = vle32(&A_im[k*gvl]);      // unit-stride
     B_re = vle32(&B_re[k*gvl]); B_im = vle32(&B_im[k*gvl]);
     B0r = vfmv.f.s(B_re, 0); ... 8+8 次提取 → 16 个标量
     16 vfmul + 16 FMA + 16 vfadd（FMA 分组与累加顺序不变）
     ```
   - correctness contract（不可破坏）：Hermitian 共轭符号（B^H 虚部取反，`VFMACC_RI`/`vfmsac` 的符号语义）、alpha 缩放（`vfmacc/vfnmsac` with fa0/fa4）、re/im 的 lane 语义、FMA 分组与累加顺序（方案 A/B 均不改 FMA 结构 → FP 舍入行为不变）；packing 布局与 kernel 读取顺序严格一致（本 pattern 最易出错处）。
   - 限制/风险：方案 A 改动 packing 层 + kernel 两层，blast radius 更大；方案 B 的 `vlseg2` 在 X100 上的每元素执行代价未知，需实测；两方案均需在 OoO（X100）与其它的 VLEN 上确认不回退（按 `references/kernel-conventions.md` §2 保持 `LMUL*peak_live ≤ 32`；tile 结构不变则 LMUL=1 合法、m2 不合法）。
   - 修复后预期 Profile signals：k-loop 内 `vlse32.v` 与 `flw` 行样本消失/骤降，出现 `vle32`/`vlseg2e32`；C 更新 `vlse32/vsse32` 行消失；IPC 上升、k-step 周期数下降、GFLOPS 提升。
3. **Baseline facts 回填**：hardware ISA = RVV 1.0 + zve64d/zve64f/zvbb/zvbc/zfa/zfh（SpacemiT X100，OoO）；build ISA = `baseline_gap: build ISA`（annotate mnemonic 证明执行 binary 为 RVV 1.0）；VLEN = 256 bits（vlenb=32）；bound type = issue/LSU 事务受限（IPC 0.897、L1D miss 1.574%）。
4. **收益上界**：primary finding 的直接 evidence 行 = k-loop 访存行（12/31）+ C 更新 strided 行（3/31）= **15/31 ≈ 48.4% 局部样本份额**（当前 sampled event=cpu-clock 下的局部份额；采样语义四条件中同一窗口与函数级贡献成立，可进一步折算：48.4% × 42.47% ≈ 20.6% 的 workload 级上限，且因 k-loop 区间整体占 90.3% 局部、LSU 压力释放后区间整体受益，实际可达上界更高；按契约只对直接证据行加总）。
5. **三维路由判定**：current source = compiler-generated `__riscv_v*` intrinsic kernel（非 `.S`、非 JIT，annotate 内嵌源码行为证）；implementation existence/reachability = kernel 已存在、已选中、已执行（42.47% 样本落入）；function-level policy = 无 OpenBLAS 复数 Level-3 kernel 强制独立 `.S` 的政策证据 → 不进入 policy-backed missing-`.S` 分支，修复留在 intrinsic kernel + packing 层。
6. **Related PRs 小节**：
   - `patterns/rvv_complex_arithmetic_kernels.md`：Related PRs：3 条 URL
     - https://github.com/OpenMathLib/OpenBLAS/commit/d3bf5a5401e623e107a23fb70151c7102cbd14c7
     - https://github.com/OpenMathLib/OpenBLAS/commit/18d7afe69daa196902cd68b63cc381aaafc9d26e
     - https://github.com/OpenMathLib/OpenBLAS/commit/63cf4d01668f8f6c73a05039bc36785ba78b0940
   - `patterns/rvv_floating_point_matmul_and_gemv_kernels.md`（supporting）：Related PRs：37 条 URL
     - https://github.com/OpenMathLib/OpenBLAS/commit/0a967797a15617239523053633bf14be7895b25a
     - https://github.com/OpenMathLib/OpenBLAS/commit/0acb60aab3c0134e879a68292904d8346dcd50ef
     - https://github.com/OpenMathLib/OpenBLAS/commit/1cc377ef61d498b75c852aa4b9b042fe9422c347
     - https://github.com/OpenMathLib/OpenBLAS/commit/2d82d144e2791e37d7a314237b638d85b156a2ec
     - https://github.com/OpenMathLib/OpenBLAS/commit/809e1cba8f1f3f89972581e8b82f2ec52e51eadb
     - https://github.com/OpenMathLib/OpenBLAS/commit/376d3a138faa0a0fe483a8fa8d4fa1ab0d395acf
     - https://github.com/OpenMathLib/OpenBLAS/commit/2ae019161a85333a35018b517d4b34474a7694e9
     - https://github.com/uxlfoundation/oneDNN/commit/d6f82a2d0d0db41e6daaf20fbb4fd352843aac64
     - https://github.com/ggml-org/llama.cpp/pull/17318
     - https://github.com/ggml-org/llama.cpp/pull/17448
     - https://github.com/ggml-org/llama.cpp/pull/17314
     - https://github.com/ggml-org/llama.cpp/pull/17161
     - https://github.com/alibaba/MNN/pull/4426
     - https://github.com/ggml-org/llama.cpp/pull/18199
     - https://github.com/ggml-org/llama.cpp/pull/20627
     - https://github.com/uxlfoundation/oneDNN/commit/3bac96b8bc1fc9c348c986f38f65285693943d2f
     - https://github.com/uxlfoundation/oneDNN/commit/8b48a77091062ce78959ccb96e43a8ee4e97022d
     - https://github.com/uxlfoundation/oneDNN/commit/8c52facbe61845d86062c76280b4bc515160c03e
     - https://github.com/uxlfoundation/oneDNN/commit/b73fc3172d3e3230cf24ae29cbb6a07a09507a43
     - https://github.com/uxlfoundation/oneDNN/commit/bd984d09dc5985a19fb427ac46d19d2cbd5558dd
     - https://github.com/uxlfoundation/oneDNN/commit/d2a44b9b855706a0df33f9b6d4fb84f5420fdeaf
     - https://github.com/uxlfoundation/oneDNN/commit/d6107ddb8be72041dade165a233c0de69f7a1387
     - https://github.com/uxlfoundation/oneDNN/commit/fe04323ab0b4bba79ee60109fc391bb36052e43c
     - https://github.com/uxlfoundation/oneDNN/pull/4410
     - https://github.com/uxlfoundation/oneDNN/pull/4414
     - https://github.com/uxlfoundation/oneDNN/pull/4545
     - https://github.com/uxlfoundation/oneDNN/pull/4620
     - https://github.com/uxlfoundation/oneDNN/pull/4770
     - https://github.com/uxlfoundation/oneDNN/pull/4824
     - https://github.com/uxlfoundation/oneDNN/pull/4840
     - https://github.com/uxlfoundation/oneDNN/pull/4850
     - https://github.com/uxlfoundation/oneDNN/pull/4945
     - https://github.com/uxlfoundation/oneDNN/pull/5157
     - https://github.com/uxlfoundation/oneDNN/pull/5294
     - https://github.com/uxlfoundation/oneDNN/pull/5403
     - https://github.com/uxlfoundation/oneDNN/pull/5405
     - https://github.com/vllm-project/vllm/pull/44324

## Phase 5 — Verification forecast / 验证预测：cgemm_kernel_r
- **消失侧锚定（对 Phase 3(a) 引用行）**：修复后同函数 annotate 中以下行应消失或明显缩小：`735c: vlse32.v v1,(t2),a4`、`7360/7364/7378/7384/7390/73d4: flw`、`74c8: vlse32.v v1,(a2),a4`、`74d0: vlse32.v v10,(t1),a4`；k-loop 区间样本集中度从 ~90.3% 下降。
- **出现侧锚定（`patterns/rvv_complex_arithmetic_kernels.md` §Verification）**：annotate 应显示 real/imag 在同一 vector loop 中共同存活、重复 traversal 或 gather 次数下降；对 unit-stride、strided、二字段 segmented（vlseg2e32）三种方案实测，检查 LMUL/NFIELD 合法性与 spill；对 Hermitian 路径构造非零虚部数据暴露共轭符号错误；覆盖 n=0/1、tail、未对齐、interleaved/split-complex、纯实数/纯虚数输入；覆盖 NaN/Inf/subnormal/±0 并声明 FP 重排容差。
- **正确性合同**：方案 A/B 不改 FMA 分组与累加顺序 → 预期 bit-identical 或容差内一致；若采用 `.vv` broadcast 形态（改变 FMA 结构）必须声明 FP reassociation 容差。
- **性能验证**：对同一函数重跑 annotate + benchmark（cher2k_512x512）；对比 IPC（基线 0.897）与 GFLOPS（基线 16.65）；按 `references/kernel-conventions.md` §Verification 覆盖 K 的短/中/长；在 OoO（X100）与其它的 VLEN 核上确认不回退。
- **补采数据（升级到完全 profile-backed 所需）**：`perf record -e cpu-clock:P` 或 `perf evlist -v` 核对 `precise_ip`（消解 `baseline_gap: sampling IP precision`）；对 `cher2k.goto` 执行 `readelf -A`（消解 `baseline_gap: build ISA`）。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；[cgemm_kernel_r] |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline（含 Sampling IP precision 行）+ 2 个 L0 gate + bound-type gate；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision` |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；`Classes scanned:` rows-vectorized-tuning.md、rows-operator-rvv.md、rows-codegen.md；顶层 finding 1（RVV Complex Arithmetic Kernels）+ supporting 1（RVV Floating-Point Matmul and GEMV Kernels）；evidence 锚点：`12.90 :  735c: vlse32.v`、`6.45 :  7378: flw`、`6.45 :  73fc: vfmsac.vf`、`6.45 :  74c8: vlse32.v`、`3.23 :  74d0: vlse32.v`；排除 12 条；推导式 2 组 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：rvv_complex_arithmetic_kernels.md（命中 row：RVV Complex Arithmetic Kernels）+ rvv_floating_point_matmul_and_gemv_kernels.md（supporting）；引用短语："interleaved 布局若逐标量解包会浪费向量带宽"、"让 real/imag 在同一 strip-mined traversal 中共同存活"；The fix 含 before/after 伪代码、correctness、风险、Profile 信号；Related PRs：3 + 37 条 URL |
| 5 | 路径合规 | ✅ | 入口模式 A（profile_backed）；8 类扫描集；primary+supporting 仲裁；leaf 均来自过 gate row；动态份额排序（48.4% local） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`735c: vlse32.v`、`74c8: vlse32.v`、`73d4: flw` 等；出现侧：rvv_complex_arithmetic_kernels.md §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改/补丁生成、无追问；交付物止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 |

修正记录：无