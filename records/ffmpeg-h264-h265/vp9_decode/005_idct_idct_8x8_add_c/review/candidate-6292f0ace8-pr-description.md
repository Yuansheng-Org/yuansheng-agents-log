avcodec/riscv: add RVV VP9 8x8 inverse DCT

Add an RVV implementation of the 8-bit VP9 8x8 DCT_DCT inverse
transform. The function includes the DC-only path, clears the input
coefficients, and is selected only when Zve32x is available with a
vector length of at least 128 bits.

The full transform keeps 32-bit multiply and rounding operations in
vector registers and uses a segment store for the transposed 16-bit
intermediate block. The DC-only path uses eight-byte vector loads and
saturating adds or subtracts.

Tested on a SpacemiT X100 with VLEN=256 using GCC 15.2.0. Existing VP9
checkasm coverage passed for 100 successive repetitions, and the
complete VP9 DSP checkasm test passed for 10 repetitions. Five VP9 FATE
vectors covering quantizer extremes, sharpness, tiling, and trac3849
also passed. Frame MD5 output matched the C baseline for 1,000-frame
640x360 and 5,000-frame 352x288 8-bit VP9 streams.

Mean checkasm results over 30 runs, in nanoseconds per call:

                               C       RVV    improvement
  8x8 DCT_DCT, eob=1       72.85     64.92       10.92%
  8x8 DCT_DCT, sub2       321.45    295.17        8.17%
  8x8 DCT_DCT, sub4       321.43    295.38        8.10%
  8x8 DCT_DCT, full       321.48    295.62        8.04%

Single-threaded decoding reduced cycles by 0.40% on the 640x360 stream
and by 0.21% on the 352x288 stream. CPU frequency was fixed at 2.2 GHz,
the process was pinned to one X100 core, and baseline and patched runs
were interleaved.

No test files, public interfaces, generic code, or non-RISC-V sources
are changed. An x86_64 VP9-only build from the same source revision also
completed successfully.
