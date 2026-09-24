# `006_strrchr_vector` K3 hybrid 版本验证

测试日期：2026-09-21（Asia/Shanghai）

## 结论

- 修改后的 patch 在 K3 上通过 `make test t=string/test-strrchr`。
- 最终 7 轮/侧、2232 个 case 的 point-median 几何平均：
  - 直接提升 **25.68%**；
  - generic control 提升 **0.02%**；
  - generic 归一化提升 **25.66%**。
- 原 006 的页边界场景归一化回退 20.16%；hybrid 版本改为提升 **7.46%**。
- 页边界非零字符的 160 个 case 从回退 24.67% 改为提升 **8.89%**，且全部得到提升。
- 最终没有任何 case 归一化回退超过 5%；最差回退为 **2.31%**。
- 该版本只以 K3 为性能目标，并依赖 K3 的 e8/m2 活动元素索引可由 8 bit 表示；它不是面向任意大 VLEN 的通用上游版本。

## 修改内容

主循环按距下一个 4 KiB 边界的距离选择加载方式：

- 剩余空间至少为一个 VLMAX：使用普通 `vle8.v`；
- 距页边界不足一个 VLMAX：使用 `vle8ff.v`，随后读取实际 `vl`；
- 页边界 FOF 慢路径放在 `search_zero` 之后，避免改变零字符热路径的关键布局；
- K3 继续使用 e8/m2 的 `vid.v + vredmaxu.vs` 计算块内最后匹配位置。

最终反汇编确认：

- 普通路径包含 `vle8.v`；
- 页边界路径包含 `vle8ff.v` 和 `csrr ..., vl`；
- `search_zero` 位于 `0x78`，与原 006 的有效布局一致。

## 清理与隔离

- 测试开始前没有发现仍在运行的 `bench-strrchr` 或 `test-strrchr` 任务。
- 清除了旧 006 测试目录中的源码和两个约 1 GiB 的构建目录；历史 `results/` 保留，旧目录由约 2.4 GiB 降为约 20 MiB。
- 没有清除全局 Linux page cache，以免干扰机器上的其他任务。
- 本次隔离目录：`<HOME>/glibc-strrchr-k3-hybrid-validation-20260921`。
- baseline 与 patched 使用独立 build；baseline build 在应用 patch 前冻结保存。
- 上游 `master` 与官方远端一致：`d5bd0f22305277fa0d431bb9d733aa9456706573`。
- baseline 只完整构建一次；后续候选调整全部复用 build 做增量构建，没有 clean 或重新 configure。
- 没有运行完整 `make check`、完整 `make bench` 或 `make install`。

## Correctness

最终候选只运行目标单项：

```text
make test t=string/test-strrchr
PASS: string/test-strrchr
original exit status 0
```

## Performance 方法

- 只运行官方单项 `bench-strrchr`。
- baseline、patched 各预热一次，正式各 7 轮。
- 奇数轮 baseline → patched，偶数轮 patched → baseline。
- 固定 K3 CPU2，userspace governor，频率始终为 2.2 GHz。
- 每个样本开始前要求：load < 0.75、CPU2 busy < 5%、CPU PSI avg10 < 0.05。
- 正式样本开始前环境范围：load 0.64–0.73、CPU2 busy 0–3.96%、PSI 0–0.04。
- 14 个正式 JSON 全部通过解析校验，没有删除或筛选轮次。
- 对每个 case 分别取 7 次 timing 中位数，再计算时间比值的几何平均。
- 归一化公式：

```text
(patched vector / patched generic) /
(baseline vector / baseline generic)
```

重复参数 case 使用 JSON 顺序和 ordinal 匹配，没有按参数键去重。

## 最终结果

| 字符串长度 | case 数 | 直接结果 | generic control | 归一化结果 |
|---|---:|---:|---:|---:|
| 0–4 | 72 | 提升 21.20% | 提升 0.10% | 提升 21.12% |
| 5–16 | 312 | 提升 22.91% | 提升 0.02% | 提升 22.89% |
| 17–64 | 474 | 提升 23.53% | 提升 0.03% | 提升 23.51% |
| 65–256 | 558 | 提升 27.23% | 提升 0.01% | 提升 27.23% |
| >256 | 816 | 提升 27.24% | 提升 0.01% | 提升 27.23% |
| Overall | 2232 | **提升 25.68%** | **提升 0.02%** | **提升 25.66%** |

### 对齐场景

| 场景 | case 数 | 原 006 归一化结果 | hybrid 归一化结果 |
|---|---:|---:|---:|
| `align=0` | 1824 | 提升 25.43% | 提升 27.20% |
| 普通非零对齐 | 216 | 提升 26.89% | 提升 27.03% |
| 页边界 `align=4080–4095` | 192 | 回退 20.16% | **提升 7.46%** |
| 页边界、`seek=0` | 32 | 基本不变 | 基本不变（提升 0.004%） |
| 页边界、`seek=23` | 160 | 回退 24.67% | **提升 8.89%** |

### 分布与稳定性

- 2023 个 case 提升、180 个回退、29 个相等。
- 回退超过 5%：**0 个**，原 006 为 160 个。
- 最差归一化 case：`len=24, pos=23, align=4084, freq=1, seek=0, max_char=127`，回退 2.31%；目标 vector 直接时间没有变化，差异来自 generic control。
- 7 轮配对归一化结果范围：提升 22.52%–26.40%，中位数提升 25.63%。

## 与原 006 对比

| 指标 | 原 006 | hybrid 版本 |
|---|---:|---:|
| Overall 归一化 | 提升 22.45% | **提升 25.66%** |
| 页边界 | 回退 20.16% | **提升 7.46%** |
| 页边界非零搜索 | 回退 24.67% | **提升 8.89%** |
| 回退超过 5% | 160 | **0** |

## 产物

K3：`<HOME>/glibc-strrchr-k3-hybrid-validation-20260921/results`

- 最终 patch：`006-strrchr-k3-hybrid.patch`
- 最终正式数据：`baseline-run-1.json` … `baseline-run-7.json`、`patched-run-1.json` … `patched-run-7.json`
- 汇总：`analysis.json`、`per-case.csv`、`performance-manifest.tsv`
- 中间候选的完整数据分别保存在 `hybrid-v1/` 和 `hybrid-v2/`，未与最终结果混合。

最终归档 patch SHA-256（邮箱匿名化后）：

```text
38250fbe1e14f1d06dff360956718e1bb01ac88eedfb84c3649ffc0f126e8d0e
```
