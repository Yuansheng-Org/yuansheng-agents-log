Functions under analysis: [PinBuffer]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`PinBuffer`，`024-PinBuffer-annotate.txt`，152 samples，覆盖函数全部控制流：fast/slow path、CAS loop、BM_LOCKED/WaitBufHdrUnlocked 分支、epilogue）
- perf stat：已提供（`19-postgresql-pgbench-benchmark-riscv-select_only_clients_16.txt`：cpu_cycle / instruction / cache / branch / L1 / LLC counters）
- workload/binary/DSO/source context：已提供（`postgres` binary，annotate 头 `Disassembly of section .text` + 源码行号对应 `bufmgr.c`，含 DWARF 行信息）
- readelf -A（build ISA）：缺失（无独立 build_isa 文件；metadata JSON 的 `cpuinfo.isa` 与 `binaries` 字段提供目标 ISA 近似，详见 Phase 1）
- hardware ISA：已提供（metadata `cpuinfo.isa` = `rv64imafdcv_zicbom_zicboz_..._zve64x_..._svpbmt`；core = XuanTie C920v2，SG2044）
- `vlenb`：已提供（metadata `vector.vlenb` = 16 → VLEN 128 bits；本函数无向量指令）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock`，frequency=99，percent type=`local period`，单次运行窗口）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcv_...`，含 `a`（AMO/LR/SC）、`v`（RVV 1.0）；**无 `zacas`**（无单指令 CAS）、**无 `ztso`**（RVWMO） |
| Build ISA | 无独立 `readelf -A`；metadata `binaries` = "ELF64 little-endian PIE; Machine: RISC-V; RVC, double-float ABI"。`cpuinfo.isa` 作为 build 目标近似 → 部分 `baseline_gap: build ISA`（不影响本 finding，本 finding 不依赖向量或 optional 扩展） |
| Vector flavor | annotate 内全 scalar，零 `v*`、零 `th.v*`；函数为同步/原子控制流代码，与向量化无关 → 无 flavor mismatch |
| VLEN | 128 bits（`vlenb`=16）；本函数无向量使用，VLEN 不影响结论 |
| Bound type | IPC=0.431；branch_miss_rate=12.55%；L1_dcache_load_miss_rate=4.61%；LLC_load_miss_rate=3.35% → 非极端 memory-bound；低 IPC + 高原子/一致性 traffic → **coherence / contention-bound（原子同步主导）** |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=`local period`（**非 global-period**）；单次运行窗口；函数级 workload 贡献**未知** → 四条不齐，**不得**称 workload 级 Amdahl，只能表述为函数局部份额 |
| Sampling IP precision | `cpu-clock`@99Hz，无 `precise_ip`/Exact-IP 证据 → `baseline_gap: sampling IP precision`（样本 IP 有 skid，最高行只锚定 interval，不锚定单指令 latency/cycle cost） |

L0 baseline gate：hardware 有 `v` 但本函数为 scalar 同步代码，无 vectorization 关联、无 `th.v*`，不构成 mismatch；annotate 已提供 → 入口条件 A（profile_backed）。Bound-type gate：coherence/contention-bound 成立，与本 finding（原子/一致性开销）方向一致，不降低本 finding 的 route confidence。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致：`PinBuffer`（`src/backend/storage/buffer/bufmgr.c`）。

hot interval：`4716fe`（BufferDescriptorGetBuffer）→ `4717e4`（CAS loop 的 SC 失败回跳），覆盖 GetPrivateRefCountEntry 一入口缓存检查、两条 `pg_atomic_read_u64(&buf->state)` 调用、以及 refcount/usagecount 的 CAS loop。最高行：

- `42.11 :  471710: beq  a5,a0,471776`（`PrivateRefCountEntryLast != -1` 检查）
- `36.18 :  4717e4: bnez t1,4717d8`（CAS loop 的 SC 失败重试回跳）

Sampling IP precision：无 precise_ip，最高行只锚定所属 interval；但本 finding 的根因是**静态 codegen 事实**（把 no-barrier 的 64 位原子读错误降级为 LR/SC + seq_cst CAS-0 循环），由反汇编直接证明，不依赖单指令样本归因。

