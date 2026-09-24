Functions under analysis: [internal/runtime/maps.memHashFallback]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`026-internal／runtime／maps.memHashFallback-annotate.txt`，覆盖 0x13eb0–0x14460 全函数（469 行反汇编），含长度分派（<4/==4/<8/==8/<=16/>48/16 字节循环）、48 字节主循环（0x14224–0x1444a）、尾部混合（0x14082–0x140b0）与零 panic 冷路径）
- perf stat（可选 bound/context）：已提供（`2-go-benchmark-riscv-BenchmarkAddMulVVWW_words_100000_impl_asm.txt`：IPC 0.776；branches 424,843,559 / branch_misses 20,607,457（4.85%）；L1_dcache_loads 1,158,850,266 / misses 14,311,913（1.24%）；LLC_loads 210,614,900 / misses 25,918,181（12.3%））
- workload/binary/DSO/source context：已提供（annotate 头部 DSO 为 `link`（Go 链接器 cmd/link——其自身 map 数据结构走本哈希）；源码链 `internal/runtime/maps/runtime_hash64.go:21-65` `memHashFallback` → `r8/r4`（:86-87）→ `runtime_alg.go:79-93` `readUnaligned32/64`（`goarch.BigEndian` 编译期常量折叠后取 `byteorder.LEUint64/LEUint32`）→ 8×`lbu`+shift+OR byte-assembly；`memhash_aes.go` 的 `memHashAESImplemented` 在 riscv64 为 false（AES 版仅有 amd64/386/arm64 汇编，见 memhash_amd64.s/memhash_386.s/memhash_arm64.s，无 memhash_riscv64.s），且硬件 ISA 无 `zk*`/`zvk*` 扩展 → fallback 是唯一路径；go commit 82215dc6c01ed6efb6f4f236cd5c87ceb6dc318a，master）
- readelf -A（热点 object 的 Tag_RISCV_arch）：缺失（详见 Phase 1）
- hardware ISA（/proc/cpuinfo / riscv_hwprobe）：已提供（metadata cpuinfo snapshot：`rv64imafdcv_zicbom_..._zba_zbb_zbc_zbs_...`，C920v2；无 zk*/zvk* 标量/向量 crypto 扩展）
- vlenb：已提供（vlenb=16 → VLEN=128 bits）
- 采样元数据（event / percent type / scope / 窗口）：已提供（annotate 头部原文 `for cpu-clock (2 samples, percent: local period)`；perf.data 共 264 samples；本函数局部份额 2/264）。重要上下文：DSO 为 `link`（Go 链接器），采样窗口覆盖 benchmark 构建/链接阶段子进程，非被测 benchmark 内核；`comparability_gap: 采样窗口与 perf stat counting 窗口可能不一致（sampling 覆盖 build/link 阶段）`
- Sampling IP precision：缺失（precise_ip / Exact-IP 未知，详见 Phase 1）

样本量声明：本函数仅 2 个 sample（264 总样本约 0.76%，rank 026），且落在 link 工具进程；样本量进入 confidence 推导（impact 降级），不省略任何小节。

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zba_zbb_zbc_zbs_zicond_zfa_...`（T-Head XuanTie C920v2，out-of-order，RVV 1.0；lbu/ld/lwu/slli/or/mul/mulhu 均为 base ISA；无 `zk*`/`zvk*` crypto 扩展） |
| Build ISA | `baseline_gap: build ISA`（精确 Tag_RISCV_arch 未提供）。32 位 zext 均为 `slli+srli`(0x20) 两指令形态、无 add.uw → 推断 Go 工具链（含 cmd/link）以默认 GORISCV64=rva20u64 构建。可选命令：`readelf -A <link 二进制>` |
| Vector flavor | 全函数 scalar、零 `v*`、零 `th.v*`；无 flavor mismatch。L0 基线事实：硬件有 `v` 而本函数不用 RVV——哈希混合为串行依赖链（mix = mulhu/mul/xor），RVV 不适用 |
| VLEN | 128 bits（vlenb=16） |
| Bound type | 全 workload counting：IPC 0.776、L1-dcache miss 1.24%、LLC miss 12.3%、branch miss 4.85%。函数级：hot path 为 8×`lbu`+shift+OR 依赖链与 mul 混合 → compute-bound |
| Sampling semantics | event=cpu-clock（可解释为时间）；percent type=local period（非 global-period）；同一采样窗口；函数贡献仅局部可知（2/264）。→ 不能称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata` 针对该声明）。样本归属 `link` 进程（构建/链接阶段），与 benchmark 计时指标（ns_per_op）不直接相关 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（precise_ip 未知）→ 140b0/141a2 行只锚定尾部混合区间与 ≤16 字节路径的 8 字节读取区间，不做单指令 latency 归因 |

