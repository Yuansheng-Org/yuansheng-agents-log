# CPU: RV64: use widening accumulation for f16 layer normalization

## Description

This PR reduces the instruction count of the first f16 sum pass in the RV64 layer normalization JIT kernel. For profitable shapes on VLEN 128-256 systems, it replaces the separate f16-to-f32 conversion and f32 addition with `vfwadd.wv`, which directly accumulates the narrow f16 input into the wide f32 accumulator.

The optimized sequence is selected when `VLEN <= 256` and the normalization axis is at least 129 elements. The decision is made while the primitive creates its JIT kernel, so the row-processing hot path does not gain a runtime branch. Wider-vector systems and shorter axes retain the existing `vfwcvt.v.f.v` plus `vfadd.vv` sequence.

## Implementation details

The f32 accumulator uses an e32/m8 register group and the f16 input uses a non-overlapping e16/m4 register group. The widening instruction is emitted as `vfwadd.wv(v_sum, v_sum, v_ld)`, where the wide source and destination are the accumulator and the narrow source is the newly loaded f16 vector.

The accumulation uses a tail-undisturbed policy. This is required because the final chunk may be shorter than VLMAX while inactive accumulator lanes still contain partial sums from preceding chunks. The subsequent reduction processes the complete wide accumulator group.

## Performance evaluation

Performance was measured with the same `benchdnn` executable for baseline and patched libraries. Each performance problem used 100 fixed executions. Tests covered 1 and 8 OpenMP threads on K1/X60 (VLEN 256), SG2044-179 (VLEN 128), K3 X100 (VLEN 256), and K3 A100 (VLEN 1024).

The official f16 forward layer normalization cases contained 48 problems per configuration, including 44 RVV JIT cases. A supplementary 68-case batch covered normalization axes 15/16/17, 31/32/33, 63/64/65, 127/128/129, 255/256/257, 1024, and 4096 with plain and `axb` layouts and with and without scale/shift flags.

The table reports the reduction in the sum of minimum execution times for cases that activate the new path. Higher is better.

| Platform   | VLEN | Official 1T | Official 8T | Axis sweep 1T | Axis sweep 8T |
| ---------- | ---: | ----------: | ----------: | ------------: | ------------: |
| K1 / X60   |  256 |      +2.92% |      -0.84% |        +2.56% |        +2.63% |
| SG2044-179 |  128 |      +2.45% |      +6.05% |        +4.92% |        +4.87% |
| K3 X100    |  256 |      +7.54% |      +6.46% |        +4.43% |        +3.37% |

K1 official 8-thread minimum-time sums varied by -0.84%, while their average-time sums improved by 11.49% and the dedicated 8-thread axis sweep improved by 2.63%. The other VLEN 128-256 systems showed positive 8-thread results. K3 A100 does not activate the new sequence; its four aggregate control measurements remained within -0.36% to +0.23% of baseline.

## Correctness validation

The full official `test_lnorm_ci` batch completed with `failed:0` on all four platforms. K1/X60 reported 5064 passed, 4752 skipped, and 312 mistrusted tests. SG2044-179, K3 X100, and K3 A100 each reported 8944 passed, 576 skipped, and 608 mistrusted tests.

The targeted official f16 batch and supplementary axis sweep also completed with `failed:0` for baseline and patched libraries in both 1-thread and 8-thread configurations. Baseline and patched runs selected identical implementations for every measured case.