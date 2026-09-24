# cpu: rv64: reuse shared xf16 softmax strided JIT kernels

## Description

The RV64 non-contiguous f16/bf16 softmax path invokes gather and scatter kernels once for every softmax block. The existing dispatcher already keeps one function-local static JIT kernel for each direction, but every call still executes compiler-generated thread-safe-static guard bookkeeping in the hot path.

This PR exposes the existing shared gather/scatter kernels through `get_xf16_strided_kernel<>()`, resolves them once when a relevant softmax primitive is created, stores non-owning pointers in the primitive, and invokes the kernels directly during execution. This removes the per-block dispatcher and static-guard overhead without generating JIT code for every primitive.

The accessors are invoked only for f16/bf16 primitives with `inner_size > 1`. For contiguous softmax the two pointer members remain null and no strided kernel initialization occurs, while all strided xf16 primitives share the same process-wide gather/scatter JIT code.

## Implementation details

- `get_xf16_strided_kernel<true/false>()` owns the same two function-local static kernels used by the existing dispatcher.
- The old gather/scatter free functions remain valid and obtain their kernels through the same accessor, so there is only one shared JIT kernel per direction.
- `rvv_softmax_fwd_t` has two non-owning `const` pointer members that are initialized only for the non-contiguous f16/bf16 path.
- The execution loop builds the existing call parameter structures and directly invokes the cached kernel pointers.
- No strided `jit_generator_t` or 256 KiB strided-kernel code buffer is allocated per primitive.

## Test batches

Performance was measured with the following shapes for both f16 and bf16, using `abc`, axis 1, forward inference, and both softmax and logsoftmax:

```text
2048x16x32
2048x32x16
2048x64x8
1024x128x8
512x256x8
256x512x8
```

```bash
benchdnn --mode=P -v1 --engine=cpu --softmax \
    --fix-times-per-prb=100 --perf-template=csv \
    --batch=rvv_softmax_f16_strided_sweep.batch

benchdnn --mode=P -v1 --engine=cpu --softmax \
    --fix-times-per-prb=100 --perf-template=csv \
    --batch=rvv_softmax_bf16_strided_sweep.batch
```

## Performance evaluation

Lower execution time is better. Baseline and patched measurements used the same executable, runtime settings, thread counts, and CPU affinity.

| Platform         | CPU group               | Cores |  RVV VLEN | Test frequency | Frequency policy |
| ---------------- | ----------------------- | ----: | --------: | -------------: | ---------------- |
| K1 / Muse Pi Pro | SpacemiT X60            |     8 |  256 bits |        1.6 GHz | performance      |
| K3 X100          | SpacemiT X100, CPU 0-7  |     8 |  256 bits |        2.4 GHz | fixed userspace  |
| K3 A100          | SpacemiT A100, CPU 8-15 |     8 | 1024 bits |        2.0 GHz | fixed userspace  |

The K3 A100 tests start in a fresh process that switches to the AI-core group before oneDNN or any RVV code runs. One-thread measurements are pinned to one core, and eight-thread measurements are pinned to the corresponding eight-core group. K3 was returned to its original 2.2 GHz X100 and 1.8 GHz A100 settings after testing.

### Detailed performance results

The following tables list every measured case. The K3 bf16 logsoftmax rows use `ref:any` and are shown as unaffected controls. K1 bf16 cases are omitted because they are not supported by the available ISA.

#### K1 / Muse Pi Pro

