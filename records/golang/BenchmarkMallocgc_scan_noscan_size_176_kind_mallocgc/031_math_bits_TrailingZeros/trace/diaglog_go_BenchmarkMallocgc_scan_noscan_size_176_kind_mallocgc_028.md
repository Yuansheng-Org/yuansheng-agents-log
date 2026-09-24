Functions under analysis: [runtime.nextFreeFast]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

| Evidence | 状态 |
|---|---|
| 单函数完整 perf annotate | 已提供（`028-runtime.nextFreeFast-annotate.txt`，cpu-clock，3 samples，percent: local period，完整覆盖函数 2b2e8–2b388，含 hot fast-path body） |
| perf stat | 已提供（`2-go-benchmark-riscv-BenchmarkMallocgc_scan_noscan_size_176_kind_mallocgc.txt`，含 cycles/instructions/IPC/cache/branch counters） |
| workload/binary/DSO/source context | 已提供（`go` 可执行文件（cmd/go 驱动进程）内 `runtime.nextFreeFast`；source context 经工作区 go 源码核对：commit `82215dc6c0`（go1.27-devel）与 metadata 一致；`src/runtime/malloc.go:969`、`src/internal/runtime/sys/intrinsics.go:53`、`src/cmd/compile/internal/ssagen/intrinsics.go`、`src/cmd/internal/obj/riscv/obj.go`） |
| readelf -A（Tag_RISCV_arch） | 缺失（详见 Phase 1；反汇编形态为强间接证据） |
| hardware ISA（/proc/cpuinfo / hwprobe） | 已提供（metadata cpuinfo：`rv64imafdcv_..._zbb_zbc_zbs_...`，含 Zbb/Zba/Zbs；C920v2，OoO） |
| vlenb | 已提供（metadata：vlenb=16 → VLEN=128 bits） |
| 采样元数据 | 部分提供（event=`cpu-clock`，percent type=local period，单次运行同窗口；函数 workload 贡献已知为整记录 3/465=0.65%，但记录混合 go/link/compile 多进程——详见 Phase 1） |
| Sampling IP precision | 缺失（`precise_ip` 未知；cpu-clock@99Hz——详见 Phase 1） |

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zba_zbb_zbc_zbs_...`（C920v2，OoO superscalar；`zbb` 存在 → 硬件提供 `ctz`/`zext.h`/`sext.b` 等） |
| Build ISA | `baseline_gap: build ISA`（无 `readelf -A` artifact）；间接证据：annotate 内 ctz 为 De Bruijn 软件序列（2b2f6–2b314）、零扩展为 `slli+srli` 对，与 Go 默认 `GORISCV64=rva20u64`（zbootstrap.go `DefaultGORISCV64 = rva20u64`）一致；同 commit 工具链在 `GORISCV64>=22` 时生成 ACTZ/AZEXTH（obj.go:4070、4088、4620） |
| Vector flavor | annotate 无任何 `v*`/`th.v*`；硬件为 RVV 1.0；本函数全 scalar，无 flavor mismatch 影响 |
| VLEN | 128 bits（vlenb=16）——对 scalar ctz finding 不构成 gate |
| Bound type | 混合/控制型：IPC 0.676；L1-dcache miss 率 27.4M/1960.9M=1.4%；LLC load miss 34.6M/160.9M≈21.5%；branch miss 44.7M/921.0M=4.85%；非明显 memory-bound，非纯 compute-bound |
| Sampling semantics | event=`cpu-clock`（时间基，非 cycles）；`--percent-type` local period；同一运行窗口；函数级贡献=整记录 3/465=0.65%，但 perf record 混合了 `go` 驱动、`link`、`compile` 进程（benchmark 本体 `runtime.test` 未符号化）→ 不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（`precise_ip` 未知；单行占比只锚定 basic block / interval，不承担单指令 latency 归因） |

L0 baseline gate 1（hardware 有 `v`、build 无 `v`）：**成立**——硬件暴露 RVV 1.0 而 Go 默认 rva20u64 构建不生成 `v*` 代码；作为最高优先级 baseline finding 记录（影响全二进制），但不停扫，且对 nextFreeFast 不产生 no-vectorization 命中（该函数非循环、无数据级并行，见 Phase 3 排除）。L0 baseline gate 2（`th.v*` flavor gate）：不适用（无 `th.v*` mnemonic）。Bound-type gate：混合 bound，本轮 findings 的 performance-impact confidence 因此不封顶于 memory-bound 下调，但仍受采样元数据与样本量限制。

## Phase 2 — Scope / 分析边界

函数清单与承诺一致：`[runtime.nextFreeFast]`（1 个）。hot interval：fast-path body `2b2e8–2b382`（`ld s1,56(a0)` 起至 `ret`）。最高占比行（各 33.33%，3 样本中占 3 行）：

- `33.33 :  2b2f6:  neg  a1,s1` —— ctz64 De Bruijn 区间起点
- `33.33 :  2b304:  mul  a1,a1,a2` —— ctz64 De Bruijn 区间内（deBruijn64 乘法）
- `33.33 :  2b36c:  sh   s1,96(a0)` —— allocCount 16-bit 自增存储

ctz64 De Bruijn 区间（`2b2f6–2b314`）持有 2/3 样本；`2b36c` 独立持有 1/3。Sampling IP precision 未知 → 以上行只锚定所在 basic block / interval（ctz 区间、allocCount 存储点），不做单指令 cycle 归因。annotate 覆盖完整（含全部 exit 路径 `2b34c`/`2b382`/`2b388`），入口模式 **A（profile_backed）**。

## Phase 3 — Pattern scan / 模式扫描：runtime.nextFreeFast

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源为 compiler-generated（Go cmd/compile riscv64 backend 生成，annotate 指令形态 + 同 commit 工具链源码证实），非手写 `.S`；无 missing-`.S` 的 policy/existence 证据（Go 无 ctz 手写汇编政策）。
2. `rows-operator-rvv.md` — **exclude**：无算子语义循环；nextFreeFast 是标量控制型分配快路径（无 loop、无数据级并行、无 elementwise 合同）；hardware-V 但无 loop 时不适用 no-vectorization。
3. `rows-string-memory.md` — **exclude**：无 copy/fill/compare/checksum/memclr/back-reference 语义。
4. `rows-vectorized-tuning.md` — **exclude**：annotate 无任何 `v*` 指令。
5. `rows-codegen.md` — **include（必选）**：compiler-generated 标量代码，热点落在指令形态问题（ISA 扩展替换、窄状态零扩展、常量池访问、控制流形态）上。
6. `rows-offload.md` — **exclude**：无矩阵引擎 / packed-SIMD / 权重重排 / 多线程分块证据。
7. `rows-crypto.md` — **exclude**：非密码学原语（deBruijn64tab 查表是 ctz 算法实现，不是密码查表）。
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel 侧 timer/ISR/CSR/PMP 热点；是用户态 Go runtime mcache 分配快路径。

Classes scanned: `rows-codegen.md`（27 rows 全部逐行评估）。

### Local performance pattern scan: `runtime.nextFreeFast`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RISC-V ISA Extension-Specific Instruction Substitution（primary） | 逐字 annotate：`33.33 :  2b2f6:  neg a1,s1` 与 `33.33 :  2b304:  mul a1,a1,a2`（De Bruijn ctz 区间 2b2f6–2b314，10 条指令合成 `ctz`）；`2b32a: slli a4,a2,0x30; 2b32e: srli a4,a4,0x30` 与 `2b370–2b374` 两处 `slli+srli` zext 对（MOVHU 零扩展）；hardware ISA 含 `zbb`；build 为默认 rva20u64（无 Zbb 代码生成） | High | Low | `patterns/isa_extension_specific_instruction_substitution.md` |
| Runtime CPU Feature Dispatch for ISA-Specific Intrinsics and VDSO（supporting） | 同一 ctz 序列；`internal/cpu.RISCV64.HasZbb`（cpu_riscv64_linux.go:98，riscv_hwprobe）已存在，`runtime.riscv64HasZbb`（proc.go:797）已设置，但编译器只在 OnesCount 做运行时 dispatch（intrinsics.go:1168-1199 `makeOnesCountRISCV64`），TrailingZeros64 在 rva20u64 下无任何 Zbb 接线 | — | — | `patterns/runtime_isa_specific_instruction_dispatch.md` |

**Finding 1（primary）三件套：**

(a) 逐字 evidence 引用：
- `33.33 :  2b2f6:  neg  a1,s1`（ctz64 De Bruijn 区间起点，`sys.TrailingZeros64(s.allocCache)` 内联体）
- `33.33 :  2b304:  mul  a1,a1,a2`（同一区间，`(x&-x)*deBruijn64` 乘法；区间内另含 `auipc+ld` 常量池取 deBruijn64 常数、`srli 0x3a`、`auipc+addi` deBruijn64tab 基址、`lbu` 查表）
- 区间整体 `2b2f6–2b314` 持 2/3 函数内样本；零扩展对 `2b32a: slli a4,a2,0x30` / `2b32e: srli a4,a4,0x30` 与 `2b370: slli s1,a2,0x30` / `2b374: srli s1,s1,0x30`（另有 `2b340–2b344` 一对）为 `lhu` 后 MOVHU 的 `slli+srli` 合成。Sampling IP precision 未知 → 锚定 interval 级别机制。

(b) 互斥邻居排除：
- **runtime-ISA-dispatch row**（rows-codegen line 34）：行内互斥判据「单个 instruction sequence 的替换走 isa_extension_specific_instruction_substitution」→ 本证据是单一指令序列替换，归 primary；dispatch 接线缺口作为 supporting（同一机制的另一修复路径），不另立顶层 finding。
- **algebraic-simplification row**（line 26）：`neg;and` 是 De Bruijn ctz 配方的一部分（`x & -x`），不是可独立消除的代数冗余；行内互斥「ISA extension 的 native instruction 替换 → ISA-substitution row」→ 排除。
- **native-word-size row**（line 33）：`slli+srli` zext 对的行内互斥「核心证据是 Zbb/Zba native extension 替换 → isa_extension_specific_instruction_substitution」→ 排除（随 primary 一并修复为 `zext.h`）。
- **ALU-constant row**（line 27）：`auipc+ld` 取 deBruijn64 常量，行内互斥「Zba/Zbb 单指令 substitution → ISA-substitution row」；且 64-bit 常量无法用更短立即数序列物化，ctz 替换后该常量访问整体消失 → 排除。
- **code-layout/constant-pool row**（line 19）：无 frontend/i-cache 或常量池 footprint 独立证据（3 样本无一落在 `auipc` 行）；两次常量池引用是 ctz 序列组成部分 → 排除。
- **native-width-state row**（line 32）：mspan 的 `freeindex`/`nelems`/`allocCount` 为 uint16（mheap.go:448/451/504），行内互斥「packed layout … 依赖窄宽度时不命中」——mspan 为保持热结构紧凑而刻意打包窄字段；且非 loop-carried → 排除。
- **register-pressure/save-restore row**（line 21）：s1（X9）在 Go riscv64 内部 ABI 中为参数/调用者保存寄存器（abi-internal.md「X10–X17, X8, X9, X18–X23 for integer arguments」），函数无 prologue/epilogue 保存，无 spill/reload、无冗余 `mv` → 排除。
- **resource-aware-scheduling row**（line 25）：需 target-core PMU 与 dependency/barrier 证据；本证据仅 3 个 cpu-clock 样本且 IP precision 未知，row gate 不成立 → 排除。
- **control-flow row**（line 18）：分支（bnez/bge/bgeu/bnez/beq）是 nextFreeFast 算法性守卫（allocCache==0、theBit≥64、result≥nelems、freeidx 边界），无 branch-diamond/indirect-call/RAS 污染证据 → 排除。
- **zero-based-comparison row**（line 36）：`li a1,64`/`li a2,64` 是宽度常量物化，非 `li reg,0`+branch / SLT / SNEZ zero-operand 形态 → 排除。
- **hot-helper-inlining row**（line 17）：`sys.TrailingZeros64` 已被内联（函数内无 call/jal），无 missed-inline 证据 → 排除。
- **kernel-selection row**（line 13）：无 generic/fallback vs 专用 kernel 的分派；nextFreeFast 本身即分配快路径 → 排除。
- **trap-based-guard / hardware-atomic / spin-wait / redundant-sync / tail-call / JIT / loop-IV / FP 各 row**：本函数无 fault-guard、无原子操作、无轮询、无同步、无 wrapper、非 JIT 代码、无循环、无 FP 指令 → 逐项排除。

(c) 双 Confidence 推导式：
- route：hardware Zbb 直接证据（cpuinfo）+ 工具链 rva22u64 路径可达（同 commit intrinsics.go:962 `if cfg.goriscv64 >= 22`、obj.go:4070/4088/4620）+ 行内互斥邻居全部以判别性观察排除 → **High**。
- impact：局部样本份额明确（ctz 区间 2/3 函数内样本）但缺采样语义四条（cpu-clock 时间基、record 混合多进程、函数 workload 贡献非 benchmark 本体）、`baseline_gap: sampling IP precision`、样本量仅 3 → **Low**。

**Supporting evidence（primary 之下）**：(a) `internal/cpu.RISCV64.HasZbb` 经 riscv_hwprobe 检测（cpu_riscv64_linux.go:98）+ `runtime.riscv64HasZbb`（proc.go:797、cpuflags.go:45）已接线，但编译器对 TrailingZeros64 在 `goriscv64<22` 下无 ctz dispatch（intrinsics.go:962-983 仅静态 gate）；`supporting because: 同一机制（Zbb ctz 可达性）的另一修复路径，编译期构建 gate 与运行时 dispatch 都针对同一 De Bruijn 序列，随 primary 修复后该信号消失。两种 confidence 保持 `—`，不另立顶层 finding、不计入命中数。

