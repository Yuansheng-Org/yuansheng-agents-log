Functions under analysis: [`void mxnet::op::im2col<float>(mshadow::Stream<mshadow::cpu>*, float const*, mxnet::TShape const&, mxnet::TShape const&, mxnet::TShape const&, mxnet::TShape const&, mxnet::TShape const&, mxnet::TShape const&, float*) [clone .isra.0]`]（1 个）→ 本输出含 1 组 Phase 3–5

---

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`002-...im2col...-annotate.txt`，event=`cpu-clock`，2245 samples，`percent: local period`；含 hot loop body 12af550–12af586）
- perf stat（可选 bound/context）：已提供（`11-mxnet-opperf-benchmark-riscv-category-convolution.txt`，IPC=0.639，L1_dcache_load_miss_rate=0.628%，LLC_load_miss_rate=14.894%）
- workload/binary/DSO/source context：已提供（libmxnet.so，`im2col<float>` 为 compiler-generated C++ 模板 `src/operator/nn/im2col.h` 的 2D `im2col_cpu` 内联展开；本地 checkout 源码与反汇编结构吻合）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `libmxnet.so-elf-A`，含 `v1p0`/`zve64d`/`zvl128b`）
- hardware ISA（`/proc/cpuinfo` / hwprobe）：已提供（metadata cpuinfo，`rv64imafdcv_...zve64d_zvfh...`，T-Head C920v2，OoO）
- `vlenb`：已提供（metadata `vlen_bits: 128`，`vlenb: 16`）
- 采样元数据（event / percent type / scope / 窗口）：部分缺失（event=`cpu-clock`；percent type=`local period` 非 global-period；同一运行窗口；函数级 workload 贡献未知 → 详见 Phase 1）
- Sampling IP precision：缺失（未见 `precise_ip`/Exact-IP 信息 → 详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`（C920v2，含标准 `v`，RVV 1.0） |
| Build ISA | 已提供：libmxnet.so `Tag_RISCV_arch` = `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`（**build 含 `v`**） |
| Vector flavor | annotate 内该函数 zero `v*` 也 zero `th.v*`（全 scalar）；hardware 与 build 均为 RVV 1.0（`v`），无 flavor mismatch |
| VLEN | 已提供：`vlenb=16` → VLEN=128 bits；SEW=32、LMUL=1 时 4 lanes/vector op |
| Bound type | 部分判定：IPC=0.639（OoO 核偏低）、L1_dcache_load_miss_rate=0.628%（数据高度 L1 驻留）、LLC_load_miss_rate=14.894%；hot loop 为标量指令 issue/latency-bound，非 memory-bandwidth-bound。注：`cache_miss_rate=100%` 为 counter 别名伪影（`cache_references≈cache_misses`），不作证据 |
| Sampling semantics | `baseline_gap: sampling metadata`：event=`cpu-clock`（近似时间事件），percent type=`local period`（非 global-period），同一运行窗口成立，函数 workload 贡献未知 → 只能表述函数内局部样本份额，**不得**称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`：未见 `precise_ip`/Exact-IP 信息 → 单指令占比只锚定 basic block / loop interval，不做 instruction-latency 归因 |

L0 baseline gate 判定：
- hardware 有 `v` 且 build 有 `v` → **无 hardware/build mismatch**，无 `vector_flavor_mismatch`；依赖 vector flavor 的 route 不冻结。
- 无 `th.v*` 出现。
- Bound-type gate：workload 非 memory-bandwidth-bound（L1 miss 0.628%），compute/issue-bound 特征成立；不因此降低本地 vectorization fix 的 impact 判断，但 sampling 语义与 IP precision 缺口封顶收益表述。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个函数：`mxnet::op::im2col<float>` 的 `.isra.0` clone）。

