# cpu: rv64: vectorize f32 softmax max reduction with RVV

## Description

This PR optimizes the first-stage maximum reduction in the RV64 f32 softmax and logsoftmax implementations.

Previously, the maximum along the softmax axis was computed with a scalar loop. This change introduces an RVV JIT kernel using `vfredmax.vs`.

Measurements on both VLEN=128 and VLEN=256 systems showed that the crossover is platform-dependent, so the vector path is enabled for VLEN>=256 and reduction lengths of at least 15. VLEN=128 keeps the scalar path.

The platform VLEN is queried once per primitive execution, outside the row loop. All changes are restricted to `src/cpu/rv64`; no common oneDNN interfaces or non-RISC-V implementations are modified.

## Implementation details

The new kernel uses LMUL=4, dynamically strip-mines arbitrary lengths with `vsetvli`, and handles a final partial vector. The accumulator is initialized to negative infinity.

The scalar implementation updates the maximum only when `value > max_value`, so a NaN never replaces the current maximum. The vector kernel detects NaNs by self-comparison and merges those lanes to negative infinity before `vfredmax.vs`. This preserves the scalar behavior for normal values, NaNs, infinities, and signed zero.

## Test batches

Performance was first measured with the complete built-in oneDNN softmax CI batch. It expands to 13,888 problems:

```bash
benchdnn --mode=P --engine=cpu --softmax \
    --batch=test_softmax_ci
```

Because the complete batch includes mostly unaffected data types, backward passes, and short reductions, the 128 affected f32 forward cases from `test_softmax_ci` were also measured as a group:

```bash
benchdnn --mode=P --engine=cpu --softmax \
    --batch=test_softmax_ci_f32_fwd_affected
```

That subset is too small to characterize the reduction-length crossover, so 32 contiguous and non-contiguous softmax/logsoftmax cases were added:

```bash
benchdnn --mode=P --engine=cpu --softmax \
    --batch=rvv_softmax_lmul_sweep.batch
```

Correctness used the complete built-in batch and both focused batches with `--mode=C`.

## Performance evaluation

Tests used Release builds with OpenMP and the performance governor. Lower aggregate execution time is better.

### Aggregate official and supplemental results

| System | VLEN | Batch | Threads | Baseline (ms) | Patched (ms) | Improvement |
|---|---:|---|---:|---:|---:|---:|
| Muse Pi Pro / SpacemiT X60 | 256 | built-in `test_softmax_ci` | 1 | 9637.37 | 9533.84 | +1.07% |
| Muse Pi Pro / SpacemiT X60 | 256 | built-in `test_softmax_ci` | 8 | 7829.37 | 7717.57 | +1.43% |
| Muse Pi Pro / SpacemiT X60 | 256 | affected official cases | 1 | 23.6196 | 23.4946 | +0.53% |
| Muse Pi Pro / SpacemiT X60 | 256 | affected official cases | 8 | 49.0237 | 39.9763 | +18.46% |
| Muse Pi Pro / SpacemiT X60 | 256 | supplemental | 1 | N/A | N/A | +19.18% |
| Muse Pi Pro / SpacemiT X60 | 256 | supplemental | 8 | N/A | N/A | +10.30% |
| SG2044 | 128 | built-in `test_softmax_ci`, guarded no-op | 1 | 6563.62 | 6659.02 | -1.45% |
| SG2044 | 128 | built-in `test_softmax_ci`, guarded no-op | 8 | 4564.03 | 4544.54 | +0.43% |
| SG2044 | 128 | affected official cases, guarded no-op | 1 | 12.3091 | 11.7385 | +4.64% |
| SG2044 | 128 | affected official cases, guarded no-op | 8 | 14.0190 | 14.2869 | -1.91% |
| SG2044 | 128 | supplemental | 1 | N/A | N/A | +1.44% |
| SG2044 | 128 | supplemental | 8 | N/A | N/A | +1.73% |

The complete built-in batch improves on VLEN=256 despite containing many unaffected cases. The VLEN=128 rows exercise the guarded scalar path; their changes are run-to-run variation and are not attributed to the RVV reduction.

### Muse Pi Pro supplemental cases, single thread

| Axis length | Improvement |
|---:|---:|
| 4 | -0.37% |
| 8 | +0.73% |
| 12 | -1.18% |
| 15 | +1.88% |
| 16 | +6.70% |
| 17 | +10.64% |
| 24 | +7.77% |
| 31 | +8.61% |
| 32 | +24.80% |
| 33 | +5.77% |
| 48 | +25.79% |
| 64 | +32.87% |
| 96 | +34.65% |
| 128 | +37.97% |
| 256 | +40.83% |
| 1024 | +43.73% |

### Muse Pi Pro supplemental cases, eight threads

| Axis length | Improvement |
|---:|---:|
| 4 | +1.39% |
| 8 | -1.67% |
| 12 | -2.03% |
| 15 | +1.08% |
| 16 | +9.69% |
| 17 | +7.86% |
| 24 | +7.51% |
| 31 | +5.61% |
| 32 | +20.40% |
| 33 | +3.40% |
| 48 | +19.99% |
| 64 | +18.90% |
| 96 | +16.84% |
| 128 | +19.98% |
| 256 | +19.55% |
| 1024 | +10.47% |

The VLEN=256 vector path becomes positive around the selected crossover and reaches more than 40% improvement for long single-thread reductions.

## Correctness validation

`ctest` passed.