**多候选仲裁小段**：仅 1 个顶层 finding（primary）。支持性候选 runtime-ISA-dispatch 与 primary 共享同一 evidence（2b2f6–2b314 De Bruijn 序列）与同一修复机制（Zbb ctz），不可分账 → 按 arbitration「多个候选解释同一 hot loop、同一机制 → 选择唯一 primary，较弱候选改为 supporting」。Evidence-mechanism layer：本次 evidence 指向 L4（compute/codegen micro-structure — `isa_extension_specific_instruction_substitution` 属 L4 白名单）；L0 build-profile 观察（hardware 有 `zbb` 而 build 无）作为 Phase 1 baseline finding 记录，按「上层修复使下层 signal 消失则下层并入 supporting」由 L4 primary 认领根因。入口条件 A：顶层 finding evidence sample share 加总 = ctz 区间 2/3 函数内样本（66.7% local；整记录 0.43%，2/465）。

## Phase 4 — Root-cause blueprint / 根因蓝图：runtime.nextFreeFast

纳入蓝图的 pattern：`patterns/isa_extension_specific_instruction_substitution.md`，对应 Phase 3 已通过 gate 的 row「RISC-V ISA Extension-Specific Instruction Substitution（primary）」。

1. **Root cause**：Go runtime 分配快路径 `nextFreeFast`（`src/runtime/malloc.go:969`）调用 `sys.TrailingZeros64(s.allocCache)`；`internal/runtime/sys.TrailingZeros64` 是 `math/bits.TrailingZeros64` 的 alias（intrinsics.go:1290-1291）。在默认 `GORISCV64=rva20u64` 构建下 riscv64 后端不内联化为 ctz（intrinsics.go:962 `if cfg.goriscv64 >= 22` 为唯一 gate），改为内联纯 Go De Bruijn 实现（`intrinsics.go:53-68`：`deBruijn64tab[(x&-x)*deBruijn64>>(64-6)]`）。annotate 中该实现占 10 条指令（`2b2f6–2b314`），含常量池 `auipc+ld` 取 deBruijn64 常数与 `auipc+addi`+`lbu` 查表。依据 pattern §Why this is slow 第 1 点：多指令模拟「consumes extra decode, issue, and retirement slots」；且 `neg→and→mul→srli→add→lbu` 是串行依赖链，`mul` 居中。另：MOVHU 零扩展以 `slli+srli` 对合成（obj.go:4096-4103 rva20u64 分支），函数内 3 对。C920v2 硬件暴露 Zbb，`ctz` 与 `zext.h` 均为单指令。

