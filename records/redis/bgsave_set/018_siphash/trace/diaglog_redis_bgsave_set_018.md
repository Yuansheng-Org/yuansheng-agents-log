Functions under analysis: [siphash]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 annotate：已提供（`018-siphash-annotate.txt`，函数 `siphash`，覆盖完整 hot loop body `[1d8964,1d8a00]`，含 cold setup / tail switch / epilogue）
- perf stat（可选 bound/context）：已提供但仅为 benchmark 级汇总（`21-redis-benchmark-riscv-bgsave_set.txt`：requests_per_second 15,457.14、avg_latency 2.732ms、p50 2.351ms、p95 4.583ms、p99 10.703ms、max 210.047ms）；无 cycles/instructions/cache/branch counters → 无法判定 bound type（详见 Phase 1）
- workload/binary/DSO/source context：已提供（redis `unstable` @ `0d6266f2deb9c401b686898e263ebd6fe35bc2b0`；`src/siphash.c` 源码确认 `siphash()`/`siphash_nocase()` 为纯 C 宏实现，compiler-generated，非 `.S`；经 `dict.c:113-121` `dictSdsHash`/`dictSdsCaseHash` 供 dict 使用，bgsave_set 的每次 SET 键哈希均经过）
- readelf -A（热点 object 的 `Tag_RISCV_arch`）：已提供（`redis-server-elf-A`：`rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0`——**无 `zbb`/`zba`/`zbc`/`zbs`，无 `zicclsm`**）
- hardware ISA（`/proc/cpuinfo` 或 `riscv_hwprobe`）：已提供（metadata_snapshot cpuinfo.isa = `rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`——**含 `zba` `zbb` `zbc` `zbs`**，无 `zicclsm`）
- `vlenb`：已提供（metadata_snapshot vector：RVV 1.0，vlen_bits=128，vlenb=16）
- 采样元数据（event / percent type / scope / 窗口）：已提供（header：`cpu-clock:u (6 samples, percent: local period)`；单次 bgsave_set 采样窗口；函数级 workload 贡献未知）
- Sampling IP precision：缺失（annotate header 无 `precise_ip`/Exact-IP 信息）→ `baseline_gap: sampling IP precision`
- 样本总量：6（极少；按 Forbidden shortcuts 规则写入 Phase 0 并进入 confidence 推导——结论降级，结构不降级）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：`rv64imafdcv_..._zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_...`（SOPHGO SG2044 / XuanTie C920v2，OoO，RVV 1.0）。**硬件暴露 `zba`/`zbb`/`zbc`/`zbs`**；无 `zk*`/`zvk*` crypto 扩展名目 |
| Build ISA | 已提供：`Tag_RISCV_arch` = `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_...`——**无 `zbb`/`zba`/`zbc`/`zbs`**，无 `zicclsm`；无 IFUNC/multiversion（单实现） |
| Vector flavor | annotate 内零 `v*`、零 `th.v*`（全 scalar）；hardware 有 `v`（RVV 1.0），build 有 `v1p0`+`zve64d`+`zvl128b`。无 `th.v*` flavor mismatch |
| VLEN | 128 bits（vlenb=16，metadata_snapshot） |
| Bound type | `baseline_gap: bound type`——perf stat 为 benchmark 级 latency/throughput 汇总，无 cycles/instructions/cache/branch counters。可选命令：`perf stat -e cycles,instructions,branch-misses -- ./src/redis-server redis.conf`（配 redis-benchmark 复现 bgsave_set） |
| Sampling semantics | event=`cpu-clock:u`（时间可解释）；percent type=**local period**；同一运行窗口（单次采样）；函数级 workload 贡献未知 → 百分比仅为函数内局部样本份额，禁止 workload 级 Amdahl 表述 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（precise_ip / Exact-IP / skid 能力未知）→ 单行占比只能锚定所属 basic block / loop interval，不承担 instruction-latency 归因 |

