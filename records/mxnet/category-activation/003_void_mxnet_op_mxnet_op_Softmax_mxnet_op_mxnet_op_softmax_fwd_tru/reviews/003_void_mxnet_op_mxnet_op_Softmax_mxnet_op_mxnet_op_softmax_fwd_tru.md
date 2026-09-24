# [Performance][RISC-V] Add an RVV softmin forward path

## Motivation

Softmin over a large contiguous last axis performs the same reductions and exponentiation as softmax after negating the input. The generic CPU path does not use RVV efficiently for this workload, so the forward pass remains dominated by scalar exponential and reduction code. This change adds a dedicated vector path for the common float32 case.

## Implementation

- Detect RVV compiler support in CMake, preferring `rv64gcv_zfhmin_zvfhmin` and falling back to `rv64gcv`.
- Add a contiguous float32 fast path for forward softmin with `temperature == 1` and no length mask.
- Run three vector passes over each negated input row: maximum reduction, exponentiation and sum reduction, then normalization.
- Use range reduction and a seventh-degree polynomial for vectorized exponentiation.
- Reduce a non-VLMAX remainder separately with a scalar seed so that tail-agnostic lanes cannot enter the final reduction.
- Retain the existing CPU path for unsupported layouts, data types, and parameter combinations. Float output is always supported by the fast path; half output is enabled when the required vector half-precision support is available.

## Testing

The patch was tested against commit `b84609d3fc73d20929c114eab95faaa56e6c5ede` on a RISC-V host with eight SpacemiT A100 cores (CPUs 0-7), GCC 14.3.0, and Linux 6.18.3.

Apply and build:

```bash
git apply patch-opt.diff
make -j8
```

The full build completed successfully. Functional operator tests were not run as part of this performance-validation pass; the results below are performance measurements, not a numerical-correctness claim.

The benchmark used MXNet's `benchmark.opperf.utils.benchmark_utils.run_performance_test` with the native profiler, float32 CPU tensors, 50 warm-up iterations, and 300 measured iterations. Each result below is from an independent invocation with `OMP_NUM_THREADS=8`, `OMP_DYNAMIC=FALSE`, and `taskset -c 0-7`.

```text
operator: mx.nd.softmin
input:    data=(1024, 1024), axis=-1
metric:   avg_time_forward_softmin (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 10.5023 ms | 10.4555 ms | 10.4310 ms | 10.4555 ms |
| After `git apply patch-opt.diff` | 2.8986 ms | 2.8773 ms | 2.8593 ms | 2.8773 ms |

The median latency is reduced by **72.48%**, corresponding to a **3.63x** speedup. The patched run-to-run range was 1.37% of its median.