L0 baseline gate 1：hardware 有 `v`、build（推断 rva20u64）无 `v` → 记录为 baseline finding，不停止扫描；本函数串行哈希链使向量化不适用。L0 baseline gate 2：无 `th.v*` → 无 vector-flavor mismatch。
Bound-type gate：函数级 compute-bound；不因 workload 级 memory 信号下调 route confidence，只约束 impact。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`internal/runtime/maps.memHashFallback`（rank 026，2/264 samples，入口条件 A / profile_backed，DSO=`link`）。
annotate 覆盖完整（0x13eb0–0x14460）：长度分派（s==0 返回、s<4 三点采样、s==4、s<8、s==8、s<=16、s>48 走 48 字节主循环 0x14224–0x1444a、否则 16 字节循环 0x140b4–0x1416a）、尾部混合（0x14082–0x140b0）与 epilogue；函数内无 panicBounds（unsafe 裸指针读取，调用方保证 s 字节可读）。
hot interval 1：尾部混合/返回区间 [0x14082..0x140b0]——sample 1 落于 `ret`（0x140b0）。
hot interval 2：`s<=16` 路径尾部 8 字节读取区间 [0x14170..0x14220]——sample 2 落于第二段 8 字节 byte-assembly 的 `slli t0,t0,0x20`（0x141a2）。
最高行原文：`50.00 : 140b0: ret` 与 `50.00 : 141a2: slli t0,t0,0x20`。
Sampling IP precision 不足 → 只锚定上述区间，不归因单条指令。
函数语义（runtime_hash64.go:21-65）：wyhash 系非加密哈希 fallback；`r8/r4`（:86-87）→ `readUnaligned64/32`（runtime_alg.go:79-93，LE 分支）→ `byteorder.LEUint64/LEUint32` 的 8×/4×`lbu`+shift+OR 字节拼装；主循环每 48 字节做 6 次 8 字节读取与 6 次 `mix`（bits.Mul64 的 mulhu/mul/xor）。调用者：cmd/link 内所有 map 的字符串/字节键哈希（如 goobj.BuiltinIdx 的 builtinMap 查询，见同批次 019 preloadSyms 的 mapaccess2_faststr 路径）；因 `memHashAESImplemented=false`（riscv64 无 memhash asm）且硬件无 `zk*`，fallback 是唯一哈希路径。

## Phase 3 — Pattern scan / 模式扫描：internal/runtime/maps.memHashFallback

### Class selection trace（8 项）

| Class | 判定 | 触发观察 |
|---|---|---|
| rows-asm.md | exclude | 当前代码来源是编译器生成的 Go（internal/runtime/maps/runtime_hash64.go），非手写 `.S`；riscv64 不存在 memhash 汇编实现（policy/existence 四证缺 policy 证据，见替代解释） |
| rows-operator-rvv.md | exclude | 无算子语义循环合同（哈希混合为串行 mul 依赖链，非 elementwise/GEMM 等）；零 `v*` |
| rows-string-memory.md | include | 固定宽度 logical access：8 字节/4 字节 word 以 `lbu`+shift+OR 逐字节拼装（13ef8-13f86、13f8a-13fce、140b4-1416a、14170-14220、14224-1444a 多处），wide-scalar-memory row signal；LE 语义（byteorder.LEUint64） |
| rows-vectorized-tuning.md | exclude | 函数内零 `v*` |
| rows-codegen.md | include | 编译器生成代码的指令形态：缺失 LE word-load 折叠（ld/lwu 未发出）；多处 zext 站点与 nop 填充待评估 |
| rows-offload.md | exclude | 无矩阵引擎 / packed-SIMD / 权重重排证据 |
| rows-crypto.md | exclude | memHashFallback 是非加密数据哈希（非 AES/SHA/SM 等密码学原语），无 profile evidence 点名密码原语；且硬件无 `zk*`/`zvk*` |
| rows-runtime-os.md | exclude | 用户态 Go 运行时/链接器代码；无 RTOS/kernel 侧 timer/CSR/PMP 证据 |

