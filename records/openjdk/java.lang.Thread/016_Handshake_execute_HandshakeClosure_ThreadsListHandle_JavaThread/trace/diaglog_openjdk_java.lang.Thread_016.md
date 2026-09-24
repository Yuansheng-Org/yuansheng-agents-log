Functions under analysis: [Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单
- 单函数完整 perf annotate：已提供（`016-Handshake：：execute(...)-annotate.txt`，event=cpu-clock，109 samples，`percent: local period`，806 行，入口 82b5a0 → 栈检查失败路径 82bc0e 完整覆盖）
- perf stat（bound/context）：已提供（`3-openjdk-benchmark-riscv-java.lang.Thread.txt`）
- workload/binary/DSO/source context：已提供（libjvm.so；DWARF 行 handshake.cpp:73/154-236/394-455、threadSMR.hpp:313、safepointMechanism.inline.hpp、orderAccess_linux_riscv.hpp、atomicAccess_linux_riscv.hpp）
- readelf -A（build ISA）：已提供（`Tag_RISCV_arch "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_zifencei2p0_zmmul1p0_zaamo1p0_zalrsc1p0_zca1p0_zcd1p0_zve32f1p0_zve32x1p0_zve64d1p0_zve64f1p0_zve64x1p0_zvl128b1p0_zvl32b1p0_zvl64b1p0"` —— 不含 zba/zbb/zbc/zbs/zicond/zihintpause/zfa）
- hardware ISA：已提供（`rv64imafdcv_..._zicond_zihintpause_zawrs_zfa_zfh_..._zba_zbb_zbc_zbs_...`，T-Head C920v2 —— 含 B 扩展、Zicond、Zihintpause）
- `vlenb`：已提供（16 bytes → VLEN 128 bits）
- 采样元数据：部分已提供（event=cpu-clock，percent-type=local period，同一运行窗口；函数级 workload 贡献未知 → `baseline_gap: sampling metadata`）
- Sampling IP precision：缺失（→ `baseline_gap: sampling IP precision`）

## Phase 1 — Baseline / 基线
| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcv_..._zicond_zihintpause_zawrs_zfa_zfh_..._zba_zbb_zbc_zbs_...`（含 `v`、B 扩展族、Zicond、Zihintpause；C920v2，OoO） |
| Build ISA | `Tag_RISCV_arch` 含 `v1p0`/`zvl128b1p0`，**缺 `zba`/`zbb`/`zbc`/`zbs`/`zicond`/`zihintpause`/`zfa`** |
| Vector flavor | annotate 有 RVV 1.0 `v*`（`vsetivli`/`vmv.v.i`/`vse32.v`，reset_state 内）→ 无 flavor mismatch |
| VLEN | 128 bits（vlenb=16） |
| Bound type | 全局 IPC 0.451；本函数为 handshake 目标线程等待的自旋循环（spin/latency-bound）；非 memory-bound |
| Sampling semantics | event=cpu-clock；percent type=local period；同一运行窗口；函数级 workload 贡献未知 → 收益上界只能表述为函数内局部份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision`（单行占比只锚定 interval/basic block） |

L0 baseline gate 判定：
- hardware 有 `v` 且 build 有 `v` → 无 `v` mismatch；无 `th.v*`。
- **非 vector 扩展 mismatch（第一优先级 baseline finding）**：hardware 暴露 `zba`/`zbb`/`zbc`/`zbs`/`zicond`/`zihintpause`/`zfa`，而承载热点地址的 libjvm.so `Tag_RISCV_arch` 全部缺失 → 编译器无法对 hot path 发出 `sh1add/sh2add/sh3add`、`andn/orn/xnor`、`czero.eqz`、`pause` 等原生指令。该 finding 置顶但不停止 row 扫描。

## Phase 2 — Scope / 分析边界
函数清单与承诺声明一致：`Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)`（1 个）。
hot interval：**自旋等待循环** 82b6e2 → 82b8fa（回边 `beqz s3,82b8fa` @82b8fa），循环体含 add_result（82b6e2-82b6fa）、safepoint poll 检查（82b6fc-82b708）、state_changed 快照比较（82b7a0-82b7da）、超时检查（82b7de-82b7e4）、wait_raw（82b7e8-82b806）、reset_state（82b8a8-82b8d0）、try_process 分派（82b8de-82b8fa）。
最高占比行（trace anchors，函数内局部占比）：
- `24.77 : 82b706: andi a5,a5,1`（poll 检查区间：依赖 `ld a5,0(a5)`@82b700 与 `fence r,rw`@82b702）
- `10.09 : 82b6f8: addiw a4,a4,1`（add_result 计数区间：load-use 依赖 `lw a4,16(a5)`@82b6f6 3.67%）
- `6.42 : 82b7c6: lw a4,-284(s0)`、`5.50 : 82b7ba: lw a4,-288(s0)`、`4.59 : 82b7d2: lw a4,-280(s0)`（state_changed 快照比较区间）
annotate 覆盖完整。Sampling IP precision 未确认 → 结论收敛到 interval 级机制。

## Phase 3 — Pattern scan / 模式扫描：Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)

### Class selection trace（8 项）
1. `rows-asm.md` — exclude：compiler-generated（DWARF 行 handshake.cpp 等），无 `.S` provenance
2. `rows-operator-rvv.md` — exclude：非算子语义计算 loop（自旋同步循环）
3. `rows-string-memory.md` — exclude：非 copy/fill/scan/compare/checksum
4. `rows-vectorized-tuning.md` — include：annotate 有 `v*`（reset_state 的 `vsetivli zero,4,e32,m1,ta,ma`/`vmv.v.i v1,0`/`vse32.v v1,(a5)`）
5. `rows-codegen.md` — include：compiler-generated 自旋/轮询/分派/同步代码
6. `rows-offload.md` — exclude：无矩阵引擎/packed-SIMD 证据
7. `rows-crypto.md` — exclude：非密码学原语
8. `rows-runtime-os.md` — exclude：userspace HotSpot runtime，非 RTOS/kernel 路径

Classes scanned: `triggers/rows-codegen.md`, `triggers/rows-vectorized-tuning.md`

### Local performance pattern scan: `Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)`
| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RISC-V ISA Extension-Specific Instruction Substitution（primary） | hardware 有 zba/zbb/zbs/zicond/zihintpause，build `Tag_RISCV_arch` 缺；hot loop 索引计算为 slli+add 多指令 scaled-address 链（82b6ea-82b6f4、82b8c6-82b8ce），自旋无 pause hint | High | Medium | `patterns/isa_extension_specific_instruction_substitution.md` |

**(a) 逐字 evidence 引用**（函数内局部占比；interval 归属见括号）：
- `0.00 : 82b6ea: slli a5,a4,0x2` / `0.00 : 82b6ee: add a5,a5,a4` / `0.00 : 82b6f0: add a5,a5,s11` / `0.00 : 82b6f2: slli a5,a5,0x2` / `0.00 : 82b6f4: add a5,a5,s6`（add_result 区间 slot 索引 = 5 条链；同区间 `10.09 : 82b6f8: addiw a4,a4,1`）
- `3.67 : 82b8a8: andi s1,s1,1` / `4.59 : 82b8c6: slli a5,s1,0x2` / `0.00 : 82b8ca: add a5,a5,s1` / `0.00 : 82b8cc: slli a5,a5,0x2` / `0.00 : 82b8ce: add a5,a5,s0`（reset_state 区间索引 = 4 条链）
- L0 证据：build `Tag_RISCV_arch "rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_v1p0_zicsr2p0_..."`（无 zba/zbb/zbs/zicond/zihintpause）；hardware `rv64imafdcv_..._zicond_zihintpause_..._zba_zbb_zbc_zbs_...`

**(b) 互斥邻居排除**：
- bounded_spin_wait_backoff row：排除 —— 循环**已有**有界退避（自旋上限 `max(100000, 5000*(nproc-1))` ns @82b696-82b69a + 超时检查 `bge` @82b7e4）与阻塞 crossover（`os::naked_short_sleep(10000)` @82b848、`naked_short_sleep(1)` @82b864），row 正向 gate「循环没有有界/递增退避或阻塞 crossover」不成立
- hardware_atomic_operations row：排除 —— 循环内 `Mutex::try_lock/unlock` 为 out-of-line 调用（82b720/82ba88，0.00 samples）；poll 检查为内联 `ld; fence r,rw`（正确 acquire lowering，非 full-fence/seq_cst helper）；无 helper 未 lowering 为 AMO 的证据
- redundant_synchronization_elimination row：排除 —— poll word（s2+40）与 handshake state 有并发写者（safepoint 机制与目标线程），无法证明「无并发写者」
- register_pressure_and_save_restore row：排除 —— 432 字节 frame + 11 个 callee-saved 的 prologue/epilogue 样本≈0（82b5a0-82b5c4 与 82b9a8-82b9b8 全 0.00），成本在循环体数据流而非保存恢复
- hot_helper_forced_inlining row：排除 —— 循环内 `os::javaTimeNanos()`/`__tls_get_addr@plt`/virtual `jalr` 的 callsite 全部 0.00 samples，样本不主导在 callsite 或 call-induced spill
- rvv_vector_state_management row（rows-vectorized-tuning）：排除 —— reset_state 每轮仅 1 次 `vsetivli`（0.92%），无重复兼容 vtype 建立/vector-state churn
- rvv_operand_form_selection row（rows-vectorized-tuning）：排除 —— `vmv.v.i v1,0` 后接 `vse32.v` 为清零惯用法，RVV 无 vector-immediate store 等价形态
- 其余 rows-vectorized-tuning rows（autovec control / maximal LMUL / unrolling / register-group / inactive-lane）：排除 —— 无手写 intrinsic kernel、无 RVV main loop、无 LMUL/live-set/spill 证据；v* 仅为 16 字节 autovec 清零（3×0.92%）

**(c) 双 Confidence 推导式**：
- route：build ISA 缺 zba/zbb/zbs/zicond/zihintpause（直接 provenance：冻结 `Tag_RISCV_arch`）+ hardware 具备（冻结 hw profile）+ 反汇编多指令 scaled-address 序列直接可见 → **High**
- impact：受影响索引链区间局部样本份额 ≈10%（82b8c6 4.59 + 82b8a8 3.67 + 82b6e6 0.92 + 82b6ee 0.92 + 82b8d0 0.00 等），采样语义仅函数内局部、`baseline_gap: sampling IP precision` → **Medium**

### 多命中仲裁与负向证据
- 顶层命中仅 1 个：primary = RISC-V ISA Extension-Specific Instruction Substitution（L4 机制；其 L0 build-config 证据见 Phase 1，同一 evidence/机制链条，不另列顶层 finding）。
- **主导区间的负向证据（no local pattern matched，first-principles 微观分析）**：
  - poll 检查区间（82b700-82b708，≈26.6%）：`ld _poll_word; fence r,rw; andi 1; beqz` —— SafepointMechanism 每迭代 load-acquire（fence 序列化 + load→andi 依赖链），属结构性每迭代固定开销；`fence r,rw` 为 RISC-V acquire 的标准 lowering，无 ISA 单指令替换，无 row 命中。
  - state_changed 区间（82b7a0-82b7da，≈20.2%）：10 次栈内快照 `lw` + 5 条 `bne` 串行比较 spin-state 2×5 数组，属 HandshakeSpinYield 状态检测机制本身；栈上数据流量非 save/restore（排除 register-pressure row），无 row 命中。
  - 结论：自旋 yield 机制的每迭代固定开销（poll + 快照比较 + 计数/重置）主导本函数成本；其中唯一可直接修复的 codegen 缺陷为 build 缺 B 扩展导致的索引链放大与缺 spin hint。

## Phase 4 — Root-cause blueprint / 根因蓝图：Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)

纳入蓝图的 pattern：`patterns/isa_extension_specific_instruction_substitution.md`（对应 Phase 3 通过 gate 的 row：RISC-V ISA Extension-Specific Instruction Substitution）。

1. **Root cause**：承载本热点的 libjvm.so 构建 ISA（`Tag_RISCV_arch`）未包含目标硬件已具备的 `Zba`/`Zbb`/`Zbs`/`Zicond`/`Zihintpause`，导致自旋循环中 (a) results 数组 slot 索引计算退化为 5 条（add_result：`slli`+`add`×2+`slli`+`add`）与 4 条（reset_state）的多指令 scaled-address 链 —— 按 pattern §The fix #3「Scaled address generation (Zba)」本可用 `sh2add` 收敛为 3 条/2 条；(b) busy-wait 循环体无 `pause` hint（Zihintpause）。机制句（§Why this is slow #1）：「Excessive instruction count from multi-instruction emulation... consumes extra decode, issue, and retirement slots, reducing sustained IPC」；#2：「Extended latency due to dependent shift pairs」。L0 层面这是构建配置缺口（hardware/build extension mismatch），影响整个 libjvm 而非仅本函数。
2. **The fix / 修复方式**（与 pattern §The fix #3/#4 一致）：
   - 构建配置层：libjvm 以包含 `zba`/`zbb`/`zbs`（可选 `zicond`/`zihintpause`/`zfa`）的 `-march` 重建（硬件快照确认 C920v2 全部支持；pattern §Profile signals 第 5-6 行：`readelf -A` 缺 Zbb/Zba + `/proc/cpuinfo` 确认支持 → 扩展未启用）。
   - 修复前后伪代码（Zba sh2add 替换）：
     - Before（add_result 索引链）：`slli a5,a4,0x2; add a5,a5,a4; add a5,a5,s11; slli a5,a5,0x2; add a5,a5,s6`（5 条）
     - After：`sh2add a5,a4,a4; sh2add a5,s11,a5; add a5,a5,s6`（3 条）
     - Before（reset_state 索引链）：`slli a5,s1,0x2; add a5,a5,s1; slli a5,a5,0x2; add a5,a5,s0`（4 条）→ After：`sh2add a5,s1,s1; sh2add a5,s0,a5`（2 条）
   - Zihintpause：在 HandshakeSpinYield 自旋路径（pattern §The fix #4）加入 feature-gated `pause`（`.word 0x0100000f`），只作等待提示、不改变同步语义。
   - Correctness contract（不可破坏）：`sh2add rd,rs1,rs2 = rs1 + (rs2<<2)` 的无符号地址算术结果必须逐位一致；`pause` 不得改变锁/内存序语义；HotSpot 构建工具链需支持对应扩展 gate（pattern 明确 RISC-V operand order 与 feature gate 精确性要求）。
   - 限制/风险：需确认 OpenJDK 构建链中扩展启用路径（pattern 内 OpenJDK 已有 Zbb bit-op 与 Zihintpause 提交可作 provenance）；若 toolchain 默认 `-march=rv64gc` 则需显式扩展；pause 收益为微架构依赖（hint 不保证延迟）。
   - 预期 Profile 信号：82b6ea-82b6f4 与 82b8c6-82b8ce 的 slli/add 索引链消失、`sh2add` 出现；libjvm 全局 scaled-address/位操作序列受益；本函数指令数下降。
3. **Baseline facts 回填**：hardware ISA `rv64imafdcv_..._zba_zbb_zbc_zbs_zicond_zihintpause...`；build ISA `...v1p0...`（缺 B 扩展族）；VLEN=128 bits；bound type：spin/latency-bound（全局 IPC 0.451）。
4. **收益上界**：当前 sampled event（cpu-clock, local period）下函数内局部样本份额 ≈10%（索引链区间加总：82b8c6 4.59 + 82b8a8 3.67 + 82b6e6 0.92 + 82b6ee 0.92 + 其余 0.00）；`baseline_gap: sampling metadata`（函数级 workload 贡献未知）→ 不可称 workload 级 Amdahl 上界；poll/state_changed 结构性区间不在本 finding 收益内。
5. **三维路由判定**：
   - current source：compiler-generated（HotSpot C++ 产物，gcc/g++）
   - implementation existence/reachability：不适用（非 kernel 选择/分派问题；指令形态由 build `-march` 决定）
   - function-level policy：libjvm 构建配置（`-march` 未启用硬件已具备的 B/Zicond/Zihintpause 扩展）→ **config_mismatch**
6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。
7. **Related PRs**（`patterns/isa_extension_specific_instruction_substitution.md` §Related PRs，同 pattern 内 URL 去重，81 条 URL）：
   - Go：https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0 、https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c 、https://github.com/golang/go/pull/59488 、https://github.com/golang/go/commit/63ab68ddc5f1307e552cf27ae7a6f0dfda2bb962 、https://github.com/golang/go/commit/1951afc9193f8e197cb7dfaf6afed70ea02404cb
   - OpenCV：https://github.com/opencv/opencv/commit/a00818047ff5671586c7b296cac6175b250f85d3 、https://github.com/opencv/opencv/commit/7e2c8cc9f4c49a3afeee084a2f01f7bac1d9a2dc 、https://github.com/opencv/opencv/commit/f0d29cd33c5d2b8f6c9c5c3177cbb3a359ee6b33 、https://github.com/opencv/opencv/pull/21351
   - OpenSSL：https://github.com/openssl/openssl/commit/03ce37e117 、https://github.com/openssl/openssl/commit/ca6286c382 、https://github.com/openssl/openssl/commit/48b6776678 、https://github.com/openssl/openssl/commit/6136408e6a 、https://github.com/openssl/openssl/commit/e4fd3fc379 、https://github.com/openssl/openssl/commit/80c664db43 、https://github.com/openssl/openssl/commit/08c8dd6b8cede3cdbe5b1866c1a7544e0fe7a378 、https://github.com/openssl/openssl/commit/49a3e7adc3 、https://github.com/openssl/openssl/commit/a41f9135f0 、https://github.com/openssl/openssl/commit/4dbb537bd1 、https://github.com/openssl/openssl/commit/608cadfbdb 、https://github.com/openssl/openssl/commit/b1b889d1b3 、https://github.com/openssl/openssl/commit/657d1927c6 、https://github.com/openssl/openssl/commit/611685adc0 、https://github.com/openssl/openssl/commit/7ae2bc9df6
   - Linux Kernel RISC-V：https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712 、https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49 、https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92 、https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de 、https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355 、https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb 、https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68
   - OpenJDK：https://github.com/openjdk/jdk/commit/6b89954c65342bc601633d24075dab4f4b248f4b 、https://github.com/openjdk/jdk/commit/a7631ccf18e468d6ecba121865f7fed29cbf2186 、https://github.com/openjdk/jdk/pull/22752 、https://github.com/openjdk/jdk/commit/08d563ba15047020fd5f5fea80547e18898bbab2 、https://github.com/openjdk/jdk/commit/b1a21b563e3ae13fa5c409a4f0c04686c3f5b34a 、https://github.com/openjdk/jdk/pull/22410 、https://github.com/openjdk/jdk/commit/9f582e56baee0e7f5af20da0f395cd935bf5a962 、https://github.com/openjdk/jdk/pull/24096 、https://github.com/openjdk/jdk/commit/1a4bbb0027ae9e6df3b668454fa155861d531f72 、https://github.com/openjdk/jdk/commit/2ed7ad4b5c7d2344ae6571c186f8a2903770aa57 、https://github.com/openjdk/jdk/commit/edfe28541a6ed94357f873aa69778c7eba707cbb 、https://github.com/openjdk/jdk/commit/b891bfa7e67c21478475642e2bfa2cdc65a3bffe 、https://github.com/openjdk/jdk/commit/3d3b7820371058b40f2e694536c98aa3900abb5f 、https://github.com/openjdk/jdk/commit/a7a09f69abc6c4730599d3de9067c2fde75c5172 、https://github.com/openjdk/jdk/commit/bcc33d5ef3bdbfaee51c45014851c54028da03f1 、https://github.com/openjdk/jdk/pull/22386 、https://github.com/openjdk/jdk/commit/5866b16dbca3f63770c8792d204dabdf49b59839 、https://github.com/openjdk/jdk/commit/8cb9b479c529c058aee50f83920db650b0c18045 、https://github.com/openjdk/jdk/pull/11921
   - llama.cpp：https://github.com/ggml-org/llama.cpp/pull/17784
   - V8：https://github.com/v8/v8/commit/6f100865663fb99df2628144fc65977e55ab7e68 、https://github.com/v8/v8/commit/e62c1e307d207e1e56219c172929289a1b474530 、https://github.com/v8/v8/commit/200b5212ae4752f6345dbcc6ecf23f651434bbf8 、https://github.com/v8/v8/commit/5136fb5200c1f3a33939419fed9de32e8d85bc1e 、https://github.com/v8/v8/commit/e02d2238f6ea6f920a6ac31f888b4727bb0b0f80 、https://github.com/v8/v8/commit/94a3c420e2a4f76d42e367e7d3b3f85ecd1b0a3f 、https://github.com/v8/v8/commit/1818e36d54d12540080dd58dbfb924155a920cba 、https://github.com/v8/v8/commit/5f433dd5024a566fa051317dd0c9c1d9921a9162 、https://github.com/v8/v8/commit/9bbbde26bb90bd493afe33777d7b6be0c4f2afb4 、https://github.com/v8/v8/commit/758956654f28861ba0d5e94f03fff6016b3a2998 、https://github.com/v8/v8/commit/32c5d22333c14a9d6bb645af3ea4d3fb28b541fe 、https://github.com/v8/v8/commit/89719bc239f48735bbffe83e0803fd504b3c485a 、https://github.com/v8/v8/commit/223d7fb26b12145bdd335d51da2fa2ed1c58ce3a 、https://github.com/v8/v8/commit/b2852080c401cb0980f332918531597967de36c1 、https://github.com/v8/v8/commit/8b842dbb9f6d7ea03990d80dc945ef8c6c6ca144 、https://github.com/v8/v8/commit/e855c14cab2c6522a2a2d9b6be51e886fd3aae74 、https://github.com/v8/v8/commit/8033cc56e2afb7b19214aa9a6f776a59fc509d2c 、https://github.com/v8/v8/commit/a654e27b50feb457a99ad27582dd555316eab587 、https://github.com/v8/v8/commit/21289aa92c80a4161cc5f0d9915a0002a644d8b4 、https://github.com/v8/v8/commit/19a0f69f4c4b1257b567290412acda411568d565
   - QEMU：https://github.com/qemu/qemu/commit/3de1fb712a072992d72bc99c2b70978132ee44d0 、https://github.com/qemu/qemu/commit/6ef5843182382f6a84995590ad91047b0f2bc1fa
   - LLVM：https://github.com/llvm/llvm-project/pull/170824 、https://github.com/llvm/llvm-project/commit/4c1e1e05cb901a2ed9055e5d6ac6ce60b826a288 、https://github.com/llvm/llvm-project/pull/92926 、https://github.com/llvm/llvm-project/pull/152744 、https://github.com/llvm/llvm-project/pull/122698 、https://github.com/llvm/llvm-project/commit/13e32a8a3c95b23af51f081865db1bd259d269a4 、https://github.com/llvm/llvm-project/commit/787eeb8597fac22decb366a42176b11f52ec1bf0 、https://github.com/llvm/llvm-project/commit/c705b7b04dba467a67871a1bbb77907d0ed7fc19

## Phase 5 — Verification forecast / 验证预测：Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)
- 应消失/缩小（锚定 Phase 3(a) 引用行）：`slli a5,a4,0x2`@82b6ea、`add a5,a5,a4`@82b6ee、`add a5,a5,s11`@82b6f0、`slli a5,a5,0x2`@82b6f2、`add a5,a5,s6`@82b6f4 五条索引链收敛为 3 条；`slli a5,s1,0x2`@82b8c6、`add a5,a5,s1`@82b8ca、`slli a5,a5,0x2`@82b8cc、`add a5,a5,s0`@82b8ce 收敛为 2 条；本函数 instructions 计数下降。
- 应出现（锚定 `patterns/isa_extension_specific_instruction_substitution.md` §Verification）：重建后 objdump/annotate 中 `sh2add`/`shNadd` 出现在索引链位置；`pause`（`.word 0x0100000f`）仅在 feature gate 开启时出现于自旋路径；OpenJDK handshake/ThreadStartJoin 回归测试通过；libjvm 二进制 `readelf -A` 的 `Tag_RISCV_arch` 出现 `zbb`/`zba`。
- 重建与重测：以含 `zba zbb zbs zicond zihintpause` 的 `-march` 重建 libjvm，重跑 java.lang.Thread benchmark perf annotate；若索引链指令未消失或引入回归（如 toolchain 不支持 gate），该 fix 失败。

## Phase 6 — Completion check / 完成自检
| # | Check item | Status & Anchor |
|---|---|---|
| 1 | 承诺声明兑现：1 组 Phase 3–5 全部出现 | ✅ 1/1 组；`Handshake::execute(HandshakeClosure*, ThreadsListHandle*, JavaThread*)` |
| 2 | Phase 1 输出要求：7 行 baseline + 2 个 L0 gate + bound gate + 扩展 mismatch finding | ✅ 7 行结论齐（含 `Sampling IP precision` 行）；gap 标签：`baseline_gap: sampling metadata`、`baseline_gap: sampling IP precision`；L0 finding：hardware 有 zba/zbb/zbs/zicond/zihintpause 而 build 缺 |
| 3 | Phase 3 输出要求：8 项 Class selection trace + Classes scanned + 1 个顶层 finding + 三件套 + 负向微观分析 | ✅ 8 项 trace；`Classes scanned: triggers/rows-codegen.md, triggers/rows-vectorized-tuning.md`；顶层 finding 1（RISC-V ISA Extension-Specific Instruction Substitution）；evidence 锚点 `24.77 : 82b706: andi a5,a5,1`、`10.09 : 82b6f8: addiw a4,a4,1`、`0.00 : 82b6ea: slli a5,a4,0x2`、`4.59 : 82b8c6: slli a5,s1,0x2`；supporting 0；排除 7 条 row；推导式 1 条（route High / impact Medium） |
| 4 | Phase 4 输出要求：已读 pattern `isa_extension_specific_instruction_substitution.md`；命中 row RISC-V ISA Extension-Specific Instruction Substitution；引用短语首词「Excessive instruction count」「Extended latency」「Scaled address generation (Zba)」；The fix before/after、correctness、风险、Profile 信号齐；Related PRs：81 条 URL | ✅ |
| 5 | 路径合规：入口模式 A；class 集合 8 项 trace；单 finding 无仲裁冲突；L0 扩展 mismatch 未停扫；leaf 来自通过 gate 的 row | ✅ 模式 A + `rows-codegen.md`/`rows-vectorized-tuning.md` + 8 项 trace |
| 6 | Phase 5 两侧锚定：消失侧对 `slli a5,a4,0x2` @82b6ea / `slli a5,s1,0x2` @82b8c6 等；出现侧标注 `patterns/isa_extension_specific_instruction_substitution.md` §Verification | ✅ |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；交付止于证据+蓝图+The fix+验证预测 | ✅ |

修正记录：无