Functions under analysis: [__strrchr_vector]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

| Evidence | 状态 |
|---|---|
| 单函数完整 annotate | 已提供（`__strrchr_vector`，libc.so，event=cpu-clock，1435 samples，local period） |
| perf stat | 已提供（string-benchset 全用例：IPC=1.251，branch_miss 1.624%，L1 miss 0.716%，LLC NA） |
| workload/binary/DSO/source context | 已提供（glibc bench-strrchr；annotate 内嵌 .S 源行 + 工作区 `sysdeps/riscv/rvv/strrchr.S` 逐指令核对） |
| readelf -A | 部分提供（metadata bench 二进制 zvl128b/RVV 1.0；libc.so 直接 attribute 未采集） |
| hardware ISA | 已提供（SpacemiT K3/X100，RVV 1.0，VLEN=256，OoO） |
| vlenb | 已提供（vlenb=32 → VLEN=256） |
| 采样元数据 | 部分提供（cpu-clock；percent type=**local period** → `baseline_gap: sampling metadata`） |
| Sampling IP precision | 缺失 → `baseline_gap: sampling IP precision` |

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_…`（RVV 1.0）+ B/Zvbb/Zvk/Zvkg + Zicbom/Zicbop/Zicboz + SSTC |
| Build ISA | bench 二进制 zvl128b/RVV 1.0；libc.so 执行 `v*` → 构建含 `v`；`baseline_gap: build ISA (libc.so 直接 attribute)` |
| Vector flavor | RVV 1.0 `v*`（`vle8ff.v`/`vmseq`/`vmsbf.m`/`vmand.mm`/`vfirst.m`/`vredmaxu.vs`/`vid.v`）；无 `th.v*` |
| VLEN | 256 bits（vlenb=32） |
| Bound type | 局部：scan/CSR 读/归约链主导；`baseline_gap: cache counters`（LLC NA） |
| Sampling semantics | cpu-clock；local period → 收益上界限局部样本份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision` → 单行归因受限，锚定 loop interval |

**L0 baseline gate**：hardware 有 `v`、build 有 `v` → 无 mismatch；无 `th.v*`。

## Phase 2 — Scope / 分析边界

- 函数清单：`__strrchr_vector`（1 组 Phase 3–5）
- **hot loop interval**：`a5f3e–a5f92`（`L(loop)`，ch≠0 主扫描）；最高行 `35.26 : a5f5c: csrr a6,vl`
- 次要区间：`L(search_zero)`（ch==0 路径）a5fa8–a5fc6（≈3.2%）
- Sampling IP precision 未确认 → 锚定 loop interval；annotate 覆盖完整

## Phase 3 — Pattern scan / 模式扫描：__strrchr_vector

**Class selection trace（8 项）**：

| # | Class | include/exclude | 触发观察 |
|---|---|---|---|
| 1 | rows-asm.md | include | annotate 内嵌 .S 源行（`ENTRY (STRRCHR)`/`.option arch, +v`）+ symbol → 直接 `.S` provenance；samples 主导在 kernel 内 |
| 2 | rows-string-memory.md | include | glibc/libc 单输入 sentinel scan（strrchr），判别 semantic/integration row |
| 3 | rows-operator-rvv.md | exclude | 已向量化手写 kernel，非 compiler-generated scalar loop |
| 4 | rows-vectorized-tuning.md | exclude | 手写 `.S`（class 索引限定"非手写 `.S`"才扫） |
| 5 | rows-codegen.md | exclude | 非 compiler/JIT 生成；vector 实现已被采用 |
| 6 | rows-offload.md | exclude | 无矩阵引擎/packed-SIMD 信号 |
| 7 | rows-crypto.md | exclude | 非密码学原语 |
| 8 | rows-runtime-os.md | exclude | 非 kernel/RTOS/timer 热点 |

**Classes scanned**: `rows-asm.md`; `rows-string-memory.md`

