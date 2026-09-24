Functions under analysis: [caffe::im2col_cpu<float>]（1 个）→ 本输出含 1 组 Phase 3–5

# RISC-V 性能根因分析报告

- 函数：`caffe::im2col_cpu<float>`（rank 004 / 6）
- 软件：caffe（BVLC，commit 9b891540183ddc834a02b2bd81b31afae71b2153，test_branch=master）
- 测试用例：bvlc_alexnet
- 目标二进制：libcaffe.so.1.0.0（热点地址 0x228656）
- 采样：cpu-clock，49 samples，percent: local period

## Phase 0 — Evidence inventory / 证据清单

| Evidence 类别 | 状态 | 出处 |
|---|---|---|
| 单函数完整 annotate（含 hot loop body） | 已提供 | `004-void caffe：：im2col_cpu＜float＞(...)-annotate.txt`（49 samples，cpu-clock，local period，覆盖完整函数体 0x228656–0x228b74，含两个热循环与 memset 路径） |
| perf stat（bound/context） | 已提供（workload 级，非函数级） | `bvlc_alexnet/12-caffe-benchmark-riscv-bvlc_alexnet.txt`（cycles 56,413,930,048 / instructions 129,543,036,792 / IPC≈2.30 / branch_miss 0.160% / L1_dcache_load_miss 0.281%） |
| workload / binary / DSO / source context | 已提供 | annotate header 注明 DSO `libcaffe.so.1.0.0`；源码 `src/caffe/util/im2col.cpp`（im2col_cpu<float> 显式实例化，lines 18–55）与 annotate 内嵌源码行逐字匹配 |
| readelf -A（热点 object 的 Tag_RISCV_arch） | 已提供 | metadata `binaries.caffe-elf-A`：`Tag_RISCV_arch: "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_..._zvl128b1p0_zvl32b1p0_zvl64b1p0"` |
| hardware ISA | 已提供 | `hardware_profile_snapshot.cpuinfo.isa`：SpacemiT X100（mvendorid=0x710），`rv64imafdcvh_...` 含 `v`（RVV 1.0）、zba/zbb/zbc/zbs、zfa/zfh/zfhmin、zvbb/zvbc/zvkg/zvkned/zvknha/zvknhb/zvksed/zvksh/zvkt 等 |
| vlenb / VLEN | 已提供 | `hardware_profile_snapshot.vector`：RVV 1.0，vlenb=32，VLEN=256 bits |
| 采样元数据（event / percent type / scope / 窗口） | 已提供（部分） | event=`cpu-clock`；percent type=`local period`；scope=单函数 annotate；窗口=单次运行。percent 为 local period → 百分比仅代表函数内局部样本份额 |
| Sampling IP precision（precise_ip / Exact-IP） | 缺失 | 无 `perf report --header-only` / `perf evlist -v` 信息，precise_ip 未知 → `baseline_gap: sampling IP precision`；单行占比只锚定所属 basic block / loop interval |

