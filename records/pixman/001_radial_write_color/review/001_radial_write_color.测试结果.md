# radial gradient RVV 修订与测试结果

- 结论：合格。
- 基线：Pixman `e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20`。
- 测试机：K3 RISC-V，GCC 15.2.0，CPU2，userspace governor，2.2 GHz。
- 编译：通过，Meson/Ninja 完成 118 个构建步骤。
- 功能：通过，Meson 全量测试 35/35，Fail 0。
- 图像验证：基线与修订版 `radial.png` 的 SHA-256 均为 `6cf120eb32d67cc4fbb99883fb43795b557c831c4e831eae614d3e8448f976d3`。
- 性能工具：仅运行官方 benchmark，未运行 `perf` 或指令采样。

## 修改日志

1. 原 patch 仅增加一个没有调用者的 helper，且 RVV widening intrinsic 的 LMUL 类型不匹配，无法编译也无法产生性能收益。
2. 将 RVV helper 改为从 `double` 数组批量求解非退化二次方程，使用匹配的 `f64m4`、`i64m4`、`u8mf2` 和 `b16` 类型。
3. 在窄像素、affine、无 mask、`a != 0` 且运行时确认 RVV 的 radial scanline 路径中接入 helper。
4. 精确的 32.32 固定点 `b/c/dc` 递推及有状态 gradient walker 仍按原顺序标量执行；mask、wide、退化方程、projective 和非 RVV CPU 均保留原标量路径。
5. 修复原 helper 未在批次之间推进递推状态的问题，避免重复计算首批像素。

## 性能判定

命令：`taskset -c 2 ./test/radial-perf-test`

单位为秒，越低越好。

| 版本 | 第 1 次 | 第 2 次 | 第 3 次 | 中位数 |
|---|---:|---:|---:|---:|
| 基线 | 0.008715 | 0.008748 | 0.010088 | 0.008748 |
| 修订 | 0.005958 | 0.005936 | 0.005960 | 0.005958 |

中位耗时降低 31.89%，即 1.468 倍速度；三次修订结果均优于对应基线，并且输出图像完全一致，因此满足合格条件。