L0 baseline gate 判定：
- hardware 有 `v`，build 也有 `v`（`v1p0`）→ 无 hardware-v/build-no-v mismatch。
- annotate 无 `th.v*` → vector flavor gate 不触发。
- **hardware 有 `zbb`（及 `zba/zbc/zbs`），build 无 `zbb`** → 构建能力缺口：编译器无法发出原生 `rol`/`rori`，`ROTL(x,b)` 退化为 `slli`+`srli`+`add` 三指令序列（首优先级 baseline finding，驱动 `isa_extension_specific_instruction_substitution`）。
- Bound-type gate：`baseline_gap: bound type` → 本轮命中 finding 的 performance-impact confidence 封顶 Medium。

## Phase 2 — Scope / 分析边界
- 函数清单与承诺声明一致：`siphash`（1 个）。
- hot loop 锚点（main loop interval `[1d8964, 1d8a00]`，每 8 字节块一次）：
  - `16.67 : 1d8996: or t1,t1,t5`（U8TO64_LE m 拼装）
  - `16.67 : 1d89b6: srli t1,a5,0x33`（SIPROUND ROTL(v1,13) 三指令序列之一）
  - `16.67 : 1d89ec: slli t6,a4,0x20`（SIPROUND ROTL(v0,32) 三指令序列之一）
  - `16.67 : 1d89f2: addi a0,a0,8`（循环步进）
  - cold setup 采样：`16.67 : 1d8862: lbu a7,9(a2)`（k1 载入）、`16.67 : 1d8954: andi t4,a1,7`（left=inlen&7）
  - 主循环区间持有 4/6 函数内局部样本（66.7%）；cold setup 2/6（33.3%）。
- annotate 覆盖完整（hot loop body 在列），入口条件 A（profile_backed）。
- Sampling IP precision 未知 → 最高行仅锚定 loop interval，不归因单指令 latency；hot interval 内证据以区间聚合 + 指令形态交叉确认。

## Phase 3 — Pattern scan / 模式扫描：siphash

### Class selection trace（8 项）
1. `rows-asm.md` — **exclude** — 当前代码来源为 compiler-generated C（annotate 无 `.S`/`v*` 特征；`src/siphash.c` 为纯 C 宏 `ROTL`/`SIPROUND`/`U8TO64_LE`）；Redis 对 `siphash` 无 per-function 汇编 dispatch policy，policy/existence 四证不成立。
2. `rows-operator-rvv.md` — **include** — compiler-generated scalar hot loop，需评估 no-vectorization 及 operator semantic rows。
3. `rows-string-memory.md` — **include** — 固定宽度 logical access（U8TO64_LE 逐 byte 拼装 64-bit word）→ 评估 wide-scalar-memory / alignment rows。
4. `rows-vectorized-tuning.md` — **exclude** — annotate 零 `v*`（无向量代码可调）。
5. `rows-codegen.md` — **include** — 指令形态：hot path 出现 rotation 多指令合成序列，而硬件 Zbb 已具备、build 未启用 → `isa_extension_specific_instruction_substitution`。
6. `rows-offload.md` — **exclude** — 无矩阵引擎 / packed-SIMD / GEMM 证据。
7. `rows-crypto.md` — **include** — `siphash` 为键控哈希（crypto-adjacent），需核对 4 个 crypto rows。
8. `rows-runtime-os.md` — **exclude** — 用户态 Redis 应用函数，非 RTOS/kernel timer/CSR 热点。

Classes scanned: `rows-operator-rvv.md`, `rows-string-memory.md`, `rows-codegen.md`, `rows-crypto.md`

### Local performance pattern scan: `siphash`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RISC-V ISA Extension-Specific Instruction Substitution（primary） | SIPROUND 内 ROTL 为 `slli`+`srli`+`add` 三指令序列（如 `1d89b6: srli t1,a5,0x33` / `1d89bc: slli a3,a5,0xd` / `1d89c4: add a3,a3,t1`）；硬件 cpuinfo 含 `zbb`，build `Tag_RISCV_arch` 无 `zbb` | High | Medium（bound type 缺失封顶） | `patterns/isa_extension_specific_instruction_substitution.md` |

#### Finding 1（primary）：RISC-V ISA Extension-Specific Instruction Substitution

