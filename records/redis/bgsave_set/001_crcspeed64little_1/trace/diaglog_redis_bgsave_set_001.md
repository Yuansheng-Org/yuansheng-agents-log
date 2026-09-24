Functions under analysis: [crcspeed64little]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`001-crcspeed64little-annotate.txt`，cpu-clock:u，215 samples，percent: local period，含 hot loop 完整覆盖，共 852 行）
- perf stat（可选 bound/context）：已提供（`21-redis-benchmark-riscv-bgsave_set.txt`，但为 redis-benchmark 汇总指标：requests_per_second=15,457.14、avg_latency_ms=2.732 等，无 cycles/instructions/IPC/cache counter → bound-type 结论不可得，见 Phase 1）
- workload/binary/DSO/source context：已提供（`21-redis_rv64_metadata.json`，redis-server ELF：RISC-V 64、DYN/PIE、RVC；源码上下文在工作区 `src/crcspeed.c` / `src/crc64.c` 已核对）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（metadata `redis-server-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`）
- hardware ISA（/proc/cpuinfo 或 riscv_hwprobe）：已提供（metadata `cpuinfo.isa`：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`）
- `vlenb`：已提供（metadata `vector.vlen_bits=128`、`vlenb=16`）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=`cpu-clock:u`，percent type=local period，215 samples，单运行窗口；函数级 workload 贡献未知，percent 为 local 而非 global-period → 收益上界只能表述为函数内局部份额）
- Sampling IP precision（precise_ip / Exact-IP / skid）：缺失（详见 Phase 1；未提供 `precise_ip`、Exact-IP 或 PMU skid 能力信息）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zba_zbb_zbc_zbs_..._v_...`：暴露 `v`（RVV 1.0）、`zbc`（clmul/clmulh）、`zbb`/`zba`/`zbs`、`zfh`/`zfa`；**无 `zvbc`、无 `zvk*`（无 Zvkg/Zvkn 等 vector crypto）** |
| Build ISA | `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_..._zve32f/zve32x/zve64d/zve64f/zve64x_zvl32b/zvl64b/zvl128b`：包含 `v` 全族与 zvl128b；**不含任何 `zb*`（无 zba/zbb/zbc/zbs）**、无 `zvbc` |
| Vector flavor | annotate 全函数 **zero 标准 `v*` 与 vendor `th.v*`**（纯 scalar）；无 flavor mismatch（硬件 V 与 build V 一致），但 Zbc 存在 hardware/build mismatch（见 L0 finding） |
| VLEN | 128 bits（vlenb=16） |
| Bound type | `baseline_gap: bound type`；perf stat 仅含 benchmark 级 RPS/延迟指标，无 cycles/instructions/IPC/cache counter。可选命令：`perf stat -e cycles,instructions,cache-misses,cache-references -p <redis-server pid>` 或对 `redis-server --test-memory` 类 crc64 密集段采样 |
| Sampling semantics | event=`cpu-clock:u`（时间可解释），percent type=**local period**（非 global-period），同一运行窗口，函数 workload 贡献未知 → 收益上界只能表述「当前 sampled event 下的函数内局部样本份额」，不得称 workload 级 Amdahl 上界 |
| Sampling IP precision | `baseline_gap: sampling IP precision`；未提供 precise_ip/Exact-IP/skid 信息；单行高占比只锚定 basic block / loop interval，不做单指令 latency/cost 归因 |

