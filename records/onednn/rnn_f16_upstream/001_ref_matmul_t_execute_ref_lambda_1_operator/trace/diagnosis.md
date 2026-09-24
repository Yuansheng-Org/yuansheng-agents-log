Functions under analysis: [dnnl::impl::cpu::matmul::ref_matmul_t::execute_ref(...)::\{lambda(long const*, long, long)#1\}::operator()(long const*, long, long) const]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（rank 001，`ref_matmul_t::execute_ref` 的 `ker` lambda，40 samples，`cpu-clock:u`，percent: local period）
- perf stat（可选 bound/context）：已提供（`8-onednn-benchdnn-benchmark-riscv-rnn_f16_upstream.txt`，详见 Phase 1）
- workload/binary/DSO/source context：部分提供（libdnnl.so.3.14，annotate 内含 DWARF 源行，映射到 `src/cpu/matmul/ref_matmul.cpp:169-237`；metadata 警告 `No ELF executable binaries were found` → `source_context_gap: 无独立 ELF 可执行文件`）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：缺失（无 ELF object 可查；详见 Phase 1 `baseline_gap: build ISA`）
- hardware ISA（cpuinfo/hwprobe）：已提供（metadata cpuinfo isa 行，含 `v`/`zfh`/`zfhmin`/`zba`/`zbb`/`zbc`/`zbs` 等）
- `vlenb`：已提供（metadata vector: vlenb=16 → VLEN=128 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock:u`；percent=local period；单运行窗口 40 samples；函数级 workload 贡献未知 → 见 Phase 1 采样语义 gate）
- 采样 IP precision：缺失（无 `precise_ip`/Exact-IP 信息 → `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcv_zicbom_..._zfa_zfh_zfhmin_zba_zbb_zbc_zbs_zve64d_zvfh_zvfhmin_...` → 暴露 RVV 1.0 `v` 与 `zfh`/`zvfh` |
| Build ISA | `baseline_gap: build ISA`；annotate 中承载热点地址的整个函数零 `v*`/`th.v*`，libdnnl.so.3.14 无 ELF 文件可供 readelf；可选命令 `readelf -A libdnnl.so.3.14`（对应 build 机器上的原始库） |
| Vector flavor | annotate 全函数零 `v*` 且零 `th.v*`（全 scalar）；无 flavor mismatch 需要判定，但该零向量形态本身是 baseline finding 的下游症状 |
| VLEN | 已提供：vlenb=16 → VLEN=128 bits（SEW=32/m1 → 4 lanes；SEW=16/m1 → 8 lanes） |
| Bound type | 已提供：IPC=1.271；L1_dcache_load_miss_rate=0.200%，LLC_load_miss_rate=17.148%，branch_miss_rate=0.155%，cache_miss_rate=100.019%（cache_misses>cache_references，counter 异常不可用）→ compute/latency-bound（非 memory-bound；热点非 cache 主导） |
| Sampling semantics | event=`cpu-clock:u`；percent=local period；同一运行窗口；函数占 workload 贡献未知 → 收益上界只能表述为「当前 sampled event 下函数内局部样本份额」 |
| Sampling IP precision | `baseline_gap: sampling IP precision`；单行高占比只锚定 loop interval，不承担单指令 latency 归因 |

L0 baseline gate：hardware 有 `v`（RVV 1.0），build ISA 未知（`baseline_gap: build ISA`）；无 `th.v*`，无 vector_flavor_mismatch。hot main loop 全 scalar + zero `v*` + hardware 暴露 `v`，且该函数是 ref/fallback 路径（见 Phase 3 kernel-selection finding），构成置顶 baseline finding：该 workload 的 matmul 执行在标量参考实现上。
Bound-type gate：compute/latency-bound，本地 RVV 重写不被 memory-bound 否决；但 sampling IP precision 缺失使本轮所有 instruction-level 归因封顶为 interval-level。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个）。hot loop 锚点：k-loop interval `5f2e2a–5f3438`（`for k in group_k` 的 matmul 微内核），最高占比行 `10.00 : 5f2dc0: sd a3,0(a4)`（每次 lambda 调用的 setup：`weights_dims_idx[ndims-1] = n`）；k-loop 内部热行为包括：
- `7.50 : 5f2f40: bne t1,a4,5f2f32`（src `memory_desc_wrapper::off_v` 外层 dims 循环结束分支）
- `5.00 : 5f2fcc: blez a3,5f303c` + `7.50` 区间（weights `off_v` 的 inner_nblks/div/rem 路径）
- `7.50 : 5f3420: flw fa4,-1232(s0)`（`acc += s*w` 中 s 的栈重载）
- `7.50 : 5f3852: slli s9,s9,0x1`、`7.50 : 5f3868: bnez a4,5f438e`、`5.00 : 5f43ca: sd a5,-1232(s0)`（src/weights f16→f32 软件位操作转换）
annotate 覆盖完整（40/40 samples 落在该区间）。`baseline_gap: sampling IP precision` → 上述行只作为 basic block / loop interval 锚点。

## Phase 3 — Pattern scan / 模式扫描：dnnl::impl::cpu::matmul::ref_matmul_t::execute_ref(...)::{lambda(long const*, long, long)#1}::operator()

### Class selection trace（8 项）
1. `rows-asm.md` — exclude — 当前代码来源是 compiler-generated C++（DWARF 源行直达 `ref_matmul.cpp` lambda），非手写 `.S`；且仓库存在 RVV matmul 实现（policy-backed missing `.S` 的「目标实现缺失」证不成立）
2. `rows-operator-rvv.md` — include — compiler-generated scalar 循环，语义为 nested M/N/K FP16 matmul 微内核（必选 class；逐行评估全部 20 rows）
3. `rows-string-memory.md` — exclude — 非 copy/fill/sentinel/compare/checksum 语义
4. `rows-vectorized-tuning.md` — exclude — annotate 零 `v*`，无 RVV 配置/寄存器/展开对象
5. `rows-codegen.md` — include — dispatch 可达性证据：profile 符号证明请求进入 reference/fallback 而仓库存在更高优先级 RVV 内核（kernel-selection row）；评估跨 pass 流量与 codegen rows
6. `rows-offload.md` — exclude — C920v2/SG2044 ISA 无矩阵引擎（无 AME/MME/RVM）、无 packed-SIMD P 扩展证据
7. `rows-crypto.md` — exclude — 非密码学原语
8. `rows-runtime-os.md` — exclude — 非 kernel/RTOS/timer 热点

### Local performance pattern scan: ref_matmul_t::execute_ref::ker

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Kernel Selection and Runtime Specialization（primary） | profile 符号为 `ref_matmul_t::execute_ref`（impl 列表 `cpu_matmul_list.cpp:102` 的最后 fallback）；同一列表 `:96-97` 中 `rvv_brgemm_matmul_t`、`rvv_matmul_t` 优先级更高；热循环内 `off_v` 的 inner-block `divw/remw/div/rem` 路径被大量采样 → ref 核在 blocked layout 上逐元素重算偏移 | Medium | Medium | `patterns/kernel_selection_and_runtime_specialization.md` |

Supporting evidence（同一机制的次级命中，随 primary）：
- **No vectorization（supporting）**：(a) hot main loop 全 scalar（`flw`/`fmadd.s fs0,fa5,fa4,fs0`@5f342a/`fsw`），zero `v*`、zero `th.v*`，hardware 暴露 `v`；supporting because: 只解释「被选中的 ref 路径没有任何向量执行」，不决定贡献载体；`patterns/no-vectorization.md`
- **RVV Floating-Point Matmul（supporting）**：(a) k-loop 是 FP16 dot-product 微内核，`7.50 : 5f3420: flw fa4,-1232(s0)` + `fmadd.s`（累加器 `fs0` 依赖）+ per-element `off_v`（`5f2f40`/`5f2fcc`）与软件 f16 转换（`5f3852`/`5f3868`/`5f43ca`）共享样本；supporting because: 该 loop 正是浮点 matmul 合同（无 int8 zero-point/requantization），ref 内核的向量化/强度削减是 kernel-selection 修复之外的同一机制另一载体；`patterns/rvv_floating_point_matmul_and_gemv_kernels.md`

三件套（primary）：

**(a) 逐字 evidence 引用**（k-loop interval `5f2e2a–5f3438`，40/40 samples）：
- `10.00 : 5f2dc0: sd a3,0(a4)`（setup：`weights_dims_idx[ndims - 1] = n`，每次调用写一次）
- `7.50 : 5f2f40: bne t1,a4,5f2f32`（src `off_v` dims 循环结束分支）
- `5.00 : 5f2fcc: blez a3,5f303c`（weights `off_v` inner_nblks>0 判定）
- `2.50 : 5f2eda: remw a6,a0,a7` / `2.50 : 5f2ee2: divw a0,a0,a7`（src `off_v` 32-bit 除/余；64-bit 路径 `5f2f0a: rem`/`5f2f12: div`）
- `7.50 : 5f3420: flw fa4,-1232(s0)`（`acc += s*w` 的 s 重载）
- `7.50 : 5f3852: slli s9,s9,0x1`、`7.50 : 5f3868: bnez a4,5f438e`、`5.00 : 5f43ca: sd a5,-1232(s0)`（`io::load_float_value` f16→f32 软件解码：无 `fcvt.s.h`/`vfwcvt`）
- `2.50 : 5f342e: blt s11,a5,5f2e2a`（k-loop 回边）
采样语义：`cpu-clock:u` local period；`baseline_gap: sampling IP precision` → 以上只锚定 loop interval。

**(b) 互斥邻居排除**：
- 非 `runtime_isa_specific_instruction_dispatch`：硬件 isa 行直接列出 `v`/`zvfh`（metadata cpuinfo），无「能力已具备但 dispatch 未接入」的独立证据缺口——问题不在 feature-detection 接线。
- 非 `policy_backed_missing_riscv_assembly_kernel`：四证中「目标实现缺失」不成立——`src/cpu/rv64/rvv_matmul.hpp`、`rvv_brgemm_matmul.hpp` 存在且注册于 `cpu_matmul_list.cpp:96-97`。
- 非「正确内核已命中」：热点位于 ref 计算体内部不构成 operator-row primary——ref 是 fallback，不是适配内核；`check_layouts`（`rvv_matmul.hpp:198-204`，仅 row/col-major）与 hp path 的 `bias_mdw.is_zero()`/default-attr 约束说明 RNN f16 matmul 的合同（blocked layout 证据来自热循环 `off_v` 的 inner-block 除/余路径）很可能不在 RVV 内核覆盖域，这正是 kernel-selection/specialization 缺口。
- 非 `no-vectorization` primary：行 30 行内互斥「本行只认领未被更具体 semantic/codegen/assembly row 解释的 generic scalar main loop」；本 loop 由 kernel-selection（更上层、L0）与 fp-matmul semantic row 解释。

**(c) 双 Confidence 推导式**：
- route：profile 符号（ref/fallback 被选中，直接）+ impl 列表优先级（源码直接）+ RVV 内核存在（源码直接）→ gate 成立；但「该内核满足 RNN 当前 dtype/layout/属性合同」为间接推断（annotate 显示 blocked-layout `off_v` 路径）→ **Medium**
- impact：VLEN=128 已知、bound=compute/latency 已知、函数内 sample share=40/40（局部）；workload 级贡献未知（采样语义四条不全：local period）→ **Medium**

仲裁小段：单 hot loop、单机制链（ref 被选中 → ref 全 scalar + 每元素 off_v + 软件 f16 转换）。因果消除测试：修复 kernel-selection（让 RVV matmul 覆盖 RNN f16 合同并命中）将整体消除 ref 标量 loop → kernel-selection 为 primary（L0，顶层认领 root cause）；no-vectorization 与 fp-matmul 为 supporting，不另立顶层 finding、不计命中数。`dynamic priority unavailable` 不适用（入口条件 A）；本函数内 evidence sample share 加总：k-loop/off_v/转换区间 = 36/40 = 0.90（局部）。

## Phase 4 — Root-cause blueprint / 根因蓝图：dnnl::impl::cpu::matmul::ref_matmul_t::execute_ref(...)::{lambda(...)#1}::operator()

对应 Phase 3 通过 gate 的 row：`Kernel Selection and Runtime Specialization`（primary）；supporting：`No vectorization`、`RVV Floating-Point Matmul`。

1. **Root cause**：RNN f16 的 matmul 请求落入 `ref_matmul_t`（`src/cpu/matmul/ref_matmul.cpp`，impl 列表最后一项），而不是树内已注册且优先级更高的 RVV 内核 `rvv_matmul_t`/`rvv_brgemm_matmul_t`（`src/cpu/matmul/cpu_matmul_list.cpp:96-97`）。依据 `patterns/kernel_selection_and_runtime_specialization.md` §Why this is slow：`优化实现不可达`（优先级/属性 gate 决定后续所有执行）与 `循环不变量被重复求值`（dtype/layout 本可在调用期固定）。被选中后 ref 核在 k-loop 中对每个 (m,n,k) 元素：
   - 用 `memory_desc_wrapper::off_v` 重新计算 src/weights 偏移——blocked layout 触发 `divw/remw/div/rem`（多周期除法链）加 `mul`/`add` 累加（约 37.5% 样本，含 `5f2f40` 7.5% + `5f2fcc` 5% 等）；
   - `io::load_float_value` 用 jump-table 分发 data_type 并对 f16 做逐位软件解码（`srli`/`andi`/`slliw`/分支处理 ee==0/ee==0x1F 特殊值，约 35% 样本，`5f3852`/`5f3868`/`5f43ca` 等）——硬件含 `zfh` 但无 `fcvt.s.h` 使用；
   - 单个标量 FMA `fmadd.s fs0,fa5,fa4,fs0`（约 10% 样本）。
   依据 `patterns/rvv_floating_point_matmul_and_gemv_kernels.md`：`浮点 microkernel 沿错误轴向量化、FMA accumulator 依赖过长` 是 ref 核内部的机制；依据 `patterns/no-vectorization.md`：hot main loop 全 scalar、zero `v*`，vector unit 未参与执行。

2. **The fix / 修复方式**（绑定真实生产决策点，仅蓝图、不实施）：
   - 当前入口调用链：`matmul::get_matmul_impl_list()`（`cpu_matmul_list.cpp:114`）→ `rvv_brgemm_matmul_t::pd_t::init`（`rvv_brgemm_matmul.hpp`）→ `rvv_matmul_t::pd_t::init`（`rvv_matmul.hpp:44-163`）→ 该 RNN f16 请求的合同未通过 `check_layouts`（`:198-204`，仅 row-major src/dst + row/col-major weights）、hp path 的 `bias_mdw.is_zero()`（`:101-103`）或 `attr()->has_default_values(smask_t::none)`（`:106-109`）→ 落入 `ref_matmul_t`（`cpu_matmul_list.cpp:102`）→ 每输出元素标量 k-loop。
   - 修复后两条路径：
     a) 扩展 `rvv_matmul_t::pd_t::init` 对 RNN f16 合同的支持——`check_layouts` 增加 RNN 权重 blocked tag（如 `ldigo`/`ldio` 及 group/batch 广播）的 layout 处理，或 hp path 放宽 zero-bias/default-attr 限制（把 bias 并入 post-op/injector 或 pre/post pass）；使 `rvv_matmul_t` 覆盖该请求，`vfmacc.vv`（SEW=16/32）+ `vfwcvt`/`fcvt.s.h`（Zvfh）沿 N/K 向量化，off_v 代之以调用前一次性的指针增量/步长预计算（对齐 §The fix「hoist invariant dispatch out of hot loop」）。
     b) 若 RNN 合同不可扩展：在 RNN 侧把 matmul 输入 reorder 到 RVV 内核支持的 row/col-major layout（可摊薄的一次性 reorder），再进入 `rvv_matmul_t`。
   - 适用前提：硬件 `v`/`zvfh` 已确认（metadata cpuinfo）；build 需以含 `v` 的 `-march` 编译 RV64 路径使 `DNNL_RV64`/`CPU_INSTANCE_RV64` 实际编入（`baseline_gap: build ISA`，需 readelf 确认）。
   - correctness contract：FP16 累加精度与 FMA contraction、NaN/Inf/±0/subnormal、per-group src/wei scales 与 zero-point 语义、bias 广播与 post-op 顺序、layout/stride 与 alias 合同不变；RNN 数值输出须逐位/按误差上限对照 ref。
   - 限制/风险：VLEN=128 下 SEW=16 每向量 8 lanes，寄存器分块收益有限；小 shape（benchdnn `rnn_f16_upstream` 若为短 K）可能出现 JIT/启动开销超过计算收益的 crossover，需要短/中/长 shape 分别验证；`implementation_shape_gap: RNN matmul 确切 dtype/layout/attr 合同与 rvv_matmul 被拒的具体 VDISPATCH gate 未知（需 DNNL_VERBOSE=1 或 dispatch 断点取证）`。
   - 修复后预期 Profile signals：annotate 中 `5f2f40`/`5f2fcc`/`5f2eda`/`5f2ee2`（off_v div/rem）、`5f3852`/`5f3868`/`5f43ca`（软件 f16 解码）、`5f3420`/`5f342e`（标量 FMA/回边）份额消失或大幅缩小；出现 `vsetvli`、`vle16.v`/`vle32.v`、`vfmacc.vv`/`vfwcvt`/`fcvt.s.h`；IPC 与 benchdnn gflops 上升。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zfh_zfhmin_zvfh...`（`v` 与 `zfh` 暴露）；build ISA=`baseline_gap: build ISA`；VLEN=128（vlenb=16）；bound type=compute/latency（IPC 1.271，L1 miss 0.2%，非 memory-bound）。

4. **收益上界**：入口条件 A；当前 sampled event（`cpu-clock:u`）下的函数内局部样本份额——k-loop/off_v/转换区间合计 36/40 = 0.90（函数内）。采样语义四条不全（percent=local period、函数 workload 贡献未知）→ 不得表述为 workload 级 Amdahl 上界。ref 路径被 RVV 内核替换的收益以该函数为容器；若修复仅作用于 ref 内部向量化，同区间 0.90 为该方向上限。

5. **三维路由判定**：
   - `current source`：compiler-generated C++（`ref_matmul.cpp:169-237` lambda，DWARF 源行内联 off_v/load_float_value）——非 `.S`、非 JIT。
   - `implementation existence/reachability`：存在且已注册（`rvv_matmul_t`/`rvv_brgemm_matmul_t`，`cpu_matmul_list.cpp:96-97`，`CPU_INSTANCE_RV64`）但未命中该请求；拒绝 gate 为 layout/bias/attr 合同（推断，`implementation_shape_gap` 标注）。
   - `function-level policy`：impl 列表顺序 + `pd_t::init` 的 `VDISPATCH_MATMUL` gate 决定选择；无独立 `.S` policy 参与。
   - 不进入 missing `.S` 分支（目标实现存在）。

6. Implementation-shape proof：不适用（非 policy-backed missing `.S`）。

7. **Related PRs 小节**：
   - `patterns/kernel_selection_and_runtime_specialization.md` → Related PRs：14 条 URL（oneDNN [cc92e4b29240](https://github.com/uxlfoundation/oneDNN/commit/cc92e4b292403096ddd78046914eec0d61147727)、[9233aa44f246](https://github.com/uxlfoundation/oneDNN/commit/9233aa44f24630b959e72417b8ecccabbec4c6a5)、[#4463](https://github.com/uxlfoundation/oneDNN/pull/4463)、[e19dc70ec1fb](https://github.com/uxlfoundation/oneDNN/commit/e19dc70ec1fb50dc23203386ce09e6e537a4b0b7)、[#4363](https://github.com/uxlfoundation/oneDNN/pull/4363)、[276cb7cd00e8](https://github.com/uxlfoundation/oneDNN/commit/276cb7cd00e868e4a7e1e34c5848efbd09be4efd)、[#5453](https://github.com/uxlfoundation/oneDNN/pull/5453)、[b2f18637a7da](https://github.com/uxlfoundation/oneDNN/commit/b2f18637a7da6e2bd2739b84f1db6cd529396de4)、[ec2bfa125433](https://github.com/uxlfoundation/oneDNN/commit/ec2bfa1254336d52fc95af71ea6dcbe7e8231766)、OpenJDK [#21083](https://github.com/openjdk/jdk/pull/21083)、OpenBLAS [ef8e7d0279df](https://github.com/OpenMathLib/OpenBLAS/commit/ef8e7d0279dfd1f9d9bec32b514a853d10bfdda7)、OpenBLAS [bef47917bd72](https://github.com/OpenMathLib/OpenBLAS/commit/bef47917bd72f35c151038fee0cf485445476863)、oneDNN [#4620](https://github.com/uxlfoundation/oneDNN/pull/4620)、[#4945](https://github.com/uxlfoundation/oneDNN/pull/4945)、[#5403](https://github.com/uxlfoundation/oneDNN/pull/5403)）——计入 primary。
   - `patterns/rvv_floating_point_matmul_and_gemv_kernels.md`（supporting）→ Related PRs：oneDNN RVV matmul 实现 [3bac96b](https://github.com/uxlfoundation/oneDNN/commit/3bac96b8bc1fc9c348c986f38f65285693943d2f)、[8b48a77](https://github.com/uxlfoundation/oneDNN/commit/8b48a77091062ce78959ccb96e43a8ee4e97022d)、[#4410](https://github.com/uxlfoundation/oneDNN/pull/4410)、[#4414](https://github.com/uxlfoundation/oneDNN/pull/4414)、[#4545](https://github.com/uxlfoundation/oneDNN/pull/4545)、[#4770](https://github.com/uxlfoundation/oneDNN/pull/4770)、[#4824](https://github.com/uxlfoundation/oneDNN/pull/4824)、[#4840](https://github.com/uxlfoundation/oneDNN/pull/4840)、[#4850](https://github.com/uxlfoundation/oneDNN/pull/4850)、[#5157](https://github.com/uxlfoundation/oneDNN/pull/5157)、[#5294](https://github.com/uxlfoundation/oneDNN/pull/5294)、[#5405](https://github.com/uxlfoundation/oneDNN/pull/5405) 等；OpenBLAS [0a967797a156](https://github.com/OpenMathLib/OpenBLAS/commit/0a967797a15617239523053633bf14be7895b25a)、[0acb60aab3c0](https://github.com/OpenMathLib/OpenBLAS/commit/0acb60aab3c0134e879a68292904d8346dcd50ef) 等。
   - `patterns/no-vectorization.md`（supporting）→ Related PRs：OpenCV scalable RVV intrinsics [#22179](https://github.com/opencv/opencv/pull/22179)、[#22520](https://github.com/opencv/opencv/pull/22520)、[#23980](https://github.com/opencv/opencv/pull/23980) 等。

## Phase 5 — Verification forecast / 验证预测：dnnl::impl::cpu::matmul::ref_matmul_t::execute_ref(...)::{lambda(...)#1}::operator()

primary（Kernel Selection）验证：
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`5f2f40: bne t1,a4,5f2f32`、`5f2fcc: blez a3,5f303c`、`5f2eda: remw`、`5f2ee2: divw`、`5f3420: flw fa4,-1232(s0)`、`5f342e: blt s11,a5,5f2e2a`、`5f3852: slli s9,s9,0x1`、`5f3868: bnez a4,5f438e`、`5f43ca: sd a5,-1232(s0)` 的样本份额显著下降或消失。
- 应出现侧（锚定 `patterns/kernel_selection_and_runtime_specialization.md` §Verification）：`perf annotate` 同一符号/调用点出现 `rvv_matmul_t`（`jit:rvv`）执行，出现 `vsetvli`、`vle16.v`/`vle32.v`、`vfmacc.vv`（或 `vfwcvt`/`fcvt.s.h`）指令；用 `DNNL_VERBOSE`/实现标识确认真实路径进入 RVV 内核而非仅代码存在；ref_matmul sample share 下降。
- 覆盖：f16 `is_hp_path_` 的 M/N/K 边界（0、1、VLEN 倍数、tail）、bias/scale/attr 组合、小/中/大 shape 的 crossover；对照 reference 校验 FP16 累加精度与 FMA contraction、NaN/Inf/±0/subnormal；确认不兼容请求仍回退 ref 且结果正确。
- 本函数如需把结论从「probable」升级为「confirmed」：最少需采集 `DNNL_VERBOSE=1` 下的 impl 选择日志（确证 rvv_matmul 被拒的具体 gate），以及全 workload 的 function-level sample share。

supporting（No vectorization / FP matmul）不单独验证，跟随 primary 的验证预测。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅ | `1/1 组；ref_matmul_t::execute_ref::ker` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行；两个 L0 gate 判定 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding=1（kernel-selection）；evidence 锚点 `10.00 : 5f2dc0: sd a3,0(a4)`、`7.50 : 5f2f40: bne`、`7.50 : 5f3420: flw`、`7.50 : 5f3852: slli`、`7.50 : 5f3868: bnez`、`5.00 : 5f2fcc: blez`、`5.00 : 5f43ca: sd`；supporting=2（no-vectorization、fp-matmul）；排除条数=4（runtime-dispatch、missing-.S、operator-primary、no-vec-primary）；推导式=2（route/impact） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`kernel_selection_and_runtime_specialization.md`（引用「优化实现不可达」「hoist invariant dispatch」）、`rvv_floating_point_matmul_and_gemv_kernels.md`（引用「浮点 microkernel 沿错误轴向量化、FMA accumulator 依赖过长」）、`no-vectorization.md`；`The fix` 含 before/after 调用链、correctness、风险、Profile signals；`implementation_shape_gap: RNN matmul 合同与 rvv_matmul 被拒 gate 未知`；Related PRs 已按 pattern 分组输出 URL |
| 5 | 路径合规：trace 可解释扫描集、零/多命中与 evidence-mechanism layer 合规 | ✅ | 模式=A（profile_backed）；8-class trace 全覆盖；kernel-selection=L0 primary，supporting=L1（no-vectorization）/L1（fp-matmul）；收益上界仅局部份额 0.90（函数内） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧=`5f2f40`/`5f2fcc`/`5f2eda`/`5f3420`/`5f3852`/`5f3868`/`5f43ca`/`5f342e`；出现侧=pattern §Verification（`vfmacc`/`vle16`/`fcvt.s.h` + `DNNL_VERBOSE` 确认真实路径） |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁 | ✅ | 交付止于 evidence + 蓝图 + The fix + 验证预测；gap 均标注不追问 |

修正记录：无
