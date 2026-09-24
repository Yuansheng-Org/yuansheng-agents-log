# AI 补丁审核报告

## 审核范围

- 补丁：`Configure`（从 `%disabled` 列表移除 `ec_nistp_64_gcc_128`，使 P-256/P-384/P-521 专用 `EC_GFp_nistp*_method` 默认启用）
- 蓝图：`bp-openssl-ecdhp256-009`（`ossl_ec_GFp_simple_ladder_step`，ecdhp256 rank 009）
- 根因（`config_mismatch`，confirmed）：P-256 ECDH 落入通用 `EC_GFp_mont_method` 路径（`ec_nistp_64_gcc_128` 默认禁用 → `ec_curve.c` P-256 条目 method=0 回退），每 ladder bit 约 23 次 generic BN 调用，编排层保存/恢复 + 参数搬运开销主导热点。
- 审核输入事实来源：`patch.diff`、`patch-plan.json`、`patch-candidate.json`、`blueprint`。

## RISC-V 架构审核

`arch-scan` 判定：`archSpecific: false`（纯配置变更，无汇编/inline-asm/RVV intrinsic/ISA 特性）。PatchCandidate 首字段已自动写入 `archReviewWarning` 警告，按规则跳过架构专项审核，仅做通用质量审核。

## 审核结果

- **根因解决**：移除 `ec_nistp_64_gcc_128` 的 default-disable，使 `ec_curve.c:2580` 的 `#elif !defined(OPENSSL_NO_EC_NISTP_64_GCC_128)` 分支成立、P-256 选择 `EC_GFp_nistp256_method`（专用 `__uint128_t` 常数时间 field 实现），`ossl_ec_GFp_simple_ladder_step` 不再被 P-256 热点调用，命中"kernel-selection 根因"。
- **约束保持**：`mustPreserve`（Montgomery ladder 常量时间结构、EC 点运算/P-256 曲线语义、`__uint128_t` 依赖、BN_CTX 语义）全部保持——`EC_GFp_nistp256_method` 按 CHANGES.md:11690 提供"constant-time single point multiplication"，且 `ecp_nistp256.c:49-51` 的 `#error "Your compiler doesn't appear to support 128-bit integer types"` 提供不支持平台上的 fail-fast，不会静默退化。
- **最小性**：变更仅删除 1 行（`ec_nistp_64_gcc_128 => "default"`），无函数语义修改。
- **产物一致性**：`PatchCandidate.gitDiff` 与 `patch.diff` 一致，`changedFiles=[Configure]`。
- **安全**：无硬编码密钥、无危险命令/路径；不改变任何密码学算法语义。

## 发现问题

| 严重度 | 类别 | 说明 |
|--------|------|------|
| suggestion | 平台兼容 | 启用后 `ecp_nistp224/256/384/521.c` 在所有平台编译；非 `__uint128_t` 平台（如 MSVC、32-bit）会触发 `#error` 编译失败（fail-fast，非静默破坏）。建议在 Configure 侧按 `__SIZEOF_INT128__` 检测做条件启用，或在文档中明确该 feature 的 128-bit 类型依赖。 |
| suggestion | 收益验证 | blueprint `alternativeExplanations` 指出 RISC-V 上 `__uint128_t` 乘法扩展为 `mulh`+`mul` 指令对，nistp256 相对 generic Montgomery 的实际收益需在目标上 A/B 实测；且本 build 的 configdata 未直接核对。 |

无 critical/major/minor 级问题。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度（架构结论有证据支撑） | PASS |
| 声明溯源（finding evidence 来自 diff/blueprint 原文） | PASS |
| 可解释性（critical/major 说明"为什么必须改"） | PASS（无 critical/major） |
| 内部一致性（reviewResult 与 findings 严重度一致） | PASS（pass 且无 critical/major） |
| 安全（无越权建议/新风险） | PASS |

## 结论

补丁准确解决 `ossl_ec_GFp_simple_ladder_step` 根因（`config_mismatch`：默认启用 `ec_nistp_64_gcc_128` 使 P-256 走专用常数时间方法），行为保持、最小，通过通用质量审核与防幻觉锚点校验（无架构代码，跳过架构专项审核）。审核通过（pass）。