- hot loop / loop interval：内层 `output_col` 循环（源 `im2col.h` L124–131），地址区间 **12af550–12af586**（含 bounds-check 快速路径分支 `12af550–12af586`；12af59c–12af6b8 为 row-out-of-bounds / tail 路径，样本 0.00–0.04%）。
- 最高行原文：`32.92 :   12af57c:        ld      a2,-136(s0)`（hot interval 内每轮重复的 loop-invariant 栈 reload）。
- hot interval 样本合计：20.00+2.49+0.71+0.89+1.20+4.86+16.26+32.92+7.31+10.29+1.92 = **98.85%**（2245 samples 中约 2219）。
- Sampling IP precision 不足（`baseline_gap: sampling IP precision`）→ 全部归因收敛到 **interval-level mechanism**，不把单行 32.92% 读作单条指令的 cycle cost。

## Phase 3 — Pattern scan / 模式扫描：mxnet::op::im2col<float>

### Class selection trace

| # | Class | include/exclude | 触发观察 |
|---|---|---|---|
| 1 | `rows-asm.md` | exclude | 当前代码来源是 compiler-generated C++ 模板（im2col.h L88–138），非手写 `.S`；mxnet 无 im2col 的 assembly-default policy / dispatch slot，policy/existence 四证不成立 |
| 2 | `rows-operator-rvv.md` | **include**（必选） | compiler-generated scalar loop，语义为卷积 im2col（固定 stencil 邻域抽取 + padding 边界 + data_col 连续写）；逐行评估 spatial-convolution / gather / strided-layout / layout-packing / no-vectorization 等行 |
| 3 | `rows-string-memory.md` | exclude | data_col 写虽连续，但读侧是 bounds-check 的固定 stride 邻域 gather（卷积 patch 抽取），非 copy/fill/sentinel/compare/checksum 语义 |
| 4 | `rows-vectorized-tuning.md` | exclude | 该行要求 annotate 已有 `v*`；本函数 hot loop zero `v*` |
| 5 | `rows-codegen.md` | **include** | hot interval 内出现每轮重复的 `ld a2,-136(s0)`（32.92%）栈 reload（loop-invariant stride 被 spill）——register-pressure/save-restore 信号；另评估 addressing-mode / induction / control-flow 行 |
| 6 | `rows-offload.md` | exclude | 目标 SG2044/C920v2 无矩阵引擎（ISA 字符串无 `x*` 矩阵扩展；core-profiles.md 仅列 RVV 1.0）；im2col 为通用 CPU lowering，`matrix_engine_im2col_convolution_lowering` 准入 gate 不成立 |
| 7 | `rows-crypto.md` | exclude | 非密码学原语 |
| 8 | `rows-runtime-os.md` | exclude | 用户态库函数，无 timer/CSR/特权域访问 |

Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: mxnet::op::im2col<float>

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Spatial Convolution and Pooling Kernels（primary） | 固定 stencil 邻域地址生成 + interior/border 分支 + overlap-sensitive 连续 store 主导 hot interval（98.85%）；zero `v*` | High | Medium | `patterns/rvv_spatial_convolution_and_pooling_kernels.md` |
| No vectorization (autovec gap / RVV kernel not built)（supporting） | 同一 hot main loop 全 scalar、zero `v*`/`th.v*`，hardware 与 build 均暴露 `v` | — | — | `patterns/no-vectorization.md` |
| Register Pressure and Save/Restore（supporting） | 同一 hot interval 内 `ld a2,-136(s0)`（32.92%）每轮 reload loop-invariant stride；自身 row gate + 直接反汇编证据成立 | — | — | `patterns/register_pressure_and_save_restore.md` |

#### Finding 1（primary）— RVV Spatial Convolution and Pooling Kernels

