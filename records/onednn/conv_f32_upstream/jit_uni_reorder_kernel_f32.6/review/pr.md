# cpu: rv64: use LMUL=m2 for contiguous reorder on VLEN>=1024

## Description

This PR reduces loop overhead in the RV64 JIT pure-copy reorder kernel on wide-vector systems. When the runtime VLEN is at least 1024 bits, both source and destination accesses are unit-stride, and the innermost extent exceeds the LMUL=m1 capacity, the kernel uses LMUL=m2. All other cases retain LMUL=m1.

The narrower scope is intentional. Larger vector groups did not provide consistent results for VLEN=256 implementations or for strided gather/scatter accesses. Restricting the optimization to wide-vector, contiguous copies preserves the existing path on VLEN=128 and VLEN=256 systems while retaining the measurable benefit on VLEN=1024.

The loop remains vector-length agnostic. `vsetvli` determines the active vector length for every iteration, the returned `vl` controls both address advancement and the remaining-element count, and the final partial vector is handled without a separate scalar tail.

## Implementation details

The pure-copy kernel obtains the runtime VLEN with `get_platform_vlen()` and computes the LMUL=m1 capacity as:

```text
VLMAX_m1 = (VLEN / 8) / element_size
```

LMUL=m2 is selected only when all of the following conditions hold:

- RVV is available;
- VLEN is at least 1024 bits;
- the source innermost stride is one;
- the destination innermost stride is one;
- the innermost extent is greater than `VLMAX_m1`.

The kernel has one live vector data group at `v8`, which is correctly aligned for LMUL=m2. Strided loads and stores continue to use LMUL=m1.

## Test batches

The performance evaluation uses the exact f32 plain-transpose section extracted from the oneDNN `test_reorder_ci` batch:

```bash
benchdnn --mode=P --fix-times-per-prb=100 --engine=cpu --reorder \
    --batch=test_reorder_ci_f32_plain
```

Two supplemental batches cover the LMUL transition and larger unit-stride workloads:

```bash
benchdnn --mode=P --fix-times-per-prb=100 --engine=cpu --reorder \
    --batch=rvv_reorder_pure_copy_sweep
benchdnn --mode=P --fix-times-per-prb=100 --engine=cpu --reorder \
    --batch=rvv_reorder_contiguous_large
```

## Performance evaluation

Performance was measured on the K3 A100 cores with RV64 and RVV VLEN=1024. The A100 tests used CPUs 8-15 at 1.8 GHz with the `userspace` governor. One-thread measurements were pinned to CPU 8 and eight-thread measurements were pinned to CPUs 8-15. The baseline and patched libraries were produced from the same upstream base with the same GCC 11.4 RV64 cross toolchain and Release/OpenMP/Graph configuration. Lower execution time is better.

### Official f32 plain-transpose cases

