# feat(rvv): add bit-exact RVV encode_vector for codec-template quantizers

First encode follow-up on #5535 (decode direction landed in the two previous PRs).
Gives `ScalarQuantizer::compute_codes()` / `sa_encode()` RVV kernels for the
codec-template types.

## What this adds

- `Codec8bit` / `Codec4bit` gain `encode_m8_components` kernels plus an
  `encode_alignment` trait (1 / 2 components per packed byte)
- `QuantizerTemplate<Codec, {UNIFORM, NON_UNIFORM}, RISCV_RVV>::encode_vector`
  covering `QT_8bit`, `QT_8bit_uniform`, `QT_4bit`, `QT_4bit_uniform`
- `quantizers.h`: the scalar bases' `encode_vector` becomes overridable at the
  two codec-template sites

## Bit-exactness

- The normalization clamps are masked merges (`vmflt` / `vmfgt` / `vmfne`), not
  `vfmin`/`vfmax`: those would resolve NaN lanes to a number, while the scalar
  branches leave NaN untouched — NaN flows into the codec and both paths turn
  it into the same fcvt all-ones code. `vmfne` reproduces the C `!=` semantics
  for the non-uniform `vdiff[i] != 0` check.
- `Codec4bit`'s `(int)(x * 15.0)` literal is a double, so the scalar reference
  multiplies in double precision; the product is exact for a float `x`, and the
  kernel evaluates it exactly in the integer domain:
  `floor(15 * x) = (15 * m) >> (150 - e)` with
  `m = (bits & 0x7fffff) | 0x800000` (the exponent is stripped before OR-ing
  the implicit bit). `Codec8bit`'s `255` literal is an int, so that kernel
  keeps the float multiply + round-toward-zero conversion.
- Encoded sub-byte packs are shared across components, so chunks are kept
  pack-aligned via the `encode_alignment` trait; misaligned tails of fewer than
  ALIGN components fall back to the inherited scalar reference.
- **QT_6bit stays scalar for both directions** (see the decode PRs; the 6-bit
  encode kernel measured 0.78–0.88x on hardware and is intentionally dropped).

## Performance (SG2044, d=128, n=2000, single-threaded batch encode)

- QT_8bit family (8bit / 8bit_uniform): **1.94–2.38x**, geomean 2.22x
- QT_4bit / QT_4bit_uniform: 1.06–1.14x (correct, modest)
- Whole-process counters: instructions −66%, branches −77%
- All `ndiff_for_idempotence` = 0; recons errors bit-identical to scalar

## Tests

- New `RVVEncodePathParity`: encodes the same vectors through the NONE and
  RISCV_RVV quantizers into zeroed buffers and requires every code byte to
  match — the encode kernels are bit-exact by construction. Inputs include
  extreme values (out-of-range, denormals, NaN, ±Inf, the fp16 65504 overflow
  boundary), dimensions around the VLMAX boundaries.
- CI: the two existing QEMU SQ parity steps (VLEN=128 and VLEN=1024) extended
  with the `*EncodePathParity*` filter.

### Co-authors

- ihb2032 <hebome@foxmail.com>
- lyd1992 <liuyudong@iscas.ac.cn>
- Yuansheng <yuansheng@isrc.iscas.ac.cn>