**(a) 逐字 evidence 引用**
- 主循环 SIPROUND 内 ROTL(v1,13) 三指令序列（interval `[1d8964,1d8a00]`，块内第 2 个 SIPROUND 起点附近）：
  - `16.67 : 1d89b6: srli t1,a5,0x33`
  - `0.00  : 1d89bc: slli a3,a5,0xd`
  - `0.00  : 1d89c4: add a3,a3,t1`
- 主循环 SIPROUND 内 ROTL(v0,32) 三指令序列：
  - `0.00  : 1d89e8: srli a2,a4,0x20`
  - `16.67 : 1d89ec: slli t6,a4,0x20`
  - `0.00  : 1d89f8: add t6,t6,a2`
- 源语义（`src/siphash.c:76`）：`#define ROTL(x, b) (uint64_t)(((x) << (b)) | ((x) >> (64 - (b))))`——即 `(x<<b)|(x>>(64-b))` 标准 rotation，编译器因 build 无 Zbb 而展开为 shift+add 序列。
- 每 8 字节块 2×SIPROUND、每 round 6 次 ROTL → 12 次 rotation/块，每次 3 条指令 = 36 条；整个主循环体约 78 条指令（8 lbu + 7 slli + 7 or 的 m 拼装 22 条 + 2×SIPROUND 52 条 + 2 xor + addi + bne），rotation 占循环体约 46%。

**(b) 互斥邻居排除**
- `No vectorization`（rows-operator-rvv）：排除——SipHash 主循环为 loop-carried 串行状态（v0..v3 跨块依赖、`v3^=m` 后 SIPROUND 再 `v0^=m`），单流哈希无 lane-independent 并行轴，RVV 变换不适用于本调用契约（dict 单键哈希）；且主导信号（rotation 三指令）由更具体的 codegen row 认领，符合该 row 行内互斥"compiler/JIT codegen 形态 → 各自更具体 row"。
- `Wide Scalar Memory-Access Code Generation`（rows-string-memory）：排除——U8TO64_LE 逐 byte 拼装是源码**主动选择的 Zicclsm-gated 回退**（`src/siphash.c:67-96`：`UNALIGNED_LE_CPU` 仅当 `__riscv_zicclsm` 定义时启用单次 `ld`；本硬件 cpuinfo 与 build arch 均无 `zicclsm`），不是编译器漏用合法 snapshot load；无 alignment/lifetime/over-read 证明，不满足该 row gate。
- `Resource-Aware Instruction Scheduling`（rows-codegen）：排除——问题在指令**选择**（未选原生 `rol`），非已选指令的调度/placement；pattern 原文明确 "If the native instruction is already selected but placed poorly... route to `resource_aware_instruction_scheduling.md`"，此处未选中原生指令。
- `Runtime CPU Feature Dispatch`（rows-codegen）：排除——`siphash` 为静态编译单实现，无 runtime dispatch 接线；修复点在 build ISA（-march），非运行时 feature dispatch。
- `Dedicated Vector Crypto Instructions`（rows-crypto）：排除——`siphash` 不在 rows-crypto 命名原语集（AES/SHA/SM3/SM4/GHASH/CRC/GF(2^k)）内，且无对应 Zvk* 指令；crypto row 自身互斥"单条标量 crypto 指令可替换 → isa_extension_specific_instruction_substitution.md"。
- `Carry-less Multiplication` / `Cipher-Mode Parallelism` / `Polynomial Table Reuse`（rows-crypto）：排除——SipHash 无 GF(2^k) 乘法、无块密码 mode、无查找表。

**(c) 双 Confidence 推导式**
- `route: 主循环 rotation 三指令序列（annotate 逐字行）+ 硬件 cpuinfo 含 zbb + readelf build arch 无 zbb + ROTL 宏源码语义 → High`
- `impact: 主循环区间函数内局部样本份额 4/6（66.7%）+ rotation 为循环体最大指令类（约 46%）→ Medium；但缺 bound type（baseline_gap: bound type）、采样语义仅 local period、函数 workload 贡献未知 → 不得升 High，不得给 workload 级收益`

