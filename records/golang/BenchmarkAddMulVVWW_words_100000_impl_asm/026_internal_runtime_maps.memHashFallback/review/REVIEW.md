# Review: fixed-size map-key hashing on riscv64

## Provenance

- Project: Go
- Trace/Craft target: `internal/runtime/maps.memHashFallback`
- Original candidate: `craft/patch.diff`
- Reviewed result: `review/026_internal_runtime_maps.memHashFallback.patch`
- Final validation base: Go commit `a532f497dcd53f62bdf9890d0908f6c840a41efa`
- Final gate: `PASS_SUBMISSION_READY`

## Original optimization hypothesis

The trace-driven Craft candidate identified byte-wise little-endian loads in
`memHashFallback` as a RISC-V hashing hotspot. It replaced them with native
32- and 64-bit loads when the pointer was aligned.

The wide-load hypothesis was correct: aligned inputs became substantially
faster. The original implementation was not acceptable because alignment
checks were placed in the generic read path. Small, odd-sized, and unaligned
inputs therefore paid repeated dispatch costs and showed material regressions.

## Expert review and iteration

The patch was redesigned through measured iterations:

1. The original generic per-read dispatch proved the optimization potential
   but regressed general inputs.
2. A single function-entry alignment check removed repeated checks, but still
   penalized small and odd-sized keys.
3. The final design leaves generic `memHashFallback` unchanged.
4. On riscv64, the compiler selects a dedicated entry only for fixed 32-, 64-,
   and 128-byte map keys.
5. The dedicated helper uses native 64-bit loads for aligned keys and calls the
   portable fallback for a truly unaligned key pointer.
6. Other sizes, including 31 and 33 bytes, retain the original compiler and
   runtime path.

This changes the contribution from a broad runtime micro-optimization into a
compiler-directed fixed-size specialization.

## Final validation

- Full Go `all.bash`: pass.
- Focused correctness, patch identity, amd64/arm64 cross-build, and riscv64
  disassembly checks: pass.
- 15 alternating-order rounds; 210 paired samples.
- Six fixed-size target conditions: median improvement 21.0% to 37.0%;
  every target condition improved in 15/15 pairs.
- 31- and 33-byte controls: no reproducible regression.
- Direct unaligned fallback overhead: 2.69% to 5.11%, approximately
  1.58 to 2.25 ns/op, within the recorded acceptance bound.
- The final machine code contains native word loads and an explicit branch to
  the portable fallback for unaligned pointers.

## Review conclusion

The reviewed patch is a substantial benchmark-driven refactoring of the
original Craft candidate. The original patch supplied the hotspot and wide-load
hypothesis; failed general-path measurements determined the final
specialization boundary.

The final patch is submission-ready with the unaligned fallback cost disclosed.
Performance was measured on one RISC-V system, so upstream review should avoid
claiming identical gains on every microarchitecture.

