Functions under analysis: [pq_getmsgstring]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`030-pq_getmsgstring-annotate.txt`，event=`cpu-clock`，140 samples，percent type=`local period`，覆盖 336b36–336bc0 全函数含 hot loop body）。
- perf stat（可选 bound/context）：已提供（`19-postgresql-pgbench-benchmark-riscv-select_only_clients_16.txt`，含 cycles/instructions/cache/branch/L1/LLC counters）。
- workload/binary/DSO/source context：已提供（postgresql PIE，symbol `pq_getmsgstring`，annotate 头含 source line 583–598，对应 `src/common/pqformat.c`；函数为 PostgreSQL 前端/后端协议 `StringInfo` 字符串读取）。
- readelf -A / build ISA：已提供（metadata `binaries.postgres-elf-h` = "ELF64 little-endian PIE; Machine: RISC-V; Flags: 0x5, RVC, double-float ABI"；annotate 出现 `vsetvli/vle8ff.v/vmseq.vi/vfirst.m` 等 `v*` 指令，直接证明热点 object 的 build 含 `v`）。
- hardware ISA：已提供（metadata `cpuinfo.isa` = `rv64imafdcv_zicbom_..._zve32f_zve64d_...`，vendor SOPHGO / SG2044 / XuanTie C920v2，RVV 1.0）。
- vlenb：已提供（metadata `vector.vlen_bits=128, vlenb=16`）。
- 采样元数据：已提供（event=`cpu-clock`，`perf_record_frequency=99`，percent type=`local period`，call graph=`none`，单次运行；140 samples）。

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | RVV 1.0 `v`（含 `zve32f/zve64d/zve64x` 等）；SOPHGO SG2044 / T-Head C920v2，OoO（见 `references/core-profiles.md` SG2044 行）。无 vector flavor mismatch。 |
| Build ISA | 热点 object 含 `v`（annotate 见 `vsetvli/vle8ff.v/vmseq.vi/vfirst.m`）；ELF Flags 0x5 = RVC + double-float ABI。 |
| Vector flavor | RVV 1.0 `v*` mnemonic（非 `th.v*`）；无 `th.v*`，无 mismatch。 |
| VLEN | `vlenb=16` → VLEN=128 bits。 |
| Bound type | 全 workload IPC=0.431（整体偏 memory-bound，L1 load miss 4.613%）；本函数 hot loop 内 `vle8ff.v` 占 0.00%（数据访存近零样本）而 `vmseq/csrr/vfirst/bltz` config+reduction 指令主导 → 本函数 hot interval 属 control/config-bound，非 memory-bound。函数级 bound counter 缺失。 |
| Sampling semantics | event=`cpu-clock`（可解释为时间）；percent type=`local period`（函数内局部，非 global）；同一运行窗口；函数 workload 级贡献未知 → 收益上界只能表述为函数内局部份额，不得称 workload 级 Amdahl 上界。 |
| Sampling IP precision | `baseline_gap: sampling IP precision`；`cpu-clock`@99Hz，无 `precise_ip`/Exact-IP 记录，OoO skid 明显（`336b38: ld a4,0(a0)` 单次执行却占 20.00%，为循环 skid 证据）。单行只锚定 loop interval，不单指令归因。 |

L0 baseline gate：
- hardware 有 `v`，build 也有 `v`（annotate `v*` 直接证据）→ 无 hardware/build mismatch。
- annotate 无 `th.v*`，向量 flavor 为 RVV 1.0，无 `vector_flavor_mismatch`；无 `th.v*` gate 触发。

