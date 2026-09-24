Functions under analysis: [void mxnet::op::mxnet_op::Kernel<mxnet::op::mxnet_op::op_with_req<mxnet::op::mxnet_op::backward_grad_tuned<mxnet::op::mshadow_op::gelu_erf_grad>, 1>, mshadow::cpu>::LaunchTuned<mxnet::op::mxnet_op::backward_grad_tuned<mxnet::op::mshadow_op::gelu_erf_grad>, float, float*, float*, float*, float*>(mshadow::Stream<mshadow::cpu>*, unsigned long, float*, float*, float*, float*) [clone ._omp_fn.0]]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（rank 018 `Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>, cpu>::LaunchTuned<..., float, float*, float*, float*, float*> [clone ._omp_fn.0]`，132 samples，event=`cpu-clock`，`percent: local period`；覆盖完整函数体，单 hot loop）
- perf stat（可选 bound/context）：已提供（整 testcase；IPC=0.562、LLC_load_miss_rate=43.99%、L1_dcache_load_miss_rate=5.16%、branch_miss_rate=2.72%）
- workload/binary/DSO/source context：已提供（`libmxnet.so`，mxnet master commit b84609d3fc73d20929c114eab95faaa56e6c5ede；`src/operator/mshadow_op.h:620,628-631` erf_grad/gelu_erf_grad、`mxnet_op.h:1077-1096` LaunchTuned；当前实现是 compiler-generated C++ 模板）
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
| Bound type | 函数内由每元素 libm `expf@plt` 调用（erf_grad 内 exp）与串行 FP 链（fdiv/fmul/fadd/fmul）主导（helper-call/FP-latency bound）；全局 IPC=0.562、LLC miss 43.99% 为竞争性约束 |
| Sampling semantics | event=`cpu-clock`；percent type=`local period`（非 global-period）→ 只有函数内局部份额；禁止 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision` → 单行只锚定 loop interval |

L0 baseline gate：hardware 含 `v`、build 含 `v` → 无 mismatch。`th.v*` gate：不适用。Bound-type gate：helper-call 主导，向量化收益受 vector-math（exp）可达性约束。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：仅 `Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>>::LaunchTuned<..., float, float*, float*, float*, float*> [._omp_fn.0]`。

hot loop 边界与 trace anchor（source: `mshadow_op.h:628-631` gelu_erf_grad + `mxnet_op.h:1077-1096`；单 main loop，unit-stride 连续，每元素处理一个输出）：
- **Loop interval（11e04ee – 11e053e）**：`11e04f6: flw fs5,0(s1)`（4.55%，grad_out load）、`11e04fa: fdiv.s fa0,fs0,fs4`（a/√2）、`11e050a: fdiv.s fs3,fs3,fs0`（1.52%，b/a）、`11e0516: jalr <expf@plt>`（erf_grad 内 `exp(-(a/√2)²)`）、`11e051a/11e0522/11e0526: fcvt.d.s → fmul.d → fcvt.s.d`（erf_grad 的 double 往返，`2.0` 字面量提升）、`11e051e: fmul.s fs0,fs0,fs2`（7.58%）、`11e052a: fmul.s fs0,fs0,fa0`（15.91%）、`11e0532: fadd.s fs0,fs0,fs3`、`11e0536: fmul.s fs0,fs0,fs5`（59.85%，×grad_out）、`11e053a: fsw fs0,-4(s2)`（10.61%）、`11e053e: bne`（back-edge）。
- 最高占比行：`59.85 :  11e0536:  fmul.s  fs0,fs0,fs5`；interval 样本合计 ≈ **100.0%**（4.55+1.52+7.58+15.91+59.85+10.61 = 100.02，132 样本 local period）。
- 语义链：`out[i] = input_1[i] * gelu_erf_grad(input_2[i], input_3[i])`；`gelu_erf_grad(a,b) = b/a + 0.5f·a·erf_grad(a/√2)/√2`；`erf_grad(t) = 2.0/√π · exp(-t²)`（`2.0` 为 double 字面量 → 每元素 fcvt.d.s/fmul.d/fcvt.s.d 往返）。erf_grad 的 expf 主体样本归 libm。

Sampling IP precision 不足 → 结论收敛到 interval-level mechanism。annotate 覆盖完整。入口模式 A（profile_backed）。

## Phase 3 — Pattern scan / 模式扫描：void mxnet::op::mxnet_op::Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>>::LaunchTuned<...>

### Class selection trace（8 项）

1. `rows-asm.md` — exclude：compiler-generated C++ 模板（OpenMP clone），无手写 `.S`/policy 四证。
2. `rows-operator-rvv.md` — include：compiler-generated scalar loop + 明确 activation 语义（GELU(erf) 家族逐元素激活 backward）。
3. `rows-string-memory.md` — exclude：非 string/memory/copy/compare 语义。
4. `rows-vectorized-tuning.md` — exclude：全函数 zero `v*`。
5. `rows-codegen.md` — include：compiler-generated 指令形态需逐 row 排除（erf_grad 的 float64 往返 fcvt.d.s/fmul.d/fcvt.s.d、libm 调用、循环控制）。
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD 信号。
7. `rows-crypto.md` — exclude：无密码原语。
8. `rows-runtime-os.md` — exclude：无 timer/ISR/CSR 信号。

### Classes scanned: `rows-operator-rvv.md`, `rows-codegen.md`

### Local performance pattern scan: `Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>>::LaunchTuned`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Elementwise Activation Kernels（primary） | gelu_erf_grad 逐元素激活 backward：每元素 libm `expf@plt`（11e0516，erf_grad 的 exp）+ 串行 FP 链（`59.85 : 11e0536: fmul.s`、`15.91 : 11e052a: fmul.s`、`7.58 : 11e051e: fmul.s`、`1.52 : 11e050a: fdiv.s`、`10.61 : 11e053a: fsw`）；lane-independent，无跨 lane 归约 | High | Low | `patterns/rvv_elementwise_activation_kernels.md` |
| No vectorization（supporting） | 同一 main loop 全 scalar、zero `v*`；hardware+build 均含 `v` | — | — | `patterns/no-vectorization.md` |
| Eliminate Unnecessary Float32/Float64 Conversions（supporting） | erf_grad 的 `2.0` double 字面量强制每元素 `0.00 : 11e051a: fcvt.d.s fa0,fa0` + `0.00 : 11e0522: fmul.d fa0,fa0,fs1` + `0.00 : 11e0526: fcvt.s.d fa0,fa0` float64 往返；向量化重写（float 常量）自然消除 | — | — | `patterns/eliminate_unnecessary_precision_conversions.md` |

**顶层 finding 三件套（primary — RVV Elementwise Activation Kernels）：**

(a) **逐字 evidence 引用**（单 main loop interval 11e04ee–11e053e）：
- `59.85 :  11e0536:  fmul.s  fs0,fs0,fs5` —— `backward_grad_tuned` 的 `a * GRAD_OP::Map(...)`（×grad_out，interval 主导行）；
- `0.00 :  11e0516:  jalr    -786(ra) # 59f200 <expf@plt>` —— erf_grad 的 `math::exp(-(a·a))`（a=a/√2），exp 主体样本归 libm；
- `15.91 :  11e052a:  fmul.s  fs0,fs0,fa0` —— `0.5f·a·erf_grad(a/√2)` 组合；
- `1.52 :  11e050a:  fdiv.s  fs3,fs3,fs0` —— `b/a`；
- `10.61 :  11e053a:  fsw     fs0,-4(s2)` —— `KERNEL_ASSIGN(out[i], req=1, ...)` 输出 store；
- `0.00 :  11e051a/11e0522/11e0526: fcvt.d.s / fmul.d / fcvt.s.d` —— erf_grad 的 float64 往返（supporting 证据行）。

