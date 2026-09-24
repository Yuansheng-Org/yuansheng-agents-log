Functions under analysis: [crypto/internal/fips140/sha256.block.abi0]（1 个）→ 本输出含 1 组 Phase 3–5

# RISC-V 性能诊断报告：crypto/internal/fips140/sha256.block.abi0

（批次 batch-001 / function batch-001-function-001 / rank 001；run_id trace-20260910T061316Z-6646b60a；software=go；testcase=BenchmarkAddMulVVWW_words_100000_impl_asm）

## Phase 0 — Evidence inventory / 证据清单

- 单函数完整 annotate：已提供（`001-crypto／internal／fips140／sha256.block.abi0-annotate.txt`，149,307 字节，3,857 行，覆盖 0x536418–0x539b38 全函数：prologue、全部 64 轮展开体、W schedule、epilogue、回边 0x539b2e）
- perf stat（可选 bound/context）：已提供（`2-go-benchmark-riscv-BenchmarkAddMulVVWW_words_100000_impl_asm.txt`，RISC-V 侧计数：IPC 0.776、cpu_cycle 7,894,466,427、instruction 6,127,841,358、branches 424,843,559、branch_misses 20,607,457、L1_dcache_loads 1,158,850,266、L1_dcache_load_misses 14,311,913、LLC_loads 210,614,900、LLC_load_misses 25,918,181、ns_per_op 301,056、264 samples 记录）
- workload/binary/source context：已提供（software_name=go，repository https://github.com/golang/go.git，branch master，commit 82215dc6c01ed6efb6f4f236cd5c87ceb6dc318a；binary 为 riscv64 ELF EXEC；函数为 Go 标准库 crypto/internal/fips140/sha256 的 block 压缩函数，Go ABI 前缀可见）
- readelf -A（热点 object 的 Tag_RISCV_arch）：缺失（详见 Phase 1）
- hardware ISA（/proc/cpuinfo 或 riscv_hwprobe）：已提供（metadata cpuinfo isa：`rv64imafdcv_zicbom_zicboz_zicntr_zicond_zicsr_zifencei_zihintntl_zihintpause_zihpm_zawrs_zfa_zfh_zfhmin_zca_zcb_zcd_zba_zbb_zbc_zbs_zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_sstc_svinval_svnapot_svpbmt`；含 `v`、Zba/Zbb/Zbc/Zbs，无任何 `zk*`/`zvk*`）
- `vlenb`：已提供（vector.vlenb=16 → VLEN=128 bits；flavor=RVV 1.0）
- 采样元数据（event / percent type / scope / 窗口）：已提供（event=cpu-clock，19 samples，`percent: local period`，单次运行窗口；annotate 由 container-v4 top-30 workflow 于 2026-09-08 基于保留的 965-case 数据集再生成，benchmark 未重跑——见 metadata warnings）
- Sampling IP precision（precise_ip / Exact-IP / skid）：缺失（详见 Phase 1）

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | 已提供：C920v2（XuanTie，mvendorid 0x5b7，out-of-order）暴露 `v`（RVV 1.0, VLEN=128, zve64d）与 Zba/Zbb/Zbc/Zbs、Zicond、Zfa、Zfh；**无任何 `zk*`/`zvk*`（无 Zvknha/Zvknhb 向量 SHA-256 crypto，无 Zvbb，无 Zksh 标量 SHA-256）** |
| Build ISA | `baseline_gap: build ISA`；未提供热点 object 的 `readelf -A` arch 字符串。可选命令：`readelf -A <go binary>`。执行代码内 3,857 行零 `v*`，可确认当前构建未向该函数发出任何向量指令 |
| Vector flavor | annotate 全 scalar（零 `v*`、零 `th.v*`），无 flavor mismatch；硬件为 RVV 1.0 |
| VLEN | 已提供：vlenb=16 → VLEN=128 bits；SEW=32 时 VLMAX(LMUL=1)=4 lanes |
| Bound type | 全运行 IPC=0.776；L1-dcache load miss ≈1.2%（14.3M/1158.9M）、LLC load miss ≈12.3%（25.9M/210.6M）、branch miss ≈4.85%（20.6M/424.8M）；注意 `cache_references≈cache_misses`（210,602,294 vs 210,604,389）为该 PMU 的计数一致性问题，不作过度解读。对本函数：K 表（256B rodata，s2 基址 + 立即数偏移 0..252）与 W 缓冲（栈 64B）均 L1 驻留，非 memory-bound；轮函数为串行加法链（T1 = H+W+K+Σ1+Ch 四条依赖 add）→ 判定 compute/latency-bound |
| Sampling semantics | event=cpu-clock（时间可解释）；percent-type=local period（annotate 头部原文 `percent: local period`）；同一保留窗口（264-sample 记录中的 19 个 sample）；函数级 workload 贡献未知（benchmark 本体是 math/big addMulVVW asm，sha256.block 的 19 个 sample 大概率来自二进制启动期的 crypto/FIPS140 自检路径，调用栈未保留，无法证实）→ 仅可表述局部样本份额，**禁止 workload 级 Amdahl 上界** |
| Sampling IP precision | `baseline_gap: sampling IP precision`；无 precise_ip/Exact-IP 信息，单行 5.26% 只锚定 basic block / loop interval，不作单指令 latency 归因 |

