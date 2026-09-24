# 002_saxpy_k 验证摘要

- 原始 patch：`002_saxpy_k/craft/patch.diff`
- SHA-256：`3d0b542c93191cf364302f48ce9719a2a3261753a010f0d34d9638596c16ba84`
- 去重覆盖：002_daxpy_k
- 修改模块：kernel/riscv64/axpy_vector.c
- 修改内容：ZVL256B 实数 AXPY 的 LMUL 从 m2 提升到 m4。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`
- 测试机：SpaceMiT K3/X100，RISC-V 64，GCC 15.2；固定 CPU7，采样前后均为 2.2 GHz。
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；官方 `make tests` 返回 0，125/125 utest、1473/1473 extension tests 及 CBLAS L1/L2/L3 无新增失败。
- 官方 benchmark：`benchmark/saxpy.goto`, `benchmark/daxpy.goto`
- 性能方法：每个尺寸预热 1 次、实测 3 次并取中位数；Level-1 每次内部循环 10000 次，Level-2 每次内部循环 100 次。
- PASS 门槛：每个共享精度入口的 512 中位数提升 ≥2%，三次 patched 均高于 baseline 512 中位数，且任一尺寸无 ≤−3% 回退。

## 性能结果

| Benchmark | Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---|---:|---|---|---:|---:|---:|
| daxpy | 256 | 2173.15, 2168.87, 2166.46 | 2784.49, 2917.04, 2789.62 | 2168.87 | 2789.62 | +28.62% |
| daxpy | 512 | 2535.53, 2526.07, 2537.03 | 3311.49, 3451.06, 3437.93 | 2535.53 | 3437.93 | +35.59% |
| daxpy | 1024 | 2698.39, 2701.35, 2701.74 | 3729.33, 3787.78, 3792.61 | 2701.35 | 3787.78 | +40.22% |
| saxpy | 256 | 3598.21, 3781.60, 3603.45 | 4276.26, 4644.56, 4518.35 | 3603.45 | 4518.35 | +25.39% |
| saxpy | 512 | 4511.64, 4505.13, 4519.35 | 5764.33, 5801.53, 5948.24 | 4511.64 | 5801.53 | +28.59% |
| saxpy | 1024 | 5288.00, 5318.93, 5214.57 | 7035.53, 6838.67, 6882.56 | 5288.00 | 6882.56 | +30.15% |

## 最终结论

`PASS`。补丁应用、构建、完整官方功能测试和所有共享入口的性能门槛均通过。
