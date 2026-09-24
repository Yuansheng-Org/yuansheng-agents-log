# 001_bits_image_fetch_bilinear_affine_normal_a8r8g8b8.patch 修改与测试结果

## 结论

`PASS`（合格 patch）。该实现基于 Pixman upstream master `e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20` 重构，远程 K3 release 编译和官方完整功能测试通过；相关官方 `affine-bench` 的 5 轮交错中位数吞吐提升 `0.685%`，因此归档到 `3-fun` 成功目录。

## 修改日志

- 删除原 patch 对 `pixman-fast-path.c` 的通用 RVV hook、`_pixman_have_rvv` 全局分派以及 `pixman-private.h`/`pixman-riscv.c` 改动。
- 将实现收敛到 `pixman/pixman-rvv.c` 的现有 `rvv_iters[]` 架构，新增 `rvv_fetch_bilinear_affine_normal_a8r8g8b8`，不污染通用 fast-path/accessor 代码。
- 使用 RVV VLA 循环生成 affine 坐标、NORMAL repeat 包裹坐标、四 tap gather 和逐通道双线性插值；支持 mask、负 rowstride 和变换失败清零路径。
- 根据图像最大 byte offset 在 `vluxei32` 与 `vluxei64` 之间选择，32 位路径使用正确的 byte offset（像素索引乘 4）。
- 保持 scale row-cache iterator 在前，新增 normal-repeat iterator 在现有 affine cover iterator 后，遵循 RVV iterator dispatch 顺序。
- 最终 commit subject：`rvv: Add normal-repeat a8r8g8b8 bilinear affine fetch`。

## 远程验证

- 机器：Spacemit X100 K3，riscv64，CPU2；`scaling_governor=userspace`，当前/锁定频率 `2200000 kHz`。
- 基线：`e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20`，远程 `/home/tiann/pixman-baseline` clean。
- 构建：`meson setup <build> <source> --buildtype=release -Drvv=enabled -Dtests=enabled`；`ninja -C <build> -j2`：119/119，通过。实现产生 1 条 C90 声明顺序 warning，无编译错误。
- 功能：`meson test -C <build> --print-errorlogs`：35/35 通过，0 失败。
- 性能：官方 `affine-bench -b 1 0 0 1 src a8r8g8b8 a8r8g8b8`；baseline 使用同一 release 构建、`PIXMAN_DISABLE=rvv` 以测量上游 normal scalar fetch，patched 使用 RVV 构建；两者均 `taskset -c 2`，5 轮交错并通过 `flock -x /home/tiann/.pixman-benchmark.lock` 串行化。单位为 MPix/s，越高越好；完整原始值和中位数见同目录性能 TSV。
- baseline 中位数：1155.58 MPix/s；patched 中位数：1163.50 MPix/s；变化：`+0.685%`。5 轮中 4 轮 patched 更快，未观察到稳定性能回退。
- 未运行 `perf record`、`perf report`、`perf annotate` 或任何指令采样。

## 输入与清理审计

- 输入 patch SHA-256：`a8994e2af968911a83569702c3a099644e3a6cf2f3d4df835c7f40135100b48e`；测试前后保持不变。
- 最终 patch SHA-256：`c0f9317ffd6ed22ddd724ae1f7122ff3b6870d9fe0fd4a79381097bac2a3c015`。
- 远程测试使用独立 worktree/build；归档后删除 worktree、build、benchmark 原始临时文件和应用 diff，保留 baseline 仓库 clean。
