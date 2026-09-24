Functions under analysis: [__memcmp_vector]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

| Evidence | 状态 |
|---|---|
| 单函数完整 annotate | 已提供（`__memcmp_vector`，libc.so，event=cpu-clock，614 samples，local period） |
| perf stat | 已提供（string-benchset 全用例：IPC=1.251，branch_miss 1.624%，L1 miss 0.716%，LLC NA） |
| workload/binary/DSO/source context | 已提供（glibc bench-memcmp；annotate 内嵌 .S 源行 + 工作区 `sysdeps/riscv/rvv/memcmp.S` 逐指令核对） |
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
| Vector flavor | RVV 1.0 `v*`（`vle8.v`/`vmsne.vv`/`vfirst.m`/`vrgather.vx`/`vmv.x.s`） |
| VLEN | 256 bits（vlenb=32）；kernel 用 **e8/m8（256B/iter，已最大 LMUL）** |
| Bound type | 局部：比较链（vle8 src2 17.43% + vmsne 24.76% + bgez 18.73%）+ found 路径提取链（≈34.4%）主导；`baseline_gap: cache counters`（LLC NA） |
| Sampling semantics | cpu-clock；local period → 收益上界限局部样本份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision` → 锚定 loop interval |

**L0 baseline gate**：hardware 有 `v`、build 有 `v` → 无 mismatch；无 `th.v*`。主循环已最大 LMUL（m8）→ LMUL 不是杠杆。

## Phase 2 — Scope / 分析边界

- 函数清单：`__memcmp_vector`（1 组 Phase 3–5）
- **hot loop interval**：`a4760–a477e`（`L(loop)`，256B/iter）+ found 路径 `a4784–a479e`
- 最高行：`25.08 : a4790: vmv.x.s a4,v24`（found 路径第二提取点）+ `24.76 : a476c: vmsne.vv v16,v0,v8`（比较）
- Sampling IP precision 未确认 → 锚定 loop interval；annotate 覆盖完整

## Phase 3 — Pattern scan / 模式扫描：__memcmp_vector

**Class selection trace（8 项）**：

| # | Class | include/exclude | 触发观察 |
|---|---|---|---|
| 1 | rows-asm.md | include | annotate 内嵌 .S 源行（`ENTRY (MEMCMP)`/`.option arch, +v`）+ symbol → 直接 `.S` provenance |
| 2 | rows-string-memory.md | include | glibc/libc two-input compare（memcmp），判别 compare/integration row |
| 3 | rows-operator-rvv.md | exclude | 已向量化手写 kernel |
| 4 | rows-vectorized-tuning.md | exclude | 手写 `.S`；LMUL 已最大（m8） |
| 5 | rows-codegen.md | exclude | 非 compiler/JIT 生成；vector 实现已被采用 |
| 6 | rows-offload.md | exclude | 无矩阵引擎/packed-SIMD 信号 |
| 7 | rows-crypto.md | exclude | 非密码学原语 |
| 8 | rows-runtime-os.md | exclude | 非 kernel/RTOS/timer 热点 |

**Classes scanned**: `rows-asm.md`; `rows-string-memory.md`

**Local performance pattern scan: `__memcmp_vector`**

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| Existing RISC-V Assembly Kernel Bottlenecks（primary） | 手写 RVV memcmp kernel 主循环已最大 LMUL（e8/m8, 256B/iter）；热点含 found 路径字节提取链：vrgather.vx×2 + vmv.x.s×2（vmv.x.s a4 25.08% + vrgather 2.28% ≈27.4%，另 zext 5.54%）；命中分支时 src1/src2 仍指向 chunk 基址 → 可直接标量 lbu 提取差异字节 | High | Medium | `patterns/riscv-assembly-kernel-performance-optimization.md` |

**(a) 逐字 evidence 引用**（loop interval a4760–a477e + found 路径 a4784–a479e）：
```
17.43 :   a4768:  vle8.v  v8,(a1)             # src2 chunk（比较链）
24.76 :   a476c:  vmsne.vv        v16,v0,v8   # 差异掩码
18.73 :   a4776:  bgez    a4,a4784            # 命中差异分支
 1.30 :   a4784:  vrgather.vx     v16,v0,a4   # 提取差异字节1（found 路径）
 0.98 :   a4788:  vrgather.vx     v24,v8,a4   # 提取差异字节2（found 路径）
25.08 :   a4790:  vmv.x.s a4,v24              # 第二次提取的解析归因点
 5.54 :   a4794:  zext.b  a0,a0
