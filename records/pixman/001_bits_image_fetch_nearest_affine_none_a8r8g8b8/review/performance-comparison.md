# 001_bits_image_fetch_nearest_affine_none_a8r8g8b8 验证摘要

- 修复后 patch：`patch.diff`
- SHA-256：`12e71066603ff0e92db16a623f4e73ceaf773d29171c7cfb8898b0a3c72bc0f2`
- 修改模块：`pixman-fast-path.c`、`pixman-private.h`、`pixman-riscv.c`、`pixman-rvv.c`
- 修改内容：保留 a8r8g8b8 nearest RVV gather，修正字节偏移、越界 masked load、负 stride 和 64-bit offset fallback；加入短行阈值并限制为 scale transform，避免旋转和错切回退。
- Baseline commit：Pixman `c47e712c165d988a0b664a295471823a1a0ff577`
- 构建配置：Meson release，`-Drvv=enabled -Dtests=enabled -Dtimers=true -Dopenmp=disabled`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；Pixman 官方 Meson test 35/35
- 官方 benchmark：`taskset -c 2 ./test/lowlevel-blt-bench -m 1000 -n -c add_8888_8888`
- 性能方法：固定 K3 CPU2、2.2 GHz；baseline/patched 各预热 1 次，正式交错实测 3 次并取中位数
- 明显回退阈值：中位数变化小于 -5%；所有指标均为正向变化，所以明显回退：否
- 可重复提升口径：至少一个指标提升大于 3% 且 3/3 胜出；结果：满足

## 性能结果

单位：MPix/s，越高越好。

| Metric | Baseline runs | Patched runs | Baseline median | Patched median | Change |
|---|---|---|---:|---:|---:|
| L1 | 247.889, 257.802, 248.306 | 422.116, 432.566, 424.803 | 248.306 | 424.803 | +71.08% |
| L2 | 237.327, 238.472, 238.467 | 393.255, 391.024, 390.335 | 238.467 | 391.024 | +63.97% |
| M | 218.605, 219.010, 218.567 | 347.166, 344.984, 345.714 | 218.605 | 345.714 | +58.15% |
| HT | 133.867, 136.664, 138.890 | 141.733, 145.707, 146.524 | 136.664 | 145.707 | +6.62% |
| VT | 73.4676, 71.9580, 73.7167 | 76.9483, 82.9094, 81.4598 | 73.4676 | 81.4598 | +10.88% |
| R | 65.4582, 64.2455, 65.1721 | 65.6180, 67.8600, 67.1778 | 65.1721 | 67.1778 | +3.08% |
| RT | 23.8961, 22.6689, 24.4156 | 23.0421, 25.0760, 24.1887 | 23.8961 | 24.1887 | +1.22% |

## 最终结论

`PASS`。Patch 应用、构建、官方功能测试和性能门槛均通过。
