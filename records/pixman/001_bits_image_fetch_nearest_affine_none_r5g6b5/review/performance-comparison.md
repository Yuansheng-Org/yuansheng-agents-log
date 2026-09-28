# 001_bits_image_fetch_nearest_affine_none_r5g6b5 验证摘要

- 修复后 patch：`patch.diff`
- SHA-256：`0afe68078cab3f2963e91bb0804c422d063d4d41fd3d143bfbb0d2d2e2f95882`
- 修改模块：`pixman-fast-path.c`、`pixman-private.h`、`pixman-riscv.c`、`pixman-rvv.c`
- 修改内容：保留 r5g6b5 nearest affine 的 RVV gather 与 565→8888 转换，修正字节偏移、fault-safe masked load、opaque alpha、负 stride 及 32/64-bit offset；加入短行阈值。
- Baseline commit：Pixman `c47e712c165d988a0b664a295471823a1a0ff577`
- 构建配置：Meson release，`-Drvv=enabled -Dtests=enabled -Dtimers=true -Dopenmp=disabled`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；Pixman 官方 Meson test 35/35
- 官方 benchmark：`taskset -c 2 ./test/lowlevel-blt-bench -m 1000 -n -c src_0565_8888`
- 性能方法：固定 K3 CPU2、2.2 GHz；baseline/patched 各预热 1 次，正式交错实测 3 次并取中位数
- 明显回退阈值：中位数变化小于 -5%；本 patch 最差 -1.75%，所以明显回退：否
- 可重复提升口径：至少一个指标提升大于 3% 且 3/3 胜出；结果：满足

## 性能结果

单位：MPix/s，越高越好。

| Metric | Baseline runs | Patched runs | Baseline median | Patched median | Change |
|---|---|---|---:|---:|---:|
| L1 | 179.687, 181.784, 180.538 | 399.020, 407.098, 391.540 | 180.538 | 399.020 | +121.02% |
| L2 | 171.561, 172.518, 172.496 | 363.133, 361.890, 361.402 | 172.496 | 361.890 | +109.80% |
| M | 172.558, 172.618, 168.805 | 367.888, 367.837, 364.018 | 172.558 | 367.837 | +113.17% |
| HT | 103.733, 99.4655, 97.6516 | 125.013, 126.505, 126.914 | 99.4655 | 126.505 | +27.18% |
| VT | 75.2243, 73.4700, 73.2324 | 80.9493, 80.5997, 82.2040 | 73.4700 | 80.9493 | +10.18% |
| R | 73.8370, 74.5835, 74.2399 | 86.3545, 85.5026, 85.8750 | 74.2399 | 85.8750 | +15.67% |
| RT | 25.9800, 26.8880, 26.8067 | 26.6114, 26.3369, 25.7771 | 26.8067 | 26.3369 | -1.75% |

## 最终结论

`PASS`。Patch 应用、构建、官方功能测试和性能门槛均通过。