2. **The fix / 修复方式**（构建/工具链层指令替换，不修改 mcache/malloc 源码语义）：
   - **首选（静态构建 gate）**：以 `GORISCV64=rva22u64` 重建 Go 工具链与 runtime → `TrailingZeros64` 内联为单条 `ctz`（intrinsics.go:962-983 已实现），MOVHU 零扩展降为 `zext.h`（obj.go:4088-4090 已实现）。修正对象是构建配置。
   - **备选（运行时 dispatch）**：把 `makeOnesCountRISCV64` 式 dispatch（intrinsics.go:1168-1199，加载 `runtime.riscv64HasZbb` 后 BranchLikely 分派）扩展到 `TrailingZeros64/32/16/8`，使 rva20u64 默认构建在 Zbb 硬件同样发出 `ctz`，非 Zbb 硬件保留 De Bruijn fallback。修正对象是 `cmd/compile/internal/ssagen/intrinsics.go`。
   - Before/After 伪代码：
     ```asm
     ; Before（rva20u64，annotate 2b2f6–2b314，10 条）
       neg   a1, s1
       and   a1, a1, s1
       auipc a2, 0x5e1
       ld    a2, -308(a2)      ; rodata deBruijn64 常量
       mul   a1, a1, a2
       srli  a1, a1, 0x3a
       auipc a2, 0xca8
       addi  a2, a2, 438        ; deBruijn64tab
       add   a1, a1, a2
       lbu   a1, 0(a1)
     ; After（rva22u64 / Zbb dispatch）
       ctz   a1, s1             ; 单条；ctz(0)=64 与 Go TrailingZeros64(0)=64 一致
     ```
     ```asm
     ; Before（MOVHU 零扩展）
       slli  a4, a2, 0x30
       srli  a4, a4, 0x30
     ; After
       zext.h a4, a2
     ```
   - 适用前提：硬件 Zbb（C920v2 满足，cpuinfo 含 `zbb`）；工具链 ≥ go1.27-dev@82215dc（rva22u64 路径已存在）。
   - 不可破坏的 correctness contract：RISC-V `ctz` 对 x=0 返回 XLEN=64，与 Go `TrailingZeros64(0)=64` 语义逐位一致（pattern §Verification「ctz(0) … semantics must match the language or runtime helper, including any width-specific return value」）；非零 x 返回 trailing-zero 数 ≤63 与 De Bruijn 结果一致。`zext.h` 对 uint16 值精确零扩展；`result<nelems` 守卫（2b330）保证窄值语义，freeidx 的 `&63`/`==nelems` 边界检查（2b338/2b346）不受影响。fallback（rva20u64 或非 Zbb）行为与现版本一致。
   - 限制/风险：rva22u64 全量重建影响整个二进制（全部 Zbb 替换：ctz/clz/cpop/zext.h/sext.b 等），需完整跑 Go 测试套件（math/bits、runtime、crypto、cmd/compile SSA 测试）；运行时 dispatch 方案需保证 `internal/cpu` init（cpuinit → proc.go:797）先于任何 ctz 调用；两方案均不改变分配算法语义。
   - 修复后预期 Profile signals：nextFreeFast 内 `2b2f6–2b314` 区间全部指令消失并出现 `ctz`；3 对 `slli+srli` 变 `zext.h`；函数指令数从 49 条降至约 34–37 条；ctz 区间样本显著下降（现占 2/3 函数内样本）。

