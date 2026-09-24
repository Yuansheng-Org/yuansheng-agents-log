# CPU: RV64: hoist xf16 softmax exp coefficients on VLEN >= 256

## Description

This PR reduces scalar instruction overhead in the RV64 f16/bf16 softmax exponential kernel. The kernel evaluates the exponential polynomial once for every vector chunk. Previously, its five polynomial coefficients were reconstructed inside that loop with an integer immediate load followed by `fmv.w.x`, adding ten scalar instructions per chunk.

On systems with VLEN of at least 256 bits, the coefficients are now materialized once in the JIT preamble and kept in caller-saved floating-point registers for the lifetime of the kernel. VLEN=128 systems retain the existing per-chunk sequence because measurements found a regression in multithreaded strided logsoftmax. The selection is made while generating the JIT kernel and does not add a runtime branch to generated code. The polynomial evaluation order and constants are unchanged, so the numerical behavior is preserved.

## Implementation details

- Materialize the five f32 polynomial coefficients before entering the vector loop when `VLEN >= 256`.
- Keep each coefficient in a dedicated caller-saved floating-point register.
- Reuse those registers for every f16 or bf16 vector chunk.
- Emit the original single-register per-chunk coefficient sequence when `VLEN == 128`.
- Preserve all vector operations, coefficient bit patterns, polynomial order, tail handling and reduction behavior.
- Keep the existing short-axis reduce-max threshold unchanged.

## Performance evaluation

Performance was measured with Release OpenMP builds on the following RV64 systems. The baseline and patched measurements used the same `benchdnn` executable, with only `libdnnl.so` changed. Reported values are `benchdnn` average execution times; lower is better.

| System      | Core          |      VLEN | Threads |
| ----------- | ------------- | --------: | ------: |
| Muse Pi Pro | SpacemiT X60  |  256 bits |    1, 8 |
| K3 X100     | SpacemiT X100 |  256 bits |    1, 8 |
| K3 A100     | SpacemiT A100 | 1024 bits |    1, 8 |
| SG2044      | T-Head C920   |  128 bits |    1, 8 |

### Test cases

The main f16 batch covers contiguous reduction axes of 64, 128, 256, 1024 and 4096 elements and non-contiguous axes of 64 and 256 elements. Each case contains approximately 32 million elements and is tested with both softmax and logsoftmax. On A100, the affected f16 and bf16 JIT paths are tested at axis lengths 256, 1024 and 4096 plus a non-contiguous axis-256 case. The bf16 A100 table contains softmax only because bf16 logsoftmax dispatches to `ref:any` on this system and does not exercise the changed JIT kernel.

### Muse Pi Pro / X60, f16

| Algorithm and shape             | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| softmax, 524288x64              |         331.4470 |        336.5820 |         -1.55% |          48.9991 |         48.3770 |         +1.27% |
| logsoftmax, 524288x64           |         364.9720 |        342.7780 |         +6.08% |          52.1228 |         50.8943 |         +2.36% |
| softmax, 262144x128             |         256.0430 |        257.2060 |         -0.45% |          38.2661 |         37.2072 |         +2.77% |
| logsoftmax, 262144x128          |         275.4370 |        260.9420 |         +5.26% |          47.5864 |         47.1681 |         +0.88% |
| softmax, 131072x256             |         230.9700 |        223.8990 |         +3.06% |          39.5582 |         34.9558 |        +11.63% |
| logsoftmax, 131072x256          |         239.1430 |        223.9320 |         +6.36% |          48.9099 |         48.0658 |         +1.73% |
| softmax, 32768x1024             |         214.2540 |        203.7550 |         +4.90% |          35.6089 |         38.7565 |         -8.84% |
| logsoftmax, 32768x1024          |         205.0190 |        195.2040 |         +4.79% |          51.9765 |         50.2875 |         +3.25% |
| softmax, 8192x4096              |         223.0480 |        214.6350 |         +3.77% |          41.7978 |         39.6273 |         +5.19% |
| logsoftmax, 8192x4096           |         193.2380 |        182.4550 |         +5.58% |          50.4799 |         49.1068 |         +2.72% |
| softmax, 8192x64x64, axis 1     |         531.4590 |        532.5690 |         -0.21% |          91.1474 |         93.8782 |         -3.00% |
| logsoftmax, 8192x64x64, axis 1  |         569.5390 |        565.6630 |         +0.68% |          95.9698 |         97.7506 |         -1.86% |
| softmax, 2048x256x64, axis 1    |         743.8920 |        737.2220 |         +0.90% |         185.6400 |        183.3210 |         +1.25% |
| logsoftmax, 2048x256x64, axis 1 |         759.6130 |        743.8850 |         +2.07% |         193.9540 |        195.5440 |         -0.82% |

The arithmetic mean across these cases is +2.95% for 1T and +1.32% for 8T.

### K3 X100, f16

