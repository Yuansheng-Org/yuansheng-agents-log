# [RISC-V] Vectorize the ReLU backward kernel (`relu_grad`) with RVV

> **中文说明（提交前必读）**
>
> - 本文件是可直接粘贴到 GitHub PR 描述框的正文，目标仓库 **`apache/mxnet`**，目标分支 `master`。
> - 基线提交：`b84609d3fc73d20929c114eab95faaa56e6c5ede`（2023-01-26）。
> - **推荐提交的是 v2**：`mxnet_019_relu_grad_rvv_v2_shared_header.patch`
>   单文件 `src/operator/mshadow_op.h`，`+160/−0`，`git apply --check` 通过，`--whitespace=error-all` 干净。
> - v1（`mxnet_019_relu_grad_rvv.patch`，两文件 `+156/−0`）功能等价、也能编译，但把 `Kernel` 偏特化放在
>   `activation-inl.h`，存在跨 TU 的 ODR 违规（下文 *Why the specialization lives in mshadow_op.h* 有实测证据）。
>   **v2 已修掉该问题，且顺带让 `_backward_relu` 路径也吃到优化。**
> - 本文所有数字都标注了来源与测量条件；**端到端 OpPerf 数字尚未采集**，见 *Not verified*。

---

## Summary

Adds an RVV (RISC-V Vector 1.0) fast path for the float32 CPU implementation of the ReLU backward kernel `relu_grad`, i.e. `out[i] = lhs[i] * relu_grad(rhs[i])`. On RVV hardware this replaces the per-element scalar compares and branches (`feq.s` NaN test + `flt.s` sign test + `beqz`/`bnez`) with vector compare masks and `vmerge` selects. Non-RVV builds, and non-float32 dtypes on RVV builds, keep the existing scalar code path unchanged.

## Motivation

`relu_grad` is on the hot path of every model that uses ReLU: the gradient of `relu` is computed by the `_backward_relu` op and by the `Activation` op's backward. On RISC-V the existing implementation is fully scalar — `mxnet_op::backward_grad_tuned<mshadow_op::relu_grad>` is launched through `Kernel<op_with_req<...>, cpu>::LaunchTuned`, which runs `relu_grad::Map` once per element. That kernel is trivially lane-parallel (no reduction, no cross-lane dependency), so it is a good fit for RVV.

## What changed

One file, `+160/−0`: **`src/operator/mshadow_op.h`**.

1. Adds `#include <riscv_vector.h>` under the RVV guard.

2. Adds `namespace relu_grad_rvv` (right after `struct relu_grad`) with two `MSHADOW_XINLINE` helpers:
   - `relu_grad_block_f32m1<accumulate>(out, lhs, rhs, vl)` — one contiguous block, `e32m1`:
     ```
     x = vle32(rhs); g = vle32(lhs)
     gt  = vmfgt(x, 0.0f)                 // a > 0  -> 1, else 0
     m   = vmerge(0.0f, 1.0f, gt)
     nan = vmfne(x, x)                    // true exactly on NaN lanes
     m   = vmerge(m, x, nan)              // IsNan(a) -> a, payload preserved
     res = vfmul(g, m)
     if (accumulate) res = vfadd(res, vle32(out))
     vse32(out, res)
     ```
   - `relu_grad_apply_range<accumulate>(out, lhs, rhs, n)` — fixed-VL main loop plus a runtime-VL tail, so there is never an over-read or over-write.

3. Adds a partial specialization of `mxnet_op::Kernel` for
   `op_with_req<backward_grad_tuned<mshadow_op::relu_grad>, req>` on `mshadow::cpu`, placed at the end of `mshadow_op.h` inside an explicit `namespace mxnet { namespace op { namespace mxnet_op {` block:
   - an exact-match `Launch(Stream<cpu>*, size_t N, float*, float*, float*)` overload that runs the RVV kernel (OpenMP only when `N >= 4096`, mirroring the existing `UseOMP` workload threshold);
   - a variadic fallback overload reproducing the original `LaunchTuned` scalar behaviour for every other dtype/signature, including the `tuned_op<...>::UseOMP` decision.

All three additions are wrapped in `#if defined(__riscv) && defined(__riscv_v)`, so the patch is fully inert on x86/ARM/GPU builds and on RISC-V builds without the V extension.

### Why the specialization lives in `mshadow_op.h`

`backward_grad_tuned<mshadow_op::relu_grad>` is instantiated from **two** translation units:

- `src/operator/nn/activation.cc`, via `ActivationBackward` (`activation-inl.h:149`), and
- `src/operator/tensor/elemwise_unary_op_basic.cc`, which registers `_backward_relu` with
  `unary_bwd<mshadow_op::relu_grad>` (= `backward_grad_tuned<relu_grad>`, `elemwise_unary_op.h:527`)
  and launches it through `ElemwiseBinaryOp::Compute` → `Kernel<op_with_req<OP, Req>, cpu>::Launch`
  (`elemwise_binary_op.h:510`).

The second TU does **not** include `activation-inl.h`. If the specialization is declared there, the same class template specialization is defined differently in two translation units — a formal ODR violation (IFNDR). This is not hypothetical; it was reproduced on real objects:

| `elemwise_unary_op_basic.o`, `-O2 -march=rv64gcv` | specialization in `activation-inl.h` (v1) | specialization in `mshadow_op.h` (v2) |
|---|---|---|
| `Kernel<op_with_req<backward_grad_tuned<relu_grad>, 1>, cpu>::Launch(float*, float*, float*)` | **0** | **emitted** |
| generic `…>::LaunchTuned<…relu_grad…>` | used (62 symbols) | n/a |
| RVV kernel in the object | absent | present (see disassembly below) |

