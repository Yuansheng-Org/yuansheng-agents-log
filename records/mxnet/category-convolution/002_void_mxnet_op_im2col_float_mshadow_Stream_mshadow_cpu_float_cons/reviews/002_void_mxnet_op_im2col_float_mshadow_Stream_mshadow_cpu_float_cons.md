# [Performance][RISC-V] Add an RVV fast path for CPU im2col

## Motivation

The CPU `im2col` loop checks image bounds and copies one element at a time for every kernel position. For convolution-sized tensors, the per-element branch and scalar strided loads account for a large share of the operator cost. The valid elements for a fixed input row and kernel column form a contiguous output interval, so the padding checks can be moved out of the inner loop and the copy can be vectorized.

## Implementation

- Compute the valid output-column interval once for each input row and kernel-column position.
- Fill the left and right padding spans with zero without performing a bounds check for every element.
- Add an RVV float32 row-copy helper. Unit-stride source rows use regular vector loads; dilated or strided rows use vector strided loads and contiguous stores.
- Preserve the scalar copy loop for non-float types and builds without RVV support.

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
operator:    mx.nd.im2col
input:       data=(8, 32, 128, 128)
parameters:  kernel=(3, 3), pad=(1, 1), stride=(1, 1), dilate=(1, 1)
metric:      avg_time_forward_im2col (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 54.5718 ms | 54.5018 ms | 54.5230 ms | 54.5230 ms |
| After `git apply patch-opt.diff` | 16.6754 ms | 16.7200 ms | 16.6896 ms | 16.6896 ms |

The median latency is reduced by **69.39%**, corresponding to a **3.27x** speedup. The patched run-to-run range was 0.27% of its median.