Bound-type gate：全 workload IPC=0.431 属 memory-bound 整体，但本函数 hot interval 的 `vle8ff.v` 0.00% 样本证明数据访存非主导，config/reduction 指令主导 → 本函数按 control/config-bound 处理，LMUL/loop-overhead 是候选杠杆。impact confidence 因缺函数级 bound counter 与 `baseline_gap: sampling IP precision` 下调（不因 route 的直接 provenance 提升 impact）。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（单函数 `pq_getmsgstring`）。hot interval 边界：`336b40`（`vsetvli`）–`336b56`（`bltz`）的 vector sentinel-scan 循环；循环前 `336b38: ld a4,0(a0)`（20.00%）与 `336b3a: li a3,0`（0.71%）为 skid 证据，并入同一 interval 成本。

trace anchor 最高行（cpu-clock, local period）：
- `17.86 : 336b56: bltz a6,336b40 <pq_getmsgstring+0xa>`（回边分支，最高占比）
- `15.71 : 336b52: vfirst.m a6,v1`
- `15.00 : 336b4a: vmseq.vi v1,v1,0`
- `12.86 : 336b4e: csrr a3,vl`
- `4.29 : 336b40: vsetvli a2,zero,e8,m1,ta,ma`
- `20.00 : 336b38: ld a4,0(a0)`（skid）

Sampling IP precision 不足 → 不单指令归因，结论锚定 loop interval（config/reduction 主导）这一 mechanism，而非单条指令 latency。

## Phase 3 — Pattern scan / 模式扫描：pq_getmsgstring

### Class selection trace（8 项）

1. `rows-asm.md` — **exclude**：当前代码来源是 compiler/libc-generated 的 RVV strlen（inlined 进 `pq_getmsgstring`），无手写 `.S` 的 source/DWARF/object-mapping 证据；symbol 为 C 函数。
2. `rows-operator-rvv.md` — **exclude**：热点已向量化（有 `v*`），且语义是 sentinel scan（strlen）而非尚未向量化的 elementwise/matmul/activation 等算子合同；已向量化的 string 语义归 string-memory/vectorized-tuning 类评估。
3. `rows-string-memory.md` — **include（逐行评估，未命中顶层）**：热点语义是 sentinel scan（strlen）。`rvv_sentinel_scan_copy_implementation` 行要求 "scalar loop" 而本循环已含 `vle8ff.v`+`vmseq.vi`+`vfirst.m`（已向量化）→ 不命中；`glibc RVV IFUNC/ABI Integration` 行要求 IFUNC/fallback 不可达证据，而 strlen 已 inlined（无 resolver/fallback 不可达）→ 不命中；copy/fill/compare/checksum/back-reference/word-wide 各行与 sentinel-scan 语义互斥 → 不命中。
4. `rows-vectorized-tuning.md` — **include（必选，命中）**：热点已有 `v*` 且非 `.S`，修正对象是 RVV 配置/LMUL。命中 `RVV Register-Group Utilization and LMUL Sizing` 行。
5. `rows-codegen.md` — **exclude（逐行评估）**：无分派可达性/跨 pass 流量/JIT/同步实现/indvar 信号；`vsetvli` 每迭代重置 vl 是 `vle8ff.v` fault-only-first 后 vl 可能收缩的语义需要，非可消除的重复 vtype 建立（`rvv_vector_state_management` 不命中）；happy path 已正确 tail-call（无栈帧），错误路径独立栈帧，非 code-layout 问题。
6. `rows-offload.md` — **exclude**：目标无矩阵引擎/packed-SIMD（P/DSP）卸载语义；这是普通 strlen sentinel scan。
7. `rows-crypto.md` — **exclude**：无密码学原语（AES/SHA/SM3/SM4/GHASH/CRC/GF(2^k)）。
8. `rows-runtime-os.md` — **exclude**：非 RTOS/kernel 侧 timer/ISR/CSR 热点。

Classes scanned: `rows-vectorized-tuning.md`（全读）、`rows-string-memory.md`（全读）；其余 6 个 class 逐项记录 exclude 理由如上。

