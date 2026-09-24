# CPU: RV64: enable f16 destination for BRGEMM inner product

## Description

This PR enables the RV64 BRGEMM inner-product implementation for f16 source, f16 weights, and f16 destination when Zvfh is available. These problems previously selected the reference implementation because the RV64 primitive descriptor accepted only an f32 destination.

The BRGEMM kernel continues to accumulate in f32 and narrows to f16 only at the final C store.



## Implementation details

- Accept f16 destination only for the existing f16/Zvfh BRGEMM path.
- Track the destination store type in the BRGEMM descriptor and use the actual destination element size for row and column address calculation.
- Keep the existing e16/m2 input and e32/m4 accumulator organization.
- Narrow four-column main blocks and single-column tails with `vfncvt.f.f.w` and store them with `vse16.v`.
- For nonzero beta, load f16 C, widen it to f32, add the f32 accumulator, then narrow once for the final store.
- Preserve f32 bias addition before destination narrowing.
- Restore e32/m4 after each f16 store sequence so the next loop iteration starts with the expected accumulator vtype.
- Process the complete K range before an f16 destination store. This prevents an intermediate K block from being rounded to f16 and widened again before the remaining accumulation.
- Leave the f32, bf16, and int8 BRGEMM store paths unchanged.

## Performance evaluation

The same `benchdnn` executable was used with baseline and patched libraries. The batch contains 12 no-bias and 5 f32-bias f16-destination problems. It covers four-column blocks, single-column and M/N tails, K values of 64, 255, 256, 257, 512, 1024, and 2048, and the ResNet-50 ip1 dimensions. Source and destination use `ab`, and weights use `ba`.

```bash
benchdnn --mode=P --max-ms-per-prb=500 -v1 --engine=cpu --ip \
    --batch=ip_f16_dst
```

Lower execution time is better. The tables report the per-problem minimum time. The baseline selects `ref:any` for all 17 problems, while the patched library selects `brgemm:rvv_zvfh` for all 17.

| Platform    | CPU group     |  RVV VLEN | Binding                        |
| ----------- | ------------- | --------: | ------------------------------ |
| Muse Pi Pro | SpacemiT X60  |  256 bits | CPU 0 for 1T; CPUs 0-7 for 8T  |
| K3          | SpacemiT X100 |  256 bits | CPU 0 for 1T; CPUs 0-7 for 8T  |
| K3          | SpacemiT A100 | 1024 bits | CPU 8 for 1T; CPUs 8-15 for 8T |

### Muse Pi Pro / X60

| Case                      | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| no bias: mb1ic64oc16      |         0.124023 |        0.003662 |        +97.05% |         0.024902 |        0.007568 |        +69.61% |
| no bias: ic255oc31        |         1.824710 |        0.007812 |        +99.57% |         0.247314 |        0.013428 |        +94.57% |
| no bias: mb3ic256oc32     |         2.824220 |        0.009766 |        +99.65% |         0.368652 |        0.015625 |        +95.76% |
| no bias: mb4ic257oc33     |         3.908690 |        0.013184 |        +99.66% |         0.519043 |        0.014648 |        +97.18% |
| no bias: mb5ic512oc64     |        19.477500 |        0.034912 |        +99.82% |         2.576170 |        0.024414 |        +99.05% |
| no bias: mb7ic257oc127    |        26.573700 |        0.065185 |        +99.75% |         3.386960 |        0.043701 |        +98.71% |
| no bias: mb8ic512oc128    |        62.360100 |        0.083496 |        +99.87% |         8.299800 |        0.076660 |        +99.08% |
| no bias: mb9ic1024oc129   |       156.035000 |        0.569824 |        +99.63% |        24.129200 |        0.686523 |        +97.15% |
| no bias: mb16ic64oc256    |        32.147900 |        0.049072 |        +99.85% |         4.205320 |        0.025879 |        +99.38% |
| no bias: mb32ic512oc256   |       502.547000 |        0.633301 |        +99.87% |        66.701900 |        0.140869 |        +99.79% |
| no bias: ic2048oc1000     |       512.643000 |        2.408940 |        +99.53% |        69.165500 |        1.681880 |        +97.57% |
| no bias: mb32ic2048oc1000 |      8250.450000 |       20.506800 |        +99.75% |      1293.680000 |        3.213130 |        +99.75% |
| bias: mb1ic64oc16         |         0.125244 |        0.004395 |        +96.49% |         0.025635 |        0.008301 |        +67.62% |
| bias: mb4ic257oc33        |         3.933840 |        0.013672 |        +99.65% |         0.518311 |        0.015137 |        +97.08% |
| bias: mb5ic512oc64        |        19.515100 |        0.035889 |        +99.82% |         2.568850 |        0.024658 |        +99.04% |
| bias: mb8ic512oc128       |        62.680900 |        0.084228 |        +99.87% |         8.404300 |        0.081543 |        +99.03% |
| bias: mb32ic2048oc1000    |      8372.210000 |       20.560500 |        +99.75% |      1312.820000 |        3.187500 |        +99.76% |

