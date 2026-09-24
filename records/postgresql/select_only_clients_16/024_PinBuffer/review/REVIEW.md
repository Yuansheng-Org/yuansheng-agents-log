# Review: RV64 single-copy atomic reads for PinBuffer

## Provenance

- Project: PostgreSQL
- Trace/Craft target: `PinBuffer`
- Original candidate: `craft/patch.diff`
- Reviewed result: `review/024_PinBuffer.patch`
- Validation base: PostgreSQL commit `86f7c82cf1023e3599f40f939727791a7090cd44`
- Final gate: `READY_FOR_SUBMISSION_REVIEW`

## Original optimization hypothesis

The Craft candidate identified the lock-buffer reference-count read in
`PinBuffer` as paying for an `lr.d`/`sc.d` compare/exchange loop. On RV64,
a naturally aligned XLEN-width load has the single-copy atomicity required by
PostgreSQL's existing `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY` contract, so the
read can compile to a single `ld`.

The original patch added the architecture include in `atomics.h`, but omitted
the referenced `arch-riscv.h`. It was therefore incomplete and could not be
submitted as a self-contained change.

## Expert review and iteration

The reviewed patch completes and scopes the architecture support:

1. Keep the RISC-V include hook in `atomics.h`.
2. Add `port/atomics/arch-riscv.h`.
3. Define `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY` only when
   `__riscv_xlen == 64`.
4. Leave RV32 and non-RISC-V builds on the existing generic fallback.
5. Preserve actual compare/exchange update operations; only the no-barrier
   atomic read becomes a plain load.

## Final validation

- Configure, build, install, and PostgreSQL regression: 240/240 pass.
- Disassembly: the target read changes from an `lr.d`/`sc.d` loop to
  `ld`; update CAS paths remain present.
- Extended pgbench: 13 conditions, 10 pairs each, 130 pairs total.
- 114/130 pairs improved; median improvement 1.665%.
- Bootstrap 95% CI for the median: [1.358%, 1.897%].
- 80-client/40-thread/60-second select and read/write soak: zero failed
  transactions.
- Persistent balance invariants and `pg_amcheck`: pass.

## Review conclusion

The final patch is the self-contained completion of the original Craft idea.
Its contribution is primarily correctness and portability scoping: RV64 gains
the intended single-load read while RV32 and other architectures retain their
established fallback.

The result is ready for submission review. Performance evidence comes from one
RV64 system and should be presented as validation of the optimization, not as a
universal hardware guarantee.

