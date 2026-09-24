# `001_rvv_composite_over_n_8888_8888_ca.patch` 修订验证报告

## 结论

**合格（PASS）**。修订 patch 可在上游基线干净应用，RISC-V release 编译通过，Pixman 测试 35/35 通过，目标函数命中得到验证；两组正式性能负载各完成 5 轮交错对比，主要缓存/流式场景提升 3.57%～26.06%，最差项目为 -2.72%，未达到规范定义的明显回退阈值（-5%）。

原始 patch 保持不变：

`/home/tiann/yuansheng-agent-debug/craft/pixman/001-pixman-pr/1-success/001_rvv_composite_over_n_8888_8888_ca.patch`

原始文件 SHA-256：`7be017a3b2990b58fa6cd4d37e97c3516644333ef1445ad62ba28b476daa1305`

## 修订内容

- 输出为可直接提交上游的标准 `git format-patch`，补齐主题、动机、实现说明、测试环境、复现命令、性能数据和 `Signed-off-by`。
- 在 `rvv_composite_over_n_8888_8888_ca()` 中缓存一次 `vlmax`，供固色广播和满向量循环共同使用。
- 将每行处理拆成 `vlmax` 满向量循环和至多一次动态 `vsetvl` 的尾部处理，保持 VLA，不假定固定 VLEN。
- 仅修改 `pixman/pixman-rvv.c`；代码差异为 24 行新增、2 行删除。

基线 commit：`e1f49d9e6665ce354b90a189bf7bb5bbde9a1f20`  
修订 commit：`a88176ac62e5a1023849262a4a82685194cc20b6`  
Stable patch-id：`f946b718d3a995bc791c6aeca68255da2eb416c4`  
修订 patch SHA-256：`29c736dbf95b404b02c666195c3d8effa73dab6150053ca18ff55d817721e5cc`

## 质量门禁

| 检查项 | 结果 | 证据 |
|---|---:|---|
| 基线干净应用 | PASS | 最终验证结果已汇总于本报告 |
| RISC-V release 编译 | PASS | 最终验证结果已汇总于本报告 |
| Pixman 功能测试 | PASS，35/35 | 最终验证结果已汇总于本报告 |
| 目标路径覆盖 | PASS，目标函数占样本 71.66% | 最终覆盖结果已汇总于本报告 |
| 性能复测 | PASS，每组 5 轮 | 下表及两份 `performance-*-summary.tsv` |
| Patch 隔离 | PASS | 基线仓库和修订 worktree 测试后均为 clean；原 patch 未修改 |

## 测试环境与方法

- 设备：SpacemiT K3，`Spacemit(R) X100`，固定 CPU 2。
- 性能测试时 CPU 2 使用 `userspace` governor，所有轮次观测频率均为 2.2 GHz。
- 编译器：Bianbu GCC 15.2.0；Meson 1.10.1；Ninja 1.13.2。
- 两个独立 build 目录来自同一上游基线；release/O3，RVV 以 `-march=rv64gcv1p0` 编译。
- 每组基线和修订版各运行 5 次，奇数轮按 baseline→patched、偶数轮按 patched→baseline 交错，结果取中位数。
- 数值单位为 MPix/s，越高越好。

复现命令：

```sh
taskset -c 2 ./test/lowlevel-blt-bench -c over_n_8888_8888_ca
taskset -c 2 ./test/lowlevel-blt-bench -b -c over_n_8888_8888_ca
```

### 直接 component-alpha OVER

| Metric | Baseline median | Patched median | Change | Patched wins |
|---|---:|---:|---:|---:|
| L1 | 364.5690 | 459.5790 | +26.06% | 5/5 |
| L2 | 352.9840 | 441.5550 | +25.09% | 5/5 |
| M | 351.1010 | 439.3950 | +25.15% | 5/5 |
| HT | 207.1490 | 232.1720 | +12.08% | 5/5 |
| VT | 178.8190 | 185.2000 | +3.57% | 5/5 |
| R | 97.2561 | 94.9562 | -2.36% | 0/5 |
| RT | 30.9784 | 30.7400 | -0.77% | 0/5 |

### Nearly 1x bilinear source transform

| Metric | Baseline median | Patched median | Change | Patched wins |
|---|---:|---:|---:|---:|
| L1 | 367.9390 | 461.6340 | +25.46% | 5/5 |
| L2 | 352.5880 | 440.2140 | +24.85% | 5/5 |
| M | 351.8940 | 439.1500 | +24.80% | 5/5 |
| HT | 208.1450 | 233.8880 | +12.37% | 5/5 |
| VT | 180.0700 | 188.5420 | +4.70% | 5/5 |
| R | 97.6480 | 94.9878 | -2.72% | 0/5 |
| RT | 31.3596 | 30.9081 | -1.44% | 0/5 |

R/RT 是唯一出现负值的项目，范围为 -0.77%～-2.72%；这里完整保留并披露，未超过 -5% 明显回退门槛。优化直接针对满向量热循环，在 L1、L2、M、HT、VT 两组测试的 5/5 轮次均获胜。

## 保留的最终结果

- `001_rvv_composite_over_n_8888_8888_ca.patch`：最终标准 patch。
- `summary.md`：构建、功能、覆盖、环境、隔离审计和性能结论。
- `performance-direct-summary.tsv`：直接 component-alpha OVER 最终性能统计。
- `performance-bilinear-summary.tsv`：Nearly 1x bilinear source transform 最终性能统计。

逐轮输出、构建/功能日志、反汇编、覆盖率中间文件和 `perf.data` 已在汇总后删除，不作为最终交付物保留。