## Phase 3 — Pattern scan / 模式扫描：PinBuffer

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码来源是编译器生成的 C（`bufmgr.c`，GCC 输出：`auipc`+GOT、标准原子序列），无手写 `.S`/intrinsic kernel。
2. `rows-operator-rvv.md` — **exclude** — 无可向量化的算子/计算循环；纯同步/原子控制流，零 `v*`，不属 `no-vectorization`（缺失的向量化载体）语义。
3. `rows-string-memory.md` — **exclude** — 非 copy/fill/scan/compare/checksum/back-ref 内存循环。
4. `rows-vectorized-tuning.md` — **exclude** — annotate 内零 `v*`，无 RVV 配置/寄存器/展开/策略修正对象。
5. `rows-codegen.md` — **include（primary）** — 热点是编译器生成代码的**同步实现**（原子 LR/SC、CAS loop、自旋），本组承载 `hardware_atomic_operations_for_synchronization` / `bounded_spin_wait_backoff` / `redundant_synchronization_elimination` 等 rows。
6. `rows-offload.md` — **exclude** — 无矩阵引擎 / packed-SIMD / GEMM / 权重重排 / 跨 VLEN 可移植层。
7. `rows-crypto.md` — **exclude** — 无密码学原语。
8. `rows-runtime-os.md` — **exclude** — 用户态数据库，非 RTOS/kernel timer/ISR/PMP/CSR 记账热点。

`Classes scanned:` `rows-codegen.md`（仅此一个 class）。

### Local performance pattern scan: `PinBuffer`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Intrinsifying Atomic Operations for Hardware Synchronization | no-barrier 的 `pg_atomic_read_u64(&buf->state)` 被编译为 `lr.d.aqrl` + `sc.d.rl zero` CAS-0 循环（`471732`/`471738`、`47179e`/`4717a4`）；源码 `generic.h` 319-335 回退 + `atomics.h` 无 `__riscv` 分支 → `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY` 未定义 | High | Medium | `patterns/hardware_atomic_operations_for_synchronization.md` |

#### 三件套（唯一顶层 finding）

**(a) 逐字 evidence 引用**（source：`024-PinBuffer-annotate.txt`）：

fast path 读（source line 3378 `result = (pg_atomic_read_u64(&buf->state) & BM_VALID) != 0;`）：
```
     0.00 :   471732: lr.d.aqrl       a5,(a4)
     0.00 :   471736: bnez    a5,47173e <PinBuffer+0x4a>
     0.00 :   471738: sc.d.rl a2,zero,(a4)
     0.00 :   47173c: bnez    a2,471732 <PinBuffer+0x3e>
```
slow path 读（source line 3312 `old_buf_state = pg_atomic_read_u64(&buf->state);`）：
```
     0.00 :   47179e: lr.d.aqrl       a0,(a1)
     1.32 :   4717a2: bnez    a0,4717aa <PinBuffer+0xb6>
     0.00 :   4717a4: sc.d.rl a5,zero,(a1)
     0.00 :   4717a8: bnez    a5,47179e <PinBuffer+0xaa>
```
同一原子区间内的热点样本（skid 载体，锚定 interval）：
```
    42.11 :   471710: beq     a5,a0,471776 <PinBuffer+0x82>
    36.18 :   4717e4: bnez    t1,4717d8 <PinBuffer+0xe4>
```
另：真正的 refcount CAS loop（source line 3351，seq_cst，正确地保留了 LR/SC）：
```
     5.92 :   4717dc: bne     a6,a0,4717e8 <PinBuffer+0xf4>
     0.00 :   4717e0: sc.d.rl t1,a5,(a1)
    36.18 :   4717e4: bnez    t1,4717d8 <PinBuffer+0xe4>
```

**(b) 互斥邻居排除**：

