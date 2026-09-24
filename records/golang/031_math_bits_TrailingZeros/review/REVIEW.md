# Review: RISC-V Zbb TrailingZeros intrinsics

## Provenance

- Project: Go
- Actual original Craft source:
  `craft/go/028_nextFreeFast/craft/patch.diff`
- Corrected contribution identity: `math/bits.TrailingZeros*`
- Reviewed result: `review/031_math_bits_TrailingZeros.patch`
- Final validation base: Go commit `d6eb7bd6aee56dd33e0542b56a507b028692db20`
- Final gate: `PASS_SUBMISSION_READY`

The original Craft directory was misnamed: its trace target was
`nextFreeFast`, but the generated diff modified compiler intrinsics for
`TrailingZeros*` and `OnesCount*`. The original diff is preserved here under
the corrected contribution directory so the lineage remains auditable.

## Original optimization hypothesis

The original candidate observed that Zbb provides `CTZ`/`CTZW`, while
`rva20u64` cannot assume Zbb at compile time. It proposed a runtime CPU-feature
dispatch so public bit operations could still use Zbb when available.

The candidate was too broad:

- it bundled `OnesCount*` changes unrelated to the validated contribution;
- it also routed `internal/runtime/sys.TrailingZeros*` aliases through the
  runtime feature check;
- that runtime aliasing reproducibly regressed 13 focused map and allocator
  controls.

## Expert review and iteration

The final patch narrows and hardens the idea:

1. Retain only public `math/bits.TrailingZeros{8,16,32,64}` dispatch.
2. On `rva20u64`, use a feature-flag branch with `CTZ`/`CTZW` fast paths
   and the pure-Go fallback.
3. On `rva22u64`, emit direct `CTZ`/`CTZW` because Zbb is guaranteed.
4. Keep `internal/runtime/sys` aliases generic on `rva20u64`, removing the
   runtime hot-path regression.
5. Add intrinsic-table tests and RISC-V asmcheck coverage for rva20/rva22.

## Final validation

- Full latest-master `all.bash`: pass.
- Forced `GODEBUG=cpu.zbb=off`: pass.
- amd64/arm64 cross-builds and RISC-V asmcheck: pass.
- Five target benchmarks, 15 paired rounds each: 75/75 pairs improved.
- Median improvements: 8.405% to 51.624%.
- The 13 former regression controls: 0 classified regressions.
- rva20 emits feature dispatch plus fallback; rva22 emits direct Zbb
  instructions.

## Review conclusion

The final contribution was extracted from a target-mismatched Craft candidate.
It preserves the useful Zbb dispatch hypothesis, removes unrelated
`OnesCount*` changes, and fixes the runtime regression exposed by validation.

This is therefore a narrowed and tested refinement of the original patch, not a
new patch without provenance. The final result is submission-ready.