`mshadow_op.h` is the natural common home: it defines `relu_grad`, it already pulls in `mxnet_op.h` (it uses `mxnet_op::tunable`), and both instantiating TUs include it. Placing the specialization there makes every instantiation see the same definition **and** extends the optimization to the `_backward_relu` path.

## Correctness

The vector path is written to be **bit-identical** to the scalar one, not merely close:

- `a > 0` (including `+Inf`) → `1`; `a <= 0` (including `-0.0f` and `-Inf`) → `0`, matching the scalar `a > DType(0)` arm.
- NaN lanes return the input `x` unchanged via `vmerge` (payload preserved), matching `IsNan(a) -> a`.
- The `lhs * m` operand order and the resulting sign of zero are the same as the scalar `DType(a * GRAD_OP::Map(...))`.
- `kAddTo` accumulates in the same order within a lane; `kNullOp` returns early, matching `KERNEL_ASSIGN`'s no-op semantics.

Measured on real hardware: `max_abs_diff = 0.000e+00` over 4 M float32 elements — **bitwise identical** to the scalar path.

## Performance

**Environment**: SOPHGO SG2044 (XuanTie C920v2), 64 cores, RVV 1.0, **VLEN = 128** (`vlenb = 16`), GCC 15.1.0, flags `-O2 -march=rv64gcv -mabi=lp64d -std=gnu++17`.

**Method**: standalone harness including the real headers, `taskset -c 62` (single core pinned), 5 warm-up iterations, median of 25 runs, ABBA interleaving ×3 rounds. 16 MiB float32 tensors (4,194,304 elements), 2 loads + 1 store per element.

| Variant | Median | Throughput |
|---|---|---|
| scalar `relu_grad::Map` loop | 10.754 / 11.001 ms | 4.58 – 4.68 GB/s |
| RVV `relu_grad_apply_range` | 6.265 / 6.395 ms | 7.87 – 8.03 GB/s |

**Speed-up: 1.68× – 1.76×** (conservative figure 1.68×), with bit-identical output.

## Build / reachability evidence

Both instantiating translation units were compiled with the project's own flags and the resulting objects compared against unpatched objects. `nm`/`objdump` were used to confirm the fast path is actually reached rather than being dead code.

**`src/operator/nn/activation.cc`**

| | vector instructions | object size |
|---|---|---|
| baseline | 36 | 28,526,584 B |
| patched | **100** (+64) | 28,570,416 B (+43,832) |

**`src/operator/tensor/elemwise_unary_op_basic.cc`** (v2 only)

The patched `Launch` overload is emitted, and disassembling it shows the added kernel — fixed-VL main loop, runtime-VL tail, and the `kAddTo` variant:

```
f340: vsetvli a5,zero,e32,m1,ta,ma      # vsetvlmax_e32m1()
f37c: vle32.v  v1,(a2)                  # rhs
f380: vle32.v  v2,(a1)                  # lhs
f38c: vmfgt.vf v0,v1,fa5                # x > 0
f390: vmerge.vvm v3,v6,v5,v0            # select 0 / 1
f398: vmerge.vvm v3,v3,v1,v0            # NaN -> x
f39c: vfmul.vv v2,v2,v3                 # g * m
f3a0: vse32.v  v2,(a3)                  # store
f3b4: vsetvli a0,a0,e32,m1,ta,ma        # runtime VL (tail)
...
```

Symbol inspection on `activation.cc` also confirms the split is exactly as intended: `relu_grad` goes through the patched `Launch`, while `softrelu_grad` (untouched by this patch) still goes through the generic `LaunchTuned` path.

Both TUs compile with the project's real flags with 0 errors, and the patch applies cleanly to `master` (`git apply --check` OK, `--whitespace=error-all` clean).

## Not verified

- **End-to-end benchmark numbers.** Only kernel-level A/B and object-level reachability were measured. The `mxnet` OpPerf case `activation-relu` (`run_backward=True`) exercises this kernel, but the OpPerf run itself was **not** executed here, so no end-to-end speed-up figure is claimed.
- **VLEN sensitivity.** Only VLEN = 128 hardware was available. The code derives `vlmax` at runtime via `__riscv_vsetvlmax_e32m1()`, so it should be VLEN-agnostic, but VLEN = 256 was not measured.
- **Other dtypes.** Only float32 takes the vector path; float64 / float16 were not benchmarked (they intentionally keep the scalar path).
- **Sparse path.** `_backward_relu`'s sparse/`ComputeEx` path uses `MissingLValueOp<...>` rather than `op_with_req<...>`, so it is unaffected by this specialization and keeps the scalar path.

## Reviewer notes

1. **Guard macro.** The patch uses `__riscv_v`, which is only defined by newer toolchains (measured: `__riscv_v=1000000`, `__riscv_vector=1`, `__riscv_v_intrinsic=12000` on GCC 15.1.0 with `-march=rv64gcv`). On older GCC the guard is simply false and the optimization is silently skipped — no build breakage, but also no benefit. Happy to widen it to `#if defined(__riscv_v_intrinsic) && __riscv_v_intrinsic >= 10000` (the RVV intrinsics v1.0 marker) so GCC 13+ also gets the fast path; please tell me which convention you prefer.
2. **`-0.0f` sign.** The `a > 0` comparison maps `-0.0f` to `0`, and `lhs * 0.0f` keeps the sign of `lhs`, identical to the scalar code. Called out explicitly because this is the kind of detail that tends to differ between scalar and vector rewrites.
3. **OpenMP threshold.** The vector path reuses the `N >= 4096` cutoff as a proxy for the `tuned_op::UseOMP` workload threshold, to avoid OpenMP launch overhead on small tensors. If you would rather route this through the existing tuning machinery, I can restructure it.