| Data type | Algorithm  | Shape      | Implementation | Baseline 1T (ms) | Patched 1T (ms) |      1T | Baseline 8T (ms) | Patched 8T (ms) |      8T |
| --------- | ---------- | ---------- | -------------- | ---------------: | --------------: | ------: | ---------------: | --------------: | ------: |
| f16       | SOFTMAX    | 2048x16x32 | `jit:rvv_zvfh` |          40.5786 |         33.0884 | +18.46% |          11.9265 |         12.3425 |  -3.49% |
| f16       | LOGSOFTMAX | 2048x16x32 | `jit:rvv_zvfh` |          42.9914 |         36.6135 | +14.84% |          11.1829 |         10.9948 |  +1.68% |
| f16       | SOFTMAX    | 2048x32x16 | `jit:rvv_zvfh` |          22.6596 |         19.0643 | +15.87% |           5.0354 |          3.8213 | +24.11% |
| f16       | LOGSOFTMAX | 2048x32x16 | `jit:rvv_zvfh` |          24.0590 |         20.8493 | +13.34% |           7.6817 |          4.6014 | +40.10% |
| f16       | SOFTMAX    | 2048x64x8  | `jit:rvv_zvfh` |          16.4001 |         14.4635 | +11.81% |           3.5770 |          3.2338 |  +9.59% |
| f16       | LOGSOFTMAX | 2048x64x8  | `jit:rvv_zvfh` |          17.2217 |         15.4387 | +10.35% |           5.3961 |          3.4160 | +36.69% |
| f16       | SOFTMAX    | 1024x128x8 | `jit:rvv_zvfh` |          13.4311 |         12.5307 |  +6.70% |           3.6244 |          3.5117 |  +3.11% |
| f16       | LOGSOFTMAX | 1024x128x8 | `jit:rvv_zvfh` |          14.0880 |         12.9317 |  +8.21% |           4.5273 |          3.1585 | +30.23% |
| f16       | SOFTMAX    | 512x256x8  | `jit:rvv_zvfh` |          12.4561 |         11.9598 |  +3.98% |           3.5365 |          3.0493 | +13.78% |
| f16       | LOGSOFTMAX | 512x256x8  | `jit:rvv_zvfh` |          12.6364 |         12.2277 |  +3.23% |           3.5330 |          3.5400 |  -0.20% |
| f16       | SOFTMAX    | 256x512x8  | `jit:rvv_zvfh` |          13.0742 |         12.2225 |  +6.51% |           3.3646 |          3.3041 |  +1.80% |
| f16       | LOGSOFTMAX | 256x512x8  | `jit:rvv_zvfh` |          12.4681 |         12.1619 |  +2.46% |           4.1532 |          3.2923 | +20.73% |

The K1 f16 aggregate improves from 242.0643 ms to 213.5520 ms (+11.78%) at one thread and from 67.5386 ms to 58.2656 ms (+13.73%) at eight threads. Every one-thread case improves, with gains ranging from +2.46% to +18.46%. Nine of the twelve eight-thread cases improve, `512x256x8` logsoftmax is effectively unchanged at -0.20%, and the other two cases change by -0.80% and -3.49%. The largest gains appear in shapes that execute many short gather/scatter operations, where eliminating repeated dispatcher and static-guard bookkeeping removes a larger fraction of the total work.

#### K3 X100

