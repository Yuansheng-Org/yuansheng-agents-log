# pixman：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`pixman`；目录标识：`pixman`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 26     | 77     | 24               | 5      | 5             | 1            |

## 已筛选补丁的产物归档

本分支沿用 `add-pixman` 已筛选的 6 个补丁目录，并从 `yuansheng-agent-debug` 补齐对应函数的 Trace 和 Craft 中间产物；未归档该仓库中的其他 pixman 函数。目录按 `records/pixman/<测试用例>/<函数>/` 组织，测试用例名称取自对应 Blueprint 的 `source.testcaseIds`。

| 补丁目录 | 原始测试用例 | Blueprint ID |
| --- | --- | --- |
| [001_bits_image_fetch_bilinear_affine_normal_a8r8g8b8](affine-bench-bilinear-rotate-30deg/001_bits_image_fetch_bilinear_affine_normal_a8r8g8b8/) | `affine-bench-bilinear-rotate-30deg` | `bp-pixman-affine-bench-bilinear-rotate-30deg-001` |
| [001_bits_image_fetch_bilinear_no_repeat_8888](lowlevel-blt-bilinear-022-022/001_bits_image_fetch_bilinear_no_repeat_8888/) | `lowlevel-blt-bilinear-022-022` | `bp-pixman-lowlevel-blt-bilinear-022-022-001` |
| [001_radial_write_color](radial-perf-full/001_radial_write_color/) | `radial-perf-full` | `bp-pixman-radial-perf-full-001` |
| [001_rvv_composite_over_n_8888_8888_ca](lowlevel-blt-bilinear-094-094/001_rvv_composite_over_n_8888_8888_ca/) | `lowlevel-blt-bilinear-094-094` | `bp-pixman-lowlevel-blt-bilinear-094-094-001` |
| [002_fast_fetch_bilinear_cover](affine-bench-bilinear-scale-2x/002_fast_fetch_bilinear_cover/) | `affine-bench-bilinear-scale-2x` | `bp-pixman-affine-bench-bilinear-scale-2x-002` |
| [007_fetch_scanline_a8](lowlevel-blt-bilinear-068-068/007_fetch_scanline_a8/) | `lowlevel-blt-bilinear-068-068` | `bp-pixman-lowlevel-blt-bilinear-068-068-007` |

每个补丁目录中，`trace/` 保留 Agent Debug 的函数级原始文件；`craft/` 保留原有 `patch.diff`，新增 `patch-plan.json`、`patch-candidate.json` 和 `reviews/` 中的 AI 审核产物；顶层 `review/*.patch` 是 `add-pixman` 已有的专家调整后补丁。归档时核对了原有 `craft/patch.diff` 与 Agent Debug 对应文件的字节内容，以及 Blueprint → Plan → Candidate → AI Review 的 ID 引用。

上表列出 6 个已筛选目录；历史汇总中的“开源专家复核后补丁数量”仍为 5。两者统计口径尚未逐项核实，因此本次只补齐产物，不改动历史汇总，也不把 AI 审核通过视为编译、回归或社区接收的证明。