**仲裁小段**：单命中（1 个 primary），无 supporting/companion/independent 并列。Evidence-mechanism layer：L4 compute/codegen micro-structure（`isa_extension_specific_instruction_substitution` 属 L4 行）；其上游 build-capability 缺口（hardware zbb vs build 无 zbb）属 L0 build/path 事实，作为 baseline finding 置顶（Phase 1），不改变本 pattern 归属。动态优先级：入口条件 A 有局部份额但无 workload 级份额，按函数内局部样本份额 4/6 排序（仅此一个顶层 finding）。

## Phase 4 — Root-cause blueprint / 根因蓝图：siphash

对应 Phase 3 通过 gate 的 row：`RISC-V ISA Extension-Specific Instruction Substitution`（`patterns/isa_extension_specific_instruction_substitution.md`）。

1. **Root cause**：redis-server 二进制按不含 `zbb` 的 `Tag_RISCV_arch`（`rv64i2p1_..._v1p0_...`）构建，而目标硬件 SG2044/C920v2 的 cpuinfo 明确暴露 `zbb`。`src/siphash.c:76` 的 `ROTL(x,b) = (x<<b)|(x>>(64-b))` 在无 Zbb 的 build 下被编译器展开为 `slli`+`srli`+`add` 三指令合成序列（annotate `1d89b6/1d89bc/1d89c4` 等），主循环每 8 字节块 12 次 rotation 消耗约 36/78 条指令。依据 pattern §Why this is slow 第 1 条："Without the Zbb extension, a byte rotation is implemented with 3–4 base instructions (e.g., `slli`, `srli`, `or`)"，且实测参考"replacing the synthesized rotation with the native `ror` instruction in SHA‑512 yields a 55.68% throughput increase"；依据 §The fix 第 1 条"Native Rotation Instructions (Zbb)"。典型替换表："`SLLI + SRLI + OR` for rotation by constant → `ROR` / `ROL` (Zbb) | Zbb extension (rva22u64)"。
2. **The fix / 修复方式**：以包含 `zbb` 的 `-march` 重建 redis-server，使编译器把 `ROTL` 宏 lowering 为单条 `rol`。
   - 修复前（当前 annotate 形态，ROTL(v1,13)）：
     ```asm
     srli t1,a5,0x33
     slli a3,a5,0xd
     add  a3,a3,t1     # 3 指令合成 rotation
     ```
   - 修复后（Zbb）：
     ```asm
     rol  a3,a5,13     # 1 条原生 rotation
     ```
   - 具体操作：`make` 时传 `CFLAGS="-march=rv64gc_zba_zbb_zbc_zbs"`（或工具链支持时用与 cpuinfo 等价的完整 march / `-march=native`），clean rebuild 整个 redis（server 与 benchmark 均受益，因 siphash.o 同时链接进 server/cli/benchmark）。不要求改 `src/siphash.c` 源码；如需保留多目标可移植性，可用 `__riscv_zbb` 宏 guard 内联 `rol` asm，但最简路径是 build ISA 修正。
   - 适用前提：目标部署核必须真正支持 Zbb（本平台 cpuinfo 已含 `zbb`）；kernel/OpenSBI 与 userspace 使用同一 ISA 认知。
   - correctness contract：`rol` 与 `(x<<b)|(x>>(64-b))` 位级等价，SipHash 输出必须 bit-identical——dict 哈希稳定性（dict 查找/插入、RDB 键序、`siphash_test` 已知向量）不得改变。
   - 限制/风险：无正确性风险；仅改变指令形态。若未来要跑在无 Zbb 的旧核上，需保留 fallback build（部署决策，非本诊断范畴）。
   - 修复后预期 Profile signals：annotate 中 `srli`+`slli`+`add` rotation 三指令序列消失，代之以 `rol`；主循环每 8 字节块指令数约减少 24（36→12 条 rotation），循环体指令数约 -31%。