3. **Baseline facts 回填**：hardware ISA = `rv64imafdcv_..._zbb...`（C920v2，OoO）；build ISA = `baseline_gap: build ISA`（无 readelf -A；反汇编形态 + 默认 rva20u64 一致）；VLEN = 128 bits（scalar finding 不依赖）；bound type = 混合（IPC 0.676、L1 miss 1.4%、branch miss 4.85%）。

4. **收益上界**：当前 sampled event（cpu-clock）下局部样本份额 —— ctz 区间 2/3 函数内样本（66.7% local；整记录 2/465≈0.43%）。Phase 1 采样语义四条不全成立（事件为时间基 cpu-clock、record 混合多进程、函数贡献不在 benchmark 本体进程）→ **不得表述为 workload 级 Amdahl 上界**；仅表述为局部份额。

5. **三维路由判定**：
   - current source：compiler-generated（Go cmd/compile riscv64 backend；annotate 形态 + 同 commit 工具链源码证据）。
   - implementation existence/reachability：`ctz`/`zext.h` 已在工具链实现（obj.go:4620 ACTZ/AZEXTH，GORISCV64>=22 gate）且 rva22u64 构建路径可达；运行时 dispatch 已存在于 OnesCount（intrinsics.go:1168）但未接到 TrailingZeros64。
   - function-level policy：Go 无手写汇编 policy 约束本函数；修复落在构建配置/编译器代码生成层，不进入 missing-`.S` 分支。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支，无 `implementation_shape_gap`）。