Classes scanned: rows-string-memory.md（全文 12 行逐一评估）、rows-codegen.md（全文 27 行逐一评估）

### Local performance pattern scan: `internal/runtime/maps.memHashFallback`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Wide Scalar Memory-Access Code Generation（primary） | 8 字节/4 字节 word 全部以 `lbu`+shift+OR byte-assembly 物化（每 8 字节 22 条指令），sample 落于 ≤16 字节路径的 8 字节读取区间与尾部混合；源码为 `byteorder.LEUint64/LEUint32`（LE 分支） | High | Low | `patterns/wide_scalar_memory_access_codegen.md` |
| 其余 rows-string-memory / rows-codegen 行 | 全部排除（见逐 row 排除表） | — | — | — |

**(a) 逐字 evidence 引用**（采样 IP precision 不足，单行只锚定 interval）：
- `50.00 : 140b0: ret` —— 尾部混合/返回区间 [0x14082..0x140b0]（`mix(m5^s, mix(a^hashkey[1], b^seed))` 的 mulhu/mul/xor 链之后）。
- `50.00 : 141a2: slli t0,t0,0x20` —— `s<=16` 路径第二段 8 字节 byte-assembly（[0x14170..0x14220]）内；序列原文：`0.00 : 14170: addi a4,a2,-16` / `0.00 : 1417c: lbu a6,0(a4)` … `0.00 : 141bc: lbu a4,7(a4)` / `0.00 : 141c0: slli a4,a4,0x38`（第一段 8 字节）与 `0.00 : 141c2: lbu t0,0(a5)` … `0.00 : 14202: lbu a5,7(a5)` / `0.00 : 14206: slli a5,a5,0x38`（第二段 8 字节）—— 每段 8×`lbu`+7×`slli`+7×`or` = 22 条指令。
- 16 字节路径同型序列：`0.00 : 13ef8: add a1,a1,a0` … `0.00 : 13f82: or a3,a3,a4` / `0.00 : 13f84: or a1,a1,a5`（两段 8 字节）。
- 48 字节主循环（0x14224–0x1444a）每轮 6 段 8 字节 byte-assembly + 6 次 mulhu/mul/xor；回边 `0.00 : 14454: bltu a1,a2,14224`。
- 源码锚点：runtime_hash64.go:40-42（`a = r8(p); b = r8(add(p, s-8))`）、:86-87（`r8 → readUnaligned64`）、runtime_alg.go:87-93（`readUnaligned64` 的 `byteorder.LEUint64(q[:])` 分支）；`goarch.BigEndian` 在 riscv64 为编译期常量 false（LE 分支唯一存活，反汇编无 BE 分支）。

