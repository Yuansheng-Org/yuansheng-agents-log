# CPU: RV64: widen f16 max pooling vectors for VLEN=256 nspc

## Description

This PR widens the channel vector used by the RV64 native f16 max pooling kernel from e16/m1 to e16/m2 for nspc layouts on VLEN=256 systems. Processing twice as many contiguous channels per iteration reduces channel-loop, pointer-update, and epilogue overhead.

The wider path is deliberately limited to `f16`, max pooling, nspc, and VLEN=256. Ncsp uses strided channel accesses and did not benefit from a larger vector group. F16 average pooling and bf16 retain their existing accumulator layout. VLEN=128 and VLEN=1024 also retain the existing path because cross-core measurements did not show stable gains there.

## Implementation details

For the enabled path, the kernel uses e16/m2 for the f16 max accumulator, source load, result store, and argmax vector. Fused post-ops widen the accumulator to e32/m4, invoke the post-op injector with a group stride of four registers, and narrow back to e16/m2. Max-training workspace conversion uses e8/m1 for u8 indices and e32/m4 for s32 indices. The selected LMUL is carried consistently through reduction, post-ops, destination storage, workspace conversion, pointer advancement, and the channel-loop decrement.

The vector groups remain non-overlapping: `v4-v5` for the accumulator, `v8-v9` for the load/result, `v10-v11` for argmax, `v12-v23` for injector auxiliaries, `v24-v27` for the f32 widened max, and `v28-v31` for binary/workspace scratch.

## Performance evaluation

The measurements used locally cross-compiled Release/OpenMP libraries and the same `benchdnn` executable for baseline and patched runs. Only `libdnnl.so` was switched. Tests used 1 and 8 OpenMP threads with fixed CPU affinity.

### Test batches

`pool_f16_official` uses the official `shapes_basic` set with f16, forward training and inference, nspc and ncsp layouts, and max/avg_np/avg_p algorithms. `pool_f16_channel_sweep` adds C=15/16/17, 31/32/33, 63/64/65, and 127/128/129 to cover channel-vector boundaries. `pool_f16_epilogue` covers max-training u8/s32 workspaces and per-oc add and linear post-ops.

```bash
benchdnn --mode=P --fix-times-per-prb=100 -v1 --engine=cpu --pool --batch=<batch>
```

The following table reports the reduction in the sum of minimum execution times for cases that activate the new path, namely nspc f16 max pooling. Higher is better.

| Platform          | VLEN | Threads | Official shapes | Channel sweep | Epilogue/workspace/post-op |
| ----------------- | ---: | ------: | --------------: | ------------: | -------------------------: |
| Muse Pi Pro / X60 |  256 |       1 |         +10.07% |       +16.68% |                    +15.86% |
| Muse Pi Pro / X60 |  256 |       8 |          +3.69% |        +7.29% |                    +21.34% |
| K3 X100           |  256 |       1 |          +9.61% |       +19.97% |                    +25.00% |
| K3 X100           |  256 |       8 |         +14.35% |        +7.31% |                    +11.49% |

### Representative detailed results

| Platform | Threads | Case                                        | Baseline (ms) | Patched (ms) | Improvement |
| -------- | ------: | ------------------------------------------- | ------------: | -----------: | ----------: |
| X60      |       1 | FWD_D max, nspc, C33, 1D                    |      0.017334 |     0.015625 |      +9.86% |
| X60      |       1 | FWD_D max, nspc, C129, KW257, s32 workspace |      0.250244 |     0.218506 |     +12.68% |
| X60      |       1 | FWD_I max, nspc, C129, per-oc add           |      0.038818 |     0.028564 |     +26.42% |
| X60      |       8 | FWD_D max, nspc, C129, KW257, s32 workspace |      0.153564 |     0.102539 |     +33.23% |
| X100     |       1 | FWD_D max, nspc, C33, 1D                    |      0.007080 |     0.006104 |     +13.79% |
| X100     |       1 | FWD_D max, nspc, C129, KW257, s32 workspace |      0.096924 |     0.069336 |     +28.46% |
| X100     |       1 | FWD_D max, nspc, C129, per-oc add           |      0.016602 |     0.011475 |     +30.88% |
| X100     |       8 | FWD_D max, nspc, C129, KW257, s32 workspace |      0.031250 |     0.025635 |     +17.97% |

SG2044 with VLEN=128 and K3 A100 with VLEN=1024 were also tested. The final implementation keeps the original m1 path on those systems, avoiding the regressions observed when the wider path was applied outside VLEN=256.

## Correctness validation

On X60, X100, SG2044, and A100, baseline and patched libraries were each tested with 1 and 8 threads. For every configuration, the official basic group completed with 360 passed and 24 skipped, the channel sweep completed with 144 passed, and the epilogue group completed with 52 passed; all reported zero failures.

The complete official `test_pool_float16` batch was additionally run on X100 with the final patched library:

| Threads | Tests | Passed | Skipped | Failed |
| ------: | ----: | -----: | ------: | -----: |
|       1 |  4014 |   3654 |     360 |      0 |
|       8 |  4014 |   3654 |     360 |      0 |

In each full-batch run, 2669 cases used `jit:rvv_zvfh`.