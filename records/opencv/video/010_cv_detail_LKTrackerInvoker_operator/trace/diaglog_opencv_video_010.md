# RISC-V 性能诊断报告 — opencv/video rank 010

## Phase 0 — 硬件与构建基线

- 硬件: SpacemiT K3 (spacemit-x100), RVV 1.0, VLEN=256 (vlenb=32), OoO
- Build ISA: v 存在, 无 zvbb
- 函数: cv::detail::LKTrackerInvoker::operator() — libopencv_video.so.5.1.0 (库代码)
- 语义: Lucas-Kanade 光流跟踪 (金字塔 LK): 逐特征点迭代——梯度/图像读取 (lh/lbu) + 坐标转换 (fcvt.s.w) + 索引计算 (mulw/slli) + 部分向量化 (vsetvli e16/e8)

## Phase 1 — 采样与事件

- 热点: 坐标/索引计算 (mulw 6.01% + fcvt.s.w 6.28% + slli 3.50% ≈16%) + 部分向量化 (vsetvli 9.96%+3.32% ≈13%) + 数据读取 (lh/lbu ≈5%) + 分支 (bnez 4.22%)

## Phase 2 — 热点指令定位

| 地址 | 指令 | 占比 | 说明 |
|---|---|---|---|
| 305a0 | vsetvli e16 | 9.96% | 向量配置 (含采样归属放大) |
| 30a84 | bnez t1 | 4.22% | 迭代分支 |
| 3011a/30146 | mulw | 6.01% | 索引计算 |
| 301ac/301b4 | fcvt.s.w | 6.28% | 坐标转换 |
| 305b8 | slli | 3.50% | 索引 |
| 30594 | vsetvli e8 | 3.32% | 向量配置 |
| 30122/3012e | lh/lbu | 5.11% | 梯度/图像读取 |

关键形态: LK 跟踪为标量为主 (特征点迭代: 坐标/索引计算 + 梯度图像读取) + 部分向量化 (vsetvli e16/e8 的 patch 计算)。迭代求解 (bnez) 为金字塔 LK 语义。

## Phase 3 — 瓶颈分类

LK 跟踪 (标量为主 + 部分向量化):
1. 坐标/索引计算 ≈16% (mulw/fcvt.s.w/slli, 特征点数据相关)
2. 部分向量化 (vsetvli e16/e8 ≈13%) patch 梯度计算
3. 梯度/图像读取 (lh/lbu ≈5%)
4. 迭代求解 (bnez) 数据相关

## Phase 4 — 根因链

LKTracker (标量为主): 特征点坐标/索引 → 梯度图像读取 → 部分向量化 patch 计算 → 迭代求解。

根因类型: LK 跟踪标量为主 (特征点迭代数据相关, 部分向量化)

## Phase 5 — 修复方向 (实现层面, 不改语义)

1. 坐标/索引计算保持 (特征点数据相关)
2. 已向量化部分 (e16/e8) 保持
3. 迭代求解保持 (LK 语义)

## Phase 6 — 置信度与判定

- rvvPattern: 无强匹配 (LK 跟踪标量为主 + 部分向量化)
- patternConfidence: MEDIUM
- benefitUpperbound: 0.08
- overallConfidence: 0.50
- recommendToCraft: conditional
- evidenceNote: LK 光流跟踪 (拼接/跟踪路径); perf_stat 为用例级共享指标