**Local performance pattern scan: `__strrchr_vector`**

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Existing RISC-V Assembly Kernel Bottlenecks（primary） | 手写 RVV strrchr kernel；每迭代 `csrr a6,vl` 占 35.26%（fault-first 推进量的 CSR 读，位于 vle8ff→vmseq→vmsbf→vmseq.vx→vmand→vfirst→csrr 链末端）；found 路径 vtype 切换（e8/m2→e16/m4，3.34%）+ masked vredmaxu 归约链（j 行 17.70% 归因） | High | Medium | `patterns/riscv-assembly-kernel-performance-optimization.md` |

**(a) 逐字 evidence 引用**（main loop interval a5f3e–a5f92）：
```
35.26 :   a5f5c:  csrr    a6,vl             # 每迭代读 post-load vl（fault-first 必需）
16.72 :   a5f60:  bltz    a5,a5f7a <__strrchr_vector+0x42>  # first_idx 分支
17.70 :   a5f78:  j       a5f7c <__strrchr_vector+0x44>     # found 路径 vredmaxu 链归因点
10.31 :   a5f84:  bne     a5,a6,a5f94 <__strrchr_vector+0x5c> # len_valid vs cur_vl（tail 判定）
 3.34 :   a5f64:  vsetvli zero,a6,e16,m4,ta,ma  # found 路径 vtype 切换（e8/m2→e16/m4）
 1.05 :   a5f54:  vmand.mm        v0,v10,v12     # vhit = ch-match ∧ before-NUL
 1.05 :   a5f58:  vfirst.m        a5,v0
```

**(b) 互斥邻居排除**：
- **RVV Sentinel Scan/Copy Implementation**：该 row 行内信号为 *scalar loop*；本 loop 已向量化（`vle8ff.v`/`vmseq`/`vfirst.m`）→ route gate 不成立，排除
- **RVV Vector-State Management**：pattern 行内互斥明确手写 `.S` 的配置/CSR 调度归 assembly row §3 → 不设独立顶层 finding（csrr 观察并入 primary）
- **RVV Register-Group/LMUL**：e8/m2 为扫描所用 LMUL；found 路径 e16/m4 是 last-hit 归约的形态选择，非 LMUL 根因 → 排除
- **Maximal LMUL**：非 e8/m1 小 LMUL overhead 问题 → 排除
- **glibc RVV IFUNC/ABI Integration**：`__strrchr_vector` 已被执行 → 排除
- **No vectorization**：loop 有 `v*` → 排除

**(c) 双 Confidence 推导式**：
- `route: 直接 .S provenance（annotate 源行 + 工作区源码核对）+ samples 主导在 kernel 内 + 互斥排除成立 → High`
- `impact: csrr 行局部份额 35.26% 大，但 precise_ip 未确认（csrr 行可能混入前置向量链 skid 归因）、csrr latency 在 K3 上未实测、found 路径归约为算法固有 → Medium`

**多候选仲裁**：单一顶层 primary finding；无 companion/supporting/independent。

## Phase 4 — Root-cause blueprint / 根因蓝图：__strrchr_vector

**1. Root cause**

`__strrchr_vector` 是 glibc 手写 RVV `strrchr` kernel（`sysdeps/riscv/rvv/strrchr.S`）。主扫描 loop 每块（e8/m2，VLEN=256 下 64B）执行：`vle8ff.v` fault-first 加载 → `vmseq.vi`(NUL 掩码) → `vmsbf.m`(NUL 前有效区) → `vmseq.vx`(ch 掩码) → `vmand.mm`(有效 ch 命中) → `vfirst.m`(首命中) → `csrr a6,vl`(读 post-load vl) → `bltz`(命中判定)。found 块再追加 e16/m4 的 `vid.v`+masked `vredmaxu.vs` 求块内最后命中索引。

