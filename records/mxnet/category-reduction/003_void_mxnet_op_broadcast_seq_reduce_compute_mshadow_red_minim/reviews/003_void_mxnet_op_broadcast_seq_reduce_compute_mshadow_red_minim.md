# [Performance][RISC-V] Vectorize float32 minimum reductions with RVV

## Motivation

Single-axis minimum reductions over long rows are a common tensor operation. The current sequential CPU reduction processes those rows one element at a time, which limits throughput on RVV hardware. A specialized vector reduction improves this case while preserving the general reduction machinery for more complex shapes and operators.

## Implementation

- Add an RVV specialization for float32 minimum reductions.
- Enable the fast path only for a single reduced axis, identity mapping, no index output, and sufficiently long rows.
- Support non-unit input strides with vector strided loads.
- Preserve NaN handling and select the first matching minimum when equal values are present, including signed-zero cases.
- Keep the existing sequential reduction kernel as the fallback for all other reductions.

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
operator: mx.nd.min
input:    data=(4096, 1024), axis=1
metric:   avg_time_forward_min (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 3.9407 ms | 3.9122 ms | 3.9528 ms | 3.9407 ms |
| After `git apply patch-opt.diff` | 2.0157 ms | 2.0267 ms | 2.0153 ms | 2.0157 ms |

The median latency is reduced by **48.85%**, corresponding to a **1.96x** speedup. The patched run-to-run range was 0.57% of its median.