语义链（source `mshadow_op.h:620,628-631` + `op_with_req::Map` 的 `KERNEL_ASSIGN(out[i], req, OP::Map(input_1[i], input_2[i], input_3[i]))` + `backward_grad_tuned` 的 `a * GRAD_OP::Map(...)`）：`out[i] = input_1[i] · (input_3[i]/input_2[i] + 0.5f·input_2[i]·erf_grad(input_2[i]/√2)/√2)`，完全 lane-independent。

(b) **互斥邻居排除**：
- elementwise-arithmetic row：排除 —— 每元素含 `expf@plt` 超越函数调用（11e0516）与 erf_grad 组合，非纯 add/sub/mul/affine；
- normalization row：排除 —— 行内互斥 "统计归约+transform → normalization row"；无跨 lane statistics/reduction（loop 内无归约指令）；
- precision-conversion row / eliminate-unnecessary-precision-conversions row（作为独立 finding）：排除 —— 该 row 本身不命中顶层（conversion 往返行样本 0.00%，非主导），且向量化重写会自然消除该 signal（因果消除测试：上层向量化改写 → 下层 fcvt 往返消失）→ 降为 primary 的 supporting，不单独成 finding；
- matmul/其它 operator rows：排除 —— 无 matmul/reduction/conv 语义；
- kernel-selection row：排除 —— mxnet 无该 activation 的既有 RVV kernel/dispatch slot 证据。

