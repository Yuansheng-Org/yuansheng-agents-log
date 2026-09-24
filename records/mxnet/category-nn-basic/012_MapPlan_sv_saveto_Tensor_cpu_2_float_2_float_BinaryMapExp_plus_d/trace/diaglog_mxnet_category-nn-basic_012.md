Functions under analysis: [`mshadow::MapPlan<mshadow::sv::saveto, mshadow::Tensor<mshadow::cpu, 2, float>, 2, float, mshadow::expr::BinaryMapExp<mshadow::op::plus, BinaryMapExp<div, BinaryMapExp<mul, ...>, UnaryMapExp<mxnet::op::mshadow_op::square_root, ...>>, BinaryMapExp<minus, ...>>>::operator() [clone ._omp_fn.0]`]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`012-...MapPlan...square_root...-annotate.txt`，266 行，5115 samples，含 hot loop body `11b41ea`–`11b424a`）
- perf stat（可选 bound/context）：已提供（testcase 级，非函数级）
- workload/binary/DSO/source context：已提供（`libmxnet.so`，DWARF 行号映射至 `src/operator/mshadow_op.h:908`、`src/operator/math_functions-inl.h:42-55,91`、`3rdparty/mshadow/mshadow/expr_eval.h`、`3rdparty/mshadow/mshadow/expr_scalar-inl.h:654-658,684-689`）
- readelf -A：已提供（metadata `binaries` → `..._v1p0_..._zve32f1p0_zve64d1p0_zve64f1p0_zvl128b1p0`）
- hardware ISA：已提供（SG2044 profile：RVV 1.0 + `zve32f/zve64f/zve64d/zvfh/zfa/zicond`）
- `vlenb`：已提供（vlenb=16 → VLEN=128）
- 采样元数据：已提供（`cpu-clock`，`percent: local period`，单次运行窗口）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0；`v` 存在；含 `zve32f`（f32 向量算术足够）（出处：sg2044 hw profile） |
| Build ISA | `Tag_RISCV_arch = rv64i2p1_..._v1p0_..._zve32f1p0_zvl128b1p0`（出处：metadata `binaries`） |
| Vector flavor | annotate 内零 `v*`、零 `th.v*`（全 scalar），无 flavor mismatch |
| VLEN | VLEN=128 bits（vlenb=16） |
| Bound type | 函数级 bound type 缺失 → `baseline_gap: bound type`；testcase 级 IPC=0.567926、L1 miss 0.974%、LLC miss 22.822%、branch miss 0.960% |
| Sampling semantics | event=`cpu-clock` ✓；percent type=**local period**（✗）；同一窗口 ✓；函数级 workload 贡献未知 ✗ → `baseline_gap: sampling metadata` |
| Sampling IP precision | `precise_ip`/Exact-IP 未知 → `baseline_gap: sampling IP precision`；`fsflags`/`flt.s`/`bnez`/`fsqrt.s` 的高占比只锚定同一 loop interval，不声明单条指令 cycle 成本 |

L0 gate ①：hardware 与 build 均有 `v` → 无 mismatch。L0 gate ②：无 `th.v*` → 不触发 flavor gate。Bound-type gate：`baseline_gap: bound type` + `baseline_gap: sampling metadata` → impact confidence 封顶 Medium。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致（1 个函数，L2Normalization 前向复合表达式 `plus(div(mul(...), sqrt(plus(...))), minus(...))` 的 MapPlan）。

hot loop 边界：`11b41ea` → `11b424a`（回边 `bne t1,a3,11b41ea`）= `expr_eval.h:177` 的 `for (index_t x = 0; x < shape[1]; ++x)`。

trace anchor（最高行原文）：
```
16.09 :   11b4218:        fsqrt.s fa0,fa0
14.80 :   11b4216:        bnez    a5,11b426a
```
Sampling IP precision 不允许单指令归因 → 只锚定 loop interval。

## Phase 3 — Pattern scan / 模式扫描：MapPlan<saveto, Tensor<cpu,2,float>, 2, float, BinaryMapExp<plus, div, mul, square_root, minus>>

