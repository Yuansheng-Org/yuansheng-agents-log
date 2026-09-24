Functions under analysis: [void mxnet::op::mxnet_op::Kernel<mxnet::op::mxnet_op::op_with_req<mxnet::op::mxnet_op::backward_grad_tuned<mxnet::op::mshadow_op::relu_grad>, 1>, mshadow::cpu>::LaunchTuned<mxnet::op::mxnet_op::backward_grad_tuned<mxnet::op::mshadow_op::relu_grad>, float, float*, float*, float*>(mshadow::Stream<mshadow::cpu>*, unsigned long, float*, float*, float*) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（rank 019 `Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>, cpu>::LaunchTuned<..., float, float*, float*, float*> [clone ._omp_fn.0]`，114 samples，event=`cpu-clock`，`percent: local period`；覆盖完整函数体，单 hot loop）
- perf stat（可选 bound/context）：已提供（整 testcase；IPC=0.562、LLC_load_miss_rate=43.99%、L1_dcache_load_miss_rate=5.16%、branch_miss_rate=2.72%）
- workload/binary/DSO/source context：已提供（`libmxnet.so`，mxnet master commit b84609d3fc73d20929c114eab95faaa56e6c5ede；`src/operator/mshadow_op.h:1536-1545` relu_grad、`mxnet_op.h:1077-1096` LaunchTuned；当前实现是 compiler-generated C++ 模板）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（`libmxnet.so-elf-A`：`rv64i2p1_..._v1p0_..._zvl128b1p0`，含 `v`、`zvl128b`）
- hardware ISA：已提供（`rv64imafdcv_..._zve64d_zvfh...`，含 `v`）
- `vlenb`：已提供（16 bytes → VLEN=128 bits）
- 采样元数据：已提供（event=`cpu-clock`，`percent: local period` 非 global-period；workload 贡献未知）→ Phase 1 采样语义 gate
- Sampling IP precision：缺失（无 precise_ip/Exact-IP 标记）→ `baseline_gap: sampling IP precision`

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zve64d_zvfh...`：暴露标准 `v`（RVV 1.0），VLEN=128 |
| Build ISA | `Tag_RISCV_arch` = `rv64i2p1_..._v1p0_..._zvl128b1p0`：含 `v`、`zvl128b`；无 IFUNC/multiversion |
| Vector flavor | 全函数 zero `v*`/`th.v*`（全 scalar）→ 无 flavor mismatch |
| VLEN | `vlenb`=16 → VLEN=128 bits；SEW=32 时 VLMAX(m1)=4 lanes |
| Bound type | 函数内由 compare/branch/select（NaN 检查 feq.s/beqz + 符号 flt.s/bnez）与访存/指针更新主导（branch/FP-latency bound）；全局 IPC=0.562、LLC miss 43.99%、branch miss 2.72% 为上下文约束 |
| Sampling semantics | event=`cpu-clock`；percent type=`local period`（非 global-period）→ 只有函数内局部份额；禁止 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision` → 单行只锚定 loop interval |

L0 baseline gate：hardware 含 `v`、build 含 `v` → 无 mismatch。`th.v*` gate：不适用。Bound-type gate：branch 密集，向量化（mask 化消除分支）的收益方向明确但幅度受全局上下文约束。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：仅 `Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>>::LaunchTuned<..., float, float*, float*, float*> [._omp_fn.0]`。

hot loop 边界与 trace anchor（source: `mshadow_op.h:1536-1545` relu_grad + `mxnet_op.h:1077-1096`；单 main loop，unit-stride 连续，每元素处理一个输出）：
- **Loop interval（124e298 – 124e2ba）**：`124e298: flw fa5,0(a3)`（1.75%，rhs/x load）、`124e29c: flw fa4,0(a5)`（18.42%，lhs/grad load）、`124e2a0: feq.s a1,fa5,fa5`（13.16%，IsNan 检查）、`124e2a4: beqz a1,124e2cc`（10.53%，NaN 分支）、`124e2a6: flt.s a1,fa3,fa5`（21.05%，0<x）、`124e2aa: bnez a1,124e2b0`（0.88%）、`124e2ac: fmul.s fa4,fa4,fa3`（0.00%，x<=0 → grad×0）、`124e2b0: fsw fa4,0(a4)`（18.42%，输出 store）、`124e2b4: addi a5,a5,4`（14.04%，指针递增）、`124e2ba: bne a5,a2,124e298`（1.75%，back-edge）；NaN 路径 `124e2cc: fmul.s fa4,fa4,fa5`（grad×a，NaN 传播）。
- 最高占比行：`21.05 :  124e2a6:  flt.s   a1,fa3,fa5`；interval 样本合计 ≈ **100.0%**（1.75+18.42+13.16+10.53+21.05+0.88+18.42+14.04+1.75 = 100.00，114 样本 local period）。
- 语义链：`out[i] = lhs[i] * relu_grad(rhs[i])`；`relu_grad(a)`：`IsNan(a) → a`（NaN 传播），`a > 0 → 1`，否则 `0`。每元素 2 次 compare（feq.s/flt.s）+ 2 次分支（beqz/bnez），branch 密集。

Sampling IP precision 不足 → 结论收敛到 interval-level mechanism。annotate 覆盖完整。入口模式 A（profile_backed）。

## Phase 3 — Pattern scan / 模式扫描：void mxnet::op::mxnet_op::Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>>::LaunchTuned<...>

### Class selection trace（8 项）

1. `rows-asm.md` — exclude：compiler-generated C++ 模板（OpenMP clone），无手写 `.S`/policy 四证。
2. `rows-operator-rvv.md` — include：compiler-generated scalar loop + 明确 activation 语义（ReLU 家族逐元素激活 backward，piecewise compare/branch/select）。
3. `rows-string-memory.md` — exclude：非 string/memory/copy/compare 语义。
4. `rows-vectorized-tuning.md` — exclude：全函数 zero `v*`。
5. `rows-codegen.md` — include：compiler-generated 指令形态需逐 row 排除（compare/branch 形态、zero 物化、循环控制）。
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD 信号。
7. `rows-crypto.md` — exclude：无密码原语。
8. `rows-runtime-os.md` — exclude：无 timer/ISR/CSR 信号。

### Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: `Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>>::LaunchTuned`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Elementwise Activation Kernels（primary） | relu_grad 逐元素 piecewise activation backward：compare/branch/select 主导（`21.05 : 124e2a6: flt.s`、`13.16 : 124e2a0: feq.s`、`10.53 : 124e2a4: beqz`、`0.88 : 124e2aa: bnez`）+ 访存/指针（`18.42 : 124e29c: flw`、`18.42 : 124e2b0: fsw`、`14.04 : 124e2b4: addi`）；lane-independent，无跨 lane 归约 | High | Low | `patterns/rvv_elementwise_activation_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`；hardware+build 均含 `v` | — | — | `patterns/no-vectorization.md` |

**顶层 finding 三件套（primary — RVV Elementwise Activation Kernels）：**

(a) **逐字 evidence 引用**（单 main loop interval 124e298–124e2ba）：
- `21.05 :  124e2a6:  flt.s   a1,fa3,fa5` —— `a > 0` 符号检查（interval 主导行）；
- `13.16 :  124e2a0:  feq.s   a1,fa5,fa5` 与 `10.53 :  124e2a4:  beqz    a1,124e2cc` —— `IsNan(a)` 检查与 NaN 分支；
- `18.42 :  124e29c:  flw     fa4,0(a5)`（lhs/grad load）与 `18.42 :  124e2b0:  fsw     fa4,0(a4)`（`KERNEL_ASSIGN(out[i], req=1, ...)` store）；
- `0.00 :  124e2ac:  fmul.s  fa4,fa4,fa3`（x<=0 → grad×0）与 `0.00 :  124e2cc:  fmul.s  fa4,fa4,fa5`（NaN 路径 grad×a）；
- `1.75 :  124e298:  flw     fa5,0(a3)`（rhs/x load）、`14.04 :  124e2b4:  addi    a5,a5,4`（指针递增）、`1.75 :  124e2ba:  bne     a5,a2,124e298`（back-edge）。

语义链（source `mshadow_op.h:1536-1545` + `op_with_req::Map` 的 `KERNEL_ASSIGN(out[i], req, OP::Map(lhs[i], rhs[i]))` + `backward_grad_tuned` 的 `a * GRAD_OP::Map(...)`）：`out[i] = lhs[i] * relu_grad(rhs[i])`，`relu_grad(a)=NaN→a / a>0→1 / 0`，完全 lane-independent。

(b) **互斥邻居排除**：
- elementwise-arithmetic row：排除 —— 该行只认领 add/sub/mul/div/min/max/affine 连续算术；此处是 ReLU 分段激活（NaN 检查 + 符号选择），compare/branch/select 主导（feq.s 13.16% + flt.s 21.05% + 分支 11.41%），属 piecewise activation；
- normalization row：排除 —— 行内互斥 "统计归约+transform → normalization row"；无跨 lane statistics/reduction；
- extrema / arg-extrema rows：排除 —— 无 min/max/index 选择语义；
- precision-conversion / eliminate-unnecessary-precision-conversions：排除 —— 全 float 无转换主导；
- zero-based-comparison row（codegen）：排除 —— `fmv.w.x fa3,zero`（124e288）只物化一次 FP 零常量供比较，非逐元素 redundant zero temporary，样本不主导（该行 0.00%），属语义必需比较操作数；
- kernel-selection row：排除 —— mxnet 无该 activation 的既有 RVV kernel/dispatch slot 证据。

(c) **双 Confidence 推导式**：
- route：compiler-generated provenance（annotate 源码行映射）+ source 确认 relu_grad lane-independent piecewise activation 语义（NaN→a / a>0→1 / 0）+ hardware `v`/build `v` 下全 scalar main loop + 互斥排除 → High；
- impact：函数内局部份额成立（interval ≈100%），但采样语义四条不全（`local period` 非 global-period）、workload 贡献未知、无 ARM 对照、branch-miss 全局 2.72% 表明分支收益方向需实测 → Low（只能讨论方向，不估算收益幅度）。

**多候选仲裁小段：** primary = RVV Elementwise Activation Kernels；supporting = No vectorization（同一 hot loop、同一向量化载体机制）。evidence-mechanism layer：L1（vectorization / semantic dispatch）。supporting 不计顶层 finding 数。顶层 finding 局部样本份额加总 ≈ **100.0%**（114 样本 local period，函数内；非 workload 级）。

**零命中路径不适用。** 其余 rows 无对应 signal（见 class trace 与 (b) 排除）。

## Phase 4 — Root-cause blueprint / 根因蓝图：void mxnet::op::mxnet_op::Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>>::LaunchTuned<...>

**纳入蓝图的 pattern 与对应 row**：primary row「RVV Elementwise Activation Kernels」（`rows-operator-rvv.md`，已过 gate）；supporting row「No vectorization」。

1. **Root cause**：relu_grad 是 lane-independent piecewise activation backward，但每元素以 2 次标量 compare（`feq.s` NaN 检查 13.16% + `flt.s` 符号检查 21.05%）+ 2 次分支（`beqz` 10.53% + `bnez` 0.88%）+ 3 次访存/指针（flw 18.42% + fsw 18.42% + addi 14.04%）的标量分支形态执行，branch/compare 密集且无向量 mask/select；函数零 `v*`，RVV compare/mask/select 完全未参与。依据 `patterns/rvv_elementwise_activation_kernels.md` §Why this is slow：逐元素 activation 的根因是 lane-independent 工作仍以标量分支/数学 helper 执行；主要 leverage point 是 RVV compare/mask/select。
2. **The fix / 修复方式**（与 `patterns/rvv_elementwise_activation_kernels.md` §2「Vectorize simple piecewise activations with RVV masks」一致；修复对象 = mxnet `backward_grad_tuned<relu_grad>` kernel 路径，普通代码/intrinsic 向量化）：

```cpp
// Before（当前标量形态，mshadow_op.h:1536-1545 + op_with_req::Map + backward_grad_tuned）
for (i = 0; i < N; ++i) {   // #pragma omp parallel for（保留分片）
  float x = rhs[i];
  float m;
  if (IsNan(x)) m = x;              // 124e2a0 feq.s + 124e2a4 beqz（NaN 传播）
  else m = (x > 0.0f) ? 1.0f : 0.0f; // 124e2a6 flt.s + 124e2aa bnez
  out[i] = lhs[i] * m;              // 124e2ac/124e2cc fmul + 124e2b0 fsw
}

// After（RVV 转换形态示意；fixed-VL main + runtime-VL tail，branch-free mask/select）
while (remaining > 0) {
  const size_t vl = __riscv_vsetvl_e32m1(remaining);
  vfloat32m1_t x = __riscv_vle32_v_f32m1(rhs + off, vl);
  vfloat32m1_t g = __riscv_vle32_v_f32m1(lhs + off, vl);
  vbool32_t gt  = __riscv_vmfgt_vf_f32m1_b32(x, 0.0f, vl);   // a > 0（vflt.vf 语义等价形态）
  vfloat32m1_t m = __riscv_vmerge_vvm_f32m1(__riscv_vfmv_v_f_f32m1(0.0f, vl),
                                            __riscv_vfmv_v_f_f32m1(1.0f, vl), gt, vl); // a>0→1 else 0
  vbool32_t nan = __riscv_vmfeq_vv_f32m1_b32(x, x, vl);      // IsNan：!(x==x)
  nan = __riscv_vmnot_b32(nan, vl);
  m = __riscv_vmerge_vvm_f32m1(m, x, nan, vl);               // NaN → x（NaN 传播，保留 payload 位）
  __riscv_vse32_v_f32m1(out + off, __riscv_vfmul_vv_f32m1(m, g, vl), vl);
  off += vl; remaining -= vl;
}
```

前置/适用前提：
- NaN 语义：`IsNan(a) → a`（返回 NaN 输入本身再 ×grad）—— 向量化用 `vmfeq(x,x)` 取反生成 NaN mask，`vmerge` 保持 NaN payload 位（与 reference 的 `feq.s`+分支逐位可比）；需验证 NaN payload 传播与 ±NaN canonicalization 语义（vector-math-conventions §Numeric contract）；
- `a > 0 → 1` 的 signed-zero 语义：`-0.0f > 0` 为 false → 输出 0；`vmerge` 的 select 必须保持该语义（pattern §2 注明不得默认 `vfmax` 等价）；
- `backward_grad_tuned` 的 `a * GRAD_OP::Map(...)` 合同与 `KERNEL_ASSIGN` req 语义（本实例 req=1 直写）保持；
- 乘法的 FP 舍入：`lhs × (0|1|x)` 中 ×1 与 ×0 的舍入/符号位行为（×0 得 ±0 的符号位）须与 reference 一致。

Correctness contract 不可破坏项：relu_grad 三分支语义（NaN→a / a>0→1 / else 0）、NaN payload 传播、signed-zero 行为、`out[i]=lhs[i]·relu_grad(rhs[i])` 乘法顺序、OpenMP 分片边界与 in-place/alias 合同。

限制/风险：mask 化消除分支后，若输入分布极端（大量 NaN 或全零），需实测确认无退化；NaN payload 语义与项目 canonicalization 的一致性需验证；小 N 退化需 tail 判据与短输入基准（kernel-conventions §3）。

修复后预期 Profile signals：`124e2a6: flt.s`（21.05%）、`124e2a0: feq.s`（13.16%）、`124e2a4: beqz`（10.53%）、`124e29c: flw`（18.42%）、`124e2b0: fsw`（18.42%）、`124e2b4: addi`（14.04%）份额消失；出现 `vsetvli`/`vle32`/`vmfgt`/`vmfeq`/`vmerge`/`vfmul`/`vse32`；函数局部 cycles/element 改善（方向性）。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcv_..._zve64d_zvfh...`（RVV 1.0，C920v2/SG2044，OoO）；build ISA = `rv64i2p1_..._v1p0_..._zvl128b1p0`（含 `v`、`zvl128b`）；VLEN = 128 bits；bound type = 函数内 compare/branch/select + 访存主导（branch/FP-latency）。
4. **收益上界**：当前 sampled event（cpu-clock）下的函数内局部样本份额 ≈ **100.0%**（114 样本 local period）；`baseline_gap: sampling metadata` → 禁止 workload 级 Amdahl 上界。
5. **三维路由判定**：`current source` = compiler-generated C++ 模板（OpenMP clone）；`implementation existence/reachability` = mxnet 无 relu 的既有 RVV kernel/dispatch slot → 不走 missing `.S` 分支；`function-level policy` = `relu_grad` 是 `tunable` 算子（走 LaunchTuned OpenMP 分片），无要求独立 `.S` 的 policy → 修复载体为普通代码/intrinsic 向量化。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。shape 要点：N 动态（运行期）→ `LMUL_min(W): N/A`，按 live set/unroll/吞吐选 LMUL（SEW=32, VLEN=128 → m1=4 lanes；live set ≈ 3-4 vectors → m1–m2 稳妥）；tail 用 runtime-VL；`implementation_shape_gap: 无`。
7. **Related PRs 小节**：
   - `patterns/rvv_elementwise_activation_kernels.md`：Related PRs：10 条 URL（OpenCV `524d8ae01c63`、`86241653a781`、`#27072`；oneDNN `aa23ab557391`、`d5ae44880903`；llama.cpp `#17227`、`#15057`；MNN `#4508`、`#4484`、`#4044`、`b7268aa`、`#4359`）。
   - `patterns/no-vectorization.md`（supporting）：Related PRs：16 条 URL（OpenCV `#22179`、`#22520`、`#23980`、`#24058`、`#24132`、`#24166`、`#24301`、`#24325`、`#27160`、`#27119`、`#27097`、`#27007`、`#26958`、`#26865`、`b902a8e792e1`、`2c16f3b7d2b2`、`e06502a254f7`、`a2d784b6f53a`、`83104bed3209`）。

## Phase 5 — Verification forecast / 验证预测：void mxnet::op::mxnet_op::Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>>::LaunchTuned<...>

（primary = RVV Elementwise Activation Kernels；supporting 不单独验证）

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`124e2a6: flt.s a1,fa3,fa5`（21.05%）、`124e2a0: feq.s`（13.16%）、`124e2a4: beqz`（10.53%）、`124e29c: flw fa4,0(a5)`（18.42%）、`124e2b0: fsw fa4,0(a4)`（18.42%）、`124e2b4: addi`（14.04%）、`124e2ba: bne`（1.75%）应消失或大幅缩小。
- **应出现侧**（锚定 `patterns/rvv_elementwise_activation_kernels.md` §Verification）：annotate 出现 vector compare/mask/select（`vsetvli`/`vle32`/`vmfgt`/`vmfeq`/`vmerge`/`vfmul`/`vse32`）；"scalar branch/math-helper share 下降"；cycles/element 改善。
- **正确性/数值合同**：NaN payload 传播（vmfeq+取反+vmerge vs feq.s 分支的逐位可比）、signed-zero（-0.0f → 0）、±0/±Inf/subnormal、×0/×1 的符号位行为、N=0/小 N/step 整倍数/全 tail 长度（kernel-conventions §3）。
- **收益顺序**：单一顶层 primary；幅度待补采。
- **补采升级**：至少需 (1) `precise_ip`/Exact-IP 确认；(2) `--percent-type=global-period` 重采量化 global 份额；(3) 向量化就绪后重跑 annotate 并 benchmark 短/中/长 N 与极端输入分布（大量 NaN/全零）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>>::LaunchTuned [._omp_fn.0]` |
| 2 | Phase 1 输出要求 | ✅ | 7 行 baseline；gap：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（primary=activation）+ supporting 1（no-vectorization）；锚点 `21.05 : 124e2a6: flt.s`、`13.16 : 124e2a0: feq.s`、`10.53 : 124e2a4: beqz`、`18.42 : 124e2b0: fsw`；supporting 1；排除条数 ≥7；推导式 2 组 |
| 4 | Phase 4 输出要求 | ✅ | 已读 pattern：`patterns/rvv_elementwise_activation_kernels.md`（命中 row=Elementwise Activation；引用 `Vectorize simple piecewise activations with RVV masks`）、`patterns/no-vectorization.md`（supporting；`zero v*`）；The fix before/after/correctness/风险/Profile signals 齐备；`Related PRs：10 条 URL`（activation）+ `Related PRs：16 条 URL`（no-vectorization supporting） |
| 5 | 路径合规 | ✅ | 模式 A；primary=activation（L1）→ supporting=no-vectorization；leaf 均来自已过 gate 的 row；动态份额仅函数内局部（100.0%）并显式 `baseline_gap: sampling metadata` |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 `124e2a6: flt.s`/`124e2a0: feq.s`/`124e2a4: beqz`/`124e2b0: fsw`；出现侧 `rvv_elementwise_activation_kernels.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；无追问；交付止于证据、根因蓝图、完整 The fix、验证预测 |

修正记录：无