L0 baseline gate #1（hardware 有 `v` / build 无 `v`）：**不成立**——build 已含 `v1p0`+zve64d+zvl128b，V 能力一致。
L0 baseline gate #2（th.v* flavor gate）：**不成立**——annotate 无 `th.v*`，硬件支持 RVV 1.0，无 flavor mismatch。
L0 baseline finding（置顶）：**hardware 暴露 `zbc`（clmul/clmulh）而 build ISA 不含任何 `zb*`**——当前二进制在 ISA 层面无法发射 `clmul`/`clmulh`，这是 carry-less 路由（Phase 3 finding 1）的实现准入前提，本身作为最高优先级 baseline finding 报告，同时继续扫描其余 row。
Bound-type gate：`baseline_gap: bound type`，本轮所有命中的 performance-impact confidence 封顶（不得 High）。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`[crcspeed64little]`（1 个）。
- 函数职责（源码 `src/crcspeed.c:179`，CRC-64/GO-ISO，Jones 多项式 0xad93d23594c935a9，ReflectIn/Out=true）：按长度分派 slice-by-1（对齐预热）/ tri（>16KB）/ dual（>1KB）/ single-8（<1KB）/ byte-tail，全部为 16KB 表 `little_table[8][256]` 查表 GF(2) 更新。
- **hot loop interval**：dual loop `0x18faf2 – 0x18fbfe`（源码 `while (len >= 8)` + `DO_8_1`×2 + `DO_8_2`×2，每迭代 16 字节、16 次表查询），该区间占总样本约 94%（详见 Phase 3）。
- trace anchor（最高占比行）：`6.98 : 18fb08: srli a0,a4,0x5`（dual loop 内索引移位）与 `6.51 : 18fb5a: ld a6,-2048(a0)`（表查询 load）。
- 其它区间：tri loop `0x18f6fa–0x18f89e` 为 0 样本（非 x86 分支 cutoff=16KB/1KB，bgsave_set 写入块长主要落在 dual 区间，与 profile 一致）；single-8 loop `0x18f950–0x18f9d4` ~0.5%；byte-tail/epilogue/prologue 各 <1%。cold 区间不参与热点根因。
- Sampling IP precision 未确认：最高行只锚定 dual loop interval，不承担单指令 latency 归因。
- 当前代码来源：compiler-generated C（`crcspeed.c` 宏 `DO_8_2` 展开），非手写 `.S`、非 JIT；`crcspeed64native` 仅按端序分派 little/big，无任何 ISA dispatch。

## Phase 3 — Pattern scan / 模式扫描：crcspeed64little

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — 当前代码来源为 compiler-generated C（源码宏展开 + 反汇编无手写 `.S` 特征），且无 policy-backed missing `.S` 四证（redis crc64 无独立 `.S` policy、无 dispatch slot、无 assembly 约定）。
2. `rows-operator-rvv.md` — **include** — compiler-generated scalar loop、main loop 全 scalar，需逐行评估 no-vectorization / indexed-gather 等 row。
3. `rows-string-memory.md` — **include** — 热点属 checksum/CRC 家族，需评估 Scalar SWAR Checksum 行及其互斥。
4. `rows-vectorized-tuning.md` — **exclude** — annotate 全函数 zero `v*`，无 RVV 配置/寄存器/policy 可调。
5. `rows-codegen.md` — **include** — compiler-generated 指令形态（寻址生成、寄存器压力、调度、ISA substitution）需逐行评估。
6. `rows-offload.md` — **exclude** — 目标无矩阵引擎/P-DSP，问题非权重重排、可移植层或分块算术。
7. `rows-crypto.md` — **include** — profile 点名 CRC64 原语（CRC/GF(2^k) 明确列于该 class 触发条件）。
8. `rows-runtime-os.md` — **exclude** — 用户态热点函数，无 timer/CSR/特权路径。

Classes scanned: `rows-crypto.md`、`rows-operator-rvv.md`、`rows-string-memory.md`、`rows-codegen.md`

### Local performance pattern scan: `crcspeed64little`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Carry-less Multiplication for Binary-Field Arithmetic（primary） | ① hot dual loop（0x18faf2–0x18fbfe，函数内样本 ~94%）是 GF(2) 表驱动乘法：每 8 字节 8 次 `srli/andi 2040/add` 索引链 + 表 load（`6.51 : 18fb5a: ld a6,-2048(a0)`、`2.79 : 18fb1c: ld s1,0(a0)`、`3.26 : 18fb1e: ld t2,-2048(t2)`），加 xor 归并；② 全函数 zero `clmul`/`clmulh`；③ 硬件 cpuinfo 含 `zbc`；④ build `Tag_RISCV_arch` 无 `zb*`；⑤ 源码 `DO_8_2` 宏 = 8 表查询 xor，`CRC64_REVERSED_POLY=0x95ac9329ac4bc9b5`（反射域常量）已在源码中 | **High**（HW `zbc` 直接可观测 + build 无 `zb*` 直接可观测 + 内层循环纯查表逐字可观测 + 无 clmul 发射；无 Zvbc 故仅标量路径） | **Medium**（函数内 local share ~94% 可加总、VLEN=128 已知；但 `baseline_gap: bound type`、`baseline_gap: sampling metadata`（workload 级份额未知）→ 封顶） | `patterns/carry_less_multiplication_for_binary_field_arithmetic.md` |

Supporting evidence：无（见下方排除与互斥）。

