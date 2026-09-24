# 004_ctrmm_kernel_LN 验证摘要

- 原始 patch：`004_ctrmm_kernel_LN.patch`
- SHA-256：`57108b44f3937c22aeb13bff60c6b07ddef10423410ca8788c6cbf6557d32e81`
- 修改模块：kernel/riscv64/ctrmm_kernel_8x8_zvl256b.c；CTRMM LN 复数微内核
- 修改内容：把临时复数乘积与 vfadd 链折叠为直接累加的带符号 FMA。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`（2026-09-16T07:27:05+02:00 Merge pull request #6036 from martin-frbg/f2c-lapack-i）
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；`make tests` 返回 0，125/125 utest 与 1473/1473 extension tests 通过，CBLAS L1/L2/L3 官方测试无新增失败。
- 官方 benchmark：`benchmark/ctrmm.goto`
- 性能方法：固定 CPU7（X100），全程 2.2 GHz，单线程；每个尺寸预热 1 次、实测 3 次，取中位数。
- 明显回退阈值：中位数变化 `<= -3%`；本补丁最差变化 +2.77%，所以明显回退：否。
- 可重复提升口径：512 目标尺寸中位数至少提升 2%，且 patched 三次均高于 baseline 中位数；结果：满足。

## 性能结果

| Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---:|---|---|---:|---:|---:|
| 256 | 15675.85, 15549.04, 15597.22 | 18061.32, 17713.72, 18361.10 | 15597.22 | 18061.32 | +15.80% |
| 512 | 16392.89, 16010.69, 16203.20 | 17620.41, 17630.56, 17646.76 | 16203.20 | 17630.56 | +8.81% |
| 1024 | 14877.64, 15023.29, 15039.81 | 15528.87, 15344.97, 15439.92 | 15023.29 | 15439.92 | +2.77% |

## 最终结论

`PASS`。补丁应用、构建、官方功能测试和性能门槛均通过。