3. **Baseline facts 回填**：hardware ISA=`rv64imafdcv_..._zba_zbb_zbc_zbs_...`；build ISA=`rv64i2p1_..._v1p0_...`（无 zbb）；VLEN=128（本 finding 为 scalar，不依赖 VLEN）；bound type=`baseline_gap: bound type`。
4. **收益上界**：当前 sampled event（cpu-clock:u，local period）下，siphash 函数局部样本共 6 个，主循环区间占 4/6≈66.7%；rotation 指令占主循环体约 46%（每块 36/78 条）。仅表述为「当前 sampled event 下的函数内局部样本份额」，**不得**称 workload 级 Amdahl（采样语义四条件不成立：percent type=local period、函数 workload 贡献未知；bound type 缺失）。
5. **三维路由判定**：
   - `current source`：compiler-generated scalar C（`src/siphash.c` 宏实现，annotate 无 `.S` 证据，非 intrinsic/手写汇编）。
   - `implementation existence/reachability`：单一编译实现，无 dispatch/fallback；修复位于 build/toolchain 层（-march 加入 zbb），不涉及 runtime 可达性。
   - `function-level policy`：Redis 对 `siphash` 无独立 `.S` 载体 policy（非 missing-`.S` 分支）；不满足 policy/existence 四证，不走 assembly-primary 结构。
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。
7. **Related PRs 小节**（来自 `patterns/isa_extension_specific_instruction_substitution.md` §Related PRs，同 pattern 内按 URL 去重）：
   - Go：https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0 、https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c 、https://github.com/golang/go/pull/59488 、https://github.com/golang/go/commit/63ab68ddc5f1307e552cf27ae7a6f0dfda2bb962 、https://github.com/golang/go/commit/1951afc9193f8e197cb7dfaf6afed70ea02404cb
   - OpenCV：https://github.com/opencv/opencv/commit/a00818047ff5671586c7b296cac6175b250f85d3 、https://github.com/opencv/opencv/commit/7e2c8cc9f4c49a3afeee084a2f01f7bac1d9a2dc 、https://github.com/opencv/opencv/commit/f0d29cd33c5d2b8f6c9c5c3177cbb3a359ee6b33 、https://github.com/opencv/opencv/pull/21351
   - OpenSSL：https://github.com/openssl/openssl/commit/03ce37e11729 、https://github.com/openssl/openssl/commit/ca6286c382a7 、https://github.com/openssl/openssl/commit/48b6776678d7 、https://github.com/openssl/openssl/commit/6136408e6abf 、https://github.com/openssl/openssl/commit/e4fd3fc379d7 、https://github.com/openssl/openssl/commit/80c664db430d 、https://github.com/openssl/openssl/commit/08c8dd6b8cede3cdbe5b1866c1a7544e0fe7a378 、https://github.com/openssl/openssl/commit/49a3e7adc392 、https://github.com/openssl/openssl/commit/a41f9135f082 、https://github.com/openssl/openssl/commit/4dbb537bd1ea 、https://github.com/openssl/openssl/commit/608cadfbdbdb 、https://github.com/openssl/openssl/commit/b1b889d1b3fc 、https://github.com/openssl/openssl/commit/657d1927c68b 、https://github.com/openssl/openssl/commit/611685adc04a 、https://github.com/openssl/openssl/commit/7ae2bc9df6e0
   - Linux Kernel RISC-V：https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712 、https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49 、https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92 、https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de 、https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355 、https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb 、https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68
   - OpenJDK：https://github.com/openjdk/jdk/commit/6b89954c65342bc601633d24075dab4f4b248f4b 、https://github.com/openjdk/jdk/commit/a7631ccf18e468d6ecba121865f7fed29cbf2186 、https://github.com/openjdk/jdk/pull/22752 、https://github.com/openjdk/jdk/commit/08d563ba15047020fd5f5fea80547e18898bbab2 、https://github.com/openjdk/jdk/commit/b1a21b563e3ae13fa5c409a4f0c04686c3f5b34a 、https://github.com/openjdk/jdk/pull/22410 、https://github.com/openjdk/jdk/commit/9f582e56baee0e7f5af20da0f395cd935bf5a962 、https://github.com/openjdk/jdk/pull/24096 、https://github.com/openjdk/jdk/commit/1a4bbb0027ae9e6df3b668454fa155861d531f72 、https://github.com/openjdk/jdk/commit/2ed7ad4b5c7d2344ae6571c186f8a2903770aa57 、https://github.com/openjdk/jdk/commit/edfe28541a6ed94357f873aa69778c7eba707cbb 、https://github.com/openjdk/jdk/commit/b891bfa7e67c21478475642e2bfa2cdc65a3bffe 、https://github.com/openjdk/jdk/commit/3d3b7820371058b40f2e694536c98aa3900abb5f 、https://github.com/openjdk/jdk/commit/a7a09f69abc6c4730599d3de9067c2fde75c5172 、https://github.com/openjdk/jdk/commit/bcc33d5ef3bdbfaee51c45014851c54028da03f1 、https://github.com/openjdk/jdk/pull/22386 、https://github.com/openjdk/jdk/commit/5866b16dbca3f63770c8792d204dabdf49b59839 、https://github.com/openjdk/jdk/commit/8cb9b479c529c058aee50f83920db650b0c18045 、https://github.com/openjdk/jdk/pull/11921
   - llama.cpp：https://github.com/ggml-org/llama.cpp/pull/17784
   - V8：https://github.com/v8/v8/commit/6f100865663fb99df2628144fc65977e55ab7e68 、https://github.com/v8/v8/commit/e62c1e307d207e1e56219c172929289a1b474530 、https://github.com/v8/v8/commit/200b5212ae4752f6345dbcc6ecf23f651434bbf8 、https://github.com/v8/v8/commit/5136fb5200c1f3a33939419fed9de32e8d85bc1e 、https://github.com/v8/v8/commit/e02d2238f6ea6f920a6ac31f888b4727bb0b0f80 、https://github.com/v8/v8/commit/94a3c420e2a4f76d42e367e7d3b3f85ecd1b0a3f 、https://github.com/v8/v8/commit/1818e36d54d12540080dd58dbfb924155a920cba 、https://github.com/v8/v8/commit/5f433dd5024a566fa051317dd0c9c1d9921a9162 、https://github.com/v8/v8/commit/9bbbde26bb90bd493afe33777d7b6be0c4f2afb4 、https://github.com/v8/v8/commit/758956654f28861ba0d5e94f03fff6016b3a2998 、https://github.com/v8/v8/commit/32c5d22333c14a9d6bb645af3ea4d3fb28b541fe 、https://github.com/v8/v8/commit/89719bc239f48735bbffe83e0807fd504b3c485a 、https://github.com/v8/v8/commit/223d7fb26b12145bdd335d51da2fa2ed1c58ce3a 、https://github.com/v8/v8/commit/b2852080c401cb0980f332918531597967de36c1 、https://github.com/v8/v8/commit/8b842dbb9f6d7ea03990d80dc945ef8c6c6ca144 、https://github.com/v8/v8/commit/e855c14cab2c6522a2a2d9b6be51e886fd3aae74 、https://github.com/v8/v8/commit/8033cc56e2afb7b19214aa9a6f776a59fc509d2c 、https://github.com/v8/v8/commit/a654e27b50feb457a99ad27582dd555316eab587 、https://github.com/v8/v8/commit/21289aa92c80a4161cc5f0d9915a0002a644d8b4 、https://github.com/v8/v8/commit/19a0f69f4c4b1257b567290412acda411568d565
   - QEMU：https://github.com/qemu/qemu/commit/3de1fb712a072992d72bc99c2b70978132ee44d0 、https://github.com/qemu/qemu/commit/6ef5843182382f6a84995590ad91047b0f2bc1fa
   - LLVM：https://github.com/llvm/llvm-project/pull/170824 、https://github.com/llvm/llvm-project/commit/4c1e1e05cb901a2ed9055e5d6ac6ce60b826a288 、https://github.com/llvm/llvm-project/pull/92926 、https://github.com/llvm/llvm-project/pull/152744 、https://github.com/llvm/llvm-project/pull/122698 、https://github.com/llvm/llvm-project/commit/13e32a8a3c95b23af51f081865db1bd259d269a4 、https://github.com/llvm/llvm-project/commit/787eeb8597fac22decb366a42176b11f52ec1bf0 、https://github.com/llvm/llvm-project/commit/c705b7b04dba467a67871a1bbb77907d0ed7fc19
   - Related PRs：81 条 URL