### Class selection trace（8 项）
1. `include — rows-codegen.md`：hot FP interval 为保持 invalid-conversion/exception-flag 语义执行 FCSR save/restore（`frflags`/`fsflags`）+ branch diamond（`flt.s`/`bnez`），构成独立可验证信号。
2. `include — rows-operator-rvv.md`：compiler-generated scalar loop，逐元素求值复合 elementwise 表达式（mul/div/plus/minus + sqrt），sample 主导在 unit-stride 与广播索引访存、简单算术、store 与循环控制。
3. `exclude — rows-asm.md`：symbol 为 C++ 模板实例化的 OpenMP outline clone（`._omp_fn.0`），无手写 `.S` provenance；无 policy/existence 四证。
4. `exclude — rows-string-memory.md`：无 copy/fill/sentinel/two-input compare/checksum/back-reference 语义。
5. `exclude — rows-vectorized-tuning.md`：hot interval 零 `vsetvli`/`vle`/`vf*`，不是已向量化 RVV 循环。
6. `exclude — rows-offload.md`：SG2044/C920v2 无矩阵引擎或 P/DSP packed-SIMD 资源被使用；无权重重排/GEMM 分块语义。
7. `exclude — rows-crypto.md`：evidence 未点名任何密码学原语。
8. `exclude — rows-runtime-os.md`：hot interval 在用户态算子代码，无 timer/ISR/CSR/PMP 证据。

`Classes scanned:` `rows-codegen.md`（27 rows 逐行评估）、`rows-operator-rvv.md`（20 rows 逐行评估）。

### Scan 表

| # | Row | Class | 关系 | 局部样本份额 |
|---|---|---|---|---|
| F1 | Floating-Point Semantic Lowering | rows-codegen | **primary** | 42.76% |
| F2 | RVV Contiguous Elementwise Arithmetic Kernels | rows-operator-rvv | **independent** | 41.12% |
| — | RVV Precision Conversion Kernels | rows-operator-rvv | excluded（互斥） | — |
| — | Eliminate Unnecessary Float32/Float64 Conversions | rows-codegen | excluded（互斥） | — |
| — | RISC-V ISA Extension-Specific Instruction Substitution | rows-codegen | excluded（互斥） | — |
| — | RVV Normalization Kernels | rows-operator-rvv | excluded（互斥） | — |

#### F1 — Floating-Point Semantic Lowering（primary）

**(a) 逐字 evidence 引用**（loop interval `11b420a`–`11b4218`，位于 `x` 循环体内）：
```
0.00 :   11b420a:        frflags a2
14.41 :   11b420e:        flt.s   a5,fa0,fa2
13.55 :   11b4212:        fsflags a2
14.80 :   11b4216:        bnez    a5,11b426a
16.09 :   11b4218:        fsqrt.s fa0,fa0
```
DWARF 归属：`math_functions-inl.h:91` 的 `MXNET_UNARY_MATH_FUNC(sqrt)`（其 `float name(float a) { return ::name##f(a); }` 在 `math_functions-inl.h:44-46`）被 `mshadow_op.h:908` 的 `MXNET_UNARY_MATH_OP(square_root, math::sqrt(a))` 引用。慢路径确认：`bnez` 目标 `11b426a` 保存 8 个活跃寄存器后 `jalr → 59cce0 <sqrtf@plt>`（`11b428e`），返回后恢复并 `j 11b421c` 汇合——即「负输入 → 调 libm `sqrtf`（置 errno）；非负 → 用 `fsqrt.s`」的 errno 保持型 lowering。

**(b) 互斥邻居排除**
- **非 Eliminate Unnecessary Float32/Float64 Conversions**：该 row 认领完全不必要的 f32↔f64 往返；本 evidence 的 `flt.s`/`fsflags` 是 FCSR 状态保存与比较，不含任何精度转换指令。
- **非 RVV Precision Conversion Kernels**：该 row 认领标量 hot path 的主要工作为 FP16/BF16/FP32 或 float↔int 转换；本 evidence 无转换指令，`fsqrt.s` 是 FP 算术而非转换。
- **非 RISC-V ISA Extension-Specific Instruction Substitution**：该 row 认领纯 Zfa/Zfh native instruction 替换；本 evidence 的 `fsqrt.s` 已是 native 指令，剩余开销在 FCSR 保存/恢复与 branch diamond，不是指令替换问题。
- **非 RVV Normalization Kernels**：该 row 认领 Softmax/LayerNorm/RMSNorm 等完整算子流程；本函数是 L2Normalization 前向表达式，但 evidence 主导机制是 sqrt 的语义 lowering，不是归约与逐元素变换的耦合结构。

