# [Performance][RISC-V] Add an RVV softmax forward path

## Motivation

The generic CPU softmax implementation does not make effective use of the RISC-V Vector extension for large contiguous rows. The exponentiation and reduction stages therefore dominate the runtime on RVV systems. This change adds a dedicated path for the common float32, last-axis case while leaving the existing implementation in place for unsupported layouts and data types.

## Implementation

- Detect a usable RVV compiler target in CMake. The build prefers `rv64gcv_zfhmin_zvfhmin` and falls back to `rv64gcv` when the half-precision vector extensions are unavailable.
- Add a float32 RVV kernel for contiguous rows with `temperature == 1` and no length mask.
- Compute each row in three passes: maximum reduction, exponentiation and sum reduction, then normalization.
- Use range reduction and a sixth-degree polynomial for vectorized exponentiation.
- Keep the existing CPU expression path as the fallback for all unsupported cases. Float output is always supported by the fast path; half output is enabled when the required vector half-precision support is available.

## Testing

The patch was tested against commit `b84609d3fc73d20929c114eab95faaa56e6c5ede` on a RISC-V host with eight SpacemiT A100 cores (CPUs 0-7), GCC 14.3.0, and Linux 6.18.3.

Apply and build:

```bash
git apply patch-opt.diff
make -j8
```

The full build completed successfully. Functional operator tests were not run as part of this performance-validation pass.

The benchmark used MXNet's `benchmark.opperf.utils.benchmark_utils.run_performance_test` with the native profiler, float32 CPU tensors, 50 warm-up iterations, and 300 measured iterations. Each result below is from an independent invocation with `OMP_NUM_THREADS=8`, `OMP_DYNAMIC=FALSE`, and `taskset -c 0-7`.

```text
operator: mx.nd.softmax
input:    data=(1024, 1024), axis=-1
metric:   avg_time_forward_softmax (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 10.4387 ms | 10.5257 ms | 10.6410 ms | 10.5257 ms |
| After `git apply patch-opt.diff` | 2.9741 ms | 2.9668 ms | 2.9728 ms | 2.9728 ms |

The median latency is reduced by **71.76%**, corresponding to a **3.54x** speedup. All three patched runs were more than 1% faster than the baseline median.