附加证据说明：函数内样本总量 49，其中两个标量 gather 循环合计 45 样本（91.8%）。样本量偏小但热点归属清晰（两热区间样本份额压倒性），不影响结构完整性，只影响 instruction 级归因的精细度（受 IP precision gate 约束）。

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_zicbom_zicbop_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zimop_zaamo_zalrsc_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zcmop_zba_zbb_zbc_zbs_zkt_zvbb_zvbc_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_zvkb_zvkg_zvkned_zvknha_zvknhb_zvksed_zvksh_zvkt_smaia_smstateen_ssaia_sscofpmf_sstc_svinval_svnapot_svpbmt_sdtrig`（SpacemiT X100 / K3，OoO，含标准 `v` = RVV 1.0）。有 `v`，无 `th.v*` 痕迹 |
| Build ISA | 热点 object `libcaffe.so.1.0.0` 的 `Tag_RISCV_arch` 含 `v1p0` + `zve32f/zve64d/zve64f/zve64x` + `zvl128b`（min VLEN 128 假设）；无 IFUNC/multiversion 证据，函数实现与 object-level attribute 一致（全函数标量，见 Phase 3） |
| Vector flavor | annotate 内 zero `v*`、zero `th.v*`，全标量指令 → 无 flavor mismatch；build 与 hardware 均为 RVV 1.0 |
| VLEN | vlenb=32 → VLEN=256 bits（SEW=32、LMUL=1 时 VLMAX=8 floats；m2=16、m4=32、m8=64） |
| Bound type | workload 级 perf stat：IPC=129.5G/56.4G≈2.30；L1_dcache_load_miss_rate=0.281%；branch_miss_rate=0.160% → 非 cache-memory-bound、非 branch-bound；函数内热循环为指令发射/循环控制主导的纯数据搬运（标量逐元素），bound type 归类为 compute/issue-bound（im2col 数据膨胀导致的写流量不构成该函数样本内主导，见 Phase 4 风险段） |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=`local period`（非 global-period）；同一运行窗口；该函数占整个 workload 的贡献未知 → 百分比仅限函数内局部样本份额，**不得**称 workload 级 Amdahl 上界 |
| Sampling IP precision | precise_ip / Exact-IP / PMU skid 能力未知 → `baseline_gap: sampling IP precision`；热行只锚定 loop interval，不做单指令 latency 归因 |

L0 baseline gate 判定：
- hardware 有 `v`，build ISA 也含 `v1p0` → **无 hardware/build mismatch**（非 "hardware 有 v 而 build 无 v"）。
- 无 `th.v*`（hardware 为 RVV 1.0）→ 无 vector_flavor_mismatch；依赖标准 RVV 的 route **不冻结**。
- 结论：本函数在 v-enabled build + v-capable hardware 下仍整体编译为标量（autovectorization gap / 无 RVV kernel），这是首要 baseline finding。

Bound-type gate：workload 非 cache-memory-bound（L1 miss 0.281%）、非 branch-bound（miss 0.160%），热循环是标量指令流主导 → 本地 RVV 数据搬运改写是合理第一杠杆，performance-impact confidence 不被 memory-bound 下调（函数局部层面）。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`caffe::im2col_cpu<float>`（1 个）。

热区间划分（按地址分账）：

1. **Loop A（stride_w==1 路径，源 `input_col += stride_w` 编译为 `addiw a4,a4,1`）**：基本块 0x2287ae–0x2287d0，anchor 行 `34.69 : 2287ca: addiw a4,a4,1` 与 `24.49 : 2287c0: addi a3,a3,4`。区间样本 33/49 = 67.3%。对应源码行 41–49（`for (int output_col = output_w; ...)` 的 in-bounds 分支）。
2. **Loop B（stride_w!=1 路径，`addw a4,s10,a4` = input_col += stride_w）**：基本块 0x228aae–0x228ad0，anchor 行 `8.16 : 228ac0: addw a4,s10,a4` 与 `6.12 : 228ab6: addi t1,t1,4`。区间样本 12/49 = 24.5%。对应同一源码行 41–49 的 stride_w!=1 分支（bvlc_alexnet conv1：kernel 11×11、stride 4、pad 0 → 该路径）。
3. 其余样本：`2.04 : 2287ee`（memset@plt 调用，padding 整行清零）、`4.08 : 2288cc`（padding 分支 input_col 递增）、`2.04 : 2288dc`（跨路径 jump）。合计 4/49 = 8.2%，cold/辅助路径。

两热循环合计 45/49 = 91.8% 函数内局部样本份额。annotate 覆盖完整（prologue 0x228656–0x22867a 与 epilogue 0x2288ba–0x2288c6 均为 0.00%）。

Sampling IP precision：未确认 precise_ip → 单行占比不承担 instruction-latency 归因，根因收敛到 loop-interval 级机制（标量逐元素地址重建 + 窄粒度 flw/fsw + 归纳变量更新）。Loop A/B 共享同一机制，合并为一个 primary finding。

## Phase 3 — Pattern scan / 模式扫描：caffe::im2col_cpu<float>

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源是 compiler-generated scalar C++（annotate DSO `libcaffe.so.1.0.0` 全标量指令，源码 `src/caffe/util/im2col.cpp` 为普通 C++ 模板，无 `.S` provenance）；caffe 对 im2col 无 assembly-default policy，policy-backed missing `.S` 四证不成立。
2. `rows-operator-rvv.md` — **include**：compiler-generated scalar loop，算子语义明确（im2col 固定 stride/连续布局变换 gather）；候选 strided-layout、spatial-conv、gather、no-vectorization 行。
3. `rows-string-memory.md` — **include**：函数含 padding 整行清零的 memset 路径（0x2287ee）与 stride_w==1 时的连续复制形态；需排除 copy/fill、wide-scalar 等行。
4. `rows-vectorized-tuning.md` — **exclude**：annotate 全程 zero `v*`，无已向量化 main loop，修正对象不是 RVV 配置/展开/policy。
5. `rows-codegen.md` — **include**：compiler-generated 指令形态——每轮 index scaling（`addw`+`slli`+`add`）、归纳变量更新、大 prologue/epilogue（240B 栈帧、大量 s-register save/restore）；候选 loop-induction-strength-reduction。
6. `rows-offload.md` — **exclude**：hardware snapshot 仅 RVV 1.0，annotate 无矩阵引擎/packed-SIMD 指令，无引擎卸载证据；`matrix_engine_im2col_convolution_lowering` 需"目标具备矩阵引擎"证据，不成立。
7. `rows-crypto.md` — **exclude**：非密码学原语热点。
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel/timer/CSR 热点。

`Classes scanned:` rows-operator-rvv.md、rows-string-memory.md、rows-codegen.md

### Local performance pattern scan: `caffe::im2col_cpu<float>`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Strided Memory Access for Layout Transforms（**primary**） | 两个热 gather 循环（Loop A 33/49=67.3%、Loop B 12/49=24.5%，合计 91.8%）全标量：每轮 `addw/slli/add` 重建地址 + 单元素 `flw`/`fsw` + 归纳变量更新 + 边界分支 | High | High（函数内局部份额，无 workload 级 Amdahl） | `patterns/rvv_strided_layout_transform_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`，hardware 与 build ISA 均含 `v`；只解释缺少向量执行，不决定贡献载体 | — | — | `patterns/no-vectorization.md` |
| Loop Induction Variable Strength Reduction（supporting） | 同一热循环每轮 `addw a5,a4,s2; slli a5,a5,0x2; add a5,a5,s4` index scaling + 每轮 limit/end 比较（`bne a4,s3`/`bne t1,a3`）；因果消除测试：向量化改写自然消除该 signal | — | — | `patterns/loop_induction_variable_strength_reduction.md` |

#### Primary finding — RVV Strided Memory Access for Layout Transforms（三件套）

**(a) 逐字 evidence 引用**（Loop A，基本块 0x2287b8–0x2287d0，对应源码行 41–49 in-bounds 分支）：

```
2.04 :   2287b8: addw    a5,a4,s2     ; 每轮地址重建 base+index
0.00 :   2287bc: slli    a5,a5,0x2    ; index*4
0.00 :   2287be: add     a5,a5,s4     ; + data_im 基址
24.49 :   2287c0: addi    a3,a3,4      ; data_col 指针递增（目标侧连续）
4.08 :   2287c2: bgeu    a4,s7,...     ; in-bounds 检查（is_a_ge_zero_and_a_lt_b）
2.04 :   2287c6: flw     fa5,0(a5)     ; 单元素浮点加载
34.69 :   2287ca: addiw   a4,a4,1       ; input_col += 1（stride_w==1）
0.00 :   2287cc: fsw     fa5,-4(a3)     ; 单元素浮点存储
0.00 :   2287d0: bne     a4,s3,...      ; 循环控制
```

（Loop B，基本块 0x228aae–0x228ad0，stride_w!=1 分支，conv1 stride 4 实际走此路径）：

```
2.04 :   228aae: addw    a5,a4,s5     ; 地址重建
0.00 :   228ab2: slli    a5,a5,0x2
0.00 :   228ab4: add     a5,a5,s9
6.12 :   228ab6: addi    t1,t1,4      ; data_col++
0.00 :   228ab8: bgeu    a4,s8,...     ; in-bounds 检查
0.00 :   228abc: flw     fa5,0(a5)
8.16 :   228ac0: addw    a4,s10,a4     ; input_col += stride_w
4.08 :   228ac4: fsw     fa5,-4(t1)
2.04 :   228ac8: bne     t1,a3,...
```

区间样本归属：Loop A=33（67.3%），Loop B=12（24.5%），合计 45/49（91.8%）。因 `baseline_gap: sampling IP precision`，以上单行占比只锚定 loop interval，不归因单指令 latency。

**(b) 互斥邻居排除**：

- **spatial-convolution row**（行内互斥"卷积固定邻域 → spatial-convolution row"）：判别性观察——两热循环指令流为 `flw`/`fsw` + `addw`/`slli`/`add`/`bgeu`/`addiw`/`addi`，**zero FP 算术**（无 `fmadd`/`fadd`/`fmax`/`fmin`/`fsum`），无 stencil 算术形态；im2col 是卷积 GEMM 前的纯数据布局变换（image → column），不是卷积/池化算术算子。
- **elementwise row**（行内互斥"纯连续 arithmetic → elementwise row"）：无逐 lane 算术；输出位置到输入位置是经 stride/dilation 的仿射映射（布局变换），Loop B 中相邻输出对应相隔 stride_w 的输入（stride 4），非同 lane 单位步长算术。
- **indexed gather row**（行内互斥"任意 indexed gather/scatter → gather row"）：地址是循环变量的**仿射函数**（`base + i*stride_w`，base 每 (kernel_row, kernel_col, output_row) 固定），非数据相关索引/LUT。
- **RVV copy/fill row（rows-string-memory）**：行内每元素有 in-bounds 分支（0x2287c2 `bgeu`）与 padding 零填（0x2288c8 `sw zero`），不是纯 counted copy/fill；整行越界清零路径已由 glibc `memset@plt`（0x2287ee，2.04%）承担，无需本蓝图处理。
- 附加排除（完整性）：kernel-selection row（函数内无 dispatch/多实现信号，仓库无 im2col RVV 实现）、cache-aware-blocking row（无 tile-residency/尺寸拐点证据，annotate 与 perf stat 均无）。

**(c) 双 Confidence 推导式**：

- `route: 当前代码来源=compiler-generated scalar C++（DSO+源码双 provenance）、语义合同=固定 stride 布局变换 gather（地址计算+标量访存主导样本）、行内互斥均有判别性观察 → High`
- `impact: 函数内热循环样本份额 45/49=91.8%（采样语义 local period → 仅函数内局部份额，四项 Amdahl 条件未全满足）、VLEN=256 已知、bound type=非 L1 cache-memory-bound（IPC 2.30/L1 miss 0.281%/branch miss 0.160%）、build/dispatch baseline 已知（build 含 v1p0 而函数零 v*）→ High（函数内局部）；竞争瓶颈：无（两热区间均为纯 gather 循环）`

#### Supporting evidence（两行，不单独分级、不计入顶层 finding 数）

- **No vectorization**（`patterns/no-vectorization.md`）— (a) 同一 Loop A/B 逐字行内 zero `v*`/`th.v*`（引 0x2287c6 `flw fa5,0(a5)` / 0x2287cc `fsw fa5,-4(a3)` 区间无任何 `v*`），hardware ISA 含 `v`（Phase 1），build `Tag_RISCV_arch` 含 `v1p0`；supporting because: 与 primary 同一机制，只解释"v-enabled 构建下该函数仍零向量指令"（autovectorization gap / RVV kernel 未构建），不决定贡献载体（载体由 strided-layout 行决定）。
- **Loop Induction Variable Strength Reduction**（`patterns/loop_induction_variable_strength_reduction.md`）— (a) 每轮 `addw a5,a4,s2; slli a5,a5,0x2; add a5,a5,s4`（引上述 Loop A 行）对连续/固定 stride 访问做 index scaling；supporting because: 与 primary 同一机制，因果消除测试通过——RVV 向量化改写（primary 的 The fix）自然消除逐元素 index scaling，故降为 supporting，不作为独立顶层 finding。

#### 多命中仲裁小段

- 因果关系：三命中解释**同一热循环同一机制**（标量逐元素 gather）。primary = strided-layout（L1 vectorization/semantic 层，按 `arbitration.md` L1 层归属 rows-operator-rvv 全体）；no-vectorization 与 strength-reduction 均因"同机制 + 上层改写自然消除下层 signal"降为 supporting。无 independent/companion 命中（无 companion 白名单成员）。
- 收益上界排序（入口条件 A）：按 evidence sample share 加总 → primary 45/49 = 91.8%（函数内局部，采样语义 local period）；supporting 不单独排序。
- 归因说明：Loop A 与 Loop B 是同一函数的两个 stride 分支（stride_w==1 vs >1），机制同源、共享修复对象，合并为一个 primary；cold 路径（memset 2.04%、padding 4.08%、jump 2.04%）不参与命中归属。

## Phase 4 — Root-cause blueprint / 根因蓝图：caffe::im2col_cpu<float>

依据 Phase 3 通过 gate 的 row：`RVV Strided Memory Access for Layout Transforms`（rows-operator-rvv.md 行内判据），supporting：`no-vectorization`、`loop_induction_variable_strength_reduction`。

1. **Root cause**：
   `caffe::im2col_cpu<float>` 把输入图像按 `(channel, kernel_row, kernel_col)` 展开为 GEMM 列缓冲（`src/caffe/util/im2col.cpp` lines 30–54）。最内层循环对每个 output_col 以 `input_col += stride_w` 的固定 stride 执行 gather：`*(data_col++) = data_im[input_row*width + input_col]`。依据 `patterns/rvv_strided_layout_transform_kernels.md` §Why this is slow：
   - **Scalar address-generation overhead / 标量地址生成开销**（§1）："标量循环需要为每个元素重复计算基址、索引和 stride 偏移，并执行边界判断和分支。RVV `vlse*`、`vsse*` 可以在一条向量访存指令中描述多个固定间隔地址，从而减少显式地址计算和循环控制指令"——本函数每元素执行 `addw`+`slli`+`add` 三条地址重建指令（0x2287b8–0x2287be）。
   - **Narrow load/store instruction stream / 窄粒度访存指令流**（§2）："逐元素实现会产生大量标量 load/store。使用动态 `vl` 的 RVV 主循环可以一次描述多个元素的连续或跨步访问，减少前端取指、译码和指令发射压力"——每元素一条 `flw`（0x2287c6）+ 一条 `fsw`（0x2287cc）。
   - 叠加每元素 in-bounds 分支（0x2287c2 `bgeu`）、两个归纳变量更新（0x2287ca `addiw`、0x2287c0 `addi`）与循环比较（0x2287d0 `bne`），纯数据搬运的每元素指令数约 8 条；Loop B（stride_w>1）形态相同（0x228aae–0x228ac8）。
   - 关键条件：build ISA 含 `v1p0`、硬件 RVV 1.0 VLEN=256，但该函数零 `v*`（supporting no-vectorization）——v-enabled 构建下纯搬运循环未向量化。perf stat 显示 workload IPC≈2.30、L1 miss 0.281%，非 cache-memory-bound，热循环成本来自标量指令流本身（issue/loop-control 主导）。

2. **The fix / 修复方式**（依据 pattern §The fix 第 4 节"Strided load and contiguous store"、第 1 节 RVV 访存组合表与第 10 节 unit-stride/non-unit-stride 分派）：
   用 RVV intrinsic 重写 im2col 最内层 gather（SEW=e32），对每个 `(kernel_row, kernel_col)` 的 output row 一次性仿射裁剪出 in-bounds 区间，三段处理：

```cpp
// Before（scalar，每元素约 8 条指令，引 annotate 0x2287b8–0x2287cc）：
int input_col = -pad_w + kernel_col * dilation_w;
for (int output_col = output_w; output_col; output_col--) {
  if (is_a_ge_zero_and_a_lt_b(input_col, width)) {
    *(data_col++) = data_im[input_row * width + input_col];
  } else {
    *(data_col++) = 0;
  }
  input_col += stride_w;
}