(c) **双 Confidence 推导式**：
- route：compiler-generated provenance（annotate 源码行映射）+ source 确认 gelu_erf_grad lane-independent activation 语义（含 erf_grad 的 exp）+ hardware `v`/build `v` 下全 scalar main loop + 互斥排除 → High；
- impact：函数内局部份额成立（interval ≈100%），但采样语义四条不全（`local period` 非 global-period）、workload 贡献未知、vector-math（exp 的向量实现）可达性未证实（mxnet 无 vector-math stub）、libm 主体成本落在本符号外、无 ARM 对照 → Low（只能讨论方向，不估算收益幅度）。

**多候选仲裁小段：** primary = RVV Elementwise Activation Kernels；supporting 1 = No vectorization（同一 hot loop、同一向量化载体机制）；supporting 2 = eliminate-unnecessary-precision-conversions（同一 loop 的 fcvt.d.s/fmul.d/fcvt.s.d 往返，向量化 float 重写自然消除，route gate 成立但作为同一机制的次级命中，不单独成顶层 finding）。evidence-mechanism layer：L1（vectorization / semantic dispatch；conversion 属 L4 compute/codegen micro-structure，因被上层向量化覆盖而降级 supporting）。supporting 不计顶层 finding 数。顶层 finding 局部样本份额加总 ≈ **100.0%**（132 样本 local period，函数内；非 workload 级）。

**零命中路径不适用。** 其余 rows 无对应 signal（见 class trace 与 (b) 排除）。

## Phase 4 — Root-cause blueprint / 根因蓝图：void mxnet::op::mxnet_op::Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>>::LaunchTuned<...>

**纳入蓝图的 pattern 与对应 row**：primary row「RVV Elementwise Activation Kernels」（`rows-operator-rvv.md`，已过 gate）；supporting rows「No vectorization」与「Eliminate Unnecessary Float32/Float64 Conversions」（`rows-codegen.md`）。

