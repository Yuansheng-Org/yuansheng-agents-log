# cpu rv64: hoist eltwise constants out of the main loop

## Description

Several RV64 eltwise algorithms repeatedly materialize the same floating-point constants inside their fixed-VL processing loop. This PR moves that work into a preamble for sufficiently large workloads, allowing the loop to reuse scalar registers containing the constants.

The implementation discovers constants from the existing eltwise injector, keeping algorithm coefficients in a single source. During JIT generation, it selects constant hoisting when the estimated work per thread is at least 2048 elements. Smaller workloads retain inline constant materialization. Each primitive emits one loop implementation, so this selection adds no execution-time dispatch branch.

## Implementation details

- An independent collector injector discovers the constants while the main injector's host pointer remains immutable.
- Constant discovery uses an 8 KiB stack-backed Xbyak buffer that does not require executable memory.
- A fixed-capacity array stores constant bit patterns without per-constant heap allocation.
- The preamble materializes collected constants into scalar registers, which the generated loop reuses.
- The first vector load precedes constant setup.
- The bf16 group stride uses LMUL=m2.

The workload condition is evaluated during JIT generation:

```cpp
utils::div_up(nelems, dnnl_get_max_threads()) >= 2048
```

For a representative small f32 forward GELU-ERF workload, `5x16x3`, the baseline and patched JIT code on X60 are both 568 bytes and byte-for-byte identical.

## Validation batches

Correctness testing uses batches derived from the official oneDNN `test_eltwise_all`, selecting data types supported by the standalone RV64 eltwise JIT:

- `test_eltwise_rv64_x60`: f32, f16, s32, s8 and u8.
- `test_eltwise_rv64_x100_a100`: the same types plus bf16.

Performance testing uses the supplementary batches `rvv_eltwise_large_covered_x60` and `rvv_eltwise_large_covered_x100_a100`. Each workload contains 1,048,576 elements. The X60 batch contains 23 cases, and the X100/A100 batch contains 31 cases.

The supplementary cases cover floating-point forward and backward implementations of soft_relu, gelu_erf, gelu_tanh, mish, exp and relu where implemented, together with s32, s8 and u8 forward relu, linear and clip. These workloads exercise the constant-hoisting path with both one and eight threads.

## Performance evaluation

| System      | Core |      VLEN | Tested data types           |
| ----------- | ---- | --------: | --------------------------- |
| Muse Pi Pro | X60  |  256 bits | f32, f16, s32, s8, u8       |
| K3          | X100 |  256 bits | f32, f16, bf16, s32, s8, u8 |
| K3          | A100 | 1024 bits | f32, f16, bf16, s32, s8, u8 |

The code was cross-compiled locally. Baseline and patched measurements use the same `benchdnn` executable, selecting the corresponding `libdnnl.so` through `LD_LIBRARY_PATH`. The baseline source commit is `8e9be4b38e0692e587fd4c8d222bafcc4ddec4d5`, and the patched source commit is `de3124ac2efb7bc6799ef33a0474e9132bfa656a`.

Measurements were collected in baseline → patched order for one-thread and eight-thread configurations, using `--fix-times-per-prb=100`. The tables use `min_time`. Aggregate improvements are calculated as `100 × (1 − sum(patched times) / sum(baseline times))`; positive values indicate lower execution time. These aggregates describe the listed supplementary workloads.

### Overall results

| Core | 1 thread | 8 threads |
| ---- | -------: | --------: |
| X60  |  +16.15% |   +13.27% |
| X100 |   +0.09% |    -0.03% |
| A100 |  +14.64% |   +13.60% |

The measured workloads improve on X60 and A100. X100 is approximately neutral in both thread configurations.

### X60 results by data type

| Data type | 1 thread | 8 threads |
| --------- | -------: | --------: |
| f16       |  +18.91% |   +17.81% |
| f32       |  +15.27% |   +13.46% |
| s32       |   +5.42% |    +0.03% |
| s8        |   +7.40% |    +3.82% |
| u8        |   +8.76% |    +3.98% |

### X100 results by data type

| Data type | 1 thread | 8 threads |
| --------- | -------: | --------: |
| bf16      |   +0.13% |    +0.02% |
| f16       |   +0.15% |    +0.02% |
| f32       |   +0.00% |    -0.08% |
| s32       |   +0.96% |    -0.15% |
| s8        |   +0.00% |    +0.00% |
| u8        |   -0.03% |    -0.06% |

### A100 results by data type

| Data type | 1 thread | 8 threads |
| --------- | -------: | --------: |
| bf16      |  +16.66% |   +16.22% |
| f16       |  +16.58% |   +16.25% |
| f32       |  +11.75% |   +11.76% |
| s32       |   +9.21% |    -0.07% |
| s8        |   +7.67% |    +7.61% |
| u8        |   +7.66% |    +7.78% |

## Correctness validation

| Core |  Tests | Passed | Skipped | Mistrusted | Failed |
| ---- | -----: | -----: | ------: | ---------: | -----: |
| X60  | 28,575 |  9,766 |  18,760 |         49 |      0 |
| X100 | 47,685 | 15,906 |  31,695 |         84 |      0 |
| A100 | 47,685 | 15,906 |  31,695 |         84 |      0 |