**(c) 双 Confidence 推导式**
`route: 直接 provenance（DWARF 映射到 math_functions-inl.h:44-46,91 与 mshadow_op.h:908，慢路径 jalr 59cce0 <sqrtf@plt> 逐字确认 errno 语义）+ 语义合同（invalid-conversion/exception-flag 保持）+ 互斥排除（3 项均以具体指令观察判别）→ High`
`impact: 缺 global-period 采样语义、缺函数级 bound type、Sampling IP precision 未知 → Medium（sample share 42.76% 与 VLEN=128 已到位）`

#### F2 — RVV Contiguous Elementwise Arithmetic Kernels（independent）

**(a) 逐字 evidence 引用**（同一 loop interval，表达式求值区段）：
```
14.27 :   11b423a:        slli    a5,a5,0x2
10.36 :   11b423e:        flw     fa4,0(a5)
4.91 :   11b4242:        fadd.s  fa5,fa4,fa5
3.13 :   11b4246:        fsw     fa5,-4(a1)
2.13 :   11b421e:        ld      a7,144(a4)
1.51 :   11b422e:        fdiv.s  fa5,fa5,fa0
1.15 :   11b4224:        div     a5,s2,a5
1.06 :   11b41fe:        flw     fa3,88(a4)
```
DWARF 归属：`expr_scalar-inl.h:654-658`（`op::div::Map` → `fdiv.s`）、`:637`（`op::plus::Map` → `fadd.s`）、`:629`（`op::mul::Map`）、`:684-689`（`sv::saveto::Save` → `fsw`）；广播索引来自 `expr_eval.h:146`（`src_.Eval(0, (y / ystride_) % length_)` → `div a5,s2,a5`）与 `:161-166`（`Broadcast1DExp::Eval` → `return src_.Eval(0, x)`），`slli a5,a5,0x2` 为该广播源元素的 `*4` 地址 scaling。

**(b) 互斥邻居排除**
- **非 RVV Normalization Kernels**：该 row 的互斥条款把「跨元素 accumulator」划归 reduction row；本 evidence 无跨元素归约（`fadd.s fa5,fa4,fa5` 的 `fa5` 来自同 lane 的 `fmul.s`，`fa4` 来自广播源的同列元素），是纯逐元素表达式。
- **非 RVV Indexed Gather for Table Lookup and Data-Dependent Access**：`flw fa4,0(a5)` 的地址由 `div`/`rem` 与 `slli` 从 `x` 线性推导，与数据值无关。
- **非 RVV Layout and Channel Packing Kernels**：无 deinterleave/packing/跨 lane 置换，`Broadcast1DExp::Eval` 只是同列重复读取。
- **非 Floating-Point Semantic Lowering**：F1 只认领 FCSR/branch-diamond 的 4 条指令；本条只认领表达式求值与广播索引的指令组，两者指令集不重叠。

**(c) 双 Confidence 推导式**
`route: 直接 provenance（DWARF 映射到 expr_scalar-inl.h 与 expr_eval.h）+ 语义合同（逐元素、同 lane 依赖、无跨元素归约）+ 互斥排除（3 项均以具体指令观察判别）→ High`
`impact: 缺 global-period 采样语义、缺函数级 bound type → Medium（share 41.12% 与 VLEN=128 已到位）`

### 多候选仲裁小段
F1 与 F2 可分账：F1 = `flt.s`(14.41) + `fsflags`(13.55) + `bnez`(14.80) = **42.76%**；F2 = `slli a5,a5,0x2`(14.27) + `flw fa4,0(a5)`(10.36) + `fadd.s`(4.91) + `fsw fa5,-4(a1)`(3.13) + `ld a7,144(a4)`(2.13) + `fdiv.s`(1.51) + `flw fa4,0(t5)`(1.33) + `ld a0,128(a4)`(1.27) + `div a5,s2,a5`(1.15) + `flw fa3,88(a4)`(1.06) = **41.12%**。另有 `fsqrt.s`(16.09%) 为必要工作，不属任何可删 finding。指令组不重叠、修复对象不同（F1 删 FCSR diamond；F2 向量化逐元素求值）→ **primary (F1) / independent (F2)**。

L0–L4 归属：F1 属 **L3（FP 语义 lowering / 编译配置层）**，F2 属 **L1（数据并行层）**。入口条件 A 按 evidence sample share 排序：**F1 (42.76%) > F2 (41.12%)**，与 primary 归属一致。

