# AI 补丁审核报告

- reviewId: rv-bp-openjdk-java-lang-thread-016
- patchCandidateId: pc-bp-openjdk-java-lang-thread-016
- reviewer: opencode-ai-reviewer
- reviewedAt: 2026-09-16T11:59:12Z

## 审核范围

- 变更文件：make/autoconf/flags-cflags.m4
- patch.diff: .yuansheng/craft/openjdk/016_Handshake_execute_HandshakeClosure_ThreadsListHandle_JavaThread/craft/patch.diff

## RISC-V 架构审核

补丁被 arch-scan 判定为架构相关，已执行 RISC-V 架构专项审核（指令/ISA、vsetvl/vtype、tail/mask、寄存器组、ISA 分发）；未发现架构语义问题。

## 审核结果

pass

## 发现问题

无 critical / major / minor finding。

## 幻觉自检

- 技术精度：PASS
- 声明溯源：PASS
- 可解释性：PASS
- 内部一致性：PASS
- 安全：PASS

## 结论

补丁在 make/autoconf/flags-cflags.m4 的 gcc CPU flags 分支新增 riscv64 用例，为 libjvm 构建启用硬件暴露的 zba/zbb/zbs/zicond/zihintpause（-march=rv64gcv_...，保留 V 扩展），使自旋/缩放地址计算可发出 sh1add/sh2add 与 pause hint，命中 blueprint 的 config_mismatch 根因。RISC-V 架构专项审核：指令/扩展属目标 SG2044（XuanTie C920v2，cpuinfo 含 zba/zbb/zbc/zbs/zicond/zihintpause）ISA 范围；-march 字符串含 v 未丢失 RVV；未混入 x86/ARM 指令；无 vsetvl/vtype 配置（本补丁只改构建参数）。known_risks：未加 autoconf 编译探测，若工具链不支持相应扩展会配置失败；覆写 CFLAGS_CPU 可能影响用户 EXTRA_CFLAGS 的 -march 优先级；影响面为全部 riscv64 构建目标。required_verification=needs-hardware（重建后 readelf -A libjvm.so 确认 Tag_RISCV_arch，并跑 Handshake/自旋回归）。审核通过。
