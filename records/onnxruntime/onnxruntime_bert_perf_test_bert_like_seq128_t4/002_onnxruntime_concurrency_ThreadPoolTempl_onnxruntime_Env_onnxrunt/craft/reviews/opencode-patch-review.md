# AI 补丁审核报告

## 审核范围

- **blueprint**: `.yuansheng/trace/onnxruntime/onnxruntime_bert_perf_test_bert_like_seq128_t4/002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt/blueprint_onnxruntime_onnxruntime_bert_perf_test_bert_like_seq128_t4_002.json`
- **patch-plan**: `.yuansheng/craft/onnxruntime/002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt/craft/patch-plan.json`
- **patch.diff**: `.yuansheng/craft/onnxruntime/002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt/craft/patch.diff`
- **patch-candidate**: `.yuansheng/craft/onnxruntime/002_onnxruntime_concurrency_ThreadPoolTempl_onnxruntime_Env_onnxrunt/craft/patch-candidate.json`
- 目标硬件：SpacemiT X100 (K3)，RVV 1.0 VLEN=256；热点函数 `ThreadPoolTempl<...>::WorkerLoop(int)`，rank 002

## 根因摘要（来自 blueprint）

WorkerLoop 的 idle 自旋/等待机制在 RISC-V 上以全速运行：`SpinPause()` 缺失架构 spin hint（Zihintpause），退避循环每轮只执行一次编译器屏障（`std::atomic_signal_fence(seq_cst)`），无法降低轮询密度与资源争用。blueprint evidence：`spin_pause.cc:30-63：SpinPause() 对 RISC-V 无专用分支，落入 #else 的 std::atomic_signal_fence(std::memory_order_seq_cst) 纯编译器屏障`；`hardware cpuinfo 含 zihintpause`。recommendedFirstAction：在 `#if defined(__riscv_zihintpause)` 下发射架构 hint（`__asm__ __volatile__("pause" ::: "memory")`），否则保留 fallback。

## RISC-V 架构审核

- `arch-scan` 判定 `archSpecific: true`（命中 `inline-asm` 与 `riscv-macro` 规则），执行架构专项审核。
- 补丁新增分支 `#elif defined(__riscv) && defined(__riscv_zihintpause)` → `__asm__ __volatile__("pause" ::: "memory")`：
  1. **指令/特性在目标硬件 ISA 内**：blueprint evidence 明确 `hardware cpuinfo 含 zihintpause`（SpacemiT X100 支持 Zihintpause 扩展），`pause` 是 Zihintpause 标准 hint 助记符。✅
  2. **feature-gate 正确**：`__riscv_zihintpause` 由编译器在 -march 含 zihintpause 时定义；当前 build Tag_RISCV_arch 不含 zihintpause 时宏未定义、自动落入原有 `#else` fallback，行为与基线完全一致——构建侧需增补 zihintpause 后才启用，与 blueprint 建议一致。✅
  3. **无 vsetvl/vtype/tail/mask 状态**：补丁不涉及向量指令。✅
  4. **语义保持**：`pause` 是 hint，`"memory"` clobber 仅防优化器移除，不改变内存序，不替代 fence/atomic，满足 `constraints.mustPreserve` 第 1 条。✅
  5. **未混入 x86/ARM 专属指令**：仅新增 RISC-V 分支，ARM/x86 分支未动。✅

## 通用审核清单

1. **根因解决**：补丁在 `SpinPause()` 中为 RISC-V 增加 Zihintpause `pause` 发射路径，直接解决"RISC-V 无任何 pause 发射路径"根因，与 recommendedFirstAction 逐字一致。✅
2. **约束保持**：SpinPause 语义/公共 API/线程池同步协议均未改变；`EigenNonBlockingThreadPool.h` 无需改动（热点为 WorkerLoop 内调用 SpinPause 的等待路径，SpinPause 修复即覆盖）。✅
3. **最小性**：仅新增一个 `#elif` 分支（6 行），无无关重构与格式噪音。✅
4. **产物一致性**：PatchCandidate.gitDiff 与 patch.diff 字节一致（913 bytes）。✅
5. **安全**：无硬编码密钥、无危险命令/路径。✅

## 审核结果

**reviewResult: pass**，findings 为空。

## 发现问题

无。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度 | [PASS] 架构结论均引用 blueprint evidence（cpuinfo 含 zihintpause、spin_pause.cc:30-63 落入 fallback） |
| 声明溯源 | [PASS] 无 findings，全部结论来自 diff/blueprint 原文 |
| 可解释性 | [PASS] 无 critical/major 需要解释 |
| 内部一致性 | [PASS] reviewResult=pass 与 findings 空一致 |
| 安全 | [PASS] 未引入越权建议或新风险 |

## 结论

补丁准确、最小、架构正确，通过全部审核维度，流转至终态 done。