**(a) 逐字 evidence 引用**（hot interval = 内层 output_col 循环，源 `im2col.h` L124–131）：
```
20.00 :   12af566:        addw    a2,a0,t0
 2.49 :   12af56a:        slli    a2,a2,0x2
 0.71 :   12af56c:        fmv.w.x fa5,zero
 0.89 :   12af570:        add     a2,a2,a6
 1.20 :   12af572:        addi    a4,a4,4
 4.86 :   12af574:        bgeu    a0,t6,12af57c
16.26 :   12af578:        flw     fa5,0(a2)
32.92 :   12af57c:        ld      a2,-136(s0)
 7.31 :   12af580:        fsw     fa5,-4(a4)
10.29 :   12af584:        addw    a0,a0,a2
 1.92 :   12af586:        bne     a4,a1,12af566
```
所属 interval：12af550–12af586（内层 output_col 循环，interval 合计 98.85%）。`bgeu a0,t6,12af57c` 即源 `is_a_ge_zero_and_a_lt_b(input_col, width)`（im2col.h L78–80 的 unsigned-cast 技巧）编译产物；`fmv.w.x fa5,zero` 是 padding 默认零（L128）；`flw fa5,0(a2)` 是 `data_im[input_row*width+input_col]`（L126）；`fsw fa5,-4(a4)` 是连续 `*(data_col++)`（L126/L128）。Sampling IP precision 不足，单行只锚定 interval 机制，不承担 instruction-latency 归因。

**(b) 互斥邻居排除**：
- RVV Indexed Gather（`rvv_gather_indexed_memory_access.md`）行：im2col 的 input 地址是仿射步进（`addw a0,a0,a2`，`input_col += stride_w`），无 LUT / 数据相关索引；行内互斥「固定 element/byte stride → strided-layout row」先排除 gather。
- RVV Strided Layout Transform（`rvv_strided_layout_transform_kernels.md`）行：行内互斥「卷积固定邻域 → spatial-convolution row」——im2col 是卷积 patch 抽取，固定邻域语义成立，由 spatial-convolution 认领。
- RVV Layout and Channel Packing（`rvv_layout_and_channel_packing_kernels.md`）行：本函数是固定邻域抽取 + padding 零填充 + 连续写，非 channel interleave / 位域 / numeric panel packing；固定邻域访问归 spatial row。
- RVV Resampling 行：无运行期分数坐标 / index / weight 合同，排除。
- `matrix_engine_im2col_convolution_lowering.md`：目标无矩阵引擎，准入 gate 不成立，排除。

**(c) 双 Confidence 推导式**：
- `route: compiler-generated scalar（im2col.h 源码吻合）+ hardware v + build v1p0 + hot loop zero v* + 固定 stencil/边界语义 → High`
- `impact: interval 函数内样本份额 98.85% + VLEN=128 + 非 memory-bound（L1 miss 0.628%）→ Medium（缺 global-period 采样语义与 workload 贡献，Amdahl 上界不可声明）`

#### Finding 2（supporting of Finding 1）— No vectorization

**(a) 逐字 evidence 引用**：hot main loop（12af566–12af586）逐指令为 scalar：`addw/slli/fmv.w.x/add/addi/bgeu/flw/ld/fsw/bne`，**zero `v*`、zero `th.v*`**；hardware `isa` 含 `v`，build `Tag_RISCV_arch` 含 `v1p0`。supporting because: 同一机制——该 scalar loop 未使用目标核的 RVV 执行资源，只解释「缺少向量执行」的存在性，不决定收益载体。

**(b) 互斥邻居排除**：行内互斥「operator semantic shape → 各自更具体 row」——spatial-convolution 更具体并已命中为 primary，本行让位。

**(c) 双 Confidence 推导式**：supporting 行不单独推导（两种 confidence 保持 `—`）。

#### Finding 3（supporting of Finding 1）— Register Pressure and Save/Restore

**(a) 逐字 evidence 引用**：`32.92 :   12af57c:        ld      a2,-136(s0)` —— hot interval 每轮重复从栈 reload loop-invariant 的 stride 值（紧随其后的 `10.29 : 12af584: addw a0,a0,a2` 用它推进 `input_col += stride_w`），interval 内无任何对 -136(s0) 的 store。supporting because: 同一机制——scalar codegen 因 live set 压力把 loop-invariant 值 spill 到栈，每轮 reload；按 `rvv_spatial_convolution_and_pooling_kernels.md` §The fix 1 与 arbitration 的因果消除测试，向量化主循环后该 reload 随 scalar 流一起消失，故为 supporting（自身 row gate + 直接反汇编证据均成立）。

