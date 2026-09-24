# AI 补丁审核报告

- reviewId: `pr-bp-vllm-vllm-bench-throughput-002-r1`
- patchCandidateId: `pc-bp-vllm-vllm-bench-throughput-002`
- patchPlanId: `pp-bp-vllm-vllm-bench-throughput-002`
- root_cause_blueprint: `.yuansheng/trace/vllm/vllm-bench-throughput/002_cpu_attention_AttentionMainLoop_cpu_attention_AttentionImpl_cpu_/blueprint_vllm_vllm-bench-throughput_002.json`
- 审核对象: `.yuansheng/craft/vllm/002_cpu_attention_AttentionMainLoop_cpu_attention_AttentionImpl_cpu_/craft/patch.diff`
- 审核模式: 独立只读审核（仅基于落盘事实来源：blueprint、diaglog、patch-plan、patch.diff、patch-candidate）

## 审核范围

- 变更文件（3）：`csrc/cpu/cpu_attn_rvv.hpp`、`csrc/cpu/cpu_attn_impl.hpp`、`csrc/cpu/cpu_arch_macros.h`；`git diff --stat` = 192 insertions / 14 deletions。
- 产物一致性：`patch-candidate.json.gitDiff` 与 `craft/patch.diff` 字节一致（脚本比对为 true）；`changedFiles` = 上述 3 个文件，与 diff 一致；`archReviewWarning` 为空字符串，与 `arch-scan` 判定 `archSpecific: true` 一致，故执行 RISC-V 架构专项审核。
- 根因对照：blueprint `rootCause.summary` 指向「FP32 attention GEMM microkernel 的 A-slice 标量 gather 载入产生 cache 不友好的跨行逐标量访问，flw→vfmacc.vf 紧邻 load-use 依赖在 memory-bound 工作负载下主导周期；reduce_splits 无退避轮询为次要独立因素」。diaglog Phase 3/4 的锚定热行为 `254340: flw fa2,0(t1)`(10.74%)、`254356: vfmacc.vf v8,fa2,v14`(7.66%)、`25437c`(8.67%)、`2543a2`(10.30%) 与 `254b6a: flw fa2,0(t6)`(10.54%)、`254b82`(7.28%)、`254baa`(9.06%)、`254bd2`(10.44%)。
- 补丁对应关系：
  - 主要修复：新增 `gemm_micro_rvv_fma_Mx16_Ku4<M<=4, kv_cache_t>`（16 列 tile），并在 `gemm_macro_rvv_fma_Mx8_Ku4` 中对 `mb<=4` 的 M tile 以 16 列步进调用。逐输出元素 FP32 累加顺序仍为 k 递增（未做 reassociation）。
  - 次要修复：`reduce_splits` 的 `flags[split_idx]` 轮询与 guard_counter 等待改为 `bounded_spin_wait` 有界递增退避 + `CPU_ATTN_SPIN_PAUSE()`。

## RISC-V 架构审核

判定依据仅引用 blueprint / diaglog / patch.diff 中已有证据；无证据处标注「无法验证」并降级。

1. 指令与扩展在目标 ISA 内（已核实）
   - 新增代码只使用 RVV 1.0 浮点/载入指令路径：`vfmacc_vf`、`vle32_v`、`vse32_v`、`vfmv_v_f`（经 `RVVI(...)` 宏映射），与 blueprint `source.targetHardware`（RVV 1.0, VLEN=128）和 diaglog Phase 1 Build ISA（`v1p0` + `zvl128b`，含 `zve32f/zve64f`）一致。
   - LMUL 映射一致：`LMUL_256` 在 VLEN=128 下为 `m2`（`csrc/cpu/cpu_types_riscv_defs.hpp` 的 `#if __riscv_v_min_vlen == 128` 分支），diaglog Phase 1 记录 `vlenb=16` → VLEN=128；blueprint 与 diaglog 的 VLEN 证据自洽。
   - 新增 inline asm 仅发射 `pause`：`patch.diff` 中 `+    #define CPU_ATTN_SPIN_PAUSE() __asm__ __volatile__(".word 0x0100000f")`。diaglog Phase 1/4 记录硬件 cpuinfo 含 `zihintpause`，故目标机支持该 hint。
