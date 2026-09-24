# 001_bits_image_fetch_bilinear_no_repeat_8888.patch 修改与测试结果

## 结论

`PASS`（合格 patch）。该实现基于 Pixman upstream master `e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20` 重构，远程 K3 release 编译和官方完整功能测试通过；官方 `scaling-bench` 覆盖 0.10–10.00 共 991 个缩放比例，3 轮交错中 990/991 个比例的 patched 中位数更快，全部比例的中位数像素时间提升 `38.236%`，没有稳定性能回退，因此归档到 `3-fun` 成功目录。

## 修改日志

- 删除原 patch 对通用 fast-path/accessor 的 RVV hook、`_pixman_have_rvv` 全局分派以及 `pixman-private.h`/`pixman-riscv.c` 改动。
- 将实现收敛到 `pixman/pixman-rvv.c` 的现有 `rvv_iters[]` 架构，为 `a8r8g8b8` 和 `x8r8g8b8` 的 `PIXMAN_REPEAT_NONE` 双线性 affine fetch 增加 RVV iterator 条目。
- 主区域采用 RVV VLA 2x2 gather 和逐通道双线性插值；左/右边界、上下越界、x8 alpha、mask 与变换失败路径保持标量语义。
- 根据图像最大 byte offset 在 `vluxei32` 与 `vluxei64` 之间选择，32 位路径使用正确的 byte offset（像素索引乘 4），超过范围时扩展到 64 位索引。
- 保持 scale row-cache iterator 在前，避免改变现有 RVV dispatch 优先级；no-repeat 条目仅在自身 flags 匹配时生效。
- 最终 commit subject：`rvv: Add no-repeat 8888 bilinear affine fetch`。

## 远程验证

- 机器：Spacemit X100 K3，riscv64，CPU2；`scaling_governor=userspace`，测试时频率 `2200000 kHz`。
- 基线：`e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20`，远程 `/home/tiann/pixman-baseline` clean。
- 构建：`meson setup <build> <source> --buildtype=release -Drvv=enabled -Dtests=enabled`；`ninja -C <build> -j2`：119/119，通过。实现复用 normal helper 的 1 条 C90 声明顺序 warning，无编译错误。
- 功能：`meson test -C <build> --print-errorlogs`：35/35 通过，0 失败。
- 性能：官方 `/home/tiann/pixman-build-baseline/test/scaling-bench` 与 `/home/tiann/lunaworker-none-build/test/scaling-bench`；baseline 使用 `PIXMAN_DISABLE=rvv`，patched 使用 RVV，均 `taskset -c 2`，3 轮交错并通过 `flock -x /home/tiann/.pixman-benchmark.lock` 串行化。单位为 `ns/pixel`，越低越好；完整的 991 个比例、每轮原始 time/ns 数值和中位数见同目录性能 TSV。
- 全部比例中位数：baseline `10.0025 ns/pixel`，patched `7.2358 ns/pixel`，速度提升 `38.236%`。
- `0.20` 比例的中位数为 `-0.820%`，但三轮为 baseline `[9.3908, 9.4684, 9.3908]`、patched `[9.3908, 9.4684, 9.7789]`，不是稳定回退；其余 990 个比例均为 patched 中位数提升。
- 未运行 `perf record`、`perf report`、`perf annotate` 或任何指令采样。

## 输入与清理审计

- 输入 patch SHA-256：`390ec15f6b334dbd85a55be32f6d8bcc18a00bea9acdcf5cd878482f2637411e`；测试前后保持不变。
- 最终 patch SHA-256：`63ce6dad5844620e2f2c5fc08286ee3873670a8579a98d5649227594107426c8`。
- 远程测试使用独立 worktree/build；归档后删除 worktree、build、benchmark 原始临时文件和应用 diff，保留 baseline 仓库 clean。
