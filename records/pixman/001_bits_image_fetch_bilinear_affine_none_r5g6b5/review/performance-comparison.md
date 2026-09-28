# 001_bits_image_fetch_bilinear_affine_none_r5g6b5 验证摘要

- 修复后 patch：`patch.diff`
- SHA-256：`9df9c8b2452f40b39f87f77bace23e313a71cec5501d83eb4728368955b5bf7f`
- 修改模块：`pixman-fast-path.c`、`pixman-private.h`、`pixman-riscv.c`、`pixman-rvv.c`
- 修改内容：保留 r5g6b5 双线性 affine 的 RVV gather + blend 方向，修正字节偏移、四 tap masked gather、565 opaque alpha、精确双线性权重和负 stride；短行保留标量路径。
- Baseline commit：Pixman `c47e712c165d988a0b664a295471823a1a0ff577`
- 构建配置：Meson release，`-Drvv=enabled -Dtests=enabled -Dtimers=true -Dopenmp=disabled`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS；在干净 baseline 上 `git apply --check` 通过
- 编译：PASS
- 功能测试：PASS；Pixman 官方 Meson test 35/35，无新增失败
- 官方 benchmark：`taskset -c 2 ./test/lowlevel-blt-bench -m 1000 -b -c src_0565_8888`
- 性能方法：固定 K3 CPU2、2.2 GHz；baseline/patched 各预热 1 次，正式交错实测 3 次并取中位数
- 明显回退阈值：中位数变化小于 -5%；本 patch 最差 -1.40%，所以明显回退：否
- 可重复提升口径：至少一个相关指标中位数提升大于 3%，且 patched 3/3 胜出；结果：满足

## 性能结果

单位：MPix/s，越高越好。

| Metric | Baseline runs | Patched runs | Baseline median | Patched median | Change |
|---|---|---|---:|---:|---:|
| L1 | 34.0095, 33.8928, 33.8635 | 90.2445, 90.2215, 89.4961 | 33.8928 | 90.2215 | +166.20% |
| L2 | 33.8504, 33.8565, 33.7651 | 88.0425, 88.1603, 87.9677 | 33.8504 | 88.0425 | +160.09% |
| M | 33.9619, 33.9697, 33.6946 | 88.3445, 88.4253, 88.3294 | 33.9619 | 88.3445 | +160.13% |
| HT | 30.0585, 29.9653, 29.8551 | 42.3330, 42.6643, 41.9133 | 29.9653 | 42.3330 | +41.27% |
| VT | 27.2765, 27.2527, 27.0207 | 36.9750, 37.0320, 36.2433 | 27.2527 | 36.9750 | +35.67% |
| R | 26.7692, 26.8135, 26.8455 | 36.1985, 36.3309, 35.7891 | 26.8135 | 36.1985 | +35.00% |
| RT | 16.1467, 15.8578, 16.0761 | 16.1139, 15.8500, 15.8514 | 16.0761 | 15.8514 | -1.40% |

## 最终结论

`PASS`。Patch 应用、构建、官方功能测试和性能门槛均通过。
