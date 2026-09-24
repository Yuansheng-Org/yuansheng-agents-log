# 007_fetch_scanline_a8.patch 修改与测试结果

## 结论

`PASS`。该 patch 已基于 Pixman upstream master `e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20` 重构，在远程 K3 完成 release 编译、完整功能测试和目标路径 benchmark；串行复测的 A8 source workload 三轮中位数均提升。

## 修改日志

- 删除原 patch 对 `pixman-access.c` 通用 accessor 的 RVV 硬替换。
- 删除 `_pixman_have_rvv` 全局状态和 `pixman-riscv.c` 改动。
- 在 `pixman-rvv.c` 增加 `rvv_fetch_a8()`，采用运行时 `vsetvl_e8m1` 的 VLA 循环，将 A8 alpha 字节扩展为 `a8r8g8b8`。
- 在现有 `rvv_iters[]` 中增加精确的 `PIXMAN_a8`、`FAST_PATH_ID_TRANSFORM`、`FAST_PATH_BITS_IMAGE`、`FAST_PATH_SAMPLES_COVER_CLIP_NEAREST` 表项；generic accessor 和其他格式路径不变。
- commit subject：`rvv: Add vectorized A8 source scanline fetch`，包含性能数据、`Signed-off-by` trailer。

## 远程验证

- 机器：Spacemit X100 K3，riscv64，CPU 2，userspace governor，固定 2.2 GHz（2200000 kHz）。
- 构建：`meson setup <build> <source> -Dbuildtype=release -Dtests=enabled -Ddemos=disabled -Drvv=enabled`；`ninja -C <build>`：通过。
- 功能：`meson test -C <build> --print-errorlogs`：35/35 通过，0 失败。
- 目标覆盖：官方 `lowlevel-blt-bench -c src_8_0565` 和 `-c src_8_8888` 命中 A8 source 路径；串行复测两种 case 的 L1/L2/M/HT/VT/R/RT 均无回退。
- 性能：在 `/home/tiann/.pixman-benchmark.lock` 上使用 `flock -x`，固定 CPU 2/2.2 GHz，baseline/patched 交错运行 3 轮；原始数值及中位数见同目录 `007_fetch_scanline_a8.性能数据.tsv`。单位为 MPix/s，数值越高越好。
- 最终 patch 在干净 baseline worktree 上执行 `git apply --check`：通过。
- 未运行 `perf record`、`perf report`、`perf annotate` 或任何指令采样。

## 输入完整性

输入 patch SHA-256：`934c03252bd0b21ecb0f774d72003897b7f3d3b3ea2cd4eff18d5394512fc370`。测试前后保持不变。