// After（RVV 代表性形态：每 row 一次区间裁剪，主段向量化搬运）：
// base = -pad_w + kernel_col*dilation_w;
// lclip = max(0, ceil_div(-base, stride_w));  rclip = min(output_w, ceil_div(width - base, stride_w));
// data_col + output_row 起点由外层 kernel_row/stride_h 给出
__riscv_vse32_v_f32m1(out, __riscv_vmv_v_x_f32m1(0.0f, vl_head), vl_head);   // [0, lclip) 零填
const ptrdiff_t bs = (ptrdiff_t)stride_w * sizeof(float);
for (ptrdiff_t i = lclip; i < rclip; i += vl) {
  vl = __riscv_vsetvl_e32m1((rclip - i));
  if (stride_w == 1) {                       // unit-stride 快速路径：vle32.v + vse32.v
    vfloat32m1_t v = __riscv_vle32_v_f32m1(data_im + input_row*width + base + i, vl);
    __riscv_vse32_v_f32m1(out + i, v, vl);
  } else {                                   // 固定 stride：vlse32.v + vse32.v（byte stride = stride_w*4）
    vfloat32m1_t v = __riscv_vlse32_v_f32m1(data_im + input_row*width + base + i*stride_w, bs, vl);
    __riscv_vse32_v_f32m1(out + i, v, vl);
  }
}
__riscv_vse32_v_f32m1(out + rclip, __riscv_vmv_v_x_f32m1(0.0f, vl_tail), vl_tail); // [rclip, output_w) 零填
```

   - **适用前提**：stride_w/dilation_w 在单次调用内固定（im2col 调用合同保证）；输入输出无重叠（col 缓冲新分配）；元素为 32-bit float（SEW=e32）。
   - **正确性 contract**（不可破坏）：输出布局必须与标量版本逐位一致——同一 `(channel, kernel_row, kernel_col, output_row, output_col)` 映射到 `data_im[(kr*dilation_h - pad_h + or*stride_h)*width + (kc*dilation_w - pad_w + oc*stride_w)]` 或 0；纯搬运建议用整数向量类型（`vuint32m1_t`）保证 bit-preserving（NaN payload / signed zero 不变）；零填为精确 `+0.0f`；整行 `input_row` 越界时保留现有 `memset` 整行清零路径（已由 glibc 向量化）；非 RVV 构建保留 scalar fallback（`__riscv_v_intrinsic` / `__riscv_v` gate）。
   - **LMUL / tail**：动态 `N = rclip - lclip`（每调用已知），按 `kernel-conventions.md` §2/§3 选型——单循环 strip-mine（control-flow-heavy 场景允许每轮 `vsetvl`）；LMUL 从合法候选 `m1/m2/m4` 中按 live-register 预算（peak live 1–2 组 → m2/m4 架构合法）结合生成代码与目标平台 A/B benchmark 决定，不硬编码 m8；VLEN-agnostic（不假定固定 lane 数）。
   - **限制 / 风险**：stride_w>1 时 `vlse32` 的 cache-line 利用率仍低于连续访问（每 64B line 取 stride_w 个元素），对 conv1（stride 4）应按 pattern §8 与分块/连续化路径对比；im2col 本身的数据膨胀（conv1：3×55×55×121≈1.1M floats 写出）是算法层代价，本蓝图不消除，完整消除需 implicit-GEMM/直接卷积下沉（属本函数外的算法改动，不进入本蓝图）。
   - **预期 Profile signals**：annotate 出现 `vsetvli`/`vsetvl`、`vle32.v`、`vlse32.v`、`vse32.v`；标量热行 0x2287b8–0x2287cc、0x228aae–0x228ac8 样本消失/骤降；函数内每元素指令数约 8→约 2（一条向量 load + 一条向量 store 摊薄），branches 大幅下降。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcvh_...`（SpacemiT X100/K3，RVV 1.0，OoO）；build ISA = `rv64i...v1p0...zvl128b`（含 `v`，min zvl128b）；VLEN = 256（vlenb=32）；bound type = 指令发射/循环控制主导（workload IPC 2.30、L1 miss 0.281%、branch miss 0.160%）；采样语义 = cpu-clock/local period（函数内局部份额）；`baseline_gap: sampling IP precision`。

