Functions under analysis: [caffe::PoolingLayer<float>::Forward_cpu]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`006-caffe::PoolingLayer<float>::Forward_cpu...-annotate.txt`，libcaffe.so.1.0.0，816 行，event=cpu-clock，共 14 samples，percent: local period，含 hot loop 全部三层循环与 AVG/MAX/STOCHASTIC 三分支；源行与反汇编逐行内联）
- perf stat（可选 bound/context）：已提供（`12-caffe-benchmark-riscv-bvlc_alexnet.txt`，全 workload 计数：cycles 56,413,930,048；instructions 129,543,036,792；branches 4,984,941,186；branch_misses 7,963,448；L1_dcache_loads 53,701,921,459；L1_dcache_load_misses 150,729,109；LLC 事件 NA）
- workload/binary/DSO/source context：已提供（caffe bvlc_alexnet benchmark；libcaffe.so.1.0.0；annotate 内嵌 DWARF 源行，定位到 `src/caffe/layers/pooling_layer.cpp` 的 `PoolingLayer<Dtype>::Forward_cpu`）
- readelf -A（热点 object 的 Tag_RISCV_arch）：已提供（metadata `caffe-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`）
- hardware ISA：已提供（metadata cpuinfo + 批次 hardware_profile_snapshot：SpacemiT X100（K3 SoC），`rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`）
- vlenb：已提供（hardware_profile_snapshot：`vlen_bits: 256`，vlenb=32）
- 采样元数据（event / percent type / scope / 窗口）：部分（annotate 头声明 event=cpu-clock、percent local period、单运行窗口；函数级 workload 贡献未知——本函数仅 14 samples、rank 006，见 Phase 1）
- Sampling IP precision（precise_ip / Exact-IP）：缺失（annotate 未提供 precise_ip 信息 → `baseline_gap: sampling IP precision`）
- 样本量注记：全函数仅 14 samples，且 13/14 集中落在同一 MAX-pooling w-loop 区间（0x19df80–0x19dfaa）。样本量小带来统计噪声，结论只锚定该 loop interval，不作为单指令 latency/cost 证据（见 Phase 1 Sampling IP precision gate）。

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh...`（含 `v`、`zvbb`、`zvbc`、`zvk*`、`zvfh*`、`zfa`、`zba/zbb/zbc/zbs`）；SpacemiT X100（K3），乱序 OoO（hardware_profile_snapshot 权威；core-profiles 静态表中 K1/X60 行标 in-order，实机快照声明 OoO，以快照为准） |
| Build ISA | `Tag_RISCV_arch: rv64i2p1_..._v1p0_..._zvl128b1p0_...`（含 `v1p0`、`zve*`、`zvl128b`）；与 hardware 的 `v` 无 mismatch |
| Vector flavor | annotate 内 hot interval 零 `v*`、零 `th.v*`（全 scalar：flw/flt.s/fsw/addiw/mulw/bne）；build/hardware 均为 RVV 1.0，无 flavor mismatch |
| VLEN | vlenb=32 → VLEN=256 bits（`zvl128b` 为 build 最低声明，实机 256 优先） |
| Bound type | 全 workload perf stat：IPC = 129.5B/56.4B ≈ 2.30；branch miss 0.160%；L1 dcache load miss 0.281% → 全 workload 为 compute/latency-bound，非 memory-bound 亦非 branch-bound；无 tile-residency/尺寸拐点证据 → 不命中 cache-aware-blocking；注意该 perf stat 是全 benchmark 级，非本函数级（函数级 counter 缺失） |
| Sampling semantics | event=cpu-clock（可解释为时间的周期事件）；percent type=local period；同一运行窗口；函数级 workload 贡献未知（14 samples，rank 006/6）→ 收益上界只能表述为函数内局部样本份额，不得给 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`，函数贡献项） |
| Sampling IP precision | 未知（无 precise_ip / Exact-IP 信息）→ `baseline_gap: sampling IP precision`；单指令占比（如 19dfa8 addiw 42.86%）只锚定 loop interval，不承担单指令 latency/cost 根因 |