1. **Root cause**：gelu_erf_grad 是 lane-independent transcendental activation backward，但每元素以 libm `expf@plt`（11e0516）执行 erf_grad 的 `exp(-t²)`（t=a/√2），配合 `2.0` double 字面量强制的 float64 往返（11e051a/11e0522/11e0526：fcvt.d.s→fmul.d→fcvt.s.d）与串行 FP 链（fdiv/fmul/fadd/fmul，含 b/a 除法与 0.5f·a·erf_grad 组合），输出 store（11e053a 10.61%）与最终 ×grad_out（11e0536 59.85%）承载主要样本；函数零 `v*`，RVV vector-math 完全未参与。依据 `patterns/rvv_elementwise_activation_kernels.md` §Why this is slow：逐元素 activation 的根因是 lane-independent 工作仍以标量分支/数学 helper 执行，或 mask 只屏蔽写回却仍计算昂贵超越函数。
2. **The fix / 修复方式**（与 `patterns/rvv_elementwise_activation_kernels.md` §4「Treat transcendental activations as vector-math pipelines」一致，并吸收 `patterns/eliminate_unnecessary_precision_conversions.md` §1 的 float32 常量吸收；修复对象 = mxnet `backward_grad_tuned<gelu_erf_grad>` kernel 路径，普通代码/intrinsic 向量化）：

```cpp
// Before（当前标量形态，mshadow_op.h:620,628-631）
for (i = 0; i < N; ++i) {   // #pragma omp parallel for（保留分片）
  float a = input_2[i], b = input_3[i];
  float t = a / SQRT_2;
  // erf_grad(t): 2.0/sqrt(PI)*expf(-t*t) —— 2.0 字面量 → fcvt.d.s/fmul.d/fcvt.s.d 往返
  float eg = 2.0f /* 实际 2.0 double */ / math::sqrt(PI) * math::exp(-(t * t));
  out[i] = input_1[i] * (b / a + 0.5f * a * eg / SQRT_2);  // 串行 fdiv/fmul/fadd/fmul
}

// After（RVV 转换形态示意；fixed-VL main + runtime-VL tail；全 float 常量）
while (remaining > 0) {
  const size_t vl = __riscv_vsetvl_e32m1(remaining);
  vfloat32m1_t a = __riscv_vle32_v_f32m1(input_2 + off, vl);
  vfloat32m1_t b = __riscv_vle32_v_f32m1(input_3 + off, vl);
  vfloat32m1_t g = __riscv_vle32_v_f32m1(input_1 + off, vl);
  vfloat32m1_t t = __riscv_vfdiv_vf_f32m1(a, SQRT_2f, vl);
  vfloat32m1_t tt = __riscv_vfmul_vv_f32m1(t, t, vl);
  vfloat32m1_t ex = /* 项目真实存在且通过验证的 RVV exp 实现 */;          // exp(-t*t)
  ex = __riscv_vfneg_vv_f32m1(ex, vl);
  vfloat32m1_t eg = __riscv_vfmul_vf_f32m1(ex, TWO_OVER_SQRTPI_f, vl);    // float 常量，无 double 往返
  vfloat32m1_t term = __riscv_vfmul_vv_f32m1(a, eg, vl);                  // a*erf_grad(t)
  term = __riscv_vfdiv_vf_f32m1(term, SQRT_2f, vl);
  vfloat32m1_t r = __riscv_vfadd_vv_f32m1(__riscv_vfdiv_vv_f32m1(b, a, vl), term, vl); // b/a + ...
  r = __riscv_vfmul_vv_f32m1(r, g, vl);                                    // ×grad_out
  __riscv_vse32_v_f32m1(out + off, r, vl);
  off += vl; remaining -= vl;
}
```

前置/适用前提：
- `exp` 无 RVV 单指令（vector-math-conventions）：必须提供通过误差合同验证的向量 exp 实现（多项式+range-reduction），写明确 ULP/relative error、domain、NaN/±Inf/overflow/underflow 行为；mxnet 当前无 vector-math stub → 最大工程前置；
- 消除 double 往返：`erf_grad` 的 `2.0/sqrt(PI)` 改为 float 常量（`2.0f/sqrtf(PI)` 预计算或直接系数），保持 float 数据流 —— 需验证与 reference（double 中间精度）的容差（eliminate-unnecessary-precision-conversions §Verification：中间 double 精度必须证明不属于可观察合同；erf_grad 输入输出均为 float，double 只是 `2.0` 字面量的编译器提升，属可消除冗余）；
- `b/a` 除法语义：a==0 时的除零行为（±Inf/NaN）必须与 reference 一致（向量化不得改变）；0.5f 与 SQRT_2 常量语义保持；
- `backward_grad_tuned` 的 `a * GRAD_OP::Map(...)` 合同与 `KERNEL_ASSIGN` req 语义（本实例 req=1 直写）保持。