- `bounded_spin_wait_backoff`（rows-codegen.md row）：排除 —— 本 finding 不是"等待另一执行单元推进状态的紧密自旋"；`4717d8–4717e4` 是 lock-free RMW 的 CAS 重试（SC 失败因 16 后端并发改 `buf->state` 导致 cache-line 竞争），重试是算法固有成本，加入退避只会推迟重试、不降低竞争，且无法证明"producer/退出条件"语义。该行互斥判据明确要求 spin-wait 语义，本函数不具备。
- `redundant_synchronization_elimination_in_uncontended_contexts`（rows-codegen.md row）：排除 —— `buf->state` 存在 **16 后端真实并发写者**，同步是必要的；该行只有"可证明无并发写者"才命中，此处不成立（与 `hardware_atomic` 行方向相反）。
- `no-vectorization` / `rows-operator-rvv`：排除 —— 无 vectorizable compute loop，函数为同步控制流，不属于"未向量化的算子语义循环"。
- 本行内互斥"普通 lock algorithm/cache contention 不命中本 row"：本 finding 命中的**不是** contention 本身，而是"no-barrier 原子读被错误降级为 `lr.d.aqrl`+`sc.d.rl zero` 的 seq_cst CAS-0 序列"这一 lowering 缺陷（对应本行 disjunct "IR 只要求更弱序，generated code 却用 seq_cst helper"）。CAS loop 的 36% 竞争因此作为**背景/negative-evidence** 呈现，不单独成顶层 finding。

**(c) 双 Confidence 推导式**：

- route: current source = GCC 生成标量代码（`bufmgr.c`，`generic-gcc.h` 的 `__atomic_compare_exchange_n`）+ 反汇编直接显示 `lr.d.aqrl`+`sc.d.rl zero` + 源码证明 `generic.h` 319-335 回退与 `atomics.h` 66-72 无 `__riscv` 分支 → **High**（直接证据，无相邻未消解候选）。
- impact: 存在竞争 bottleneck（CAS loop 36% SC 重试，不被本 fix 消除）；percent type=local period（非 global-period）且函数 workload 贡献未知；read fallback 自身 0.00% 直接样本（skid，成本通过邻近分支样本间接体现）→ **Medium**（方向明确，幅度无法精确量化）。

## Phase 4 — Root-cause blueprint / 根因蓝图：PinBuffer

命中 row（Phase 3 通过 route gate）：`Intrinsifying Atomic Operations for Hardware Synchronization` → `patterns/hardware_atomic_operations_for_synchronization.md`。

1. **Root cause**：PostgreSQL 在 RISC-V 上没有定义 `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY`（`atomics.h` 的 arch 头选择分支第 66-72 行只有 arm/x86/ppc，无 `__riscv` 分支，也没有 `arch-riscv.h`），导致 `generic.h` 的 `pg_atomic_read_u64_impl` 走 `#else` 回退分支（第 319-335 行）：把一次"无屏障语义"的 64 位读实现为 `pg_atomic_compare_exchange_u64_impl(ptr, &old=0, 0)`，即 `__atomic_compare_exchange_n(..., __ATOMIC_SEQ_CST, __ATOMIC_SEQ_CST)`。该序列编译为 `lr.d.aqrl` + `bnez` + `sc.d.rl zero` + 重试回跳。依据 `patterns/hardware_atomic_operations_for_synchronization.md` §"Typical Atomic Intrinsics Table" 第 1 行——"对纯 load 使用 `SC` 是错误的……**不得使用 `LR.D`/`SC.D`**，自然对齐的 doubleword load 在 RV64 上本身原子"——这条 LR/SC 序列对纯读是过度实现：`lr.d.aqrl` 会在共享的 `buf->state` cache line 上建立 reservation（需独占/保留态，产生一致性流量），且 `.aqrl` 施加 acquire+release 全屏障，与读操作"无屏障"的合约相悖。在 16 后端并发 pin/unpin 同一共享缓冲头时，这些多余的 `lr.d.aqrl` 与 refcount CAS loop 一起放大 cache-line 竞争与一致性 stall。

