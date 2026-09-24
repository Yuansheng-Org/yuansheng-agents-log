Functions under analysis: [CPUConvolutionDepthwise::BasicFloatExecution::onResize lambda]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`std::_Function_handler<...>::_M_invoke`，34 samples，cpu-clock，local-period，含 hot loop）
- perf stat：已提供（testcase 共享：IPC=0.83，L1 miss 1.04%，branch miss 2.72%）
- source context：已提供（`CPUConvolutionDepthwise::BasicFloatExecution::onResize` 的 per-thread lambda，源码第 216–237 行：`memset` 输入 padding + `memcpy` 每行 + `kernelFunc` 调用）
- build ISA：rv64…v1p0…zvl128b…zve32f/64d/64f/64x；hardware：RVV 1.0 VLEN=256；vlenb=32
- 采样元数据：cpu-clock，local-period，本函数，单次运行

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0（`v`），VLEN=256，out-of-order |
| Build ISA | rv64…v1p0 + zve32f/64d/64f/64x + zvl128b |
| Vector flavor | RVV 1.0 `v*`（`vle8.v`/`vse8.v`），无 `th.v*` |
| VLEN | 256 bit（vlenb=32） |
| Bound type | **loop-control-bound**（本函数 hot loop：`sub a3,a3,a1` 47.06% + `add a5,a5,a1` 29.41% vs `vle8` 0% + `vse8` 5.88%，回边/计数主导） |
| Sampling semantics | cpu-clock，local-period，无 workload 级贡献 |
| Sampling IP precision | `baseline_gap: sampling IP precision` |

L0 gate：无 mismatch。Bound-type gate：loop-control-bound → 属 LMUL/配置开销主导。

## Phase 2 — Scope / 分析边界

hot loop = `::memcpy(dst, src, src_width*unit*bytes)` 的向量化字节拷贝（0x111f5a–0x111f6c，源码第 230 行）。trace anchor：`47.06 : 111f62: sub a3,a3,a1`（剩余长度减量，skid/控制开销）。区间：`vsetvli e8,m1` + `vle8.v` + `vse8.v` + 计数/指针更新 + 回边。Sampling IP precision 不足 → interval 级归因。

## Phase 3 — Pattern scan / 模式扫描：CPUConvolutionDepthwise::BasicFloatExecution::onResize lambda

### Class selection trace（8 项）

1. `rows-asm.md` — exclude：非 `.S`。
2. `rows-operator-rvv.md` — exclude：非算子语义（是 memcpy 数据搬移）。
3. `rows-string-memory.md` — include：copy/fill 语义（memcpy 输入 padding）。
4. `rows-vectorized-tuning.md` — include：已 RVV 化 byte loop（e8,m1），评估最大 LMUL row。
5. `rows-codegen.md` — exclude：非 dispatch/sync/codegen 形态。
6. `rows-offload.md` — exclude：非矩阵引擎/repack。
7. `rows-crypto.md` — exclude。
8. `rows-runtime-os.md` — exclude。

`Classes scanned:` `rows-string-memory.md`、`rows-vectorized-tuning.md`

### Local performance pattern scan

| Pattern | Evidence | Route | Impact | Detail file |
|---|---|---|---|---|
| Maximal LMUL for Misaligned Byte Processing（**primary**） | 已向量化 byte loop（`vsetvli e8,m1`+`vle8.v`+`vse8.v`），但 loop-control 主导（sub 47.06%+add 29.41%），m1 每轮仅 vlenb=32 字节 | High | Medium | `patterns/maximal_lmul_for_misaligned_byte_processing.md` |

### Finding 1（primary）

**(a) 逐字 evidence 引用**：
```
 2.94 :   111f5a: vsetvli a1,a3,e8,m1,ta,ma   # 每轮 e8,m1 → VLMAX=32 字节
 0.00 :   111f5e: vle8.v  v1,(a4)             # 字节 load
47.06 :   111f62: sub     a3,a3,a1             # 剩余长度减量（主导，skid+控制）
 5.88 :   111f66: vse8.v  v1,(a5)             # 字节 store
29.41 :   111f6a: add     a5,a5,a1             # dst 指针步进（主导）
 0.00 :   111f6c: bnez    a3,111f5a           # 回边
```
源码第 230 行：`::memcpy(dst, src, src_width * unit * bytes);`——输入 padding 的逐行字节拷贝，编译器自动向量化为 e8,m1。

