# feat(rvv): add bit-exact RVV decode_vector for codec-template quantizers

Builds on #5535 (please merge that first). This is the first of the decode/encode
follow-ups that give the standard quantizer types the RVV kernels #5535 added for
the Lloyd-Max types.

## What this adds

`ScalarQuantizer::decode()` currently inherits the scalar `decode_vector` from the
NONE base even when an RVV quantizer is selected. This PR adds the codec-template
decode kernels:

- `QuantizerTemplate<Codec, {UNIFORM, NON_UNIFORM}, RISCV_RVV>::decode_vector`,
  built on the `Codec::decode_m8_components` primitives from #5535
- covers `QT_8bit`, `QT_8bit_uniform`, `QT_4bit`, `QT_4bit_uniform`
- `quantizers.h`: the scalar NONE bases' `decode_vector` becomes overridable
  (`final` -> `override`) at the two codec-template sites — the only change
  outside `sq-rvv.cpp`

## Design

- **Bit-exactness is the contract.** The kernels keep the scalar reference's
  separate multiply and add (`vfmul` then `vfadd`, no `vfmadd` fusion) so the
  rounding matches the scalar expression exactly. Verified on hardware:
  `sql2_recons_error` and `ndiff_for_idempotence` are bit-identical to the
  scalar path for every quantizer below.
- **QT_6bit stays scalar (both directions).** The 6-bit codec provides no RVV
  kernels: its gather-based decode measured 0.51x on SG2044 (micro-coded indexed
  loads — the same root cause as the 6-bit distance fallback settled on #5535),
  and the 6-bit encode measured 0.78–0.88x. The `codec_has_decode_m8_v`
  constraint routes it to the scalar QuantizerTemplate fallback.
- All kernels are single strip-mined `while` loops driven by `vsetvl`, with
  m8-class vectors.

## Performance (SG2044, d=128, n=2000, 20 iterations, single-threaded)

Timed loop body is a single `sq.decode()` call over the whole 2000-vector batch;
stat and record phases agree.

| benchmark       |   baseline |   this PR | speedup |
|-----------------|-----------:|----------:|--------:|
| QT_8bit_uniform |  1,334,617 |   410,968 |   3.25x |
| QT_8bit         |  1,342,880 |   469,559 |   2.86x |
| QT_4bit         |  1,379,050 |   689,926 |   2.00x |
| QT_4bit_uniform |  1,304,938 |   736,746 |   1.77x |

## Tests

No new test code: `RVVDistancePathParity` from #5535 already lists these types,
so landing this PR flips them from the scalar fallback to the RVV kernels under
the existing tests, including the CI QEMU runs at VLEN=128 and VLEN=1024.
Hardware-validated on SG2044 (recons errors and idempotence bit-identical).

### Co-authors

- ihb2032 <hebome@foxmail.com>
- lyd1992 <liuyudong@iscas.ac.cn>
- Yuansheng <yuansheng@isrc.iscas.ac.cn>
