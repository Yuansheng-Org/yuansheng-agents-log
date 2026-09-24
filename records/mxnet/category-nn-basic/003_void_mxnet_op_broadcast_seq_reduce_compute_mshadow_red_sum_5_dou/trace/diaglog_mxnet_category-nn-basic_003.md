Functions under analysis: [`void mxnet::op::broadcast::seq_reduce_compute<mshadow::red::sum, 5, double, float, float, mxnet::op::mshadow_op::identity, mxnet::op::mshadow_op::set_index_no_op<double, long> >(...) [clone ._omp_fn.0]`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`/data/perfdata/mxnet-rv64-sg2044-c/category-nn-basic/annotate/003-...-fe85bb1be3bf-annotate.txt`，header `cpu-clock (15123 samples, percent: local period)`，含热点 `13bc9b0`–`13bca3c`）
- perf stat（可选 bound/context）：已提供（testcase 级共享文件 `11-mxnet-opperf-benchmark-riscv-category-nn-basic.txt`；为整个 category 的计数，非本函数专属）
- workload/binary/DSO/source context：已提供（DSO `libmxnet.so`；仓库 `apache/mxnet@b84609d3fc73d20929c114eab95faaa56e6c5ede` 即当前工作区，源码 `src/operator/tensor/broadcast_reduce-inl.h:349-390`（`seq_reduce_assign` 串行 k 循环）、`src/operator/mxnet_op.h`（`unravel<5>`/`dot<5>`）、`3rdparty/mshadow/mshadow/base.h:987-1005`（`mshadow::red::sum::Reduce`）可绑定该 symbol）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `binaries["libmxnet.so-elf-A"]`）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（hw profile `sg2044`）
- `vlenb`：已提供（16 bytes → VLEN = 128 bit）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=`cpu-clock`；percent type=`local period`；scope 与采样窗口未记录；函数级 workload 贡献未知）
- Sampling IP precision（`precise_ip` / Exact-IP / PMU skid）：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 有 `v`：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`（hw profile sg2044 / metadata `cpuinfo.isa`）；含 `zve32f`、`zve64f`、`zve64d` |
| Build ISA | `Tag_RISCV_arch: "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0"`（metadata `binaries["libmxnet.so-elf-A"]`）；**object 已含 `v1p0` + `zve32f/zve64f/zve64d` + `zvl128b`**（annotate 内确见编译器生成的 `vsetvli a5,a4,e8,m1`/`vle8.v`/`vse8.v`，用于 40 字节 `Shape<5>` 拷贝）。函数级 provenance 由 symbol 与源码绑定：`seq_reduce_compute`（`broadcast_reduce-inl.h:445`）→ `seq_reduce_assign`（`:349`）→ `mshadow::red::sum::Reduce`（`base.h:993`），非 IFUNC/multiversion |
| Vector flavor | 本热点区间内**除编译器生成的 `Shape<5>` 字节拷贝外无浮点/算术 `v*`**；无 vendor `th.v*`；无 flavor mismatch |
| VLEN | `vlenb`=16 → **VLEN = 128 bit**（hw profile `vector.vlen_bits=128`，metadata 一致） |
| Bound type | 由 testcase 级 perf stat：`IPC 0.567926`（低）、`L1_dcache_load_miss_rate 0.974%`、`LLC_load_miss_rate 22.822%`、`branch_miss_rate 0.960%`、`duration_s 1286.224210` → 归为 **compute/latency-bound（每元素多维索引计算主导）**。注意该计数覆盖整个 category，不是本函数专属 |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=`local period`（**非** global-period）；scope/采样窗口未记录；该函数占整个 workload 的贡献未知 → **四项不齐**，只能表述为「本函数内局部样本份额」，禁止称 cycle/workload 耗时份额 |
| Sampling IP precision | 未提供 `precise_ip`/Exact-IP 信息，目标 PMU 对该 event 的 skid 能力未知 → `baseline_gap: sampling IP precision`；可选命令 `perf report --header-only` / `perf evlist -v` 核对 `precise_ip` |

L0 baseline gate：
- **hardware 有 `v`、build 也有 `v`** → 无 hardware/build mismatch finding。
- **本热点无 `th.v*`** → 无 `vector_flavor_mismatch`；flavor-dependent route 不冻结。
- annotate 已提供且覆盖 hot loop，不走入口条件 D。

Bound-type gate：结论为 compute/latency-bound（非 memory-bound），**不触发 memory-bound gate**；但 `baseline_gap: sampling metadata` 与 `baseline_gap: sampling IP precision` 仍限制 impact 等级（见 Phase 3(c)）。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数）。hot basic block / loop interval 边界（源码 `src/operator/tensor/broadcast_reduce-inl.h:349-390`，ndim=5 变体）：

- **索引计算区间：`13bc9b0` – `13bca06`**（每轮 reduce 元素一次）—— 含 `unravel<5>`（5 次 `div`/`rem` + 5 次栈 store 到 `coord`，回边 `bne a1,a5,13bc9ba`）、**编译器生成的 40 字节 `Shape<5>` 拷贝**（`vsetvli a5,a4,e8,m1` + `vle8.v`/`vse8.v` 循环，`sub a4,a4,a5` + `add a3,a3,a5` + `bnez`）、`dot<5>`（5 次 `mul`/`add`，回边 `bne a5,a2,13bc9f6`）。样本合计 **≈81.3%**；最高行 `25.87 : 13bc9e2: sub a4,a4,a5`。
- **归约区间：`13bca0a` – `13bca3c`** —— 地址成形（`add a3,a3,a6` / `slli a3,a3,0x2` / `add a3,a3,t3`）、`flw` 载入、`fcvt.d.s` 加宽、补偿求和（`fld`×2 / `fsub.d` / `fadd.d`）、`fclass.d` + `andi 0x81` + `beqz` 的 Inf 守卫、`fsd`×2 写回 volatile 累加器。样本合计 **≈17.1%**。
- 区间外（`13bc95a`–`13bc9ae` 的外层 idx 循环与 prologue、`13bca40`–`13bca62` 的 `assign` 收尾）占比均 <1%。
- 最高占比行（trace anchor）：`25.87 :   13bc9e2:        sub     a4,a4,a5`。
- Sampling IP precision 未确认（`baseline_gap: sampling IP precision`）→ **不允许把 `sub a4,a4,a5` 的 25.87% 单独归因为该指令的 latency/cost**；该行只锚定索引计算区间。区间级机制成立：**每个 reduce 元素都要重新做一次 5 维索引分解（5 div + 5 rem）、一次 40 字节 `Shape<5>` 结构拷贝和一次 5 元素点积**，而实际内存访问只有 1 次 `flw`——索引计算成本远高于访问本身。

## Phase 3 — Pattern scan / 模式扫描：`mxnet::op::broadcast::seq_reduce_compute<mshadow::red::sum, 5, double, float, float, identity, set_index_no_op<double,long>>`

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：无 `.S` source/DWARF/object-mapping 证据；symbol 与源码绑定到 `broadcast_reduce-inl.h:349` 的模板 C++ 函数，mxnet 对该路径无函数级 assembly policy。
2. `rows-operator-rvv.md` — **include**：归约区间是 compiler-generated、单输入、固定 stride 序列上的标量加性归约（`mshadow::red::sum`，widening float→double + Inf 守卫），硬件有 `v`、build 有 `v`；同时需评估其它 semantic row 的认领关系。
3. `rows-string-memory.md` — **exclude**：无 copy/fill/sentinel/compare/checksum/back-reference；target 非 glibc/libc。（区间内虽有编译器生成的 `vle8.v`/`vse8.v` 40 字节拷贝，但它是 `Shape<5>` 结构物化的副产物，不是批量内存契约。）
4. `rows-vectorized-tuning.md` — **exclude**：热点内无浮点/算术 `v*`（唯一 `vsetvli`/`vle8.v`/`vse8.v` 属编译器生成的字节拷贝）；无 LMUL/register-group/vtype 可 tuning。
5. `rows-codegen.md` — **include**：索引计算区间是每轮重复的 `unravel<5>`/`dot<5>` + `Shape<5>` 物化，属循环级索引演化问题；需逐行评估 induction-variable、register-pressure、algebraic-simplification、control-flow、kernel-selection 等 row。
6. `rows-offload.md` — **exclude**：目标 ISA line 无矩阵引擎、无 packed-SIMD（P/DSP）扩展；热点非 GEMM、无 weight-repack/portable-layer 证据。
7. `rows-crypto.md` — **exclude**：profile 未点名任何密码学原语。
8. `rows-runtime-os.md` — **exclude**：热点在用户态算子 kernel，不在 timer/ISR/PMP/CSR 路径。

Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: `seq_reduce_compute<mshadow::red::sum, 5, double, float, float, identity, set_index_no_op<double,long>>`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Loop Induction Variable Strength Reduction（primary） | 索引计算区间 `13bc9b0`–`13bca06` 占 **≈81.3%** 样本：`unravel<5>` 每轮 5 次 `div`/`rem`（`rem a4,s5,s6` 0.77、`div s5,s5,s6` 0.00）与 5 次 `coord` 栈写（`sd a4,8(a3)` 15.06、`addi a3,a3,-8` 0.79、回边 `bne a1,a5,13bc9ba` 17.75）；**每轮一次 40 字节 `Shape<5>` 结构拷贝**（`vsetvli a5,a4,e8,m1` + `vle8.v`/`vse8.v` 循环：`sub a4,a4,a5` 25.87、`add a3,a3,a5` 8.37、`bnez a4` 0.77）；`dot<5>` 每轮 5 次 `mul`/`add`（`ld a4,0(a5)` 1.55、`ld s6,0(s5)` 0.75、`addi s5,s5,8` 1.62、`mul a4,a4,s6` 1.50、回边 `bne a5,a2,13bc9f6` 5.69）。而实际访问只有 `flw fa5,0(a3)`（2.43），索引计算远重于访存 | High | Medium | `patterns/loop_induction_variable_strength_reduction.md` |
| RVV Widening Additive Reduction Kernels（independent） | 归约区间 `13bca0a`–`13bca3c` 占 **≈17.1%** 样本：`flw fa5,0(a3)`(2.43) → `fcvt.d.s`(0.00) → `fld fa3,-160(s0)`(3.09)/`fld fa4,-168(s0)` → `fsub.d`/`fadd.d` → `fclass.d a5,fa4`(6.10) + `andi a5,a5,129` + `beqz a5,13bca82`(4.57)（`isinf(t)` 守卫）→ `fsd fa2,-160(s0)`/`fsd fa4,-168(s0)`(0.87)（volatile 累加器 `dst`/`residual` 栈往返）；累加器为循环携带依赖，float→double 加宽，无 RVV 归约载体 | High | Medium | `patterns/rvv_widening_reduction_kernels.md` |
| （excluded）RVV Precision Conversion Kernels | 区间内确有 `fcvt.d.s`，但该 row 行内 gate 要求「核心工作是数值转换 …… 而非主算子计算」；本处转换占比 ≈0%，核心是加性归约，故不归该 row | — | — | `patterns/rvv_precision_conversion_kernels.md` |
| （excluded）RVV Normalization / Extrema / Arg-Extrema / Matmul / FFT / Fixed-Size-Separable | 本处是 `identity` 映射后的纯 sum 归约，无跨 lane 统计 + 变换的完整算子、无 Min/Max 或 value+index、无 M/N/K 嵌套或蝶形网络 | — | — | `rows-operator-rvv.md` 对应行 |
| （excluded）No vectorization | 索引与归约区间均无用户级 `v*`，但该 row 行内自述「本行只认领未被更具体 semantic/codegen/assembly row 解释的 generic scalar main loop」——两处语义已分别被 induction-variable 与 widening-reduction row 认领 | — | — | `patterns/no-vectorization.md` |
| （excluded）rows-codegen 其余相关 row | `Register Pressure and Save/Restore`：`coord`/`dst`/`residual` 的栈驻留来自 `Shape<5>` 按值返回/物化与 `volatile` 形参，不是 RA spill，故不归 RA row（属两个 primary/independent 的机制证据）；`Algebraic Simplification`：补偿求和链与 Inf 守卫不可代数化简；`Control-Flow Layout and Transfer Selection`：`bne` 均为普通计数循环回边，无 jump-table/RAS/relocation 证据；`Resource-Aware Instruction Scheduling`：无目标核 PMU/dependency 证据且 `baseline_gap: sampling IP precision` 禁止单指令 stall 归因；`Kernel Selection`：仓库内不存在该 op 的 RVV/专用实现（全仓 grep 无 riscv/RVV 引用） | — | — | `rows-codegen.md` 对应行 |

### 三件套（primary finding：Loop Induction Variable Strength Reduction）

**(a) 逐字 evidence 引用**（所属索引计算区间 `13bc9b0`–`13bca06`）：

```
    25.87 :   13bc9e2:        sub     a4,a4,a5
    17.75 :   13bc9cc:        bne     a1,a5,13bc9ba
    15.06 :   13bc9c6:        sd      a4,8(a3)
     8.37 :   13bc9ea:        add     a3,a3,a5
     5.69 :   13bca06:        bne     a5,a2,13bc9f6
     1.62 :   13bc9fe:        addi    s5,s5,8
     1.55 :   13bc9f6:        ld      a4,0(a5)
     1.50 :   13bca00:        mul     a4,a4,s6
     0.77 :   13bc9c2:        rem     a4,s5,s6
     0.79 :   13bc9c0:        addi    a3,a3,-8
```

源码锚点（同区间，annotate 行内交错显示）：`365 coord      = mxnet_op::unravel(k, rshape);`、`682 for (index_t i = ndim - 1, j = idx; i >= 0; --i)`、`683 auto tmp = j / shape[i];`、`684 ret[i]   = j - tmp * shape[i];`、`366 AType temp = OP::Map(big[j + mxnet_op::dot(coord, rstride)]);`、`696 ret += coord[i] * stride[i];`（`src/operator/tensor/broadcast_reduce-inl.h` 的 `seq_reduce_assign` 与 `mxnet_op::unravel<5>`/`dot<5>`）。`baseline_gap: sampling IP precision` 未确认，故 `sub a4,a4,a5` 的 25.87% 只作为区间内主导行，不单独承担 instruction-latency 根因。

**(b) 互斥邻居排除**（逐行判据点名的相邻 row）：

- **Loop Induction Variable Strength Reduction 自身互斥项**：该 row 行内排除「peephole 级单指令代数恒等式化简」与「`li reg,0`/`SLT`/`SNEZ` 类 zero-register 比较形态」；本 interval 是循环级的每轮索引重建（5 div/rem + 40 字节结构拷贝 + 5 元素点积），既非单指令恒等式也非零比较寄存器优化，故正确归本 row。
- **RVV Widening Additive Reduction**：该 row 认领归约算术与累加器；本 interval 无任何 FP 累加指令（`fadd.d`/`fsub.d` 全部落在 `13bca20`/`13bca24`，属另一区间），只有整数 div/rem/mul/add 与字节拷贝，故不归该 row。
- **RVV Precision Conversion**：本 interval 无 `fcvt.*`（唯一的 `fcvt.d.s` 在 `13bca1c`），故不归该 row。
- **Register Pressure and Save/Restore**：`coord` 的栈写入（`sd a4,8(a3)`）不是 RA spill，而是 `unravel<5>` 按值返回 `Shape<5>` 后的结构物化与拷贝；`dst`/`residual` 的栈驻留来自 `volatile` 形参——两者都不是分配器 spill，故不归 RA row。
- **Control-Flow Layout and Transfer Selection**：`bne a1,a5,13bc9ba`（unravel 的 5 次迭代）与 `bne a5,a2,13bc9f6`（dot 的 5 次迭代）与 `bnez a4,13bc9da`（40 字节拷贝的 3 次迭代）都是普通计数循环回边，无 jump-table/RAS/relocation/fallthrough 语义，故不归该 row。
- **Algebraic Simplification**：本 interval 无 `NEG`+`MUL`/冗余 `SEXT.W`/同 operand 运算等可删恒等式，故不命中。
- **No vectorization**：其行内 gate 要求「未被更具体 semantic/codegen/assembly row 解释」，本 interval 已被 induction-variable row 认领，故该 row 不成立（不是 supporting）。
- **Resource-Aware Instruction Scheduling**：该 row 要求 annotate 与目标 core PMU/资料共同指向 load-use / resource contention / 长 dependency stall；`baseline_gap: sampling IP precision` 下 25.87% 不能归因到单指令 stall，缺少判别性 PMU 证据，故不命中。

**(c) 双 Confidence 推导式**：

```
route: compiler-generated 模板 C++（broadcast_reduce-inl.h:349 seq_reduce_assign 的 for k 循环
       + mxnet_op::unravel<5>/dot<5> 按值返回与 5 维分解）
       + 源码逐字绑定「每轮重建 5 维索引 + 40 字节 Shape<5> 物化 + 5 元素点积，而实际访问仅 1 次 flw」
       + 硬件有 v + build 有 v1p0/zve32f/zve64f/zve64d
       + 行内互斥判据逐条排除相邻 row（见 (b)）→ High
impact: VLEN=128 已知、bound type=compute/latency-bound（IPC 0.568）已知、区间局部份额 ≈81.3%；
        但 baseline_gap: sampling metadata（percent type=local period、函数级 workload 贡献未知）
        + baseline_gap: sampling IP precision 两项未消解 → Medium
```

### 三件套（independent finding：RVV Widening Additive Reduction Kernels）

**(a) 逐字 evidence 引用**（所属归约区间 `13bca0a`–`13bca3c`）：

```
     6.10 :   13bca28:        fclass.d        a5,fa4
     4.57 :   13bca30:        beqz    a5,13bca82
     3.09 :   13bca14:        fld     fa3,-160(s0)
     2.43 :   13bca10:        flw     fa5,0(a3)
     0.87 :   13bca36:        fsd     fa4,-168(s0)
     0.00 :   13bca24:        fadd.d  fa4,fa5,fa4
     0.00 :   13bca20:        fsub.d  fa5,fa5,fa3
```

源码锚点：`996 DType y = src - residual;`、`997 DType t = dst + y;`、`998 if (isinf_typed::IsInf(t)) {`、`999 residual = 0;`、`1003 dst = t;`（`3rdparty/mshadow/mshadow/base.h` 的 `mshadow::red::sum::Reduce`，形参为 `volatile DType& dst, volatile DType src, volatile DType& residual`）；调用点位于 `src/operator/tensor/broadcast_reduce-inl.h:368`。`baseline_gap: sampling IP precision` 下 `fclass.d` 的 6.10% 只锚定该归约 interval。

**(b) 互斥邻居排除**：

- **Loop Induction Variable Strength Reduction**：本 interval 无 div/rem/mul 索引计算与结构拷贝，故不归该 row（索引部分由 primary 认领）。
- **RVV Precision Conversion**：`fcvt.d.s` 在本 interval 占比 ≈0%，主体是补偿求和 + Inf 守卫 + volatile 栈往返，故不归该 row。
- **RVV Extrema / Arg-Extrema**：reducer 为 `mshadow::red::sum`，不产生 Min/Max 或 value+index，故不归这两个 row。
- **RVV Normalization / Matmul**：无跨 lane 统计 + 变换的完整算子、无 M/N/K 结构，故不归这两个 row。
- **Register Pressure and Save/Restore**：`dst`/`residual` 的栈驻留来自 `volatile` 形参（`fld fa3,-160(s0)`/`fsd fa4,-168(s0)`），非 RA spill，故不归 RA row。
- **No vectorization**：本 interval 已被 widening-reduction row 认领，故该 row 不成立（不是 supporting）。
- **Resource-Aware Instruction Scheduling**：无目标核 PMU/dependency 证据且 `baseline_gap: sampling IP precision` 禁止单指令 stall 归因，故不命中。

**(c) 双 Confidence 推导式**：

```
route: compiler-generated 模板 C++（broadcast_reduce-inl.h:368 调用 mshadow::red::sum::Reduce）
       + 源码逐字绑定「单输入加性归约 + widening(float→double) + 补偿项 + isinf 守卫 + volatile 累加器」
       + 累加器为循环携带依赖 + 硬件有 v + build 有 v1p0/zve32f/zve64f/zve64d → High
impact: VLEN=128 已知、bound type=compute/latency-bound（IPC 0.568）已知、区间局部份额 ≈17.1%；
        但 baseline_gap: sampling metadata/IP precision 未消解
        + 浮点重排容限未知（补偿求和 + Inf 守卫对顺序敏感）→ Medium
```

### 多候选仲裁小段

- 顶层 finding 数：**2**（primary = Loop Induction Variable Strength Reduction；independent = RVV Widening Additive Reduction Kernels）。
- evidence mechanism layer：primary 属 **L4（compute / codegen micro-structure：`loop_induction_variable_strength_reduction.md`）**；independent 属 **L1（vectorization / semantic dispatch：`rvv_widening_reduction_kernels.md`）**。二者 evidence 落在不相交地址集合（`13bc9b0`–`13bca06` vs `13bca0a`–`13bca3c`），机制（整数索引/结构物化 vs FP 补偿累加）、修复对象与验证方法均可分离 → 判为 **independent，分别输出**。
- 因果消除测试：把索引计算强度削减后，归约区间的标量补偿链与 volatile 栈往返不消失；把归约向量化后，若 reduce 维 stride 不规则则索引计算仍需逐元素进行（若规则则可顺带用基址推进消除——此点已写入 primary 的 `The fix` 作为实现选项）。故不存在单向的「上层改写自然消除下层」关系，两者不互为 supporting。
- 入口条件 A 排序：primary evidence sample share 合计 **≈81.3%**，independent 合计 **≈17.1%**（其余 ≈1.6% 为外层 idx 循环与 `assign` 收尾，不计入任一 finding）。**两项均仅为当前 sampled event（`cpu-clock`, `local period`）下本函数内的局部样本份额**，非 workload 级 Amdahl 上界。
- supporting：无（`no-vectorization.md`、`register_pressure_and_save_restore.md`、`rvv_precision_conversion_kernels.md` 的自身 row gate 均不成立，不得写成 supporting）。
- companion：无（无 glibc semantic row 参与）。

## Phase 4 — Root-cause blueprint / 根因蓝图：`mxnet::op::broadcast::seq_reduce_compute<mshadow::red::sum, 5, double, float, float, identity, set_index_no_op<double,long>>`

### 4.1 primary — Loop Induction Variable Strength Reduction

命中 row：Phase 3 中通过 gate 的 **Loop Induction Variable Strength Reduction**（`rows-codegen.md` 第 39 行；leaf `patterns/loop_induction_variable_strength_reduction.md`）。

**Root cause**：ndim=5 的归约外层用扁平的 reduce 下标 `k` 作为归纳变量，**每个元素都重新做一次 5 维索引分解**：`coord = unravel<5>(k, rshape)`（5 次 `div`/`rem` 求商求余，并把 5 个坐标写回栈上的 `coord`，且 `unravel` 按值返回 `Shape<5>`，导致编译器再插入一次 40 字节结构拷贝——反汇编中以 `vsetvli a5,a4,e8,m1` + `vle8.v`/`vse8.v` 的 RVV 字节拷贝循环出现），随后 `dot<5>(coord, rstride)` 再做 5 次 `mul` + 5 次 `add` 把坐标折算成线性偏移。而每轮的**实际内存访问只有 1 次 `flw`**。依据 `patterns/loop_induction_variable_strength_reduction.md` §When to apply：**「修复是把索引归纳变量替换成指针递增 + 预计算 end-pointer 比较，用强度削减把每轮的乘法/移位降为一次加法」**；§Why this is slow 第 1 点：**「index-based loop 每轮把 `i` 乘 stride 再加 base，增加 integer pipeline 压力；这份计算在连续访问里是纯冗余，指针本身已携带地址」**；第 3 点：**「循环控制在小循环体中占比高：当循环体只做一两次访存时，几条循环控制指令就能占据大部分 cycle，削减它们直接降低总 cycles」**。该 pattern 的定位描述与本处完全吻合——「索引计算比实际访存还重」的规则遍历循环。

**The fix / 修复方式**（仅描述实现层面的优化方向，不改变函数语义）：

修复对象：`src/operator/tensor/broadcast_reduce-inl.h:349-390` 的串行 k 循环中 `coord = unravel(k, rshape); big[j + dot(coord, rstride)]` 的**索引演化方式**（`src/operator/mxnet_op.h` 的 `unravel<ndim>`/`dot<ndim>` 调用点）。

修复前（当前生成代码形态，与 annotate 逐字对应，每轮 1 个 reduce 元素）：

```asm
# unravel<5>：5 次 div/rem + 5 次 coord 栈写
13bc9ba: ld      s6,0(a5)
13bc9c2: rem     a4,s5,s6
13bc9c6: sd      a4,8(a3)          # coord[i] = j - tmp*shape[i]   15.06%
13bc9c8: div     s5,s5,s6
13bc9cc: bne     a1,a5,13bc9ba     # 5 维循环回边                  17.75%
# 40 字节 Shape<5> 结构拷贝（编译器自动向量化）
13bc9d6: li      a4,40
13bc9da: vsetvli a5,a4,e8,m1,ta,ma
13bc9de: vle8.v  v1,(s5)
13bc9e2: sub     a4,a4,a5                                          # 25.87%
13bc9e6: vse8.v  v1,(a3)
13bc9ea: add     a3,a3,a5                                          #  8.37%
13bc9ec: bnez    a4,13bc9da
# dot<5>：5 次 mul/add
13bc9f6: ld      a4,0(a5)
13bc9f8: ld      s6,0(s5)
13bc9fe: addi    s5,s5,8
13bca00: mul     a4,a4,s6
13bca04: add     a3,a3,a4
13bca06: bne     a5,a2,13bc9f6     # 5 维循环回边                   5.69%
13bca10: flw     fa5,0(a3)         # 实际唯一的一次访存             2.43%
```

修复后（循环级强度削减：把每轮的 5 维分解改为增量维护）：

```cpp
// 结构示意：进入 k 循环前预计算起始偏移与每维的进位增量，循环内只做加法与比较。
// 必须保持 offset(k) == dot(unravel<5>(k, rshape), rstride) 对 [0, M) 逐点等价。
Shape<5> coord;                       // 仅维护一份，不再每轮重建/拷贝
Reducer::SetInitValue(val, residual);
index_t offset = 0;                   // k = 0 时 coord 全 0，offset = 0
for (size_t k = 0; k < M; ++k) {
    AType temp = OP::Map(big[j + offset]);   // 直接用已维护的偏移
    Reducer::Reduce(val, temp, residual);
    // 增量推进 coord/offset：最低维 +1，越界则回绕并把进位加到上一维
    for (int i = ndim - 1; i >= 0; --i) {
        if (++coord[i] < rshape[i]) { offset += rstride[i]; break; }
        offset -= (rshape[i] - 1) * rstride[i];   // 回绕：抵消 (shape-1) 个 stride
        coord[i] = 0;
    }
}
// 若 reduce 维在 rstride 下规则且最快维 stride 固定，还可进一步把 offset 直接按常量步进推进，
// 并按 references/kernel-conventions.md §3 预计算 end-pointer 做 `p != end` 比较
```

适用前提：`rshape`、`rstride`、`M` 在循环内不变（已由 annotate 证实——它们在循环外一次算出）；`offset(k)` 的推进只依赖 `rshape`/`rstride`，与元素值无关，故可与访存/归约解耦。

不可破坏的 correctness contract：

- **索引映射等价**：`offset(k) == dot(unravel<ndim>(k, rshape), rstride)` 对全部 `k ∈ [0, M)` 必须逐点成立，包括每个维度的进位顺序（`unravel` 从最后一维开始分解，故增量推进也必须从最低维开始进位）与 `k = 0` 时 coord 全 0 的初值。
- **边界与退化输入**：`M == 0` 时循环零次执行；`rshape[i] == 0` 的退化输入不得引入除零（原实现由 `div`/`rem` 语义定义，改造后须显式保证等价）；`j + offset` 不得越出 `big` 的合法范围。
- **溢出**：`offset` 与回绕补偿量 `(rshape[i]-1) * rstride[i]` 在 `index_t` 范围内不得溢出。
- **不改变 `seq_reduce_assign` 的归约语义、`Reducer::Reduce` 的调用次数与顺序、`assign(&small[idx], addto, OType(val))` 的收尾语义**。
- **不改变 OMP 结构**（`seq_reduce_compute` 对 `idx` 的并行与 `N >= thread_count` 分派）。

限制/风险：若 `rshape` 的维度数很大且各维长度很小，增量推进的进位分支频率上升，收益下降（需以真实 `rshape` 分布评估）；把 `coord` 从「每轮重建」改为「跨轮维护」会把 5 个整数放进长生命周期寄存器，可能带来新的寄存器压力（需检查生成代码与 spill）；若某次归约的 reduce 维在 `rstride` 下不规则（例如多段不连续），则只能做索引维护而不能直接用固定步进指针，需在实现中分流。预期 Profile signals：`13bc9b0`–`13bca06` 内 `div`/`rem`/`sd a4,8(a3)`/`bne a1,a5`/`sub a4,a4,a5`/`add a3,a3,a5` 与 `mul a4,a4,s6` 的样本份额大幅下降，替换为少量 `add`/`addi` 偏移推进与一次比较；`flw` 占比相对上升；instructions/element 明显下降。

### 4.2 independent — RVV Widening Additive Reduction Kernels

命中 row：Phase 3 中通过 gate 的 **RVV Widening Additive Reduction Kernels**（`rows-operator-rvv.md` 第 8 行；leaf `patterns/rvv_widening_reduction_kernels.md`）。

**Root cause**：归约区间以标量逐元素执行 `mshadow::red::sum` 的补偿求和：`y = src - residual; t = dst + y; if (isinf(t)) residual = 0; else residual = (t - dst) - y; dst = t;`，输入经 `fcvt.d.s` 从 float 加宽到 double；`dst`/`residual` 为 `volatile` 形参，被强制驻留栈上并每轮读写（`fld fa3,-160(s0)` 3.09% / `fsd fa4,-168(s0)` 0.87%）。依据 `patterns/rvv_widening_reduction_kernels.md` §Why this is slow 第 1 点：**「标量 `sum += src[i]` … 的本轮结果依赖上一轮，形成贯穿完整序列的串行链。RVV 可以在当前 `vl` 个元素内做向量部分和 … 缩短依赖链」**；第 2 点：**「Premature writeback / 过早回写：先把向量 lane 写回内存再求和会增加 load/store 与 cache 流量。RVV reduction 可以在寄存器内合并多个部分和」**。§The fix §1 要求先确定稳定算法（Kahan/Welford），§9 要求**「必须继续保持 … Inf/NaN 和 overflow/underflow 防护，不能用朴素 `sum(x*x)` 换性能」**——本处 `isinf(t)` 守卫必须保留。

**The fix / 修复方式**：

修复对象：`src/operator/tensor/broadcast_reduce-inl.h:368` 调用的 `mshadow::red::sum::Reduce`（`3rdparty/mshadow/mshadow/base.h:993-1004`）在该归约循环中的执行载体。

修复前：每元素 1 次 `flw` + `fcvt.d.s` + `fsub.d`/`fadd.d` + `fclass.d`/`andi`/`beqz`（Inf 守卫）+ 2 次 `fld`/2 次 `fsd`（volatile 累加器栈往返）。

修复后（RVV 加宽 + 寄存器内多路补偿累加器 + 掩码化 Inf 守卫）：

```cpp
// 结构示意：保留外层 OMP-over-idx 与 offset 计算（见 4.1），仅把最内层归约改为 RVV。
vfloat64m2_t vSum = __riscv_vfmv_v_f_f64m2(0.0, __riscv_vsetvlmax_e64m2());
vfloat64m2_t vRes = __riscv_vfmv_v_f_f64m2(0.0, __riscv_vsetvlmax_e64m2());
while (k < M) {
    const size_t vl = __riscv_vsetvl_e32m1(M - k);
    vfloat32m1_t src32 = __riscv_vle32_v_f32m1(big + base + offset, vl); // 连续时用 vle32；否则 vlse32
    vfloat64m2_t src   = __riscv_vfwcvt_f_f_v_f64m2(src32, vl);          // float -> double 加宽
    vfloat64m2_t y = __riscv_vfsub_vv_f64m2(src, vRes, vl);              // y = src - residual
    vfloat64m2_t t = __riscv_vfadd_vv_f64m2(vSum, y, vl);                // t = dst + y
    vfloat64m2_t r = __riscv_vfsub_vv_f64m2(__riscv_vfsub_vv_f64m2(t, vSum, vl), y, vl); // (t-dst)-y
    // isinf(t) 守卫：|t| >= +inf 的 lane 上把 residual 置 0（与 red::sum 语义一致）
    vbool32_t isInf = __riscv_vmfge_vf_f64m2_b32(__riscv_vfabs_v_f64m2(t, vl), /*+inf*/ INF, vl);
    vRes = __riscv_vmerge_vvm_f64m2(r, __riscv_vfmv_v_f_f64m2(0.0, vl), isInf, vl);
    vSum = t;
    k += vl; /* 同步按 vl 推进 offset（见 4.1） */
}
// 末尾按 red::sum::Merge(dst_val,dst_residual,src_val,src_residual) 的（含 isinf(t1) 分支的）
// Neumaier 合并式把各 lane 的 (vSum, vRes) 归并为一个标量 (val, residual)
```

适用前提：`M`、`rshape`、`rstride` 循环内不变；reduce 维在 `rstride` 下规则（连续用 `vle32.v`，固定跨步用 `vlse32.v`，byte stride 按 `sizeof(float)` 换算）；源 `big` 与目标 `small` 不重叠。

不可破坏的 correctness contract：

- **补偿与 Inf 守卫语义**：必须保留 `y/t/residual` 递推与 `isinf(t) → residual = 0` 分支（不得用朴素 `vfredusum` 取代）；跨 lane 归并必须使用 `red::sum::Merge` 的同一公式（含 `isinf(t1)` 分支）。
- **浮点顺序**：向量化改变加法结合顺序，结果可能与标量 reference 存在末位差异 → 须按项目容差验证；要求确定性时不得默认 unordered 等价。
- **加宽/收窄**：`vfwcvt` 与最终 `OType(val)`（double→float）舍入须与标量 `fcvt.d.s`/`fcvt.s.d` 一致。
- **空输入与初值**：`M == 0` 时 `val` 为 `SetInitValue` 的 0；`vl` 循环不得执行访存。
- **越界与推进**：`k`/`offset` 必须按实际 `vl` 推进；不得用最后一次 `vl` 做最终归约（§Verification「最终归约范围验证」）。
- **不改变 OMP 结构与每线程独立累加器**；不得修改共享 `red::sum::Reduce` 的 volatile 形参在其他路径上的既有语义。

限制/风险：reduce 维 `M` 较短时 `vsetvli`/末尾归并开销可能抵消收益；`vfwcvt` 的 destination EMUL = 2 × source，须按 destination 做寄存器预算（`references/kernel-conventions.md` §2）；`vlse32` 吞吐随 `rstride` 变化；仓库当前不存在任何 RVV 后端，需新增实现载体与运行时 gate。预期 Profile signals：`13bca0a`–`13bca3c` 内 `fclass.d`/`beqz`/`fld`/`fsd` 与标量 `fsub.d`/`fadd.d` 的份额下降；出现 `vsetvli`、`vle32.v`/`vlse32.v`、`vfwcvt.f.f.v`、`vfsub.vv`、`vfadd.vv`、`vfabs.v`、`vmfge.vf`、`vmerge.vvm` 与末尾归并序列。

**Baseline facts 回填**：hardware ISA `rv64imafdcv_..._zve32f_zve64f_zve64d`（有 `v`）；build ISA `rv64i2p1_..._v1p0_..._zve32f1p0_zve64f1p0_zve64d1p0_zvl128b1p0`（无 mismatch）；VLEN = 128 bit；bound type = compute/latency-bound（IPC 0.568）；缺口 `baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`。

**收益上界**：primary = 本函数 15123 个样本中的 **≈81.3%**（索引计算区间）；independent = **≈17.1%**（归约区间）。二者均为当前 sampled event（`cpu-clock`, `local period`）下本函数内的局部样本份额，不构成 workload 级 Amdahl 上界。

**三维路由判定（两 finding 共用）**：
- `current source`：**compiler-generated C++ 模板代码**（`broadcast_reduce-inl.h:349` 的 `seq_reduce_assign` + `mxnet_op::unravel<5>`/`dot<5>` + `mshadow::red::sum::Reduce`）。非 `.S`、非 JIT。
- `implementation existence/reachability`：**目标 RVV/专用实现不存在**（全仓 grep 无 `riscv`/`RVV` 引用；mshadow 仅有 x86 `MSHADOW_USE_SSE` 路径）；`Kernel Selection` row 不命中。
- `function-level policy`：**无 assembly-default policy**，不进入 policy-backed missing `.S` 分支；修复载体为循环级 C++ 改写 + RVV intrinsic。

**Implementation-shape proof**：不适用（N/A）——未走 policy-backed missing `.S` 分支（policy/existence 四证不齐），不输出 shape-first 字段，不补猜 VL/LMUL/tail。

**Related PRs / 关联提交**

`patterns/loop_induction_variable_strength_reduction.md`（primary）— Related PRs：3 条
- https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d
- https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df
- https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec

`patterns/rvv_widening_reduction_kernels.md`（independent）— Related PRs：16 条
- https://github.com/opencv/opencv/pull/27096
- https://github.com/opencv/opencv/commit/33d632f85e4c1cc4d70bc7210c2f126fa8a4b0bb
- https://github.com/opencv/opencv/pull/26624
- https://github.com/uxlfoundation/oneDNN/pull/5361
- https://github.com/uxlfoundation/oneDNN/commit/a95f0060cfcb75aeee7b937f18dbb2ff32064f50
- https://github.com/openjdk/jdk/commit/72297d22d19e34ff26bd34644dc087a1dec9527e
- https://github.com/openjdk/jdk/commit/08a2f841ec78a10f8d6d54b2ac3a92e89f765f14
- https://github.com/openjdk/jdk/commit/2c1e4c381615ce52276f4bf331a1e7a845af4b6e
- https://github.com/openjdk/jdk/pull/20910
- https://github.com/openjdk/jdk/commit/134b63f0e8c4093f7ad0a528d6996898ab881d5c
- https://github.com/openjdk/jdk/pull/16629
- https://github.com/openjdk/jdk/commit/1aebab780c5b84a85b6f10884d05bb29bae3c3bf
- https://github.com/openjdk/jdk/commit/1b6281d98cf0e7c5435c563bfedd6f07b79bfa62
- https://github.com/OpenMathLib/OpenBLAS/commit/c37509c213a34a8cae449ededd7bc7064675ecc4
- https://github.com/OpenMathLib/OpenBLAS/commit/3918d8504e7720d94221025ae6078a2459ccb104
- https://github.com/alibaba/MNN/pull/4433

## Phase 5 — Verification forecast / 验证预测：`seq_reduce_compute<mshadow::red::sum, 5, double, float, float, identity, set_index_no_op<double,long>>`

**primary（Loop Induction Variable Strength Reduction）**

- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`25.87 : 13bc9e2: sub a4,a4,a5`、`17.75 : 13bc9cc: bne a1,a5,13bc9ba`、`15.06 : 13bc9c6: sd a4,8(a3)`、`8.37 : 13bc9ea: add a3,a3,a5`、`5.69 : 13bca06: bne a5,a2,13bc9f6`、`1.50 : 13bca00: mul a4,a4,s6`、`0.77 : 13bc9c2: rem a4,s5,s6` —— 每轮的 5 维 div/rem 分解、40 字节 `Shape<5>` 拷贝与 5 元素点积应被增量偏移推进取代。
- 应出现侧（锚定 `patterns/loop_induction_variable_strength_reduction.md` §Verification）：「`perf annotate` / `objdump -d` 确认循环体中的 `slli`/`add`（index scaling）与 limit reload 减少，替换为指针递增与指针比较」；「确认 instructions 下降而 memory bandwidth 不变，印证收益来自 loop control」。
- 边界验证（§Verification）：「遍历测试覆盖 zero length（`n==0` 不进循环）、single element、large count 和地址上界附近的 overflow boundary」；「对遍历结果做等价性比对（与索引版本逐元素对照），确认无 off-by-one」→ 本处必须逐点核对 `offset(k) == dot(unravel<5>(k, rshape), rstride)` 覆盖各维进位边界。

**independent（RVV Widening Additive Reduction Kernels）**

- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`6.10 : 13bca28: fclass.d a5,fa4`、`4.57 : 13bca30: beqz a5,13bca82`、`3.09 : 13bca14: fld fa3,-160(s0)`、`0.87 : 13bca36: fsd fa4,-168(s0)`、`0.00 : 13bca24: fadd.d fa4,fa5,fa4` —— 逐元素标量 Inf 守卫与 volatile 栈往返应被向量掩码守卫与寄存器内累加取代。
- 应出现侧（锚定 `patterns/rvv_widening_reduction_kernels.md` §Verification）：「指令验证：`objdump`/`llvm-objdump`/`perf annotate` 确认出现 `vsetvli`、widening 累加、`vcpop`、`vredsum`/`vfredusum` 等指令」→ 本处对应 `vsetvli`、`vle32.v`/`vlse32.v`、`vfwcvt.f.f.v`、`vfsub.vv`、`vfadd.vv`、`vfabs.v`、`vmfge.vf`、`vmerge.vvm` 与末尾归并。
- 附加验证（同文件 §Verification）：「结合性与浮点顺序验证 …… 要求确定性时不得默认 unordered」；「accumulator 溢出验证：用最大长度和极值输入检查所选 accumulator 位宽是否溢出」；「最终归约范围验证：构造有效值集中在早期数据块、最后一块较短的输入」；「长度、tail 与 VLEN 无关性验证：覆盖零、单元素、`VLMAX` 整倍数和 tail」。
- 组合验证顺序（`arbitration.md` 第 5 条）：先分别验证两项可分离机制（索引强度削减 / 归约向量化），再做组合验证。

**要把结论升级到 profile-backed 精度，最少需补采的数据**：在目标核上用 `perf annotate --stdio -l -s '<symbol>' --percent-type=global-period` 重采（或至少核对 `perf report --header-only` 的 `precise_ip`），以消解 `baseline_gap: sampling metadata` 与 `baseline_gap: sampling IP precision`；否则 `sub a4,a4,a5` 的 25.87% 只能停留在 interval-level 机制，不能升级为单指令归因。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | `1/1 组；mxnet::op::broadcast::seq_reduce_compute<mshadow::red::sum, 5, double, float, float, identity, set_index_no_op<double,long>> [clone ._omp_fn.0]`（含 2 个顶层 finding 的 Phase 4/5 分节 4.1、4.2） |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表齐全 + 2 个 L0 gate + bound-type gate 结论；gap 标签 2 项：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`（含 `Sampling IP precision` 行） |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 2 个；evidence 锚点 `25.87 : 13bc9e2: sub a4,a4,a5`、`6.10 : 13bca28: fclass.d a5,fa4`；supporting 0 个；排除 6 条；推导式 2 条（route 均 High / impact 均 Medium） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件：`patterns/loop_induction_variable_strength_reduction.md`（命中 row：Loop Induction Variable Strength Reduction）+ `patterns/rvv_widening_reduction_kernels.md`（命中 row：RVV Widening Additive Reduction Kernels）+ `references/kernel-conventions.md`；引用短语首词：「索引计算比实际访存还重」「循环控制在小循环体中占比高」「标量 `sum += src[i]` … 形成贯穿完整序列的串行链」「不能用朴素 `sum(x*x)` 换性能」；`The fix` 含 before（`13bc9ba ld s6,0(a5)`…`13bca10 flw fa5,0(a3)` / `13bca0a`…`13bca3c`）/after（增量 offset 维护 + 预计算 end-pointer / `vsetvli`+`vle32.v`+`vfwcvt.f.f.v`+`vfabs.v`+`vmfge.vf`+`vmerge.vvm`）、correctness（索引映射逐点等价、进位顺序、退化输入、溢出；补偿与 Inf 守卫语义、浮点顺序容差、最终归约范围）、风险（rshape 分布、寄存器压力、`vfwcvt` EMUL、仓库无 RVV 后端）；预期 Profile 信号锚点；missing `.S` 不适用（N/A）；`Related PRs：3 条 + 16 条 URL` |
| 5 | 路径合规：8 项 trace 可解释扫描集；零/多命中、关系与 evidence-mechanism layer 合规；每个 blueprint leaf 来自已通过 gate 的 row；入口条件 A 按动态份额排序；`th.v*` 未全局停扫 | ✅ | 模式 A（profile_backed）；2 个 leaf 均通过各自 row gate（L4 / L1）；排序按局部份额 81.3% > 17.1%；independent 关系经因果消除测试说明；`th.v*` 不存在故无 flavor 冻结，class 列表 `rows-operator-rvv.md, rows-codegen.md` |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3 引用行、出现侧标注 pattern 文件 §Verification | ✅ | 消失侧 `13bc9e2: sub a4,a4,a5`、`13bc9cc: bne a1,a5,13bc9ba`、`13bc9c6: sd a4,8(a3)` / `13bca28: fclass.d a5,fa4`、`13bca14: fld fa3,-160(s0)`；出现侧 `loop_induction_variable_strength_reduction.md §Verification`（指令构成对照 + 边界/尾部 + 等价性比对）与 `rvv_widening_reduction_kernels.md §Verification`（指令验证 + 结合性/浮点顺序 + accumulator 溢出 + 最终归约范围 + tail） |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成或契约外实施分支；除 object-clarification 外无追问；交付物止于证据、蓝图、完整 `The fix` 和验证预测 | ✅ | 全文无提问、无「是否实施」分支；Implementation-shape proof 明确 N/A 未编造 shape 字段 |

修正记录：无