## Phase 5 — Verification forecast / 验证预测：siphash

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：以含 `zbb` 的 `-march` 重建后，`perf annotate --stdio -l -s siphash` 主循环区间 `[1d8964,1d8a00]` 内不再出现 `1d89b6: srli t1,a5,0x33`、`1d89bc: slli a3,a5,0xd`、`1d89c4: add a3,a3,t1`（ROTL(v1,13) 三指令序列）及 `1d89e8: srli a2,a4,0x20`、`1d89ec: slli t6,a4,0x20`、`1d89f8: add t6,t6,a2`（ROTL(v0,32) 三指令序列），代之以单条 `rol`。
- **应出现侧**（锚定 `patterns/isa_extension_specific_instruction_substitution.md` §Verification）：disassembly/annotate 含原生 `rol`/`rori` 而非 shift-or 序列（对应 §Verification "confirm that SHA‑512 hot loops contain native rori... instead of shift‑or sequences"）；功能等价——`siphash_test()` 全部已知向量 bit-identical、dict 哈希值不变（对应 §Verification "ensure all tests pass identically"）。
- **收益顺序**：唯一 primary finding，无并列；收益上界仅为函数内局部份额（主循环区间 4/6≈66.7%），不声明 workload 级收益幅度。
- **补充验证**：同一 bgsave_set 采样重跑后，主循环区间每 8 字节块指令数下降约 24（rotation 36→12），函数内局部样本份额预期下移；如可能，补 `perf stat -e cycles,instructions,branch-misses` 以闭合 bound-type 缺口。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组 Phase 3–5（`siphash`） | ✅（1/1 组；siphash） |
| 2 | Phase 1 输出要求满足：7 行 baseline 表齐全（hardware/build ISA、vector flavor、VLEN、bound type、sampling semantics、sampling IP precision），结论+出处/gap 标签；两个 L0 gate 判定；bound-type gate 判定；`baseline_gap: bound type`、`baseline_gap: sampling IP precision` 已点名 | ✅（7 行；gap 标签：bound type、sampling IP precision；含 Sampling IP precision 行） |
| 3 | Phase 3 输出要求满足：8 项 Class selection trace；`Classes scanned: rows-operator-rvv.md, rows-string-memory.md, rows-codegen.md, rows-crypto.md`；顶层 finding 1 个（evidence 锚点：`1d89b6: srli t1,a5,0x33`、`1d89bc: slli a3,a5,0xd`、`1d89c4: add a3,a3,t1`、`1d89ec: slli t6,a4,0x20`）；supporting 0；排除 6 条（no-vectorization、wide-scalar-memory、resource-aware-scheduling、runtime-ISA-dispatch、dedicated-vector-crypto、carry-less/cipher-mode/polynomial）；推导式 1 条（route High / impact Medium） | ✅ |
| 4 | Phase 4 输出要求满足：已读 pattern `patterns/isa_extension_specific_instruction_substitution.md`；对应命中 row 1（ISA Extension-Specific Instruction Substitution）；引用短语首词：§Why-this-is-slow "Without the Zbb extension"、"55.68% throughput increase"；§The fix "Native Rotation Instructions (Zbb)"、"rol"；§Typical table "SLLI + SRLI + OR"；The fix 含 before/after（`srli/slli/add` → `rol`）、correctness（bit-identical、siphash_test、dict 稳定性）、风险（无正确性风险；fallback 为部署决策）、预期 Profile signals（rotation 序列消失、每块 -24 指令）；非 missing `.S`，无 implementation-shape；Related PRs：81 条 URL | ✅ |
| 5 | 路径合规：入口模式 A（profile_backed）按函数内局部份额排序；1 个顶层 primary，无 supporting/companion/independent 混排；blueprint leaf 均来自通过 gate 的 row；`th.v*` 未全局停扫（无 th.v* 证据，无 flavor 依赖 route 被冻结） | ✅ |
| 6 | Phase 5 两侧锚定：消失侧对 Phase 3(a) 引用行（`1d89b6: srli t1,a5,0x33` 等）；出现侧标注 `patterns/isa_extension_specific_instruction_substitution.md` §Verification | ✅ |
| 7 | 契约边界合规：无实施询问、无源码修改、无补丁生成；无向用户追问；交付物止于 Profile 证据、根因蓝图、完整 The fix 与验证预测 | ✅ |

修正记录：无
