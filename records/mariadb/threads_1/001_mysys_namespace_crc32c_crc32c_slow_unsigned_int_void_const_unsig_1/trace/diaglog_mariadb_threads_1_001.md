**Functions under analysis: [mysys_namespace::crc32c::crc32c_slow(unsigned int, void const*, unsigned long)]（1 个）→ 本输出含 1 组 Phase 3–5**

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-mysys_namespace：：crc32c：：crc32c_slow(...)-annotate.txt`，37 samples，含 16-byte 主循环 hot loop body，覆盖完整）
- perf stat（bound/context）：已提供（`20-mariadb-sysbench-benchmark-riscv-threads_1.txt`，IPC/LLC/L1/branch counters 齐全）
- workload/binary/DSO/source context：已提供（mariadb-sysbench `mariadbd`，metadata 含 commit 02c842c30dcb05a962eaac58c69920e6be3369cf、ELF header）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `mariadbd-elf-A`，`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`）
- hardware ISA（`/proc/cpuinfo` / hwprobe）：已提供（metadata cpuinfo：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`）
- `vlenb`：已提供（metadata vector：vlen_bits=128，vlenb=16）
- 采样元数据（event / percent type / scope / 窗口）：部分提供（annotate header：`cpu-clock (37 samples, percent: local period)`，单函数窗口；函数级 workload 贡献未知 → 见 Phase 1 采样语义 gate）
- Sampling IP precision：缺失（未提供 `precise_ip`/Exact-IP/skid 信息 → `baseline_gap: sampling IP precision`，详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`v`（RVV 1.0）、`zba`、`zbb`、`zbc`、`zbs`、`zfa`、`zfh`、`zcb/zcd`；**无** `zk*`、`zvk*`、`zvbb`、`zvbc`、`zbkb` |
| Build ISA | 已提供：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0`——**无 `v`，也无 `zbc/zbb/zba/zbkb`** |
| Vector flavor | annotate 内 zero `v*` 与 zero `th.v*`（全 scalar）；无 flavor mismatch（无 `th.v*` 证据） |
| VLEN | 已提供：`vlenb=16` → VLEN=128 bits |
| Bound type | perf stat：IPC=0.570491；L1_dcache_load_miss_rate=2.182%；LLC_load_miss_rate=11.527%；branch_miss_rate=5.559% → **latency-bound**（查表 load-use + 串行 CRC 状态依赖链），非 memory-bandwidth-bound；查表工作集（4×1KB 表 + reversal 表）基本驻留 L1 |
| Sampling semantics | event=`cpu-clock`；percent type=**local period**；同一运行窗口（单函数）；函数级 workload 贡献未知 → 不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（无 `precise_ip`/skid 信息）→ 单行占比只锚定 basic block / loop interval，不承担单指令 latency 归因 |

L0 baseline gate 判定：
- **hardware 有 `v` 而 build 无 `v`：成立（最高优先级 baseline finding）**——mariadbd 按无 `v` 的 rv64gc 形态构建，任何 RVV 路径在构建层面不可达。
- **hardware 有 `zbc/zbb/zba/zbs` 而 build 无这些扩展：同样成立**——`clmul`/`clmulh`（Zbc）与 `zext.w`/`rev8`（Zbb）在当前二进制中不可达；这是本函数 carry-less 修复的构建层前置障碍。
- `th.v*` flavor gate：不适用（annotate 无 `th.v*`）。
- Bound-type gate：latency-bound 成立（IPC 0.57）→ 本地计算形态修复（clmul folding）**不**因 memory-bound 下调 performance-impact；相反，依赖链缩短正是本修复的主要收益机制。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个）。hot loop 锚点与边界：

- **16-byte 主循环**：`ffa47a`–`ffa61c`（`while ((e-p) >= 16)`，每轮 4 次内联 `Slow_CRC32`，每次处理 4 字节：4×lbu+4×sb+1×lw 物化 + 4 次 256-entry 表查表 + 串行 XOR 链）。
- 最高占比行（全部位于 16-byte 主循环区间）：
  - `18.92 :  ffa5b6: andi a4,a4,1020`（table1_ 索引 mask）
  - `13.51 :  ffa4e8: add a7,a7,a5`（table2_ 索引地址合成）
  - `8.11 :  ffa49e: lbu t0,4(a1)`（uint4korr 逐字节物化的 byte load）
- 次级循环：8-byte 循环 `ffa636`–`ffa70c`、byte tail `ffa71c`–`ffa73a`、cold setup/prologue（`ffa40e`–`ffa45c`，仅函数前 0.00–2.70% 样本）。
- Sampling IP precision 未确认 → 所有行只作为 loop interval 锚点，不做单指令 cycle 归因。
- annotate 覆盖完整（cold + 主循环 + tail 均在），不走入口条件 B/C/D，入口模式 **A（profile_backed）**。

## Phase 3 — Pattern scan / 模式扫描：mysys_namespace::crc32c::crc32c_slow

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude** — 当前代码来源为 compiler-generated C++（annotate 为普通 scalar 指令序列，无 `.S` source/DWARF/object-mapping 证据）；无 dispatch-slot 与 policy/existence 四证
2. `rows-operator-rvv.md` — **include** — 代码来源为 compiler-generated scalar loop，须显式排除 generic `no-vectorization`（其行内互斥把 crypto/checksum 语义路由到 crypto/string-memory 更具体 row）
3. `rows-string-memory.md` — **include** — CRC32C 属 checksum 语义（class 索引"何时扫描"列出 checksum）；评估 SWAR / remainder / wide-memory / alignment 行
4. `rows-vectorized-tuning.md` — **exclude** — annotate 内 zero `v*` 且非手写 RVV；该类修正对象是已有 `v*` 的配置/寄存器/policy
5. `rows-codegen.md` — **include** — compiler-generated 指令形态证据：uint4korr 的 byte-load→stack→word-load 物化、`slli+srli` zero-extend 对、表寻址 offset 折叠；含 kernel-selection/cache-blocking cross-cutting rows
6. `rows-offload.md` — **exclude** — 无矩阵引擎/packed-SIMD 证据
7. `rows-crypto.md` — **include** — class 索引"何时扫描"显式点名 CRC；HW crypto baseline：`zbc` 有、`zk*`/`zvk*`/`zvbb`/`zvbc`/`zbkb` 无
8. `rows-runtime-os.md` — **exclude** — 用户态 mariadbd 热点，非 RTOS/kernel timer/ISR/CSR 路径

### Classes scanned: rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md, rows-crypto.md

### Local performance pattern scan: `mysys_namespace::crc32c::crc32c_slow`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Carry-less Multiplication for Binary-Field Arithmetic（**primary**） | 16-byte 主循环每 4 字节 4 次表查表（`lw` at ffa4d6/ffa4ee/ffa4f8/ffa4c6 等）+ 串行 shift/xor 依赖链；HW isa 含 `zbc`；整个 annotate zero `clmul`；build `Tag_RISCV_arch` 无 `zbc` | High | Medium | `patterns/carry_less_multiplication_for_binary_field_arithmetic.md` |
| Wide Scalar Memory-Access Code Generation（supporting） | `ffa47a`–`ffa49a`：`uint4korr` 的 4 字节 `memcpy` 降为 4×lbu+4×sb（栈）+1×lw，而指针已被 ALIGN(4) 保证 4 对齐、块内不越界 | — | — | `patterns/wide_scalar_memory_access_codegen.md` |

**顶层 finding 三件套（primary：Carry-less Multiplication for Binary-Field Arithmetic）**

**(a) 逐字 evidence 引用**（均属 16-byte 主循环 interval `ffa47a`–`ffa61c`）：
- `13.51 :  ffa4e8: add a7,a7,a5`（table2_ 索引地址合成）
- `18.92 :  ffa5b6: andi a4,a4,1020`（table1_ 索引 mask）
- `8.11 :  ffa49e: lbu t0,4(a1)`（uint4korr 逐字节物化）
- 表查表 load：`ffa4c6: lw t4,0(a0)`、`ffa4d6: lw t1,-1024(t1)`、`ffa4ee: lw t3,1024(a7)`、`ffa4f8: lw a4,-2048(a4)`——每 4 字节 4 次 256-entry 查表
- 全函数 37 样本全部落在表驱动循环区间；zero `clmul`/`vclmul` opcode

**(b) 互斥邻居排除**：
- Precomputed Lookup Table Reuse for Polynomial Kernels（`rows-crypto.md` 行 4）：该行要求 table ownership/reuse 证据（重复私有 256-entry 表、每调用点重建、wrapper state-ordering 不匹配）——本函数 `table0_`/`table1_`/`table2_`/`table3_` 为进程级 static 全局表，无重建/复制证据，行内明文"单纯'CRC32 很热'不命中" → 排除。
- Dedicated RISC-V Vector Crypto Instructions（`rows-crypto.md` 行 1）：要求 HW 暴露 `Zvkned/Zvknha/Zvksh/Zvksed/Zvkg`——metadata cpuinfo 无任何 `zvk*` → 准入 gate 不成立。
- No vectorization（`rows-operator-rvv.md` 行 30）：行内互斥"crypto block-mode → 各自更具体 row"；CRC/GF(2) 由 carry-less row 认领 → 排除。
- Scalar SWAR Checksum Reduction（`rows-string-memory.md` 行 1）：行内互斥"CRC/GHASH/GF(2) → carry-less row 或 polynomial row" → 排除。
- ISA Extension-Specific Instruction Substitution（`rows-codegen.md` 行 31）：pattern §Boundary 明示"与二元域乘法无关的通用标量单指令替换"才归该行；本热点核心是 GF(2^k) 乘法被查表模拟，专属 carry-less 情形 → 排除；`slli+srli` zero-extend（ffa41e/ffa426、ffa610/ffa618、ffa700/ffa708）为 build 缺 Zbb 下的 baseline codegen，归入 L0 build-ISA finding，不作为独立顶层 finding。

**(c) 双 Confidence 推导式**：
- `route: HW zbc 直接证据（metadata cpuinfo）+ 查表实现直接证据（annotate 行）+ 语义合同（CRC32C=GF(2) 反射多项式归约）→ High`
- `impact: sample share=函数内 37/37 局部（采样语义四条件不成立：local period、函数级贡献未知，不得称 workload 级）+ VLEN=128 已知 + bound=latency → Medium`（Impact evidence 缺全局采样语义，禁止 High）

**supporting evidence 行（Wide Scalar Memory-Access Code Generation）**：
(a) 逐字引用：`ffa47a: lbu t3,0(a1)`、`ffa48a: sb t3,-20(s0)`、`ffa492: sb a0,-18(s0)`、`ffa49a: lw a4,-20(s0)`——4 字节 `memcpy`（`uint4korr`）被降为 4×lbu+4×sb 栈往返 + 1×lw，共 9 次访存；指针已由 `ALIGN(pval,4)`（ffa416–ffa424）保证 4 对齐，16-byte/8-byte 循环守卫 `(e-p)>=16/8` 保证块内不越界，LE 目标（`my_letoh32` 恒等）。
supporting because: 与 primary 同属 16-byte 主循环同一机制链（每 4 字节的输入物化开销）；carry-less 主修复引入的新 folding 路径会自然消除该 byte 拼装信号，故并入 supporting。

**多命中仲裁小段**：1 个顶层 primary（Carry-less Multiplication，evidence-mechanism layer **L1 vectorization/semantic dispatch**）+ 1 个 supporting（Wide Scalar Memory-Access，layer **L2 data movement**）。因果消除测试：L1 算法修复（clmul folding 取代查表）会自然消除 L2 的 byte 拼装信号 → L2 并入 supporting，不另列顶层 finding。入口条件 A：顶层 finding 的 evidence sample share = 37/37（函数内局部份额）；同一调用链、区间重叠，不与 supporting 叠加计算。收益上界只能表述为"当前 sampled event（cpu-clock local period）下的局部样本份额"，**不**称 workload 级（Phase 1 采样语义 gate）。

## Phase 4 — Root-cause blueprint / 根因蓝图：mysys_namespace::crc32c::crc32c_slow

**命中 row**：`Carry-less Multiplication for Binary-Field Arithmetic`（`triggers/rows-crypto.md` 行 3，Phase 3 通过 gate）；supporting leaf `Wide Scalar Memory-Access Code Generation`（`triggers/rows-string-memory.md` 行 10）。

1. **Root cause**：CRC32C 归约是 GF(2) 域（无进位）多项式运算，当前 `crc32c_slow` 以 slicing-by-4 查表实现——每 4 字节执行 4 次 256-entry 表查表（table0_–table3_）+ 串行 XOR 依赖链；而目标核 C920v2 的硬件 ISA 暴露 `Zbc`（`clmul`/`clmulh`），构建产物 `Tag_RISCV_arch` 却不含 `zbc`（也无 `v`），原生 carry-less 乘法从未到达。关键机制句（短引）："on an out-of-order core such as SG2044 / T-Head C920v2, the architectural dependency remains and the lookup table still consumes D-cache and issue bandwidth"（依据 `patterns/carry_less_multiplication_for_binary_field_arithmetic.md` §Why this is slow 第 1 点）；以及 "The root cause is a carry-less field multiply implemented as a table walk while the hardware exposes `clmul`/`vclmul`"（§The fix 首句）。IPC=0.57 + 查表 load-use 依赖链与 OoO 发射带宽占用一致。

2. **The fix / 修复方式**：用 Zbc carry-less folding 取代表驱动主循环（对标该 pattern §Leverage point 2/4 的 CRC/RS folding shape 与 Linux kernel RISC-V Zbc CRC 提交形态）：

```
// Before（当前形态，每 4 字节）：
uint32_t c = (uint32_t)l ^ le32toh(load32(p));      // 实际被降为 4×lbu+4×sb+lw
l = table3_[c & 0xff] ^ table2_[(c>>8) & 0xff] ^
    table1_[(c>>16) & 0xff] ^ table0_[c >> 24];      // 4 次 256-entry 查表 + 串行 XOR