**交互说明**：F1 的修复（消除 per-element FCSR 保存/比较/恢复与 errno 分支）**不依赖 RVV**，是纯构建/源码层面的低风险改动；F2 的修复（把 `x` 循环向量化）需要在 F1 之后才有意义，因为 F1 的分支菱形会阻断向量化。故实施顺序应为 F1 → F2。

## Phase 4 — Root-cause blueprint / 根因蓝图：MapPlan<saveto, Tensor<cpu,2,float>, 2, float, BinaryMapExp<plus, div, mul, square_root, minus>> [._omp_fn.0]

### F1（primary）— Floating-Point Semantic Lowering

**Root cause**：`mxnet::op::math::sqrt`（`math_functions-inl.h:91` 的 `MXNET_UNARY_MATH_FUNC(sqrt)`，`float` 重载在 `:44-46` 直接转发 `::sqrtf`）在**未启用 `-fno-math-errno`** 的构建下，必须保持 C 标准的 `errno` 语义，于是每个元素都要：保存 FCSR（`frflags a2`）→ 做比较（`flt.s a5,fa0,fa2`，即 `a < 0`）→ 恢复 FCSR（`fsflags a2`）→ 条件分支（`bnez a5,11b426a`）到 libm `sqrtf@plt` 慢路径；只有非负输入才走 `fsqrt.s`。依据 `patterns/floating_point_semantic_lowering.md §Why this is slow`：「FCSR 访问可能序列化浮点流水线，helper call 和 NaN/rounding branch 会增加控制流与保存恢复开销」，而该文件同时给出正确做法——「优化必须从语义等价证明出发：只删除目标硬件已经保证的工作，或把少数特殊值留给明确 slow path」。

**The fix / 修复方式**（与 `patterns/floating_point_semantic_lowering.md §The fix` 一致；代表性形态，非可直接套用的补丁）：

修复前（现状）：
```cpp
// src/operator/math_functions-inl.h:42-46
#define MXNET_UNARY_MATH_FUNC(name)                       \
  MSHADOW_XINLINE float name(float a) {                   \
    return ::name##f(a);   /* 未加 -fno-math-errno → errno 保持型 lowering */ \
  }
// src/operator/mshadow_op.h:908
MXNET_UNARY_MATH_OP(square_root, math::sqrt(a));
// 每元素生成: frflags / flt.s / fsflags / bnez → sqrtf@plt | fsqrt.s
```

修复后（分离常见数值范围与异常结果，消除 per-element FCSR 往返与 helper 分支）：
```cpp
// 方式 1（构建层，首选）：为该 TU/目标加 -fno-math-errno
//   编译器可直接生成 fsqrt.s，不再为 errno 语义插入 FCSR save/compare/restore
//
// 方式 2（源码层）：把 sqrt 明确落到硬件指令语义，并把异常输入交给显式 slow path
MSHADOW_XINLINE float sqrt(float a) {
  // IEEE-754 / RISC-V 语义：fsqrt.s 对负有限输入产生 canonical qNaN 并置 NV flag，
  // 与参考实现（sqrtf 返回 NaN）在数值上一致；errno 不在算子语义合同内
  return __builtin_sqrtf(a);
}
// 若确需 errno 兼容：只在 (a < 0) 的冷路径调用 ::sqrtf，热路径保持 fsqrt.s 且不触碰 FCSR
```
适用前提与 correctness contract（不可破坏）：
1. **errno 依赖必须先行确认**：必须证明 mxnet 算子路径在 `square_root` 之后不读取 `errno`；这是删除 errno 语义的唯一合法性来源。
2. **NaN 语义**：`fsqrt.s` 对负有限输入返回 canonical qNaN 并置 NV flag；`sqrtf` 返回 NaN 并置 `errno=EDOM`。若参考实现依赖 NaN payload 传播或依赖 NV flag 的可观察状态，需保留显式 slow path。
3. **`-0.0` / `+inf` / NaN 输入**：`fsqrt.s(-0.0) = -0.0`、`fsqrt.s(+inf) = +inf`、NaN 输入产生 qNaN —— 与 IEEE-754 一致，须在回归中覆盖。
4. **不得用 `-ffast-math` 之类会改变 NaN/Inf 或重结合语义的开关**替换本项修复。
5. 限制/风险：该修复只消除 FCSR diamond（42.76%），`fsqrt.s`（16.09%）本身是必要工作；剩余收益需 F2 的向量化承接。