L0 baseline gate：hardware 有 `v` 且 build ISA 含 `v1p0` → 无 hardware/build mismatch；annotate 无 `th.v*` → 无 vector flavor mismatch。Bound-type gate：全 workload 为 compute-bound，本地 RVV 向量化 fix 的 performance-impact 不受 memory-bound 压制；但函数级 bound 证据缺失，impact confidence 仍需按采样语义与样本量下调。

## Phase 2 — Scope / 分析边界

- 承诺函数清单（1 个）：`caffe::PoolingLayer<float>::Forward_cpu`（libcaffe.so.1.0.0，0x19da72–0x19e08c）
- 函数结构：prologue（0x19da72–0x19dab8）→ pool 方法 switch（0x19dac8：MAX=0 默认；AVE=1 → 0x19dca6；STOCHASTIC=2 → NOT_IMPLEMENTED）→ MAX 分支两拷贝：
  - 拷贝 1（`use_top_mask==true`，top.size()>1，top_mask 路径）：0x19db10–0x19dc6c，内层指针递增代码形态（`addi a1,a1,4`）——0.00 samples；
  - 拷贝 2（`use_top_mask==false`，mask=max_idx_.mutable_cpu_data() 路径）：0x19dee0–0x19e004，内层每轮 mulw 重算索引——**13/14 samples 集中于此**。
- AlexNet pool1/pool2/pool5 均为 MAX、单 top blob → top.size()==1 → 运行时走拷贝 2，与热点归属一致。
- hot loop 锚点（拷贝 2 的 w-loop interval 0x19df80–0x19dfaa，ph/pw 外层 0x19df10–0x19dfba）：
  - 最高占比行：`42.86 :   19dfa8: addiw   a4,a4,1`（w 循环增量，占 6/14）
  - 其余命中：`7.14 : 19df1c: mulw a4,a4,t4`、`7.14 : 19df26: addw a3,a3,a4`、`7.14 : 19df5c: bge a2,a3,19df64`、`7.14 : 19df80: mv a4,t1`、`7.14 : 19df94: add a3,a3,s11`、`7.14 : 19df9a: flt.s a3,fa4,fa5`、`7.14 : 19dfaa: bne a4,a0,19df82`（合计 13/14 ≈ 92.9%）
- annotate 覆盖完整（含两个 MAX 拷贝、AVE 分支、epilogue 与 stack_chk_fail 路径）；无 annotate_gap。
- Sampling IP precision 不足以做单指令归因：`42.86%` 落在 loop back-edge 附近增量指令上，仅证明整个 w-loop interval 是热点，不证明该 addiw 本身昂贵。

## Phase 3 — Pattern scan / 模式扫描：caffe::PoolingLayer<float>::Forward_cpu

### Class selection trace（8 项）

| # | Class 文件 | include/exclude | 触发观察 |
|---|---|---|---|
| 1 | rows-asm.md | exclude | 当前代码是 compiler-generated C++（pooling_layer.cpp 模板实例化，annotate 含 DWARF 源行）；无 `.S` provenance、无 dispatch slot、无 assembly-default policy 证据 → 四证不成立 |
| 2 | rows-operator-rvv.md | include（第一级必选） | compiler-generated scalar loop，operator 语义明确：固定 stencil max pooling（邻域归约、逐 tap load、max 比较） |
| 3 | rows-string-memory.md | exclude | 非 copy/fill/sentinel scan/compare/checksum/back-reference 语义；caffe_set/memset 仅在 cold setup（0x19dafc、0x19dedc 调用），0.00 samples |
| 4 | rows-vectorized-tuning.md | exclude | hot interval 零 `v*`/`th.v*`，非 already-vectorized loop，无 vsetvl/LMUL/unroll 修正对象 |
| 5 | rows-codegen.md | include（补充） | 拷贝 2 的 w-loop 每轮 `lw a5,760(s1)`（width_ 重载）+ `mulw a5,a5,a1`（h*width_ 重算）+ `slli/add` index scaling，属循环级归纳变量形态 |
| 6 | rows-offload.md | exclude | 无矩阵引擎/packed-SIMD 证据；pooling 非 GEMM lowering；build ISA 无 `_xsmtvdotii` |
| 7 | rows-crypto.md | exclude | 无密码学原语 |
| 8 | rows-runtime-os.md | exclude | 用户态 C++ layer 代码，非 kernel/RTOS timer/CSR 路径 |

