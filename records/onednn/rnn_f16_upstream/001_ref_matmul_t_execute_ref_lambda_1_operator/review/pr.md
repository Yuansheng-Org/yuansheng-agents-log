# CPU: RV64: support f32 bias in f16 and bf16 matmul JIT

## Description

This PR extends the RV64 half-precision matmul path to support an optional f32 1xN bias. Matmul problems with f16 or bf16 source, weights, and destination can now keep f32 accumulation in the RVV GEMM kernel, add the bias before narrowing, and avoid falling back to a reference implementation.

The supported bias form has f32 data type, matches the output N dimension, and broadcasts over output M and all batch dimensions. Other bias data types, scalar bias, per-M bias, full bias, and post-ops continue to use the existing fallback implementations.

## Implementation details

The RV64 f16/bf16 GEMM microkernel keeps output N in its f32 accumulator lanes and output M in its unrolled columns. The bias epilogue therefore loads one f32 vector along the accumulator lanes with `vle32.v` and adds it to every active accumulator column with `vfadd.vv` before narrowing the result to f16 or bf16.

The bias pointer follows the output-N tile offset through thread partitioning, M blocking, and M tails. The same 1xN vector is reused across output-M columns and batch entries, including the weights-broadcast form. The primitive descriptor accepts this path only when `is_bias_1xN()` is true, so unsupported matmul broadcast masks retain their existing behavior.

Biased and unbiased JIT kernel tables are instantiated independently and lazily. A primitive without bias does not construct or retain the additional bias kernels.

## Performance evaluation

The baseline is oneDNN commit `d4224705892d1bb1598c4fc1ce1632ebf493ddb9`. Baseline and patched measurements use the same Release/OpenMP `benchdnn` executable; only `libdnnl.so` changes. Threads are pinned to the tested CPU group with `OMP_PROC_BIND=close` and `OMP_PLACES=cores`. Lower time is better.

```bash
benchdnn --mode=P --fix-times-per-prb=20 -v1 --engine=cpu --matmul --batch=<batch>
```

The performance batches cover f32 1xN bias, 2D and batched matmul, weights-broadcast and per-batch weights, M/N/K tails, transposed weights, and both 1-thread and 8-thread execution. Only cases that select `jit:rvv` in the patched build are reported below. A transposed-source control remained on its existing implementation and is excluded from the optimization results.

### Muse Pi Pro / X60, VLEN=256, f16

| Case                  | Shape                   | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------------------- | ----------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| 2d_small_tail         | `7x31:31x13`            |         0.328125 |        0.007312 |        +97.77% |         0.358276 |        0.012585 |        +96.49% |
| 2d_both_tails         | `63x127:127x65`         |        51.704100 |        0.156140 |        +99.70% |         7.810420 |        0.042139 |        +99.46% |
| 2d_square             | `64x128:128x64`         |        51.686500 |        0.105432 |        +99.80% |        15.124400 |        0.028259 |        +99.81% |
| 2d_wide               | `128x256:256x512`       |      1707.890000 |        3.522130 |        +99.79% |       328.527000 |        0.524524 |        +99.84% |
| 2d_tall               | `512x256:256x128`       |      1682.090000 |        3.182360 |        +99.81% |       328.774000 |        0.487585 |        +99.85% |
| 2d_large_k            | `32x4096:4096x128`      |      1676.590000 |        7.347670 |        +99.56% |       323.587000 |        4.279850 |        +98.68% |
| 2d_large_n            | `256x512:512x1024`      |     16729.000000 |       27.674100 |        +99.83% |      2395.370000 |       10.336100 |        +99.57% |
| 2d_transposed_weights | `65x127:127x63`         |        50.938100 |        0.161084 |        +99.68% |        13.849900 |        0.038660 |        +99.72% |
| 3d_weights_broadcast  | `2x64x128:1x128x256`    |       506.483000 |        0.830396 |        +99.84% |       104.915000 |        0.410815 |        +99.61% |
| 3d_per_batch_weights  | `2x64x128:2x128x256`    |       505.235000 |        0.749194 |        +99.85% |       113.328000 |        0.406409 |        +99.64% |
| 3d_broadcast_tails    | `3x31x127:1x127x65`     |        89.302000 |        0.249634 |        +99.72% |        16.777200 |        0.062415 |        +99.63% |
| 3d_per_batch_tails    | `3x31x127:3x127x65`     |        90.758400 |        0.236182 |        +99.74% |        17.146700 |        0.108472 |        +99.37% |
| 4d_weights_broadcast  | `2x3x31x127:1x1x127x65` |       207.026000 |        0.416309 |        +99.80% |        43.351600 |        0.104236 |        +99.76% |
| 4d_per_batch_weights  | `2x3x31x127:2x3x127x65` |       212.050000 |        0.479077 |        +99.77% |        37.970900 |        0.106750 |        +99.72% |