**(b) 互斥邻居排除**：
- vs. `kernel_selection_and_runtime_specialization.md`（rows-codegen）：行内互斥要求"仓库中存在满足合同的专用内核"——riscv64 不存在任何 memhash 汇编（memhash_amd64.s/memhash_386.s/memhash_arm64.s 之外无 riscv64 版本），`memHashAESImplemented=false`（memhash_noaes.go:14）→ 无可用专用实现，kernel-selection row 不命中（互斥："没有可用专用实现且 loop 为 scalar → 对应 operator/no-vectorization row"）。
- vs. `policy_backed_missing_riscv_assembly_kernel.md`（rows-asm）：policy/existence 四证缺"函数适用 assembly-default policy"证据——Go runtime 对 riscv64 memhash 无书面汇编政策（amd64/arm64 有实现仅为既有事实，非政策）；且硬件无 `zk*`，AES 版 memhash 即使实现也不可用。→ 仅记录为替代解释（hypothesis），不构成 matched finding。
- vs. `scalar_swar_checksum_reduction.md`（rows-string-memory）：本函数是 wyhash 系非加密哈希，主循环为 8 字节读取 + `mix`（bits.Mul64 的 mulhu/mul/xor），非"逐 byte load + s1+=byte; s2+=s1"型 checksum 依赖链 → 不命中。
- vs. `native_word_size_instruction_selection.md`（rows-codegen）：zext 站点（如 14014-1401c、14048-1404c、13f74 后）为 uint32→uint64/移位宽度语义必需，非 producer 已扩展后的冗余延伸 → 不命中。
- vs. `alignment_priming_for_word_sized_memory_operations.md`：`readUnaligned64` 语义即未对齐读取，RISC-V `ld` 无对齐要求、无 fault 路径；问题纯粹是编译器未折叠 → 不命中。
- vs. `rvv_compare_first_difference`/`rvv_memory_copy_fill` 等行：非两输入比较/连续 copy——单输入哈希读取。
- vs. rows-crypto 各 row：memHashFallback 非密码学原语（无 profile 点名），硬件无 `zk*`/`zvk*` → 不命中。
- vs. `resource_aware_instruction_scheduling.md`：函数内多处 `nop` 填充（13f7a-13f80、13fc4-13fc8、14012、1401e、14046、1415c-14166、14208-14212、14428-1444a）为编译器调度/对齐产物，但无目标 core 的 latency/resource-model 与 PMU 交叉证据 → 不命中。
- vs. `trap_based_guard_and_failure_paths.md`：函数内无 bounds 检查（unsafe 裸指针，调用方保证可读），无 trap/fault 语义可借 → 不适用。

**(c) 双 Confidence 推导式**：
- route：compiler-generated provenance（runtime_hash64.go → runtime_alg.go → byteorder.LEUint64 源码链与反汇编逐条对应）+ LE 语义编译期确定（`goarch.BigEndian` 常量折叠）+ 精确 8/4 字节读取（单条 `ld`/`lwu` 与语义长度完全一致、无 over-read）+ 未对齐读取由 `readUnaligned64` 命名契约显式许可 → High。
- impact：sample share 极小（2/264 ≈ 0.76%）且位于 `link` DSO（构建/链接阶段）；percent type=local period；IP precision 未知；单调用收益比例高（≤16 字节键约 44 条 byte-assembly → 2×`ld`，节省约 36 条/调用）但调用频率样本不可分账 → Low。

### 多命中仲裁小段
本函数唯一顶层 finding：Wide Scalar Memory-Access Code Generation（rows-string-memory.md，L2 data movement / integration detail 层）。kernel-selection 因无 riscv64 专用内核不命中；missing-`.S` 因缺 policy 四证仅作替代解释；SWAR、native-word、alignment、crypto 各行因行内互斥/信号不符被排除。无 companion / independent 命中。入口条件 A：primary finding 的 evidence sample share 加总 = 2/264 ≈ 0.76%（当前 sampled event 下的局部份额，且归属 link 进程）。

## Phase 4 — Root-cause blueprint / 根因蓝图：internal/runtime/maps.memHashFallback

（纳入蓝图的 pattern：Wide Scalar Memory-Access Code Generation；依据 `patterns/wide_scalar_memory_access_codegen.md` §Why this is slow / §The fix）