L0 baseline gate #1（hardware 有 `v` 而执行代码无 `v`）：**成立** —— 硬件暴露 RVV 1.0，但本函数 3,857 行反汇编零 `v*`。作为最高优先级 baseline finding 报告（前置所有 pattern），继续扫描。
L0 baseline gate #2（`th.v*` flavor gate）：不适用（无 `th.v*`）。
Bound-type gate：compute/latency-bound，非 memory-bound；本地 RVV/指令替换 fix 不被 bound 类型反驳。

## Phase 2 — Scope / 分析边界

函数清单与承诺声明一致（1 个）：`crypto/internal/fips140/sha256.block.abi0`。

- 函数地址区间：0x536418–0x539b38（约 13.8 KB，≈3,520 条指令，全部 4 字节编码、无压缩指令）。
- hot loop / loop interval：消息字装载与 64 轮完全展开的压缩体 0x536476–0x539b2e（每处理一个 64B 消息块后经 `539b2a: beq t3,t4,539b32` 与 `539b2e: j 536476` 回边再进入下一块）。annotate 覆盖完整（含 prologue、epilogue、回边）。
- 区间锚点（trace anchor）：19 个 sample 以 **5.26%（每行 1 个）** 均匀散布于整个展开体（如 `5.26 : 53690c: or s0,t6,s0`、`5.26 : 537526: add a1,a1,s0`、`5.26 : 538314: slliw s0,a3,0xa`、`5.26 : 5393a2: srliw t6,t0,0x11`、`5.26 : 539ae6: add a0,a0,t0`），prologue 0 sample。均匀散布表明成本分布于每轮展开体全区间，无单点热点。
- Sampling IP precision 不足 → 本报告全部归因停留在 interval-level（轮函数体 / schedule 段 / 消息装载段），不将单行 sample 升级为单指令 latency 根因。
- 函数结构事实（供 Phase 3 引用）：
  - 消息装载段（rounds 0–15，每轮一条 32-bit BE 消息字）：`536476: lbu t0,0(t4)` `53647a: lbu t1,1(t4)` `53647e: lbu t2,2(t4)` `536482: lbu s0,3(t4)` `536486: slli t0,t0,0x18` `536488: slli t1,t1,0x10` `53648a: or t0,t1,t0` `53648e: slli t2,t2,0x8` `536490: or t0,t2,t0` `536494: or t0,s0,t0` `536498: sw t0,0(s3)` —— 每条 BE 消息字 10 条指令（4×lbu + 3×slli + 3×or），16 条字共 160 条指令。
  - 压缩轮（每轮）：Σ1 三条旋转三元组（如 `5364a2: srliw t6,a4,0x6` / `5364a6: slliw t1,a4,0x1a` / `5364aa: or t1,t6,t1`，rotr 6；同样形态 rotr 11、rotr 25）+ 2 xor；Ch 三指令 `xor/and/xor`；4 条依赖 add（`5364a0: add a7,a7,t0` → `5364ae: add a7,a7,s0`(K) → `5364d8: add a7,a7,t1`(Σ1) → `5364de: add t0,t0,a7`(Ch)）；Σ0 三条旋转三元组（`5364e0: srliw t6,a0,0x2` / `5364e4: slliw t1,a0,0x1e` / `5364e8: or t1,t6,t1` 等）；Maj 三指令；K 常量装载 `lwu s0,N(s2)`（N=0..252 立即数折叠）。
  - W schedule（rounds 16–63，每轮）：4×`lwu` 栈装载（如 `536f3e: lwu t0,56(s3)`=W[t-2]、`536f42: lwu t1,4(s3)`=W[t-15]、`536f46: lwu s1,36(s3)`=W[t-7]、`536f4a: lwu s5,0(s3)`=W[t-16]）+ σ1（`536f4e: srliw t6,t0,0x11` / `536f52: slliw t2,t0,0xf` / `536f56: or`（rotr17）、`536f5a: srliw t6,t0,0x13` / `536f5e: slliw s0,t0,0xd` / `536f62: or`（rotr19）、`536f66: srli t0,t0,0xa`（shr10）、2 xor）+ σ0（同形 rotr7/rotr18/shr3）+ 3 add + 1×`sw`（`536f9c: sw t0,0(s3)`）；s3=sp+8 为 64B W 栈缓冲（round 窗口旋转复用）。
  - Epilogue：`539ad6` 起 8×`lwu`(digest) + 8×add + 8×`sw`，`539b26: addi t4,t4,64`，回边。

## Phase 3 — Pattern scan / 模式扫描：crypto/internal/fips140/sha256.block.abi0

### Class selection trace（8 项）

1. `rows-asm.md` — include — 当前代码为 compiler-generated（Go ABI prologue `536418: ld t1,16(s11)` + `jal t0,481d98 <runtime.morestack_noctxt-tramp3>` + Go 栈分裂检查；无 .S 风格手工调度），需逐条核对 policy-backed missing `.S` 四证与 hand-tuned-assembly 互斥。
2. `rows-operator-rvv.md` — include — compiler-generated scalar 主循环（零 `v*`），评估 no-vectorization 及全部更具体 semantic 行互斥。
3. `rows-string-memory.md` — include — BE 消息字经 4×lbu+shift+OR 拼装 = 固定宽度 logical access 的标量窄物化，评估 wide-scalar-memory-access / endianness-helper 行。
4. `rows-vectorized-tuning.md` — exclude — annotate 零 `v*`，无 RVV 配置/LMUL/unroll 修正对象。
5. `rows-codegen.md` — include — compiler-generated 指令形态：rotation 三元组（ISA-substitution 信号）、W 栈缓冲（register-pressure 信号）、frontend footprint（code-layout 信号）、K 装载（load/store addressing 信号）。
6. `rows-offload.md` — exclude — 无矩阵引擎/packed-SIMD；SHA-256 压缩不是 GEMM/权重重排。
7. `rows-crypto.md` — include — profile evidence 点名 SHA-256 原语：逐行核对 dedicated-vector-crypto / cipher-mode / carry-less / precomputed-LUT 四行的 hardware gate。
8. `rows-runtime-os.md` — exclude — 用户态 crypto，无 RTOS/kernel/timer/CSR 信号。