### Local performance pattern scan: `pq_getmsgstring`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Register-Group Utilization and LMUL Sizing（primary） | 已向量化 sentinel-scan 循环 `vsetvli a2,zero,e8,m1,ta,ma` + `vle8ff.v`+`vmseq.vi`+`csrr vl`+`vfirst.m`+`bltz`；LMUL=m1 → VLMAX=16B/iter；peak live=1（仅 `v1`）；loop/config 指令（vmseq+csrr+vfirst+bltz ≈ 61%）主导样本 | High | Medium | `patterns/rvv_register_group_utilization.md` |

#### 三件套（primary finding）

**(a) 逐字 evidence 引用**（所属 interval：`336b40`–`336b56` 循环体）：
- `17.86 : 336b56: bltz a6,336b40 <pq_getmsgstring+0xa>`
- `15.71 : 336b52: vfirst.m a6,v1`
- `15.00 : 336b4a: vmseq.vi v1,v1,0`
- `12.86 : 336b4e: csrr a3,vl`
- `4.29 : 336b40: vsetvli a2,zero,e8,m1,ta,ma`
- `7.86 : 336b44: add a1,a1,a3`
- `0.00 : 336b46: vle8ff.v v1,(a1)`（数据访存近零样本，证明非 memory-bound）
- `20.00 : 336b38: ld a4,0(a0)`（循环前一行，skid）

**(b) 互斥邻居排除**：
- `Maximal LMUL for Misaligned Byte Processing`：行内 gate 明确限定 "Go byte-processing"，目标为 C（PostgreSQL）→ 排除。
- `RVV Sentinel Scan/Copy Implementation`：该行要求 "scalar loop"，本循环已含 `vle8ff.v`/`vfirst.m`（已向量化）→ 排除。
- `No vectorization`：已向量化（有 `v*`），方向相反 → 排除。
- `Register-Budgeted RVV Loop Unrolling`：当前 LMUL=m1 远低于合法 frontier（m8），应先按 register-group 行提高 LMUL 减少迭代；unroll 仅在 LMUL 已达最优边界后考虑（arbitration 判定）→ 本轮由 register-group 行认领。
- `RVV Vector-State Management`：`vsetvli` 每迭代重置 vl 是 `vle8ff.v` fault-only-first 后 vl 收缩的语义需要，非可消除的重复 vtype 建立 → 排除。
- assembly rows（`riscv-assembly-kernel-performance-optimization` 等）：非 `.S` → 排除。

**(c) 双 Confidence 推导式**：
- route: 当前代码来源=compiler/libc-generated RVV（直接证据：annotate `vsetvli/vle8ff.v/vfirst.m`，无 `.S` provenance）；已向量化 gate 成立（`v*` 存在）；LMUL=m1 < 合法候选 frontier（m8），peak live=1（`v1` 单寄存器组，`8×1=8 ≤ 32`）；loop/config 开销主导（vmseq 15.00 + vfirst 15.71 + csrr 12.86 + bltz 17.86 ≈ 61.43%）；互斥排除齐备 → **High**。
- impact: sample share（函数内局部，loop body ≈73.58% + skid ≈20.71% ≈ 94%）与 VLEN=128 已提供；但 percent=local-period（函数 workload 贡献未知 → 无 Amdahl 上界）、`baseline_gap: sampling IP precision`（skid）、函数级 bound 未知、pattern 明确"更大 LMUL 不必然更快" → **Medium**。

#### 多候选仲裁小段

单顶层 finding（primary），无 supporting / companion / independent。Evidence mechanism layer 归属：**L3（vector/runtime configuration — LMUL/register-group 选型，`rvv_register_group_utilization.md`）**。归属按地址分账：`336b40`–`336b56` 循环体的 sample 归 primary；`336b38: ld a4,0(a0)`（20.00%）与 `336b3a: li a3,0`（0.71%）为 skid，并入同一 interval 成本；`336b5a: add a1,a1,a6`（1.43%）为循环尾收尾，同 interval。入口条件 A：evidence sample share 加总（cpu-clock, local period）= 4.29+7.86+15.00+12.86+15.71+17.86+1.43 = 75.01%（loop body）+ 20.00%（skid `ld`）+ 0.71%（skid `li`）≈ 95.7% 函数内局部份额。