**(b) 互斥邻居排除**：
- Loop Induction Variable Strength Reduction 行：data_col 侧已用指针递增（`addi a4,a4,4` + 预计算 end pointer `bne a4,a1`）；input 侧索引 `a0` 因 bounds check（`bgeu a0,t6`）与仿射 stride 步进必须保留，index scaling 无法完全消除 → 排除。
- Load/Store Addressing-Mode Fusion 行：`addw a2,a0,t0; slli a2,a2,0x2; add a2,a2,a6` 三段地址生成对 strided gather 不可折叠（RISC-V 寻址模式仅 base+imm12，无 reg<<2+reg）→ 排除。
- Control-Flow Layout and Transfer Selection 行：无 branch-mispredict 主导证据（整体 branch_miss_rate 仅 1.07%）→ 排除。
- Kernel Selection 行：无现有向量 kernel 未选中（mxnet 仅有通用模板实现）→ 排除。

**(c) 双 Confidence 推导式**：supporting 行不单独推导（两种 confidence 保持 `—`）。

### 多候选仲裁

- 三个命中解释同一 hot loop（12af550–12af586，98.85%）的同一机制链：scalar compiler-generated 卷积 patch 抽取。**primary = RVV Spatial Convolution and Pooling Kernels**（L1 vectorization/semantic dispatch）；**No vectorization（L1）与 Register Pressure（L4）为 supporting**，写入 primary 的 Supporting evidence 行，不计顶层 finding 数、不单独排序。
- 因果消除测试：沿输出位置（output_col lane）向量化 im2col 抽取后，逐元素 `bgeu` 边界分支、`flw/fsw` 标量流、`ld -136(s0)` 栈 reload 与 `fmv.w.x` 零物化整条 scalar 链消失 → 上层（数据访问改写 / 向量化）为 primary，下层 codegen 症状并入 supporting。
- 入口条件 A：顶层 finding 仅 1 个（primary），evidence sample share = 函数内 98.85%（cpu-clock，local period）。收益上界表述为「当前 sampled event 下的函数内局部样本份额」；因采样语义四条不成立（local period、workload 贡献未知），**不**称 workload 级 Amdahl 上界。

## Phase 4 — Root-cause blueprint / 根因蓝图：mxnet::op::im2col<float>

纳入蓝图的 pattern 对应 Phase 3 通过 gate 的 row：
- primary：`rows-operator-rvv.md` row「RVV Spatial Convolution and Pooling Kernels」→ `patterns/rvv_spatial_convolution_and_pooling_kernels.md`（已读全）
- supporting：row「No vectorization」→ `patterns/no-vectorization.md`（已读全）；`rows-codegen.md` row「Register Pressure and Save/Restore」→ `patterns/register_pressure_and_save_restore.md`（已读全）

### 1. Root cause

`im2col_cpu<float>`（im2col.h L88–138）的 patch 抽取主循环在 RVV-capable 目标（hardware + build 均含 `v`，VLEN=128）上完全由编译器生成标量代码：固定 stencil（kernel_row/kernel_col 嵌套）的邻域地址生成 + `is_a_ge_zero_and_a_lt_b` 的 interior/border 分支 + data_col 连续 store，每元素移动 4 字节需要约 11 条指令，其中 `ld a2,-136(s0)` 每轮重复 reload loop-invariant stride。依据 `patterns/rvv_spatial_convolution_and_pooling_kernels.md` §Why this is slow：固定 stencil 的根因是「重复读取高度重叠的邻域、错误向量化轴…interior/border 混在主循环」，本函数正属「interior/border 分支混在主循环 + 逐元素标量抽取」的未向量化形态；leverage point 是「沿独立输出位置/通道向量化、把 border 与 overlap contract 显式分离」。supporting 依据 `patterns/no-vectorization.md` §Why this is slow：vector unit 未处理 hot main-loop 的并行元素（`VLMAX = LMUL × VLEN / SEW`，SEW=32、LMUL=1、VLEN=128 → 4 lanes/op）。supporting 依据 `patterns/register_pressure_and_save_restore.md`：hot interval 的栈 reload 属 spill/reload 主导，直接反汇编证据成立。