The 17-case aggregate changes from 18029.380927 ms to 45.084638 ms (+99.75%) with one thread and from 2797.641857 ms to 9.261465 ms (+99.67%) with eight threads.

### K3 X100

| Case                      | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| no bias: mb1ic64oc16      |         0.042236 |        0.001709 |        +95.95% |         0.010742 |        0.003418 |        +68.18% |
| no bias: ic255oc31        |         0.616211 |        0.002686 |        +99.56% |         0.084961 |        0.004639 |        +94.54% |
| no bias: mb3ic256oc32     |         0.948730 |        0.003418 |        +99.64% |         0.126221 |        0.005615 |        +95.55% |
| no bias: mb4ic257oc33     |         1.310550 |        0.005371 |        +99.59% |         0.176514 |        0.005371 |        +96.96% |
| no bias: mb5ic512oc64     |         6.594240 |        0.012695 |        +99.81% |         0.874023 |        0.010254 |        +98.83% |
| no bias: mb7ic257oc127    |         8.892580 |        0.017578 |        +99.80% |         1.156490 |        0.007568 |        +99.35% |
| no bias: mb8ic512oc128    |        20.944600 |        0.033203 |        +99.84% |         3.038820 |        0.062988 |        +97.93% |
| no bias: mb9ic1024oc129   |        48.805400 |        0.134277 |        +99.72% |         6.355710 |        0.289307 |        +95.45% |
| no bias: mb16ic64oc256    |        10.405800 |        0.017334 |        +99.83% |         1.330320 |        0.007080 |        +99.47% |
| no bias: mb32ic512oc256   |       166.837000 |        0.246826 |        +99.85% |        22.865200 |        0.120117 |        +99.47% |
| no bias: ic2048oc1000     |       206.692000 |        1.386470 |        +99.33% |        52.688000 |        0.783691 |        +98.51% |
| no bias: mb32ic2048oc1000 |      3330.730000 |        4.793460 |        +99.86% |       640.462000 |        3.234130 |        +99.50% |
| bias: mb1ic64oc16         |         0.042725 |        0.001953 |        +95.43% |         0.010498 |        0.003662 |        +65.12% |
| bias: mb4ic257oc33        |         1.326900 |        0.005615 |        +99.58% |         0.177490 |        0.005615 |        +96.84% |
| bias: mb5ic512oc64        |         6.584960 |        0.012939 |        +99.80% |         0.885498 |        0.010254 |        +98.84% |
| bias: mb8ic512oc128       |        20.715600 |        0.033203 |        +99.84% |         3.114010 |        0.062988 |        +97.98% |
| bias: mb32ic2048oc1000    |      3337.820000 |        4.825440 |        +99.86% |       639.778000 |        3.237550 |        +99.49% |