7. **Related PRs 小节**：`Related PRs：按 patterns/isa_extension_specific_instruction_substitution.md §Related PRs 本地表，Go 条目 5 条 URL：3659b8756a2b、a6ecdf29e34d、golang/go#59488、63ab68ddc5f1、1951afc9193f；同表另含 OpenCV（a00818047ff5 等 3+1）、OpenSSL（03ce37e11729 等 10）、Linux Kernel RISC-V（e8620bd7e5e0 等 7）、OpenJDK（6b89954c6534 等 20+）、V8（6f100865663f 等 20+）、QEMU（3de1fb712a07 等 2）、LLVM（#170824 等 8）——按本地表原文引用，不联网补找。`

## Phase 5 — Verification forecast / 验证预测：runtime.nextFreeFast

- **应消失/缩小侧（锚定 Phase 3(a) 引用行）**：`2b2f6: neg a1,s1`、`2b304: mul a1,a1,a2`（ctz 区间 33.33% 行）以及 `2b32a: slli a4,a2,0x30` / `2b32e: srli a4,a4,0x30`、`2b370–2b374` zext 对。以 `GORISCV64=rva22u64` 重建并对同一函数重跑 annotate 后，这些行应消失或明显缩小，出现单条 `ctz` 与 `zext.h`。
- **应出现侧（锚定 pattern §Verification）**：按 `isa_extension_specific_instruction_substitution.md` §Verification —— `go tool objdump -S` 确认 rva22u64 构建含 `ctz`/`zext.h`（「Verify instruction disassembly … confirm that … sign/zero extensions become `sext.b`, `sext.h`, `zexth`」），rva20u64 回退形态重现；边界条件「`ctz(0)` … semantics must match the language or runtime helper」——对 `TrailingZeros64(0)=64`、`1<<63`、全 1、交替 pattern 跑 `math/bits`/`runtime` 测试，rva20u64 与 rva22u64 两路径结果逐位一致；跑 `go test math/bits runtime cmd/compile/internal/ssa`。
- **把结论升级到 profile-backed 的补采数据**（模式 A 已有 profile，但样本与采样属性不足）：用 `perf record -e cycles:u -F 999`（或更高频率 + 确认 precise_ip/Exact-IP）重采该 benchmark，使函数样本数从 3 大幅提升；对 benchmark 本体 `runtime.test` 符号化后再核对分配路径；`perf stat -e cycles,instructions,branch-misses` 精化 bound type。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor |
|---|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅ | 1/1 组；[runtime.nextFreeFast]；`## Phase 3 — Pattern scan / 模式扫描：runtime.nextFreeFast`、`## Phase 4 — Root-cause blueprint / 根因蓝图：runtime.nextFreeFast`、`## Phase 5 — Verification forecast / 验证预测：runtime.nextFreeFast` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 行 baseline 表（Hardware ISA / Build ISA / Vector flavor / VLEN / Bound type / Sampling semantics / Sampling IP precision）；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`；含 Sampling IP precision 行；两个 L0 gate 判定 + bound-type gate 结论 |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`（rows-asm/operator-rvv/string-memory/vectorized-tuning/codegen/offload/crypto/runtime-os）；`Classes scanned: rows-codegen.md`；顶层 finding 1 个（ISA substitution，evidence 锚点 `2b2f6: neg a1,s1`、`2b304: mul a1,a1,a2`、`2b32a: slli a4,a2,0x30`）；supporting 1 条（runtime-ISA-dispatch）；排除条数 16+（含互斥邻居逐条观察）；推导式 2 条（route High / impact Low） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 pattern 文件：`patterns/isa_extension_specific_instruction_substitution.md`（+ `patterns/runtime_isa_specific_instruction_dispatch.md` supporting）；命中 row「RISC-V ISA Extension-Specific Instruction Substitution」；引用短语首词：`consumes extra decode, issue, and retirement slots`（§Why this is slow 1）、`ctz(0)`（§Verification）、`SLLI + SRLI`/`ZEXT.H`（替换表行）；`The fix` 含 before/after 伪代码、correctness（ctz(0)=64 语义）、风险（rva22u64 全量重建）、预期 Profile 信号；`Related PRs：Go 5 条 URL + 本地表其余项目原文引用` |
| 5 | 路径合规 | ✅ | 入口模式 A（profile_backed）；单 finding 按动态份额排序（ctz 区间 2/3 local）；supporting 不单独排序；evidence-mechanism layer：L4 认领 + L0 build-profile baseline finding 置顶 Phase 1；`th.v*` 未全局停扫（不适用，无 th.v*）；无 missing-`.S` 分支 |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧对 Phase 3(a) 引用行（`2b2f6: neg a1,s1`、`2b304: mul a1,a1,a2`、`2b32a-2b32e`、`2b370-2b374`）；出现侧标注 `isa_extension_specific_instruction_substitution.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问、无代码修改、无补丁生成、无契约外实施分支；无向用户追问；交付物止于 Profile 证据、根因蓝图、完整 `The fix` 与验证预测 |

修正记录：无。