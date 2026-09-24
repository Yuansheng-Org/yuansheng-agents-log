# CPU: RV64: add f16 NCX batch normalization training statistics JIT

## Description

This PR extends the RV64 batch normalization JIT implementation to f16 forward training with NCX layouts. The existing RV64 JIT path requires precomputed mean and variance, so training with computed statistics falls back to the generic NCSP implementation. The new path computes per-channel mean and variance with RVV and then reuses the existing RV64 normalization kernel.

The optimization is deliberately limited to f16 NCX data when global statistics are not supplied. Cross-core measurements showed clear and stable gains for this path, while f32 and NXC variants did not provide consistently positive results. Global-statistics execution, f32, bf16, and channels-dense layouts retain their existing implementations.

## Implementation details

The statistics stage uses two process-wide JIT kernels. The first kernel loads f16 values with `vle16.v`, widens them to f32, and performs an ordered sum reduction. The second kernel makes a separate pass over the same data, subtracts the computed mean, squares the difference, and reduces it to obtain the variance. Both kernels determine VL dynamically and handle the final partial vector.

Each channel is parallelized independently. For an NCX tensor, every minibatch contributes one contiguous spatial slice to that channel, and the scalar accumulator is carried across those slices. Mean and variance are written directly to the required training output buffers and then consumed by the existing normalization stage.

The separate variance pass avoids the cancellation error of `E[x^2] - E[x]^2`. The two generated kernels are shared instead of being recreated for each primitive or channel, and their references are resolved once before entering the per-channel parallel loop.

The primitive descriptor accepts computed statistics only when all of the following conditions hold:

- propagation kind is forward training;
- source data type is f16;
- the source uses an NCX layout;
- global statistics are not supplied.

Unsupported combinations continue through oneDNN's normal primitive fallback mechanism.

## Test batches

The full official batch normalization CI batch was used for correctness:

```bash
benchdnn --mode=C -v1 --engine=cpu --bnorm --batch=test_bnorm_ci
```

Performance was measured with all forward-training combinations from the official `shapes_ci` file, including f32 and NXC control paths that remain unchanged:

```bash
benchdnn --mode=P -v1 --engine=cpu --bnorm --fix-times-per-prb=20 --perf-template=csv \
    --inplace=true,false --dir=FWD_D --dt=f32,f16 --tag=abx,axb \
    --flags=,C,H,CH --batch=shapes_ci
```

The f16 NCX portion contains 48 cases. A seven-shape topology batch was also added to cover representative spatial sizes from ResNet-50 and GoogLeNet-v3:

```text
mb8ic64ih112
mb8ic256ih56
mb8ic128ih28
mb8ic512ih28
mb8ic1024ih14
mb8ic2048ih7
mb8ic384ih8
```

The topology batch was run with f32/f16, NCX/NXC, `--flags=CH`, and `--inplace=false`; the tables below report the seven f16 NCX cases changed by this PR.

## Performance evaluation

The baseline and patched measurements used the same cross-compiled Release/OpenMP `benchdnn` executable. Only `libdnnl.so` was changed. Each baseline or patched result comes from one benchdnn invocation with 20 executions per problem, and the reported value is benchdnn's minimum execution time. Lower time is better, and the percentage column reports `(baseline - patched) / baseline`.

| Platform    | CPU group     |  RVV VLEN |                     Frequency | Binding                        |
| ----------- | ------------- | --------: | ----------------------------: | ------------------------------ |
| Muse Pi Pro | SpacemiT X60  |  256 bits | 1.6 GHz, performance governor | CPU 0 for 1T; CPUs 0-7 for 8T  |
| K3          | SpacemiT X100 |  256 bits |   2.2 GHz, userspace governor | CPU 0 for 1T; CPUs 0-7 for 8T  |
| K3          | SpacemiT A100 | 1024 bits |   1.8 GHz, userspace governor | CPU 8 for 1T; CPUs 8-15 for 8T |

The A100 process was assigned to the AI-core group before oneDNN initialization. X100 and A100 measurements were run sequentially because the two core groups share the same system.

### Official `shapes_ci` f16 NCX results

The following values are the sums of the per-case minimum times over the 48 f16 NCX cases. The other 176 combinations in the measurement matrix are controls and retain their original implementation.

| Platform          | Threads | Baseline (ms) | Patched (ms) | Time reduction |
| ----------------- | ------: | ------------: | -----------: | -------------: |
| Muse Pi Pro / X60 |       1 |    283.837888 |    89.496326 |        +68.47% |
| Muse Pi Pro / X60 |       8 |     37.605949 |    13.003417 |        +65.42% |
| K3 X100           |       1 |    123.789310 |    52.135252 |        +57.88% |
| K3 X100           |       8 |     16.019767 |     7.495118 |        +53.21% |
| K3 A100           |       1 |    248.156122 |    81.701419 |        +67.08% |
| K3 A100           |       8 |     32.838389 |    12.026612 |        +63.38% |

### Muse Pi Pro / X60 topology detail