### K3 / X100, VLEN=256, f16

| Case                  | Shape                   | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------------------- | ----------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| 2d_small_tail         | `7x31:31x13`            |         0.110217 |        0.003027 |        +97.25% |         0.027918 |        0.004956 |        +82.25% |
| 2d_both_tails         | `63x127:127x65`         |        17.826000 |        0.055505 |        +99.69% |         2.376680 |        0.013611 |        +99.43% |
| 2d_square             | `64x128:128x64`         |        17.974300 |        0.041235 |        +99.77% |         2.346530 |        0.009521 |        +99.59% |
| 2d_wide               | `128x256:256x512`       |       592.806000 |        1.110550 |        +99.81% |        97.584100 |        0.264258 |        +99.73% |
| 2d_tall               | `512x256:256x128`       |       590.264000 |        1.045870 |        +99.82% |        83.802900 |        0.163574 |        +99.80% |
| 2d_large_k            | `32x4096:4096x128`      |       574.557000 |        1.419860 |        +99.75% |        94.345900 |        0.733057 |        +99.22% |
| 2d_large_n            | `256x512:512x1024`      |      5971.500000 |        8.353920 |        +99.86% |      1605.440000 |        2.225890 |        +99.86% |
| 2d_transposed_weights | `65x127:127x63`         |        17.788900 |        0.046350 |        +99.74% |         2.384910 |        0.012769 |        +99.46% |
| 3d_weights_broadcast  | `2x64x128:1x128x256`    |       156.241000 |        0.282898 |        +99.82% |        23.339200 |        0.057471 |        +99.75% |
| 3d_per_batch_weights  | `2x64x128:2x128x256`    |       156.812000 |        0.314478 |        +99.80% |        22.216600 |        0.162085 |        +99.27% |
| 3d_broadcast_tails    | `3x31x127:1x127x65`     |        27.555900 |        0.078223 |        +99.72% |         3.795280 |        0.023144 |        +99.39% |
| 3d_per_batch_tails    | `3x31x127:3x127x65`     |        27.649100 |        0.094788 |        +99.66% |         3.837130 |        0.034900 |        +99.09% |
| 4d_weights_broadcast  | `2x3x31x127:1x1x127x65` |        60.597900 |        0.146716 |        +99.76% |         8.401550 |        0.034424 |        +99.59% |
| 4d_per_batch_weights  | `2x3x31x127:2x3x127x65` |        60.440700 |        0.186292 |        +99.69% |         8.089500 |        0.037207 |        +99.54% |

### K3 / A100, VLEN=1024, f16

| Case                  | Shape                   | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------------------- | ----------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| 2d_small_tail         | `7x31:31x13`            |         0.275708 |        0.008557 |        +96.90% |         0.067749 |        0.009375 |        +86.16% |
| 2d_both_tails         | `63x127:127x65`         |        44.542600 |        0.166699 |        +99.63% |         5.739930 |        0.048669 |        +99.15% |
| 2d_square             | `64x128:128x64`         |        44.865900 |        0.136230 |        +99.70% |         5.676250 |        0.024524 |        +99.57% |
| 2d_wide               | `128x256:256x512`       |      1457.390000 |        2.230810 |        +99.85% |       194.121000 |        0.503784 |        +99.74% |
| 2d_tall               | `512x256:256x128`       |      1453.970000 |        2.134640 |        +99.85% |       194.510000 |        0.537549 |        +99.72% |
| 2d_large_k            | `32x4096:4096x128`      |      1420.970000 |        2.796480 |        +99.80% |       180.243000 |        2.660160 |        +98.52% |
| 2d_large_n            | `256x512:512x1024`      |     12222.900000 |       18.558100 |        +99.85% |      1583.280000 |        3.287170 |        +99.79% |
| 2d_transposed_weights | `65x127:127x63`         |        44.393700 |        1.082140 |        +97.56% |         5.663490 |        0.386475 |        +93.18% |
| 3d_weights_broadcast  | `2x64x128:1x128x256`    |       436.296000 |        0.543628 |        +99.88% |        59.466800 |        0.089282 |        +99.85% |
| 3d_per_batch_weights  | `2x64x128:2x128x256`    |       434.454000 |        0.571118 |        +99.87% |        59.195000 |        0.302161 |        +99.49% |
| 3d_broadcast_tails    | `3x31x127:1x127x65`     |        78.065900 |        0.242700 |        +99.69% |        10.349600 |        0.069519 |        +99.33% |
| 3d_per_batch_tails    | `3x31x127:3x127x65`     |        78.043200 |        0.251770 |        +99.68% |        10.245700 |        0.091150 |        +99.11% |
| 4d_weights_broadcast  | `2x3x31x127:1x1x127x65` |       180.580000 |        0.477808 |        +99.74% |        23.558100 |        0.125769 |        +99.47% |
| 4d_per_batch_weights  | `2x3x31x127:2x3x127x65` |       180.483000 |        0.493262 |        +99.73% |        23.594700 |        0.089563 |        +99.62% |