## Phase 4 — Root-cause blueprint / 根因蓝图：pq_getmsgstring

命中 row：`RVV Register-Group Utilization and LMUL Sizing`（`patterns/rvv_register_group_utilization.md`）。

1. **Root cause**：`pq_getmsgstring` 内 inlined 的 RVV `strlen` sentinel-scan 循环使用 `vsetvli a2,zero,e8,m1,ta,ma`（LMUL=1）。在 VLEN=128 下 VLMAX = 16 bytes，每轮只扫描 16 字节，导致 loop-config 与 mask-reduction 开销（`vsetvli`、`vmseq.vi`、`csrr vl`、`vfirst.m`、`bltz`）按 ≈N/16 次迭代重复，成为函数内 cpu-clock 局部份额的主导（loop body ≈75% + skid ≈20%）。依据 `patterns/rvv_register_group_utilization.md` §Why this is slow 第 1/2 点：`"Underutilized register-group frontier"`（当前 LMUL 小于合法无 spill 候选边界，单次迭代有效元素数偏低）与 `"Loop/config overhead dominates mid-size inputs"`（小 LMUL 让中等长度输入需要更多次迭代，每次都重复 `vsetvli` 评估、指针算术和分支）。关键证据：`vle8ff.v` 占 0.00% 而 `vmseq/vfirst/bltz` 合计 ≈48.57%——数据访存近零成本，配置与 reduction 指令才是主导，说明这是 LMUL/loop-overhead 问题而非 memory-bound 问题。

2. **The fix / 修复方式**：把 sentinel-scan 循环的 LMUL 从 `m1` 提高到寄存器预算允许的最大合法值（候选 `m2/m4/m8`，peak live vector = 1 个寄存器组 `v1`，`LMUL × peak_live_vectors ≤ 32` 在 m8 下 `8×1=8` 仍满足）。修正对象是 LMUL/每轮覆盖量，不是把 scalar 循环向量化（已向量化），也不改变 strlen 语义。

```asm
; Before（当前生成代码形态，e8,m1，每轮 16B）
336b40: vsetvli a2, zero, e8, m1, ta, ma   ; VLMAX = 16
336b44: add     a1, a1, a3                 ; str + offset
336b46: vle8ff.v v1, (a1)                  ; 最多 16B fault-only-first
336b4a: vmseq.vi v1, v1, 0                ; NUL mask
336b4e: csrr    a3, vl
336b52: vfirst.m a6, v1                   ; 第一个 NUL 下标
336b56: bltz    a6, 336b40                ; 无 NUL 继续

; After（候选形态：e8,m8，每轮最多 128B；仅示意，LMUL 由预算+benchmark 选定）
vloop:
    vsetvli  t0, zero, e8, m8, ta, ma     ; VLMAX = 128
    add      a1, a1, a3
    vle8ff.v v8, (a1)                     ; 最多 128B fault-only-first
    vmseq.vi v8, v8, 0
    csrr     a3, vl
    vfirst.m a6, v8
    bltz     a6, vloop
```