| Shape           | Baseline 1T (ms) | Patched 1T (ms) | 1T time reduction | Baseline 8T (ms) | Patched 8T (ms) | 8T time reduction |
| --------------- | ---------------: | --------------: | ----------------: | ---------------: | --------------: | ----------------: |
| `mb8ic64ih112`  |       341.377000 |       54.002000 |           +84.18% |        44.339400 |       10.275400 |           +76.83% |
| `mb8ic256ih56`  |       347.001000 |       70.500200 |           +79.68% |        45.571500 |       11.651900 |           +74.43% |
| `mb8ic128ih28`  |        42.252900 |       11.655800 |           +72.41% |         5.558590 |        1.889890 |           +66.00% |
| `mb8ic512ih28`  |       171.194000 |       46.811300 |           +72.66% |        22.202900 |        7.150390 |           +67.80% |
| `mb8ic1024ih14` |        87.704800 |       38.877900 |           +55.67% |        11.425000 |        5.414790 |           +52.61% |
| `mb8ic2048ih7`  |        48.104200 |       50.243200 |            -4.45% |         6.164550 |        5.893070 |            +4.40% |
| `mb8ic384ih8`   |        11.321300 |        9.342770 |           +17.48% |         1.496580 |        1.051030 |           +29.77% |

The seven-case aggregate time changes from 1048.955200 ms to 281.433170 ms (+73.17%) with one thread and from 136.758520 ms to 43.326470 ms (+68.32%) with eight threads. The smallest-spatial, high-channel X60 case is slightly slower with one thread, while the same case is positive with eight threads and on both K3 core types.

### K3 X100 topology detail

| Shape           | Baseline 1T (ms) | Patched 1T (ms) | 1T time reduction | Baseline 8T (ms) | Patched 8T (ms) | 8T time reduction |
| --------------- | ---------------: | --------------: | ----------------: | ---------------: | --------------: | ----------------: |
| `mb8ic64ih112`  |       148.044000 |       34.543900 |           +76.67% |        19.401100 |        5.557130 |           +71.36% |
| `mb8ic256ih56`  |       148.082000 |       43.983600 |           +70.30% |        19.344000 |        6.078120 |           +68.58% |
| `mb8ic128ih28`  |        18.591800 |        6.672360 |           +64.11% |         2.345950 |        0.951660 |           +59.43% |
| `mb8ic512ih28`  |        74.469200 |       29.747600 |           +60.05% |         9.389160 |        3.720210 |           +60.38% |
| `mb8ic1024ih14` |        37.774900 |       21.258100 |           +43.72% |         4.760740 |        2.785640 |           +41.49% |
| `mb8ic2048ih7`  |        19.636200 |       19.133300 |            +2.56% |         2.488530 |        2.320800 |            +6.74% |
| `mb8ic384ih8`   |         4.781740 |        4.248290 |           +11.16% |         0.645264 |        0.619629 |            +3.97% |

The seven-case aggregate time changes from 451.379840 ms to 159.587150 ms (+64.64%) with one thread and from 58.374744 ms to 22.033189 ms (+62.26%) with eight threads.

### K3 A100 topology detail

| Shape           | Baseline 1T (ms) | Patched 1T (ms) | 1T time reduction | Baseline 8T (ms) | Patched 8T (ms) | 8T time reduction |
| --------------- | ---------------: | --------------: | ----------------: | ---------------: | --------------: | ----------------: |
| `mb8ic64ih112`  |       296.536000 |       39.981700 |           +86.52% |        37.963600 |        6.171880 |           +83.74% |
| `mb8ic256ih56`  |       299.048000 |       53.230500 |           +82.20% |        38.470200 |        7.318850 |           +80.98% |
| `mb8ic128ih28`  |        37.065900 |       10.748000 |           +71.00% |         4.808590 |        1.518310 |           +68.43% |
| `mb8ic512ih28`  |       148.332000 |       43.558300 |           +70.63% |        19.136200 |        5.788330 |           +69.75% |
| `mb8ic1024ih14` |        76.292000 |       36.497100 |           +52.16% |         9.874760 |        5.119870 |           +48.15% |
| `mb8ic2048ih7`  |        40.341800 |       32.973100 |           +18.27% |         5.200440 |        4.280270 |           +17.69% |
| `mb8ic384ih8`   |         9.677250 |        6.487060 |           +32.97% |         1.313960 |        0.890625 |           +32.22% |

The seven-case aggregate time changes from 907.292950 ms to 223.475760 ms (+75.37%) with one thread and from 116.767750 ms to 31.088135 ms (+73.38%) with eight threads.

Across VLEN=256 and VLEN=1024, both the official-shape aggregate and the topology aggregate show substantial reductions for one-thread and eight-thread execution. The benefit is largest when each channel contains enough spatial work to amortize per-channel setup. The implementation gate excludes the f32 and NXC paths that did not show a stable benefit during evaluation.

## Correctness validation

The final library passed the complete official `test_bnorm_ci` batch on all three core types:

| Platform          | Tests | Passed | Skipped | Mistrusted | Failed |
| ----------------- | ----: | -----: | ------: | ---------: | -----: |
| Muse Pi Pro / X60 |  4445 |   1464 |    1449 |       1532 |      0 |
| K3 X100           |  4445 |   2292 |     133 |       2020 |      0 |
| K3 A100           |  4445 |   2292 |     133 |       2020 |      0 |

The explicit 224-case forward-training matrix completed with 128 passed, 96 mistrusted, and 0 failed on each core type. The 28-case topology matrix completed with 12 passed, 16 mistrusted, and 0 failed on each core type. The mistrusted results are benchdnn data-trust classifications rather than numerical failures.

The final source was formatted with clang-format 18.1.8, passed `git diff --check`, and was cross-compiled locally with GCC 11 using a maximum of six parallel build jobs.