| Algorithm and shape             | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| softmax, 524288x64              |         210.6130 |        209.9770 |         +0.30% |          27.3312 |         27.7041 |         -1.36% |
| logsoftmax, 524288x64           |         227.5140 |        228.5790 |         -0.47% |          30.0492 |         30.5470 |         -1.66% |
| softmax, 262144x128             |         157.5360 |        158.5150 |         -0.62% |          20.7337 |         21.3633 |         -3.04% |
| logsoftmax, 262144x128          |         168.6530 |        169.6320 |         -0.58% |          22.2447 |         22.3541 |         -0.49% |
| softmax, 131072x256             |         131.9060 |        132.2600 |         -0.27% |          17.2767 |         18.1203 |         -4.88% |
| logsoftmax, 131072x256          |         139.9780 |        140.4520 |         -0.34% |          18.4469 |         18.4828 |         -0.19% |
| softmax, 32768x1024             |         111.9530 |        111.9320 |         +0.02% |          14.8214 |         14.9489 |         -0.86% |
| logsoftmax, 32768x1024          |         117.5750 |        117.7330 |         -0.13% |          15.9458 |         15.8512 |         +0.59% |
| softmax, 8192x4096              |         106.9450 |        106.9810 |         -0.03% |          14.7485 |         14.9767 |         -1.55% |
| logsoftmax, 8192x4096           |         112.1850 |        112.1780 |         +0.01% |          15.0951 |         15.0503 |         +0.30% |
| softmax, 8192x64x64, axis 1     |         350.3020 |        343.7840 |         +1.86% |          49.8002 |         47.5944 |         +4.43% |
| logsoftmax, 8192x64x64, axis 1  |         372.3180 |        364.1820 |         +2.19% |          51.1560 |         50.2796 |         +1.71% |
| softmax, 2048x256x64, axis 1    |         277.0300 |        273.0730 |         +1.43% |          44.0881 |         43.0603 |         +2.33% |
| logsoftmax, 2048x256x64, axis 1 |         282.6190 |        278.6280 |         +1.41% |          41.0487 |         39.1889 |         +4.53% |

The arithmetic mean across these cases is +0.34% for 1T and -0.01% for 8T. The contiguous cases are effectively neutral on X100, while the non-contiguous cases improve by approximately 1.4% to 4.5%.

### K3 A100, f16

| Algorithm and shape             | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| softmax, 131072x256             |         148.1920 |        142.7090 |         +3.70% |          20.1554 |         19.5086 |         +3.21% |
| logsoftmax, 131072x256          |         153.2200 |        149.0750 |         +2.71% |          23.7508 |         23.2499 |         +2.11% |
| softmax, 32768x1024             |         106.1610 |        101.9380 |         +3.98% |          15.4564 |         14.9501 |         +3.28% |
| logsoftmax, 32768x1024          |         108.8540 |        105.2370 |         +3.32% |          19.6300 |         19.4333 |         +1.00% |
| softmax, 8192x4096              |          84.0829 |         80.1704 |         +4.65% |          10.8035 |         10.2908 |         +4.75% |
| logsoftmax, 8192x4096           |          86.0709 |         82.1006 |         +4.61% |          11.2290 |         11.1752 |         +0.48% |
| softmax, 2048x256x64, axis 1    |         566.1280 |        560.5660 |         +0.98% |          83.5364 |         82.0568 |         +1.77% |
| logsoftmax, 2048x256x64, axis 1 |         571.6480 |        562.5420 |         +1.59% |          84.7294 |         82.9358 |         +2.12% |

The arithmetic mean is +3.19% for 1T and +2.34% for 8T.

### K3 A100, bf16 softmax

| Shape               | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| 131072x256          |         146.1300 |        144.9940 |         +0.78% |          19.9342 |         19.5626 |         +1.86% |
| 32768x1024          |         106.2050 |        102.9560 |         +3.06% |          15.4320 |         15.0547 |         +2.44% |
| 8192x4096           |          85.1477 |         81.2666 |         +4.56% |          10.9615 |         10.4208 |         +4.93% |
| 2048x256x64, axis 1 |         561.7470 |        561.9940 |         -0.04% |          83.3426 |         82.7728 |         +0.68% |

The arithmetic mean is +2.09% for 1T and +2.48% for 8T.

### SG2044 VLEN=128 control

The coefficient hoist is disabled on VLEN=128. A focused 8T baseline-patched-patched-baseline control confirmed that the final generated loop remains at baseline performance, including the strided logsoftmax workload that motivated the gate.

| Algorithm and shape            | Baseline mean (ms) | Patched mean (ms) | Difference |
| ------------------------------ | -----------------: | ----------------: | ---------: |
| softmax, 131072x256            |            24.8601 |           25.8156 |     -3.84% |
| softmax, 32768x1024            |            20.1282 |           20.3304 |     -1.00% |
| softmax, 8192x64x64, axis 1    |            52.5508 |           50.3682 |     +4.15% |
| logsoftmax, 8192x64x64, axis 1 |            53.0614 |           53.0548 |     +0.01% |

The mixed small differences are run-to-run variation around identical VLEN=128 generated code; the previously repeatable strided logsoftmax regression is removed.

## Correctness validation

The patched library passed the official oneDNN softmax batches in both 1T and 8T configurations:

| System            | Batch                   | Result per thread configuration                              |
| ----------------- | ----------------------- | ------------------------------------------------------------ |
| Muse Pi Pro / X60 | `test_softmax_float16`  | 9237 tests, 4660 passed, 4320 skipped, 257 mistrusted, 0 failed |
| K3 X100           | `test_softmax_float16`  | 9237 tests, 4660 passed, 4320 skipped, 257 mistrusted, 0 failed |
| K3 A100           | `test_softmax_float16`  | 9237 tests, 4660 passed, 4320 skipped, 257 mistrusted, 0 failed |
| K3 A100           | `test_softmax_bfloat16` | 9237 tests, 4745 passed, 4320 skipped, 172 mistrusted, 0 failed |

Targeted f16/bf16 batches covering contiguous and non-contiguous axes also passed for baseline and patched libraries in 1T and 8T configurations.