修复后预期 Profile signals：`11b420a: frflags a2`、`11b420e: flt.s a5,fa0,fa2`、`11b4212: fsflags a2`、`11b4216: bnez a5,11b426a` 从循环体消失，`11b428e: jalr 59cce0 <sqrtf@plt>` 慢路径不再从热路径可达。

**Baseline facts 回填**：hardware ISA = RVV 1.0 + `zve32f`；build ISA = `..._v1p0_..._zve32f1p0_zvl128b1p0`；VLEN = 128 bits；bound type = `baseline_gap: bound type`。

**收益上界**：**42.76%**（`cpu-clock`/local-period 局部样本份额）。因 `baseline_gap: sampling metadata`，不得表述为 workload 级 Amdahl 上界。

**三维路由判定**：
- `current source`：compiler-generated scalar code（DWARF 映射到 `math_functions-inl.h:44-46,91` 与 `mshadow_op.h:908`）→ 非手写 `.S`。
- `implementation existence/reachability`：无 RVV 实现存在；DSO 已按含 `v` 的 ISA 构建，无 dispatch gate 问题。
- `function-level policy`：无 policy 要求该算子由独立 `.S` 承载；不走 missing-`.S` 分支。

**Related PRs（Floating-Point Semantic Lowering）**：6 条
https://github.com/v8/v8/commit/9256d2ff895ba2f8cbd978e60030f8b08bea6162 ・ https://github.com/v8/v8/commit/f3e199b1b6be7c7724defbed48a8f0d72e973b46 ・ https://github.com/v8/v8/commit/ec0a91140c0df826932a740205595d038e8469f9 ・ https://github.com/v8/v8/commit/9656c3fbe244ce15e26e1bf1788caade0b805d3c ・ https://github.com/v8/v8/commit/30ec825beb1be3efdd06aa52f8046cdb0a2ed74c ・ https://github.com/v8/v8/commit/c39d21639f3dbbdacf39e9fd0ffe5624c54a3f15

### F2（independent）— RVV Contiguous Elementwise Arithmetic Kernels

**Root cause**：`MapPlan::operator()` 在 `expr_eval.h:177` 的 `for (index_t x = 0; x < shape[1]; ++x)` 内逐元素求值整棵表达式树：每元素重新做广播索引计算（`expr_eval.h:146` 的 `(y / ystride_) % length_` → `div a5,s2,a5` + 两次 `rem`）、`slli a5,a5,0x2` 地址 scaling（14.27%）、从 Plan 对象成员重新读取（`ld a7,144(a4)`、`ld a0,128(a4)`、`flw fa3,88(a4)`）、逐元素 `flw`/`fadd.s`/`fdiv.s`/`fsw`。依据 `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md §Why this is slow`：①「Scalar per-element overhead / 标量逐元素开销」——「同一组指针更新、边界判断和回跳分支需要按元素重复执行」，该文件指出 RVV「可以在一次迭代中处理当前 `vl` 个元素，将固定控制开销分摊到多个结果上」；②「Narrow load/store instruction stream / 窄粒度访存指令流」——「标量实现为每个输入和输出元素分别发出 load/store」。输出区间 `fsw fa5,-4(a1)` 配合 `addi a1,a1,4`（`11b422c`）为 unit-stride 连续，满足适用前提。

**The fix / 修复方式**（代表性形态，非可直接套用的补丁）：

修复前（现状）：
```cpp
// 3rdparty/mshadow/mshadow/expr_eval.h:177-180
for (index_t x = 0; x < shape[1]; ++x) {
  out[x] = Plan<Exp>::Eval(y, x);   // 逐元素：广播 div/rem + slli + flw + 算术 + fsw
}
```

