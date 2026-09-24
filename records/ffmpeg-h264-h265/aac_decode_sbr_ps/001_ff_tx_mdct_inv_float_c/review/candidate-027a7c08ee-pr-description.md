# avutil/riscv: add an RVV inverse float MDCT

Add an out-of-place inverse float MDCT codelet for RV64 with the
zve32f and zve64x extensions.

The folding and twiddle stages use vector-length-agnostic loops.  The
common four-byte input stride has an e32/m4 indexed-load path, while
arbitrary strides use e64/m4 indices.  Complex products use fused
multiply-add instructions.  The post-twiddle stage uses segmented loads
and stores for interleaved complex data.  Transforms shorter than 64
points keep the C codelet because RVV setup costs more than it saves at
those lengths.

The codelet reuses the existing float MDCT initialization and sub-FFT.
It requires RVV F32 during codelet selection and checks RVV I64 in the
initializer.  It is registered as aligned with a priority that leaves
non-power-of-two transforms to the existing PFA codelets.  No public API
or test file is changed.

Tested against FFmpeg 590fb407b09bbf19aac0ea2fbd0ba16f93494601 on a
SpacemiT RV64 system with VLEN=256 and GCC 15.2.0.  `checkasm` passed
20 seeds with the standard lengths, 20 seeds with 64 through 16384
power-of-two lengths, and 20 seeds with an eight-byte input stride.
The available FATE subset and AAC, AC-3, and Vorbis decoding also pass.

`checkasm --duration=100000`, 15 runs, pinned to one 2.2 GHz CPU:

| iMDCT length | C (ns) | RVV (ns) | speedup |
|---:|---:|---:|---:|
| 64 | 364.987 | 321.867 | 1.134x |
| 128 | 843.280 | 734.620 | 1.148x |
| 256 | 1880.480 | 1635.207 | 1.150x |
| 512 | 4170.707 | 3650.040 | 1.143x |
| 1024 | 9118.687 | 8061.040 | 1.131x |
| 2048 | 19699.447 | 17778.987 | 1.108x |
| 4096 | 43903.007 | 39561.213 | 1.110x |
| 8192 | 96915.687 | 89660.700 | 1.081x |
| 16384 | 227891.460 | 216959.993 | 1.050x |

Single-thread end-to-end decode tests, ten order-rotated runs after
warm-up, improved AAC by 1.018x, AC-3 by 1.136x, and Vorbis by 1.033x.
