# cpu: rv64: widen vector groups in the eltwise JIT kernel



## Description

This PR increases the RV64 eltwise JIT compute group from LMUL=1 to LMUL=2 for f32 and from the e16/m1-to-e32/m2 widening pair to e16/m2-to-e32/m4 for f16 and bf16. Processing more elements per strip-mined iteration amortizes `vsetvli`, pointer-update, and loop-branch overhead.

The injector `group_stride` now follows the actual compute LMUL: 2 for f32 and 4 for the f16, bf16, and other m4 compute paths. This keeps the injector register-range and alignment contract consistent with the vector groups used by the caller.

The worst backward path keeps seven LMUL=m4 data groups live in `v4` through `v31`. The mask uses the architectural mask register `v0` independently, so the register allocation remains legal without spilling. Tail processing remains vector-length agnostic and is controlled by `vsetvli`.

## Test batches

Performance was measured with the built-in oneDNN f16 eltwise batch:

```bash
benchdnn --mode=P --fix-times-per-prb=5 --engine=cpu --eltwise \
    --batch=test_eltwise_float16
```

The built-in batch contains broad f16 algorithm coverage but relatively few large, compute-heavy cases, so it was supplemented with eight f16 rows, eight bf16 rows, and six f32 rows covering forward and backward `soft_relu`, `gelu_erf`, `gelu_tanh`, `mish`, `exp`, and `relu` workloads at 1048576 elements:

| Data type | Direction | Algorithms                                     | Alpha | Beta | Tag  |   Shape | Systems                       |
| --------- | --------- | ---------------------------------------------- | ----: | ---: | ---- | ------: | ----------------------------- |
| f16       | FWD       | `soft_relu`                                    |     1 |    0 | `a`  | 1048576 | Muse Pi Pro, K3 X100, K3 A100 |
| f16       | FWD       | `gelu_erf`, `gelu_tanh`, `mish`, `exp`, `relu` |     0 |    0 | `a`  | 1048576 | Muse Pi Pro, K3 X100, K3 A100 |
| f16       | BWD       | `gelu_erf`, `gelu_tanh`                        |     0 |    0 | `a`  | 1048576 | Muse Pi Pro, K3 X100, K3 A100 |
| bf16      | FWD       | `soft_relu`                                    |     1 |    0 | `a`  | 1048576 | K3 A100                       |
| bf16      | FWD       | `gelu_erf`, `gelu_tanh`, `mish`, `exp`, `relu` |     0 |    0 | `a`  | 1048576 | K3 A100                       |
| bf16      | BWD       | `gelu_erf`, `gelu_tanh`                        |     0 |    0 | `a`  | 1048576 | K3 A100                       |
| f32       | FWD       | `soft_relu`                                    |     1 |    0 | `a`  | 1048576 | Muse Pi Pro, K3 X100, K3 A100 |
| f32       | FWD       | `gelu_erf`, `gelu_tanh`, `mish`                |     0 |    0 | `a`  | 1048576 | Muse Pi Pro, K3 X100, K3 A100 |
| f32       | BWD       | `gelu_erf`, `gelu_tanh`                        |     0 |    0 | `a`  | 1048576 | Muse Pi Pro, K3 X100, K3 A100 |

```bash
benchdnn --mode=P --fix-times-per-prb=5 --engine=cpu --eltwise \
    --batch=rvv_eltwise_large_complex
benchdnn --mode=P --fix-times-per-prb=5 --engine=cpu --eltwise \
    --batch=rvv_eltwise_large_complex_bf16
```

Correctness was checked with the built-in `test_eltwise_ci` batch and the supplemental batch in correctness mode:

```bash
benchdnn --mode=C --engine=cpu --eltwise --batch=test_eltwise_ci
benchdnn --mode=C --engine=cpu --eltwise \
    --batch=rvv_eltwise_large_complex
benchdnn --mode=C --engine=cpu --eltwise \
    --batch=rvv_eltwise_large_complex_bf16
```

## Performance evaluation

