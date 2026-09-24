# AI 补丁审核报告

## 审核范围

- PatchCandidate: `pc-bp-mnn-benchmark-mobilenetv2-224-mnn-1-004`
- Blueprint: `bp-mnn-benchmark_MobileNetV2_224.mnn_1-004`（根因：输入 padding 的 memcpy 被自动向量化为 e8,m1，LMUL 过小导致循环控制/配置开销占比过高）
- 变更文件：`source/backend/cpu/CPUConvolutionDepthwise.cpp`
- 变更内容：新增 `_memcpyBytesRVV`（e8,m8 显式 RVV 字节拷贝），并将 `BasicFloatExecution::onResize` lambda 中第 230 行的 `::memcpy` 在 `MNN_USE_RVV` 下替换为该 helper

## RISC-V 架构审核

（arch-scan 判定 `archSpecific: true`，执行架构专项审核）

- 指令合法性：`__riscv_vsetvl_e8m8` / `__riscv_vle8_v_u8m8` / `__riscv_vse8_v_u8m8` 均为 RVV 1.0 指令，SpacemiT K3（RVV 1.0，VLEN=256）支持；与 blueprint/diaglog 的 Build ISA 证据（`v1p0` + zve*）一致。
- vsetvl/vtype 一致性：`vsetvl_e8m8` 与 `vle8/vse8` 的 SEW/LMUL（e8,m8）一致，无状态错乱。
- tail/mask policy：默认 `ta,ma`，无 mask，`vsetvl` 按剩余字节数返回 vl，处理 tail 正确。
- 标量↔向量边界：`uint8_t*` 逐字节 raw copy，无算术，字节序/对齐无关（unit-stride byte access 不要求对齐）。
- 无 x86/ARM 专属指令混入：RVV 代码由 `#ifdef MNN_USE_RVV` 守卫，非 RVV 构建回退 `::memcpy`。

## 审核结果

- 根因解决：是。以 e8,m8（VLEN=256 下每轮 256 字节）替代编译器默认 e8,m1（每轮 32 字节），直接降低循环控制/`vsetvli`/回边开销。
- 约束保持：是。逐字节一致（raw byte copy 无数值变化）；`for (size_t vl; n > 0; ...)` 天然处理 n=0（不进入循环）；unit-stride byte access 无对齐要求。
- 最小性：是。新增一个静态 helper + 单点替换，无无关重构。
- 产物一致性：是。PatchCandidate.gitDiff 与 patch.diff 字节一致，changedFiles 准确。
- 安全：是。无硬编码密钥、无危险命令/路径。

## 发现问题

无（findings 为空）。

## 幻觉自检

- [PASS] 技术精度：架构结论（e8,m8 在 K3 RVV 1.0 内）有 blueprint/diaglog Build ISA 证据支撑。
- [PASS] 声明溯源：无 critical/major finding，无需引用 diff hunk 原文。
- [PASS] 可解释性：无 critical/major finding。
- [PASS] 内部一致性：reviewResult=pass 与 findings 为空一致。
- [PASS] 安全：无越权建议、无引入新风险。

## 结论

审核通过（reviewResult: pass）。补丁最小、正确、架构安全，准确解决 blueprint 指出的「memcpy e8,m1 → 循环控制开销高」根因。