Correctness contract 不可破坏项：gelu_erf_grad 公式（b/a + 0.5f·a·erf_grad(a/√2)/√2）、`b/a` 除零语义、NaN/±Inf/subnormal/exp overflow 行为、float 数据流（除 erf_grad 的 double 提升外）、`out[i]=input_1[i]·gelu_erf_grad(...)` 乘法顺序（FP 舍入）、OpenMP 分片边界与 in-place/alias 合同。

限制/风险：向量 exp 的误差验证工作量与数值风险最大；`b/a` 与串行链的 FP 舍入顺序在向量化下需容差验证；小 N 退化需 tail 判据与短输入基准（kernel-conventions §3）；double 往返消除的容差验证（eliminate-unnecessary-precision-conversions 的前置语义合同）。

修复后预期 Profile signals：`11e0516: jalr <expf@plt>` 调用点与 `11e0536: fmul.s`（59.85%）、`11e052a: fmul.s`（15.91%）、`11e051e: fmul.s`（7.58%）、`11e053a: fsw`（10.61%）、`11e050a: fdiv.s`（1.52%）份额消失；`11e051a/11e0522/11e0526` 的 fcvt.d.s/fmul.d/fcvt.s.d 往返消失；出现 `vsetvli`/`vle32`/`vfdiv`/`vfmul`/向量 exp/`vse32`；函数局部 cycles/element 改善（方向性）。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcv_..._zve64d_zvfh...`（RVV 1.0，C920v2/SG2044，OoO）；build ISA = `rv64i2p1_..._v1p0_..._zvl128b1p0`（含 `v`、`zvl128b`）；VLEN = 128 bits；bound type = 函数内 helper-call/FP-latency 主导（全局 IPC=0.562、LLC miss 43.99% 竞争性约束）。
4. **收益上界**：当前 sampled event（cpu-clock）下的函数内局部样本份额 ≈ **100.0%**（132 样本 local period）；`baseline_gap: sampling metadata` → 禁止 workload 级 Amdahl 上界。
5. **三维路由判定**：`current source` = compiler-generated C++ 模板（OpenMP clone）；`implementation existence/reachability` = mxnet 无 gelu 的既有 RVV kernel/dispatch slot → 不走 missing `.S` 分支；`function-level policy` = `gelu_erf_grad` 是 `tunable` 算子（走 LaunchTuned OpenMP 分片），无要求独立 `.S` 的 policy → 修复载体为普通代码/intrinsic 向量化。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。shape 要点：N 动态（运行期）→ `LMUL_min(W): N/A`，按 live set/unroll/吞吐选 LMUL（SEW=32, VLEN=128 → m1=4 lanes；live set ≈ 5-6 vectors → m1–m2 稳妥，m4 需 spill 检查）；tail 用 runtime-VL；`implementation_shape_gap: 无`。
7. **Related PRs 小节**：
   - `patterns/rvv_elementwise_activation_kernels.md`：Related PRs：10 条 URL（OpenCV `524d8ae01c63`、`86241653a781`、`#27072`；oneDNN `aa23ab557391`、`d5ae44880903`；llama.cpp `#17227`、`#15057`；MNN `#4508`、`#4484`、`#4044`、`b7268aa`、`#4359`）。
   - `patterns/no-vectorization.md`（supporting）：Related PRs：16 条 URL（OpenCV `#22179`、`#22520`、`#23980`、`#24058`、`#24132`、`#24166`、`#24301`、`#24325`、`#27160`、`#27119`、`#27097`、`#27007`、`#26958`、`#26865`、`b902a8e792e1`、`2c16f3b7d2b2`、`e06502a254f7`、`a2d784b6f53a`、`83104bed3209`）。
   - `patterns/eliminate_unnecessary_precision_conversions.md`（supporting）：Related PRs：4 条 URL（Go `#75463`、`2b50ab2aee75`；V8 `8981e6aae4f1`、`30f71750eac2`）。

