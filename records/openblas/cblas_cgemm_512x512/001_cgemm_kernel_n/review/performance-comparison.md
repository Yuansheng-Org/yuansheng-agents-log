# 001_cgemm_kernel_n 验证摘要

- 原始 patch：`001_cgemm_kernel_n.patch`
- SHA-256：`516f0d1b68ec4b4f354f950193d1fc55283e8078739d5e8184c9d8ad2fec285e`
- 修改模块：kernel/riscv64/cgemm_kernel_8x8_zvl256b.c；CGEMM normal 复数微内核
- 修改内容：用 segment load 替代跨步 load，并把临时复数乘积折叠为直接累加 FMA。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`（2026-09-16T07:27:05+02:00 Merge pull request #6036 from martin-frbg/f2c-lapack-i）
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；`make tests` 返回 0，125/125 utest 与 1473/1473 extension tests 通过，CBLAS L1/L2/L3 官方测试无新增失败。
- 官方 benchmark：`benchmark/cblas_cgemm.goto`
- 性能方法：固定 CPU7（X100），全程 2.2 GHz，单线程；每个尺寸预热 1 次、实测 3 次，取中位数。
- 明显回退阈值：中位数变化 `<= -3%`；本补丁最差变化 +42.25%，所以明显回退：否。
- 可重复提升口径：512 目标尺寸中位数至少提升 2%，且 patched 三次均高于 baseline 中位数；结果：满足。

## 性能结果

| Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---:|---|---|---:|---:|---:|
| 256 | 16034.46, 15501.38, 15699.00 | 22748.46, 22654.40, 22601.79 | 15699.00 | 22654.40 | +44.30% |
| 512 | 15541.74, 15510.84, 15780.01 | 22565.59, 22534.59, 22531.14 | 15541.74 | 22534.59 | +44.99% |
| 1024 | 15426.78, 15730.83, 15204.22 | 21934.19, 21951.76, 21944.20 | 15426.78 | 21944.20 | +42.25% |

## 最终结论

`PASS`。补丁应用、构建、官方功能测试和性能门槛均通过。
