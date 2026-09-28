# 002_bits_image_fetch_affine_no_alpha 验证摘要

- 修复后 patch：`patch.diff`
- SHA-256：`62f58da5e4a8e825961451e1664d9bd701c3d264535dced5d24b9b8c1bc5608b`
- 修改模块：`pixman-bits-image.c`、`pixman-private.h`、`pixman-riscv.c`、`pixman-rvv.c`
- 修改内容：原补丁错误地以 nearest gather 替代 bilinear affine，并重复递增迭代器行号；修复后只匹配 a8r8g8b8/NONE/bilinear/no-mask，使用四 tap masked gather 与精确逐通道插值，并支持负 stride、32/64-bit offset 和短行标量阈值。
- Baseline commit：Pixman `c47e712c165d988a0b664a295471823a1a0ff577`
- 构建配置：Meson release，`-Drvv=enabled -Dtests=enabled -Dtimers=true -Dopenmp=disabled`
- 编译器：`gcc (Bianbu 15.2.0-16ubuntu1bb1) 15.2.0`
- Patch 应用：PASS
- 编译：PASS
- 功能测试：PASS；Pixman 官方 Meson test 35/35
- 官方 benchmark：`taskset -c 2 ./test/lowlevel-blt-bench -m 1000 -b -c pixbuf`
- 性能方法：固定 K3 CPU2、2.2 GHz；baseline/patched 各预热 1 次，正式交错实测 3 次并取中位数
- 明显回退阈值：中位数变化小于 -5%；本 patch 最差 -0.53%，所以明显回退：否
- 可重复提升口径：至少一个指标提升大于 3% 且 3/3 胜出；R +3.67%、3/3 胜出，结果：满足

## 性能结果

单位：MPix/s，越高越好。

| Metric | Baseline runs | Patched runs | Baseline median | Patched median | Change |
|---|---|---|---:|---:|---:|
| L1 | 12.4612, 12.2749, 12.2762 | 12.1709, 12.6598, 12.2141 | 12.2762 | 12.2141 | -0.51% |
| L2 | 12.2680, 12.2374, 12.2546 | 12.2242, 12.2708, 12.3429 | 12.2546 | 12.2708 | +0.13% |
| M | 12.1494, 12.1942, 12.1274 | 12.6247, 12.4747, 12.5142 | 12.1494 | 12.5142 | +3.00% |
| HT | 11.4128, 11.4008, 11.4174 | 11.4390, 11.8611, 11.7096 | 11.4128 | 11.7096 | +2.60% |
| VT | 11.4536, 11.4244, 11.4437 | 11.9035, 11.7307, 11.7349 | 11.4437 | 11.7349 | +2.54% |
| R | 11.0527, 11.0533, 11.0792 | 11.1321, 11.4589, 12.0024 | 11.0533 | 11.4589 | +3.67% |
| RT | 8.66841, 8.69287, 8.70741 | 8.61622, 8.93103, 8.64672 | 8.69287 | 8.64672 | -0.53% |

## 最终结论

`PASS`。Patch 应用、构建、官方功能测试和性能门槛均通过。