**(b) 互斥邻居排除**：
- `rvv_memory_copy_fill_implementation.md`：其 signal 是「scalar 逐元素 copy 未向量化」→ 本循环已 `vle8.v`/`vse8.v` 向量化，排除（问题不是缺向量化，是 LMUL 过小）。
- `scalar_remainder_handling_for_wide_kernels.md`：非 scalar tail 主导，是 main loop 本身 m1 → 排除。
- `overlap_expanding_backreference_copy.md`：无 self-expanding 依赖 → 排除。

**(c) 双 Confidence 推导式**：
- `route: 已向量化 byte loop（vle8/vse8 e8,m1）+ loop-control 主导（sub/add/bnez vs load/store）→ High`
- `impact: VLEN 已知(256)、bound 明确；但样本数少(34)、memcpy 是 depthwise 的次要组件 → Medium`

## Phase 4 — Root-cause blueprint / 根因蓝图

### Finding 1（primary）

**1. Root cause**：输入 padding 的 `memcpy` 被编译器向量化为 `e8,m1` 字节拷贝，每轮仅覆盖 `vlenb`=32 字节（VLEN=256），使 `vsetvli`/指针更新/回边（sub 47.06%+add 29.41%）主导，实际 load/store（vle8 0%+vse8 5.88%）占比低。依据 `patterns/maximal_lmul_for_misaligned_byte_processing.md` §"Why this is slow"：「LMUL 过小 → 每轮覆盖量低，loop-control 与配置成本占比过高」。

**2. The fix**：将 e8 的 LMUL 从 m1 提升到 live-register 预算允许的最大值（m4/m8，VLEN=256 下每轮 128/256 字节），减少迭代次数与回边开销。修复前/后：
```asm
; Before: e8,m1（每轮 32 字节）
vsetvli a1,a3,e8,m1,ta,ma ; vle8.v v1,(a4) ; vse8.v v1,(a5) ; sub/add/bnez

; After: e8,m8（每轮 256 字节，VLEN=256）
vsetvli a1,a3,e8,m8,ta,ma ; vle8.v v8,(a4) ; vse8.v v8,(a5) ; sub/add/bnez
```
适用前提：byte copy 连续、无 overlap/EMUL/spill（live vector 仅 1 个，m8 合法）；短输入保留 scalar fast path。限制/风险：m8 需寄存器预算允许（此处仅 1 个 live vector，安全）；未对齐访问延迟需实测。预期 Profile 信号：`sub`/`add`/`bnez` 迭代开销份额下降，`vle8.v`/`vse8.v` 份额上升（每轮覆盖量 ×8）。

**3. Baseline facts**：hardware RVV 1.0；build v1p0+zvl128b；VLEN=256；bound=loop-control-bound。

**4. 收益上界**：局部样本份额（loop-control 约 76%）；不称 Amdahl。

**5. 三维路由**：current source=compiler 自动向量化 memcpy（非手写）；existence=RVV 路径已采用；policy=无。

**6. Implementation-shape**：不适用。

**7. Related PRs**：`patterns/maximal_lmul_for_misaligned_byte_processing.md` → `Related PRs：4 条 URL`（Go f532f87a9895、d83b16fcb8de、3406a617d964、75ea2d05c019）。

## Phase 5 — Verification forecast / 验证预测

- 消失侧：`111f62: sub a3,a3,a1`（47.06%）、`111f6a: add a5,a5,a1`（29.41%）迭代开销份额下降。
- 出现侧（pattern §Verification）：`vsetvli` 使用 e8,m4/m8，回边/指针更新次数下降；反汇编确认无新增 spill。
- 正确性合同：memcpy 语义（逐字节一致）、零长度/短输入回退、未对齐地址、page/cache-line 边界。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现 | ✅（1/1 组） |
| 2 | Phase 1 输出要求满足 | ✅（7 行 + 2 L0 gate + bound-type gate；gap：sampling IP precision） |
| 3 | Phase 3 输出要求满足 | ✅（8 项 trace；`Classes scanned: rows-string-memory.md, rows-vectorized-tuning.md`；1 finding 三件套；排除 3 条；推导式 1 条） |
| 4 | Phase 4 输出要求满足 | ✅（已读 `maximal_lmul_for_misaligned_byte_processing.md` §Why this is slow 短引「LMUL 过小 → 每轮覆盖量低，loop-control 与配置成本占比过高」+ §The fix；The fix 含 before/after/前提/风险/信号；baseline；收益上界；三维路由；Related PRs） |
| 5 | 路径合规 | ✅（模式 A；8 项 trace；leaf 来自通过 gate 的 row） |
| 6 | Phase 5 两侧锚定 | ✅（消失侧 `111f62/111f6a`；出现侧 pattern §Verification） |
| 7 | 契约边界合规 | ✅ |

修正记录：无
