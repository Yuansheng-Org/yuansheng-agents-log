# [Performance][RISC-V] Vectorize col2im accumulation in convolution backward

## Motivation

CPU convolution backward-data uses `col2im` to accumulate columns back into the input-gradient tensor. The inner loop performs a bounds check and scalar addition for every output column. For a fixed kernel position, the in-bounds columns form one contiguous interval, which allows the padding work to be skipped and the accumulation to be vectorized on RVV systems.

## Implementation

- Compute the valid output-column interval outside the innermost `col2im` loop.
- Skip padding positions in blocks while advancing the column buffer consistently.
- Add a float32 RVV accumulation helper for `dst += src`; use unit-stride loads and stores where possible and strided accesses when required by the destination layout.
- Keep the existing scalar accumulation path for non-float types and builds without RVV.
- Make the patch self-contained by adding the required headers and local RVV feature guard. It does not depend on the separate `im2col` optimization.

## Testing

The patch was tested against commit `b84609d3fc73d20929c114eab95faaa56e6c5ede` on a RISC-V host with eight SpacemiT A100 cores (CPUs 0-7), GCC 14.3.0, and Linux 6.18.3.

Apply and build:

```bash
git apply patch-opt.diff
make -j8
```

The full build completed successfully. Functional operator tests were not run as part of this performance-validation pass; the results below are performance measurements, not a numerical-correctness claim.

The benchmark used MXNet's `benchmark.opperf.utils.benchmark_utils.run_performance_test` with the native profiler, float32 CPU tensors, 50 warm-up iterations, and 300 measured iterations. Backward execution was enabled. Each result below is from an independent invocation with `OMP_NUM_THREADS=8`, `OMP_DYNAMIC=FALSE`, and `taskset -c 0-7`.

```text
operator:    mx.nd.Convolution (backward)
inputs:      data=(32, 3, 256, 256), weight=(1, 3, 3, 3), bias=(1,)
parameters:  kernel=(3, 3), num_filter=1, layout=NCHW,
             pad=(0, 0), stride=(1, 1), dilate=(1, 1)
metric:      avg_time_backward_Convolution (ms)
```

## Performance

Lower is better.

| Build | Run 1 | Run 2 | Run 3 | Median |
| --- | ---: | ---: | ---: | ---: |
| Baseline | 261.8669 ms | 261.7465 ms | 261.6667 ms | 261.7465 ms |
| After `git apply patch-opt.diff` | 194.4698 ms | 195.1131 ms | 192.8183 ms | 194.4698 ms |

The median latency is reduced by **25.70%**, corresponding to a **1.35x** speedup. The patched run-to-run range was 1.18% of its median.