4. **收益上界**：两标量 gather 循环共 45/49 = **91.8%**（当前 sampled event（cpu-clock）下的**函数内局部样本份额**）。采样语义四条未全满足（percent type=local period；函数对 workload 贡献未知）→ 不得称 workload 级 Amdahl 上界。函数局部收益方向明确：非 cache-memory-bound 瓶颈下，每元素指令数约 8→约 2，前端/发射压力显著下降。

5. **三维路由判定**：
   - **current source**：compiler-generated scalar C++（`im2col.cpp` 模板实例化，`libcaffe.so.1.0.0`，annotate 全标量指令）。
   - **implementation existence / reachability**：仓库中无 im2col RVV kernel、函数内无 dispatch/多实现信号 → kernel-selection row 不适用；修复需新建 RVV 路径并以 build-ISA gate（`__riscv_v` / `-march` 含 `v`）或运行时分派接入，且保留 scalar fallback。
   - **function-level policy**：caffe 对 im2col 无 assembly-default policy → policy-backed missing `.S` 四证不成立，**不进入** missing-`.S` 分支；intrinsic（`__riscv_v*`）路径与现有模板/单编译单元结构兼容，作为建议载体。
   - 注：matrix-engine 卸载（`matrix_engine_im2col_convolution_lowering.md`）因硬件快照无矩阵引擎证据而排除（Phase 3 class 6）。