结构性问题（依据 `riscv-assembly-kernel-performance-optimization.md` §3 序列化配置指令）：
- **`csrr a6,vl`（35.26%，单条最热点）每迭代执行**：fault-first 语义要求推进量 = post-load vl，但 `vsetvli a6,zero,e8,m2` 已将 VLMAX 写入 a6——完全映射的 buffer（benchmark 情形）不会触发 fault，post-load vl == a6，**csrr 在常见路径冗余**。该 CSR 读在多数实现上是慢/序列化指令，位于 vle8ff→…→vfirst→csrr→bltz 链末端，35.26% 的样本份额（含前置向量链 skid 归因）指示该区间是每迭代的主要开销点。
- **found 路径 vtype churn**：e8/m2 → e16/m4（3.34% vsetvli）→ 计算后切回 e8/m2（a5f7c），每 found 块两次额外 vsetvli + 一次 masked vredmaxu（其解析成本归因到 `j` 行 17.70%）。
- `vsetvli a6,zero,e8,m2` 用 rs1=x0（VLMAX=64B）而非剩余量 —— 每块固定 64B，块数 = len/64；对短字符串 per-call 固定开销占比高。

**2. The fix / 修复方式**（保持 strrchr 语义，仅实现层面）

修复对象：每迭代的 `csrr vl`（a5f5c）与 found 路径的 vtype 切换（a5f64/a5f7c）。

**候选 1（primary）：页边界有界加载替代 fault-first + csrr**
```asm
# Before（当前）：vle8ff + csrr 每迭代
	vsetvli	a6, zero, e8, m2, ta, ma   # VLMAX
	vle8ff.v	v4, (a0)
	... vector 比较链 ...
	csrr	a6, vl                      # 读 post-load vl（35.26%）
	bltz	a5, L(no_hit_in_block)
	...

# After：ALU 计算到下一页边界的字节数，bounded vle8 免 csrr
	li	t0, -4096                    # 页掩码（按目标页大小）
	and	t0, t0, a0                   # 页基址
	sub	t0, t0, a0                   # 页内剩余字节（≤0）
	neg	t0, t0                       # 到页尾字节数（含本字节）
	vsetvli	a6, t0, e8, m2, ta, ma    # vl = min(page_remaining, VLMAX)，写入 a6
	vle8.v	v4, (a0)                   # 常规加载（不 fault），推进量 = a6 直接用
	... vector 比较链 ...
	bltz	a5, L(no_hit_in_block)
	...
```
- 推进量直接使用 a6（vsetvli 寄存器值），**loop 内无 csrr**；页边界安全由「vl ≤ 页内剩余」保证（不会跨未映射页读取）。
- Correctness contract：strrchr 返回最后一次出现 `ch` 的指针（ch≠0）或指向 NUL 的位置（ch==0 路径，保持 `L(search_zero)` 不变）；扫描不得越过 NUL（vmsbf/vmask_end 语义保留）；页边界不越界（bounded vl）。
- 限制/风险：引入 4-5 条 ALU 指令/迭代替代 1 条 csrr；若 K3 的 csrr 实际为低延迟，则净收益有限（须 A/B 实测）；对 ch==0 的 `L(search_zero)` 路径同样适用此结构。

**候选 2：found 路径消除 vtype 切换** —— last-hit 归约改用 e8/m2 的 `vid.v`+masked `vredmaxu.vs`（块内索引 0..63，e8 足够），避免 e16/m4 的 vsetvli 切进切出；预期 `a5f64: vsetvli zero,a6,e16,m4` 与 `a5f7c: vsetvli zero,a6,e8,m2` 消失。
- Correctness：mask 位宽与元素索引范围在 e8/m2（vl≤64）下合法；vredmaxu 结果语义不变。

**3. Baseline facts 回填**：hardware RVV 1.0 / VLEN=256；bound=scan/CSR/归约链主导；local period；precise_ip 缺失。