#### (a) 逐字 evidence 引用（primary，dual loop interval 0x18faf2–0x18fbfe）
- `6.98 : 18fb08: srli a0,a4,0x5` —— dual loop 内表索引移位（对应 `DO_8_2` 的 `crc>>40` 取表索引）
- `6.51 : 18fb5a: ld a6,-2048(a0)` —— 表查询 load（`little_table[k][idx]`）
- `4.19 : 18fb04: zext.b t2,a4`、`3.26 : 18fb1e: ld t2,-2048(t2)`、`3.26 : 18fb42: xor a1,a1,t2`、`4.19 : 18fb46: ld a3,0(a3)` —— 同一区间内的索引截断、表 load 与 xor 归并链
- 全函数范围内无任何 `clmul`/`clmulh`（annotate 852 行全量核对）
- 最高占比行只锚定 dual loop interval（Sampling IP precision 未确认），不承担单指令 latency 归因

#### (b) 互斥邻居排除
- **vs `Precomputed Lookup Table Reuse for Polynomial Kernels`（rows-crypto 行 4）**：该行 gate 要求 table ownership/reuse 证据（私有 256-entry 表无法复用、每调用点重新生成/复制、或 wrapper state-ordering 语义不匹配）。redis `crc64_table[8][256]` 是单份 static global，`crc64_init()` 启动期一次性生成（`crcspeed64little_init`），所有 callsite 复用同一表，无重新生成/复制 → 行 4 gate 不成立；且行 4 行内互斥写明「CLMUL/RVV polynomial folding 是主杠杆 → carry-less row」。排除。
- **vs `Dedicated RISC-V Vector Crypto Instructions`（rows-crypto 行 1）**：硬件无 `zvk*`/`zvg*`（cpuinfo isa 无 Zvkg/Zvkned 等），hot loop zero `v*` → 行 1 gate 不成立。排除。
- **vs `Scalar SWAR Checksum Reduction`（rows-string-memory 行 1）**：行内互斥明确「CRC/GHASH/GF(2) → carry-less row 或 polynomial row」；本热点是 CRC-64，直接路由到 carry-less row。排除。
- **vs `No vectorization`（rows-operator-rvv 末行）**：该行互斥明确「crypto block-mode / 更具体 semantic row → 各自更具体 row」；GF(2) 语义由 carry-less row 认领，no-vectorization 只解释 zero-`v*` 表象、不认领机制。排除（不构成独立 finding）。
- **vs `RVV Indexed Gather for Table Lookup`（rows-operator-rvv gather 行）**：该行要求「迭代间无跨元素依赖（如 `dst[i]=table[idx[i]]`）」；CRC dual loop 的 `crc` 状态跨迭代循环依赖（本次 crc 是下次索引输入），不满足无依赖条件，且更具体的 crypto 语义 row 认领。排除。
- **vs `Load/Store Addressing-Mode Fusion`（rows-codegen）**：每查询的 `andi 2040 + add + ld` 是 RV64I 对 8 字节表项（index×8）唯一合法的寻址序列（无 reg+reg×8 或 reg+imm12 之外的折叠形式），不存在可折叠的 address-generation 缺陷。排除。
- **vs `Register Pressure and Save/Restore Optimization`（rows-codegen）**：prologue 压入 12 个 callee-saved（s0–s10、ra）仅占 ~1% 样本；hot dual loop 内无 stack spill/reload，全部值在寄存器中 → 行 gate（hot interval 由 spill/reload 主导）不成立。排除。
- **vs `Resource-Aware Instruction Scheduling`（rows-codegen）**：load-use/依赖证据同属于 carry-less 算法层证据；按因果消除测试，替换算法后调度信号整体消失 → 算法层 finding 认领 root cause；且 `baseline_gap: bound type` 与缺 target-core PMU 证据不满足该行 gate。排除。
- **vs `RISC-V ISA Extension-Specific Instruction Substitution`（rows-codegen）**：该行互斥写明「CRC/GF(2) table reuse 或 CLMUL folding 分别走 polynomial-table / carry-less rows」→ 由 carry-less row 认领。排除。

#### (c) 双 Confidence 推导式
- route：HW cpuinfo 直证 `zbc` + readelf 直证 build 无 `zb*` + annotate 直证纯查表循环且无 clmul + 源码直证 `DO_8_2` 表驱动 → **High**。
- impact：函数内 dual-loop local sample share ≈ 94% 可加总，VLEN=128 已知；缺 `bound type`、缺 workload 级贡献（`baseline_gap: sampling metadata`）→ **Medium**（不得 High，不得称 Amdahl 上界）。