1. **Root cause**：riscv64 后端未把 `byteorder.LEUint64/LEUint32`（经 `readUnaligned64/32` 内联）的 byte-or 表达式折叠为单条 `ld`/`lwu`，使 memHashFallback 每 8 字节读取付出 22 条 `lbu`+shift+OR 指令（如 pattern §Why this is slow 所言："逐 byte 拼装会增加 load、shift、OR 和依赖链"——每段读取形成 8 级 load-to-use 依赖段）。因 `memHashAESImplemented=false`（memhash_noaes.go:14，riscv64 无 memhash 汇编且硬件无 `zk*`），本函数是 link 进程所有 map 字符串/字节键哈希的唯一路径；≤16 字节键（map 键的常见形态）每次调用约 44 条 byte-assembly 指令，若折叠为 `ld` 仅 2 条。`goarch.BigEndian` 为编译期常量 false（LE 分支唯一存活），`readUnaligned64` 命名契约显式允许未对齐读取，且每段读取长度与语义长度完全一致（无 over-read）——单次宽访问在语言与平台上均安全。
2. **The fix / 修复方式**（Go 工具链 codegen 修复；不是对 maps 源码的修改）：
   - before（disasm，≤16 字节路径第一段）：`lbu a6,0(a4); lbu t0,1(a4); slli t0,t0,0x8; or a6,t0,a6; … lbu a4,7(a4); slli a4,a4,0x38`（22 条）；after：`ld a6,0(a4)`（1 条）。其余 8 字节段（13ef8 路径、140b4 循环、14170 路径、14224 主循环）同理；4 字节段（1402a、13fd2 路径）→ `lwu`。
   - 实现位置：cmd/compile/internal/ssa/gen/RISCV64.rules——增加 `Or64/Or32`+`Lsh64x*/Lsh32x*`+`ZeroExt8to64/32`+load 组合 → `MOVDload`/`MOVWUload` 折叠规则（与 amd64/arm64 后端同类折叠对齐），以读取宽度与语义长度一致（无 over-read）及 `config.LittleEndian` 为门控；BE riscv64 端口保留 portable 路径或补 `rev8`。
   - 适用前提：LE riscv64；`readUnaligned64/32` 契约（未对齐 + 精确宽度读取）；目标核未对齐 `ld` 性能可接受（pattern §Verification 实机 A/B；C920v2 硬件支持未对齐访问，`riscv_hwprobe` 的 MISALIGNED_SCALAR_PERF key 可确认）。
   - correctness contract：保持 strict-aliasing 与有效 object representation 语义（pattern §Pattern-local contract）；不 over-read（每段恰好 8/4 字节）；volatile/atomic/MMIO 语义不受影响（纯数据哈希读取）；哈希数值输出不变（LE 拼装值 == 原生 LE 加载值）。
   - 限制/风险：样本极少且属链接阶段；未对齐 `ld` 在部分核可能慢于对齐加载，须实测；折叠规则改进需 `go test cmd/compile/internal/ssa` 与 riscv64 `all.bash` 回归。
   - 预期 Profile 信号：13ef8-13f86、13f8a-13fce、140b4-1416a、14170-14220、14224-1444a 的 byte-assembly 区间各收敛为单条 `ld`/`lwu`；≤16 字节键每次调用约 44 → 8 条（约 -80%），48 字节循环每轮 132 → 12 条（约 -91%）。
3. **Baseline facts 回填**：hardware ISA rv64imafdcv+zba/zbb/zbc/zbs/zicond/zfa（C920v2，OoO，无 zk*）；build ISA `baseline_gap: build ISA`（推断 rva20u64 默认）；VLEN 128（vlenb=16，未参与）；bound type 函数级 compute-bound（全 workload IPC 0.776）。
4. **收益上界**：入口条件 A——当前 sampled event（cpu-clock）下的局部样本份额 2/264 ≈ 0.76%；该份额归属 `link`（Go 链接器）进程，反映采样窗口内构建/链接阶段执行，与 benchmark 计时指标（ns_per_op）无直接关联。percent type=local period → 不称 workload 级 Amdahl 上界（`baseline_gap: sampling metadata`）。
5. **三维路由判定**：
   - current source：编译器生成的 Go 代码（internal/runtime/maps/runtime_hash64.go:21-65 `memHashFallback`，`readUnaligned64` → `byteorder.LEUint64` 内联为 byte-assembly；go commit 82215dc6，默认 rva20u64 工具链构建）。
   - implementation existence/reachability：riscv64 无 memhash 专用实现（memhashAESImplemented=false），无 dispatch/fallback 选择问题；修复对象是 cmd/compile riscv64 后端折叠规则。
   - function-level policy：不适用——Go runtime 对 riscv64 memhash 无独立 `.S` 政策（missing-`.S` 四证缺 policy 证据）；不进入 missing-`.S` 分支。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs 小节**：`patterns/wide_scalar_memory_access_codegen.md` §Related PRs —— zlib-ng 2 条（https://github.com/zlib-ng/zlib-ng/commit/d7e121e56b64b5916810cf32615062a53f954773、https://github.com/zlib-ng/zlib-ng/commit/7a859e8cc350f3983236bd67522b5e7c2acec85a）。`Related PRs：2 条 URL`。