### 2. The fix / 修复方式

纠正对象：`im2col_cpu` 的 `output_col` 内层循环（L124–131）——将固定 stride 邻域抽取改为直接向量访问路径，并把 interior / border 分离（依据 `rvv_spatial_convolution_and_pooling_kernels.md` §The fix 5「Separate interior and border paths」与 §The fix 1「Replace fixed-stride deinterleave with a direct access path」）。

修复前（当前 scalar 形态，伪代码）：
```cpp
// im2col.h L124-131：每个 output_col 一次 4 字节移动，逐元素边界判断
int input_col = -pad_w + kernel_col * dilation_w;
for (int output_col = output_w; output_col; output_col--) {
  if (is_a_ge_zero_and_a_lt_b(input_col, width)) {
    *(data_col++) = data_im[input_row * width + input_col];
  } else {
    *(data_col++) = 0;
  }
  input_col += stride_w;
}
```

修复后（RVV intrinsic 代表形态，沿 output_col 连续位置向量化；VLEN-agnostic，按 `kernel-conventions.md` §3 取 runtime `vl`）：
```cpp
// interior 范围：input_col = -pad_w + kernel_col*dilation_w + o*stride_w
// 对所有 lane 均满足 0 <= input_col < width 的 o 区间。
// border（左/右不足一整个 vl 或越界）走独立 scalar/安全 fallback。
int input_col0 = -pad_w + kernel_col * dilation_w;      // o=0 的输入列
const ptrdiff_t byte_stride = (ptrdiff_t)stride_w * sizeof(DType);
for (int o = 0; o < output_w; ) {
  size_t vl = __riscv_vsetvl_e32m1(output_w - o);       // SEW=32；LMUL 按 live budget 选
  const float* base = data_im + input_row * width + input_col0 + o * stride_w;
  vfloat32m1_t vals;
  if (stride_w == 1) {
    vals = __riscv_vle32_v_f32m1(base, vl);             // 连续快速路径
  } else {
    vals = __riscv_vlse32_v_f32m1(base, byte_stride, vl); // 固定 stride 直接路径
  }
  __riscv_vse32_v_f32m1(data_col, vals, vl);            // data_col 连续写
  data_col += vl; o += vl;
}
```
- padding 零填充通过显式分离：整行越界的 row 沿用现有 memset 路径（L119–121）；列越界由独立 border scalar fallback 处理，**不构造越界指针再依赖 mask**（模式 §The fix 5 硬性要求）。
- 适用前提：output_w > 0；`data_im` 与 `data_col` 不 alias（mshadow 独立 buffer）；内核维度、stride、dilation、pad 在一次调用内不变。
- Correctness contract：data_col 输出布局必须与标量路径逐元素 bit-identical（channels × kernel_h × kernel_w × output_h × output_w，浮点仅搬运无算术，无 rounding 风险）；padding 位置精确写 0；`input_row` 越界时整行写 0 的语义（L118–121）保持。
- 限制/风险：`vlse32`（stride 加载）在 C920v2 的吞吐需 A/B benchmark 验证（模式 Related PRs 中 OpenCV 5be158a2b6ed 为「事后 oracle，不预设 strided load 普遍最优」）；VLEN=128 下小 output_w 可能有配置开销拐点；LMUL 选型须满足 `kernel-conventions.md` §2 的 live-register budget（`LMUL * peak_live_vectors <= 32`）。
- 修复后预期 Profile signals：12af566–12af586 区间的 `addw/slli/add/fmv.w.x/bgeu/flw/ld/fsw/addw/bne` 标量流消失或大幅缩小；出现 `vsetvli/vle32/vlse32/vse32`；`ld a2,-136(s0)` 栈 reload 消失。