**4. 收益上界**：局部样本份额。csrr 行 35.26% + found 路径 vsetvli 3.34% ≈ **38.6%**（含 skid 归因，实际可获收益为其中 csrr 自身 latency 部分）；`j` 行 17.70% 为 vredmaxu 归约的归因（候选 2 可部分缓解）。`baseline_gap: sampling metadata`（local period）→ 不得表述 workload 级 Amdahl 上界。

**5. 三维路由判定**：
- `current source`：手写 RVV `.S`（annotate 源行 + 工作区源码核对）
- `implementation existence/reachability`：存在且被采用 → kernel-selection 不适用
- `function-level policy`：IFUNC 已选中 vector variant → missing-`.S` 四证不适用

**6. Implementation-shape proof**：不适用（非 missing `.S`）。

**7. Related PRs**（`patterns/riscv-assembly-kernel-performance-optimization.md`，按 pattern 分组）：
`Related PRs：20 条 URL`（完整列表见该 pattern §Related PRs 表；§3 相关：5bc3b7f51308、982376660c58 等）。

## Phase 5 — Verification forecast / 验证预测：__strrchr_vector

- **消失/缩小侧**（锚定 Phase 3(a) 引用行）：`a5f5c: csrr a6,vl` 应从主 loop 消失或仅在页边界慢路径残留；`a5f64: vsetvli zero,a6,e16,m4,ta,ma` 与 `a5f7c: vsetvli zero,a6,e8,m2,ta,ma` 消失（若实施候选 2）；csrr 行 35.26% 份额下降。
- **出现侧**（锚定 `patterns/riscv-assembly-kernel-performance-optimization.md` §Verification）：页边界计算 ALU 序列（`andi`/`sub`/`neg` + `vsetvli a6,t0,…`）与 `vle8.v`（非 fault-first）出现。
- **benchmark 覆盖**（§Verification）：ch 在首/中/末/未出现、ch==0、短中长、未对齐、page-boundary placement。
- **A/B 判据**：同一 workload 重跑 `perf annotate`；对比 csrr/bnez 行份额与 bench-strrchr duration；实测 K3 上 csrr vl latency 与 ALU 页计算成本。
- **补采建议**：`perf evlist -v` 确认 precise_ip；`readelf -A libc.so`。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | `1/1 组；__strrchr_vector` |
| 2 | Phase 1 输出要求 | ✅ | `7 行 baseline；gap 标签：baseline_gap: sampling IP precision、sampling metadata、cache counters、build ISA (libc.so)` |
| 3 | Phase 3 输出要求 | ✅ | `Class selection trace 8 项；Classes scanned: rows-asm.md; rows-string-memory.md；顶层 finding 1 个（Existing RISC-V Assembly Kernel Bottlenecks）；evidence 锚点：35.26 : a5f5c: csrr a6,vl、16.72 : a5f60: bltz a5,a5f7a、17.70 : a5f78: j a5f7c、3.34 : a5f64: vsetvli zero,a6,e16,m4,ta,ma；supporting 0；排除 6 条；推导式 1` |
| 4 | Phase 4 输出要求 | ✅ | `已读 pattern: riscv-assembly-kernel-performance-optimization.md；命中 row: Existing RISC-V Assembly Kernel Bottlenecks；引用短语：§3 序列化的配置指令；The fix 含 before/after/correctness/风险/Profile signals；Related PRs：20 条 URL` |
| 5 | 路径合规 | ✅ | `模式 A；include class: rows-asm.md; rows-string-memory.md；finding 来自通过 gate 的 row；无 L0 mismatch` |
| 6 | Phase 5 两侧锚定 | ✅ | `消失侧：a5f5c csrr a6,vl（Phase 3 引用行）；出现侧：patterns/riscv-assembly-kernel-performance-optimization.md §Verification` |
| 7 | 契约边界合规 | ✅ | `无实施询问/代码修改/补丁生成；交付止于证据、根因、The fix、验证预测` |

修正记录：无