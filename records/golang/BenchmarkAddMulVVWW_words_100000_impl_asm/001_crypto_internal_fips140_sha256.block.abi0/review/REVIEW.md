# Review: SHA-256 Zbb scalar fallback

## Provenance

- Project: Go
- Trace/Craft target: `crypto/internal/fips140/sha256.block.abi0`
- Original candidate: `craft/patch.diff`
- Reviewed result: `review/001_crypto_internal_fips140_sha256.block.abi0.patch`
- Validation base: Go CL 752981 patch set 23, commit `760ebb7fc9663cac1eeb70739ec1268c193e1958`
- Contribution form: dependent follow-up to Go CL 752981

## Original optimization hypothesis

The trace-driven Craft candidate identified byte-at-a-time reconstruction of
the first 16 SHA-256 message words as a RISC-V scalar hot path. It proposed
using Zbb loads and byte reversal (`MOVWU + REV8 + SRL`) instead.

That hypothesis was valid: naturally aligned inputs showed a repeatable gain.
However, the original patch was not suitable for submission because it did not
safely preserve arbitrary input alignment and overlapped the abandoned Go CL
671295 approach.

## Expert review and iteration

The reviewed implementation keeps the wide-load idea but changes its safety
model:

1. Test the message pointer alignment once at function entry.
2. Use `MOVWU + REV8 + SRL` only for naturally 4-byte-aligned blocks.
3. Retain the original four-`MOVBU` reconstruction for unaligned input.
4. Interleave both loading paths with the same round body instead of
   duplicating the SHA-256 round implementation.

SHA-256 advances by 64 bytes per block, so an initially aligned pointer remains
aligned throughout the call. Intermediate designs were rejected because one
duplicated too much code and another regressed unaligned inputs.

## Final validation

- Correctness: the focused SHA-256 tests passed 20 times on both trees.
- Target: riscv64 with Zbb and without Zvknha; `GODEBUG=cpu.zvknha=off`.
- Upstream SHA-256 benchmark geomean: 2.84% faster; all 15 cases improved.
- Aligned cases: 2.20% to 3.01% faster.
- Unaligned cases: 0.23% to 1.02% faster; no measured regression.
- Code size: `blockScalar.abi0` grew by 388 bytes (4.08%).
- Disassembly contains both the Zbb wide-load path and byte-load fallback.

## Review conclusion

The final patch is a benchmark-driven refinement of the original Craft idea,
not an unrelated replacement. It fixes the alignment portability gap and
reduces code growth while preserving the measured benefit.

It is ready for submission review as a **dependent** Zbb-only follow-up.
Because CL 752981 is not yet merged, the patch is not standalone against the
current Go master. After the parent lands it must be rebased and receive a
focused correctness, disassembly, and performance recheck.

