# [Performance][RISC-V] Add an RVV log-softmax forward path

## Motivation

Log-softmax over a large contiguous last axis is dominated by row reductions and exponentiation. The generic CPU path does not generate an efficient RVV loop for this workload, leaving the RISC-V vector units underused. A dedicated float32 path substantially reduces the forward-pass latency while retaining the current implementation as a fallback.

## Implementation

- Detect RVV compiler support in CMake, preferring `rv64gcv_zfhmin_zvfhmin` and falling back to `rv64gcv`.
- Add a contiguous float32 fast path for forward log-softmax with `temperature == 1` and no length mask.
- Compute the row maximum, vectorized exponentials, and exponential sum with RVV; subtract the resulting log denominator in the output pass.
- Use range reduction and a polynomial approximation for the vector exponential.
- Pass the vector source and scalar seed to the RVV reduction intrinsics in the order required by the intrinsic API.
- Route unsupported layouts, data types, and parameter combinations through the existing CPU implementation.

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
operator: mx.nd.log_softmax
input:    data=(1024, 1024), axis=-1
metric:   avg_time_forward_log_softmax (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 9.9588 ms | 9.6534 ms | 9.6218 ms | 9.6534 ms |
| After `git apply patch-opt.diff` | 1.8245 ms | 1.8008 ms | 1.7956 ms | 1.8008 ms |

The median latency is reduced by **81.35%**, corresponding to a **5.36x** speedup. The patched run-to-run range was 1.60% of its median.