| Data type | Algorithm  | Shape      | Implementation     | Baseline 1T (ms) | Patched 1T (ms) |      1T | Baseline 8T (ms) | Patched 8T (ms) |      8T |
| --------- | ---------- | ---------- | ------------------ | ---------------: | --------------: | ------: | ---------------: | --------------: | ------: |
| f16       | SOFTMAX    | 2048x16x32 | `jit:rvv_zvfh`     |          23.0460 |         18.7696 | +18.56% |           8.5439 |          7.6095 | +10.94% |
| f16       | LOGSOFTMAX | 2048x16x32 | `jit:rvv_zvfh`     |          23.5903 |         20.2743 | +14.06% |           6.8847 |          4.9236 | +28.49% |
| f16       | SOFTMAX    | 2048x32x16 | `jit:rvv_zvfh`     |          12.6507 |         10.4490 | +17.40% |           1.6416 |          1.3941 | +15.08% |
| f16       | LOGSOFTMAX | 2048x32x16 | `jit:rvv_zvfh`     |          12.8527 |         11.1802 | +13.01% |           2.3055 |          1.4416 | +37.47% |
| f16       | SOFTMAX    | 2048x64x8  | `jit:rvv_zvfh`     |          12.5258 |         11.4446 |  +8.63% |           1.8430 |          1.5576 | +15.48% |
| f16       | LOGSOFTMAX | 2048x64x8  | `jit:rvv_zvfh`     |          12.8573 |         12.1257 |  +5.69% |           1.7690 |          1.5428 | +12.78% |
| f16       | SOFTMAX    | 1024x128x8 | `jit:rvv_zvfh`     |           8.8156 |          7.8559 | +10.89% |           1.1802 |          1.0762 |  +8.81% |
| f16       | LOGSOFTMAX | 1024x128x8 | `jit:rvv_zvfh`     |           8.9776 |          8.2600 |  +7.99% |           1.2174 |          1.1020 |  +9.48% |
| f16       | SOFTMAX    | 512x256x8  | `jit:rvv_zvfh`     |           6.4635 |          6.3635 |  +1.55% |           0.8411 |          0.7998 |  +4.91% |
| f16       | LOGSOFTMAX | 512x256x8  | `jit:rvv_zvfh`     |           6.8565 |          6.6088 |  +3.61% |           0.8701 |          0.8241 |  +5.29% |
| f16       | SOFTMAX    | 256x512x8  | `jit:rvv_zvfh`     |           5.4886 |          5.4069 |  +1.49% |           0.7750 |          0.6715 | +13.35% |
| f16       | LOGSOFTMAX | 256x512x8  | `jit:rvv_zvfh`     |           5.6577 |          5.5834 |  +1.31% |           0.7094 |          0.6861 |  +3.28% |
| bf16      | SOFTMAX    | 2048x16x32 | `jit:rvv_zvfbfwma` |          24.1217 |         19.4705 | +19.28% |           8.5656 |          7.4809 | +12.66% |
| bf16      | LOGSOFTMAX | 2048x16x32 | `ref:any`          |         260.6130 |        264.4570 |  -1.47% |          34.1976 |         34.3531 |  -0.45% |
| bf16      | SOFTMAX    | 2048x32x16 | `jit:rvv_zvfbfwma` |          13.0066 |         10.7468 | +17.37% |           1.5496 |          1.3999 |  +9.66% |
| bf16      | LOGSOFTMAX | 2048x32x16 | `ref:any`          |         255.8660 |        259.0910 |  -1.26% |          33.0605 |         33.5477 |  -1.47% |
| bf16      | SOFTMAX    | 2048x64x8  | `jit:rvv_zvfbfwma` |          12.8645 |         11.4798 | +10.76% |           1.7233 |          1.4639 | +15.05% |
| bf16      | LOGSOFTMAX | 2048x64x8  | `ref:any`          |         252.8980 |        255.5980 |  -1.07% |          32.7692 |         33.3271 |  -1.70% |
| bf16      | SOFTMAX    | 1024x128x8 | `jit:rvv_zvfbfwma` |           9.0231 |          8.1011 | +10.22% |           1.1679 |          1.0347 | +11.40% |
| bf16      | LOGSOFTMAX | 1024x128x8 | `ref:any`          |         250.8590 |        253.4710 |  -1.04% |          32.4619 |         32.8017 |  -1.05% |
| bf16      | SOFTMAX    | 512x256x8  | `jit:rvv_zvfbfwma` |           6.8371 |          6.4854 |  +5.14% |           0.8349 |          0.8313 |  +0.43% |
| bf16      | LOGSOFTMAX | 512x256x8  | `ref:any`          |         249.7740 |        252.1090 |  -0.93% |          32.1394 |         32.5728 |  -1.35% |
| bf16      | SOFTMAX    | 256x512x8  | `jit:rvv_zvfbfwma` |           5.6749 |          5.5337 |  +2.49% |           0.6900 |          0.7018 |  -1.70% |
| bf16      | LOGSOFTMAX | 256x512x8  | `ref:any`          |         248.9990 |        251.2740 |  -0.91% |          32.1148 |         32.7819 |  -2.08% |

The X100 f16 aggregate improves by +11.06% at one thread and +17.33% at eight threads. Considering only the six bf16 softmax rows that use the changed JIT path, the aggregate improves from 71.5279 ms to 61.8173 ms (+13.58%) at one thread and from 14.5313 ms to 12.9125 ms (+11.14%) at eight threads. The improvement is strongest for shapes with smaller axis sizes and larger inner strides, matching the expected reduction in per-block dispatch overhead. The bf16 logsoftmax reference rows remain close to baseline and serve as controls.

#### K3 A100

