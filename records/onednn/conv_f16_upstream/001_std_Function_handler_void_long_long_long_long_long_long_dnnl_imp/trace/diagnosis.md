Functions under analysis: [std::_Function_handler<void (long, long, long, long, long, long), dnnl::impl::cpu::ref_convolution_fwd_t::execute_forward(dnnl::impl::exec_ctx_t const&) const::{lambda(long, long, long, long, long, long)#3}>::_M_invoke]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（rank 001，libdnnl.so.3.14，`cpu-clock:u`，288527 samples，percent: local period，3951 行，含 hot ic loop body 与两套 load_float_value dispatch 链）
- perf stat（bound/context）：已提供（conv_f16_upstream 共享 perf stat：IPC 0.688、branch_miss_rate 0.29%、L1_dcache_load_miss_rate 0.021%、LLC_load_miss_rate 15.78%、benchdnn_min_gflops 4.051）
- workload/binary/DSO/source context：已提供（benchdnn 问题 `resnet_50:res2a_branch2a`，`f16:f16:f16`，direct，any:any:any；test_output.txt 实测实现 `ref:any : 1 (100%)`；源码树 commit d22de940 提供 ref_convolution.cpp 与 rv64 kernel 实现）
- readelf -A（热点 object 的 Tag_RISCV_arch）：缺失（metadata `binaries: {}`，未采集 ELF；详见 Phase 1 `baseline_gap: build ISA`）
- hardware ISA（/proc/cpuinfo）：已提供（`rv64imafdcv_..._zfa_zfh_zfhmin_..._zve32f_zve64d_zve64f_zvfh_zvfhmin_...`，含 `v`、`zfh`、`zvfh`）
- vlenb：已提供（16 bytes → VLEN 128 bits，RVV 1.0）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock:u`（时间类），freq=999，单次运行窗口；annotate 内 percent=local period；flat perf_report 全局 Overhead 91.71% 与函数内 288527/314624=91.71% 一致）
- Sampling IP precision（precise_ip / skid）：缺失（metadata.txt 未记录 precise_ip；`baseline_gap: sampling IP precision`，单行占比只锚定 interval，不做 instruction-latency 归因）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `v`（RVV 1.0）、`zfh`、`zvfh`、`zfhmin`、`zbb/zba/zbc/zbs`、`zicond`、`zfa` 均暴露（cpuinfo isa line，SG2044 / XuanTie C920v2） |
| Build ISA | `baseline_gap: build ISA`；annotate 显示热点函数全 scalar 且 f16→f32 用软件位操作（srliw/slli/or），说明当前执行体未使用 `v`/`zfh`；但无法确证是 build `-march` 缺扩展还是代码路径未选。可选命令：对 `/workspace/build/src/libdnnl.so.3.14` 运行 `readelf -A` |
| Vector flavor | annotate 无任何 `v*` 也无可 vendor `th.v*`（grep vsetvli/vle16/vle32/vfmacc/vfmul/vfadd/vse16/vse32 = 0 命中）；全 scalar |
| VLEN | 128 bits（vlenb=16，metadata vector.vlen_bits=128） |
| Bound type | compute/latency-bound：IPC=0.688；L1_dcache_load_miss_rate=0.021%（数据驻留 L1）；branch_miss_rate=0.29%；LLC_load_miss_rate=15.78%（LLC 流量本身极小，2.1M 级）；cache_miss_rate=100% 为 counter 别名伪影（cache_references==cache_misses），不采信 |
| Sampling semantics | event=`cpu-clock:u`（时间类）；annotate percent=local period；flat perf report 提供同窗口全局 Overhead（91.71%）；函数 workload 贡献已知（91.71%）→ 满足四条中的三条+全局份额；收益上界可表述为 workload 级局部份额上限 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（未记录 precise_ip/Exact-IP）；单条高占比行只能锚定 ic loop interval，不能把 `blt` 单行 21.20% 直接解释为该指令的 cycle cost |

L0 baseline gate：hardware 有 `v`（RVV 1.0）而当前执行体零 `v*` → 属于「hardware 有 v、执行体无 v」基线发现，置顶保留，但不停扫。无 `th.v*`，flavor gate 不触发。Bound-type gate：compute/latency-bound 成立 → 本地计算侧向量化修复的 impact 判断有 baseline 支撑；memory-bound 降级不适用。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致。该函数是 `ref_convolution_fwd_t::execute_forward` 经 `parallel_nd(G, MB, OC, OD, OH, OW, ...)` 分发的 OpenMP lambda `#3`，被 `std::_Function_handler::_M_invoke` 包装调用；源码对应 `src/cpu/ref_convolution.cpp` 的 `ker_plain`（`is_plain && src_ic_stride==1 && weights_kw_stride==1` 分支）。

hot loop 锚点（ic 内层循环，interval 52fa2a–52fb36）：
- 最高行：`21.20 : 52fb36: blt s3,a5,52fa2a`（循环回边）
- 次高行：`9.23 : 52fae8: add a5,a5,s5`（weights load_float_value 跳转表 base 加法）
- 其余 ~2.4–5.7% 行分布在偏移 mul 链（52fa36/42/52/60/6e/74）、dispatch（52fa7e/88/8e、52fade/e8/ea）、f16 转换（52fb94/bac、52fcb8、52fdf6/fe08/fe20）
- 采样覆盖完整（含 hot loop body）；Sampling IP precision 不足 → 归因到 interval 级（ic loop 整体机制），不做单指令 latency 断言

非 plain 通用 `ker` 路径（52e06e–52e354）及 setup 段样本均为 0.00%，按地址分账不参与热点归因。

## Phase 3 — Pattern scan / 模式扫描：std::_Function_handler<...ref_convolution_fwd_t::execute_forward...lambda#3>::_M_invoke

### Class selection trace

1. rows-asm.md：exclude — 当前代码来源是 compiler-generated C++（lambda 内联展开），无 `.S`/DWARF 证明；missing-`.S` policy 四证不适用（rv64 已有现存向量 kernel，问题在可达性不在载体缺失）
2. rows-operator-rvv.md：include — V-capable 硬件上 compiler-generated scalar 热点循环，语义为 1x1 卷积（IC-reduction / GEMM-like）；`no-vectorization` 及 spatial-conv 语义行相关
3. rows-string-memory.md：exclude — 热点循环无语义 string/memory copy/scan/compare 特征
4. rows-vectorized-tuning.md：exclude — annotate 零 `v*`，无 RVV 配置/寄存器/展开对象可调
5. rows-codegen.md：include — 分派证据（impl=ref:any）对应 kernel-selection；热点循环内出现循环级 index scaling 与跳转表 dispatch、多指令 f16 转换序列（codegen 微结构）
6. rows-offload.md：exclude — SG2044 C920v2 无矩阵引擎证据；无独立权重重排预处理层热点
7. rows-crypto.md：exclude — 非密码原语
8. rows-runtime-os.md：exclude — 用户态 benchdnn workload，无 timer/ISR/CSR/权限域热点

### Local performance pattern scan: std::_Function_handler<...lambda#3>::_M_invoke

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Kernel Selection and Runtime Specialization（primary, L0） | test_output.txt `perf,cpu,ref:any,"resnet_50:res2a_branch2a",...` 与 `\| ref:any : 1 (100%) \|`；f16 impl list 注册了 rv64 向量 kernel（cpu_convolution_list.cpp:201-203）但 pd gate 拒绝 f16 dst | High | High | `patterns/kernel_selection_and_runtime_specialization.md` |
| No vectorization（supporting, L1） | ic loop 全 scalar：`21.20 : 52fb36: blt s3,a5,52fa2a`、`0.02 : 52fb32: fmadd.s fs0,fa5,fa4,fs0`；全函数零 `v*`；hardware 有 `v`+`zvfh` | High | —（supporting） | `patterns/no-vectorization.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（independent, L4） | f16→f32 软件位操作序列：`2.37 : 52fbac: bnez a3,52fe16`、`2.46 : 52fe08: or a5,a5,a4`、`2.20 : 52fe20: slli a3,a3,0x17`；hardware 有 `zfh` | Medium | Medium | `patterns/isa_extension_specific_instruction_substitution.md` |
| Loop Induction Variable Strength Reduction（independent, L4） | 每轮重算偏移：`2.62 : 52fa36: mul a0,s9,a0`、`2.62 : 52fa42: mul a4,s10,a4`、`2.60 : 52fa52: add a0,a0,a3`、`2.65 : 52fa74: ld a4,-488(s0)`（循环不变 stride 每轮重载） | High | Medium | `patterns/loop_induction_variable_strength_reduction.md` |

#### Finding 1（primary）：Kernel Selection and Runtime Specialization

**(a) 逐字 evidence 引用**
- 实现标识（test_output.txt 原文行 2-3、7）：`perf,cpu,ref:any,"resnet_50:res2a_branch2a",FWD_D,f16:f16:f16,undef,any,any,any,direct,,mb50ic64ih56oc64oh56kh1ph0...`；`| ref:any : 1 (100%) |`
- 后果侧 annotate（ref kernel 热点，即 fallback 被选中后 91.71% 样本所在）：`21.20 : 52fb36: blt s3,a5,52fa2a`（ic loop interval 52fa2a–52fb36）
- 阻断 gate（源码逐字）：`rvv_brgemm_conv_utils.cpp:107: if (!in_dt_ok || jcp.dst_dt != f32) return status::unimplemented;`；`jit_rvv_1x1_convolution.hpp:83-85: VDISPATCH_CONV((all_f32 || sym_lowp || wei_decomp) && dst_dt == data_type::f32, ...)`
- 注册证据：`cpu_convolution_list.cpp:201-203` f16 dtype 列表含 `CPU_INSTANCE_RV64(jit_uni_dwconv_fwd_t)`、`CPU_INSTANCE_RV64(rvv_brgemm_convolution_fwd_t)`、`CPU_INSTANCE_RV64(jit_rvv_1x1_convolution_fwd_t)`，其后才是 `CPU_INSTANCE(ref_convolution_fwd_t)`；测试为 kh=1（1x1）regular conv（Gops=1.28451e9 = 2×50×64×64×56×56），jit_uni_dwconv 不适用（非 depthwise）

**(b) 互斥邻居排除**
- runtime-ISA-dispatch row：排除 — hardware ISA 已确认含 `v`、`zvfh`（cpuinfo isa line），非 feature-detection 缺口；阻断点是 dtype gate（dst f32），不是 ISA 检测不可达
- operator/no-vectorization row：排除其 primary 地位 — 仓库 f16 impl list 中已注册适配 f16 src/wei 的向量 kernel（zvfh widening FMA），并非「没有可用专用实现」；问题在 f32-dst gate 使其实不可达，属「已有适配实现因属性 gate 未选中」→ kernel-selection primary、no-vectorization supporting（row 行内互斥判据）
- policy-backed missing `.S`：排除 — 目标实现形式为现存 JIT/intrinsic kernel 而非 `.S`，无 dispatch-slot/policy 四证
- JIT-generated-code-quality row：排除 — 热点在 compiler-generated ref kernel，不在 JIT emitter 输出

**(c) 双 Confidence 推导式**
- route: test_output impl=ref:any（直接 provenance）+ f16 impl list 注册（存在性）+ 源码 f32-dst gate（可达性阻断）→ High
- impact: 91.71% workload 全局样本份额 + compute-bound（L1 miss 0.021%）+ VLEN 已知 + dispatch baseline（ref:any）→ High

#### Finding 2（supporting）：No vectorization

**(a) 逐字 evidence 引用**
- `21.20 : 52fb36: blt s3,a5,52fa2a`（ic loop interval 52fa2a–52fb36，标量回边）
- `0.02 : 52fb32: fmadd.s fs0,fa5,fa4,fs0`（每 64 次迭代一次标量 FMA 累积）
- 全函数 annotate 中 vsetvli/vle16/vle32/vfmacc/vfmul/vfadd/vse16/vse32 命中数为 0；hardware ISA 含 `v`

**(b) 互斥邻居排除**
- 更具体 semantic row（spatial-convolution/matmul）：本行不认领 primary — 1x1 conv 语义由 kernel-selection 上游解释；且 row 行内判据「现有适配 kernel 未选中 → kernel-selection primary；本行 supporting」直接适用
- vectorized-tuning rows：排除 — 无 `v*` 无 vtype/寄存器配置对象

**(c) 双 Confidence 推导式**
- route: scalar main loop + zero `v*` + hardware `v` 直接确认 → High
- impact: 作为 supporting 跟随 primary，不单独计

#### Finding 3（independent）：RISC-V ISA Extension-Specific Instruction Substitution

**(a) 逐字 evidence 引用**（f16→f32 软件转换 + dispatch 链，ic loop interval 内）
- `2.37 : 52fbac: bnez a3,52fe16`（f16 exponent==0 检查）
- `2.46 : 52fe08: or a5,a5,a4`、`2.20 : 52fe20: slli a3,a3,0x17`（指数/尾数拼装）
- `9.23 : 52fae8: add a5,a5,s5`、`2.45 : 52fa8e: jr a5`（load_float_value 跳转表）
- `5.16 : 52fb94: slli a5,a0,0x1`（f16 元素偏移 ×2）
- 硬件 `zfh`/`zvfh` 暴露（cpuinfo isa line），原生 `fcvt.s.h` 单指令可替代该 ~8–10 指令 + 分支序列

**(b) 互斥邻居排除**
- eliminate-unnecessary-precision-conversions row：排除 — 无 float32↔float64 往返，语义是必要且正确的 f16→f32 转换
- floating-point-semantic-lowering row：排除 — 无 FCSR save/restore、NaN-boxing 或 rounding 语义维护开销；纯 portable C++ 位操作实现
- RVV-precision-conversion row：排除 primary — 本 finding 认领 scalar 指令选择；批量向量转换（vfwcvt）属 primary fix 的新 kernel 设计域

**(c) 双 Confidence 推导式**
- route: hardware `zfh` 直接确认 + annotate 显示多指令 emulation → 但 build ISA 未知（`baseline_gap: build ISA`，无法确证目标二进制能否发 `fcvt.s.h`）→ Medium
- impact: 转换+dispatch 区间函数内局部份额约 40%（dispatch ~14.7%、转换 ~13.2%、f16 load ~5.5%），bound type 已知；但 build/dispatch baseline 缺失 → Medium

#### Finding 4（independent）：Loop Induction Variable Strength Reduction

**(a) 逐字 evidence 引用**（ic loop interval 52fa2a–52fa7e）
- `2.62 : 52fa36: mul a0,s9,a0`、`2.62 : 52fa42: mul a4,s10,a4`、`2.60 : 52fa52: add a0,a0,a3`、`2.53 : 52fa60: mul a4,a3,a4`、`2.61 : 52fa6e: add a0,a0,a3`
- `2.65 : 52fa74: ld a4,-488(s0)`、`0.05 : 52fa54: ld a3,-496(s0)`（循环不变 stride 每轮从栈重载）
- 对 1x1 conv（KD=KH=KW=1, pad=0）src_off 恒等于 ic（src_ic_stride=1），该 mul/add 链本可折叠为指针递增，但编译器按运行时 dims 保留通用计算

**(b) 互斥邻居排除**
- algebraic-simplification row：排除 — 非 peephole 恒等式，是循环级归纳变量形态
- zero-based-comparison row：排除 — 问题在地址重建不是比较对象（`blt s3,a5` 已是普通边界比较）
- native-width/native-word rows：排除 — 无窄整数 truncate/extension 语义

**(c) 双 Confidence 推导式**
- route: annotate 直接显示每轮 index scaling + loop-invariant reload → High
- impact: 函数内局部份额约 20%（offset 链 ~15.7% + 循环控制 21.2% 中与地址链耦合部分）；存在竞争 bottleneck（primary kernel-selection 修复会整体替换该循环）→ Medium

### 多候选仲裁小段

- 因果层次归属：primary=Kernel Selection（L0，分派层根因：f16 dst 使向量 kernel 不可达，ref:any 被选中，一切下层 scalar signal 由此而生）；supporting=No vectorization（L1，被选中 ref kernel 的标量机制，自身 row gate 成立但由 L0 认领 primary）；independent×2=ISA-substitution 与 Loop-IV（L4，ref kernel 内可分离的指令选择与循环形态机制，各自有独立修复对象）。
- 因果消除测试：若 primary 修复（f16-dst 向量 kernel 可达）落地，ref kernel ic loop 消失 → 两个 L4 signal 随之消失；故 L4 为「primary 不落地时仍有效的独立修复杠杆」，保留 independent 并标注条件。L4 与 primary 证据地址集合（fallback 执行体 vs 分派记录）可分账。
- 收益上界排序（入口条件 A）：primary 覆盖函数整体 91.71% workload 份额（flat report 全局 Overhead）；两个 independent 仅覆盖函数内局部份额（转换+dispatch ≈ 40% local、offset+控制 ≈ 20% local），且在新 kernel 落地后归零。按证据 sample share 排序：primary > ISA-substitution（conditional）> Loop-IV（conditional）。

## Phase 4 — Root-cause blueprint / 根因蓝图：std::_Function_handler<...lambda#3>::_M_invoke

### Finding 1（primary, L0）：Kernel Selection and Runtime Specialization — ref:any 因 f32-dst gate 被选中

1. **Root cause**：benchdnn 请求 `resnet_50:res2a_branch2a`（1x1 conv, f16:f16:f16）时，oneDNN CPU f16 impl list（cpu_convolution_list.cpp:201-203）中的 rv64 向量实现全部在 pd `init()` 阶段被 dtype gate 拒绝：`rvv_brgemm_conv_utils.cpp:107` 要求 `jcp.dst_dt == f32`，`jit_rvv_1x1_convolution.hpp:83-85` 要求 `dst_dt == data_type::f32`。f16×f16→f16（dst f16）在 rv64 没有任何向量化实现可达，唯一接受该合同的 `ref_convolution_fwd_t` 被选中，其 scalar 内核承担 91.71% workload 样本。依据 `patterns/kernel_selection_and_runtime_specialization.md` §Why this is slow #1「优化实现不可达：实现选择的优先级、注册条件或属性匹配决定了后续所有执行」与 §The fix #5「Separate capability gates from target performance policy：...先由硬件/build ISA、...dtype、layout、shape 和 ABI 组成 correctness gate」。本 pattern 边界合同「候选实现已经存在但没有被选中」成立（候选已注册于 f16 列表，其 compute 路径（zvfh widening FMA）与本请求语义合同匹配，仅输出 dtype gate 阻断）。
2. **The fix / 修复方式**：修复对象是 rv64 conv 向量 kernel 的 dtype 可达性，绑定两个真实决策点：
   - `src/cpu/rv64/jit_rvv_1x1_convolution.hpp` pd_t::init（行 83-85）：对 f16 dst 增加合法路径（`dst_dt == data_type::f16` 且 `mayiuse(zvfh)`），并在 kernel store 阶段用 `vfncvt.f.f.w`（e32→e16, Zvfh）窄化写出 f16 dst；bias 仍在 f32 accumulator 域内完成。
   - `src/cpu/rv64/rvv_brgemm_conv_utils.cpp` init_conf（行 107）：把 `jcp.dst_dt != f32` 的硬拒绝改为「f16 dst + mayiuse(zvfh) → 窄化 store」。
   - 修复前调用链：`benchdnn conv f16:f16:f16 → impl_list f16 → (jit_uni_dwconv 不适用; rvv_brgemm/jit_rvv_1x1 因 dst!=f32 拒绝) → ref_convolution_fwd_t (ref:any) → scalar ker_plain`。
   - 修复后蓝图：`... → rvv_brgemm / jit_rvv_1x1 (f16×f16→f16, zvfh widening FMA + narrowing store) → vectorized kernel`；不满足条件（无 zvfh / dst 其它 dtype / post-ops 超支持域）→ 保留 ref fallback。
   - 适用前提：硬件 `zvfh`（本机 cpuinfo 已含）；`vfncvt.f.f.w` 的 rounding 为 round-to-nearest-even，需与 ref 的 float→f16 转换语义一致（oneDNN f16 转换已是 RNE）；kernel 内 vtype 切换（e32↔e16）需维护 VL/VLMAX 一致性（VLEN=128, e32 m2 accumulators 与 e16 m1 的现有设计已在 `jit_rvv_1x1_conv_kernel.hpp:78-112` 注释中体现）。
   - 不可破坏的 correctness contract：conv 数值语义（f16 输入 → f32 累积 → f16 输出，含 bias、post-ops、rounding mode）；f16 输出格式 tag（nwc/nhwc + oihw 分块权重）须与现有 set_default_formats 合同一致；`dst_dt==f16` 时写回必须窄化，不得按 f32 宽度写越界。
   - 限制/风险：1x1 kernel 只覆盖 1x1 conv（本测试命中），非 1x1 f16 conv 仍需 rvv_brgemm/其它路径或 ref；brgemm f16 路径目前 jcp 计算在 f32 accumulator，窄化 store 增加每输出一次 vfncvt，成本远低于 ref kernel 的每次迭代软件转换。
   - 修复后预期 Profile signals：该 hotspot symbol 从 ref lambda 切换到向量 kernel symbol（如 jit_rvv_1x1 或 rvv_brgemm jit kernel）；ref 函数样本份额从 91.71% 大幅下降；热点 annotate 出现 `vle16`/`vfwcvt.f.f.v`/`vfncvt.f.f.w`/`vfwmacc*`；整体 IPC 上升。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zfh_zvfh_...`（含 v）；build ISA=`baseline_gap: build ISA`（未采集 ELF）；VLEN=128；bound type=compute/latency-bound（L1 miss 0.021%, IPC 0.688）。
4. **收益上界**：当前 sampled event（cpu-clock:u）下 workload 级局部样本份额上限 ≈ 91.71%（flat perf report 全局 Overhead 与函数内 288527/314624 一致）；受新 kernel 自身效率约束。
5. **三维路由判定**：current source=compiler-generated scalar（ref_convolution_fwd_t lambda 内联，annotate 证实，非 `.S`）；implementation existence/reachability=rv64 向量 kernel 存在于树并注册于 f16 impl list，但因 f32-dst gate 对 f16 dst 不可达，ref 是唯一接受合同者 → 选中；function-level policy=无独立 `.S` policy 证据（现存实现为 JIT/intrinsic），missing-`.S` 分支不适用。
6. **Related PRs**（patterns/kernel_selection_and_runtime_specialization.md §Related PRs）：oneDNN [cc92e4b29240](https://github.com/uxlfoundation/oneDNN/commit/cc92e4b292403096ddd78046914eec0d61147727)、[9233aa44f246](https://github.com/uxlfoundation/oneDNN/commit/9233aa44f24630b959e72417b8ecccabbec4c6a5)、[#4463](https://github.com/uxlfoundation/oneDNN/pull/4463)、[e19dc70ec1fb](https://github.com/uxlfoundation/oneDNN/commit/e19dc70ec1fb50dc23203386ce09e6e537a4b0b7)、[#4363](https://github.com/uxlfoundation/oneDNN/pull/4363)、[276cb7cd00e8](https://github.com/uxlfoundation/oneDNN/commit/276cb7cd00e868e4a7e1e34c5848efbd09be4efd)、[#5453](https://github.com/uxlfoundation/oneDNN/pull/5453)、[b2f18637a7da](https://github.com/uxlfoundation/oneDNN/commit/b2f18637a7da6e2bd2739b84f1db6cd529396de4)、[ec2bfa125433](https://github.com/uxlfoundation/oneDNN/commit/ec2bfa1254336d52fc95af71ea6dcbe7e8231766)、[#4620](https://github.com/uxlfoundation/oneDNN/pull/4620)、[#4945](https://github.com/uxlfoundation/oneDNN/pull/4945)、[#5403](https://github.com/uxlfoundation/oneDNN/pull/5403)；OpenJDK [#21083](https://github.com/openjdk/jdk/pull/21083)；OpenBLAS [ef8e7d0279df](https://github.com/OpenMathLib/OpenBLAS/commit/ef8e7d0279dfd1f9d9bec32b514a853d10bfdda7)、[bef47917bd72](https://github.com/OpenMathLib/OpenBLAS/commit/bef47917bd72f35c151038fee0cf485445476863)、[03a83778bb9b](https://github.com/OpenMathLib/OpenBLAS/commit/03a83778bb9b8a869dc934511f1d13bba80d09a7)、[64401b441758](https://github.com/OpenMathLib/OpenBLAS/commit/64401b4417585711b6a6876171a9eed4fd547a26)。Related PRs：14 条 URL。

### Finding 2（supporting, L1）：No vectorization — ref kernel ic loop 全标量

1. **Root cause**：被选中的 ref `ker_plain` ic 内层循环（52fa2a–52fb36）在暴露 `v`+`zvfh` 的硬件上完全标量（零 `v*`），每个输出元素对 IC=64 做 64 次标量迭代，每次迭代约 40+ 条指令只贡献 1 次 `fmadd.s`（其样本仅 0.02%）。依据 `patterns/no-vectorization.md` §Why this is slow「vector unit 没有处理 hot main-loop 的并行元素」；本 pattern 的互斥判据将 primary 让给 kernel-selection（「已有 vector kernel 未采用时修复 selection/registration/gate，而不是重写第二份 kernel」§The fix #5）。
2. **The fix / 修复方式**：不重写第二份 kernel；由 primary fix 使现存 rv64 向量 kernel（zvfh widening FMA：`vle16` + `vfwcvt.f.f.v` + `vfwmacc*`，VLEN=128 → 每向量 8 lane f16）对 f16 dst 可达。若 ref kernel 仍需保留（例如非 1x1 f16 conv），其向量化替代应遵守 kernel-conventions（fixed-VL main loop + runtime-VL tail），修复前形态 `flw/fmadd.s/fsw`（标量逐元素）→ 修复后形态 `vsetvli/vle16/vfwmacc*/vse16`（按 SEW=e16, VLEN=128 每 8 lane 一组）。
3. **Baseline facts 回填**：同 primary（hardware v+zvfh；build ISA gap；VLEN=128；compute-bound）。
4. **收益上界**：随 primary，`dynamic priority` 不单独计。
5. **三维路由判定**：同 primary（compiler-generated scalar ref；向量 kernel 存在但不可达；无 `.S` policy）。
6. **Related PRs**（patterns/no-vectorization.md §Related PRs）：OpenCV [#22179](https://github.com/opencv/opencv/pull/22179)、[#22520](https://github.com/opencv/opencv/pull/22520)、[#23980](https://github.com/opencv/opencv/pull/23980)、[#24058](https://github.com/opencv/opencv/pull/24058)、[#24132](https://github.com/opencv/opencv/pull/24132)、[#24166](https://github.com/opencv/opencv/pull/24166)、[#24301](https://github.com/opencv/opencv/pull/24301)、[#24325](https://github.com/opencv/opencv/pull/24325)、[#27160](https://github.com/opencv/opencv/pull/27160)、[#27119](https://github.com/opencv/opencv/pull/27119)、[#27097](https://github.com/opencv/opencv/pull/27097)、[#27007](https://github.com/opencv/opencv/pull/27007)、[#26958](https://github.com/opencv/opencv/pull/26958)、[#26865](https://github.com/opencv/opencv/pull/26865)、[b902a8e792e1](https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d)、[2c16f3b7d2b2](https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a)、[e06502a254f7](https://github.com/opencv/opencv/commit/e06502a254f79f9d3184de2803d087c6914b7706)、[a2d784b6f53a](https://github.com/opencv/opencv/commit/a2d784b6f53aa1fdfde21ab8e3787a93b59af24f)、[83104bed3209](https://github.com/opencv/opencv/commit/83104bed32093ff0c5c935e8920c72b6b74ae07a)。Related PRs：19 条 URL。

### Finding 3（independent, L4）：RISC-V ISA Extension-Specific Instruction Substitution — f16→f32 用软件位操作而非 fcvt.s.h

1. **Root cause**：ref kernel 每次 load_float_value(f16) 走 ~10 条位操作 + 2 个分支（52fb94-52fe30：slli/lhu/srliw/andi/srli/andi + `bnez a3,52fe16` 与 `beq a4,a2,52fea4` + slli/or 拼装），而硬件暴露 `zfh`，原生 `fcvt.s.h` 单指令完成 f16→f32（含 subnormal 正确转换，IEEE 语义与 oneDNN 的 portable 转换一致）。依据 `patterns/isa_extension_specific_instruction_substitution.md` §Why this is slow #1「Excessive instruction count from multi-instruction emulation」与典型替换表「scalar byte reverse / sign-extend ... → 单条原生指令」的同一判据（native 单指令替代多指令 emulation）。
2. **The fix / 修复方式**：修复对象是 f16 数值转换的指令选择：
   - 修复前（annotate 原文形态）：`lhu a5,0(s7); srliw a4,a5,0xa; andi a4,a4,31; srli a3,a5,0xf; andi a5,a5,1023; bnez a4,<denormal>; slli a4,a4,0x17; lui a2,0x38000; add a4,a4,a2; slliw a3,a3,0x1f; or a5,a5,a4; or a5,a5,a3`（~10 指令 + 分支）。
   - 修复后蓝图（feature-gated 单指令）：`lhu a5,0(s7); fcvt.s.h fa5,a5`（Zfh；需 build `-march` 含 zfh 或 oneDNN zfh-guarded 实现，如同 rv64 其它 kernel 用 mayiuse(zvfh) 分流）。
   - 适用前提：build ISA 含 `zfh`（`baseline_gap: build ISA` 待 readelf 确认）；`fcvt.s.h` 的 subnormal/NaN/rounding 语义与 oneDNN `float16_t::operator float()` 逐位等价（RNE、NaN payload 合同按需核对）。
   - 不可破坏的 correctness contract：denormal 输入（ee==0, mm!=0）必须按 `scalbn(mm,-24)` 的精确值转换（fcvt.s.h 在 RISC-V 下正确完成 subnormal→normal，无需软件路径）；±0 与 Inf/NaN 类不变。
   - 限制/风险：若 build 不含 zfh，需先重建或保留 portable fallback；批量转换（vfwcvt.f.f.v）在向量 kernel 内更优，本 finding 只覆盖 scalar ref 路径。
   - 修复后预期 Profile signals：annotate 中 52fb94-52fe30 区间的 srliw/slli/or/bnez 样本消失或大幅缩小，出现 `fcvt.s.h`。
3. **Baseline facts 回填**：hardware ISA 含 `zfh`/`zvfh`（已提供）；build ISA=`baseline_gap`；VLEN=128；bound type=compute-bound。
4. **收益上界**：函数内局部样本份额 ≈ 13.2%（转换序列 52fbac/52fe08/52fe20 等行加总；不含 dispatch 链 14.7%）；若 primary 修复落地该信号归零 → 条件性收益。
5. **三维路由判定**：current source=compiler-generated scalar（ref kernel）；implementation existence/reachability=Zfh 是硬件能力，需 build 支持（gap）；function-level policy=oneDNN 已有 ISA-guarded 机制（mayiuse），无 `.S` 约束。
6. **Related PRs**（patterns/isa_extension_specific_instruction_substitution.md §Related PRs）：Go [3659b8756a2b](https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0)、[a6ecdf29e34d](https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c)、[#59488](https://github.com/golang/go/pull/59488)；OpenSSL [03ce37e11729](https://github.com/openssl/openssl/commit/03ce37e11729)、[ca6286c382a7](https://github.com/openssl/openssl/commit/ca6286c382a7)、[48b6776678d7](https://github.com/openssl/openssl/commit/48b6776678d7)、[6136408e6abf](https://github.com/openssl/openssl/commit/6136408e6abf)、[e4fd3fc379d7](https://github.com/openssl/openssl/commit/e4fd3fc379d7)、[80c664db430d](https://github.com/openssl/openssl/commit/80c664db430d)、[08c8dd6b8ced](https://github.com/openssl/openssl/commit/08c8dd6b8cede3cdbe5b1866c1a7544e0fe7a378)、[49a3e7adc392](https://github.com/openssl/openssl/commit/49a3e7adc392)、[a41f9135f082](https://github.com/openssl/openssl/commit/a41f9135f082)、[4dbb537bd1ea](https://github.com/openssl/openssl/commit/4dbb537bd1ea)、[608cadfbdbdb](https://github.com/openssl/openssl/commit/608cadfbdbdb)、[b1b889d1b3fc](https://github.com/openssl/openssl/commit/b1b889d1b3fc)、[657d1927c68b](https://github.com/openssl/openssl/commit/657d1927c68b)、[611685adc04a](https://github.com/openssl/openssl/commit/611685adc04a)、[7ae2bc9df6e0](https://github.com/openssl/openssl/commit/7ae2bc9df6e0)；Linux [e8620bd7e5e0](https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712)、[5ba15d419fab](https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49)、[cc2294d3f9c9](https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92)、[36e224168721](https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de)、[e11e367e9fe5](https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355)、[75ab93a244a5](https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb)、[c64086849110](https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68)；LLVM [#170824](https://github.com/llvm/llvm-project/pull/170824)、[#92926](https://github.com/llvm/llvm-project/pull/92926)、[#152744](https://github.com/llvm/llvm-project/pull/152744)、[#122698](https://github.com/llvm/llvm-project/pull/122698)。Related PRs：33 条 URL。

### Finding 4（independent, L4）：Loop Induction Variable Strength Reduction — ic 循环每轮重建偏移

1. **Root cause**：ic 内层循环每轮用 mul/add 链重建 src/weights 偏移并重载循环不变 stride（52fa2a-52fa7e 的 `ld`+`mul`/`add`），而 1x1 plain 路径下 src_off=ic（src_ic_stride=1）、weights_off=ic×weights_ic_stride，访问实为固定 stride 连续内存；编译器因 dims 是运行时值保留通用计算。依据 `patterns/loop_induction_variable_strength_reduction.md` §Why this is slow #1「每轮重复 scale + base+offset 计算」与 #4「迭代放大」（OoO 核 C920v2 上更长 dependency chain 还阻碍访存重叠）。
2. **The fix / 修复方式**：修复对象是 ref kernel plain 路径的地址归纳变量：
   - 修复前（annotate 形态）：`ld a0,208(s2); mul a0,s9,a0; ... add a0,a0,a3; ld a4,-488(s0); mul a4,a3,a4`（每轮 ~8 条 mul/add/ld）。
   - 修复后蓝图：进入 ic 循环前把 `src_loc + ic*src_ic_stride` 与 `weights_loc + ic*weights_ic_stride` 转为指针递增，预计算 end-pointer 终止（`for (p=src0, q=wei0; p<src_end; p+=src_ic_stride, q+=wei_ic_stride)`），循环体只剩解引用与指针比较。
   - 适用前提：src_ic_stride/weights_ic_stride 循环不变（plain 路径已保证）；`end = base + IC*stride` 不溢出。
   - 不可破坏的 correctness contract：`IC==0` 时零次执行；tail 行为等价；偏移不能跨出 padded dims 边界。
   - 限制/风险：该优化只作用于 ref kernel（若 primary 落地则 moot）；编译器内联展开后该变换受 lambda 捕获结构限制，需源码级归纳变量改写。
   - 修复后预期 Profile signals：52fa2a-52fa7e 区间的 mul/add 样本消失，循环体收缩为 load+转换+fmadd+指针比较。
3. **Baseline facts 回填**：hardware ISA 含 v/zfh；build ISA=gap；VLEN=128；bound=compute-bound。
4. **收益上界**：函数内局部样本份额 ≈ 20%（offset 链 ~15.7% + 耦合循环控制），条件性（primary 落地归零）。
5. **三维路由判定**：current source=compiler-generated scalar；implementation existence/reachability=不涉及新 kernel；function-level policy=无 `.S` 约束。
6. **Related PRs**（patterns/loop_induction_variable_strength_reduction.md §Related PRs）：Linux [18be4ca5cb4e](https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d)；OpenBLAS [477dd40f073c](https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df)、[d832ee50868a](https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec)。Related PRs：3 条 URL。

## Phase 5 — Verification forecast / 验证预测：std::_Function_handler<...lambda#3>::_M_invoke

- **Primary（kernel selection）**：
  - 应消失/缩小：annotate 中 `21.20 : 52fb36: blt s3,a5,52fa2a` 所在 ic loop interval（52fa2a–52fb36）样本大幅下降；该 symbol（ref lambda `_M_invoke`）整体份额从 91.71% 显著回落；test_output `| ref:any : 1 (100%) |` 不再出现，改为 `jit_1x1:zvfh` / `brgemm:rv64:zvfh` 类标识。
  - 应出现：同一 benchdnn 问题 annotate 出现向量 kernel 指令（`vle16.v`/`vfwcvt.f.f.v`/`vfwmacc*`/`vfncvt.f.f.w` 与 `vsetvli`）；实现统计行出现 rv64 向量实现名。
  - 依据：`patterns/kernel_selection_and_runtime_specialization.md` §Verification「通过符号、日志、断点、采样或实现标识确认真实工作负载进入预期实现」；数值对照覆盖 f16 NaN/±0/subnormal 与 RNE rounding；fallback（无 zvfh / 不支持组合）路径仍可达且结果正确。
- **Supporting（no-vectorization）**：随 primary 验证，不单独列。
- **Independent（ISA substitution）**：
  - 应消失/缩小：`2.37 : 52fbac: bnez a3,52fe16`、`2.46 : 52fe08: or a5,a5,a4`、`2.20 : 52fe20: slli a3,a3,0x17` 所在转换序列样本消失。
  - 应出现：annotate 出现 `fcvt.s.h`；`readelf -A libdnnl.so.3.14` 的 Tag_RISCV_arch 含 `zfh`（先补采该 baseline 再判定）。
  - 依据：`patterns/isa_extension_specific_instruction_substitution.md` §Verification「confirm instruction disassembly... contains native ... instead of shift-or sequences」；denormal/±0/NaN 边界逐类核对。
- **Independent（Loop-IV）**：
  - 应消失/缩小：`2.62 : 52fa36: mul a0,s9,a0`、`2.62 : 52fa42: mul a4,s10,a4`、`2.60 : 52fa52: add a0,a0,a3` 所在地址链样本消失；`2.65 : 52fa74: ld a4,-488(s0)` 的循环内重载消失。
  - 应出现：循环体只剩 pointer 递增 + 指针比较；`perf stat` instructions 下降而 memory bandwidth 不变（印证收益来自 loop control）。
  - 依据：`patterns/loop_induction_variable_strength_reduction.md` §Verification「指令构成对照 + 边界回归（n==0 / single / overflow boundary）」。
- 组合注意：primary 与两个 independent 不可用一次修复互相替代验证；若 primary 落地，independent 预测按「signal 消失」核验即达预期。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现（载荷：`1/1 组；std::_Function_handler<...lambda#3>::_M_invoke`） | ✅ 1/1 组；`std::_Function_handler<...ref_convolution_fwd_t::execute_forward...lambda#3>::_M_invoke` |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 表 + L0/bound gates；`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`） | ✅ 7 行；gaps=`build ISA`、`sampling IP precision`；Sampling IP precision 行已含 |
| 3 | Phase 3 输出要求满足（载荷：8 项 Class selection trace；Classes scanned: rows-operator-rvv.md, rows-codegen.md；顶层 finding=3（1 primary + 2 independent）+ 1 supporting；evidence 锚点见各 (a)；排除条数=6（isa-dispatch, no-vec primary, missing-.S, jit-quality, precision-conversions, fp-semantic-lowering 等）；推导式条数=4） | ✅ 8 项 trace；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层=3 + supporting=1 |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern=4：kernel_selection_and_runtime_specialization.md、no-vectorization.md、isa_extension_specific_instruction_substitution.md、loop_induction_variable_strength_reduction.md；对应 row=Kernel Selection / No vectorization / RISC-V ISA Extension-Specific Instruction Substitution / Loop Induction Variable Strength Reduction；引用短语首词=「优化实现不可达」「vector unit 没有处理」「Excessive instruction count」「每轮重复 scale」；`The fix` 均含 before/after、correctness、风险、Profile signals；Related PRs=14+19+33+3 条 URL） | ✅ 4 patterns；Related PRs=14/19/33/3 |
| 5 | 路径合规：8 项 trace 扫描集；多命中仲裁（primary L0 kernel-selection、supporting L1 no-vec、independent L4×2，因果消除测试已做）；A 模式按动态份额排序（primary 91.71% 全局 > conditional independents）；`th.v*` 未停扫（无 th.v*） | ✅ 模式 A；L0/L1/L4 分层 |
| 6 | Phase 5 两侧锚定：消失侧=52fb36/52fbac/52fe08/52fe20/52fa36/52fa42/52fa52/52fa74 等 Phase 3 引用行；出现侧=`kernel_selection...md §Verification`、`no-vectorization.md §Verification`、`isa_extension_specific_instruction_substitution.md §Verification`、`loop_induction_variable_strength_reduction.md §Verification` | ✅ 8 个消失锚点 + 4 个 §Verification |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、`The fix` 与验证预测 | ✅ 边界内 |

修正记录：无