### 多候选仲裁小段
顶层 finding 仅 1 条（carry-less，primary）。L0 baseline finding（HW `zbc` vs build 无 `zb*`）先于一切 pattern，作为实现准入前提与独立发现并列报告（evidence-mechanism layer：L0 build/baseline）。无 companion/independent 顶层命中；supporting 0 条；互斥排除 8 条（见上）；推导式 1 条。入口条件 A：顶层 finding 的 evidence sample share 加总 = dual loop ≈ 94% + 函数内余量，函数 workload 级贡献未知。

## Phase 4 — Root-cause blueprint / 根因蓝图：crcspeed64little

命中 row：`rows-crypto.md` 行 3（Carry-less Multiplication for Binary-Field Arithmetic）→ 已读 `patterns/carry_less_multiplication_for_binary_field_arithmetic.md`。

1. **Root cause**：CRC-64（GO-ISO/Jones，反射序）本质是 GF(2^64) 上的无进位多项式运算（多项式模 0xad93d23594c935a9 除法）。当前实现把该二元域乘法以 **slice-by-8 预计算表**模拟：每 8 字节数据需 8 次表查询（`DO_8_2` 宏），每次查询 = 移位取字节 + `andi 2040` + `add` 基址 + `ld`，再把 8 个表值 xor 归并。依据 `patterns/carry_less_multiplication_for_binary_field_arithmetic.md` §Why this is slow：「Table-based multiply trades compute for memory latency... repeated table lookups and dependent shift/xor chains」（将一次域乘法转化为相互依赖的 load+XOR+shift 链）；「a target that *does* expose `Zbc`/`Zvbc` but never dispatches to it is leaving that whole class of instruction on the floor」——本目标 SG2044/C920v2 硬件暴露 `zbc`（`clmul`/`clmulh`），但 build `Tag_RISCV_arch` 无任何 `zb*`、`crcspeed64native` 也无 ISA dispatch，故热循环从未发射 carry-less 指令。叠加反射序语义：redis CRC 为 ReflectIn/ReflectOut=true，其反射域归约常量 `CRC64_REVERSED_POLY=0x95ac9329ac4bc9b5` 已在源码（`src/crcspeed.c:44`）中就绪，反射域 CLMUL folding 可直接在无位翻转的反射域内进行（pattern §Leverage point 3 的 reversal 开销在本反射序场景可省去——这与 GHASH 非反射场景不同，是 CRC-reflected 的特有利条件）。
2. **The fix / 修复方式**（与 pattern §The fix 一致，解释用，不实施）：
   - 纠正对象：`crcspeed64little`（及 `crcspeed64little_init` 表生成）的 hot dual/tri loop，改为 **Zbc 标量 CLMUL folding**：每迭代用 `clmul`/`clmulh` 对 8/16 字节数据做 GF(2) 折叠（两个 64×64→128 carry-less 乘积），再用 `clmul` 对字段多项式常量（反射域 0x95ac9329ac4bc9b5）两步归约折回 64-bit（pattern §Leverage point 2「plain schoolbook 4-`clmul` multiply + two more `clmul` reduction steps」形态，指令数更少、依赖链更短）。
   - Before（当前 hot loop 指令形态）：
     ```
     ld a4,0(a7); xor a4,s5,a4            # DO_8_1(crc1,next1)
     srli a0,a4,0x5; andi a0,a0,2040; add a0,a0,t3; ld s1,0(a0)   # 表查询×8
     ...（16 次查询/迭代，逐条 srli/andi/add/ld + xor 归并）
     ```
   - After（预期形态，示意）：
     ```
     ld a4,0(a7); xor a4,s5,a4            # 合并数据字
     clmulh t1,a4,k1; clmul t0,a4,k1      # 对折叠常量做 64×64→128 无进位乘积
     clmul  t2,t1,pm; clmul t3,t1,pm; xor a4,t0,...   # 两步 clmul 归约（反射域常量 pm=0x95ac...b5）
     ```
   - 适用前提与 correctness contract：必须保持 redis CRC-64 的完整语义——initial/final XOR（调用方 crc 初值全 1）、ReflectIn/ReflectOut=true、多项式 0xad93d23594c935a9（反射域 0x95ac9329ac4bc9b5）、字节序、以及既有 KAT 向量 `crc64(0,"123456789",9)==e9c6d914c4b8d9ca` 与 Lorem ipsum 向量 `c7794709e69683b3`（源码 `src/crc64.c` 已有）必须逐字节一致；短输入（<8B）与 tail 保留原 byte 路径。
   - 实施载体与运行时 gate：redis 为跨平台软件，不得静态把全局 `-march` 换成含 zbc 的字符串破坏可移植性。正解是按 pattern §Leverage point 1 的「runtime dispatch」形态：在 `crcspeed64native`/`crc64_init` 旁新增 riscv_hwprobe 检测（`RISCV_HWPROBE_KEY_IMA_EXT_0`/`EXT_ZBC`，或 `/proc/cpuinfo`），命中 `zbc` 时选择 CLMUL folding 例程（`.S` 或 `.word` 编码 `clmul`，pattern §Leverage point 1 的 raw-word 编码是汇编器不支持 Zbc 时的可移植细节），否则回退现有查表路径。可选：仅 RISC-V 构建加 `-march=rv64gc_zba_zbb_zbc` 使编译器可自行发射 clmul（在确认目标工具链与运行核支持后）。
   - 限制/风险：① HW 无 `zvbc`，无向量 tier，只走标量 Zbc；② build 当前无 `zb*`，任何使用 clmul 的路径必须带运行时 gate，否则在无 zbc 的核上 illegal instruction；③ 反射域 folding 的常量与归约步骤必须经 KAT 与随机 fuzz 验证（pattern §Verification 的 bit-exact 要求）；④ 收益受 bound type 影响——若 workload 实际 memory-bound（RDB 写入 IO 主导），crc64 吞吐提升的端到端可见性下降（`baseline_gap: bound type`）。
   - 修复后预期变化的 Profile signals：dual loop 内 `ld *,-2048(*)` 表查询行与 `srli/andi/add` 索引链消失/显著缩小，`clmul`/`clmulh` 出现于 hot loop；`crcspeed64little` 函数内样本份额大幅下降。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zbc_..._v...`（有 `zbc`、`v`，无 `zvbc`）；build ISA=`rv64i2p1..._v1p0_zve64d..._zvl128b`（无 `zb*`）；VLEN=128 bits；bound type=`baseline_gap: bound type`；采样语义=local period，仅函数内份额。
4. **收益上界**：当前 sampled event（cpu-clock:u，local period）下，dual loop interval 的函数内局部样本份额 ≈ 94%。因 percent type=local 且函数 workload 级贡献未知（`baseline_gap: sampling metadata`），不构成 workload 级 Amdahl 上界；仅表述为函数内局部份额，且 CLMUL folding 消除的是查表 load+索引链（compute/latency 部分），IO-bound 份额不在本函数内。
5. **三维路由判定**：① current source：compiler-generated C scalar loop（`src/crcspeed.c`，非 `.S`/JIT）；② implementation existence/reachability：仓库内不存在任何 Zbc/CLMUL CRC 路径，`crcspeed64native` 仅端序分派 → 目标实现缺失且不可达；③ function-level policy：redis 对 crc64 无独立 `.S` 政策（普通 C + 端序分派），故不走 policy-backed missing `.S` 分支，修复以「普通代码/`clmul` 例程 + 运行时 gate」形态落地。hardware 向量 ISA 准入 gate：无 `zvbc`，向量 tier 不适用（不提出 vclmul kernel）。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。
7. **Related PRs 小节**（来自 `patterns/carry_less_multiplication_for_binary_field_arithmetic.md` §Related PRs）：Carry-less Multiplication for GCM/GHASH（OpenSSL）：`Related PRs：3 条 URL`（https://github.com/openssl/openssl/commit/003f5698146b、https://github.com/openssl/openssl/commit/999376dcf339、https://github.com/openssl/openssl/commit/b24684369b76）；Zbc carry-less CRC in kernel（Linux Kernel RISC-V）：`Related PRs：3 条 URL`（https://github.com/torvalds/linux/commit/ee6740fd34eb53c5c76be01201c15310f461b69f、https://github.com/torvalds/linux/commit/72acff5f81851fe0858d2430b35b4b08f8f27a72、https://github.com/torvalds/linux/commit/bbe2610bc5ada51418a4191e799cfb4577302a31）；CRC32 Zvbc vector CLMUL folding（OpenJDK）：`Related PRs：3 条 URL`（https://github.com/openjdk/jdk/commit/2f4f6cc34c10c5519c74abbce8d1715013b50d5d、https://github.com/openjdk/jdk/commit/c517ffba7d9388e75b5d7bba77e565e71c0a7d76、https://github.com/openjdk/jdk/pull/22475）。

## Phase 5 — Verification forecast / 验证预测：crcspeed64little

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：修正根因后，同一 workload 重新 annotate `crcspeed64little`，dual loop interval `0x18faf2–0x18fbfe` 内的表查询 load（`18fb5a: ld a6,-2048(a0)`、`18fb1e: ld t2,-2048(t2)`、`18fb1c: ld s1,0(a0)`）与索引移位/截断链（`18fb08: srli a0,a4,0x5`、`18fb04: zext.b t2,a4`）应消失或显著缩小，函数内样本份额应大幅下降；`crcspeed64native` 的 ISA dispatch 应选中 Zbc 路径。
- **应出现侧**（锚定 pattern §Verification）：热循环内出现 `clmul`/`clmulh`；`objdump -d` 确认 carry-less 操作码实际发射且被运行时派发；KAT 向量 `e9c6d914c4b8d9ca`（"123456789"）与 `c7794709e69683b3`（Lorem ipsum）逐字节一致；runtime gate 逐 tier 切换（zbc 有/无）输出一致；边界输入（零长度、单字节、未 8 对齐、多 block 长缓冲触发反复归约）与参考一致；大缓冲区（8192/16384B 级，对应 bgsave 的 dual/tri 区间）显示 bytes/s 增益。
- **补采/升级数据（最少）**：`perf stat -e cycles,instructions`（界定 bound type，决定收益上界是否可信）；如需 workload 级收益，需 global-period 采样或 bgsave 阶段全程序 annotate 确认 `crcspeed64little` 占 bgsave_set workload 的比例。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：声明的 N 组 Phase 3–5 全部出现 | ✅（载荷：`1/1 组；crcspeed64little`） |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 2 个 L0 gate + bound gate） | ✅（载荷：7 项结论；gap 标签：`baseline_gap: bound type`、`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；`Sampling IP precision` 行已含） |
| 3 | Phase 3 输出要求满足 | ✅（载荷：8 项 `Class selection trace`；`Classes scanned: rows-crypto.md, rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md`；顶层 finding 1（carry-less primary）；evidence 锚点 `18fb08: srli a0,a4,0x5`、`18fb5a: ld a6,-2048(a0)`、`18fb04: zext.b t2,a4`、`18fb1e: ld t2,-2048(t2)`、`18fb42: xor a1,a1,t2`、`18fb46: ld a3,0(a3)`；supporting 0；互斥排除 8 条；推导式 1 条） |
| 4 | Phase 4 输出要求满足 | ✅（载荷：已读 pattern 文件 `patterns/carry_less_multiplication_for_binary_field_arithmetic.md`；命中 row `rows-crypto.md` 行 3；引用短语首词 `Table-based multiply`、`clmul`/`clmulh`、`plain schoolbook`、`runtime dispatch`；The fix 含 before/after 指令形态、correctness contract（KAT `e9c6d914c4b8d9ca`/`c7794709e69683b3`、反射序、initial/final XOR）、风险（无 zvbc 无向量 tier、build 无 zb* 需 runtime gate）、预期 Profile 信号（`ld *,-2048(*)` 消失、`clmul` 出现）；非 missing `.S` 分支；`Related PRs：9 条 URL`（3 OpenSSL + 3 Linux + 3 OpenJDK）） |
| 5 | 路径合规（trace 8 类扫描集、L0–L4 归属、入口模式 A 动态份额排序） | ✅（载荷：模式 A profile_backed；class 列表见 #3；primary=carry-less（L1 向量化/语义分派层）；L0 baseline finding（HW `zbc` vs build 无 `zb*`）先于 pattern 报告且未停扫；无硬凑命中；单顶层 finding 按函数内份额排序） |
| 6 | Phase 5 两侧锚定 | ✅（载荷：消失侧 `18fb08: srli a0,a4,0x5`、`18fb5a: ld a6,-2048(a0)`、`18fb04`、`18fb1e`、`18fb1c`；出现侧 `patterns/carry_less_multiplication_for_binary_field_arithmetic.md` §Verification 的 clmul/clmulh、KAT 等价、runtime gate、边界输入、大 buffer 吞吐） |
| 7 | 契约边界合规（无实施询问、无代码修改、无补丁生成、无向用户追问） | ✅（载荷：交付物止于 Profile 证据、根因蓝图、完整 The fix、验证预测；无 object-clarification 之外的提问） |

修正记录：无
