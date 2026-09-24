# Review: bounded RVV scan in pq_getmsgstring

## Provenance

- Project: PostgreSQL
- Trace/Craft target: `pq_getmsgstring`
- Original candidate: `craft/patch.diff`
- Reviewed result: `review/030_pq_getmsgstring.patch`
- Validation base: PostgreSQL commit `86f7c82cf1023e3599f40f939727791a7090cd44`
- Final gate: PASS, technically submission-ready

## Original optimization hypothesis

The Craft candidate identified NUL search in protocol strings as a vectorizable
RISC-V hot path and proposed an RVV scan.

The direction was valuable, but the original patch was unsafe and
non-submittable:

- it introduced duplicate `pq_getmsgstring()` definitions;
- it changed API/return semantics;
- it used an unbounded fault-only-first scan rather than respecting the
  remaining message length;
- malformed input without a NUL terminator could over-read instead of following
  PostgreSQL's existing error path.

## Expert review and iteration

The final patch was redesigned around protocol bounds:

1. Add a private helper shared by `pq_getmsgstring()` and
   `pq_getmsgrawstring()`.
2. On RVV builds with `riscv_vector.h`, scan with LMUL=8.
3. Limit every vector load's VL to the message bytes still available.
4. Preserve the missing-NUL error behavior and character-set conversion
   semantics.
5. Keep the existing `strlen` implementation as the non-RVV fallback.
6. Avoid public header or API changes; the patch changes one C source file.

Several failed attempts were classified as harness failures rather than patch
failures, including root-owned regression setup and copied-build-tree path
issues. Canonical validation used corrected non-root and short-path runs.

## Final validation

- riscv64 RVV regression: baseline GCC 14 and candidate GCC 14/GCC 15/Clang
  17 all passed 240/240.
- x86_64 fallback: GCC and Clang both passed 240/240.
- Guard-page, missing-NUL, length, and unaligned test matrices: pass.
- Disassembly confirms `vsetvli e8,m8`, `vle8.v`, compare, and first-match
  vector operations.
- End-to-end 4 KiB simple-query workload: 24 paired rounds, 23/24 positive;
  median TPS improvement 3.59%, bootstrap 95% CI [3.17%, 3.89%].
- Balanced microbenchmarks across 55 length/alignment conditions: median RVV
  speedup 3.34x versus `strlen` and 3.97x versus `memchr`.
- All measured transactions completed without failure.

## Review conclusion

The final patch preserves the original vector-search insight but replaces its
unsafe implementation with a bounded, API-compatible design. It is a
correctness-driven rewrite derived from the same trace hotspot, not a cosmetic
edit of the Craft diff.

The result is technically submission-ready. Remaining risk is limited to
generality of performance measurements and upstream design preference; it is
not a known correctness or fallback failure.

