RVV `QT_8bit_direct` currently inherits scalar encoding. This adds an RVV encoder using `e32m4`, explicit round-toward-zero float-to-unsigned conversion, and two narrowing operations before storing bytes.

The generic encoder becomes overridable so the RVV specialization can provide the implementation. The portable scalar conversion domain is finite inputs in `(-1, 256)`, whose truncated values are representable in `uint8_t`; negative fractions greater than -1 produce zero. The new regression test covers that domain, including signed zero, subnormals, values around integer boundaries, the largest float below 256, short tails, and output guards. This PR is independent of the FP16 and direct-signed-distance changes.

### Performance

Measured on a native SG2044 RISC-V host (VLEN=128), GCC 15.1, Release `-O3`, dynamic dispatch (`FAISS_OPT_LEVEL=dd`), `rv64gcv_zvfhmin/lp64d`, one thread pinned to CPU 2. The baseline is official commit `80a16564f86530dbf0bfaf96c2b71feffeb5093f`; the candidate is that same baseline plus only this kernel change. These measurements were not collected on the newer PR base `2ed4c106e9fb9686e7727e5daf8ad6ad1e164109`. The affected scalar-quantizer source files are unchanged between those bases, and the submitted kernel differs from the measured one only in comments/formatting.

| Public path | Dimension | Baseline ns/element | Candidate ns/element | Paired speedup | 95% interval |
|---|---:|---:|---:|---:|---:|
| Direct-u8 encode | 16 | 1.839 | 1.345 | 1.366x | [1.343, 1.379] |
| Direct-u8 encode | 32 | 1.839 | 1.061 | 1.750x | [1.690, 1.775] |
| Direct-u8 encode | 128 | 1.640 | 0.823 | 1.937x | [1.901, 1.992] |
| Direct-u8 encode | 768 | 1.530 | 0.740 | 2.059x | [1.883, 2.105] |

Geometric mean of the dimension-specific speedups at d=32/128/768: Direct-u8 encode **1.911x**.

There are three consecutive sessions, each with four alternating ABBA/BAAB blocks per dimension/path: **48 paired blocks** for this candidate. A is the baseline and B is the candidate. Each call is calibrated to at least 0.1 s (the shortest formal call in the full campaign was 0.192 s), with three warm-up batches and `n = max(32, floor(32768/d))`. Each block uses the ratio of the two-call geometric mean times. Reported speedup is the median of the three session medians; intervals use 5,000 hierarchical bootstrap resamples, resampling sessions and then blocks within each ABBA/BAAB order stratum. The timing columns are separate medians, so their quotient need not equal the paired speedup. All 48 paired blocks favored this candidate.

The timed operation is public `ScalarQuantizer::compute_codes`, using fixed-seed (718) finite input batches. Direct-u8 input values are `float(rng()%16777216)/65536.f`.

These results describe public encoding/distance throughput on one non-exclusive host, not end-to-end ANN search speedup. The intervals describe these sessions only, with no multiple-comparison correction; they do not establish portability across machines or vector lengths.

### Validation

- Native public encoding oracle: **7,724,670** legal-domain input elements across the five RISC-V rounding modes; zero differing output bytes or write-guard failures. Inputs include negative fractions, signed zeros, subnormals, all byte-value boundaries, and values just below 256.
- Original benchmark baseline full C++ suite: **273 passed, 7 skipped, 0 failed**.
- Submission version on `2ed4c106e9fb9686e7727e5daf8ad6ad1e164109`: all three independent candidate builds succeeded; this PR's focused C++ suite (`NONE`: 11 passed, 5 skipped, 0 failed; `RISCV_RVV`: 12 passed, 4 skipped, 0 failed) and the public-path oracle above pass on native RISC-V. The newly added RVV regression test is executed and passed in the RVV run, and skipped when RVV is disabled in the NONE run.
- All touched C++ files pass clang-format 21.1.8; `git diff --check` clean.

### Notes

- Touches three files: `faiss/impl/scalar_quantizer/quantizers.h` (one-line `final` → `override` so the generic encoder can be specialized), `faiss/impl/scalar_quantizer/sq-rvv.cpp` (the RVV encoder), and `tests/test_scalar_quantizer.cpp` (one regression test).
- The conversion uses `vfcvt.rtz.xu.f.v` (round toward zero, float to unsigned) rather than a signed conversion. The generic encoder is `code[i] = (uint8_t)x[i]`, which is a C++ float-to-unsigned-integral truncation, so round-toward-zero is the matching mode. Negative fractions greater than -1 truncate to 0 in both paths.
- Two narrowing shifts (`vnsrl` by 0) are used to go `u32m4 → u16m2 → u8m1` before the byte store, avoiding a separate saturating narrow instruction.
- The kernel is VLEN-agnostic: `e32m4` lanes with `vsetvl`-driven loop bounds. At VLEN=128 this processes 16 elements per iteration.
- Quantization is not clamped. Inputs outside `(-1, 256)` are outside the portable domain — the scalar path casts out-of-range floats to `uint8_t`, which is undefined behavior for values outside that range, so no portable contract exists to preserve there.

Co-authored-by: ihb2032 <hebome@foxmail.com>
Co-authored-by: lyd1992 <liuyudong@iscas.ac.cn>