Classes scanned: `rows-asm.md`, `rows-operator-rvv.md`, `rows-string-memory.md`, `rows-codegen.md`, `rows-crypto.md`（已读全并逐行评估）；`rows-vectorized-tuning.md`, `rows-offload.md`, `rows-runtime-os.md` 按上述触发观察整组 exclude。

### Local performance pattern scan: `crypto/internal/fips140/sha256.block.abi0`

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| No vectorization (autovec gap / RVV kernel not built)（primary, L1） | hot main loop（0x536476–0x539b2e，3,857 行 annotate）全 scalar，zero `v*`/`th.v*`；hardware `isa` 含 `v`（RVV 1.0, VLEN=128）；Go tree 对 riscv64 无 sha256 block 专用 kernel（build-time GOARCH 文件选择，非 runtime dispatch） | High | Medium | `patterns/no-vectorization.md` |
| Register Pressure and Save/Restore（supporting of A） | W schedule 栈缓冲（s3=sp+8）每轮（rounds 16–63）4×`lwu` + 1×`sw`，48 轮/块；长生命周期 temporary 未寄存器化 | — | — | `patterns/register_pressure_and_save_restore.md` |
| RISC-V ISA Extension-Specific Instruction Substitution（independent, L4） | 每轮 Σ1/Σ0 共 6 个 `srliw+slliw+or` 旋转三元组（W schedule 再 +4 个）；BE 消息字 4×`lbu+shift+or` 拼装；HW `isa` 含 `zba zbb zbc zbs`（Zbb: roriw/rori/rev8） | High | Medium | `patterns/isa_extension_specific_instruction_substitution.md` |
| Wide Scalar Memory-Access Code Generation（supporting of B） | 同段 BE 消息字拼装（4-byte `lbu`+shift+OR），逻辑 32-bit word 本可一次 lwu 快照；source 语义（p[j..j+3] 字节读）允许合并 | — | — | `patterns/wide_scalar_memory_access_codegen.md` |

#### Finding A（primary, L1）— No vectorization / RVV kernel not built

**(a) 逐字 evidence 引用**（所属 interval：0x536476–0x539b2e 消息装载 + 64 轮展开压缩体）：
- `5.26 : 53690c: or s0,t6,s0`（轮 15 Σ0 旋转三元组之 or；1 sample）
- `5.26 : 537526: add a1,a1,s0`（轮 38 K 累加；1 sample）
- `5.26 : 5393a2: srliw t6,t0,0x11`（W schedule σ1 rotr17；1 sample）
- `5.26 : 539ae6: add a0,a0,t0`（epilogue digest 累加；1 sample）
- 全函数 3,857 行零 `v*`、零 `th.v*`；19 个 sample（5.26%×19）均匀覆盖展开体
- hardware：`isa: rv64imafdcv_..._zve64d_...`（含 `v`）；`vector: flavor RVV 1.0, vlen_bits 128, vlenb 16`
- 结构证据：消息字装载段（`536476: lbu t0,0(t4)`…`536498: sw t0,0(s3)`）、轮函数段（`5364a0: add a7,a7,t0` 起 4 条依赖 add）、W schedule 段（`536f3e: lwu t0,56(s3)` 起）
- 互斥邻居排除（(b)）：
  - dedicated-vector-crypto 行（rows-crypto）：行内 gate 要求 HW 暴露 Zvkned/Zvknha/Zvknhb/Zvksh/Zvksed/Zvkg 且 vlen>=128 —— HW isa 字符串 `rv64imafdcv_zicbom_..._zve32f_zve32x_zve64d_zve64f_zve64x_zvfh_zvfhmin_sscofpmf_...` 中**无任何 `zvk*`**，gate 不成立 → 不命中（也意味着本核无法走向量 SHA-256 专用轮指令 route）。
  - cipher-mode 行：要求"已用向量 crypto 指令仍逐块处理"——本函数 zero `v*`，前提不成立 → 不命中。
  - carry-less / precomputed-LUT 行：SHA-256 轮非 GF(2^k) 二元域乘法、非 CRC/table-reuse 语义 → 不命中。
  - policy-backed missing `.S` 行（rows-asm）：四证中"dispatch slot + 官方 policy 要求独立 .S"不成立（Go crypto 对 riscv64 无 runtime dispatch slot、无强制 .S policy，纯 build-time GOARCH 文件选择）→ 不命中；本行按 no-vectorization 认领。
  - hand-tuned-assembly 行：当前代码为 compiler-generated（Go ABI prologue/morestack），非手写 `.S` → 不命中。
  - operator semantic 行（elementwise/complex/activation/normalization/…）：SHA-256 轮为 8 状态字密码学轮函数，非任何 lane-independent 算子合同 → 整组不命中。
  - kernel-selection 行：仓库中不存在可被"错误未选中"的 riscv64 block kernel（无实现存在）→ 不命中。
