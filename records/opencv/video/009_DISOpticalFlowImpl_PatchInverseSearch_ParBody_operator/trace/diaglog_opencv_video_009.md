# RISC-V 性能诊断报告 — opencv/video rank 009

## Phase 0 — 硬件与构建基线

- 硬件: SpacemiT K3 (spacemit-x100), RVV 1.0, VLEN=256 (vlenb=32), OoO
- Build ISA: v 存在, 无 zvbb
- 函数: cv::DISOpticalFlowImpl::PatchInverseSearch_ParBody::operator() — libopencv_video.so.5.1.0 (库代码)
- 语义: DIS 光流金字塔层的 patch 逆搜索 (patch 匹配/细化): 向量化计算 patch 差异 → vfmv.f.s 归约取标量 → 分支判断。多处向量运算后立即取标量

## Phase 1 — 采样与事件

- 热点: vfmv.f.s (归约取标量) 合计 ≈46% (26.91%+15.03%+多路各 2-3%) + vsetvli 3.56% + 分支 bnez 2.48%

## Phase 2 — 热点指令定位

| 地址 | 指令 | 占比 | 说明 |
|---|---|---|---|
| 184c0 | vfmv.f.s fa3,v3 | 26.91% | 归约取标量 (含采样归属放大) |
| 19486 | vfmv.f.s fa3,v3 | 15.03% | 归约取标量 |
| 18464 | vsetvli | 3.56% | VL 设置 |
| 多路 vfmv.f.s | ×7 | ~20% | 各 patch 计算取标量 |
| 184cc | bnez a4 | 2.48% | 分支 |

关键形态: DIS patch 逆搜索已向量化 (patch 差异计算), 但每 patch 向量运算后立即 vfmv.f.s 取标量 (合计 ≈46%); patch 尺寸小 (向量利用受限), 频繁取标量 + 分支为特征。

## Phase 3 — 瓶颈分类

已向量化的 DIS patch 搜索 (频繁归约取标量):
1. vfmv.f.s 取标量 ≈46% (向量→标量提取, 含采样归属放大; patch 运算小尺寸向量化收益受限)
2. patch 差异计算已向量化 (vsetvli)
3. 分支判断 (bnez) 为搜索语义

## Phase 4 — 根因链

PatchInverseSearch (已向量化): 向量 patch 差异 → vfmv.f.s 取标量 (≈46%) → 分支判断。patch 小尺寸使向量化提取开销占比高。

根因类型: 已向量化 patch 搜索 (归约取标量频繁)

## Phase 5 — 修复方向 (实现层面, 不改语义)

1. patch 尺寸与向量化匹配评估: patch 运算小尺寸时标量可能更优; 或批量处理多 patch (跨 patch lane) 摊薄 vfmv.f.s
2. 归约取标量优化 (多值一次取或保持向量)

预期收益: 跨 patch 向量化/取标量优化约 15%。

## Phase 6 — 置信度与判定

- rvvPattern: vector-state (已向量化 patch 搜索; 归约取标量频繁)
- patternConfidence: MEDIUM
- benefitUpperbound: 0.15
- overallConfidence: 0.52
- recommendToCraft: conditional
- evidenceNote: DIS 光流 patch 逆搜索; perf_stat 为用例级共享指标