Classes scanned: rows-operator-rvv.md、rows-codegen.md

### Local performance pattern scan: `caffe::PoolingLayer<float>::Forward_cpu`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Spatial Convolution and Pooling Kernels（primary） | 固定 stencil（kernel_h_/kernel_w_/stride_h_/stride_w_ 调用内不变）MAX pooling 标量邻域归约，sample 主导在固定邻域地址生成（mulw/addw/slli/add）、逐 tap load（flw）、max 比较（flt.s）与 conditional store（fsw/sw）；hot w-loop interval 0x19df80–0x19dfaa，13/14 samples | High | Low | `patterns/rvv_spatial_convolution_and_pooling_kernels.md` |
| No vectorization（supporting） | 同一 hot main loop 全 scalar、zero `v*`/`th.v*`；hardware 有 `v`（RVV 1.0）、build ISA 含 `v1p0`；只解释缺少向量执行，不决定贡献载体 | — | — | `patterns/no-vectorization.md` |
| Loop Induction Variable Strength Reduction（supporting） | 同一 w-loop 每轮重算 `h*width_`（`19df8a: mulw a5,a5,a1`）、重载 `width_`（`19df82: lw a5,760(s1)`）并以 `slli+add` 构造地址（19df90/19df94），而访问为连续固定 stride，本可用指针递增 + end-pointer | — | — | `patterns/loop_induction_variable_strength_reduction.md` |

**(a) 逐字 evidence 引用（primary）**：
```
42.86 :   19dfa8: addiw   a4,a4,1          ; w++
 7.14 :   19dfaa: bne     a4,a0,19df82     ; w < wend 回边
 7.14 :   19df9a: flt.s   a3,fa4,fa5       ; top_data[pool_index] < bottom_data[index] ?
 7.14 :   19df94: add     a3,a3,s11        ; bottom_data 基址 + index*4
 0.00 :   19df82: lw      a5,760(s1)       ; width_（w-loop 内重载，loop-invariant）
 0.00 :   19df8a: mulw    a5,a5,a1         ; h * width_（w-loop 内重算，loop-invariant）
 0.00 :   19df86: flw     fa4,0(a2)        ; top_data[pool_index] 逐 tap 重读
 0.00 :   19df96: flw     fa5,0(a3)        ; bottom_data[index] 逐 tap load
 0.00 :   19df9e: beqz    a3,19dfa8        ; 条件更新
 0.00 :   19dfa0: fsw     fa5,0(a2)        ; top_data[pool_index] = max
 0.00 :   19dfa4: sw      a5,0(a6)         ; mask[pool_index] = index（整数索引，需保留）
```
以上全部属于 MAX 拷贝 2 的 w-loop interval（0x19df80–0x19dfaa）；与源码 `if (bottom_data[index] > top_data[pool_index])` 一一对应（`flt.s fa4,fa5` 即 top<bottom 时更新，严格大于语义、首个最大值获胜）。外层命中行 `7.14 : 19df1c: mulw a4,a4,t4`（ph*stride_h_-pad_h_）、`7.14 : 19df26: addw a3,a3,a4`（hend）、`7.14 : 19df5c: bge a2,a3,19df64`（wend min）为同一 ph/pw 邻域地址生成链的一部分。