The 17-case aggregate changes from 7169.309532 ms to 11.534178 ms (+99.84%) with one thread and from 1373.134497 ms to 7.854248 ms (+99.43%) with eight threads.

### K3 A100

| Case                      | Baseline 1T (ms) | Patched 1T (ms) | 1T improvement | Baseline 8T (ms) | Patched 8T (ms) | 8T improvement |
| ------------------------- | ---------------: | --------------: | -------------: | ---------------: | --------------: | -------------: |
| no bias: mb1ic64oc16      |         0.108398 |        0.003662 |        +96.62% |         0.021240 |        0.007324 |        +65.52% |
| no bias: ic255oc31        |         1.604250 |        0.016113 |        +99.00% |         0.218750 |        0.019531 |        +91.07% |
| no bias: mb3ic256oc32     |         2.488530 |        0.017822 |        +99.28% |         0.323975 |        0.021240 |        +93.44% |
| no bias: mb4ic257oc33     |         3.447750 |        0.017334 |        +99.50% |         0.457275 |        0.021240 |        +95.36% |
| no bias: mb5ic512oc64     |        16.930700 |        0.033936 |        +99.80% |         2.198970 |        0.038086 |        +98.27% |
| no bias: mb7ic257oc127    |        23.367700 |        0.054932 |        +99.76% |         2.976810 |        0.058838 |        +98.02% |
| no bias: mb8ic512oc128    |        53.960000 |        0.043945 |        +99.92% |         7.021730 |        0.016602 |        +99.76% |
| no bias: mb9ic1024oc129   |       135.863000 |        0.318115 |        +99.77% |        19.625700 |        0.139404 |        +99.29% |
| no bias: mb16ic64oc256    |        28.110800 |        0.023682 |        +99.92% |         3.626710 |        0.011719 |        +99.68% |
| no bias: mb32ic512oc256   |       434.411000 |        0.334961 |        +99.92% |        55.765900 |        0.047852 |        +99.91% |
| no bias: ic2048oc1000     |       436.390000 |        3.965090 |        +99.09% |        58.454300 |        0.479492 |        +99.18% |
| no bias: mb32ic2048oc1000 |      6976.220000 |       12.184100 |        +99.83% |       918.943000 |        1.567630 |        +99.83% |
| bias: mb1ic64oc16         |         0.109131 |        0.004150 |        +96.20% |         0.021973 |        0.007812 |        +64.44% |
| bias: mb4ic257oc33        |         3.447020 |        0.017822 |        +99.48% |         0.466064 |        0.021484 |        +95.39% |
| bias: mb5ic512oc64        |        16.972900 |        0.034180 |        +99.80% |         2.183590 |        0.038330 |        +98.24% |
| bias: mb8ic512oc128       |        54.472400 |        0.044189 |        +99.92% |         7.044680 |        0.016846 |        +99.76% |
| bias: mb32ic2048oc1000    |      6953.410000 |       12.188700 |        +99.82% |       923.510000 |        1.576420 |        +99.83% |

The 17-case aggregate changes from 15141.313579 ms to 29.302734 ms (+99.81%) with one thread and from 2002.860667 ms to 4.089850 ms (+99.80%) with eight threads.

## Correctness validation

The focused 17-case batch passed with baseline and patched libraries, using both one and eight OpenMP threads, on all three CPU groups:

| Platform          | Configurations           | Result per configuration |
| ----------------- | ------------------------ | ------------------------ |
| Muse Pi Pro / X60 | baseline/patched × 1T/8T | 17 passed, 0 failed      |
| K3 X100           | baseline/patched × 1T/8T | 17 passed, 0 failed      |
| K3 A100           | baseline/patched × 1T/8T | 17 passed, 0 failed      |

The patched library also completed the full official `test_ip_float16` batch on K3 X100: 1836 tests, 1792 passed, 6 skipped, 38 mistrusted, and 0 failed. Mistrusted tests are not counted as passed.