The baseline and patched libraries were built from the same upstream base with the same GCC 11.4 RV64 cross toolchain and Release/OpenMP/Graph configuration. Tests used fixed CPU affinity and OpenMP thread placement. The values below are sums of the per-case average execution times; lower is better.

| System                     | VLEN | Batch                            | Threads | Baseline (ms) | Patched (ms) | Improvement |
| -------------------------- | ---: | -------------------------------- | ------: | ------------: | -----------: | ----------: |
| Muse Pi Pro / SpacemiT X60 |  256 | `test_eltwise_float16`           |       1 |      1925.810 |     1949.440 |      -1.23% |
| Muse Pi Pro / SpacemiT X60 |  256 | `test_eltwise_float16`           |       8 |       488.064 |      419.077 |     +14.13% |
| Muse Pi Pro / SpacemiT X60 |  256 | `rvv_eltwise_large_complex`      |       1 |       260.056 |      181.738 |     +30.12% |
| Muse Pi Pro / SpacemiT X60 |  256 | `rvv_eltwise_large_complex`      |       8 |        59.358 |       37.088 |     +37.52% |
| K3 X100                    |  256 | `test_eltwise_float16`           |       1 |       791.908 |      793.170 |      -0.16% |
| K3 X100                    |  256 | `test_eltwise_float16`           |       8 |       136.241 |      135.385 |      +0.63% |
| K3 X100                    |  256 | `rvv_eltwise_large_complex`      |       1 |       140.657 |       92.927 |     +33.93% |
| K3 X100                    |  256 | `rvv_eltwise_large_complex`      |       8 |        18.788 |       12.272 |     +34.68% |
| K3 A100                    | 1024 | `test_eltwise_float16`           |       1 |      1751.648 |     1751.006 |      +0.04% |
| K3 A100                    | 1024 | `test_eltwise_float16`           |       8 |       291.492 |      283.113 |      +2.87% |
| K3 A100                    | 1024 | `rvv_eltwise_large_complex`      |       1 |        89.187 |       76.884 |     +13.80% |
| K3 A100                    | 1024 | `rvv_eltwise_large_complex`      |       8 |        11.569 |       10.028 |     +13.32% |
| K3 A100                    | 1024 | `rvv_eltwise_large_complex_bf16` |       1 |        42.809 |       37.870 |     +11.54% |
| K3 A100                    | 1024 | `rvv_eltwise_large_complex_bf16` |       8 |         5.602 |        4.962 |     +11.44% |

The built-in f16 batch is dominated by small cases and is approximately neutral in the single-thread measurements. The 1048576-element cases provide a clearer measurement of the changed vector loop. The smaller A100 gain relative to the 30% to 38% improvements on VLEN=256 is consistent with VLEN=1024 already processing four times as many bits per LMUL=1 group, reducing the fraction of loop-control overhead removed by widening LMUL.

### Muse Pi Pro / SpacemiT X60 large-case detail

| Data type | Direction | Algorithm | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------- | --------- | --------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| f16       | FWD       | soft_relu |        16.899800 |       13.080000 |        +22.60% |         2.233980 |        1.726560 |        +22.71% |
| f16       | FWD       | gelu_erf  |        14.831300 |       12.169700 |        +17.95% |         2.014940 |        1.602730 |        +20.46% |
| f16       | FWD       | gelu_tanh |        13.931900 |       11.152100 |        +19.95% |        11.510400 |        1.461770 |        +87.30% |
| f16       | FWD       | mish      |        11.668700 |        9.717040 |        +16.73% |         1.768550 |        1.258150 |        +28.86% |
| f16       | FWD       | exp       |         7.698290 |        5.573390 |        +27.60% |         1.582180 |        1.152050 |        +27.19% |
| f16       | FWD       | relu      |         1.205270 |        1.080030 |        +10.39% |         0.680322 |        0.667578 |         +1.87% |
| f16       | BWD       | gelu_erf  |        23.163600 |       18.721300 |        +19.18% |         2.989600 |        2.424020 |        +18.92% |
| f16       | BWD       | gelu_tanh |        18.165600 |       14.634000 |        +19.44% |         2.407760 |        2.293650 |         +4.74% |
| f32       | FWD       | soft_relu |        28.098700 |       16.587900 |        +40.97% |         4.546970 |        2.162790 |        +52.43% |
| f32       | FWD       | gelu_erf  |        23.126600 |       14.518000 |        +37.22% |        14.112500 |        1.944630 |        +86.22% |
| f32       | FWD       | gelu_tanh |        20.253500 |       13.048600 |        +35.57% |         3.194190 |        7.336960 |       -129.70% |
| f32       | FWD       | mish      |        17.212300 |       11.101000 |        +35.51% |         3.386910 |        7.724460 |       -128.07% |
| f32       | BWD       | gelu_erf  |        36.764100 |       22.871800 |        +37.79% |         4.898580 |        2.998630 |        +38.79% |
| f32       | BWD       | gelu_tanh |        27.036300 |       17.483500 |        +35.33% |         4.030960 |        2.333500 |        +42.11% |

