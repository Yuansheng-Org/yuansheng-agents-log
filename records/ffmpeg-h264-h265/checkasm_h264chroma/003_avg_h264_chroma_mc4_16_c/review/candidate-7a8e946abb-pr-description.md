# lavc/riscv: add RVV high-bit-depth H.264 chroma MC for 8xH blocks

RISC-V currently uses the C chroma motion-compensation functions for H.264 bit depths above 8. On a SpacemiT X100, the 8xH functions take a measurable share of High 10 decoding time.

The RVV put and avg functions handle zero offsets as copies, use two weighted sources for horizontal or vertical interpolation, and reuse the middle source row when processing two bilinear rows. A single-row tail handles odd heights. Dispatch is limited to bit depths 9–16 with RVV_I32 and VLEN of at least 128 bits. The 4xH and 2xH functions retain the existing C implementation.

## Verification

Built from FFmpeg master b894a6f7c1c1af8cc1b6c5d949ba2e5ae2ab9b02 with GCC 15.2.0 on riscv64. The following passed:

- make fate-checkasm-h264chroma
- checkasm --test=h264chroma: 10/10 tests
- 27,648 independent cases covering 9, 10, 12 and 16 bit samples; 2/4/8 widths; heights 1, 2, 3, 4, 8 and 16; put/avg; all x/y offsets from 0 to 7; and random and extreme inputs
- Frame CRC comparison against the unmodified baseline for a 900-frame, 640×360 yuv420p10le High 10 stream: identical output

## Performance

Measurements used CPU 0 of a SpacemiT X100 (VLEN=256), pinned at 2.2 GHz. The same configure options and GCC 15.2.0 were used for baseline and patched builds. Per-function figures are means of nine runs with 500,000 calls per case after warm-up; no runs were discarded.

| 10-bit 8×8 case | C ns/call | RVV ns/call | Throughput gain |
| --- | ---: | ---: | ---: |
| put, x/y=0/0 | 54.32 | 28.35 | 91.6% |
| put, x/y=3/0 | 88.60 | 52.58 | 68.5% |
| put, x/y=0/3 | 91.00 | 51.88 | 75.4% |
| put, x/y=3/5 | 170.94 | 78.90 | 116.7% |
| avg, x/y=0/0 | 100.90 | 36.11 | 179.4% |
| avg, x/y=3/0 | 126.33 | 61.65 | 104.9% |
| avg, x/y=0/3 | 125.25 | 60.64 | 106.6% |
| avg, x/y=3/5 | 209.84 | 87.53 | 139.7% |

Each listed case was faster in all nine runs. The geometric-mean throughput gain across the eight cases was 107.8% at 10 bit and 108.9% at 16 bit.

For a 30-second, 900-frame High 10 stream encoded with libx264 (preset medium, CRF 20, three B-frames, three references), single-threaded decoding to the null muxer took 2.2019 ± 0.0228 s on master and 2.0931 ± 0.0144 s with the RVV functions (mean ± sample standard deviation, seven runs). That is 5.19% higher throughput; the patched build was faster in all seven paired runs.
