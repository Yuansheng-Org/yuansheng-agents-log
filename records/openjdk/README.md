# OpenJDK：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`OpenJDK`；目录标识：`openjdk`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 11     | 56     | 12               | 1      | 1             | 原图空白／待补录     |



## 已筛选补丁的产物归档

本分支仅整理 `add-openjdk` 已筛选的 1 个函数目录，按 `records/openjdk/<测试用例>/<函数>/` 归档。测例名称和 Blueprint ID 取自 Agent Debug 的 Trace；`craft/patch.diff` 与 Agent Debug 原始 Craft 补丁逐字节一致，原有专家复核文件保留。

| 测试用例 | 函数目录 | Blueprint ID |
| --- | --- | --- |
| `java.lang.Thread` | [016_Handshake_execute_HandshakeClosure_ThreadsListHandle_JavaThread](java.lang.Thread/016_Handshake_execute_HandshakeClosure_ThreadsListHandle_JavaThread/) | `bp-openjdk-java.lang.Thread-016` |

`trace/` 保留对应函数的原始追踪文件；`craft/` 补充 Plan、Candidate 和 `reviews/` 中的 AI 审核。历史统计保持原值；AI 审核与原有专家复核文件分别保留，不自动推断编译、回归或上游接收状态。
