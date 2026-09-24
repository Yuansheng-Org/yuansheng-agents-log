# AI 补丁审核报告

- 审核对象：`pc-bp-redis-bgsave-set-018`（siphash Zbb rol 化 + 运行时分派）
- 审核时间：2026-09-23
- 审核模式：独立只读（结论基于落盘事实来源）

## 审核范围

- RootCauseBlueprint：`.yuansheng/trace/redis/bgsave_set/018_siphash/blueprint_redis_bgsave_set_018.json`
- PatchPlan：`.yuansheng/craft/redis/018_siphash/craft/patch-plan.json`
- patch.diff：`.yuansheng/craft/redis/018_siphash/craft/patch.diff`
- PatchCandidate：`.yuansheng/craft/redis/018_siphash/craft/patch-candidate.json`
- 涉及文件：`src/siphash.c`（仅此一个）

## 通用审核

1. **根因解决（PASS）**：blueprint `rootCause.summary` 指向"build ISA 未启用硬件已支持的 Zbb，SIPROUND 热循环中每个 ROTL 退化为 slli+srli+add 三指令合成（主循环约 46% 指令）"。补丁将 SipHash 核心模板化为 base/Zbb 两个实例，运行时经 riscv_hwprobe 检测 Zbb 后选择 Zbb 实例（`rol` 单指令旋转）；RISC-V 反汇编确认 zbb 变体每轮 6 条 `rol`（旋转常量提升到循环外），三指令合成序列消失。采用运行时分派（非 build -march 修改），对现有不含 zbb 的构建直接生效且保持二进制可移植，与 blueprint `recommendedFirstAction` 的"如需多目标可移植，可用 __riscv_zbb guard 内联 rol asm"精神一致且覆盖更广。
2. **约束保持（PASS）**：
   - `mustPreserve[0]` SipHash 输出 bit-identical：qemu-riscv64 下 20 万组随机输入/密钥对拍 `siphash_base` vs `siphash_zbb` 零失配（含 `siphash_nocase_*` 变体）；5 万组公共分派一致性（`siphash()` 结果 ∈ {base, zbb} 且等于对应变体）；
   - `mustPreserve[1]` 公共 API 与原型不变（`siphash`/`siphash_nocase` 签名未动）；
   - `mustPreserve[2]` 测试向量：大小写折叠三条交叉检查与空输入检查通过；in-tree `siphash_test` 向量为 2-4 轮约定（源码注释说明需先回退 1-2 轮），属既有测试前提，本补丁未改变轮数与输出。
3. **范围合理性（PASS）**：仅改 `src/siphash.c`（模板化重构 + 分派），聚焦根因链；未动 Makefile（避免覆盖平台 V 扩展 march）。
4. **产物一致性（PASS）**：PatchCandidate.gitDiff 与 patch.diff 字节一致；changedFiles=[src/siphash.c]。
5. **安全（PASS）**：无硬编码密钥、无危险命令/路径。

## RISC-V 架构审核

1. **指令/特性在目标 ISA 内（PASS）**：`rol`（`.insn r 0x33, 0x1, 0x30`）为 Zbb 指令；blueprint `source.targetHardware` = SG2044/C920v2，metadata cpuinfo ISA 含 `zbb`；qemu-riscv64 实测指令执行正确。指令仅在 hwprobe 确认 ZBB 后可达，非 Zbb 核走基线路径永不执行。
2. **vsetvl/vtype 配置（N/A）**：无向量代码。
3. **tail/mask policy、标量↔向量边界（N/A）**：无向量状态。
4. **运行时 ISA 分发（PASS）**：riscv_hwprobe（key `RISCV_HWPROBE_KEY_IMA_EXT_0=4`，ZBB 位 `1ULL<<4`）与内核 UAPI 一致（对照 sysroot `<asm/hwprobe.h>`）；syscall 失败/不支持时回退基线路径；缺 `<asm/hwprobe.h>` 时回退块补齐常量与 `struct riscv_hwprobe`；分派缓存使用 `__atomic` 一次初始化，每调用仅两次 relaxed load。
5. **无 x86/ARM 专属指令（PASS）**。
6. **严格 -std 编译（PASS）**：文件顶部 `#define _DEFAULT_SOURCE`，`-std=c11 -pedantic` 下 syscall 声明无警告（RV 与 host 均验证）。

## 审核结果

**PASS**

## 发现问题

- 无 critical/major/minor finding。
- 记录（透明性）：Trace 蓝图 #018 的 `validation.regressionCommand` 为空串导致 craft schema 校验失败，已做最小数据修复（填入与 #001/#003 一致的标准命令 `agent1 patch_regression --case bgsave_set`），不改动其余字段。
- 记录（suggestion 级）：模板化产生 4 个静态变体（base/zbb × 大小写），维护上两处核心体通过宏参数复用；`rol` 寄存器形式依赖 GCC 常量提升（反汇编已验证）；`src/crcspeed.c`（rank-001 已交付）的同类 hwprobe 段在默认 gnu dialect 下无警告，严格 `-std=c11` 下同样需要 `_DEFAULT_SOURCE`，已记录为跨函数观察。

## 幻觉自检

- **技术精度** [PASS]：Zbb/rol 结论来自 blueprint/diaglog（build ISA 无 zbb、硬件含 zbb、ROTL 三指令序列证据）与 qemu 实测反汇编。
- **声明溯源** [PASS]：结论/记录均引用 patch.diff hunk 或 blueprint 字段。
- **可解释性** [PASS]：PASS 结论对应无 critical/major finding，理由充分。
- **内部一致性** [PASS]：reviewResult=pass 与 findings 严重度一致。
- **安全** [PASS]：无越权建议。

## 结论

补丁位精确实现 Zbb rol 化（20 万+随机对拍零失配），运行时分派自洽、跨平台可移植、公共 API 与哈希输出保持，架构专项审核通过。同意进入 done 终态。