6. **Implementation-shape proof**：仅 policy-backed missing `.S` 分支要求；本蓝图未进入该分支（三维路由判定见上）→ N/A。

7. **Related PRs 小节**：
   - `patterns/rvv_strided_layout_transform_kernels.md` → **Related PRs：8 条 URL**（MNN `https://github.com/alibaba/MNN/pull/4023`、`https://github.com/alibaba/MNN/pull/4026`、`https://github.com/alibaba/MNN/commit/aaf5b231cf9bc44361ad2c129100bdbcc444df05`、`https://github.com/alibaba/MNN/commit/89f47a28769596c3e2240c2cf53c509a134d3ce5`；OpenMathLib/OpenBLAS `https://github.com/OpenMathLib/OpenBLAS/commit/98a8230dee44d98d143f6dff1d944d38bba115f5`、`https://github.com/OpenMathLib/OpenBLAS/commit/aa967ef6ba7cfce9710eee0857686915c0db1c86`、`https://github.com/OpenMathLib/OpenBLAS/commit/bd45b82ed012279b2c59034311f53d45fc4b0e06`；vLLM `https://github.com/vllm-project/vllm/pull/40119`）
   - `patterns/no-vectorization.md`（supporting）→ **Related PRs：19 条 URL**（OpenCV `https://github.com/opencv/opencv/pull/22179`、`.../22520`、`.../23980`、`.../24058`、`.../24132`、`.../24166`、`.../24301`、`.../24325`、`.../27160`、`.../27119`、`.../27097`、`.../27007`、`.../26958`、`.../26865`；commits `https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d`、`.../2c16f3b7d2b28f6cac444046b8f95b40d9266a6a`、`.../e06502a254f79f9d3184de2803d087c6914b7706`、`.../a2d784b6f53aa1fdfde21ab8e3787a93b59af24f`、`.../83104bed32093ff0c5c935e8920c72b6b74ae07a`）
   - `patterns/loop_induction_variable_strength_reduction.md`（supporting）→ **Related PRs：3 条 URL**（Linux kernel `https://github.com/torvalds/linux/commit/18be4ca5cb4e5a86833de97d331f5bc14a6c5a6d`；OpenMathLib/OpenBLAS `https://github.com/OpenMathLib/OpenBLAS/commit/477dd40f073c371d175d48e8b264ea40449515df`、`https://github.com/OpenMathLib/OpenBLAS/commit/d832ee50868a48bb8a16d4343d428c11d80065ec`）

