# 001_zgemm_kernel_n 验证摘要

- 原始 patch：`001_zgemm_kernel_n.patch`
- SHA-256：`d51dd3b4ecd403401bc8deae9ae66cd89d3b0573a7dd0ce37b77bb735a8e3b2a`
- 修改模块：kernel/riscv64/zgemm_kernel_8x4_zvl256b.c；ZGEMM normal 复数微内核
- 修改内容：用 segment load 替代跨步 load，并把临时复数乘积折叠为直接累加 FMA。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`（2026-09-16T07:27:05+02:00 Merge pull request #6036 from martin-frbg/f2c-lapack-i）
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；`make tests` 返回 0，125/125 utest 与 1473/1473 extension tests 通过，CBLAS L1/L2/L3 官方测试无新增失败。
- 官方 benchmark：`benchmark/zgemm.goto`
- 性能方法：固定 CPU7（X100），全程 2.2 GHz，单线程；每个尺寸预热 1 次、实测 3 次，取中位数。
- 明显回退阈值：中位数变化 `<= -3%`；本补丁最差变化 +16.24%，所以明显回退：否。
- 可重复提升口径：512 目标尺寸中位数至少提升 2%，且 patched 三次均高于 baseline 中位数；结果：满足。

## 性能结果

| Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---:|---|---|---:|---:|---:|
| 256 | 8958.25, 9056.86, 9196.08 | 10837.64, 11418.60, 10828.57 | 9056.86 | 10837.64 | +19.66% |
| 512 | 8640.63, 8633.76, 8624.62 | 10046.71, 10048.18, 10006.75 | 8633.76 | 10046.71 | +16.37% |
| 1024 | 8947.19, 8929.60, 8948.73 | 10418.47, 10399.87, 10387.19 | 8947.19 | 10399.87 | +16.24% |

## 最终结论

`PASS`。补丁应用、构建、官方功能测试和性能门槛均通过。
