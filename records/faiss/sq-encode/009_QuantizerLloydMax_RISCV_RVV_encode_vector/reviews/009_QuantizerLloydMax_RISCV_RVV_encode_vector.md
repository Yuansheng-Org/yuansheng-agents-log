# feat(rvv): add bit-exact RVV encode_vector for LloydMax quantizers

Last follow-up of the series: the encode direction for the Lloyd-Max quantizers
(the `QT_*_eden` and `QT_*_tqmse` types, which alias `QuantizerLloydMax`).

## What this adds

`QuantizerLloydMax<1/2/3/4/8, RISCV_RVV>::encode_vector`. No header changes —
the scalar base's `encode_vector` was already overridable.

## Design

- The per-component `std::upper_bound` over the trained boundaries becomes a
  vectorized bisection (`select_index_m8`): each round gathers
  `boundaries[mid]` with an indexed load, compares, and moves lo/hi per lane;
  converged lanes clamp their gather index to K-2 and are masked out of the
  updates. NaN queries collapse to index 0 on both paths (every
  `boundaries[mid] <= NaN` comparison is false).
- Packings: 1/2/3-bit reuse a gather-OR-shift packing primitive over 24-bit
  little-endian groups (the 3-bit layout is the same packing the scalar 6-bit
  codec uses, with 3-bit values); 4-bit mirrors Codec4bit's nibble packing;
  8-bit stores one byte per component.
- Chunks stay pack-aligned (8/4/8/2/1 components) so every packed byte is owned
  by exactly one chunk — the scalar path OR-accumulates into a buffer the
  callers zero. Sub-byte misaligned tails fall back to the inherited scalar
  reference.

## Tests

`RVVEncodePathParity` extends to the ten eden/tqmse types (the tqmse types
alias the EDEN kernels; listing them keeps the dispatch coverage explicit).
Bit-exactness is checked for every code byte, including NaN / ±Inf / denormal
queries.

### Co-authors

- ihb2032 <hebome@foxmail.com>
- lyd1992 <liuyudong@iscas.ac.cn>
- Yuansheng <yuansheng@isrc.iscas.ac.cn>