### K3 X100 large-case detail

| Data type | Direction | Algorithm | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------- | --------- | --------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| f16       | FWD       | soft_relu |         8.016310 |        5.957860 |        +25.68% |         1.016600 |        0.778662 |        +23.41% |
| f16       | FWD       | gelu_erf  |         8.128030 |        6.516550 |        +19.83% |         1.041260 |        0.825195 |        +20.75% |
| f16       | FWD       | gelu_tanh |         7.256400 |        5.752390 |        +20.73% |         1.053220 |        0.731396 |        +30.56% |
| f16       | FWD       | mish      |         6.343600 |        5.181350 |        +18.32% |         0.806641 |        0.661963 |        +17.94% |
| f16       | FWD       | exp       |         3.626710 |        2.636960 |        +27.29% |         0.466797 |        0.465918 |         +0.19% |
| f16       | FWD       | relu      |         0.690820 |        0.493115 |        +28.62% |         0.096094 |        0.089893 |         +6.45% |
| f16       | BWD       | gelu_erf  |        12.204000 |        9.570800 |        +21.58% |         1.668700 |        1.205760 |        +27.74% |
| f16       | BWD       | gelu_tanh |         9.203120 |        7.279000 |        +20.91% |         1.158450 |        0.919238 |        +20.65% |
| f32       | FWD       | soft_relu |        14.656900 |        7.794680 |        +46.82% |         1.984330 |        1.112400 |        +43.94% |
| f32       | FWD       | gelu_erf  |        13.379200 |        7.931740 |        +40.72% |         1.808940 |        1.134910 |        +37.26% |
| f32       | FWD       | gelu_tanh |        11.585600 |        7.047070 |        +39.17% |         1.587110 |        0.884521 |        +44.27% |
| f32       | FWD       | mish      |         9.744680 |        6.074220 |        +37.67% |         1.240280 |        0.766553 |        +38.20% |
| f32       | BWD       | gelu_erf  |        20.900900 |       11.891400 |        +43.11% |         2.787450 |        1.523630 |        +45.34% |
| f32       | BWD       | gelu_tanh |        14.920300 |        8.799460 |        +41.02% |         2.072310 |        1.171630 |        +43.46% |

### K3 A100 large-case detail