- 适用前提：RVV 1.0、VLEN=128（实测）；peak live vector 保持 1 个寄存器组，无 spill。
- Correctness contract（不可破坏）：`vle8ff.v` fault-only-first 语义保证越过 NUL 后的不可读内存（页边界）不会 fault，`vl` 收缩到已成功加载的前缀，strlen 结果 = `offset + vfirst.m` 不变；不改变返回值语义（`slen`）、不改变 `msg->cursor` 前进量（`slen+1`）、不改变越界检查（`cursor+slen >= len` → error）。短输入（<16B）保留 scalar/fast path，避免大 LMUL 的 setup 成本大于收益。
- 限制/风险：更大 LMUL 不必然更快——必须核对（i）C920v2 上大 LMUL `vle8ff.v` fault-only-first 的微架构成本（fault-first 机制在 m8 下需检查更大地址范围/跨 cache-line）；（ii）单轮 `vle8ff.v` 读取 128B 会覆盖 NUL 之后的字节（虽不 fault，但增加单次访存足迹）；（iii）短字符串（<16B）场景 m1 已够，m8 反而多读内存。因此只建议在中等/长字符串（SQL query 文本典型 40–80B）场景下提升，最终 LMUL 由 `m1/m2/m4/m8` 候选 A/B benchmark 选定。
- 修复后预期 Profile 信号：`vsetvli … e8,m8`（或 m4/m2）出现；loop 迭代数下降；`vmseq.vi`/`csrr vl`/`vfirst.m`/`bltz` 的函数内局部样本占比缩小；`vle8ff.v` 单轮覆盖字节数从 16 提升到 32/64/128。

3. **Baseline facts 回填**：hardware ISA = RVV 1.0 `v`（C920v2 OoO）；build ISA = 含 `v`（`v*` 指令直接证据）；VLEN = 128 bits（vlenb=16）；bound type = 本函数 hot interval 属 control/config-bound（`vle8ff.v` 0.00%，config/reduction 主导），全 workload IPC=0.431（memory-bound 整体，非本函数主导）。

4. **收益上界**：当前 sampled event（`cpu-clock`, local period）下函数内局部样本份额，loop body ≈75.01% + skid ≈20.71% ≈ 95.7%。**不构成 workload 级 Amdahl 上界**（percent type=local-period、函数 workload 级贡献未知，采样语义四条不满足）。

5. **三维路由判定**：
   - current source：compiler/libc-generated RVV（strlen 已 inlined 进 `pq_getmsgstring`；无 `.S` provenance，证据 = annotate `vsetvli/vle8ff.v/vfirst.m` + source line 595 `slen = strlen(str)`）。
   - implementation existence/reachability：当前已在执行 RVV 路径（无需新增实现或 dispatch），非 missing-kernel / kernel-selection 场景。
   - function-level policy：无要求独立 `.S` 载体的 policy（非 missing `.S` 分支），不进入 policy/existence 四证。

6. **Implementation-shape proof**：不适用（非 policy-backed missing `.S` 分支）。