// After（Zbc clmul folding，每 8 字节块；shape 参照 pattern §Leverage point 2 的
//   clmul/clmulh 归约序列与 §Leverage point 4 的 CRC folding 循环）：
//   以 clmul/clmulh 对 8 字节消息块按 CRC32C 反射多项式做 GF(2) 折叠，
//   再用 clmul 两步归约折回 32-bit CRC 状态；tail（0..7 字节）保留 byte 查表路径；
//   Zbb rev8 处理反射位序；短输入直接走查表路径（低于 folding crossover）。
for (; len >= 8; p += 8, len -= 8)
    fold_block_crc32c(&crc, p, k_poly_const);        // clmul/clmulh + xor + 归约
```

- **适用前提**：rebuild 时 `-march` 加入 `zbc`（建议同时 `zbb` 用于 `rev8`/`zext.w`），或用 `riscv_hwprobe` 检测 Zbc 后经函数指针/ifunc 派发（pattern §Leverage point 1 的 dispatch shape）；当前二进制 build 无 `zbc`，不改构建不可达。
- **correctness contract**：CRC32C 的 initial/final XOR（0xffffffff）、反射顺序（reflected polynomial 0x1EDC6F41）、字节序、短输入与 tail 语义必须与标量参考逐字节一致；所有 KAT 向量在 scalar 与 clmul 两条路径上 bit-exact 等价（pattern §Verification）。
- **限制/风险**：CRC32C 是反射多项式，folding 常量与归约顺序错一位即全错；build 缺 Zbc 时若以 `.word` 硬编码 clmul（pattern §Leverage point 1 的 perlasm 形态）需保持可移植；向量路径（Zvbc）在本机**不可用**（HW 无 `zvbc`），不列入本蓝图。
- **修复后预期 Profile signals**：主循环出现 `clmul`/`clmulh`/`rev8`；表查表 `lw` 与 byte 拼装消失或显著缩小；`objdump -d` 确认 clmul opcode 被发射；依赖链缩短在 OoO 核上反映为 IPC 上升。
- **supporting（Wide Scalar Memory-Access）的修复对象**：即使保留查表路径，`uint4korr` 的 4 字节物化也应让编译器发出单条 `lwu`（对齐 + 不越界 + LE 均合法）——修复方式是给 compiler 一个合法可识别的单次访问合同（集中式 fixed-width helper / 按值快照，pattern §The fix）；在 clmul 主修复落地后该信号自然消失。

3. **Baseline facts 回填**：hardware ISA=已提供（v、zba、zbb、zbc、zbs，无 zvk*/zvbc/zvbb/zbkb）；build ISA=已提供（无 v/zbc/zbb/zba/zbkb——L0 mismatch）；VLEN=128 bits（vlenb=16）；bound type=latency-bound（IPC 0.570491、L1 miss 2.18%、LLC miss 11.53%）。

4. **收益上界**：入口模式 A——「当前 sampled event（cpu-clock，local period）下的局部样本份额：37/37（函数内 ≈100%，含 16-byte 主循环、8-byte 循环与 tail 全部落在表驱动区间）」。Phase 1 采样语义四条件不成立（percent=local period、函数级 workload 贡献未知）→ **不得**称 workload 级 Amdahl 上界；该函数为本批次 overhead 排名 001（21 个热点之首），但排名不换算为收益百分比。

5. **三维路由判定**：
   - `current source`：compiler-generated C++（annotate 全 scalar 指令序列、无 `.S` provenance）→ 不属手写 assembly 类。
   - `implementation existence/reachability`：annotate 内 zero `clmul` opcode；build `Tag_RISCV_arch` 无 `zbc` → 即使存在 clmul 源码路径在当前二进制也不可达；HW 侧 `zbc` 已确认存在。
   - `function-level policy`：`crc32c_slow` 是静态函数、无 dispatch-slot/独立 `.S` policy 证据 → **不**走 policy-backed missing `.S` 分支；修复形态为源码/构建层面的 clmul 路径引入（可配 runtime dispatch）。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs 小节**：
- Carry-less Multiplication for Binary-Field Arithmetic：`Related PRs：9 条 URL`（OpenSSL [003f5698146b](https://github.com/openssl/openssl/commit/003f569814)、[999376dcf339](https://github.com/openssl/openssl/commit/999376dcf3)、[b24684369b76](https://github.com/openssl/openssl/commit/b24684369b)；Linux Kernel RISC-V [ee6740fd34eb](https://github.com/torvalds/linux/commit/ee6740fd34eb53c5c76be01201c15310f461b69f)、[72acff5f8185](https://github.com/torvalds/linux/commit/72acff5f81851fe0858d2430b35b4b08f8f27a72)、[bbe2610bc5ad](https://github.com/torvalds/linux/commit/bbe2610bc5ada51418a4191e799cfb4577302a31)；OpenJDK [2f4f6cc34c10](https://github.com/openjdk/jdk/commit/2f4f6cc34c10c5519c74abbce8d1715013b50d5d)、[c517ffba7d93](https://github.com/openjdk/jdk/commit/c517ffba7d9388e75b5d7bba77e565e71c0a7d76)、[#22475](https://github.com/openjdk/jdk/pull/22475)）
- Wide Scalar Memory-Access Code Generation（supporting）：`Related PRs：2 条 URL`（zlib-ng [d7e121e56b64](https://github.com/zlib-ng/zlib-ng/commit/d7e121e56b64b5916810cf32615062a53f954773)、[7a859e8cc350](https://github.com/zlib-ng/zlib-ng/commit/7a859e8cc350f3983236bd67522b5e7c2acec85a)）

## Phase 5 — Verification forecast / 验证预测：mysys_namespace::crc32c::crc32c_slow

**primary（Carry-less Multiplication，收益上界顺序第一）**：
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`ffa4e8: add a7,a7,a5`（13.51%）、`ffa5b6: andi a4,a4,1020`（18.92%）、`ffa49e: lbu t0,4(a1)`（8.11%）及 4 处表查表 `lw`（ffa4c6/ffa4d6/ffa4ee/ffa4f8）应消失或显著缩小。
- 应出现侧（锚定 pattern §Verification）：同一 annotate 热循环出现 `clmul`/`clmulh`（标量 Zbc 路径）与 `rev8`（Zbb），`objdump -d` 确认 carry-less opcode 发射并被派发；CRC32C KAT 向量在 clmul 与查表路径逐字节一致；初始值/反射/多项式/final xor/短输入 fallback/长缓冲区 fold 结果与标量 reference 对齐；大缓冲区（8192/16384 字节）bytes/s 相对查表路径增长，同时功能测试无回归。**验证前置**：以含 `zbc`（+`zbb`）的 `-march` 重建 mariadbd（对应 Phase 1 L0 build-ISA finding）。
- supporting（Wide Scalar Memory-Access）不单独验证，跟随 primary 预测；若仅做 supporting 级修复（保留查表路径、仅改物化），预期 `ffa47a`–`ffa49a` 的 4×lbu+4×sb+lw 收敛为单条 `lwu`。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5 全部出现（载荷：`1/1 组；mysys_namespace::crc32c::crc32c_slow`） | ✅ |
| 2 | Phase 1 输出要求满足（载荷：7 行 baseline 结论；gap 标签：`baseline_gap: sampling IP precision`；含 Sampling IP precision 行） | ✅ |
| 3 | Phase 3 输出要求满足（载荷：8 项 `Class selection trace`；`Classes scanned: rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md, rows-crypto.md`；顶层 finding=1（carry-less，锚点 `ffa4e8: add a7,a7,a5`、`ffa5b6: andi a4,a4,1020`、`ffa49e: lbu t0,4(a1)`）；supporting=1；排除条数=5（polynomial-table、dedicated-vector-crypto、no-vectorization、scalar-SWAR、ISA-substitution）；推导式=1 条） | ✅ |
| 4 | Phase 4 输出要求满足（载荷：已读 pattern 文件名列表 + 命中 row + 引用短语首词：`carry_less_multiplication_for_binary_field_arithmetic.md` ← row `Carry-less Multiplication for Binary-Field Arithmetic`，引用 `"architectural dependency remains"`、`"trades compute for memory latency"`、`"carry-less field multiply implemented as a table walk"`；`wide_scalar_memory_access_codegen.md` ← row `Wide Scalar Memory-Access Code Generation`（supporting），引用 `"one logical aggregate snapshot"`；The fix 含 before/after 伪代码、前提、correctness、风险与预期 Profile 信号；`Related PRs：9 条 URL` + `Related PRs：2 条 URL`） | ✅ |
| 5 | 路径合规（载荷：模式=入口 A（profile_backed）；primary/supporting 经因果消除测试仲裁；每个 blueprint leaf 均来自通过 gate 的 row；`th.v*` 未停扫） | ✅ |
| 6 | Phase 5 两侧锚定（载荷：消失侧 `ffa4e8/ffa5b6/ffa49e` → 出现侧 carry_less pattern §Verification） | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 | ✅ |

修正记录：无