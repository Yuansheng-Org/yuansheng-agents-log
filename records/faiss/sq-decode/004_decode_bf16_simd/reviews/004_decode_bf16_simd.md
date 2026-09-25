## Summary

Add a RISC-V Vector (RVV) implementation to `decode_bf16_simd()`.

The RVV path preserves the existing bit-level conversion semantics:

1. load BF16 values as unsigned 16-bit integers;
2. zero-extend them to unsigned 32-bit integers;
3. shift left by 16 bits;
4. reinterpret the resulting bits as FP32.

Inputs shorter than 16 elements retain the scalar path to avoid short-vector overhead. Larger inputs use fixed-VL RVV chunks followed by a runtime-VL tail.

## Validation

Tested on native RISC-V hardware with GCC 15.3.0 and `-march=rv64gcv -mabi=lp64d`, based on Faiss commit `b5a14632a6f2bfdc824c86d683b9d6e9205c2604`.

Correctness:

- exhaustively tested all 65,536 BF16 bit patterns;
- tested 31 input sizes, including zero, short inputs, vector boundaries, tails, and large arrays;
- checked guard elements for output overwrites;
- zero correctness failures.

Representative microbenchmark speedups over the scalar implementation:

| Elements | Speedup |
|---:|---:|
| 7 | 1.01x |
| 16 | 1.58x |
| 31 | 2.43x |
| 64 | 3.46x |
| 128 | 3.88x |
| 1024 | 4.38x |
| 4096 | 4.37x |
| 65536 | 4.30x |

## Scope

This change only modifies `faiss/utils/bf16.h`. It is independent of the active ScalarQuantizer RVV changes in PRs #5535 and #5539.

## Notes

- The `n == 1` fast path and the `n < 16` scalar loop exist because at VLEN=128 an `e16m1` chunk covers only 8 elements, and for very short inputs the `vsetvl`/load/extend/store sequence costs more than the equivalent scalar work. The measured crossover is at 16 elements.
- The bulk loop uses a hoisted `vsetvlmax_e16m1()` so `vsetvl` is not re-executed per iteration; only the final partial chunk calls `vsetvl` again.
- `vuint16m1` → `vuint32m2` uses `vzext.vf2` (LMUL widening), so the 32-bit shift and the FP32 store run at `m2`. This keeps register pressure low while doubling the elements per chunk.
- No rounding or saturation is involved: BF16 → FP32 is an exact bit widening, so the vector path is bit-identical to the scalar path for every input, including NaN payloads and subnormals.
- The guard is `#elif defined(__riscv_vector)` inside the existing arch dispatch, so the AVX2/AVX512 paths are untouched and a build without RVV falls through to the scalar loop unchanged.

Co-authored-by: ihb2032 <hebome@foxmail.com>
Co-authored-by: lyd1992 <liuyudong@iscas.ac.cn>