修复后（把最后一维连续的 run 整体向量化；广播源的按列取值在 run 内保持标量提升或按 lane 展开）：
```cpp
// 快速路径前提：输出最后一维连续，且表达式各叶子在该 run 内为 unit-stride 或同列重复
index_t x = 0;
while (x < shape[1]) {
  const size_t vl = __riscv_vsetvl_e32m4(shape[1] - x);   // 动态 vl，覆盖主体与 tail
  vfloat32m4_t v = /* 逐 lane 构造表达式结果 */;
  __riscv_vse32_v_f32m4(out + x, v, vl);
  x += vl;
}
```
适用前提与 correctness contract：表达式树的每个叶子必须在 run 内可用 unit-stride `vle32` 或 broadcast（`vfmv.v.f` / 按列索引）表达；`plus/div/mul/minus` 逐元素算术可直接映射 `vfadd.vv`/`vfdiv.vv`/`vfmul.vv`/`vfsub.vv`；`square_root` 需在 F1 完成后才能进入向量体（否则 FCSR diamond 阻断）；`y` 方向的广播索引 `(y / ystride_) % length_` 必须在 run 外计算一次；tail 用运行时 `vl`，不写死 lane 数。限制/风险：`fdiv.s` 的向量形式 `vfdiv.vv` 吞吐低于标量流水化版本，需实测确认；若表达式含跨 lane 归约则不得套用本 row。预期 Profile signals：`11b423a: slli a5,a5,0x2`、`11b423e: flw fa4,0(a5)`、`11b4242: fadd.s`、`11b4246: fsw fa5,-4(a1)`、`11b422e: fdiv.s` 折叠为向量体。

**Baseline facts 回填**：同 F1（hardware/build ISA 含 `v`；VLEN=128；bound type = `baseline_gap: bound type`）。

**收益上界**：**41.12%**（local-period 局部样本份额；`baseline_gap: sampling metadata`，不得称 workload 级上界）。

**三维路由判定**：`current source` = compiler-generated scalar（DWARF 映射到 `expr_eval.h:177`、`expr_scalar-inl.h:629-658,684-689`）；`implementation existence/reachability` = 无 RVV 实现，DSO 已含 `v`，route 不冻结；`function-level policy` = 无。

**Related PRs（RVV Contiguous Elementwise Arithmetic Kernels）**：22 条
https://github.com/OpenMathLib/OpenBLAS/commit/45fd2d9b0790c5ca3698502d65d59d38d911ef4f ・ https://github.com/alibaba/MNN/pull/3913 ・ https://github.com/alibaba/MNN/commit/5376580ba19ac4034dc373f566fca846326fd612 ・ https://github.com/alibaba/MNN/pull/3779 ・ https://github.com/alibaba/MNN/commit/b35da10227477e90d5e5be55e1e5e646d4af41e4 ・ https://github.com/alibaba/MNN/commit/815ed5d6cb7052e2294d9eb6298e9e23b2fdf91d ・ https://github.com/uxlfoundation/oneDNN/commit/2b1dfe2d6233696bc2c803b48c06c00b29f7f866 ・ https://github.com/uxlfoundation/oneDNN/pull/5265 ・ https://github.com/uxlfoundation/oneDNN/commit/1184757c286814e552c137ca354060a30a9872bb ・ https://github.com/uxlfoundation/oneDNN/pull/5079 ・ https://github.com/uxlfoundation/oneDNN/commit/de1342a9d1dfe8419dd5159a16013529e567a6bc ・ https://github.com/uxlfoundation/oneDNN/commit/580b9c80484f5df175ad36870280703ea5767cbe ・ https://github.com/uxlfoundation/oneDNN/commit/a0961ab37e4ccf7dec0a0fd05fab92c1fc812e38 ・ https://github.com/uxlfoundation/oneDNN/commit/1147a0739a1fa1ee881075ac1cb8dd8f05e26cb5 ・ https://github.com/uxlfoundation/oneDNN/commit/595fc3b9bf46a5381d337af4ae0d7c529e2a9bcd ・ https://github.com/openjdk/jdk/commit/6700baa5052046f53eb1b04ed3205bbd8e9e9070 ・ https://github.com/openjdk/jdk/commit/885be2efa6b1359a7c7ab36882e19a7eaba77fb3 ・ https://github.com/openjdk/jdk/commit/9b61a7608efff13fc3685488f3f54a810ec0ac22 ・ https://github.com/v8/v8/commit/2b368def484809ad8d35b0c5d5f913bd95ad23ef ・ https://github.com/v8/v8/commit/56dd6a2f1ee28b2d37989a7888ad178a89f4f5ea ・ https://github.com/alibaba/MNN/pull/4042 ・ https://github.com/alibaba/MNN/commit/672c5862392393c171f1513bf7994d3b95e2a6a1

