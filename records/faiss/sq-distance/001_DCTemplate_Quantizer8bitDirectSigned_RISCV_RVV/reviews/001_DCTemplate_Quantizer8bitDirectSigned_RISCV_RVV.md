RVV `QT_8bit_direct_signed` distances currently fall through to the generic `DCTemplate`. This adds RVV specializations for L2 and inner product, covering both floating-point query-to-code and code-to-code distances.

The kernel reconstructs `code[i] - 128` in float, accumulates in vector registers, and performs a single reduction after the loop. Tail-undisturbed accumulation preserves earlier lanes when the final iteration is short. Query values remain floating point, including fractional queries — the query is not truncated to the integer byte domain.

### Relationship to other RVV scalar-quantizer PRs

This is an independent change based on `main`. It overlaps the direct-signed distance area of #5539, and #5535 also works on RVV scalar-quantizer primitives. #5539 also specializes `Quantizer8bitDirectSigned`, so the difference is worth stating explicitly: #5539's `set_query` truncates the query via `int(x[i]) + 128`, which changes results for fractional queries. This PR keeps the floating-point query path, matching the current RVV baseline (which falls back to the `SIMDLevel::NONE` `DCTemplate`, whose `set_query` stores the `const float*` directly) and `SimilarityL2<SIMDLevel::NONE>::add_component(float)`.

For the vectors used in this PR's test, the two approaches disagree:

| Case | This PR (float) | Integer-truncated | Equal |
|---|---|---:|---|
| `d=1`, `code=0`, `q=-127.75`, L2 | `0.0625` | `1` | no |
| `d=1`, `code=128`, `q=-0.5`, L2 | `0.25` | `0` | no |

On x86, `sq-dispatch.h` routes this qtype to `DistanceComputerByteSigned<AVX512_SPR>` (integer domain, query re-biased by `+128`) only when `d % 64 == 0`, and otherwise falls back to `DCTemplate` — so both semantics are reachable upstream depending on dimension. This PR keeps the `DCTemplate` entry point, i.e. the floating-point / all-dimensions path.

The benchmark below compares with the stated official baseline and does not compare against either PR's head.

### Performance

Measured on a native SG2044 RISC-V host (VLEN=128), GCC 15.1, Release `-O3`, dynamic dispatch (`FAISS_OPT_LEVEL=dd`), `rv64gcv_zvfhmin/lp64d`, one thread pinned to CPU 2. The baseline is official commit `80a16564f86530dbf0bfaf96c2b71feffeb5093f`; the candidate is that same baseline plus only this kernel change. These measurements were not collected on the newer PR base `2ed4c106e9fb9686e7727e5daf8ad6ad1e164109`. The affected scalar-quantizer source files are unchanged between those bases, and the submitted kernel differs from the measured one only in comments/formatting.

| Public path | Dimension | Baseline ns/element | Candidate ns/element | Paired speedup | 95% interval |
|---|---:|---:|---:|---:|---:|
| IP query-to-code | 16 | 3.341 | 2.162 | 1.545x | [1.537, 1.550] |
| IP query-to-code | 32 | 2.963 | 1.236 | 2.400x | [2.145, 2.406] |
| IP query-to-code | 128 | 2.745 | 0.543 | 5.058x | [4.887, 5.084] |
| IP query-to-code | 768 | 2.628 | 0.360 | 7.261x | [7.213, 7.371] |
| IP code-to-code | 16 | 3.524 | 2.419 | 1.457x | [1.456, 1.458] |
| IP code-to-code | 32 | 3.060 | 1.462 | 2.095x | [2.092, 2.096] |
| IP code-to-code | 128 | 2.769 | 0.749 | 3.700x | [3.693, 3.701] |
| IP code-to-code | 768 | 2.635 | 0.563 | 4.701x | [4.685, 4.706] |
| L2 query-to-code | 16 | 3.410 | 2.248 | 1.515x | [1.512, 1.521] |
| L2 query-to-code | 32 | 3.002 | 1.321 | 2.269x | [2.251, 2.279] |
| L2 query-to-code | 128 | 2.754 | 0.611 | 4.493x | [4.457, 4.512] |
| L2 query-to-code | 768 | 2.630 | 0.428 | 6.187x | [6.103, 6.230] |
| L2 code-to-code | 16 | 3.592 | 2.461 | 1.460x | [1.459, 1.462] |
| L2 code-to-code | 32 | 3.100 | 1.499 | 2.069x | [2.066, 2.070] |
| L2 code-to-code | 128 | 2.778 | 0.777 | 3.573x | [3.568, 3.581] |
| L2 code-to-code | 768 | 2.635 | 0.586 | 4.497x | [4.470, 4.529] |

Geometric mean of the dimension-specific speedups at d=32/128/768: IP query-to-code **4.450x**, IP code-to-code **3.315x**, L2 query-to-code **3.981x**, L2 code-to-code **3.215x**.

There are three consecutive sessions, each with four alternating ABBA/BAAB blocks per dimension/path: **192 paired blocks** for this candidate. A is the baseline and B is the candidate. Each call is calibrated to at least 0.1 s (the shortest formal call in the full campaign was 0.192 s), with three warm-up batches and `n = max(32, floor(32768/d))`. Each block uses the ratio of the two-call geometric mean times. Reported speedup is the median of the three session medians; intervals use 5,000 hierarchical bootstrap resamples, resampling sessions and then blocks within each ABBA/BAAB order stratum. The timing columns are separate medians, so their quotient need not equal the paired speedup. All 192 paired blocks favored this candidate.

The timed operations are public `SQDistanceComputer::query_to_code` and `symmetric_dis` calls over randomly generated codes (fixed seed 718), with fractional floating-point queries.

These results describe public encoding/distance throughput on one non-exclusive host, not end-to-end ANN search speedup. The intervals describe these sessions only, with no multiple-comparison correction; they do not establish portability across machines or vector lengths.

### Validation

- Native public-path oracle: **13,152** distance/index checks; L2/IP, query-to-code, code-to-code, top-k, fractional queries, NaN/Inf, and tails. Float results are checked against a double reference with a dimension-dependent error bound; integer code-to-code sums are required to be exact when the sum of absolute terms is at most 2^24.
- Original benchmark baseline full C++ suite: **273 passed, 7 skipped, 0 failed**.
- Submission version on `2ed4c106e9fb9686e7727e5daf8ad6ad1e164109`: all three independent candidate builds succeeded; this PR's focused C++ suite (`NONE`: 11 passed, 5 skipped, 0 failed; `RISCV_RVV`: 12 passed, 4 skipped, 0 failed) and the public-path oracle above pass on native RISC-V. The newly added RVV regression test is executed and passed in the RVV run, and skipped when RVV is disabled in the NONE run.
- All touched C++ files pass clang-format 21.1.8; `git diff --check` clean.

The added regression test `ScalarQuantizer.RVVDirectSignedDistances` covers byte boundaries (`0, 127, 128, 255`), fractional queries, both metrics, symmetric distances in both directions, and dimensions around vector tails (`1, 15, 16, 17, 31, 32, 33, 65`). It skips when RVV is unavailable.

### Notes

- Only `faiss/impl/scalar_quantizer/sq-rvv.cpp` and `tests/test_scalar_quantizer.cpp` are touched.
- The implementation is VLEN-agnostic: it uses `e8m1` loads and an `e32m4` accumulator with `vsetvl`-driven loop bounds, so the vector length adapts at runtime rather than assuming VLEN=128.
- Correctness relies on tail-undisturbed (`_tu`) accumulation, so the final short iteration does not overwrite earlier lanes.
