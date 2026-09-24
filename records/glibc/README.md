# glibc：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`Glibc`；目录标识：`glibc`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 2      | 33     | 33               | 4      | 2             | 原图空白／待补录     |

## 已筛选补丁的产物归档

本分支仅整理 `add-glibc` 已筛选的 3 个glibc补丁目录，按 `records/glibc/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [001_GI_wcschr](wcsmbs-benchset/001_GI_wcschr/) | `wcsmbs-benchset` | `trace/glibc/wcsmbs-benchset/001_GI_wcschr` | `bp-glibc-wcsmbs-benchset-001` |
| [006_strrchr_vector](string-benchset/006_strrchr_vector/) | `string-benchset` | `trace/glibc/string-benchset/006_strrchr_vector` | `bp-glibc-string-benchset-006` |
| [017_memcmp_vector](string-benchset/017_memcmp_vector/) | `string-benchset` | `trace/glibc/string-benchset/017_memcmp_vector` | `bp-glibc-string-benchset-017` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。

`001_GI_wcschr` 的 Candidate ID `pc-bp-glibc-wcsmbs-benchset-001` 与 AI Review 引用 `pc-001_GI_wcschr` 不一致；原始文件已保留。

`017_memcmp_vector` 的 Candidate ID `pc-bp-glibc-string-benchset-017` 与 AI Review 引用 `pc-017_memcmp_vector` 不一致；原始文件已保留。