**(b) 互斥邻居排除**：
- 非 resampling：tap 偏移/stride/pad 在一次调用内固定，无运行期分数坐标、index/weight 预计算或数据相关 indexed load → rows-operator-rvv 的 Resampling row 排除；
- 非 GEMM/matmul lowering：本函数无 M/N/K 累加、无 im2col、无 FMA 主导 → Floating-Point Matmul 与 matrix-engine im2col 行排除；
- 非 standalone reduction：pooling 归约是固定 stencil 邻域归约而非单输入线性序列的全局 min/max/sum → Extrema/Widening-Reduction rows 排除；
- 非 layout/color：无固定 stride 搬运、channel packing 或颜色矩阵主导 → Strided-Layout/Layout-Packing/Color rows 排除；
- 非 hand-written `.S` / 非 scalar `.S`：provenance 为 compiler-generated C++ → 整个 rows-asm.md 整组排除；
- 非 kernel-selection / dispatch：caffe 无 pooling 向量 kernel dispatch slot、无已存在未命中的专用实现 → rows-codegen 的 Kernel Selection row 排除；
- 非 register-pressure primary：prologue 压入 11 个 callee-saved（s0/s1/s2/s4/s6/s7/s8/s11/s3/s5/s9/s10 视路径）+ stack_chk_guard，但无 hot-interval spill/reload 证据，只记录为可能随数据流缩短而消失的症状，不作为顶层 finding（仲裁见下）。

**(c) 双 Confidence 推导式**：
- primary（Spatial Convolution and Pooling）：`route: compiler-generated scalar + 固定 stencil max-pooling 语义 + hardware V + build 含 v1p0 + 13/14 sample 归属该邻域归约区间 → High；impact: 样本量仅 14、采样 IP precision 缺失、函数级贡献未知 → Low`
- supporting（No vectorization）：route gate 成立（hardware V + build V + hot loop zero `v*`）；作为 supporting 不单独推导 impact（—）
- supporting（IV Strength Reduction）：route gate 成立（同一循环每轮 index scaling + limit reload + 连续固定 stride 访问，证据为 19df82/19df8a/19df90/19df94 逐字行）；因与 primary 同一 hot loop、同一机制链，降为 supporting（—）

### 多命中仲裁小段

三个候选解释同一 hot loop（MAX 拷贝 2 的 w-loop）。按 evidence-mechanism 分账：
- **primary**：RVV Spatial Convolution and Pooling Kernels。机制 = 固定 stencil 邻域归约逐元素标量执行（重复 tap load、逐元素 max/conditional store、邻域地址生成），L1 vectorization/semantic 层（arbitration L1）。证据：13/14 samples 全部落在该邻域归约区间；源码语义（pooling_layer.cpp Forward_cpu MAX 分支）与之对应。
- **supporting**：No vectorization。解释同一机制的"缺少向量执行"侧面：zero `v*` + hardware/build V → autovec gap / RVV kernel 未构建。按 rows-operator-rvv 的 No vectorization 行互斥规则（"operator semantic shape... → 各自更具体 row"），更具体的 spatial row 认领 primary，本行只作 supporting（行内语义 gate 成立）。
- **supporting**：Loop Induction Variable Strength Reduction。机制 = 索引归纳变量每轮重建地址（L4 compute/codegen micro-structure）。因果消除测试：上层 spatial fix（向量化邻域归约）会自然消除标量 mulw/slli/add 地址重建 → 不构成独立顶层 finding，作 supporting。
- 未列名/记录的症状：register save/restore（prologue 压 11 个 callee-saved）无独立 gate 证据，仅在 primary 下记录为可随数据流缩短消失的症状（arbitration：register save/restore 需自身 row gate + 直接反汇编证据才作 supporting，此处不满足）。
- 入口条件 A：顶层 finding 仅 primary 一项，evidence sample share 加总 = 13/14 ≈ 0.929（函数内局部份额）。

## Phase 4 — Root-cause blueprint / 根因蓝图：caffe::PoolingLayer<float>::Forward_cpu

### Finding 1（primary）：RVV Spatial Convolution and Pooling Kernels

对应 Phase 3 通过 gate 的 row：`rows-operator-rvv.md` 的 RVV Spatial Convolution and Pooling Kernels 行（条件 ①：scalar hot path sample 主导在固定邻域地址生成、重复 tap load、max/sum、interior/border 分支）。