```

**(b) 互斥邻居排除**：
- **RVV Compare/First-Difference Implementation**：该 row 行内信号为 *scalar loop*；本 loop 已向量化 → gate 不成立，排除
- **RVV Register-Group/LMUL**：主循环已 e8/m8 最大 → LMUL 非杠杆，排除
- **RVV Vector-State Management**：手写 `.S` 配置调度归 assembly §2/§3；无重复 vsetvli → 不设独立 finding
- **Maximal LMUL（rows-vectorized-tuning）**：compiler-generated 限定 + 已 m8 → 排除
- **glibc RVV IFUNC/ABI Integration**：`__memcmp_vector` 已被执行 → 排除
- **No vectorization**：loop 有 `v*` → 排除

**(c) 双 Confidence 推导式**：
- `route: 直接 .S provenance + found 路径结构清晰（命中分支时 src1/src2 未推进）+ samples 归属明确 → High`
- `impact: found 路径局部份额 ≈34.4% 成立，但 vrgather 与标量 lbu 在 K3 上的实际成本差未知、bench 命中位置 mix 影响收益、precise_ip 缺失 → Medium`

**多候选仲裁**：单一顶层 primary finding；无 companion/supporting/independent。

## Phase 4 — Root-cause blueprint / 根因蓝图：__memcmp_vector

**1. Root cause**

`__memcmp_vector` 是 glibc 手写 RVV `memcmp` kernel（`sysdeps/riscv/rvv/memcmp.S`）。主循环每迭代（**e8/m8，256B**）：双 `vle8.v` 载入 → `vmsne.vv` 差异掩码 → `vfirst.m` 首个差异索引 → `bgez` 命中早退；未命中则双 `add` 推进续扫。**主循环已用最大 LMUL（m8），无 LMUL 杠杆**；样本集中两处：
- **比较链**（vle8 src2 17.43% + vmsne 24.76% + bgez 18.73% ≈ 60.9%）——load+compare 的内在成本；
- **found 路径字节提取链**（vrgather.vx×2 2.28% + vmv.x.s×2 25.89% + zext 5.54% ≈ **34.4%**）——每次命中差异时执行 2 次 `vrgather.vx`（按索引 gather 差异字节）+ 2 次 `vmv.x.s` 提取。bench-memcmp 大量输入在首 chunk 即命中差异 → found 路径成为显著 per-call 成本。

关键结构事实（依据 `patterns/riscv-assembly-kernel-performance-optimization.md` §1 "先问：有没有一条指令已经能表达这个布局"）：**`bgez` 命中分支跳转时，`src1`（a0）/`src2`（a1）尚未推进**（`add src1,src1,ivl` 在未命中路径上），因此差异字节可直接用两条标量 `lbu` 从 `src+idx` 地址加载——完全不需要 `vrgather.vx`+`vmv.x.s` 的向量 gather 提取链。

**2. The fix / 修复方式**（保持 memcmp 语义，仅实现层面）

修复对象：found 路径的 `vrgather.vx`×2 + `vmv.x.s`×2 + `zext`×2 提取链 → 直接标量 `lbu` 提取。

**Before（当前，与 annotate/source 一致）**：
```asm
L(found):
	vrgather.vx	v16, v0, a4     # 提取 src1 差异字节（向量 gather）
	vrgather.vx	v24, v8, a4     # 提取 src2 差异字节
	vmv.x.s	a0, v16
	vmv.x.s	a4, v24
	zext.b	a0, a0
	zext.b	a4, a4
	sub	a0, a0, a4
	ret

# After（直接标量字节加载）：
L(found):
	add	a5, a0, a4          # src1 + idx（a0 仍为 chunk 基址）
	lbu	a0, 0(a5)           # 差异字节1
	add	a5, a1, a4          # src2 + idx
	lbu	a4, 0(a5)           # 差异字节2
	sub	a0, a0, a4
	ret