2. `vsetvl` / `vtype` 与数据流一致（已核实）
   - 新 kernel 全程只使用 `LMUL_256`（同一 vtype 的 `e32`），`vl` 固定为 8（8 个 FP32 lane），与 16 列 = 2 组 8 列的数据流一致；未出现跨 LMUL 混用。
   - 对 VLEN=128 与 VLEN=256 两个映射做了独立交叉编译，均通过；VLEN=128 反汇编显示 `vsetivli zero, 8, e32, m2` 位于循环外，循环体内 `vl2re32.v`(B 行 2 次整寄存器载入) 与 8 条 `vfmacc.vf` 使用同一 vtype。
3. tail / mask policy、标量↔向量边界、寄存器组重叠（已核实）
   - 所有载入/存储/FMA 的 `vl` 恒为 8，B 行载入 `load_row8_B_as_f32` 精确载入 8 个元素，16 列 tile 恰好等于 2×8，无未初始化 lane 参与累加，tail 无特殊策略需求。
   - A-slice 仍为标量 `flw`（`*(a##i + k + KOFF)`），经 `vfmacc.vf` 广播；标量→向量边界与 Mx8 kernel 相同，未新增 lane 索引依赖。
   - 累加器使用固定宽度向量类型 `fixed_fp32x8_t`（`riscv_rvv_vector_bits(256)`），编译器保证寄存器组不重叠；M=4 时 8 个累加器（m2，16 寄存器）+ 2 个 B 向量（4 寄存器）在 32 个向量寄存器预算内。
4. 运行时 ISA 分发路径（已核实）
   - `TileGemmRVV` 仍只由 `AttentionImpl<ISA::RVV>` 实例化，位于 `#if __riscv_v_min_vlen == 128 || 256` 守卫内；补丁未改动 `cpu_attn.cpp` 的 dispatch，也未引入 ifunc/hwprobe 新路径。
5. 未混入 x86/ARM 专属指令（已核实）
   - 唯一新增 asm 为 RISC-V `pause`，且由 `#elif defined(__riscv) || defined(__riscv__)` 守卫；`CPU_ATTN_SPIN_PAUSE()` 在其它架构回落到 `FAST_SPINNING` 或空操作。
6. 待检查风险（不影响本次结论）
   - 目标机实测未执行（无 vLLM 构建树），`pause` 的实际收益与 `bounded_spin_wait` 阈值只能在目标 SG2044 上复验。

结论：RISC-V 架构专项审核通过。

## 审核结果

`reviewResult: pass`

- 无 critical / major finding；4 条 minor/suggestion 均为覆盖范围与验证条件披露，不阻断。
- 通用审核清单逐项：
  1. 根因解决：是。16 列 tile 使每个 A 标量载入摊薄到 2 次 FMA、独立累加器链由 M 增至 2*M、B 行由 32B 扩为 64B（整 cache line），直接对应 blueprint 的 `recommendedFirstAction`（A-slice 载入形态 + K-slice 驻留/预取方向）。
  2. 约束保持：是。`mustPreserve` 逐项静态核对——paged KV block 布局与 `k/v_cache_*_stride` 合同未改（macro kernel 的 `Bn=B+n`、`Cn=Cb+n`、`ldb/ldc` 语义不变，宽 tile 读取 `B[k*ldb + n .. n+15]` 仍在本行范围内）；causal/sliding-window/softcap/alibi 掩码与 split-reduction flag/fence 协议未触碰（仍为 volatile 读 + acquire fence / release fence + store）；FP32 累加顺序未变；公共 attention API 与 OpenMP 调度未改。
  3. 范围合理性：是。仅 3 个文件、聚焦根因链（RVV kernel + 自旋退避），无无关重构、无格式噪音；`git diff --check` 通过，新增行全部 ≤80 列。
  4. 产物一致性：是（见「审核范围」）。
  5. 安全：无硬编码密钥、无危险命令/路径、无外部输入处理变更。
- 独立数值/静态验证证据（由候选补丁附注披露，审核侧复核其可复现路径）：VLEN=128 与 VLEN=256 交叉编译通过；qemu-riscv64 下对 M∈{1,2,3,4,8,12}、K∈{64,63,5} 组合，新 Mx16 路径与旧 Mx8 调用逐位相同且与标量参考一致。

## 发现问题

