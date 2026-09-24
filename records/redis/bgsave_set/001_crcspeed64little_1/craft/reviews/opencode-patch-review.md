# AI 补丁审核报告

- 审核对象：`pc-bp-redis-bgsave-set-001`（crcspeed64little Zbc 路径，修订轮 1）
- 审核时间：2026-09-23
- 审核模式：独立只读（结论基于落盘事实来源）

## 审核范围

- RootCauseBlueprint：`.yuansheng/trace/redis/bgsave_set/001_crcspeed64little/blueprint_redis_bgsave_set_001.json`
- PatchPlan：`.yuansheng/craft/redis/001_crcspeed64little/craft/patch-plan.json`
- patch.diff：`.yuansheng/craft/redis/001_crcspeed64little/craft/patch.diff`（v2，含修订）
- PatchCandidate：`.yuansheng/craft/redis/001_crcspeed64little/craft/patch-candidate.json`
- 涉及文件：`src/crcspeed.c`、`src/crcspeed.h`

## 通用审核

1. **根因解决（PASS）**：blueprint `rootCause.summary` 指向"硬件原生 clmul/clmulh（Zbc）存在却因 build 缺 zb* 且无派发而从未使用"。v2 补丁新增 `crcspeed64little_zbc`（反射域 16 字节 clmul/clmulh 折叠 + Barrett 归约），并在 `crcspeed64native` 分派点以 riscv_hwprobe 检测 zbc 后选择该例程；与 `recommendedFirstAction` 逐条对应，保留查表回退。
2. **约束保持（PASS）**：
   - KAT 位精确：qemu-riscv64 下 `crcspeed64little_zbc(0,"123456789",9) == e9c6d914c4b8d9ca`、Lorem 向量 `c7794709e69683b3` 均通过；
   - 20 万组随机输入（长度 0–1200、起始偏移 0–64、随机初始 crc）与 2000 组大缓冲区（1.2KB–21KB）对拍生产查表路径 `crcspeed64little` 零失配；
   - 公共 API（`crcspeed64native` 签名、`crc64` 语义）不变；非 zbc 硬件经 hwprobe 门回退查表路径；未做静态 -march 切换。
3. **范围合理性（PASS）**：变更仅限 `src/crcspeed.c` 与 `src/crcspeed.h`（185 行新增），聚焦根因链，无无关重构。
4. **产物一致性（PASS）**：PatchCandidate.gitDiff 与 patch.diff 字节一致；notes 已补充 F-002 相关测试语义披露与验证记录。
5. **安全（PASS）**：无硬编码密钥、无危险命令/路径。

## RISC-V 架构审核

1. **指令/特性在目标 ISA 内（PASS）**：clmul（`.insn r 0x33, 0x1, 0x5`）与 clmulh（`.insn r 0x33, 0x3, 0x5`）为 Zbc 指令；blueprint `source.targetHardware` = SOPHGO SG2044（XuanTie C920v2），metadata cpuinfo ISA 含 `zbc`，Build ISA（redis-server ELF `Tag_RISCV_arch`）无 zb* 与"硬件有、构建无、需运行时派发"的根因一致。qemu-riscv64 实测指令执行正确。指令仅在 hwprobe 确认 ZBC 后可达，非 zbc 核永不执行。
2. **vsetvl/vtype 配置（N/A）**：无向量代码。
3. **tail/mask policy、标量↔向量边界（N/A）**：无向量状态。
4. **运行时 ISA 分发（PASS）**：riscv_hwprobe（key `RISCV_HWPROBE_KEY_IMA_EXT_0=4`、ZBC 位 `1ULL<<7`）与内核 UAPI 一致（对照 sysroot `<asm/hwprobe.h>`）；syscall 失败/不支持时回退查表；`__has_include` 未命中时由回退块补齐常量与 `struct riscv_hwprobe { long key; unsigned long value; }`（rv64 long=64 位），并已用空 `asm/hwprobe.h` 模拟缺头文件场景交叉编译通过；分派处带端序检查 `&& *(char *)&n`（RISC-V 恒为小端，防御性文档化）。修订轮 1 的 F-001 已修复。
5. **无 x86/ARM 专属指令（PASS）**。

## 审核结果

**PASS**

## 发现问题

- 修订轮 1 已修复 F-001（major，编译可移植性）：v2 diff 第 60 行新增回退结构体定义，第 63 行将 `RISCV_HWPROBE_EXT_ZBC` 守卫独立；缺 `<asm/hwprobe.h>` 场景交叉编译通过。
- F-002（suggestion，测试语义）：已在 PatchCandidate notes 中披露，无需代码改动。
- F-003（suggestion，binutils 下限）：目标工具链 GCC 14.2.0 满足，仅记录。
- 本轮无新增 critical/major/minor finding。

## 幻觉自检

- **技术精度** [PASS]：架构结论（zbc 指令、hwprobe 常量、目标 ISA）均有 blueprint/diaglog 或 sysroot 内核头文件证据；qemu 实测为补充验证。
- **声明溯源** [PASS]：所有结论/发现均引用 patch.diff hunk 原文或 blueprint 字段。
- **可解释性** [PASS]：PASS 结论对应无 critical/major finding，理由充分。
- **内部一致性** [PASS]：reviewResult=pass 与 findings（仅 suggestion）严重度一致。
- **安全** [PASS]：无越权建议、无引入新风险的建议。

## 结论

修订后补丁通过全部审核项：算法位精确（KAT 与 20 万+随机对拍零失配）、运行时 Zbc 分派自洽、缺头文件工具链可编译、约束与公共 API 保持。同意进入 done 终态。
