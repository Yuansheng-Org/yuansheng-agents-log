# 002_sgemv_n 验证摘要

- 原始 patch：`002_sgemv_n/craft/patch.diff`
- SHA-256：`0ce0e8434131e70c2d8a4f9687aa8ef054ed81f455de9bb6821a4bb9c62b79fc`
- 去重覆盖：002_dgemv_n
- 修改模块：kernel/riscv64/gemv_n_vector.c
- 修改内容：单位步长实数 GEMV 使用固定 VLMAX 和四列 register blocking。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`
- 测试机：SpaceMiT K3/X100，RISC-V 64，GCC 15.2；固定 CPU7，采样前后均为 2.2 GHz。
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；官方 `make tests` 返回 0，125/125 utest、1473/1473 extension tests 及 CBLAS L1/L2/L3 无新增失败。
- 官方 benchmark：`benchmark/sgemv.goto`, `benchmark/dgemv.goto`
- 性能方法：每个尺寸预热 1 次、实测 3 次并取中位数；Level-1 每次内部循环 10000 次，Level-2 每次内部循环 100 次。
- PASS 门槛：每个共享精度入口的 512 中位数提升 ≥2%，三次 patched 均高于 baseline 512 中位数，且任一尺寸无 ≤−3% 回退。

## 性能结果

| Benchmark | Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---|---:|---|---|---:|---:|---:|
| dgemv | 256 | 2410.66, 2387.24, 2415.36 | 3158.90, 3153.58, 3167.27 | 2410.66 | 3158.90 | +31.04% |
| dgemv | 512 | 3626.29, 3708.36, 3670.95 | 4436.78, 4515.35, 4483.65 | 3670.95 | 4483.65 | +22.14% |
| dgemv | 1024 | 1896.64, 1899.48, 1899.54 | 1946.13, 1942.68, 1945.03 | 1899.48 | 1945.03 | +2.40% |
| sgemv | 256 | 5802.52, 5885.21, 6286.47 | 5833.16, 5817.72, 5816.13 | 5885.21 | 5817.72 | -1.15% |
| sgemv | 512 | 4824.60, 4848.66, 4872.43 | 6339.19, 6303.53, 6287.75 | 4848.66 | 6303.53 | +30.01% |
| sgemv | 1024 | 4555.51, 4584.54, 4658.37 | 5287.97, 5162.68, 5206.73 | 4584.54 | 5206.73 | +13.57% |

## 最终结论

`PASS`。补丁应用、构建、完整官方功能测试和所有共享入口的性能门槛均通过。
