# [Performance][RISC-V] Vectorize nearest-neighbor upsampling with RVV

## Motivation

Nearest-neighbor upsampling spends most of its time expanding input rows into a larger output tensor. The generic expression-based CPU path performs this work element by element, which leaves substantial vector throughput unused on RISC-V systems. A row-oriented RVV implementation reduces that overhead for contiguous float32 NCHW tensors.

## Implementation

- Add an RVV helper for contiguous float32 nearest-neighbor upsampling.
- Specialize the common scale-2 case: load source elements in vector chunks and use `vrgather` with duplicated indices to produce two output values per input value.
- Handle other integer scale factors with precomputed byte offsets and indexed vector loads.
- Parallelize independent output rows with OpenMP.
- Dispatch to the RVV path only for contiguous, supported tensors using `kWriteTo`; retain the existing expression implementation for other requests.

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
operator:    mx.nd.UpSampling
input:       args=(32, 3, 256, 256)
parameters:  sample_type=nearest, scale=2
metric:      avg_time_forward_UpSampling (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 10.0070 ms | 9.8817 ms | 9.8482 ms | 9.8817 ms |
| After `git apply patch-opt.diff` | 5.7245 ms | 5.7273 ms | 5.7091 ms | 5.7245 ms |

The median latency is reduced by **42.07%**, corresponding to a **1.73x** speedup. The patched run-to-run range was 0.32% of its median.