2. **The fix / 修复方式**：纠正对象 = RISC-V 的 `pg_atomic_read_u64_impl` 实现选择；操作方式 = 让"RV64 对齐 8 字节 load 单拷贝原子"这一能力被声明，使读走 `generic.h` 第 310-317 行的 `return ptr->value;`（单条 `ld`）快速路径。

   修复前（现状，RISC-V 走 `generic.h` 319-335 回退）：
   ```c
   static inline uint64
   pg_atomic_read_u64_impl(volatile pg_atomic_uint64 *ptr)
   {
       uint64 old = 0;
       /* 64-bit reads aren't atomic on all platforms → 回退为 CAS-with-0 */
       pg_atomic_compare_exchange_u64_impl(ptr, &old, 0);
       return old;
   }
   /* 编译为: lr.d.aqrl; bnez; sc.d.rl zero; bnez(重试) */
   ```
   修复后（新增 RISC-V arch 头，命中 `generic.h` 310-317 快速路径）：
   ```c
   /* src/include/port/atomics/arch-riscv.h（新增） */
   #if __riscv_xlen == 64
   #define PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY
   #endif
   ```
   ```c
   /* atomics.h 第 66-72 行后新增分支 */
   #elif defined(__riscv)
   #include "port/atomics/arch-riscv.h"
   ```
   从而使：
   ```c
   static inline uint64
   pg_atomic_read_u64_impl(volatile pg_atomic_uint64 *ptr)
   {
       AssertPointerAlignment(ptr, 8);
       return ptr->value;   /* 单条 ld；RV64 对齐 8B load 单拷贝原子、无屏障 */
   }
   ```

   适用前提：RV64（`__riscv_xlen == 64`）。不可破坏的 correctness contract：(i) 保持 `pg_atomic_read_u64` 的"无屏障语义"合约不变；(ii) 保持单拷贝原子性——RV64 基 ISA（含目标已有的 `a` 扩展）保证自然对齐 8 字节 ld/sd 单拷贝原子；(iii) **不得**改动 `pg_atomic_compare_exchange_u64` 的 seq_cst CAS 路径（refcount CAS loop 仍需 seq_cst）；(iv) `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY` 必须只在 `__riscv_xlen == 64` 下定义（RV32 8 字节 load 不保证单拷贝原子，仍须回退）。限制/风险：本修复会同时影响 PostgreSQL 全库所有 `pg_atomic_read_u64` 调用点（如 `UnpinBuffer` 等），需以并发/内存模型回归覆盖；修复只消除"读"的过度实现，CAS loop 的 16 后端 cache-line 竞争仍存在。

   预期变化的 Profile signals：`471732`/`47179e` 处 `lr.d.aqrl` 消失，被单条 `ld` 取代；`sc.d.rl zero` 与相应 `bnez` 消失；`buf->state` cache-line 的 reservation 流量下降，CAS loop 的 SC 失败率随之下降。

3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_...`（含 `a`、`v`，无 `zacas`/`ztso`）；build ISA=无独立 `readelf -A`（metadata `cpuinfo.isa` 近似，部分 gap）；VLEN=128 bits（本函数无关）；bound type=coherence/contention-bound（IPC 0.431）。

4. **收益上界**：当前 sampled event（`cpu-clock`）下的**函数局部样本份额**；本 finding 的 read fallback 指令自身 0.00% 直接样本（skid），但所在原子区间（`4716fe`–`4717e4`）局部份额合计约 78%（`471710` 42.11% + `4717e4` 36.18% + 其余分散）。该上界是方向性上限：其中 CAS loop 的 36% 是独立的固有竞争成本，不被本 fix 消除；实际收益低于 78%。因 percent type=local period（非 global-period）且函数 workload 贡献未知，**不能**表述为 workload 级 Amdahl 上界。`dynamic priority`：入口条件 A 但 sampling 语义四条不齐，本 finding 与（不成立的）contention finding 不排序。

5. **三维路由判定**：
   - current source：GCC 生成的标量 C 代码（`bufmgr.c`，经 `generic-gcc.h` 的 `__atomic` intrinsic），非手写 `.S`、非 JIT。
   - implementation existence/reachability：正确的快速路径**已存在**（`generic.h` 310-317 `return ptr->value;`），只是因 `PG_HAVE_8BYTE_SINGLE_COPY_ATOMICITY` 未定义而不可达；修复是让现有正确实现可达，非新建 kernel。
   - function-level policy：无独立 `.S` 政策；不进入 missing-assembly 分支。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`，无需 VL/LMUL/tail 等形状证明）。