## Phase 5 — Verification forecast / 验证预测：MapPlan<saveto, Tensor<cpu,2,float>, 2, float, BinaryMapExp<plus, div, mul, square_root, minus>> [._omp_fn.0]

**F1（primary，先验证）**
- 应消失/缩小：`14.41 : 11b420e: flt.s a5,fa0,fa2`、`13.55 : 11b4212: fsflags a2`、`14.80 : 11b4216: bnez a5,11b426a`、`0.00 : 11b420a: frflags a2`；慢路径 `11b428e: jalr 59cce0 <sqrtf@plt>` 不再从热路径可达。
- 应出现（依据 `patterns/floating_point_semantic_lowering.md §Verification`）：热路径只剩 `fsqrt.s`（或 RVV `vfsqrt.v`），无 FCSR `frflags`/`fsflags` 往返、无 helper call、无 invalid-conversion branch diamond。
- 数值对照：对 `a > 0`、`a == 0`、`a == -0.0`、`a < 0`、`+inf`、NaN 输入分别比较 `sqrt` 结果，确认 NaN/±0/Inf 行为与参考实现一致；确认无代码路径读取 `errno`。

**F2（independent）**
- 应消失/缩小：`14.27 : 11b423a: slli a5,a5,0x2`、`10.36 : 11b423e: flw fa4,0(a5)`、`4.91 : 11b4242: fadd.s fa5,fa4,fa5`、`3.13 : 11b4246: fsw fa5,-4(a1)`、`1.51 : 11b422e: fdiv.s fa5,fa5,fa0`、`1.15 : 11b4224: div a5,s2,a5`。
- 应出现（依据 `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md §Verification`）：`vsetvli`、`vle32.v`、`vfadd.vv`/`vfmul.vv`/`vfdiv.vv`/`vfsub.vv`、`vse32.v`。
- 数值对照：覆盖 `shape[1] < VLMAX` 的 tail、`shape[1] == 0`、以及广播源列索引在 run 边界处的取值。

**验证顺序**：按 evidence sample share F1 (42.76%) → F2 (41.12%)。F2 的向量化依赖 F1 先行（FCSR diamond 阻断向量体）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`MapPlan<saveto, Tensor<cpu,2,float>, 2, float, BinaryMapExp<plus, div, mul, square_root, minus>> [._omp_fn.0]` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行结论 + gap 标签：`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；含 `Sampling IP precision` 行；两个 L0 gate 判定 + bound-type gate 结论 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-codegen.md, rows-operator-rvv.md`；顶层 finding 2 个（F1/F2）；F1 evidence 锚点 `14.41 : 11b420e: flt.s a5,fa0,fa2`、`13.55 : 11b4212: fsflags a2`，F2 锚点 `14.27 : 11b423a: slli a5,a5,0x2`、`10.36 : 11b423e: flw fa4,0(a5)`；supporting 0；排除 4 条；推导式 2 条 |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 `patterns/floating_point_semantic_lowering.md`（row: Floating-Point Semantic Lowering；引用首词「FCSR 访问可能序列化浮点流水线」「优化必须从语义等价证明出发」）+ `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md`（row: RVV Contiguous Elementwise Arithmetic Kernels；引用首词「Scalar per-element overhead」「Narrow load/store instruction stream」）；`The fix` 含 before/after、correctness（errno 依赖、NaN 语义、±0/Inf/NaN、禁用 -ffast-math）、风险、预期 Profile signals；Related PRs 6 条 + 22 条 |
| 5 | 路径合规 | ✅ | 模式 A（profile_backed）；路径 = 8 项 trace 可解释扫描集（include 2 / exclude 6）；primary/independent 关系与 L3/L1 归属合规；每个 leaf 均来自通过 gate 的 row；按动态份额排序 F1(42.76%)>F2(41.12%)；`th.v*` 未出现，未全局停扫 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧 `11b420e: flt.s a5,fa0,fa2`、`11b4212: fsflags a2`、`11b4216: bnez a5,11b426a`、`11b423a: slli a5,a5,0x2` 对上 Phase 3(a)；出现侧标注 `patterns/floating_point_semantic_lowering.md §Verification` 与 `patterns/rvv_contiguous_elementwise_arithmetic_kernels.md §Verification` |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成；无契约外追问；交付物止于 Profile 证据 / 根因蓝图 / 完整 `The fix` / 验证预测 |

修正记录：无