1. **Root cause**：MAX 池化在 SpacemiT X100（RVV 1.0，VLEN=256）上以纯标量逐元素执行固定 stencil 邻域归约：每个输出位置对 kernel_h_×kernel_w_ 个 tap 逐次 `flw`+`flt.s`+条件 `fsw`/`sw`，每轮用 `mulw/addw/slli/add` 重建 `index=h*width_+w` 并重载 `width_`；相邻输出窗口高度重叠（3×3/stride 2），但 tap 之间零数据复用、零向量执行（hot interval 无 `v*`）。机制引用：pattern §Why this is slow——"固定 stencil 的主要根因是重复读取高度重叠的邻域、错误向量化轴、用通用 deinterleave/temporary-buffer 流程表达固定 stride、interior/border 混在主循环、deconvolution 输出重叠或 pooling 归约仍逐元素执行。主要 leverage point 是沿独立输出位置/通道向量化"；并引用 §13——"Max pooling 的初始值必须来自 reference 的精确合同，不能全局固定为 `-FLT_MAX`"（本函数正用 `caffe_set(top_count, Dtype(-FLT_MAX), top_data)` 初始化，见 0x19ded0-0x19dedc）。

2. **The fix / 修复方式**（与 pattern §13、§5、§2 及 kernel-conventions 一致，只作诊断蓝图，不实施）：
   - **向量化轴**：AlexNet pooling 为 NCHW，通道维在 Caffe 中是每通道独立连续 plane。可行的主轴是沿**输出宽度 ow**（连续输出位置）向量化 interior：对固定 3×3/stride 2 窗口，每个输出 lane 的 tap t（t∈{0,1,2}）位于 `inputY*width_ + (ow*stride_w - pad_w + t)`。逐 kernel 行（kernelY∈{0,1,2}）用 strided load 取 3 个 tap 向量：
     ```cpp
     // Before（当前标量形态，与 19df82-19dfa8 对应）
     for (h = hstart; h < hend; ++h)
       for (w = wstart; w < wend; ++w) {
         const int index = h*width_ + w;
         if (bottom_data[index] > top_data[pool_index]) {
           top_data[pool_index] = bottom_data[index];
           mask[pool_index] = index;
         }
       }
     // After（interior fast path 形态，示意；border 走独立 scalar fallback）
     // 每个 tap 一个向量：stride_w==2 → vlse32(base + tap*4, 8, vl)；
     // 对 9 个 tap 向量做 vfmax.vv 归约；NaN/tie 语义另按 reference 合同处理
     vfloat32m1_t acc = first_tap;                    // 初始值来自 reference 合同，非全局 -FLT_MAX
     acc = __riscv_vfmax_vv_f32m1(acc, tap_row1_c0, vl);
     acc = __riscv_vfmax_vv_f32m1(acc, tap_row1_c1, vl);
     acc = __riscv_vfmax_vv_f32m1(acc, tap_row1_c2, vl);
     /* ... 共 9 个 tap ... */
     __riscv_vse32_v_f32m1(top_data + pool_index0, acc, vl);
     ```
     固定 `W`/动态 `N` 与 LMUL：pooled_width 为运行期值（AlexNet 为 27/13/6），按 kernel-conventions §3 用 fixed-VL main loop + runtime-VL tail（或单 strip-mined 循环）；9 tap + accumulator + index 寄存器在 e32m1（VLEN=256 → 8 lane/vector）下约 11 个 live vector ≤ 32，m1 安全，m2 需先画 live interval 再判（kernel-conventions §2），不默认升 LMUL。
   - **mask/argmax 跟踪**：`use_top_mask==false` 时每 lane 需同步维护整数 `index`（写入 max_idx_，见 19dfa4 `sw a5,0(a6)`）。向量形态：用比较 mask 对 index 向量做 `vmerge`/`vmseq` 条件更新；tie 语义必须保持"严格大于才更新 → 首个最大值获胜"（源码 `>`，反汇编 `flt.s fa4,fa5`）。
   - **interior/border 分离**（pattern §5）：padding 窗口、left/right border 与 tail 由独立 scalar fallback 处理；interior fast path 不携带边界判断。
   - **NaN/初始值合同**（pattern §13）：不能沿用全局 `-FLT_MAX` 初值语义当合同；窗口有效元素数固定时逐 tap 取 max 可表达 reference 语义，但 `vfmax.vv` 的 NaN 传播与 Caffe 的 `>` 比较（NaN 不更新）不同，需按 reference 逐元素对照验证后再采用 vfmax，或对 NaN 特殊处理。
   - **correctness contract 不可破坏**：NCHW 布局、top/bottom offset 步进（每通道 `bottom_data += bottom[0]->offset(0,1)`）、`pooled_height_×pooled_width_` 输出几何、`use_top_mask`/`max_idx_` 选择、首胜 tie、AVG 分支的 divisor 与 rounding、NaN 语义。
   - **限制/风险**：border 与 tail 退化风险（AlexNet pooled_width 仅 27/13/6，VLEN=256 时 m1=8 lane，main loop 短）；stride 2 的 `vlse32` 是合法直接访问形态（pattern §1），但需在目标核 A/B 验证；`vfmax` NaN 语义差异需回归。
   - **预期 Profile signals**：hot interval 出现 `vsetvli`/`vle32`/`vlse32`/`vfmax.vv`/`vse32`/`vmerge`，标量 `flw`/`flt.s`/`mulw`/条件 `fsw` 的 sample share 显著下降；cycles/output 下降；函数级 annotate 的 w-loop 区间不再主导。
   - **supporting 修复对象**：(i) No vectorization → 由 spatial fix 一并覆盖（同一 loop 向量化）；(ii) IV Strength Reduction → 若保留标量路径（border/tail），将 `h*width_` 与 `width_` 提出 w-loop，w 用指针递增 + 预计算 end-pointer，同时保留整数 index 归纳变量供 mask 写入（pattern §The fix 1/2：`for (p = base, end = base + n*stride; p != end; p += stride)`，需证明 `end` 不溢出且 n==0 零次执行）。