| Data type | Direction | Algorithm | 1T baseline (ms) | 1T patched (ms) | 1T improvement | 8T baseline (ms) | 8T patched (ms) | 8T improvement |
| --------- | --------- | --------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| f16       | FWD       | soft_relu |         5.268650 |        4.386820 |        +16.74% |         0.680469 |        0.573486 |        +15.72% |
| f16       | FWD       | gelu_erf  |         6.436180 |        5.848490 |         +9.13% |         0.839844 |        0.750830 |        +10.60% |
| f16       | FWD       | gelu_tanh |         6.185740 |        5.565190 |        +10.03% |         0.790527 |        0.716650 |         +9.35% |
| f16       | FWD       | mish      |         5.526070 |        5.017530 |         +9.20% |         0.714990 |        0.650000 |         +9.09% |
| f16       | FWD       | exp       |         2.329690 |        1.954640 |        +16.10% |         0.315967 |        0.260400 |        +17.59% |
| f16       | FWD       | relu      |         0.503760 |        0.391113 |        +22.36% |         0.145850 |        0.145850 |         +0.00% |
| f16       | BWD       | gelu_erf  |         9.105960 |        8.077100 |        +11.30% |         1.170510 |        1.051610 |        +10.16% |
| f16       | BWD       | gelu_tanh |         7.725240 |        6.848490 |        +11.35% |         0.986768 |        0.879248 |        +10.90% |
| bf16      | FWD       | soft_relu |         5.282810 |        4.385400 |        +16.99% |         0.679980 |        0.568408 |        +16.41% |
| bf16      | FWD       | gelu_erf  |         6.436230 |        5.880620 |         +8.63% |         0.824023 |        0.747559 |         +9.28% |
| bf16      | FWD       | gelu_tanh |         6.037790 |        5.443550 |         +9.84% |         0.778467 |        0.701758 |         +9.85% |
| bf16      | FWD       | mish      |         5.506250 |        5.021730 |         +8.80% |         0.709668 |        0.642871 |         +9.41% |
| bf16      | FWD       | exp       |         2.338430 |        1.937300 |        +17.15% |         0.318213 |        0.261426 |        +17.85% |
| bf16      | FWD       | relu      |         0.504346 |        0.380713 |        +24.51% |         0.147266 |        0.138672 |         +5.84% |
| bf16      | BWD       | gelu_erf  |         9.092630 |        8.084620 |        +11.09% |         1.173190 |        1.035500 |        +11.74% |
| bf16      | BWD       | gelu_tanh |         7.610300 |        6.735940 |        +11.49% |         0.971582 |        0.865381 |        +10.93% |
| f32       | FWD       | soft_relu |         6.744820 |        5.202690 |        +22.86% |         0.863672 |        0.661035 |        +23.46% |
| f32       | FWD       | gelu_erf  |         7.417430 |        6.414260 |        +13.52% |         0.956445 |        0.824756 |        +13.77% |
| f32       | FWD       | gelu_tanh |         6.872310 |        5.838430 |        +15.04% |         0.890820 |        0.762354 |        +14.42% |
| f32       | FWD       | mish      |         5.978130 |        5.319380 |        +11.02% |         0.770215 |        0.676123 |        +12.22% |
| f32       | BWD       | gelu_erf  |        10.467800 |        8.854100 |        +15.42% |         1.339750 |        1.128910 |        +15.74% |
| f32       | BWD       | gelu_tanh |         8.625490 |        7.165330 |        +16.93% |         1.102730 |        0.946729 |        +14.15% |

## Correctness validation

The corrected implementation passed the following benchdnn checks:

| System                     | Batch                            | Result                                             |
| -------------------------- | -------------------------------- | -------------------------------------------------- |
| Muse Pi Pro / SpacemiT X60 | `test_eltwise_ci`                | 1544 passed, 3240 skipped, 16 mistrusted, 0 failed |
| Muse Pi Pro / SpacemiT X60 | `rvv_eltwise_large_complex`      | 14 passed, 0 failed                                |
| K3 X100                    | `test_eltwise_ci`                | 2240 passed, 2532 skipped, 28 mistrusted, 0 failed |
| K3 X100                    | `rvv_eltwise_large_complex`      | 14 passed, 0 failed                                |
| K3 A100                    | `test_eltwise_ci`                | 2240 passed, 2532 skipped, 28 mistrusted, 0 failed |
| K3 A100                    | `rvv_eltwise_large_complex`      | 14 passed, 0 failed                                |
| K3 A100                    | `rvv_eltwise_large_complex_bf16` | 8 passed, 0 failed                                 |

The full CTest suite was also run on K3 X100:

```text
100% tests passed, 0 tests failed out of 174
Total Test time (real) = 11595.71 sec
```