## Phase 5 — Verification forecast / 验证预测：caffe::im2col_cpu<float>

入口条件 A：按收益上界顺序逐项验证（仅 primary 顶层；supporting 跟随 primary 修复对象与验证，不单独验证）。

**Primary — RVV strided-layout 向量化改写**：

- **应消失 / 缩小**（锚定 Phase 3(a) 引用行）：
  - Loop A：`0x2287c6: flw fa5,0(a5)`、`0x2287cc: fsw fa5,-4(a3)`、`0x2287b8: addw a5,a4,s2`、`0x2287c2: bgeu a4,s7`、`0x2287ca: addiw a4,a4,1`、`0x2287c0: addi a3,a3,4` 的样本份额显著下降/消失；
  - Loop B：`0x228aae: addw a5,a4,s5`、`0x228abc: flw fa5,0(a5)`、`0x228ac4: fsw fa5,-4(t1)`、`0x228ac0: addw a4,s10,a4`、`0x228ab6: addi t1,t1,4` 的样本份额显著下降/消失；
  - 函数内每元素指令数与 branches 计数下降（标量 gather 由向量 load/store 摊薄）。
- **应出现**（锚定 `patterns/rvv_strided_layout_transform_kernels.md` §Verification）：同一函数的 annotate 出现 `vsetvli`/`vsetvl`（或 `vsetivli`）、连续向量 load/store（`vle32.v`/`vse32.v`）以及 `vlse32.v`/`vsse32.v` 等跨步指令；stride_w==1 的 row 走 unit-stride 连续路径、stride_w>1 走 strided 路径；标量指令不再主导热循环。
- **正确性**：与标量 reference 逐位对比（bit-preserving，含 NaN payload/signed zero）；覆盖 bvlc_alexnet 实际参数（conv1 kernel 11×11/stride 4/pad 0 → Loop B；conv2–conv5 kernel 5×5/3×3/stride 1/pad 2 → Loop A）；覆盖 output_w=0、短行、main-loop 整倍数与全部 tail length；零填精确 `+0.0f`；整行越界 memset 路径行为不变。
- **性能**：同硬件同输入同优化级别对比 cycles/instructions/有效带宽；VLEN-agnostic（不同 VLEN 环境/配置验证）；LMUL 候选（m1/m2/m4）A/B benchmark；stride 大小对比（直接 `vlse32` vs 分块/连续化，尤其 conv1 的 stride 4）。
- **可达性**：确认 RVV 构建启用（`Tag_RISCV_arch` 含 `v` 的目标文件）、实际调用进入 RVV 路径而非 scalar fallback（运行时 dispatch 或编译期 gate）。