| id | severity | category | file:line | 摘要 |
|---|---|---|---|---|
| F1 | minor | coverage | csrc/cpu/cpu_attn_rvv.hpp:194 | 16 列 tile 仅覆盖 `mb<=4`；M=8 tile 仍走 Mx8 kernel |
| F2 | minor | behavior-change | csrc/cpu/cpu_attn_impl.hpp:1528 | `bounded_spin_wait` 对非 RISC-V 路径也生效（x86 原先不自旋让出） |
| F3 | suggestion | build-flags | csrc/cpu/cpu_arch_macros.h:172 | `pause` 经 inline asm 发射，未在构建期声明 `zihintpause` |
| F4 | minor | verification | csrc/cpu/cpu_attn_rvv.hpp:182 | 未在目标机执行 vllm-bench-throughput 回归与 perf 复采 |

详情：

- **F1**（minor / coverage）：evidence = `patch.diff` 新增行 `+      if (mb <= 4) {` 与 `+        for (int32_t n = 0; n < N; n += 16) {`，以及注释 `+//   M=8 would need 32 accumulator registers, so it keeps using the Mx8 tile.`。suggestion：若目标负载存在 M=8 tile（例如 `q_tile_head_num` 为 8/16 的批），可另行评估按 N 分块或 m1 累加器的 M8x16 变体；当前实现不改动 M=8 路径，属显式设计取舍，不阻断。
- **F2**（minor / behavior-change）：evidence = `patch.diff` 新增 `+template <typename Poll>` / `+FORCE_INLINE void bounded_spin_wait(Poll&& done) {` 中 `+      std::this_thread::yield();` 分支，以及 `cpu_arch_macros.h` 中 `+  #ifdef FAST_SPINNING` / `+    #define CPU_ATTN_SPIN_PAUSE() FAST_SPINNING`。x86 原先为 `_mm_pause()` 无限自旋（不 yield），新实现退避饱和后会 yield。suggestion：有界让出优于无限自旋，建议保留；若需严格保持 x86 行为，可将 `std::this_thread::yield()` 分支限定为 RISC-V。同步协议与退出条件未变。
- **F3**（suggestion / build-flags）：evidence = `patch.diff` 新增行 `+    #define CPU_ATTN_SPIN_PAUSE() __asm__ __volatile__(".word 0x0100000f")`，注释说明 Zihintpause 的 `pause` 编码为 `fence w, 0` 的 HINT、未实现该扩展的核按 no-op 执行。suggestion：可选地在目标平台构建参数中声明 `zihintpause`，使 `FAST_SPINNING` 走 `__riscv_pause()` 并让工具链显式知晓该扩展；不声明时当前实现亦安全（汇编器已验证 0x0100000f 即 `pause`）。
- **F4**（minor / verification）：evidence = blueprint 字段 `diagnosis.recommendedVerification`（"重新运行 Agent1 vllm-bench-throughput 回归并重跑 perf annotate…确认 254340/254b6a 等 flw 行与 254356/254b82 等 vfmacc.vf 行样本份额下降…perf stat 观察 L1_dcache_load_miss_rate（基线 22.204%）与 LLC_load_miss_rate（基线 39.681%）下降"）。suggestion：本环境无 vLLM 构建树，未执行该回归；需在目标 SG2044 上复验，并对比 decode-only 与 prefill 批不退化。

## 幻觉自检

- [PASS] 技术精度：架构结论（RVV 1.0 指令路径、LMUL_256=m2、VLEN=128、`pause`=0x0100000f、硬件含 zihintpause）全部有 blueprint/diaglog/patch.diff 或本地汇编器验证支撑；无凭模型知识断言的支持状态。
- [PASS] 声明溯源：每条 finding 的 evidence 均为 patch.diff hunk 原文或 blueprint 具体字段（F1/F2/F3 引用新增行，F4 引用 `diagnosis.recommendedVerification`）；无编造锚点。
- [PASS] 可解释性：每条 finding 说明了"为什么需要关注"及建议动作；无 critical/major，故不涉及必须修改项。
- [PASS] 内部一致性：`reviewResult: pass` 且 findings 严重度集合为 {minor, suggestion}，无 critical/major，结论与严重度一致。
- [PASS] 安全：无越权建议、无引入新风险的建议；补丁未引入密钥、危险路径或输入处理变更。

## 结论

补丁准确对应 blueprint 的主要根因（A-slice 标量载入形态 + 累加器链/ cache line 利用）与次要根因（reduce_splits 无退避轮询），未破坏 `constraints.mustPreserve` 各项合同，范围聚焦、产物一致、无安全问题；RISC-V 架构专项审核通过。4 条 finding 均为 minor/suggestion 级别的覆盖范围与验证条件披露，不阻断。

**审核通过（reviewResult: pass）**，流转 `review-pass` → `phase: done`。
