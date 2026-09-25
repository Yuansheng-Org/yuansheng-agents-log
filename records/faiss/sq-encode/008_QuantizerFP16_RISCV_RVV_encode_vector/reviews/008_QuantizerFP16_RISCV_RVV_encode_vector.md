# feat(rvv): add bit-exact RVV encode_vector for raw-codec quantizers

Encode follow-up for the raw-codec quantizers.

## What this adds

`encode_vector` kernels for `QuantizerFP16`, `QuantizerBF16`,
`Quantizer8bitDirect` and `Quantizer8bitDirectSigned` (`QT_fp16`, `QT_bf16`,
`QT_8bit_direct`, `QT_8bit_direct_signed`); `quantizers.h` makes
`encode_vector` overridable at those three sites.

## Bit-exactness

- **FP16 encode is an op-for-op vector translation of the scalar
  `fp16-inl.h` `encode_fp16`** (RNE rounding through a single scaled float
  multiply, overflow clamped to 65504, NaN -> 0x7e00, Inf -> 0x7c00). Zvfhmin
  narrowing conversions round differently and do not reproduce the NaN payload
  handling, so the kernel stays in the integer / f32 domain and emits no FP16
  vector instructions.
- BF16 encode is the integer `(bits + 0x8000) >> 16` carry-round — exact for
  every input including NaN (which may round up to Inf, exactly like the
  scalar).
- The direct codecs rely on RISC-V float->unsigned conversion saturating and
  mapping NaN to all ones, which matches the scalar fcvt the references
  compile to; the narrowing store keeps the low byte like the scalar casts.

## Performance (SG2044)

The direct-codec kernels land in the 8bit-family encode gains; fp16/bf16 encode
measured flat within run-to-run noise. All `ndiff_for_idempotence` = 0 and
recons errors bit-identical to scalar.

## Tests

`RVVEncodePathParity` extends to the four raw-codec types (QT_fp16 stays in the
main encode test rather than being split off like the decode test: the encode
kernels emit no Zvfhmin instructions, so plain rv64gcv suffices).

### Co-authors

- ihb2032 <hebome@foxmail.com>
- lyd1992 <liuyudong@iscas.ac.cn>
- Yuansheng <yuansheng@isrc.iscas.ac.cn>