### K3 / A100, VLEN=1024, bf16

| Case                  | Shape                   | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------------------- | ----------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| 2d_small_tail         | `7x31:31x13`            |         0.079895 |        0.008276 |        +89.64% |         0.020959 |        0.009424 |        +55.04% |
| 2d_both_tails         | `63x127:127x65`         |        10.838700 |        0.167322 |        +98.46% |         1.437120 |        0.048840 |        +96.60% |
| 2d_square             | `64x128:128x64`         |        10.850200 |        0.135535 |        +98.75% |         1.446030 |        0.024719 |        +98.29% |
| 2d_wide               | `128x256:256x512`       |       345.306000 |        2.208070 |        +99.36% |        52.059600 |        0.511938 |        +99.02% |
| 2d_tall               | `512x256:256x128`       |       344.084000 |        2.139290 |        +99.38% |        45.053800 |        0.538831 |        +98.80% |
| 2d_large_k            | `32x4096:4096x128`      |       346.551000 |        2.785730 |        +99.20% |        59.580100 |        2.656050 |        +95.54% |
| 2d_large_n            | `256x512:512x1024`      |      2744.270000 |       18.629900 |        +99.32% |       358.332000 |        3.208920 |        +99.10% |
| 2d_transposed_weights | `65x127:127x63`         |        11.506500 |        1.083850 |        +90.58% |         1.523460 |        0.387329 |        +74.58% |
| 3d_weights_broadcast  | `2x64x128:1x128x256`    |        88.777100 |        0.550867 |        +99.38% |        12.796300 |        0.094275 |        +99.26% |
| 3d_per_batch_weights  | `2x64x128:2x128x256`    |        88.720200 |        0.566309 |        +99.36% |        12.749900 |        0.303235 |        +97.62% |
| 3d_broadcast_tails    | `3x31x127:1x127x65`     |        16.005700 |        0.244629 |        +98.47% |         2.194200 |        0.070435 |        +96.79% |
| 3d_per_batch_tails    | `3x31x127:3x127x65`     |        16.102600 |        0.250708 |        +98.44% |         2.194250 |        0.090027 |        +95.90% |
| 4d_weights_broadcast  | `2x3x31x127:1x1x127x65` |        32.247900 |        0.476440 |        +98.52% |         4.205330 |        0.125085 |        +97.03% |
| 4d_per_batch_weights  | `2x3x31x127:2x3x127x65` |        32.158300 |        0.495837 |        +98.46% |         4.206690 |        0.092127 |        +97.81% |

The f16 baseline selects `ref:any` for these bias configurations, while the patched build selects `jit:rvv`. On A100, the bf16 baseline selects `gemm:jit:bf16`, and the patched build also shows consistent gains after moving the fused bias into the RVV half-precision kernel. Every affected case is positive on both 1-thread and 8-thread measurements across the three tested cores.

## Correctness validation

Targeted correctness batches cover f16 on X60, X100, A100, and SG2044, plus bf16 on A100. They include 2D/3D/4D problems, weights broadcast, per-batch weights, M/N/K tails, transposed weights, 1-thread and 8-thread execution, and unsupported bias masks that must remain on a fallback implementation. All targeted runs completed with 0 failures.

The patched build also passed the official matmul batches on K3:

| Core | Threads | Batch                  | Result                                          |
| ---- | ------: | ---------------------- | ----------------------------------------------- |
| X100 |       1 | `test_matmul_float16`  | 8802 tests, 2975 passed, 5827 skipped, 0 failed |
| X100 |       8 | `test_matmul_float16`  | 8802 tests, 2975 passed, 5827 skipped, 0 failed |
| A100 |       1 | `test_matmul_float16`  | 8802 tests, 2975 passed, 5827 skipped, 0 failed |
| A100 |       8 | `test_matmul_float16`  | 8802 tests, 2975 passed, 5827 skipped, 0 failed |
| A100 |       1 | `test_matmul_bfloat16` | 8854 tests, 8843 passed, 11 skipped, 0 failed   |
| A100 |       8 | `test_matmul_bfloat16` | 8854 tests, 8843 passed, 11 skipped, 0 failed   |