# feat(rvv): add bit-exact RVV decode_vector for raw-codec quantizers

Second decode follow-up on top of #5535 and the codec-template decode PR.

## What this adds

`decode_vector` kernels for the raw-codec quantizers, so `ScalarQuantizer::decode()`
stops inheriting the scalar loop for:

- `QT_fp16` (`QuantizerFP16`), `QT_bf16` (`QuantizerBF16`),
  `QT_8bit_direct` (`Quantizer8bitDirect`), `QT_8bit_direct_signed`
  (`Quantizer8bitDirectSigned`)
- `quantizers.h`: `decode_vector` becomes overridable at those three sites
  (`bf16` and the Lloyd-Max types were already overridable)

## Design

- **The FP16 kernel is bit-exact with the scalar `decode_fp16` for arbitrary
  code words, not just encoder-produced ones.** The scalar inf/nan branch keeps
  the sign, the all-ones exponent and the shifted mantissa without quieting
  sNaN payloads, while an FP widening conversion would quiet them — so the
  conversion runs masked (NaN lanes excluded) and the NaN lanes are rebuilt in
  the integer domain. The kernel is unconditional, matching the Zvfhmin
  contract enforced at the top of `sq-rvv.cpp`.
- BF16 is a pure integer widen + shift. 8bitDirect is a u8 -> f32 widening
  conversion. 8bitDirectSigned flips the sign bit (`(code ^ 0x80)` as int8
  equals `code - 128`), then widens exactly.

## Performance (SG2044, d=128, n=2000, 20 iterations, single-threaded)

| benchmark            |   baseline |   this PR | speedup |
|----------------------|-----------:|----------:|--------:|
| QT_fp16              |    784,434 |   302,905 |   2.59x |
| QT_8bit_direct       |    454,681 |   210,671 |   2.16x |
| QT_bf16              |    506,924 |   243,206 |   2.08x |
| QT_8bit_direct_signed|    459,205 |   241,233 |   1.90x |

## Tests

Same story as the codec-template decode PR: the existing
`RVVFP16DistancePathParity` / `RVVDistancePathParity` coverage already includes
these types and flips to the RVV kernels when this lands. Hardware-validated on
SG2044 (bit-identical recons errors and idempotence).

### Co-authors

- ihb2032 <hebome@foxmail.com>
- lyd1992 <liuyudong@iscas.ac.cn>
- Yuansheng <yuansheng@isrc.iscas.ac.cn>
