# Go：测试与补丁追踪

资料登记日期：2026-09-23。截图软件原名：`Golang`；目录标识：`golang`。

## 截图历史汇总

| 测试用例数量 | 热点函数数量 | Agent 生成 patch 数 | 开源专家复核后补丁数量 | 已提交社区 patch 数 | 社区接收 patch 数 |
| ------:| ------:| ----------------:| ------:| ------------- | ------------ |
| 237    | 380    | 8                | 3      | 1             | 原图空白／待补录     |

## 已筛选补丁的产物归档

本分支仅整理 `add-go` 已筛选的 3 个golang补丁目录，按 `records/golang/<测试用例>/<函数>/` 组织。测例名称取自 Agent Debug Trace 的 `source.testcaseIds`；原有补丁和专家复核文件保留，新增的 Trace、Craft 中间产物来自 Agent Debug。

| 补丁目录 | 测试用例 | Agent Debug Trace 来源 | Blueprint ID |
| --- | --- | --- | --- |
| [001_crypto_internal_fips140_sha256.block.abi0](BenchmarkAddMulVVWW_words_100000_impl_asm/001_crypto_internal_fips140_sha256.block.abi0/) | `BenchmarkAddMulVVWW_words_100000_impl_asm` | `trace/go/BenchmarkAddMulVVWW_words_100000_impl_asm/001_crypto_internal_fips140_sha256.block.abi0` | `bp-go-BenchmarkAddMulVVWW_words_100000_impl_asm-001` |
| [026_internal_runtime_maps.memHashFallback](BenchmarkAddMulVVWW_words_100000_impl_asm/026_internal_runtime_maps.memHashFallback/) | `BenchmarkAddMulVVWW_words_100000_impl_asm` | `trace/go/BenchmarkAddMulVVWW_words_100000_impl_asm/026_internal_runtime_maps.memHashFallback` | `bp-go-BenchmarkAddMulVVWW_words_100000_impl_asm-026` |
| [031_math_bits_TrailingZeros](BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc/031_math_bits_TrailingZeros/) | `BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc` | `trace/go/BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc/028_nextFreeFast` | `bp-go-BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc-028` |

`craft/reviews/` 为 Agent Debug 的 AI 审核，原有 `review/` 或 `reviews/` 为同事整理的复核结果；AI 审核通过不代表编译、回归或社区接收。历史统计保持原值。

`031_math_bits_TrailingZeros` 是专家从 `craft/go/028_nextFreeFast` 候选补丁中提炼出的贡献。原始 Trace 指向 `028_nextFreeFast`，本目录保留其原始 Trace/Plan/Candidate/AI 审核，并归档 Agent Debug 中 `031_math_bits_TrailingZeros` 的 `optimized-craft/` 与 `test-results/`；详见已有 `review/REVIEW.md`。

`031_math_bits_TrailingZeros` 的 Candidate ID `pc-bp-go-benchmarkmallocgc-scan-noscan-size-176-kind-mallocgc-028` 与 AI Review 引用 `pc-bp-go-benchmarkmallocgc_scan_noscan_size_176_kind_mallocgc-028` 不一致；原始文件已保留。