### 3. Baseline facts 回填

hardware ISA：`rv64imafdcv_...zve64d_zvfh...`（C920v2，RVV 1.0，OoO）；build ISA：`libmxnet.so` `Tag_RISCV_arch` 含 `v1p0`/`zve64d`/`zvl128b`（**build 已含 V，非 build-flag 缺口**）；VLEN=128（vlenb=16）；bound type：instruction-issue/latency-bound（IPC 0.639，L1 miss 0.628%），非 memory-bandwidth-bound；`baseline_gap: sampling metadata` + `baseline_gap: sampling IP precision`。

### 4. 收益上界

primary 的 evidence sample share：**函数内局部样本份额 ≈ 98.85%**（cpu-clock，local period，2245 samples 中约 2219 落在 12af550–12af586）。采样语义四条不成立（percent type=local period、函数 workload 贡献未知）→ 仅表述「当前 sampled event 下的函数内局部样本份额」，**不**称 workload 级 Amdahl 上界。supporting 不单独计算。

### 5. 三维路由判定

- current source：compiler-generated scalar C++（`im2col.h` L88–138 模板，内联入 `im2col` wrapper 的 `.isra.0` clone）；无手写 assembly、无 intrinsic kernel。
- implementation existence/reachability：mxnet 仓库中不存在任何向量化 im2col 实现；通用模板为唯一实现，无 dispatch/registration/fallback 机制——非「内核未选中」。
- function-level policy：mxnet 无要求或禁止 im2col 使用 assembly/intrinsic 的函数级 policy → 不进入 policy-backed missing `.S` 分支；fix 载体为普通 intrinsic 代码或编译器可向量化改写（autovec 未能处理该 bounds-checked strided 抽取形态）。

### 6. Implementation-shape proof

不适用（非 policy-backed missing `.S` 分支）。

### 7. Related PRs

`patterns/rvv_spatial_convolution_and_pooling_kernels.md`（primary）→ Related PRs：15 条 URL
- https://github.com/uxlfoundation/oneDNN/pull/5506
- https://github.com/uxlfoundation/oneDNN/commit/8208c6a731edff86af37b1ede3208775cd46aaef
- https://github.com/uxlfoundation/oneDNN/commit/4f915a57ce2fd600b9ca04bf5a312c365a4b1830
- https://github.com/uxlfoundation/oneDNN/commit/563c5fcfcf8ad591b43d3dc7e0bc9292102eeba0
- https://github.com/uxlfoundation/oneDNN/commit/56a75b311f2e9fdb176b3ca30a007c7fcdc641de
- https://github.com/uxlfoundation/oneDNN/commit/a4c4855a9f4c478177e7fc799f64ac1457553ad7
- https://github.com/uxlfoundation/oneDNN/pull/4323
- https://github.com/uxlfoundation/oneDNN/commit/ce942914cb9d51c84ee0fc7f3ae253d6cd0121f6
- https://github.com/uxlfoundation/oneDNN/commit/dd1dff668fe9194d9a955333461e299d19aefb1c
- https://github.com/alibaba/MNN/pull/4042
- https://github.com/alibaba/MNN/commit/672c5862392393c171f1513bf7994d3b95e2a6a1
- https://github.com/alibaba/MNN/pull/4359
- https://github.com/uxlfoundation/oneDNN/pull/4735
- https://github.com/uxlfoundation/oneDNN/pull/5345
- https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5

supporting pattern（no-vectorization / register-pressure）不单独成蓝图，不重复输出 Related PRs。

## Phase 5 — Verification forecast / 验证预测：mxnet::op::im2col<float>

primary（RVV Spatial Convolution and Pooling Kernels）——修复对象：12af550–12af586 scalar 抽取循环 → RVV strided/连续向量路径 + interior/border 分离。

