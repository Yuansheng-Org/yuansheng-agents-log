RVV `QT_fp16` currently inherits scalar encoding. This adds an RVV encoder while preserving the generic codec's byte representation, including its rounding behavior.

The generic codec uses ryg's scale-and-round conversion, whose halfway behavior differs from a direct IEEE round-to-nearest-even FP16 conversion. The vector path mirrors its mask, scale, clamp, bias, and sign operations. Chunks containing NaN or infinity use the scalar codec. The generic encoder becomes overridable so the RVV specialization can provide this implementation.

The new regression tests compare encoded bytes with the generic codec for finite values and, separately, for NaN/Inf inputs. Keeping these corpora separate exercises the vector path without allowing special-value fallback to hide an error. Tests include ties, subnormals, overflow boundaries, short tails, and output guards. This PR is independent of the other RVV encoding/distance changes.

### Performance

Measured on a native SG2044 RISC-V host (VLEN=128), GCC 15.1, Release `-O3`, dynamic dispatch (`FAISS_OPT_LEVEL=dd`), `rv64gcv_zvfhmin/lp64d`, one thread pinned to CPU 2. The baseline is official commit `80a16564f86530dbf0bfaf96c2b71feffeb5093f`; the candidate is that same baseline plus only this kernel change. These measurements were not collected on the newer PR base `2ed4c106e9fb9686e7727e5daf8ad6ad1e164109`. The affected scalar-quantizer source files are unchanged between those bases, and the submitted kernel differs from the measured one only in comments/formatting.

| Public path | Dimension | Baseline ns/element | Candidate ns/element | Paired speedup | 95% interval |
|---|---:|---:|---:|---:|---:|
| FP16 encode | 16 | 4.714 | 2.283 | 2.014x | [1.966, 2.089] |
| FP16 encode | 32 | 4.151 | 1.776 | 2.302x | [1.802, 2.346] |
| FP16 encode | 128 | 3.929 | 1.450 | 2.697x | [1.998, 2.712] |
| FP16 encode | 768 | 3.620 | 1.556 | 2.322x | [2.272, 2.380] |

Geometric mean of the dimension-specific speedups at d=32/128/768: FP16 encode **2.434x**.

There are three consecutive sessions, each with four alternating ABBA/BAAB blocks per dimension/path: **48 paired blocks** for this candidate. A is the baseline and B is the candidate. Each call is calibrated to at least 0.1 s (the shortest formal call in the full campaign was 0.192 s), with three warm-up batches and `n = max(32, floor(32768/d))`. Each block uses the ratio of the two-call geometric mean times. Reported speedup is the median of the three session medians; intervals use 5,000 hierarchical bootstrap resamples, resampling sessions and then blocks within each ABBA/BAAB order stratum. The timing columns are separate medians, so their quotient need not equal the paired speedup. All 48 paired blocks favored this candidate.

The timed operation is public `ScalarQuantizer::compute_codes`, using fixed-seed (718) finite input batches. FP16 input values are `float(int(rng()%100000)-50000)/113.f`.

Variability is material at d=32/128: paired-ratio CV is 13.25%/13.47%, and ABBA-vs-BAAB order gaps are 13.46%/4.80%. The d=32/128/768 geometric mean for the individual sessions is 2.419x, 2.040x, and 2.463x. The aggregate 2.434x should therefore not be interpreted as a stable per-run guarantee.

These results describe public encoding/distance throughput on one non-exclusive host, not end-to-end ANN search speedup. The intervals describe these sessions only, with no multiple-comparison correction; they do not establish portability across machines or vector lengths.

### Validation

- Native public encoding oracle: **11,564,775** input elements across the five RISC-V rounding modes; zero differing output bytes or write-guard failures. The corpus covers FP16 values and neighboring floats, ties, subnormals, overflow, signed zeros, and NaN payloads. FP exception flags were recorded, not asserted equivalent.
- Original benchmark baseline full C++ suite: **273 passed, 7 skipped, 0 failed**.
- Submission version on `2ed4c106e9fb9686e7727e5daf8ad6ad1e164109`: all three independent candidate builds succeeded; this PR's focused C++ suite (`NONE`: 11 passed, 6 skipped, 0 failed; `RISCV_RVV`: 13 passed, 4 skipped, 0 failed) and the public-path oracle above pass on native RISC-V. The newly added RVV regression tests are executed and passed in the RVV run, and skipped when RVV is disabled in the NONE run.
- All touched C++ files pass clang-format 21.1.8; `git diff --check` clean.

### Notes

- Touches three files: `faiss/impl/scalar_quantizer/quantizers.h` (one-line `final` → `override` so the generic encoder can be specialized), `faiss/impl/scalar_quantizer/sq-rvv.cpp` (the RVV encoder), and `tests/test_scalar_quantizer.cpp` (two regression tests).
- The vector path operates on `e32m4` float lanes and narrows to `u16m2` at the end, with `vsetvl`-driven bounds, so it adapts to the runtime vector length rather than assuming VLEN=128.
- NaN/Inf detection is done per chunk with `vmsgeu` against `0x7f800000` on the magnitude and `vcpop_m`; if any lane is special, the whole chunk falls back to the scalar codec. This keeps the vector path free of special-value branching while guaranteeing identical bytes for non-finite inputs.
- The rounding sequence (mask low 12 bits, scale by `0x1p-112`, clamp to the largest finite FP16 exponent field, add `0x1000`, shift right by 13, re-OR the sign) is deliberately a mirror of ryg's scalar operation rather than an IEEE `vcvt` FP16 conversion, because the two disagree on halfway cases.

Co-authored-by: ihb2032 <hebome@foxmail.com>
Co-authored-by: lyd1992 <liuyudong@iscas.ac.cn>