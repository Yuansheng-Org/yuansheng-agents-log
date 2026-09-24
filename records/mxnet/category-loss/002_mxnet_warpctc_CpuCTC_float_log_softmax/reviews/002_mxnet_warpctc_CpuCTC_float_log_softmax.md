# [Performance][RISC-V] Vectorize the CPU CTC log-softmax stage with RVV

## Motivation

The CPU CTC loss implementation computes a log-softmax over every time-step and batch row before running the CTC dynamic program. On RISC-V, the scalar float implementation makes this preprocessing stage a significant part of the forward-pass cost when the alphabet is large. Vectorizing the row reductions and exponentiation improves that hot path without changing the CTC algorithm itself.

## Implementation

- Add an RVV specialization for `CpuCTC<float>::log_softmax`.
- Reduce each row to its maximum with RVV, evaluate the exponentials with range reduction and a sixth-degree polynomial, and reduce the resulting sum.
- Process full vector chunks and the final dynamic-length tail without padding the input.
- Keep the existing scalar implementation for probability types other than float.

## Testing

The patch was tested against commit `b84609d3fc73d20929c114eab95faaa56e6c5ede` on a RISC-V host with eight SpacemiT A100 cores (CPUs 0-7), GCC 14.3.0, and Linux 6.18.3.

Apply and build:

```bash
git apply patch-opt.diff
make -j8
```

The full build completed successfully. Functional operator tests were not run as part of this performance-validation pass.

The benchmark used MXNet's `benchmark.opperf.utils.benchmark_utils.run_performance_test` with the native profiler, float32 CPU tensors, 50 warm-up iterations, and 300 measured iterations. Valid integer labels were prepared once for each process. Each result below is from an independent invocation with `OMP_NUM_THREADS=8`, `OMP_DYNAMIC=FALSE`, and `taskset -c 0-7`.

```text
operator: mx.nd.CTCLoss
inputs:   data=(100, 32, 128), label=(32, 20)
metric:   avg_time_forward_CTCLoss (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 9.0618 ms | 9.0297 ms | 8.9809 ms | 9.0297 ms |
| After `git apply patch-opt.diff` | 5.8239 ms | 5.8282 ms | 5.8537 ms | 5.8282 ms |

The median latency is reduced by **35.46%**, corresponding to a **1.55x** speedup. The patched run-to-run range was 0.51% of its median.