- (c) 双 Confidence 推导式：`route: compiler-generated 纯 Go 标量主循环（Go ABI prologue 直接证据）+ hardware V（cpuinfo 直接证据）+ hot interval zero v*（annotate 全区间直接证据）+ 更具体 crypto row gate 全部失败 → High；impact: 函数级 19/19 局部样本份额 + VLEN=128 + compute/latency-bound 均到位，但 sampling IP precision 缺失（interval 级归因）、函数在本次 benchmark 中的角色为启动期 incidental（非稳态热点，调用路径未保留）→ Medium`。

#### Finding B（independent, L4）— RISC-V ISA Extension-Specific Instruction Substitution

**(a) 逐字 evidence 引用**（所属 interval：轮函数体 + W schedule 段 + 消息装载段）：
- 旋转三元组（Σ1 首轮）：`5364a2: srliw t6,a4,0x6` / `5364a6: slliw t1,a4,0x1a` / `5364aa: or t1,t6,t1`（rotr 6）；Σ1 同形 rotr 11（`5364b0: srliw t6,a4,0xb` / `5364b4: slliw t2,a4,0x15` / `5364b8: or t2,t6,t2`）、rotr 25（`5364bc/5364c0/5364c4`）
- Σ0 三元组：`5364e0: srliw t6,a0,0x2` / `5364e4: slliw t1,a0,0x1e` / `5364e8: or t1,t6,t1`（rotr 2）等 3 组
- W schedule σ1：`536f4e: srliw t6,t0,0x11` / `536f52: slliw t2,t0,0xf` / `536f56: or t2,t6,t2`（rotr 17）；`536f5a/536f5e/536f62`（rotr 19）；`536f66: srli t0,t0,0xa`（shr 10）；σ0：`536f74: srliw t6,t1,0x7` / `536f78: slliw t2,t1,0x19` / `536f7c: or`（rotr 7）、`536f80/536f84/536f88`（rotr 18）、`536f8c: srli t1,t1,0x3`（shr 3）
- BE 消息字拼装（10 条/字）：`536476: lbu t0,0(t4)` `53647a: lbu t1,1(t4)` `53647e: lbu t2,2(t4)` `536482: lbu s0,3(t4)` `536486: slli t0,t0,0x18` `536488: slli t1,t1,0x10` `53648a: or t0,t1,t0` `53648e: slli t2,t2,0x8` `536490: or t0,t2,t0` `536494: or t0,s0,t0` `536498: sw t0,0(s3)`
- hardware：`isa: ..._zba_zbb_zbc_zbs_...`（Zbb 提供 `roriw`/`rori`/`rev8`；Zbb 亦为 rva22u64 成员）
- 互斥邻居排除（(b)）：
  - native-word-size 行（rows-codegen）：该行认领"32-bit 运算后冗余 extension"；本段证据是旋转合成与字节拼装本身，非 load 后冗余扩展 → 不命中。
  - runtime-ISA-dispatch 行：本函数无 runtime dispatch 接线（build-time 代码生成），非"能力已接好但未到达入口" → 不命中。
  - dedicated scalar crypto（Zknh `sha256sum0/1`、`sha256sig0/1`）变体（行内 §典型场景）：HW isa 无 `zkn/zks/zksh`，单条标量 crypto 指令 route 的 hardware gate 失败；本行仅认领 Zbb `roriw`/`rev8` 子项。
  - endianness-helper 行（rows-string-memory）：行内互斥明文"byte-swap 算术可由 native instruction 替换 → ISA-substitution row"，直接把 BE 拼装证据转交本行。
  - code-layout/frontend 行：无 i-cache/frontend counter 证据（19 个 cpu-clock sample 不能证明 I-cache miss 成本）→ 不命中。
  - resource-aware-scheduling 行：缺目标 core 的 PMU/依赖-延迟资料，且根因是指令选择而非调度 → 不命中。
- (c) 双 Confidence 推导式：`route: 反汇编直接显示多指令合成序列（srliw+slliw+or 三元组、lbu 字节拼装）+ hardware isa 直接包含 zbb（roriw/rori/rev8 属 Zbb）+ 无更具体的 crypto 指令可用（无 zkn/zks）→ High；impact: 机制覆盖整个 64 轮体（约 39% 静态指令削减估计），19/19 局部份额同区间（与 A 不叠加），bound=compute/latency，但 sampling IP precision 缺失、函数为本次运行的 incidental 成本 → Medium`。

#### Multi-hit arbitration / 多命中仲裁

- Finding A（L1 vectorization）与 Finding B（L4 instruction substitution）为 **independent**：修复对象不同（A=新增 riscv64 RVV block kernel；B=Go 编译器 cmd/compile RISCV64 SSA/obj 规则或手写 riscv64 block 汇编改用 roriw/rev8）、验证方法不同（A=annotate 出现 `vsetvli`/`vle32` 等 `v*`；B=annotate 出现 `roriw`/`rev8`）、机制层面不同（L1 向量执行缺失 vs L4 标量指令选择）。两者共享同一 19-sample evidence 区间，sample share 不叠加、不互为因果；A 若实施（向量化 kernel），B 的标量序列随之消失（因果消除成立），但 B 拥有独立修复对象与验证方法，按仲裁规则保留为 independent。
- Supporting（计入各自 primary 下，不单列顶层、不计命中数）：register-pressure（supporting of A：A 的向量化 fix 会消除 W 栈往返 traffic，因果消除成立；自身 gate：`lwu/sw` 成组栈访问 + s3=sp+8 地址生成证据满足"长生命周期 temporary + 地址生成证据"）；wide-scalar-memory-access（supporting of B：同一 BE 拼装序列的 L2 物化视角，fix 与 B 相同为 lwu+rev8+srli）。
- 收益上界排序：A 与 B 的 evidence 均为同一函数体 19/19 局部样本份额（入口条件 A，但采样语义四条不全 → 仅局部份额，不得相加或外推 workload 级）；A 覆盖整个主循环（消息装载+轮+sched），B 覆盖轮与 schedule 内的指令选择子集 → A 的机制作用域更大，列前。
- 因果层次：A → L1（vectorization / semantic dispatch）；B → L4（compute / codegen micro-structure）；supporting 随各自 primary 层次。

