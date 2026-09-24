# `010_wcschrnul` 双平台短输入优化验证

测试日期：2026-09-20（Asia/Shanghai）

## 结论

最终方案在 RVV 主循环前对扫描位置 0–4 做 5 次直线化标量探测，使用汇编期 `.rept 5` 展开，不包含厂商、型号或运行时平台判断。其余输入继续进入原有 `e32, LMUL=8` fault-only-first 向量循环。

- K3：0–4 从原补丁回退 98.16% 改为提升 14.82%，overall 从提升 40.23% 增至 43.69%。
- LX：0–4 从原补丁回退 29.41% 改为提升 36.78%，overall 从提升 43.44% 增至 47.48%。
- 两台机器的 `make test t=wcsmbs/test-wcschrnul` 均 PASS。
- 残余风险：K3 的 5–16 桶回退 20.52%，比原补丁该桶回退 0.35% 更差。直线化前缀消除了目标 0–4 回退，但把向量启动边界后的部分成本移到了 5–16；最终 patch 和报告均保留这一结果。

## 最终性能

正值表示执行时间下降，即 `(1 - vector / generic) * 100`；每个 case 先取 6 次 vector timing 和 generic timing 的各自中位数，再对桶内 ratio 取几何平均。

| 扫描位置 | case 数 | K3 最终 | K3 原补丁 | LX 最终 | LX 原补丁 |
|---|---:|---:|---:|---:|---:|
| 0–4 | 30 | 提升 14.82% | 回退 98.16% | 提升 36.78% | 回退 29.41% |
| 5–16 | 76 | 回退 20.52% | 回退 0.35% | 提升 16.43% | 提升 20.73% |
| 17–64 | 149 | 提升 41.19% | 提升 41.33% | 提升 52.32% | 提升 46.97% |
| 65–256 | 125 | 提升 58.80% | 提升 53.37% | 提升 57.98% | 提升 54.60% |
| >256 | 100 | 提升 61.37% | 提升 60.53% | 提升 46.72% | 提升 50.52% |
| Overall | 480 | 提升 43.69% | 提升 40.23% | 提升 47.48% | 提升 43.44% |

K3 的 6 轮 overall 分别为：

```text
41.49%, 46.56%, 46.72%, 42.27%, 46.59%, 41.89%
```

其中 411 个 case 提升、69 个回退；最差 case 为 `length=6, pos=5, seek_char=851, max_char=1121, alignment=0`，回退 68.96%。

LX 的 6 轮 overall 分别为：

```text
48.83%, 49.04%, 47.92%, 46.34%, 47.40%, 47.93%
```

其中 444 个 case 提升、35 个回退、1 个持平；最差 case 为 `length=7, pos=6, seek_char=0, max_char=2147483647, alignment=0`，回退 13.26%。

## 迭代记录

没有频繁 clean。两台机器各只做了一次初始 patched 完整构建，后续方案均复用同一个 out-of-tree build 做增量构建。

1. 首字符标量探测 + `e32,m1` 首块 + `e32,m8` 主循环：LX 的 0–4 提升 40.16%，但 K3 仍回退 20.99%，未采用。
2. 首字符探测 + 一个 VLEN 自适应标量前缀 + `e32,m8`：K3 的 0–4 提升 1.55%，但 LX 仍回退 2.96%，且 K3 的 5–16 回退 20.89%，未采用。
3. 最终方案：5 次直线化标量探测后进入 `e32,m8`。两边 0–4 均转为提升，overall 也均高于原补丁。

失败方案的原始结果分别保存在远端 `results/variant-b/` 和 `results/variant-c/`，没有混入最终汇总。

## 源码、隔离与测试方法

- 两台机器的隔离根目录均为 `<HOME>/glibc-wcschrnul-portable-v2-20260920`。
- 官方上游为 `https://sourceware.org/git/glibc.git`，基线 commit 为 `d5bd0f22305277fa0d431bb9d733aa9456706573`。
- baseline 与 patched 使用独立源码目录；构建为独立 out-of-tree build。
- 没有执行 `make install`，隔离 install 前缀下不存在系统 libc，未修改两台机器的系统 glibc。
- 正确性只运行 `make test t=wcsmbs/test-wcschrnul`，没有运行完整 `make check`。
- 性能只构建并运行 `bench-wcschrnul`，没有运行完整 `make bench`。
- benchmark 固定 CPU 2，一次预热、6 次正式运行；每份正式输出包含 480 个 case，并通过 JSON 校验。
- LX 出现 Hadoop/YARN 高负载时没有采样；等待作业结束并确认 CPU 2 连续采样约 99.6% idle 后才运行最终 benchmark。

## Patch 一致性与污染检查

- 最终 patch 只包含预期的 7 个文件，`git diff --check` 通过。
- 两台机器 7 个已测试文件的 SHA-256 逐项一致。
- 最终 patch 在两台 pristine baseline 上均通过 `git apply --check`。
- 两台 pristine baseline 均保持 clean；K3 的已测试 patched 工作树生成提交后 clean，LX 的 patched 工作树仅有预期 7 个路径变化。
- 最终归档 patch SHA-256（邮箱匿名化后）：`83517398b5f235a900b42bb4a7178847a943b0e163346cdeef462427bf33876b`。

## 远端产物

两台机器的结果目录均为：

```text
<HOME>/glibc-wcschrnul-portable-v2-20260920/results
```

主要文件：

- `final.patch`
- `v4-test-wcschrnul.log`
- `v4-warmup.json`
- `v4-run-1.json` … `v4-run-6.json`
- `v4-analysis.json`
- `v4-per-case.csv`
- `v4-performance-manifest.tsv`
- `incremental-build-v4.log`
- `build-bench-wcschrnul-v4.log`
