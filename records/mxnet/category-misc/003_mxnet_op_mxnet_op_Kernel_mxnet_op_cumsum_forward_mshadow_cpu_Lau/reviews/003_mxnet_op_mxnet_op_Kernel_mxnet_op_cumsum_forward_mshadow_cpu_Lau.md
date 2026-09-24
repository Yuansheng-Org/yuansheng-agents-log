# [Performance][RISC-V] Vectorize batched cumsum over the middle axis

## Motivation

For tensors with a sizeable trailing dimension, cumulative sum over a middle axis consists of many independent scans. The scalar CPU kernel walks each scan separately and leaves the independent trailing lanes unused. Processing those lanes together with RVV preserves the cumulative-sum order while increasing throughput substantially.

## Implementation

- Add an RVV float32 kernel for cumulative sum when the trailing dimension is greater than one.
- Vectorize across the independent trailing lanes and keep the recurrence along the selected axis in its original order.
- Process the trailing dimension in VLMAX-sized chunks with a dynamic final chunk.
- Restrict the specialization at compile time to CPU float32 input and output so that other type-switch combinations do not instantiate an incompatible kernel.
- Retain the existing scalar path for other data types, devices, and the single-lane case.

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
operator: mx.np.cumsum
input:    a=(256, 64, 256), axis=1
metric:   avg_time_forward_cumsum (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 28.0359 ms | 28.0858 ms | 27.9683 ms | 28.0359 ms |
| After `git apply patch-opt.diff` | 2.6338 ms | 2.6500 ms | 2.6433 ms | 2.6433 ms |

The median latency is reduced by **90.57%**, corresponding to a **10.61x** speedup. The patched run-to-run range was 0.61% of its median.