## Phase 4 — Root-cause blueprint / 根因蓝图：crypto/internal/fips140/sha256.block.abi0

### Finding A（primary）— `patterns/no-vectorization.md`

1. **Root cause**：Go 的 `crypto/internal/fips140/sha256` 在 riscv64 无专用 block 实现，回退到纯 Go 通用实现，`block()` 的 64 轮压缩完全以标量指令执行（3,857 行 annotate 零 `v*`）。依据 `patterns/no-vectorization.md` §Why this is slow：RVV spec 定义「`VLMAX = LMUL × VLEN / SEW`，实际每次迭代的 `vl` 满足 `vl ≤ VLMAX`」——本核 VLEN=128、SEW=32 时 VLMAX(m1)=4，向量单元完全未参与 hot main-loop 的并行元素处理。硬件 gate：HW 暴露 `v`（RVV 1.0），且**无 `zvk*`**（`dedicated_vector_crypto_instructions` 行的向量 SHA-256 轮指令 route 不可用），因此可用向量化轴只能是 base-RVV（vle32/vadd/vxor/vsll/vsrl + vrgather 做端序交换）。
2. **The fix / 修复方式**（不直接编辑源码，作为诊断蓝图）：
   - 修复对象：为 `crypto/internal/fips140/sha256`（及标准库 `crypto/sha256`）新增 riscv64 专用 block 实现（Go 现有 per-arch 模式：`sha256block_<arch>.s` build-time 选择，与 amd64/arm64/ppc64le/s390x 对齐），而非修改通用标量代码。
   - 修复前（当前）：纯 Go 标量 64 轮，每轮 ≈34 条（Σ1/Ch/Σ0/Maj + 4 条依赖 add）+ K load；rounds 16–63 另 +≈18 条 W schedule（4 lwu + σ + 3 add + sw）。
   - 修复后（形状，非固定实现配方）：base-RVV kernel。因单消息块级串行（digest 链式依赖，块间无法并行），向量化轴分两层：
     a) **W schedule 向量化**：用 VLEN=128/SEW=32 一次计算 4 个 W[t]（vle32 装载窗口字 + `vsll.vi`/`vsrl.vi`/`vxor.vv` 做 σ0/σ1 旋转 + `vadd.vv`）；窗口滑动用 `vrgather.vv`（base V 指令）取 W[t-2]/[t-15] 等偏移元素；BE 消息字端序交换用固定索引 `vrgather`（无 Zvbb 时不能 `vbrev8`，但 base V 的 vrgather 可完成逐元素字节反转）。
     b) 轮函数：A 链与 E 链是两个独立串行 recurrence（E'=D+T1(E…)、A'=T1+T2(A…)），可打包为 2-lane 向量链（SEW=32、2/4 lanes 使用）并行推进；不做跨块并行（API 合同是单流块压缩）。
   - 适用前提：build 必须发出 `v*`（Go 侧需在 riscv64 + RVV 使能路径下编译该 kernel，非 V 构建走原标量 fallback）；无 `zvknha/zvknhb` 时旋转为 3 条（vsll+vsrl+vxor），收益来自 4-lane/2-lane 并行与 schedule 一次 4 字。
   - correctness contract：SHA-256 压缩结果必须 bit-identical（与 FIPS140 自检向量一致）；消息字按 big-endian 语义读取（vrgather 端序交换正确性）；digest 累加顺序不变；`block()` 的 p 指针推进与剩余长度语义不变。
   - 限制/风险：单流块间无并行 → 收益上限来自 schedule 向量化 + A/E 链打包（估计轮函数体指令削减 20–40%，非 4×）；多缓冲（4 独立消息并行）需新 API（如 multi-buffer hasher），超出本函数合同，仅记录不承诺；需在 in-order 与 OoO 两类核分别实机验证。
   - 预期 Profile signals：`sha256.block.abi0` annotate 出现 `vsetvli`、`vle32`、`vadd.vv`、`vxor.vv`、`vsll.vi`/`vsrl.vi`、`vrgather.vv`；标量指令不再主导 hot loop；同一函数重跑 annotate 后 19-sample 区间的指令组成改变。
3. **Baseline facts 回填**：hardware ISA=rv64imafdcv_…_zve64d（含 v，无 zvk*）；build ISA=`baseline_gap: build ISA`（readelf -A 未提供；当前代码零 v*）；VLEN=128（vlenb=16）；bound type=compute/latency（全运行 IPC 0.776 + 轮函数串行 add 链）。
4. **收益上界**：入口条件 A 局部样本份额 —— 该 finding 的 evidence 覆盖整个 hot interval（19/19 = 100% 局部份额；≈19/264 ≈ 7.2% 保留运行窗口，且该函数大概率是启动期 incidental 成本，workload 级 Amdahl 上界**不得**声明，采样语义四条不满足）。
5. **三维路由判定**：
   - current source：compiler-generated 纯 Go 通用代码（Go ABI prologue 直接证据：`536418: ld t1,16(s11)`、`jal t0,481d98 <runtime.morestack_noctxt-tramp3>`；非 `.S`）。
   - implementation existence/reachability：Go tree 对 riscv64 无 `sha256block_riscv64.s`/RVV kernel（build-time GOARCH 文件选择，无 runtime dispatch slot）→ 无"已存在未采用"问题；缺失实现。
   - function-level policy：Go crypto 采用 per-arch .s 但**不强制** riscv64 必须有独立 .S（policy/existence 四证中的 policy 与 dispatch-slot 两证不成立）→ 不进入 policy-backed missing `.S` 分支，route 走 no-vectorization。
   - （不适用 policy-backed missing `.S` 分支，故无 implementation-shape proof 六项；等价信息已并入上方 The fix 的 shape 描述。）