## Phase 5 — Verification forecast / 验证预测：internal/runtime/maps.memHashFallback

- 消失/缩小侧（锚定 Phase 3(a) 引用行）：实现 riscv64 LE 宽加载折叠后，反汇编中 `0.00 : 1417c: lbu a6,0(a4)` 至 `0.00 : 141c0: slli a4,a4,0x38` 与 `0.00 : 141c2: lbu t0,0(a5)` 至 `0.00 : 14206: slli a5,a5,0x38` 的两段 22 条序列各收敛为单条 `ld`；`50.00 : 141a2: slli t0,t0,0x20` 行消失；16 字节路径（13ef8-13f86）与 48 字节主循环（14224-1444a）同型序列同步收敛；`50.00 : 140b0: ret` 前混合链保持不变（算法性）。
- 出现侧（锚定 pattern §Verification）：`go tool objdump`/`go tool compile -S` 显示 memHashFallback 内出现 `ld`/`lwu`；运行 `go test cmd/compile/internal/ssa`、`go test internal/runtime/maps` 与 riscv64 `all.bash` 全绿；覆盖未对齐地址、page boundary、短输入（s=0/1/2/3/4/7/8/15/16/17/48/49）、长输入（48 倍数与非倍数尾）与错误路径（pattern §Verification "覆盖所有合法 alignment、page boundary、短 object、endianness 和 alias combinations"）；哈希输出与折叠前逐位一致（同输入同 seed）；对比指令数与 elapsed time，确认未对齐 `ld` 在 C920v2 上无回退。
- 额外核对：重跑含链接阶段的 perf record/annotate，memHashFallback 每次调用退休指令数显著下降；可用 micro-benchmark（`go test -bench` maps 相关）做指令/周期对比。
- 要把本结论升级为可定量的函数级收益判断，最少需补采：global-period annotate（`--percent-type=global-period`）、precise_ip 确认（`perf evlist -v`），以及采样窗口内 link 阶段与 benchmark 阶段的归属区分（`perf script`/comm 过滤）。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现 | ✅（载荷：1/1 组；`internal/runtime/maps.memHashFallback`） |
| 2 | Phase 1 输出要求满足：7 行 baseline 表 + 2 个 L0 gate 判定 + bound-type gate | ✅（载荷：7 行；gap 标签 4 个：`baseline_gap: build ISA`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`、`comparability_gap: 采样窗口与 counting 窗口可能不一致`；含 `Sampling IP precision` 行） |
| 3 | Phase 3 输出要求满足：8 项 Class selection trace；`Classes scanned: rows-string-memory.md, rows-codegen.md`；顶层 finding 1 个（Wide Scalar Memory-Access Code Generation）；evidence 锚点（140b0、141a2、14170-14220、13ef8-13f86、14224-1444a、runtime_hash64.go:40-42/86-87、runtime_alg.go:87-93）；supporting 0；排除 9 条；推导式 2 条 | ✅ |
| 4 | Phase 4 输出要求满足：已读 pattern 文件 `patterns/wide_scalar_memory_access_codegen.md`；对应命中 row Wide Scalar Memory-Access Code Generation；引用短语首词 "逐 byte 拼装会增加 load、shift、OR 和依赖链"；The fix 含 before/after、correctness、风险、Profile 信号；Related PRs 2 条 URL | ✅ |
| 5 | 路径合规：8 类 trace 扫描集；唯一 primary、L2 归属；入口模式 A 按动态份额排序（0.76%）；`th.v*` 未停扫（无 th.v* 出现） | ✅（载荷：模式 A；class 列表见 Phase 3） |
| 6 | Phase 5 两侧锚定：消失侧对上 Phase 3(a) 引用行（1417c-141c0、141c2-14206、141a2、13ef8-13f86、14224-1444a）；出现侧标注 pattern §Verification | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付止于 Profile 证据、根因蓝图、完整 The fix 和验证预测 | ✅ |

修正记录：无