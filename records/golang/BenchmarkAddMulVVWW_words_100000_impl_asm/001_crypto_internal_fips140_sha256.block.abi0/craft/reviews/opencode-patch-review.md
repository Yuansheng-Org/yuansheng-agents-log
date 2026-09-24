# AI 补丁审核报告

## 审核范围

- 蓝图：`.yuansheng/trace/go/BenchmarkAddMulVVWW_words_100000_impl_asm/001_crypto_internal_fips140_sha256.block.abi0/blueprint_go_BenchmarkAddMulVVWW_words_100000_impl_asm_001.json`
- 补丁计划：`.yuansheng/craft/go/001_crypto_internal_fips140_sha256.block.abi0/craft/patch-plan.json`
- 候选补丁：`.yuansheng/craft/go/001_crypto_internal_fips140_sha256.block.abi0/craft/patch-candidate.json`
- 差异文件：`.yuansheng/craft/go/001_crypto_internal_fips140_sha256.block.abi0/craft/patch.diff`
- 变更文件：`src/crypto/internal/fips140/sha256/sha256block_riscv64.s`

## 通用审核

1. 根因解决：蓝图指出 MSGSCHEDULE0 的 BE 消息字用 4×lbu+移位+or 逐字节拼装。补丁新增 hasZbb 门控的 rev8 快路径（MOVWU + REV8 + SRL 三条指令取代 11 条指令的字节拼装），在 GORISCV64>=rva22 时生效。根因已解决。（蓝图另称“无 sha256block_riscv64.s / 旋转用 srliw+slliw+or”与仓库事实不符：该文件已存在且旋转已用 RORW→roriw；补丁聚焦于仍属事实的字节拼装部分。）
2. 约束保持：REV8+SRL 与逐字节拼装结果逐位一致，SHA-256 输出 bit-identical；rva20（无 Zbb）走 #else 原字节拼装，语义/ABI 不变。mustPreserve（数值输出）保持。
3. 范围合理性：1 个文件、11 行插入，聚焦消息调度字节拼装，无无关改动。
4. 产物一致性：PatchCandidate.gitDiff 与 patch.diff 一致，changedFiles 仅 sha256block_riscv64.s。
5. 安全：无硬编码密钥、无危险命令/路径。

## RISC-V 架构审核

archSpecific=true（命中 asm-source-file）。新增 REV8 为 Zbb 指令，经 `#ifdef hasZbb` 构建期门控（asm_riscv64.h 在 GORISCV64>=rva22 定义 hasZbb）；目标 C920v2 支持 Zbb；rva20 无 Zbb 回退原字节拼装。REV8 字节反转 + SRL $32 下移的语义与原 big-endian 拼装逐位一致。无 x86/ARM 指令混入。架构用法正确。

## 审核结果

pass

## 发现问题

无 critical / major。1 条 suggestion（F-001-1）：建议实机跑 crypto/sha256 与 FIPS140 自检测试验证 bit-identical（本次仅交叉编译验证）。

## 幻觉自检

- [PASS] 技术精度：REV8 属 Zbb、门控方式、逐位一致性均锚定 asm_riscv64.h 与 diff hunk 原文。
- [PASS] 声明溯源：finding evidence 引用 patch.diff 新增行。
- [PASS] 可解释性：无 critical/major，无需逐条解释。
- [PASS] 内部一致性：pass 与 findings 无 critical/major 一致。
- [PASS] 安全：无越权建议。

## 结论

补丁准确解决已诊断根因，输出 bit-identical，架构门控正确，产物链一致，审核通过。