6. **Related PRs**：`Related PRs：16 条 URL`（patterns/no-vectorization.md §Related PRs，OpenCV scalable RVV 系列）：
   https://github.com/opencv/opencv/pull/22179、https://github.com/opencv/opencv/pull/22520、https://github.com/opencv/opencv/pull/23980、https://github.com/opencv/opencv/pull/24058、https://github.com/opencv/opencv/pull/24132、https://github.com/opencv/opencv/pull/24166、https://github.com/opencv/opencv/pull/24301、https://github.com/opencv/opencv/pull/24325、https://github.com/opencv/opencv/pull/27160、https://github.com/opencv/opencv/pull/27119、https://github.com/opencv/opencv/pull/27097、https://github.com/opencv/opencv/pull/27007、https://github.com/opencv/opencv/pull/26958、https://github.com/opencv/opencv/pull/26865、https://github.com/opencv/opencv/commit/b902a8e792e1702b40f19dbd48dff0bfdca8b36d、https://github.com/opencv/opencv/commit/2c16f3b7d2b28f6cac444046b8f95b40d9266a6a

### Finding B（independent）— `patterns/isa_extension_specific_instruction_substitution.md`

1. **Root cause**：Go riscv64 后端对哈希轮函数未做 Zbb 指令选择——每个 32-bit 旋转被合成为 `srliw+slliw+or` 三元组（Σ1/Σ0 每轮 6 个、W schedule σ0/σ1 每轮再 4 个），BE 消息字被逐字节拼装（4×lbu+3×slli+3×or=10 条/字），而目标 HW 明确支持 Zbb（`roriw`/`rori`/`rev8`）。依据 `patterns/isa_extension_specific_instruction_substitution.md` §Why this is slow 第 1 条：「Without the Zbb extension, a byte rotation is implemented with 3–4 base instructions (e.g., `slli`, `srli`, `or`). That consumes extra decode, issue, and retirement slots」——每条旋转省 2 条指令且把 2–3 级依赖链压成 1 级（rori 单周期）。
2. **The fix / 修复方式**：
   - 修复对象（两个可选载体，均属实现层，非补丁实施）：
     a) **Go 编译器后端**：在 `cmd/compile/internal/ssa/gen/RISCV64.rules` / `rewriteRISCV64.go` 增加按常量旋转 → `roriw`/`rori` 的 lowering（匹配 `(OR (SRLIW x [c]) (SLLIW x [32-c]))` 形态），并在 `cmd/internal/obj/riscv` 提供编码；BE 字节拼装/`encoding/binary.BigEndian.Uint32` 读入 → `lwu+rev8+srli 32` 形态（按 pattern §The fix 第 8 节「两指令 `rev8+srli` bswap 在结果先于全部输入被消费前写出，缺 `+&r`（early-clobber）时编译器可能让 input/output 别名同一寄存器而破坏中间值」——必须带 early-clobber 约束）；Zbb 能力通过 Go 的 riscv64 feature gate（`internal/cpu` / build flag）判定，无 Zbb 目标保持原合成序列。
     b) **手写 riscv64 block 汇编**：利用 Go 汇编器已支持的 `roriw`/`rev8` 编码重写 64 轮（与 amd64/arm64 sha256block_*.s 对齐），可直接取得本函数全部收益，无需等编译器规则。
   - 修复前/后（伪代码，32-bit）：
     ```asm
     # Before: rotr(6) of 32-bit e
     srliw t6, a4, 6
     slliw t1, a4, 26
     or    t1, t6, t1
     # After: Zbb roriw
     roriw t1, a4, 6
     ```
     ```asm
     # Before: BE message word (10 instr)
     lbu   t0, 0(t4); lbu t1, 1(t4); lbu t2, 2(t4); lbu s0, 3(t4)
     slli  t0, t0, 24; slli t1, t1, 16; or t0, t1, t0
     slli  t2, t2, 8;  or t0, t2, t0;   or t0, s0, t0
     # After: lwu + rev8 + srli (3 instr, early-clobber 防别名)
     lwu   t0, 0(t4)
     rev8  t0, t0
     srli  t0, t0, 32
     ```
   - 适用前提：目标构建 profile 明确包含 Zbb（本核 isa 含 `zba zbb zbc zbs`）；32-bit 旋转与 rev8+srli 的结果语义与合成序列 bit-identical（值域 32-bit、lwu 零扩展保证 rev8 后高半字为 0）。
   - correctness contract：SHA-256 轮函数数值输出不变（FIPS140 自检向量）；旋转移位语义（0<rot<32）与 shr 部分（σ 的 `>>` 用 `srli`/`srliw`，不可替换为 rori）保持；字节序（big-endian 消息字）保持；Go 语言层语义（uint32 运算、无溢出 UB）保持。
   - 限制/风险：Zbb 非 rv64gc 基线 → 编译器路径必须 feature-gated，无 Zbb 目标回归原序列；`roriw` 在 RV64 上对 32 位值写回会符号扩展（`roriw` 是 W 型指令，写 rd 时高位 sign-extend）——Go 的 uint32 语义依赖零扩展，须核对 SSA 层 extension 处理；rev8 两指令形态的 early-clobber 约束不可省。
   - 预期 Profile signals：重跑 annotate 后轮函数体出现 `roriw`（替代 `srliw/slliw/or` 三元组）、消息装载段出现 `lwu+rev8+srli`（替代 lbu 拼装）；静态指令数估计削减 ≈39%（16 轮×12 + 48 轮×20 旋转削减 1,152 条 + 16 字×7 拼装削减 112 条，对 ≈3,260 条/块）。