3. **Baseline facts 回填**：hardware ISA=SpacemiT X100（K3，OoO）RVV 1.0（含 `v`/`zvbb`/`zvk*`/`zvfh`/`zfa`）；build ISA=rv64i2p1...v1p0...zvl128b（含 `v`）；VLEN=256（vlenb=32）；bound type=全 workload compute-bound（IPC≈2.30，L1 dcache miss 0.281%，branch miss 0.160%）；`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`（函数级贡献未知）。

4. **收益上界**：当前 sampled event（cpu-clock）下本函数内局部样本份额 ≈ 13/14 ≈ 0.929。函数级 workload 贡献未知（rank 006、14 samples），故不得表述为 workload 级 Amdahl 上界；样本量小，份额置信度受限。

5. **三维路由判定**：
   - current source：compiler-generated 标量 C++（pooling_layer.cpp 模板实例化；annotate 源行 + 无 `.S` provenance 确认）；
   - implementation existence/reachability：build 内无任何 RVV pooling kernel（hot interval zero `v*`），caffe 无 pooling 向量 dispatch slot → 非 kernel-selection 问题，属 autovec gap / RVV 实现缺失；
   - function-level policy：无 caffe assembly-default policy 证据 → 不进入 policy-backed missing `.S` 分支；fix 载体为普通代码/intrinsic 级向量化（pattern §13），硬件向量 ISA 准入 gate 成立（hardware `v` + build `v1p0` + VLEN 256）。

6. **Implementation-shape proof**：不适用（未进入 policy-backed missing `.S` 分支）。

7. **Related PRs**：
   - `patterns/rvv_spatial_convolution_and_pooling_kernels.md`：Related PRs：14 条 URL（oneDNN #5506、8208c6a731ed、4f915a57ce2f、563c5fcf8a、56a75b311f2e、a4c4855a9f4c、#4323、ce942914cb9d、dd1dff668fe9、#4735、#5345；MNN #4042、672c586、#4359；OpenCV 5be158a2b6ed）
   - `patterns/no-vectorization.md`（supporting）：Related PRs：15 条 URL（OpenCV #22179、#22520、#23980、#24058、#24132、#24166、#24301、#24325、#27160、#27119、#27097、#27007、#26958、#26865、b902a8e792e1、2c16f3b7d2b2、e06502a254f7、a2d784b6f53a、83104bed3209）
   - `patterns/loop_induction_variable_strength_reduction.md`（supporting）：Related PRs：3 条 URL（Linux 18be4ca5cb4e；OpenBLAS 477dd40f073c、d832ee50868a）