7. **Related PRs 小节**（来源：`patterns/hardware_atomic_operations_for_synchronization.md` §Related PRs）：
   - Go：`42c25e65f321`、`497feff168b4`、`ad61343f886c`、`#36765`
   - Linux Kernel RISC-V：`1c0196b878a6`、`de39d2c4cdb6`
   - V8：`e8fa4b9c7e6c`

   Related PRs：8 条 URL。

## Phase 5 — Verification forecast / 验证预测：PinBuffer

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`471732: lr.d.aqrl a5,(a4)` 与 `47179e: lr.d.aqrl a0,(a1)` 处的 `lr.d.aqrl` 应消失、被单条 `ld` 取代；`471738: sc.d.rl a2,zero,(a4)`、`4717a4: sc.d.rl a5,zero,(a1)` 及各自 `bnez` 重试回跳应消失。对应 interval 的局部样本份额（`471710` beq、`4717e4` bnez 周边的 skid 样本）应明显下降。
- **应出现侧**（锚定 `patterns/hardware_atomic_operations_for_synchronization.md` §Verification）：反汇编中 `pg_atomic_read_u64` 调用点只出现普通 `ld`，**不得出现 `sc.d`**（负向检查：纯 load 若出现 `sc` 即误降为 read-modify-write，会写入目标地址）；seq_cst CAS loop（`4717d8–4717e4`）的 `lr.d.aqrl`/`sc.d.rl` 保持不变；`perf stat -e instructions,cycles` 显示原子区间指令数与一致性 stall 下降。
- **重建与实测**：以相同 `-march`/优化级别重建；对同一 `pgbench select-only`（16 clients）重跑 annotate + perf stat，观察 PinBuffer 局部样本与 IPC 变化；覆盖 RV32 构建确认仍走 CAS-with-0 回退（`__riscv_xlen != 64` 时不定义宏）。
- **正确性回归**：`make check` 的 buffer/lmgr 相关测试 + 并发/内存模型回归（`pg_atomic` 语义 + buffer refcount 正确性），确认把 no-barrier 读降为 `ld` 未破坏任何调用点的可见性/排序依赖。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | 1/1 组；`PinBuffer` |
| 2 | Phase 1 输出要求满足 | ✅ | 7 项 baseline 结论（Hardware/Build/Flavor/VLEN/Bound/Sampling semantics/Sampling IP precision）；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision` |
| 3 | Phase 3 输出要求满足 | ✅ | 8 项 `Class selection trace`；`Classes scanned: rows-codegen.md`；顶层 finding 数=1；evidence 锚点 `471732 lr.d.aqrl`、`471738 sc.d.rl zero`、`47179e lr.d.aqrl`、`471710 beq`、`4717e4 bnez`；supporting=0；排除条数=3（bounded_spin / redundant_sync / no-vectorization）；推导式 2 条（route High / impact Medium） |
| 4 | Phase 4 输出要求满足 | ✅ | 已读 `patterns/hardware_atomic_operations_for_synchronization.md`；命中 row `Intrinsifying Atomic Operations for Hardware Synchronization`；引用短语 "不得使用 LR.D/SC.D"、"自然对齐的 doubleword load 在 RV64 上本身原子"；`The fix` 含 before/after（`lr.d.aqrl`→`ld`）、correctness（无屏障合约+RV64 单拷贝原子+不改 seq_cst）、风险（全库 `pg_atomic_read_u64` 调用点+RV32 回退）；Related PRs：8 条 URL |
| 5 | 路径合规 | ✅ | 入口模式 A（profile_backed）；8 项 trace 可解释；唯一 blueprint leaf 来自通过 gate 的 row；按动态份额排序（single finding，无排序问题）；`th.v*` 未停扫（无 `th.v*`，且同步代码与向量无关） |
| 6 | Phase 5 两侧锚定 | ✅ | 消失侧对上 `471732/471738/47179e/4717a4`；出现侧标注 `patterns/hardware_atomic_operations_for_synchronization.md` §Verification |
| 7 | 契约边界合规 | ✅ | 无实施询问/代码修改/补丁生成；无向用户追问；交付物止于证据、蓝图、`The fix`、验证预测 |

修正记录：无