3. **Baseline facts 回填**：hardware ISA 含 `zba zbb zbc zbs`（roriw/rori/rev8 可用）；build ISA=`baseline_gap: build ISA`；VLEN=128（本 finding 为标量指令选择，VLEN 仅影响替代性的向量方案）；bound type=compute/latency。
4. **收益上界**：入口条件 A 局部样本份额 —— 与 Finding A 共享同一 19/19 局部区间（不叠加、不排序为倍数）；采样语义四条不满足，无 workload 级 Amdahl 上界；静态指令削减 ≈39% 为指令数估计，非 cycle/时间上界。
5. **三维路由判定**：current source=Go 编译器生成（非 .S）；implementation existence=编译器当前不发射 roriw/rev8（annotate 直接证据），Go 汇编器具备编码能力（载体 b 可达）；function-level policy=Go 工具链默认 rv64gc 基线、Zbb 需显式 feature gate（Go riscv64 尚无 per-extension 构建开关 → 载体 a 需先引入 gate，载体 b 可在 asm 层直接使用）。
6. **Related PRs**：`Related PRs：59 条 URL`（patterns/isa_extension_specific_instruction_substitution.md §Related PRs，截取与本 finding 直接相关的 Zbb ror/rev8 与 Go 后端子集，全部为原文 URL）：
   Go：https://github.com/golang/go/commit/3659b8756a2b81766e589e34d4fe9613b5917de0、https://github.com/golang/go/commit/a6ecdf29e34ddc82b6ed2315aaedf4c4d522b96c、https://github.com/golang/go/pull/59488、https://github.com/golang/go/commit/63ab68ddc5f1307e552cf27ae7a6f0dfda2bb962、https://github.com/golang/go/commit/1951afc9193f8e197cb7dfaf6afed70ea02404cb；
   OpenSSL：https://github.com/openssl/openssl/commit/03ce37e11729、https://github.com/openssl/openssl/commit/ca6286c382a7、https://github.com/openssl/openssl/commit/48b6776678d7、https://github.com/openssl/openssl/commit/6136408e6abf、https://github.com/openssl/openssl/commit/e4fd3fc379d7、https://github.com/openssl/openssl/commit/80c664db430d、https://github.com/openssl/openssl/commit/08c8dd6b8cede3cdbe5b1866c1a7544e0fe7a378、https://github.com/openssl/openssl/commit/49a3e7adc392、https://github.com/openssl/openssl/commit/a41f9135f082、https://github.com/openssl/openssl/commit/4dbb537bd1ea、https://github.com/openssl/openssl/commit/608cadfbdbdb、https://github.com/openssl/openssl/commit/b1b889d1b3fc、https://github.com/openssl/openssl/commit/657d1927c68b、https://github.com/openssl/openssl/commit/611685adc04a、https://github.com/openssl/openssl/commit/7ae2bc9df6e0；
   Linux Kernel：https://github.com/torvalds/linux/commit/e8620bd7e5e07c025a5e317ebc8c7df95dc63712、https://github.com/torvalds/linux/commit/5ba15d419fab848a3813eb56bbcad00e291fbc49、https://github.com/torvalds/linux/commit/cc2294d3f9c99c216ef563b83b08d2c0604f9b92、https://github.com/torvalds/linux/commit/36e22416872114cae812cdcdd84a5b99ef30b3de、https://github.com/torvalds/linux/commit/e11e367e9fe57164ea609807ed27184c85263355、https://github.com/torvalds/linux/commit/75ab93a244a516d1d3c03c4e27d5d0deff76ebfb、https://github.com/torvalds/linux/commit/c640868491105d53899f9f8e613acd4aa06cef68；
   OpenJDK：https://github.com/openjdk/jdk/commit/6b89954c65342bc601633d24075dab4f4b248f4b、https://github.com/openjdk/jdk/commit/a7631ccf18e468d6ecba121865f7fed29cbf2186、https://github.com/openjdk/jdk/pull/22752、https://github.com/openjdk/jdk/commit/08d563ba15047020fd5f5fea80547e18898bbab2、https://github.com/openjdk/jdk/commit/b1a21b563e3ae13fa5c409a4f0c04686c3f5b34a、https://github.com/openjdk/jdk/pull/22410；
   LLVM：https://github.com/llvm/llvm-project/pull/170824、https://github.com/llvm/llvm-project/commit/4c1e1e05cb901a2ed9055e5d6ac6ce60b826a288、https://github.com/llvm/llvm-project/pull/92926、https://github.com/llvm/llvm-project/pull/152744、https://github.com/llvm/llvm-project/pull/122698、https://github.com/llvm/llvm-project/commit/13e32a8a3c95b23af51f081865db1bd259d269a4、https://github.com/llvm/llvm-project/commit/787eeb8597fa22decb366a42176b11f52ec1bf0、https://github.com/llvm/llvm-project/commit/c705b7b04dba467a6781a771b667af9db7d1ce21（原文 c705b7b04dba467a6781a771b667af9db7d1ce21 按原文 URL https://github.com/llvm/llvm-project/commit/c705b7b04dba467a6781a771b667af9db7d1ce21）