| Data type | Algorithm  | Shape      | Implementation     | Baseline 1T (ms) | Patched 1T (ms) |     1T | Baseline 8T (ms) | Patched 8T (ms) |     8T |
| --------- | ---------- | ---------- | ------------------ | ---------------: | --------------: | -----: | ---------------: | --------------: | -----: |
| f16       | SOFTMAX    | 2048x16x32 | `jit:rvv_zvfh`     |          59.6879 |         57.8211 | +3.13% |           8.0880 |          7.9940 | +1.16% |
| f16       | LOGSOFTMAX | 2048x16x32 | `jit:rvv_zvfh`     |          61.7025 |         60.5687 | +1.84% |           8.4593 |          8.2456 | +2.53% |
| f16       | SOFTMAX    | 2048x32x16 | `jit:rvv_zvfh`     |          35.6916 |         35.0999 | +1.66% |           4.8929 |          4.9737 | -1.65% |
| f16       | LOGSOFTMAX | 2048x32x16 | `jit:rvv_zvfh`     |          37.2815 |         36.5396 | +1.99% |           5.1774 |          5.1287 | +0.94% |
| f16       | SOFTMAX    | 2048x64x8  | `jit:rvv_zvfh`     |          22.1751 |         21.9553 | +0.99% |           3.1335 |          3.1154 | +0.58% |
| f16       | LOGSOFTMAX | 2048x64x8  | `jit:rvv_zvfh`     |          22.7363 |         22.2784 | +2.01% |           3.2060 |          3.1490 | +1.78% |
| f16       | SOFTMAX    | 1024x128x8 | `jit:rvv_zvfh`     |          18.1780 |         17.5554 | +3.43% |           2.5892 |          2.5873 | +0.07% |
| f16       | LOGSOFTMAX | 1024x128x8 | `jit:rvv_zvfh`     |          18.6887 |         17.9822 | +3.78% |           2.6358 |          2.6173 | +0.70% |
| f16       | SOFTMAX    | 512x256x8  | `jit:rvv_zvfh`     |          16.6033 |         16.5313 | +0.43% |           2.5042 |          2.5005 | +0.15% |
| f16       | LOGSOFTMAX | 512x256x8  | `jit:rvv_zvfh`     |          16.8959 |         17.1722 | -1.64% |           2.5379 |          2.5080 | +1.18% |
| f16       | SOFTMAX    | 256x512x8  | `jit:rvv_zvfh`     |          15.6796 |         15.8528 | -1.10% |           2.1029 |          2.1196 | -0.80% |
| f16       | LOGSOFTMAX | 256x512x8  | `jit:rvv_zvfh`     |          15.8568 |         15.8615 | -0.03% |           2.1197 |          2.1265 | -0.32% |
| bf16      | SOFTMAX    | 2048x16x32 | `jit:rvv_zvfbfwma` |          63.3659 |         58.9388 | +6.99% |           8.2133 |          7.9289 | +3.46% |
| bf16      | LOGSOFTMAX | 2048x16x32 | `ref:any`          |         482.9830 |        485.8540 | -0.59% |          60.3641 |         61.2943 | -1.54% |
| bf16      | SOFTMAX    | 2048x32x16 | `jit:rvv_zvfbfwma` |          38.6444 |         35.6574 | +7.73% |           5.0046 |          5.0221 | -0.35% |
| bf16      | LOGSOFTMAX | 2048x32x16 | `ref:any`          |         475.1670 |        477.5130 | -0.49% |          60.2456 |         60.7306 | -0.81% |
| bf16      | SOFTMAX    | 2048x64x8  | `jit:rvv_zvfbfwma` |          23.9822 |         22.3010 | +7.01% |           3.1682 |          3.1421 | +0.82% |
| bf16      | LOGSOFTMAX | 2048x64x8  | `ref:any`          |         480.4550 |        478.9320 | +0.32% |          59.6132 |         60.1680 | -0.93% |
| bf16      | SOFTMAX    | 1024x128x8 | `jit:rvv_zvfbfwma` |          19.4982 |         17.8461 | +8.47% |           2.6529 |          2.6404 | +0.47% |
| bf16      | LOGSOFTMAX | 1024x128x8 | `ref:any`          |         474.8480 |        476.8900 | -0.43% |          59.9356 |         59.8499 | +0.14% |
| bf16      | SOFTMAX    | 512x256x8  | `jit:rvv_zvfbfwma` |          17.5138 |         17.0251 | +2.79% |           2.5514 |          2.5029 | +1.90% |
| bf16      | LOGSOFTMAX | 512x256x8  | `ref:any`          |         476.1840 |        477.0000 | -0.17% |          59.7406 |         59.5745 | +0.28% |
| bf16      | SOFTMAX    | 256x512x8  | `jit:rvv_zvfbfwma` |          15.7505 |         15.7535 | -0.02% |           2.1570 |          2.0943 | +2.90% |
| bf16      | LOGSOFTMAX | 256x512x8  | `ref:any`          |         475.3000 |        476.4970 | -0.25% |          60.1551 |         59.7985 | +0.59% |

The A100 f16 aggregate improves by +1.75% at one thread and +0.80% at eight threads. Considering only the six bf16 softmax rows that use the changed JIT path, the aggregate improves from 178.7549 ms to 167.5219 ms (+6.28%) at one thread and from 23.7474 ms to 23.3307 ms (+1.75%) at eight threads. Gains are most visible for the smaller-axis bf16 softmax shapes. The smaller overall A100 benefit is consistent with the per-block dispatch overhead accounting for a lower fraction of execution time with 1024-bit vectors.

## Correctness validation

`ctest` passed.