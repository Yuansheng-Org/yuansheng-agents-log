# CPU: RV64: enable f16 dst for JIT RVV 1x1 convolution

## Description

This PR enables the RV64 JIT RVV 1x1 convolution forward kernel for f16 destination tensors.

Before this change, the RV64 1x1 convolution JIT accepted f16 source and f16 weights with f32 accumulation, but the primitive descriptor required the destination data type to be f32. As a result, f16 destination 1x1 convolution fell back to the reference implementation.

The new path keeps the existing f32 accumulator computation and narrows the final accumulator values to f16 at store time when Zvfh is available.

## Implementation details

The implementation records the destination data type in `jit_1x1_conv_conf_t` and allows f16 destination tensors when Zvfh is available. Source and weight handling remains unchanged: f16 inputs are widened through the existing RVV widening FMA path and accumulated in f32.

The destination and bias pointers are addressed as byte pointers in the primitive driver. This is required because source, weights, bias and destination can have different element sizes. The kernel stores f32 destinations with `vse32.v` and f16 destinations by narrowing the f32 accumulator with `vfncvt.f.f.w` followed by `vse16.v`.

The f16 destination path keeps the full reduction inside one kernel invocation, so an intermediate partial sum is never stored to an f16 destination buffer. After a narrow f16 store, the kernel restores the accumulator vtype before the next spatial block. This keeps the no-bias accumulator initialization in the correct e32 accumulator view.

## Correctness validation

Correctness was validated on:

- Muse Pi Pro / SpacemiT X60, RVV VLEN 256
- SG2044-179, RVV VLEN 128

The affected official oneDNN conv batch segment was executed on both machines:

```bash
benchdnn --mode=C -v1 --engine=cpu --conv --reset --mb=2 --stag=axb --dtag=axb --skip-impl=ref --dir=FWD_D --dt=f16 --batch=shapes_resnet_50
```

Result:

```text
Muse Pi Pro: tests:20 passed:9 skipped:11 failed:0
SG2044-179:  tests:20 passed:9 skipped:11 failed:0
```

Additional targeted checks covered f16 destination with and without f16 bias, OC/IC tail shapes, and the existing f16 source/weight with f32 destination path.

## Performance evaluation

Performance was measured with the same `benchdnn` executable for baseline and patched runs. Only `LD_LIBRARY_PATH` was changed to select the baseline or patched `libdnnl.so`. Each problem used `--fix-times-per-prb=1` because the baseline f16 destination path falls back to `ref:any` and is much slower than the JIT path.

Lower time is better.

### Muse Pi Pro / SpacemiT X60

| Threads | Shape                   | Baseline impl | Patched impl     | Baseline avg (ms) | Patched avg (ms) | Improvement |
| ------: | ----------------------- | ------------- | ---------------- | ----------------: | ---------------: | ----------: |
|       1 | ic64ih56oc64oh56kh1ph0  | ref:any       | jit_1x1:rvv_zvfh |       1443.580000 |         5.668700 |      99.61% |
|       1 | ic64ih56oc256oh56kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |       5736.940000 |        20.950400 |      99.63% |
|       1 | ic512ih7oc2048oh7kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |       5075.340000 |        22.420400 |      99.56% |
|       8 | ic64ih56oc64oh56kh1ph0  | ref:any       | jit_1x1:rvv_zvfh |        281.844000 |         0.716309 |      99.75% |
|       8 | ic64ih56oc256oh56kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |        978.192000 |         9.687500 |      99.01% |
|       8 | ic512ih7oc2048oh7kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |        932.240000 |         8.033200 |      99.14% |

### SG2044-179

| Threads | Shape                   | Baseline impl | Patched impl     | Baseline avg (ms) | Patched avg (ms) | Improvement |
| ------: | ----------------------- | ------------- | ---------------- | ----------------: | ---------------: | ----------: |
|       1 | ic64ih56oc64oh56kh1ph0  | ref:any       | jit_1x1:rvv_zvfh |        684.376000 |         2.112060 |      99.69% |
|       1 | ic64ih56oc256oh56kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |       2732.780000 |         8.795650 |      99.68% |
|       1 | ic512ih7oc2048oh7kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |       2443.860000 |         8.837160 |      99.64% |
|       8 | ic64ih56oc64oh56kh1ph0  | ref:any       | jit_1x1:rvv_zvfh |         89.659700 |         0.274658 |      99.69% |
|       8 | ic64ih56oc256oh56kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |        344.504000 |         1.037350 |      99.70% |
|       8 | ic512ih7oc2048oh7kh1ph0 | ref:any       | jit_1x1:rvv_zvfh |        307.299000 |         1.332520 |      99.57% |

