# AI 补丁审核报告

## 审核范围

- **蓝图**：`.yuansheng/trace/mariadb/threads_1/001_mysys_namespace_crc32c_crc32c_slow_unsigned_int_void_const_unsig/blueprint_mariadb_threads_1_001.json`
- **PatchPlan**：`.yuansheng/craft/mariadb/001_mysys_namespace_crc32c_crc32c_slow_unsigned_int_void_const_unsig/craft/patch-plan.json`
- **patch.diff**：`.yuansheng/craft/mariadb/001_mysys_namespace_crc32c_crc32c_slow_unsigned_int_void_const_unsig/craft/patch.diff`
- **PatchCandidate**：`.yuansheng/craft/mariadb/001_mysys_namespace_crc32c_crc32c_slow_unsigned_int_void_const_unsig/craft/patch-candidate.json`
- **变更文件**：`mysys/CMakeLists.txt`、`mysys/crc32/crc32c.cc`、`mysys/crc32/crc32c_riscv.cc`（新增）

审核为独立只读审核，结论仅基于上述落盘事实来源。

## RISC-V 架构审核

arch-scan 判定 `archSpecific: true`（命中 riscv-macro / march-flag / hwprobe 规则），执行架构专项审核：

1. **指令/特性是否在目标硬件 ISA 内**：补丁使用 Zbc `clmul`/`clmulh` 与 Zbb `rev8`/`brev8`（经 `__builtin_riscv_*`）。蓝图 `source.targetHardware` = XuanTie C920v2 / SOPHGO SG2044，metadata cpuinfo isa 字符串含 `zba/zbb/zbc/zbs`，故 Zbc/Zbb 在目标 ISA 内。构建侧仅对新增 TU 施加 `-march=rv64gc_zbc_zbb`（与 aarch64 分支 `-march=armv8-a+crc+crypto` 的既有模式一致），不改变全局编译基线。
2. **vsetvl/vtype 数据流一致性**：本补丁不使用 RVV，无向量状态，NA。
3. **tail/mask policy、标量↔向量边界、寄存器组重叠**：无 RVV，NA。
4. **运行时 ISA 分发自洽性**：`crc32c_riscv_available()` 用 `riscv_hwprobe(RISCV_HWPROBE_KEY_IMA_EXT_0)` 探测 `RISCV_HWPROBE_EXT_ZBC`/`RISCV_HWPROBE_EXT_ZBB`，任一缺失返回 NULL 回退 `crc32c_slow`；hwprobe API 不可用时回退为可用（该 TU 仅在 `-march=rv64gc_zbc_zbb` 下编译）。分发路径自洽且保留了标量 fallback。
5. **未混入 x86/ARM 专属指令**：新增文件仅使用 RISC-V builtins，无跨架构指令。

架构审核结论：**通过**（`archReview.status = passed`）。

## 审核结果

| 检查项 | 结论 |
|--------|------|
| 根因解决 | PASS —— 以 Zbc/Zbb carry-less 折叠替换 slicing-by-4 查表 + 串行 XOR 依赖链，直击 latency-bound 根因 |
| 约束保持（CRC32C 输出逐字节一致） | PASS —— 反射多项式 0x82F63B78、init/final XOR 0xFFFFFFFF、字节序与 tail（0..3 字节）语义与标量参考等价；算法已在仿真中对标量与 KAT 0xE3069283 验证 |
| 约束保持（调用契约） | PASS —— `crc32c_riscv(unsigned, const void*, size_t)` 与 `crc32c_slow` 签名一致，经 `Choose_Extend()` 分发 |
| 范围合理性 | PASS —— 3 个文件、聚焦，新增隔离实现 + 分发 + 构建配置，镜像 aarch64/ppc64 既有模式，无无关重构 |
| 产物一致性 | PASS —— `patch-candidate.json` 的 `gitDiff` 与 `patch.diff` 字节一致，`changedFiles` 与 diff 一致 |
| 安全 | PASS —— 无硬编码密钥、无危险命令/路径 |

## 发现问题

无 critical / major / minor 问题。

suggestion 级（不影响合入正确性，供后续落地参考）：

- **F-001-1（verification）**：算法与常量已在 Python 仿真中对照标量参考与 CRC32C 标准校验值验证，但本环境无 RISC-V 工具链，未在目标硬件编译/测试。落地前应重建并跑 KAT 与 perf annotate。
- **F-001-2（planning）**：自动 PatchPlan 的 `changes[].filePath` 因蓝图缺 debug symbols 回退为占位路径，实际改动以 git diff 为准。

## 幻觉自检

- [PASS] 技术精度：架构结论均有 blueprint/metadata 证据支撑（cpuinfo isa 含 zbc/zbb；targetHardware）。
- [PASS] 声明溯源：每条 finding 的 evidence 均引用 blueprint 具体字段或 patch.diff 内容。
- [PASS] 可解释性：无 critical/major finding，故无"为什么必须改"的强断言需要单独解释。
- [PASS] 内部一致性：reviewResult=pass 与 findings 严重度（均为 suggestion）一致。
- [PASS] 安全：无越权建议、无引入新风险的建议。

## 结论

**审核通过（pass）**。补丁准确解决 blueprint 根因，约束保持完整，范围聚焦，产物一致，架构代码经专项审核通过。仅记录两条 suggestion 级落地建议（目标硬件实测验证 + 占位路径说明），不阻断合入。