## Phase 5 — Verification forecast / 验证预测：crypto/internal/fips140/sha256.block.abi0

**Finding A（no-vectorization）验证**：
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：0x536476–0x539b2e 区间内 `lbu`/`srliw`/`slliw`/`or`/`lwu`(s3) 标量序列不再主导 hot loop；`536f3e: lwu t0,56(s3)` 类 W schedule 栈往返消失。
- 应出现侧（锚定 `patterns/no-vectorization.md` §Verification）：同一函数 annotate 出现 RVV 指令（`vsetvli`、`vle32`、`vadd.vv`、`vxor.vv`、`vsll.vi`/`vsrl.vi`、`vrgather.vv`）；scalar instructions 不再主导 hot loop；覆盖 empty/短/整倍数/tail 长度对比；crypto/sha256 代表性输入 benchmark（cycles per block）。
- 附带要求：为确认真实采用，需在启用 RVV 的构建路径下编译并确认 `block` 入口进入新 kernel；非 V 构建保持标量 fallback 且结果一致。

**Finding B（ISA substitution）验证**：
- 应消失/缩小侧（锚定 Phase 3(a) 引用行）：`5364a2: srliw t6,a4,0x6`+`5364a6: slliw t1,a4,0x1a`+`5364aa: or t1,t6,t1` 三元组被单条 `roriw` 替代；`536476: lbu t0,0(t4)`…`536494: or t0,s0,t0` 拼装段被 `lwu+rev8+srli` 替代；W schedule 段 `536f4e/536f52/536f56`（rotr17）同样收敛为 `roriw`。
- 应出现侧（锚定 `patterns/isa_extension_specific_instruction_substitution.md` §Verification）：`go tool objdump -S` 确认 SHA-256 热循环出现 `roriw`（或 `rori`）与 `rev8`；无 Zbb 构建下 fallback 序列重新出现；`crypto/sha256` 测试（含 FIPS140 自检向量）bit-identical；rotation 边界（rot=0、31/32、全 1 数据）与 rev8 的 16/32/64-bit all-zero/all-one/符号位 pattern 测试通过；在 OoO（C920v2）与 in-order 核分别 benchmark 确认无回归。
- 两条 finding 分别验证（先各自验证，再做组合验证），不互相替代。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status |
|---|---|---|
| 1 | 承诺声明兑现：声明的 1 组 Phase 3–5 全部出现（载荷：`1/1 组；[crypto/internal/fips140/sha256.block.abi0]`） | ✅（载荷：1/1 组；crypto/internal/fips140/sha256.block.abi0） |
| 2 | Phase 1 输出要求满足（7 行 baseline 表 + Sampling IP precision + 2 个 L0 gate + bound gate；gap 标签：`baseline_gap: build ISA`、`baseline_gap: sampling IP precision`） | ✅（载荷：7 行结论 + `baseline_gap: build ISA` + `baseline_gap: sampling IP precision`） |
| 3 | Phase 3 输出要求满足（8 项 Class selection trace；`Classes scanned:` = rows-asm, rows-operator-rvv, rows-string-memory, rows-codegen, rows-crypto；顶层 finding 2 个（A/B），supporting 2 条，互斥排除 13 条，推导式 2 条） | ✅（载荷：8 项 trace；5 个 include class 文件名；2 顶层 + 2 supporting；证据锚点 `5364a2 srliw t6,a4,0x6`、`536476 lbu t0,0(t4)`、`536f3e lwu t0,56(s3)`、`539ae6 add a0,a0,t0`） |
| 4 | Phase 4 输出要求满足（已读 pattern：no-vectorization.md、isa_extension_specific_instruction_substitution.md、register_pressure_and_save_restore.md、wide_scalar_memory_access_codegen.md；对应命中 row：No vectorization / ISA-Substitution（+2 supporting）；引用短语首词：`VLMAX`、`Without the Zbb extension`、`Spill/reload`、`lbu`/shift/OR；The fix 均含 before/after、correctness、风险、预期 Profile signals；missing `.S` 分支不适用（policy 四证不齐）已注明；Related PRs：no-vectorization 16 条、ISA-substitution 59 条） | ✅ |
| 5 | 路径合规：8 项 trace 可解释扫描集；多命中按 independent/supporting 仲裁并给出因果消除测试；每个 blueprint leaf 来自通过 gate 的 row（dedicated-vector-crypto 因 HW 无 zvk* 未通过 gate，未纳入）；入口模式 A 按局部份额排序（19/19，不叠加）；`th.v*` 无（不适用停扫） | ✅（载荷：模式 A + independent(A/B)+supporting(2) + class 列表 5 项） |
| 6 | Phase 5 两侧锚定：消失侧对 Phase 3(a) 引用行（`5364a2`/`536f3e`/`536476`），出现侧标注 pattern §Verification | ✅（载荷：A→no-vectorization.md §Verification；B→isa_extension_specific_instruction_substitution.md §Verification） |
| 7 | 契约边界合规：无实施询问、无代码修改、无补丁生成；无向用户追问；交付物止于 Profile 证据、根因蓝图、The fix 与验证预测 | ✅ |

修正记录：无