7. **Related PRs**（pattern `rvv_register_group_utilization.md` §Related PRs，14 条 URL）：
   - OpenCV: [#26318](https://github.com/opencv/opencv/pull/26318)
   - OpenCV: [5be158a2b6ed](https://github.com/opencv/opencv/commit/5be158a2b6ed0f4f4def851d3a57c3e3b5865ad5)
   - OpenCV: [#25586](https://github.com/opencv/opencv/pull/25586)
   - Linux Kernel RISC-V: [a4348546332c](https://github.com/torvalds/linux/commit/a4348546332c9fae12b29acb514535e0a52b9b3c)
   - Linux Kernel RISC-V: [a894e8ed09c6](https://github.com/torvalds/linux/commit/a894e8ed09c6c7fa239711819db83b8c050eb7b0)
   - Linux Kernel RISC-V: [c2a658d41924](https://github.com/torvalds/linux/commit/c2a658d419246108c9bf065ec347355de5ba8a05)
   - OpenJDK: [bdd37b0e5eaa](https://github.com/openjdk/jdk/commit/bdd37b0e5eaa984e2ad2e9010af37dcd612cc05e)
   - OpenBLAS: [cc1b5794a040](https://github.com/OpenMathLib/OpenBLAS/commit/cc1b5794a0409493ed470f4cced313ea0b560870)
   - OpenBLAS: [d69be17b6ff7](https://github.com/OpenMathLib/OpenBLAS/commit/d69be17b6ff7eea5371b03a199db9c112aa6dc4b)
   - OpenBLAS: [4a12cf53ec11](https://github.com/OpenMathLib/OpenBLAS/commit/4a12cf53ec116c06e5d74073b54a3bca6046cb17)
   - OpenBLAS: [240695862984](https://github.com/OpenMathLib/OpenBLAS/commit/240695862984d4de845f1c42821a883946932df7)
   - V8: [384433993606](https://github.com/v8/v8/commit/3844339936068c529170dcb4f2aa160654d25943)
   - vLLM: [#47538](https://github.com/vllm-project/vllm/pull/47538)

## Phase 5 — Verification forecast / 验证预测：pq_getmsgstring

primary finding（`rvv_register_group_utilization.md`）验证预测：

- **应消失/缩小侧**（锚定 Phase 3(a) 引用行）：`336b40: vsetvli a2,zero,e8,m1,ta,ma` 的 `m1` 应变为 `m4/m8`；`336b56: bltz a6,336b40`、`336b52: vfirst.m a6,v1`、`336b4a: vmseq.vi v1,v1,0`、`336b4e: csrr a3,vl` 的函数内局部样本占比应显著缩小（迭代次数下降 → reduction/branch/config 指令执行次数下降）。
- **应出现侧**（锚定 `patterns/rvv_register_group_utilization.md` §Verification）：`vsetvli … e8,m8`（或 m4/m2，按候选 frontier 选定）出现在主循环；无新增 `vlmul_ext/vlmul_trunc`/转换；无 vector spill/reload；loop 迭代数下降、`vle8ff.v` 单轮覆盖字节数提升。
- **正确性对照**：对相同输入（含空串、单字符、NUL 在首/中/尾、跨 16/32/64/128B 边界、未对齐 `msg->data+cursor` 地址、页边界 placement）比较 m1 与 m4/m8 的 `slen` 结果逐字节一致；确认 fault-only-first 在页边界前正确截断 `vl`。
- **长度/VLEN 无关性**：覆盖 0/1/短/中/长字符串与 tail，不写死 lane 数；在可获得的其它 VLEN（≥128）上复验。
- **Benchmark**：对真实 pgbench select-only workload（短/中/长 query 文本）用 `benchstat` 对比 m1 vs m4/m8，确认 `pq_getmsgstring` 的 `cpu-clock` 局部样本份额与 latency 下降；不默认 m8 最优，按候选 frontier（`m1/m2/m4/m8 × unroll=1`）实测。
- **实现可达性**：通过 symbol/sample 确认真实 workload 仍走 RVV strlen 路径（无 scalar fallback 回退）。
- 门槛升级说明：本函数结论已为 profile-backed（入口 A）；若需函数级 workload 贡献，需补采 `--percent-type=global-period` 的 annotate 以确认函数级全局份额。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：1/1 组；`[pq_getmsgstring]` | ✅ |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + 2 个 L0 gate + bound-type gate + `baseline_gap: sampling IP precision`） | ✅ |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace；`Classes scanned: rows-vectorized-tuning.md, rows-string-memory.md`；顶层 finding 数=1；evidence 锚点 `17.86: 336b56 bltz`/`15.71: 336b52 vfirst.m`/`15.00: 336b4a vmseq.vi`；supporting 0；排除 6 条；推导式 2 条） | ✅ |
| 4 | Phase 4 输出要求满足（已读 `patterns/rvv_register_group_utilization.md`；命中 row 同名；引用 `"Underutilized register-group frontier"`/`"Loop/config overhead dominates mid-size inputs"`；The fix before/after + correctness + 风险 + Profile 信号；Related PRs：14 条 URL） | ✅ |
| 5 | 路径合规（8 项 trace 可解释；零/多命中、L3 归属合规；入口 A 按动态份额排序；无 `th.v*` 停扫） | ✅ |
| 6 | Phase 5 两侧锚定（消失侧对 Phase 3(a) `336b40/336b56/336b52/336b4a`；出现侧对 §Verification） | ✅ |
| 7 | 契约边界合规（无实施询问/代码修改/补丁生成/契约外实施分支；无向用户追问） | ✅ |

修正记录：无
