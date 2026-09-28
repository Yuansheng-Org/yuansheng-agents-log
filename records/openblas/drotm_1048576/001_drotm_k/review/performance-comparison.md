# 001_drotm_k 验证摘要

- 原始 patch：`craft/openblas-level1/001_drotm_k/craft/patch.diff`
- SHA-256：`75dfba64c920fa9de2b0224add1800a27aa75ca4d0e865eedc0bf3b53cc95912`
- 修改模块：`kernel/riscv64/rotm_rvv.c`
- 修改内容：为 `dflag>0`、单位步长路径增加连续 `vle/vse` 快路径，替代 `vlse/vsse`。
- Baseline：OpenBLAS `9e857d0055984e6839fc0aba94b5943ae12ddb5d`
- 测试机：SpaceMiT K3/X100，RISC-V 64，GCC 15.2；固定 CPU7，所有采样前后均为 2.2 GHz。
- 构建配置：`TARGET=RISCV64_ZVL256B NOFORTRAN=1 NO_LAPACK=1 USE_OPENMP=0 NUM_THREADS=16`
- Patch 应用：PASS；实际应用后的 diff 与原始 patch 字节一致。
- 编译：PASS。
- 功能测试：PASS；官方 `make tests` 返回 0，125/125 utest、1473/1473 extension tests 及 CBLAS L1/L2/L3 测试通过。
- 官方 benchmark：`benchmark/srotm.goto`、`benchmark/drotm.goto`；默认 `param[0]=1`，直接覆盖本 patch 修改的 `dflag>0` 分支。
- 性能方法：`OPENBLAS_INCX=1 OPENBLAS_INCY=1 OPENBLAS_LOOPS=10000`，单线程；256/512/1024 各预热 1 次、实测 3 次并取中位数。
- PASS 门槛：共享模板的每个入口在 512 的中位数提升 ≥2%，三次 patched 均高于 baseline 512 中位数，且任一尺寸无 ≤−3% 回退。

## 性能结果

| Benchmark | Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---|---:|---|---|---:|---:|---:|
| drotm | 256 | 3180.56, 3183.74, 3205.95 | 4971.69, 4976.01, 5077.66 | 3183.74 | 4976.01 | +56.29% |
| drotm | 512 | 3356.06, 3331.87, 3306.38 | 5516.53, 5480.08, 5520.87 | 3331.87 | 5516.53 | +65.57% |
| drotm | 1024 | 3409.54, 3432.65, 3429.35 | 5825.25, 5835.43, 5811.42 | 3429.35 | 5825.25 | +69.86% |
| srotm | 256 | 3167.09, 3159.03, 3102.98 | 8847.13, 8473.70, 9208.88 | 3159.03 | 8847.13 | +180.06% |
| srotm | 512 | 3285.99, 3336.20, 3331.91 | 10472.96, 10742.75, 10760.27 | 3331.91 | 10742.75 | +222.42% |
| srotm | 1024 | 3378.84, 3402.66, 3403.55 | 11501.27, 11630.72, 11563.87 | 3402.66 | 11563.87 | +239.85% |

## 最终结论

`PASS`。Patch 应用、编译、完整官方功能测试和所有共享入口的性能门槛均通过；512 中位数提升为 `drotm +65.57%`、`srotm +222.42%`，所有 patched 实测值均高于对应 baseline 中位数，且无回退。