- **应消失/缩小**（锚定 Phase 3(a) 引用行）：`32.92 : 12af57c: ld a2,-136(s0)`、`16.26 : 12af578: flw fa5,0(a2)`、`20.00 : 12af566: addw a2,a0,t0`、`7.31 : 12af580: fsw fa5,-4(a4)`、`4.86 : 12af574: bgeu a0,t6` 所在 scalar 流的函数内样本份额显著下降；`ld a2,-136(s0)` 随 scalar 流消失。
- **应出现**（锚定 `patterns/rvv_spatial_convolution_and_pooling_kernels.md` §Verification）：`vsetvli` + `vle32`/`vlse32` + `vse32` 指令出现在该函数 annotate；stride_w==1 时连续 `vle32`、stride_w>1 时 `vlse32` 固定 stride 直接路径（§1「direct access path」预期信号）；border 分支与标量邻域抽取份额下降，cycles/output 改善。
- 覆盖要求（§Verification correctness contract）：kernel/stride/dilation/padding 组合、interior 与全部 border、channel tail；`output_w` 的 0、小值、整 vl 倍数与 1..vl-1 全部 tail；A/B vs scalar reference；检查反汇编无 per-iteration `vsetvli`（`kernel-conventions.md` §Verification）。
- 升级 profile-backed 补充数据（本模式已为 profile-backed，此条为补齐采样元数据）：`perf report --header-only` / `perf evlist -v` 核对 `precise_ip` 与 event attr，以支持 instruction-level 归因。

supporting（No vectorization）：跟随 primary 验证——同一函数 annotate 出现 `v*`，scalar 指令不再主导 hot loop（`patterns/no-vectorization.md` §Verification）。
supporting（Register Pressure）：跟随 primary 验证——`ld a2,-136(s0)` 栈 reload 消失（register save/restore §Verification：re-run annotate 确认 stack/move interval 缩小）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`mxnet::op::im2col<float>`（Phase 3–5 全部出现） |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表；gap 标签：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`（include: rows-operator-rvv、rows-codegen；exclude 6 项）；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（primary，supporting 2）；evidence 锚点 `32.92 : 12af57c: ld a2,-136(s0)`、`98.85%` interval、`20.00 : 12af566: addw a2,a0,t0` 等；supporting 2 行；排除条数 8（gather/strided/layout/resampling/matrix-engine/induction/addressing-mode/control-flow/kernel-selection）；推导式 1（route High / impact Medium） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件：`rvv_spatial_convolution_and_pooling_kernels.md`、`no-vectorization.md`、`register_pressure_and_save_restore.md`；对应命中 row：spatial-convolution / no-vectorization / register-pressure；引用短语首词：`重复读取高度重叠的邻域`、`vector unit 没有处理`、`spill/reload`；The fix 含 before/after、适用前提、correctness contract、风险、预期 Profile signals 锚点（`vsetvli/vle32/vlse32/vse32`、`ld a2,-136(s0)` 消失）；missing `.S` 不适用；Related PRs：`rvv_spatial_convolution_and_pooling_kernels.md` 15 条 URL |
| 5 | 路径合规 | ✅ | 8 项 trace 可解释扫描集；primary/supporting 仲裁与 L1/L4 归属合规；每个 blueprint leaf 均来自通过 gate 的 row；入口模式 A 按动态份额排序（primary 98.85% 函数内份额）；无 `th.v*`，无全局停扫 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧：`12af57c: ld a2,-136(s0)`、`12af578: flw fa5,0(a2)`、`12af566: addw a2,a0,t0`、`12af574: bgeu a0,t6`；出现侧：`patterns/rvv_spatial_convolution_and_pooling_kernels.md` §Verification（vsetvli/vle*/vlse*/vse* 预期信号） |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；交付物止于 Profile 证据、根因蓝图、完整 The fix、验证预测 |

修正记录：无