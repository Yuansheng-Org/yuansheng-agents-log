# avcodec/riscv: add RVV VP9 16x16 inverse DCT

RISC-V currently uses the C implementation for the 8-bit VP9 16x16 DCT_DCT inverse transform.

Add RVV DC-only and full 16x16 paths. The full transform processes a variable number of columns with e32,m1, writes the first pass transposed to a 16-bit stack buffer, clears the coefficient block, and combines the rounded second pass with 8-bit destination pixels. Dispatch requires RVV_I32 and a VLEN of at least 128 bits.

No public interface, generic code, test file, bit depth, or transform type is changed.

## Verification

Built from `cb5a63736a01a9673b63fffcd7dee025490ae376` with GCC 15.2.0 on riscv64. The following passed:

- complete RISC-V build and checkasm build;
- `vp9_inv_dct_dct_16x16*` checkasm with 100 successive seeds: 600/600 tests;
- full VP9 DSP checkasm with 10 successive seeds: 710/710 tests;
- five available VP9 FATE vectors covering quantizer extremes, sharpness, tiling and trac3849;
- frame MD5 comparison against the unmodified baseline for 1000-frame 640x360 and 4997-frame 352x288 yuv420p streams;
- isolated x86_64 build with the VP9 decoder enabled.

No test files are modified because the existing VP9 checkasm test covers destination output, coefficient clearing, and sub1/sub2/sub4/sub8/sub12/sub16 inputs.

## Performance

Measurements used CPU 2 of a SpacemiT X100 with VLEN=256, pinned at 2.2 GHz. Each checkasm function was measured for 50 ms across 30 seeds.

| Function | C (ns) | RVV (ns) | Mean gain | Worst run |
|---|---:|---:|---:|---:|
| 16x16 sub1 | 248.0667 | 35.8962 | 85.5369% | 84.3956% |
| 16x16 sub2 | 1840.9210 | 1338.4056 | 27.2934% | 25.0902% |
| 16x16 sub4 | 1842.4951 | 1338.3531 | 27.3571% | 25.0771% |
| 16x16 sub8 | 1845.1215 | 1338.4817 | 27.4531% | 24.9493% |
| 16x16 sub12 | 1844.4091 | 1338.3084 | 27.4347% | 25.0780% |
| 16x16 sub16 | 1844.7091 | 1338.3472 | 27.4423% | 24.9990% |

Single-threaded decoding of the 640x360 stream reduced cycles by 0.4493% on average over 10 interleaved runs; every run was faster and instructions fell by 1.2706%. On the 352x288 stream, instructions fell by 0.2604%, while the 0.1352% mean cycle reduction remained within timing noise.