**Supporting（no-vectorization、loop-induction-strength-reduction）**：跟随 primary——向量化后 annotate 出现 `v*` 指令（no-vectorization 的 §Verification："同一函数的 annotate 现在包含 RVV instructions"）；每轮 index scaling（`slli`+`add`）消失（strength-reduction 的 §Verification："循环体中的 slli/add（index scaling）与 limit reload 减少"）；若保留标量 fallback，验证其循环体已改指针递增 + end-pointer 比较。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | 1/1 组；`[caffe::im2col_cpu<float>]`；Phase 3/4/5 标题各出现一次 |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表（Hardware ISA/Build ISA/Vector flavor/VLEN/Bound type/Sampling semantics/Sampling IP precision）；gap 标签：`baseline_gap: sampling IP precision`；两个 L0 gate 判定 + bound-type gate 结论齐备 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`（include: rows-operator-rvv/rows-string-memory/rows-codegen；exclude: rows-asm/rows-vectorized-tuning/rows-offload/rows-crypto/rows-runtime-os）；`Classes scanned:` rows-operator-rvv.md、rows-string-memory.md、rows-codegen.md；顶层 finding 数=1（primary strided-layout），supporting=2（no-vectorization、loop-induction-strength-reduction）；evidence 锚点=`34.69 : 2287ca: addiw a4,a4,1`、`24.49 : 2287c0: addi a3,a3,4`、`2.04 : 2287c6: flw fa5,0(a5)`、`8.16 : 228ac0: addw a4,s10,a4` 等；排除条数=4（spatial-conv/elementwise/gather/copy-fill）+2 附加；推导式条数=1（route High / impact High） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件：`patterns/rvv_strided_layout_transform_kernels.md`（row=`RVV Strided Memory Access for Layout Transforms`；引用短语首词=§Why this is slow "Scalar address-generation overhead"、"Narrow load/store instruction stream"）、`patterns/no-vectorization.md`（supporting）、`patterns/loop_induction_variable_strength_reduction.md`（supporting）；`The fix` 含 before/after 伪代码、适用前提、correctness contract、限制/风险、预期 Profile signals；missing `.S` 分支未进入（Implementation-shape proof N/A）；Related PRs：strided-layout 8 条 URL、no-vectorization 19 条 URL、loop-induction 3 条 URL |
| 5 | 路径合规：8 项 trace 可解释扫描集；零/多命中、关系与 evidence-mechanism layer 合规；blueprint leaf 均来自通过 gate 的 row；入口模式 A 按动态份额排序；`th.v*` 未全局停扫 | ✅ | 模式=A（profile-backed）；路径=compiler-generated scalar → rows-operator-rvv（primary strided-layout L1）+ supporting no-vectorization(L1)/loop-induction(L4, causal-elimination→supporting)；class 列表=rows-operator-rvv/rows-string-memory/rows-codegen；排序=primary 45/49=91.8%（函数内局部） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧=`0x2287c6: flw fa5,0(a5)`、`0x2287cc: fsw fa5,-4(a3)`、`0x2287b8: addw a5,a4,s2`、`0x2287ca: addiw a4,a4,1`、`0x228ac0: addw a4,s10,a4`、`0x228ab6: addi t1,t1,4` 等（对 Phase 3(a) 引用行）；出现侧=`patterns/rvv_strided_layout_transform_kernels.md` §Verification（"应看到 vsetvli 或 vsetivli、连续向量 load/store，以及 vlse16.v、vlse32.v..."）、`patterns/no-vectorization.md` §Verification、`patterns/loop_induction_variable_strength_reduction.md` §Verification |
| 7 | 契约边界合规：无实施询问、代码修改、补丁生成或契约外实施分支；无向用户追问；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 和验证预测 | ✅ | 无实施询问/无代码修改/无补丁；无 object-clarification 之外的提问；输出边界=诊断 + The fix + 验证预测 |

修正记录：无