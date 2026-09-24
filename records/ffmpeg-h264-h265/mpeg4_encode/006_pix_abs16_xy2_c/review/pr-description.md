avcodec/riscv: add RVV pix_abs16_xy2
==================================

Add an RVV implementation of the 16-wide diagonal half-pixel SAD used by
motion estimation. Select it through the existing pix_abs[0][3] entry for
RVV I32 with VLEN >= 128.

Reuse the horizontal sums between reference rows, process two rows per
iteration, and accumulate the absolute differences in 16-bit lanes before
a single widening reduction. RNU narrowing preserves the C implementation's
four-pixel rounding exactly.

Benchmarked on a Spacemit X100, VLEN=256, pinned to CPU 0, using GCC 15.2.0
and the same FFmpeg configuration for both builds.

Median of seven checkasm runs, 200 ms per function, 16x8 block, stride 64.
Values are nanoseconds per call:

```text
                                upstream       patched
pix_abs_0_3_c                     202.8           205.8
pix_abs_0_3_rvv_i32                  -              99.3
```

The RVV implementation is 2.04x faster than the separately built upstream
baseline, and 2.07x faster than the C reference in the patched binary.

Validation:

- Native motion checkasm: 1,000 consecutive seeds passed.
- Motion checkasm under QEMU at VLEN 128, 256, 512 and 1024: 100 seeds each
  passed, including forced all-ones agnostic tails.
- Isolated RV64 Zve32x/ELEN=32 and RV32 vector tests passed.