```

- **适用前提**：`bgez` 命中分支在 `add src1,src1,ivl` 之前 → 跳转时 `a0`/`a1` 仍指向当前 chunk 基址（与 annotate 顺序一致）；`lbu` 从已加载过的内存地址读取（L1 命中），无额外 fault 风险（该 chunk 已由 vle8 读取过，页已映射）。
- **Correctness contract**：memcmp 返回 `(unsigned char)s1[k] - (unsigned char)s2[k]`（k=首个差异位置）；相等返回 0；`lbu` 零扩展语义与 `zext.b` 一致（无符号字节比较）；src1/src2 在 found 分支点未推进的事实必须保持（否则需调整）。
- **限制/风险**：`vrgather.vx` 与标量 `lbu` 在 K3 上的延迟差未实测（若 K3 的 vrgather.vx 极快则收益有限，但 lbu 方案严格更少指令——7 条 vs 3 条）；须 A/B 实测确认。
- **预期 Profile signals**：`vrgather.vx`×2 与 `vmv.x.s`×2 从 annotate 消失，found 路径变为 `add+lbu+add+lbu+sub`；`vmv.x.s a4`（25.08%）行消失，found 路径份额大幅下降。

**3. Baseline facts 回填**：hardware RVV 1.0 / VLEN=256；主循环 e8/m8 已最大；bound=比较链 + found 提取链主导；local period；precise_ip 缺失。

**4. 收益上界**：局部样本份额。found 路径提取链（vrgather×2 2.28% + vmv.x.s×2 25.89% + zext 5.54% + sub/ret 0.65%）≈ **34.36%**；比较链 60.9% 为算法固有 load+compare 成本，非本修复对象。`baseline_gap: sampling metadata`（local period）→ 不得表述 workload 级 Amdahl 上界。

**5. 三维路由判定**：
- `current source`：手写 RVV `.S`（annotate 源行 + 工作区源码核对）
- `implementation existence/reachability`：存在且被采用 → kernel-selection 不适用
- `function-level policy`：IFUNC 已选中 vector variant → missing-`.S` 四证不适用

**6. Implementation-shape proof**：不适用（非 missing `.S`）。

**7. Related PRs**（`patterns/riscv-assembly-kernel-performance-optimization.md`，按 pattern 分组）：
`Related PRs：20 条 URL`（完整列表见该 pattern §Related PRs 表；§1 访存原语替代相关：79adb692a272、afb967b81e41 等）。

## Phase 5 — Verification forecast / 验证预测：__memcmp_vector

- **消失/缩小侧**（锚定 Phase 3(a) 引用行）：`a4784: vrgather.vx v16,v0,a4`、`a4788: vrgather.vx v24,v8,a4`、`a478c: vmv.x.s a0,v16`、`a4790: vmv.x.s a4,v24`、`a4794: zext.b a0,a0` 从 annotate 消失，替换为 `add+lbu+add+lbu+sub`；25.08% 的 `vmv.x.s a4` 行消失。
- **出现侧**（锚定 `patterns/riscv-assembly-kernel-performance-optimization.md` §Verification）：found 路径出现 `lbu`（标量字节加载）序列。
- **benchmark 覆盖**：bench-memcmp 覆盖相等/首字节差异/中间差异/末尾差异、短中长、未对齐、page-boundary。
- **A/B 判据**：同一 workload 重跑 `perf annotate`；对比 found 路径份额与 bench-memcmp duration；实测 K3 上 vrgather.vx vs lbu 的延迟差。
- **补采建议**：`perf evlist -v` 确认 precise_ip；若 PMU 支持采集 vrgather 相关事件。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | `1/1 组；__memcmp_vector` |
| 2 | Phase 1 输出要求 | ✅ | `7 行 baseline；gap 标签：baseline_gap: sampling IP precision、sampling metadata、cache counters、build ISA (libc.so)；主循环已最大 LMUL` |
| 3 | Phase 3 输出要求 | ✅ | `Class selection trace 8 项；Classes scanned: rows-asm.md; rows-string-memory.md；顶层 finding 1 个（Existing RISC-V Assembly Kernel Bottlenecks）；evidence 锚点：25.08 : a4790: vmv.x.s a4,v24、24.76 : a476c: vmsne.vv v16,v0,v8、18.73 : a4776: bgez a4,a4784、1.30 : a4784: vrgather.vx v16,v0,a4；supporting 0；排除 6 条；推导式 1` |
| 4 | Phase 4 输出要求 | ✅ | `已读 pattern: riscv-assembly-kernel-performance-optimization.md；命中 row: Existing RISC-V Assembly Kernel Bottlenecks；引用短语：§1 先问：有没有一条指令已经能表达这个布局；The fix 含 before/after/correctness/风险/Profile signals；Related PRs：20 条 URL` |
| 5 | 路径合规 | ✅ | `模式 A；include class: rows-asm.md; rows-string-memory.md；finding 来自通过 gate 的 row；无 L0 mismatch` |
| 6 | Phase 5 两侧锚定 | ✅ | `消失侧：a4790 vmv.x.s a4,v24（Phase 3 引用行）；出现侧：patterns/riscv-assembly-kernel-performance-optimization.md §Verification` |
| 7 | 契约边界合规 | ✅ | `无实施询问/代码修改/补丁生成；交付止于证据、根因、The fix、验证预测` |

修正记录：无