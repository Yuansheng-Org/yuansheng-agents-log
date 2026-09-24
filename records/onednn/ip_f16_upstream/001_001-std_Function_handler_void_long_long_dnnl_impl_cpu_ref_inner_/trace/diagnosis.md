Functions under analysis: [std::_Function_handler<void (long, long), dnnl::impl::cpu::ref_inner_product_fwd_t::execute_forward(dnnl::impl::exec_ctx_t const&) const::{lambda(long, long)#2}>::_M_invoke]（1 个）→ 本输出含 1 组 Phase 3–5

# RISC-V 性能诊断报告 — onednn ip_f16_upstream (batch-070 function 001, rank 001)

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 perf annotate：已提供（`001-std：：_Function_handler＜void (long, long), dnnl：：impl：：cpu：：ref_inner_product_fwd_t：：execute_forward(...) const：-905f74d4690a-annotate.txt`，libdnnl.so.3.14，cpu-clock:u，64 samples，percent: local period，共 2996 行，含完整 hot loop）
- perf stat（bound/context）：已提供（`8-onednn-benchdnn-benchmark-riscv-ip_f16_upstream.txt`，ip_f16_upstream 测试）
- workload/binary/DSO/source context：部分提供（metadata.json 含仓库/commit/硬件信息；`binaries: {}`，警告 "No ELF executable binaries were found for this run"，无 DWARF/object mapping → `source_context_gap` 已标注于 Phase 1）
- readelf -A（热点 object 的 Tag_RISCV_arch）：缺失（无 ELF binary 产物 → `baseline_gap: build ISA`，详见 Phase 1）
- hardware ISA（metadata cpuinfo）：已提供（`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`）
- `vlenb`：已提供（metadata vector: vlen_bits=128, vlenb=16）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（event=cpu-clock:u，percent=local period，64 samples；无 `perf report --header-only` 的 global-period 与 workload 级贡献信息 → `baseline_gap: sampling metadata`）
- Sampling IP precision（precise_ip / Exact-IP / PMU skid）：缺失 → `baseline_gap: sampling IP precision`

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | SOPHGO SG2044 / T-Head C920v2，RVV 1.0 `v`，且含 `zfh`/`zfhmin`/`zvfh`/`zvfhmin`/`zbb`/`zba`/`zfa` 等；OoO superscalar（metadata "instruction scheduling method": "out-of-order"） |
| Build ISA | `baseline_gap: build ISA` — metadata `binaries: {}`，无 ELF 产物可跑 readelf -A；但 annotate 反汇编全函数 0 条 `v*`/`th.v*` 指令、出现 `scalbnf@plt`/`__stack_chk_fail@plt` 调用，说明承载该热点的 libdnnl.so.3.14 至少未对这条路径启用 V 向量代码（仅静态形态证据，object 级 attribute 无法核对） |
| Vector flavor | annotate 中全 scalar（`flw`/`fmadd.s`/`fsw` + 整数 bit-twiddle），zero `v*`、zero `th.v*` → 无 flavor mismatch，但存在 hardware 有 `v` 而该路径未用 V 的 L0 baseline finding |
| VLEN | 128 bits（vlenb=16，metadata） |
| Bound type | IPC=0.668（32.68T instructions / 48.90T cycles）；cache_references≈5.53M（相对 32.7T 指令几乎可忽略）、L1_dcache_load_miss_rate=0.160%、LLC_load_miss_rate=17.011%（但 LLC_loads 仅 5.53M）、branch_miss_rate=0.077% → 非 memory-bound、非 branch-bound，为 compute/latency-bound 的标量执行开销主导 |
| Sampling semantics | event=cpu-clock:u（时间类）；percent=local period（函数内局部样本份额）；64 samples；同一运行窗口（单次 benchmark run）；函数级 workload 贡献未知 → 只能表述为「当前 sampled event 下的函数内局部样本份额」，禁止称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision` — 无 precise_ip/Exact-IP 证据；单行占比只锚定 basic block / loop interval，不承担单指令 latency 根因 |

L0 baseline gate #1（hardware 有 `v`、build/路径无 `v`）：成立 — 作为最高优先级 baseline finding 置顶；该路径整体为 compiler-generated scalar 代码，继续扫描所有可见 row。
L0 baseline gate #2（`th.v*` flavor gate）：不适用 — annotate 无任何 `th.v*`。
Bound-type gate：compute/latency-bound（标量指令开销与依赖链主导），不冻结向量化 route；VLEN 已知、build ISA 缺失 → impact confidence 封顶 Medium。
结论：入口条件 A（profile_backed）。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个函数）。本函数是 `ref_inner_product_fwd_t::execute_forward` 中经 `parallel_nd(MB, OC, ...)` 派发的 worker lambda，语义为 inner-product forward（GEMM 形态：MB×OC 输出，每个输出对 IC×KD×KH×KW 求点积累加，可视为 K=IC*KD*KH*KW 的 GEMV/GEMM）。

hot loop 锚点（kw 内层循环，回边 `55f7ee: blt s1,a4,55f4f0`）：
- src_off 重算区：`7.81 : 55ffd8: sd s9,-352(s0)`、`6.25 : 55ffdc: li a5,0`、`4.69 : 56004e: bne t4,a5,56003a`（get_data_off → off_v，ndims==2 分支，55ffa8–56005x，≈39.0%）
- wei_off 重算区：`7.81 : 560236: bne a1,a2,560226`、`4.69 : 560148: sd zero,-184(s0)`（get_weights_off → off_v，ndims==2 分支，5600d2–56023a，≈28.1%）
- fp16→fp32 软件转换区：`3.12 : 55fe9e: slli s10,s10,0x1`、`1.56 : 55feaa: andi a3,a3,31`、`3.12 : 56026e: j 55f9da`、`1.56 : 55fdd0: bnez a4,5602ee`（float16_t::operator float()，≈14.0%）
- 乘加与循环控制：`3.12 : 55fa48: bge s1,a4,55f7f2`、`1.56 : 55fa36: fmv.w.x fa5,a5`、`1.56 : 55f804: addi s7,s7,1`（≈6.2%）
- 每输出 epilogue（bias off<long> + dst off<long,long> + post_ops）：`1.56 : 55f874: li a3,5`、`1.56 : 55f8ae: sd zero,-200(s0)`、`1.56 : 55f98e: lwu a4,104(a2)`（≈4.7%）
Sampling IP precision 不足：结论收敛到 interval-level mechanism（区间聚合归属），不做单指令 latency 归因。

## Phase 3 — Pattern scan / 模式扫描：std::_Function_handler<...ref_inner_product_fwd_t::execute_forward...lambda#2>::_M_invoke

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude**：当前代码来源为 compiler-generated（annotate 显示 libstdc++ `_M_invoke` 内联展开 + libm `scalbnf@plt` + `__stack_chk_guard` 检查），无手写 `.S` provenance，也无 policy/existence 四证指向缺失 `.S`
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar loop，语义为 floating-point dot-product / GEMV（inner product forward），且含必要 FP16→FP32 精度转换；硬件有 `v`
3. `rows-string-memory.md` — **exclude**：非 copy/fill/sentinel/compare/checksum 语义，无 string/memory 操作
4. `rows-vectorized-tuning.md` — **exclude**：annotate 无任何 `v*` 指令，修正对象非 RVV 配置/寄存器/展开
5. `rows-codegen.md` — **include**：compiler/JIT 生成代码的指令形态；fp16 转换的 helper/branch/bit-manipulation 形态（FP semantic lowering 候选）、offset 重算的循环级地址重建形态（LISR 候选）、register save/restore 形态
6. `rows-offload.md` — **exclude**：SG2044 无专用矩阵引擎/AME/MME evidence；无独立 weight-repack 层证据；无多线程分块算术热点（本函数为 single-thread worker lambda）
7. `rows-crypto.md` — **exclude**：无密码学原语信号
8. `rows-runtime-os.md` — **exclude**：用户态 benchdnn 基准，无 timer/CSR/ISR/特权路径信号

### Classes scanned: rows-operator-rvv.md, rows-codegen.md

### Local performance pattern scan: std::_Function_handler<...lambda#2>::_M_invoke

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Floating-Point Matmul and GEMV Kernels（primary） | nested M/N/K 点积累加（IC*KD*KH*KW），主循环全 scalar `fmadd.s`；src/wei off 重算 + fp16 转换 + FMA 主导样本；无 Int8/zero-point 合同；无矩阵引擎 | High | Medium | `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` |
| No vectorization（supporting） | hot main loop 全 scalar（`flw`/`fmadd.s`/`fsw`），zero `v*`、zero `th.v*`，hardware 暴露 `v`（RVV 1.0） | — | — | `patterns/no-vectorization.md` |

### 顶层 finding 1：RVV Floating-Point Matmul and GEMV Kernels（primary）

**(a) 逐字 evidence 引用**（均属 kw 内层循环 interval，回边 55f7ee→55f4f0）：
- `7.81 : 55ffd8: sd s9,-352(s0)` — src_off 的 `off_v`（memory_desc_wrapper::off<long,long>）内 pos 数组写入
- `6.25 : 55ffdc: li a5,0` — 同一 off_v 的 format_kind==undef 预判
- `7.81 : 560236: bne a1,a2,560226` — wei_off 的 `off_v` 尾部 ndims 步长累加循环分支
- `4.69 : 560148: sd zero,-184(s0)` — wei_off 的 `off_v` pos_copy 清零
- `1.56 : 55f4f4: ld a3,32(s4)` / `1.56 : 55f500: li a4,2` — get_data_off 的 ndims 分发（ndims==2 → off<long,long>）
- 对照：`1.56 : 55fa36: fmv.w.x fa5,a5` + `0.00 : 55fa44: fmadd.s fs0,fa5,fa4,fs0` — 实际乘加只有 1 条 `fmadd.s`
每个 kw 迭代执行两次完整 `off_v`（src_off 于 55ffa8 区、wei_off 于 5600d2 区），每次含 11×`sd zero` pos 清零、ndims 轮 padded_offsets 累加、inner_nblks 轮 `rem/div/mul`（INT32 分支）、ndims 轮 stride `mul/add`。区间聚合份额 ≈72%（src_off 区 ≈39% + wei_off 区 ≈28% + get_data_off 分发 ≈5%）。

**(b) 互斥邻居排除**：
- Quantized Matmul row（`rvv_quantized_matmul_and_requantization_kernels.md`）：排除 — hot interval 内无 `rem/div` 之外的 Int8/INT4 decode、zero-point、requantization 整数乘移序列；`rem/div` 属于 blocking-desc 的 inner_nblks 布局解算，不是量化合同。
- Precision-conversion row（`rvv_precision_conversion_kernels.md`）：排除作 primary — fp16→fp32 转换区 ≈14% 只是输入子步骤，主算子的 offset 重算 + FMA 结构主导；按该 row 行内互斥「widening 后的 FP32 matmul 算术主导、conversion 只是输入子步骤 → 对应 operator row」，转换并入本 matmul finding 作 supporting（修复合并进 convert-once/Zvfh 方案）。
- Kernel-selection row（`kernel_selection_and_runtime_specialization.md`）：排除 — 无仓库源码证据证明存在满足该 fp16/shape 合同的专用 RVV kernel 且 dispatch 未命中；不能凭 `ref_` 前缀断言 selection 失败。
- Cache-aware-blocking row：排除 — cache_references≈5.53M（占 32.7T 指令可忽略）、L1 miss rate 0.16%，无 tile-residency/复用拐点证据。
- Matrix-engine offload（rows-offload）：排除 — 目标 SoC 无矩阵引擎证据。
- `no-vectorization.md`（同组最泛成员）：本行只解释「缺少向量执行」这一载体现状，不解释 GEMM 语义合同与 offset 重算机制 → supporting。

**(c) 双 Confidence 推导式**：
- route: `main-loop 为 compiler-generated scalar dot-product（M/N/K 语义由 ref_ip_utils/ref_inner_product 源结构确证）+ hardware V（metadata）+ 无量化/矩阵引擎/缓存 residency 邻居 + no-vectorization 独立佐证` → **High**
- impact: `函数内 sample share ≈95%（offset 重算+转换+FMA 均在 kw 循环内）、VLEN=128 已知、bound type=compute/latency 已知；但 build ISA 缺失（baseline_gap: build ISA）、采样语义非 global-period（baseline_gap: sampling metadata）` → **Medium**

### Supporting evidence（写入 primary 下方，不计顶层命中数）
- `No vectorization`：(a) `1.56 : 55fa36: fmv.w.x fa5,a5`（scalar 数据搬运）、`0.00 : 55fa44: fmadd.s fs0,fa5,fa4,fs0`（scalar FMA）、`3.12 : 55fa48: bge s1,a4,55f7f2`（scalar 循环分支）；hot main loop 区间（55f4d2–55f812、55ffa8–560236）零 `v*`/`th.v*`。supporting because: 同一 mechanism — 同一标量主循环缺向量执行，只解释缺少 vector unit 利用，不决定 GEMM 语义/offset 重算的贡献载体。两种 confidence 保持 `—`。

### 多命中仲裁小段
- 顶层 finding 数：1（primary：RVV Floating-Point Matmul and GEMV Kernels）；supporting：1（No vectorization）。
- 因果层次：本 finding 落在 L1（vectorization / semantic dispatch — floating-point matmul/GEMV）；offset 重算（L2/L4 表象）与 fp16 软件转换（L4 计算微结构表象）是同一 evidence 上的下层 signal：一旦按 matmul/GEMV 修复（沿 OC 向量化 + 指针递增微内核 + Zvfh convert-once），offset 重算与逐元素软件转换都会消失 → 不单列为 independent finding。
- 收益排序（入口条件 A）：本 finding 的函数内局部 sample share 加总 ≈95%（kw 循环区 55f4d2–56023a：offset ≈72% + fp16 转换 ≈14% + FMA/loop ≈6% + src load ≈2%）；epilogue ≈5% 不属主循环，不计入。同一调用链内不得与其它函数份额简单相加。
- L0 baseline finding（hardware `v` vs 该路径零 `v*`）作为最高优先级前置记录，已置顶。

## Phase 4 — Root-cause blueprint / 根因蓝图：std::_Function_handler<...lambda#2>::_M_invoke

命中 row：`rows-operator-rvv.md` — RVV Floating-Point Matmul and GEMV Kernels（primary）+ No vectorization（supporting，仅作机制解释）。

### 1. Root cause
执行 `ref_inner_product_fwd_t` 的 compiler-generated 标量 reference 内核：每个输出 (mb, oc) 对 IC×KD×KH×KW 做标量点积，内层 kw 循环每次迭代都经通用的 `memory_desc_wrapper::off_v()` 从零重算 src_off 与 wei_off（两次完整 ndims 循环 + inner_nblks `rem/div/mul` + stride 累加），再走软件 fp16→fp32 转换（`float16_t::operator float()`：ee/mm/ss 位拆解、denormal 走 `scalbnf@plt`、NaN quiet-bit 修正），最后单条 `fmadd.s` 累加。依据 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` §Why this is slow：「浮点 microkernel 沿错误轴向量化、FMA accumulator 依赖过长、packing 与消费合同不匹配，或短 shape/edge path 没有复用正确的 FP kernel」；本函数属于「scalar reference fallback 被真实执行，无微内核结构」形态 —— 该文件 §The fix 要求「沿输出列向量化」并「减少重复转换」。本热点完全未向量化（supporting pattern `no-vectorization.md` §Why this is slow：vector unit 没有处理 hot main-loop 的并行元素），且每轮重复的地址重建与逐元素转换使「动态指令数和分支数量随元素数线性放大」（`rvv_precision_conversion_kernels.md` §Why this is slow 第 1 条）。

### 2. The fix / 修复方式
修复对象：`ref_inner_product_fwd_t::execute_forward` 的 ker lambda（oneDNN 源码 `src/cpu/ref_inner_product.cpp`），并为其选择 RVV 向量化路径。方向（与 pattern 一致，非实施补丁）：

**修复 A（primary，沿输出列/OC 向量化 + 消除每迭代 offset 重算）**，依据 `rvv_floating_point_matmul_and_gemv_kernels.md` §The fix 第 2 节「Vectorize floating-point matmul along output columns / 沿输出列向量化浮点矩阵乘」：
- before（当前形态）：`for (mb) for (oc) { for (ic) for(kd) for(kh) for(kw) { src_off = mdw.off(mb,ic,kd,kh,kw); wei_off = mdw.off(oc,ic,kd,kh,kw); d += load(src,src_off)*load(wei,wei_off); } ... }` — 每次 kw 迭代重建 src_off/wei_off。
- after（向量化形态示意，需按真实 mdw 布局调整）：以 OC 为向量化轴、单 mb，`for (mb) for (ic_block) { ... 预计算每 (mb, oc_lane) 的基址与增量（外层一次性布局解算，内层指针递增）; for (oc_panel) { vl = vsetvl(OC-oc_panel); acc[vl] = 0; for (ic) { s = 标量 src 值; w = vle(weights+panel, vl); acc = vfmacc_vf(acc, s, w, vl); } vse(dst+panel, acc, vl); } }` — 每输出 panel 共享 src 广播与一次布局解算，消除 per-(kw) 两次 `off_v`。该文件明确「增加输出行 accumulator 数量可以提高复用，但会增加寄存器压力。不得固定使用两个、四个或八个输出行」，且「比较 N 维和 K 维方案，不能固定选择一种」——OC 轴 vs IC 轴选择需按 shape 与 packing 实测。
- correctness contract：FP 累加精度（FP16→FP32 widening 累加，`Zvfh` path 的 static/build/runtime gate 一致）、bias/post-ops/alias、layout/leading-dimension、IC 全归约、KD/KH/KW 空间维折叠顺序不变。
- 限制/风险：short-shape crossover（本测试 gops=32K MAC，极短 shape 下 vsetvl/微内核 setup 可能反噬，需 crossover 判定）；register pressure 增加（vfmacc accumulator 多路）；`vsetvli` 开销需在 128-bit VLEN 下以实测为准。

**修复 B（convert-once：硬件 fp16→fp32 转换替换软件 bit-twiddle）**，依据 `rvv_precision_conversion_kernels.md` §The fix 第 3 节「Scalar half conversion ... 用 Zfh/Zfhmin 的 scalar fcvt/half 指令替代软件序列」与第 1 节「FP16→FP32 用 `vfwcvt.f.f.v` 建 vector fast path」：
- before：`55fdd0: bnez a4,5602ee` → `lhu`+`srliw`+`andi`+`scalbnf@plt`+`fmul.s`（denormal 路径）等 ≈10–15 条标量 bit 操作/helper 调用。
- after（scalar 路径）：`fcvt.s.h fa0, fa0`（Zfh，硬件 `zfh` 已暴露）；或（向量路径）`vle16` + `vfwcvt.f.f.v`（Zvfh，硬件 `zvfh` 已暴露），在主循环稳定 vtype 区间内 convert-once。
- correctness contract：NaN payload/quiet-bit、±Inf、±0、denormal、rounding mode 必须与 scalar reference `float16_t::operator float()` 逐位一致；C/C++ 语义要求特殊值 bit pattern 与 scalar reference 一致时用 mask/slow path 修正（该文件 §The fix 第 1 节「mask 检测特殊值并跳转窄范围 slow path」）。
- 限制/风险：denormal 输入占比高时 slow path 限制收益；`Zvfh` widening 的 source/destination LMUL 预算按 kernel-conventions §2 选择。

### 3. Baseline facts 回填
- Hardware ISA：SG2044 / C920v2，RVV 1.0 `v` + `zfh/zfhmin/zvfh/zvfhmin/zbb/zba/zfa`，OoO superscalar（metadata）
- Build ISA：`baseline_gap: build ISA`（无 ELF 产物；静态反汇编形态显示该路径未启用 V）
- VLEN：128 bits（vlenb=16）
- Bound type：compute/latency-bound（IPC 0.668；cache refs≈5.53M、L1 miss 0.16%、branch miss 0.08%）

### 4. 收益上界
入口条件 A：本 finding 的 evidence sample share 加总 ≈0.95（kw 内层循环区间内：offset 重算 ≈0.72 + fp16 软件转换 ≈0.14 + FMA/循环控制 ≈0.06 + src load ≈0.02；另 epilogue ≈0.05 不计入）。表述为「当前 sampled event（cpu-clock:u, local period）下的函数内局部样本份额」；因 `baseline_gap: sampling metadata`（非 global-period、无 workload 级贡献），不得称 workload 级 Amdahl 上界。

### 5. 三维路由判定
- current source：compiler-generated scalar C++（`_M_invoke` 内联 + libm/libstdc++ PLT 调用可证），非 `.S`、非 intrinsic-RVV。
- implementation existence/reachability：`ref_` 内核被真实执行（本函数即执行体）；不存在/未证实存在满足本 shape 的专用 RVV kernel（无仓库源码证据）→ 不走 kernel-selection；policy/existence 四证不齐，不进入 missing `.S` 分支。
- function-level policy：oneDNN 对 RISC-V 未提供该 fp16 inner-product 的向量 kernel 证据；修复方向限定为 compiler-vectorizable/intrinsic 载体。

### 6. Implementation-shape proof
不适用 — 未进入 policy-backed missing `.S` 分支。

### 7. Related PRs 小节
按 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md` `## Related PRs`：Related PRs：31 条 URL（OpenBLAS 7、oneDNN 16、llama.cpp 7、MNN 1）；其中 oneDNN 直接相关示例：
- https://github.com/uxlfoundation/oneDNN/commit/d6f82a2d0d0db41e6daaf20fbb4fd352843aac64（RVV GEMM microkernel scheduling）
- https://github.com/uxlfoundation/oneDNN/commit/3bac96b8bc1fc9c348c986f38f65285693943d2f（RVV floating-point matmul implementation）
- https://github.com/uxlfoundation/oneDNN/pull/4410、/pull/4414、/pull/4545、/pull/4770、/pull/4824、/pull/4840、/pull/4850、/pull/5157、/pull/5294、/pull/5405（RVV floating-point matmul kernel）
- https://github.com/uxlfoundation/oneDNN/pull/5403（low-precision BRGEMM）
- https://github.com/OpenMathLib/OpenBLAS/commit/809e1cba8f1f3f89972581e8b82f2ec52e51eadb（FP16 RVV GEMV，Has perf data）
- https://github.com/OpenMathLib/OpenBLAS/commit/2d82d144e2791e37d7a314237b638d85b156a2ec（RVV GEMV cache-friendly traversal）

按 `patterns/no-vectorization.md` `## Related PRs`：Related PRs：16 条 URL（OpenCV 15 + 1 commit）；示例 https://github.com/opencv/opencv/pull/22179、https://github.com/opencv/opencv/pull/27160。

## Phase 5 — Verification forecast / 验证预测：std::_Function_handler<...lambda#2>::_M_invoke

**finding（primary：RVV Floating-Point Matmul and GEMV Kernels，含 supporting no-vectorization 与 fp16 convert-once）**

- 应消失/缩小侧（锚定 Phase 3(a) 逐字引用行）：
  - `7.81 : 55ffd8: sd s9,-352(s0)` 与 `6.25 : 55ffdc: li a5,0`（src_off off_v 区）应消失或份额骤降；
  - `7.81 : 560236: bne a1,a2,560226` 与 `4.69 : 560148: sd zero,-184(s0)`（wei_off off_v 区）应消失或骤降；
  - `1.56 : 55fa36: fmv.w.x fa5,a5` 与 `0.00 : 55fa44: fmadd.s fs0,fa5,fa4,fs0`（scalar FMA）应被 `vfmacc*` 取代；
  - `3.12 : 55fe9e: slli s10,s10,0x1`、`1.56 : 55feaa: andi a3,a3,31`、`3.12 : 56026e: j 55f9da`（fp16 软件转换）应被 `vfwcvt.f.f.v`/`fcvt.s.h` 取代，`scalbnf@plt` 调用消失。
- 应出现侧（锚定 pattern §Verification）：
  - `rvv_floating_point_matmul_and_gemv_kernels.md` §Verification：出现预期 `vfmacc*`/vector load；scalar FMA/loop-control share 下降；cycles/FLOP 或 throughput 改善；覆盖 M/N/K=0/1/VLEN 边界与 tail；FP16/FP32 累加精度、NaN/Inf/±0/subnormal 对照 reference 通过。
  - `no-vectorization.md` §Verification：同一函数 annotate 出现 `vsetvli`/`vle*`/`vse*`/`vfmacc*`；scalar 指令不再主导 hot loop。
  - `rvv_precision_conversion_kernels.md` §Verification：确认主转换 loop 出现 `vfwcvt.*`/scalar `fcvt.*.h`；特殊值（denormal/NaN/±Inf）与 rounding mode 逐 lane 对照 scalar reference 一致；hot loop 内 vtype 稳定。
- 验证方法：以 `-march` 含 `v/zvfh`（实测验证的精确值）重建；对 ip_f16_upstream 与代表性 short/medium shape 重跑 `perf annotate` 与本 benchmark；对比 scalar reference 数值。
- 注意：本函数 entry 条件为 profile-backed；若在真实 workload 中按 dispatch 验证，需确认新 kernel 被 runtime 采用（本 finding 不进入 missing `.S` 分支，故按 compiler/intrinsic 载体验证）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 | ✅ | 1/1 组；std::_Function_handler<...ref_inner_product_fwd_t...lambda#2>::_M_invoke |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表；gap 标签：baseline_gap: build ISA、baseline_gap: sampling metadata、baseline_gap: sampling IP precision、source_context_gap；Sampling IP precision 分行 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；Classes scanned: rows-operator-rvv.md, rows-codegen.md；顶层 finding 1（evidence 锚点 55ffd8/55ffdc/560236/560148/55f4f4/55f500/55fa36/55fa44/55fa48）；supporting 1（55fa36/55fa44/55fa48）；排除 4 条（quantized/precision-conversion/kernel-selection/cache-blocking，含判别观察）；推导式 2 组 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件：patterns/rvv_floating_point_matmul_and_gemv_kernels.md（row: RVV Floating-Point Matmul and GEMV；引用短语：沿输出列向量化 / 增加输出行 accumulator / 比较 N 维和 K 维方案）、patterns/no-vectorization.md（引用短语：vector unit 没有处理 hot main-loop）、patterns/rvv_precision_conversion_kernels.md（引用短语：fcvt.s.h / vfwcvt.f.f.v / mask 检测特殊值）；The fix 含 before/after、correctness、风险、Profile 信号锚点；Related PRs：31 条 URL（matmul）+ 16 条 URL（no-vectorization） |
| 5 | 路径合规：8 项 trace 可解释扫描集；零/多命中与 L0–L4 合规；blueprint leaf 均来自通过 gate 的 row；入口模式 A 动态份额排序 | ✅ | 模式 A（profile_backed）；路径：L1 primary + supporting；class 列表见 Phase 3 trace |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：55ffd8/55ffdc/560236/560148/55fa36/55fa44/55fe9e/55feaa/56026e；出现侧：rvv_floating_point_matmul_and_gemv_kernels.md §Verification、no-vectorization.md §Verification、rvv_precision_conversion_kernels.md §Verification |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成 | ✅ | 交付物止于 Profile 证据、根因蓝图、The fix、验证预测；无向用户追问 |

修正记录：无