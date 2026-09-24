# avcodec/riscv: add RVV HEVC weighted interpolation

## Motivation

HEVC weighted qpel and epel interpolation currently falls back to the C implementation on RISC-V. These functions account for a measurable part of weighted prediction decoding on RVV systems.

## Implementation

Add RVV implementations for:

- qpel at 8, 9, 10 and 12 bits;
- epel at 8 bits;
- horizontal, vertical and two-dimensional filters;
- uni-weighted and bi-weighted prediction.

The C wrappers retain the existing HEVC DSP ABI. A shared RVV row kernel performs widening filter accumulation, weighting, rounding, offset and clipping. Two-dimensional filtering writes the horizontal result to a 16-bit intermediate buffer before the vertical pass.

The functions are installed when `AV_CPU_FLAG_RVV_I32` is available and VLEN is at least 128 bits. qpel uses RVV for widths 6 and above; epel uses RVV for widths 8 and above. Smaller blocks keep the C functions because their vector setup overhead exceeded the useful work on the test system.

## Testing

Tested against FFmpeg `fddc59cf34b303a269493aaf7ec7c80e390f25f5` with GCC 15.2.0 on a SpacemiT X100 core, VLEN=256.

```sh
./tests/checkasm/checkasm --affinity=0 --test=hevc_pel --repeat=5 1234
```

All 239 tests passed for seeds 1234 through 1238.

The following FATE tests passed:

```text
fate-hevc-conformance-WP_A_Toshiba_3
fate-hevc-conformance-WP_B_Toshiba_3
fate-hevc-conformance-WP_A_MAIN10_Toshiba_3
fate-hevc-conformance-WP_MAIN10_B_Toshiba_3
```

No test files are changed.

## Performance

checkasm reports 234 installed weighted prediction functions. All were faster than C on the test system. The speedup ranges were 1.13x to 4.49x, depending on block width, direction and bit depth. The lowest result, 8-bit qpel bi-weighted horizontal at width 6, measured 1.12x to 1.15x over five repeated runs.

Full decoder measurements used 416x240 HEVC weighted prediction conformance streams concatenated to 2048 frames. Each binary was warmed up once and then measured five times in alternating baseline/candidate order. Decoding was single-threaded and pinned to CPU 0 at 2.2 GHz. No samples were excluded.

| Input | Baseline mean | RVV mean | Time reduction | Speedup |
|---|---:|---:|---:|---:|
| 8-bit WP B | 4.9738 s | 3.6714 s | 26.19% | 1.3547x |
| 10-bit WP B | 5.3284 s | 4.0852 s | 23.33% | 1.3043x |