## Phase 5 — Verification forecast / 验证预测：caffe::PoolingLayer<float>::Forward_cpu

**Finding 1（primary，Spatial Convolution and Pooling）**：
- 应消失/缩小（锚定 Phase 3(a) 引用行）：`19dfa8: addiw a4,a4,1`（42.86%）与 w-loop interval `19df82-19dfaa` 的 `lw a5,760(s1)`/`mulw a5,a5,a1`/`slli a3,a5,0x2`/`add a3,a3,s11`/`flw fa5,0(a3)`/`flt.s a3,fa4,fa5`/`beqz a3,19dfa8`/`bne a4,a0,19df82` 的 sample share 显著下降；外层 `19df1c mulw`/`19df26 addw`/`19df5c bge` 同步下降。
- 应出现（锚定 pattern §Verification）：同函数 annotate 出现 `vsetvli`、`vle32`/`vlse32`、`vfmax.vv`/`vfmax.vf`、`vmerge`/`vmseq`（mask index）、`vse32`；scalar 指令不再主导 hot loop；cycles/output 改善（A/B benchmark 覆盖 AlexNet pooled 尺寸 27×27/13×13/6×6 与 border/tail）；正确性覆盖 kernel/stride/dilation/padding、interior 与所有 border、NaN 与首胜 tie（pattern §Verification correctness contract）。
- 样本量补强：由于本函数仅 14 samples，验证应提高采样（`perf record -F` 提高频率或多轮叠加）后再做 before/after 对比，避免噪声误判。

**supporting（No vectorization）**：不单独验证，跟随 primary：同一 loop 出现 RVV 指令即验证该 supporting 的消除。

**supporting（IV Strength Reduction）**：若保留标量 border/tail 路径，其验证锚定 pattern §Verification：确认 border/tail 循环体内 `slli`/`add`（index scaling）与 limit reload 减少、替换为指针递增与指针比较；遍历测试覆盖 n==0、single element、overflow 边界，结果与索引版本逐元素等价。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | 1/1 组；[caffe::PoolingLayer<float>::Forward_cpu] |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表 + L0 gate×2 + bound-type gate；gap 标签：`baseline_gap: sampling IP precision`、`baseline_gap: sampling metadata`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 Class selection trace；Classes scanned: rows-operator-rvv.md、rows-codegen.md；顶层 finding=1（primary）+2 supporting；evidence 锚点 `19dfa8: addiw a4,a4,1`（42.86%）、`19df9a: flt.s a3,fa4,fa5`、`19df94: add a3,a3,s11`、`19df82: lw a5,760(s1)`、`19df8a: mulw a5,a5,a1`；supporting=2 条（no-vectorization、loop_induction_variable_strength_reduction）；排除条数≥7；推导式：route High / impact Low |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern：`rvv_spatial_convolution_and_pooling_kernels.md`（§Why this is slow、§13、§5、§1、§Verification）、`no-vectorization.md`（§Why/§The fix/§Verification）、`loop_induction_variable_strength_reduction.md`（§The fix 1/2、§Verification）；命中 row：rows-operator-rvv.md Spatial 行 ①、No-vectorization 行、rows-codegen.md Loop Induction 行；The fix 含 before/after、correctness、风险、Profile signals 锚点；Related PRs：14+15+3 条 URL |
| 5 | 路径合规 | ✅ | 入口模式 A（profile-backed）；顶层 finding 按动态份额排序（唯一 primary，share 0.929）；无 th.v* 全局停扫；class 列表 rows-operator-rvv.md、rows-codegen.md |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`19dfa8: addiw a4,a4,1`、`19df82-19dfaa` interval（含 `mulw a5,a5,a1`、`flt.s a3,fa4,fa5`）；出现侧：`rvv_spatial_convolution_and_pooling_kernels.md` §Verification（vsetvli/vle32/vlse32/vfmax/vse32）、`loop_induction_variable_strength_reduction.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 |

修正记录：无