| Case                     | Implementation | Baseline 1T (ms) | Patched 1T (ms) |     1T | Baseline 8T (ms) | Patched 8T (ms) |      8T |
| ------------------------ | -------------- | ---------------: | --------------: | -----: | ---------------: | --------------: | ------: |
| `abx->axb 1x4x84x84`     | `jit:blk`      |         0.014580 |        0.013772 | +5.54% |         0.011785 |        0.012107 |  -2.73% |
| `abx->axb 1x512x100`     | `jit:uni`      |         0.492290 |        0.493164 | -0.18% |         0.072639 |        0.073091 |  -0.62% |
| `abx->axb 1x512x101`     | `jit:uni`      |         0.498357 |        0.498430 | -0.01% |         0.073472 |        0.073206 |  +0.36% |
| `abx->axb 2x64x15x10x20` | `jit:uni`      |         6.630090 |        6.800890 | -2.58% |         0.689387 |        0.694309 |  -0.71% |
| `abx->axb 2x64x18x18x18` | `jit:uni`      |        13.462400 |       13.459700 | +0.02% |         1.855060 |        1.863500 |  -0.45% |
| `abx->axb 2x64x19x19x7`  | `jit:uni`      |         5.341010 |        5.338290 | +0.05% |         0.512017 |        0.480354 |  +6.18% |
| `abx->axb 2x64x31x32x16` | `jit:uni`      |        53.243000 |       53.248000 | -0.01% |         6.816550 |        6.820160 |  -0.05% |
| `abx->axb 2x64x8`        | `jit:uni`      |         0.012444 |        0.012559 | -0.92% |         0.013015 |        0.012844 |  +1.31% |
| `axb->abx 1x4x84x84`     | `jit:blk`      |         0.074688 |        0.074143 | +0.73% |         0.017310 |        0.018005 |  -4.02% |
| `axb->abx 1x512x100`     | `jit:uni`      |         0.857598 |        0.802627 | +6.41% |         0.208984 |        0.238433 | -14.09% |
| `axb->abx 1x512x101`     | `jit:uni`      |         0.270608 |        0.269561 | +0.39% |         0.101145 |        0.099441 |  +1.68% |
| `axb->abx 2x64x15x10x20` | `jit:uni`      |         3.974400 |        3.790050 | +4.64% |         0.478682 |        0.517256 |  -8.06% |
| `axb->abx 2x64x18x18x18` | `jit:uni`      |         7.466420 |        7.462030 | +0.06% |         1.593400 |        1.526020 |  +4.23% |
| `axb->abx 2x64x19x19x7`  | `jit:uni`      |         3.464770 |        3.463880 | +0.03% |         0.621555 |        0.628479 |  -1.11% |
| `axb->abx 2x64x31x32x16` | `jit:uni`      |        27.979500 |       27.998700 | -0.07% |         4.512500 |        4.498060 |  +0.32% |
| `axb->abx 2x64x8`        | `jit:uni`      |         0.019800 |        0.019695 | +0.53% |         0.019959 |        0.020420 |  -2.31% |
| `ab->ba 16x64`           | `jit:uni`      |         0.015217 |        0.014973 | +1.60% |         0.015317 |        0.015083 |  +1.53% |
| `ab->ba 7x65`            | `jit:uni`      |         0.010664 |        0.010745 | -0.75% |         0.010913 |        0.010908 |  +0.04% |
| `ba->ab 16x64`           | `jit:uni`      |         0.012961 |        0.012925 | +0.28% |         0.013279 |        0.012639 |  +4.82% |
| `ba->ab 7x65`            | `jit:uni`      |         0.007371 |        0.007466 | -1.29% |         0.007607 |        0.007620 |  -0.16% |

The aggregate time changes from 123.848168 ms to 123.791599 ms (+0.05%) with one thread and from 17.644576 ms to 17.621935 ms (+0.13%) with eight threads. This batch includes blocked, strided, and short-inner-axis controls that do not select the new path.

### LMUL-transition cases

| Case                | Implementation | Baseline 1T (ms) | Patched 1T (ms) |      1T | Baseline 8T (ms) | Patched 8T (ms) |      8T |
| ------------------- | -------------- | ---------------: | --------------: | ------: | ---------------: | --------------: | ------: |
| `abc->bac 64x64x4`  | `jit:uni`      |         0.143220 |        0.143669 |  -0.31% |         0.020791 |        0.021174 |  -1.84% |
| `abc->bac 64x64x5`  | `jit:uni`      |         0.154258 |        0.155056 |  -0.52% |         0.023960 |        0.024665 |  -2.94% |
| `abc->bac 64x64x8`  | `jit:uni`      |         0.142576 |        0.142522 |  +0.04% |         0.021243 |        0.021536 |  -1.38% |
| `abc->bac 64x64x9`  | `jit:uni`      |         0.150186 |        0.151533 |  -0.90% |         0.023623 |        0.024004 |  -1.61% |
| `abc->bac 64x64x16` | `jit:uni`      |         0.142219 |        0.142295 |  -0.05% |         0.020410 |        0.020000 |  +2.01% |
| `abc->bac 64x64x17` | `jit:uni`      |         0.164834 |        0.165691 |  -0.52% |         0.039710 |        0.040159 |  -1.13% |
| `abc->bac 64x64x32` | `jit:uni`      |         0.367476 |        0.368914 |  -0.39% |         0.061020 |        0.061445 |  -0.70% |
| `abc->bac 64x64x33` | `jit:uni`      |         0.298594 |        0.206895 | +30.71% |         0.071904 |        0.063860 | +11.19% |
| `abc->bac 64x64x64` | `jit:uni`      |         1.093020 |        1.177410 |  -7.72% |         0.111819 |        0.100461 | +10.16% |
| `abc->bac 64x64x65` | `jit:uni`      |         1.166190 |        1.059960 |  +9.11% |         0.166501 |        0.124636 | +25.14% |
| `abc->acb 64x4x3`   | `jit:blk`      |         0.011812 |        0.012051 |  -2.03% |         0.008914 |        0.009282 |  -4.14% |
| `abc->acb 64x5x3`   | `jit:uni`      |         0.031314 |        0.031355 |  -0.13% |         0.031638 |        0.031721 |  -0.26% |
| `abc->acb 64x8x3`   | `jit:blk`      |         0.030366 |        0.030330 |  +0.12% |         0.010937 |        0.011460 |  -4.78% |
| `abc->acb 64x9x3`   | `jit:uni`      |         0.037961 |        0.038320 |  -0.95% |         0.012588 |        0.012432 |  +1.24% |
| `abc->acb 64x16x3`  | `simple:any`   |         0.486541 |        0.486121 |  +0.09% |         0.070403 |        0.070642 |  -0.34% |
| `abc->acb 64x17x3`  | `jit:uni`      |         0.050542 |        0.049575 |  +1.91% |         0.013850 |        0.013723 |  +0.92% |
| `abc->acb 64x32x3`  | `jit:blk`      |         0.084824 |        0.084810 |  +0.02% |         0.017661 |        0.018533 |  -4.94% |
| `abc->acb 64x33x3`  | `jit:uni`      |         0.078896 |        0.079800 |  -1.14% |         0.017007 |        0.017854 |  -4.98% |
| `abc->acb 64x64x3`  | `jit:uni`      |         0.121953 |        0.120132 |  +1.49% |         0.023125 |        0.022893 |  +1.00% |
| `abc->acb 64x65x3`  | `jit:uni`      |         0.137607 |        0.138713 |  -0.80% |         0.024404 |        0.024619 |  -0.88% |

