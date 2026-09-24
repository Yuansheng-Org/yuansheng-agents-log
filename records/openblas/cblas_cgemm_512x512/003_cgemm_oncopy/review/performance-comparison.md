# 003_cgemm_oncopy 验证摘要

- 原始 patch：`003_cgemm_oncopy.patch`
- SHA-256：`c440a51c6762d5d9b580cdd9960af4fc83620f4eb4b00b2f80194b722f138d5b`
- 修改模块：kernel/riscv64/zgemm_ncopy_8_rvv.c、KERNEL.RISCV64_ZVL256B；CGEMM oncopy packing
- 修改内容：新增 RVV segment load/store packing 实现并将 CGEMMONCOPY 绑定到该实现。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`（2026-09-16T07:27:05+02:00 Merge pull request #6036 from martin-frbg/f2c-lapack-i）
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；`make tests` 返回 0，125/125 utest 与 1473/1473 extension tests 通过，CBLAS L1/L2/L3 官方测试无新增失败。
- 官方 benchmark：`benchmark/cblas_cgemm.goto`
- 性能方法：固定 CPU7（X100），全程 2.2 GHz，单线程；每个尺寸预热 1 次、实测 3 次，取中位数。
- 明显回退阈值：中位数变化 `<= -3%`；本补丁最差变化 -0.53%，所以明显回退：否。
- 可重复提升口径：512 目标尺寸中位数至少提升 2%，且 patched 三次均高于 baseline 中位数；结果：满足。

- 额外优化级别验证：真实 `COMMON_OPT=-O0` 与 `COMMON_OPT=-O3` 均构建通过，且分别通过同一套官方测试；编译日志确认目标文件分别由 `-O0`、`-O3` 编译。

## 性能结果

| Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---:|---|---|---:|---:|---:|
| 256 | 16034.46, 15501.38, 15699.00 | 15716.28, 15393.06, 15616.40 | 15699.00 | 15616.40 | -0.53% |
| 512 | 15541.74, 15510.84, 15780.01 | 16159.83, 16000.04, 16208.96 | 15541.74 | 16159.83 | +3.98% |
| 1024 | 15426.78, 15730.83, 15204.22 | 15461.78, 15689.53, 15603.20 | 15426.78 | 15603.20 | +1.14% |

## 最终结论

`PASS`。补丁应用、构建、官方功能测试和性能门槛均通过。