## Phase 5 — Verification forecast / 验证预测：void mxnet::op::mxnet_op::Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>>::LaunchTuned<...>

（primary = RVV Elementwise Activation Kernels；supporting 不单独验证）

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`11e0536: fmul.s fs0,fs0,fs5`（59.85%）、`11e052a: fmul.s`（15.91%）、`11e051e: fmul.s`（7.58%）、`11e053a: fsw`（10.61%）、`11e050a: fdiv.s`（1.52%）、`11e0516: jalr <expf@plt>` 与 `11e051a/11e0522/11e0526` 的 fcvt.d.s/fmul.d/fcvt.s.d 往返应消失或大幅缩小。
- **应出现侧**（锚定 `patterns/rvv_elementwise_activation_kernels.md` §Verification）：annotate 出现 vector compare/mask/select 与 vector-math 序列（`vsetvli`/`vle32`/`vfdiv`/`vfmul`/向量 exp/`vse32`）；"scalar branch/math-helper share 下降"；cycles/element 改善。
- **正确性/数值合同**：`b/a` 除零（a==0 → ±Inf/NaN）语义、±0/NaN/±Inf/subnormal、exp overflow（大 |t|）行为、double 往返消除后的容差比对（向量 float 路径 vs `2.0` double reference）、N=0/小 N/step 整倍数/全 tail 长度（kernel-conventions §3）。
- **收益顺序**：单一顶层 primary；幅度待补采。
- **补采升级**：至少需 (1) `precise_ip`/Exact-IP 确认；(2) `--percent-type=global-period` 重采量化 global 份额；(3) 向量 exp 实现就绪后重跑 annotate 并 benchmark 短/中/长 N。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`Kernel<op_with_req<backward_grad_tuned<gelu_erf_grad>, 1>>::LaunchTuned [._omp_fn.0]` |
| 2 | Phase 1 输出要求 | ✅ | 7 行 baseline；gap：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行 |
| 3 | Phase 3 输出要求 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-codegen.md`；顶层 finding 1（primary=activation）+ supporting 2（no-vectorization、eliminate-precision-conversions）；锚点 `59.85 : 11e0536: fmul.s`、`0.00 : 11e0516: jalr <expf@plt>`、`0.00 : 11e051a: fcvt.d.s`、`10.61 : 11e053a: fsw`；supporting 2；排除条数 ≥7；推导式 2 组 |
| 4 | Phase 4 输出要求 | ✅ | 已读 pattern：`patterns/rvv_elementwise_activation_kernels.md`（命中 row=Elementwise Activation；引用 `Treat transcendental activations as vector-math pipelines`）、`patterns/no-vectorization.md`（supporting；`zero v*`）、`patterns/eliminate_unnecessary_precision_conversions.md`（supporting；`Absorb float32↔float64 round-trips`）；The fix before/after/correctness/风险/Profile signals 齐备；`Related PRs：10 条 URL`（activation）+ `Related PRs：16 条 URL`（no-vectorization）+ `Related PRs：4 条 URL`（precision-conversions） |
| 5 | 路径合规 | ✅ | 模式 A；primary=activation（L1）→ supporting=no-vectorization（L1）+ eliminate-precision-conversions（L4，被上层向量化覆盖而降级）；leaf 均来自已过 gate 的 row；动态份额仅函数内局部（100.0%）并显式 `baseline_gap: sampling metadata` |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 `11e0536: fmul.s`/`11e0516: jalr <expf@plt>`/`11e053a: fsw`；出现侧 `rvv_elementwise_activation_kernels.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；无追问；交付止于证据、根因蓝图、完整 The fix、验证预测 |

修正记录：无