The aggregate time changes from 4.894389 ms to 4.785151 ms (+2.23%) with one thread and from 0.791508 ms to 0.735099 ms (+7.13%) with eight threads. For f32 at VLEN=1024, LMUL=m1 holds 32 elements, so the contiguous `abc->bac` rows with inner lengths 33, 64, and 65 exercise the new path. The `abc->acb` rows are strided controls and remain on LMUL=m1.

### Large contiguous cases

| Case                  | Implementation | Baseline 1T (ms) | Patched 1T (ms) |      1T | Baseline 8T (ms) | Patched 8T (ms) |      8T |
| --------------------- | -------------- | ---------------: | --------------: | ------: | ---------------: | --------------: | ------: |
| `abc->bac 512x64x9`   | `jit:uni`      |         1.366350 |        1.372080 |  -0.42% |         0.190938 |        0.187114 |  +2.00% |
| `abc->bac 512x64x16`  | `jit:uni`      |         4.480600 |        4.482800 |  -0.05% |         0.854497 |        0.861707 |  -0.84% |
| `abc->bac 512x64x17`  | `jit:uni`      |         5.103290 |        5.163990 |  -1.19% |         0.660815 |        0.721799 |  -9.23% |
| `abc->bac 512x64x32`  | `jit:uni`      |         9.911090 |        9.891360 |  +0.20% |         1.667940 |        1.691570 |  -1.42% |
| `abc->bac 512x64x33`  | `jit:uni`      |        10.261700 |        8.298650 | +19.13% |         2.031790 |        1.680610 | +17.28% |
| `abc->bac 512x64x64`  | `jit:uni`      |        16.523000 |       16.564000 |  -0.25% |         2.705040 |        2.726520 |  -0.79% |
| `abc->bac 512x64x65`  | `jit:uni`      |        12.793600 |       11.834200 |  +7.50% |         2.801980 |        2.672120 |  +4.63% |
| `abc->bac 512x64x128` | `jit:uni`      |        22.740500 |       22.548300 |  +0.85% |         5.494670 |        5.415820 |  +1.44% |
| `abc->bac 512x64x129` | `jit:uni`      |        19.218000 |       17.676900 |  +8.02% |         6.071450 |        6.008770 |  +1.03% |
| `abc->bac 512x64x256` | `jit:uni`      |        36.017300 |       36.058300 |  -0.11% |         7.121630 |        7.013190 |  +1.52% |

The aggregate time changes from 138.415430 ms to 133.890580 ms (+3.27%) with one thread and from 29.600750 ms to 28.979220 ms (+2.10%) with eight threads. Inner lengths up to 32 remain on LMUL=m1. The affected lengths greater than 32 are positive in aggregate; cases already dominated by memory bandwidth remain approximately neutral.

## Correctness validation

The focused `rvv_reorder_pure_copy_sweep` correctness batch passed all 20 cases on Muse Pi Pro/K1 (VLEN=256), K3 X100 (VLEN=256), and K3 A100 (VLEN=1024). K1 and X100 verify that the narrower-vector no-op path remains correct, while A100 exercises the new LMUL=m2 path.

The final locally cross-compiled build also passed the complete CTest suite on K3 A100:

```text
100% tests passed, 0 tests failed out of 174
Total Test time (real) = 20886.67 sec
```