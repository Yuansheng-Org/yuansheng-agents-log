Functions under analysis: [__GI___wcschr]（1 个）→ 本输出含 1 组 Phase 3–5

## Phase 0 — Evidence inventory / 证据清单

| Evidence | 状态 |
|---|---|
| 单函数完整 annotate | 已提供（`__GI___wcschr`/`wcschr`，libc.so，event=cpu-clock，2550 samples，local period） |
| perf stat | 已提供（**wcsmbs-benchset 全用例**：IPC=1.732，branch_miss 0.757%，L1 miss 0.262%，LLC NA） |
| workload/binary/DSO/source context | 已提供（glibc bench-wcschr；annotate 内嵌 C 源行 + 工作区 `wcsmbs/wcschr.c` 核对；**compiler-generated scalar**；riscv rvv/multiarch 均无 wcs* 向量实现/IFUNC 条目） |
| readelf -A | 部分提供（metadata bench 二进制 zvl128b/RVV 1.0；libc.so 直接 attribute 未采集） |
| hardware ISA | 已提供（SpacemiT K3/X100，RVV 1.0，VLEN=256，OoO） |
| vlenb | 已提供（vlenb=32 → VLEN=256） |
| 采样元数据 | 部分提供（cpu-clock；percent type=**local period** → `baseline_gap: sampling metadata`） |
| Sampling IP precision | 缺失 → `baseline_gap: sampling IP precision` |

## Phase 1 — Baseline / 基线

| Baseline item | Conclusion / gap |
|---|---|
| Hardware ISA | `rv64imafdcvh_…`（RVV 1.0）+ B/Zvbb/Zvk/Zvkg + Zicbom/Zicbop/Zicboz + SSTC |
| Build ISA | bench 二进制 zvl128b/RVV 1.0；libc.so 执行 `v*`（其它函数）→ 构建含 `v`；`baseline_gap: build ISA (libc.so 直接 attribute)` |
| Vector flavor | 本函数 hot loop **zero `v*`**（全标量）；库整体 RVV 1.0 flavor |
| VLEN | 256 bits（vlenb=32）；wchar=4B → e32 下 m8 可覆盖 64 wchar/iter |
| Bound type | 局部：逐元素扫描链主导（main loop 83.57%）；`baseline_gap: cache counters`（LLC NA） |
| Sampling semantics | cpu-clock；local period → 收益上界限局部样本份额 |
| Sampling IP precision | `baseline_gap: sampling IP precision` → 锚定 loop interval |

**L0 baseline gate**：hardware 有 `v`、build 有 `v`、**本函数 zero `v*`** → V-capable 目标上 wcschr 无向量执行路径（首要 baseline finding）。

## Phase 2 — Scope / 分析边界

- 函数清单：`__GI___wcschr`（1 组 Phase 3–5）
- **hot loop interval**：`a76d4–a76dc`（逐元素扫描，4B/iter）；最高行 `45.29 : a76d6: beq a5,a1`
- Sampling IP precision 未确认 → 锚定 loop interval；annotate 覆盖完整

## Phase 3 — Pattern scan / 模式扫描：__GI___wcschr

**Class selection trace（8 项）**：

| # | Class | include/exclude | 触发观察 |
|---|---|---|---|
| 1 | rows-string-memory.md | include | glibc/libc compiler-generated scalar string 函数（单输入 target-scan，wchar），须判别三个 RVV semantic row |
| 2 | rows-asm.md | include | 判定目标 `.S` 是否缺失（policy/existence 四证检查） |
| 3 | rows-operator-rvv.md | include | compiler-generated scalar hot loop、zero `v*`、V-capable target → 检查 no-vectorization 行 |
| 4 | rows-vectorized-tuning.md | exclude | annotate 无 `v*` |
| 5 | rows-codegen.md | exclude | 无既有 kernel 未采用信号 |
| 6 | rows-offload.md | exclude | 无矩阵引擎/packed-SIMD 信号 |
| 7 | rows-crypto.md | exclude | 非密码学原语 |
| 8 | rows-runtime-os.md | exclude | 非 kernel/RTOS/timer 热点 |

**Classes scanned**: `rows-string-memory.md`; `rows-asm.md`; `rows-operator-rvv.md`

**Local performance pattern scan: `__GI___wcschr`**

| Pattern | Evidence | Route confidence | Performance-impact confidence | Detail file |
|---|---|---|---|---|
| RVV Sentinel Scan/Copy Implementation（primary） | main loop（a76d4–a76dc, 83.57%）为标量逐元素（4B/iter）扫描：lw → beq（wc 命中）→ addi 推进 → bnez（NUL 判定）；target-scan 语义（首个 wc 或 NUL）；hot interval 内 zero `v*`；riscv rvv/multiarch 均无 wcschr 向量实现 | High | Medium | `patterns/rvv_sentinel_scan_copy_implementation.md` |

**(a) 逐字 evidence 引用**（main loop interval a76d4–a76dc）：
```
 0.20 :   a76d4:  lw      a5,0(a0)          # 载入 wchar（4B）
45.29 :   a76d6:  beq     a5,a1,a76e0       # wc 命中分支
 6.55 :   a76da:  addi    a0,a0,4           # 推进 4B
31.53 :   a76dc:  bnez    a5,a76d4          # 非 NUL 回边
```

**(b) 互斥邻居排除**：
- **RVV Memory Copy/Fill**：非 copy/fill 语义 → 排除
- **RVV Compare/First-Difference**：单输入 target-scan，非两输入比较 → 排除
- **Word-Wide First-Difference**：限 byte two-input bounded-short compare → 排除
- **No vectorization**：行内判据"不符合更具体 semantic row **才进入**"——sentinel-scan 已命中 → 依自身判据排除
- **Policy-Backed Missing `.S`**：四证不全（multiarch 无 wcschr dispatch slot ✗、无官方 policy 要求 ✗）→ gate 不成立，排除
- **glibc RVV IFUNC/ABI Integration（独立行）**：无已存在 RVV 实现可接线 → 独立 gate 不成立；接线动作归 sentinel-scan §fix step 9

**(c) 双 Confidence 推导式**：
- `route: 直接 scalar loop 证据 + 语义合同（wchar target-scan、NUL 哨兵）匹配 + 工作区源码确认无 RVV 实现 → High`
- `impact: main loop 局部份额 83.57% 成立，但向量化收益依赖 wcs 长度/命中位置 mix、缺 precise_ip、无既有 RVV kernel 可比 → Medium`

**多候选仲裁**：单一顶层 primary finding；无 companion/supporting/independent。

## Phase 4 — Root-cause blueprint / 根因蓝图：__GI___wcschr

**1. Root cause**

`__GI___wcschr` 在目标上运行 `wcsmbs/wcschr.c` 的 **compiler-generated 标量逐元素实现**。main loop（a76d4–a76dc，**83.57%**）每迭代处理 **1 个 wchar（4B）**：`lw` 载入 → `beq`（与 wc 比较）→ `addi` 推进 → `bnez`（NUL 判定回边）。硬件（RVV 1.0, VLEN=256）本可用 e32/m8 单指令覆盖 64 个 wchar，但 `sysdeps/riscv/rvv/` 无 wcschr.S、`sysdeps/riscv/multiarch/` 无 wcschr IFUNC 条目——**整个 wcs* 宽字符函数族均无 RVV 实现**。

**2. The fix / 修复方式**（保持 wcschr 语义，仅实现层面）

修复对象：新增 RVV `wcschr` kernel 并按 glibc 惯例接入 multiarch（依据 pattern §fix step 1-9）。

**Before（当前结构，与 annotate/source 一致）**：
```c
/* wcsmbs/wcschr.c — 标量逐元素 */
while (1) {
  if (*wcs == wc) return (wchar_t *)wcs;
  if (*wcs == 0) return NULL;
  ++wcs;
}
```

**After（结构性替换：RVV 双掩码 + vfirst.m，e32）**，候选实现形态（`sysdeps/riscv/rvv/wcschr.S` + multiarch 接线）：
```c
/* 每 chunk：载入 64 个 wchar（e32/m8）→ mask = (e==wc) | (e==0) → vfirst.m 取首个命中 */
while (1) {
    chunk = vle32ff_or_bounded_load(wcs, &vl);   /* 页安全（e32） */
    mask = vmseq(chunk, wc) | vmseq(chunk, 0);   /* wc 与 NUL 合并 */
    idx = vfirst_m_b1(mask);
    if (idx >= 0) return wcs + idx;              /* 首个 wc 或 NUL */
    wcs += vl;
}
```

接线：新增 `sysdeps/riscv/multiarch/wcschr-vector.S` / `wcschr-generic.c`，IFUNC 名单与 `dl-symbol-redir-ifunc.h`（非 SHARED）加入 wcschr 条目（对照 strchr 接线模式）。

- **适用前提**：bench-wcschr 典型输入；长 wcs 无早期命中时收益最大。
- **Correctness contract**：wcschr 返回首个 `wc`（含 wc==0 返回 NUL 指针）或 NULL；不得越界读（fault-first/bounded，e32 页边界）；返回值 = wcs + idx。
- **限制/风险**：e32 下 chunk 覆盖 64 wchar/iter（m8）——比 byte 函数（256B/iter）元素数少但仍是标量的 64×；向量 kernel 对短 wcs（<chunk）建链开销须实测；收益受 bench 输入 mix 限制。
- **预期 Profile signals**：main loop 的 `lw/beq/addi/bnez` 序列被向量 load + `vmseq`×2 + `vor` + `vfirst.m` 替代；83.57% 标量 loop 份额显著下降。

**3. Baseline facts 回填**：hardware RVV 1.0 / VLEN=256；本函数 zero `v*`（首要 baseline finding）；wchar=4B（e32）；bound=逐元素扫描链主导；local period；precise_ip 缺失。

**4. 收益上界**：局部样本份额。main loop = **83.57%**。`baseline_gap: sampling metadata`（local period）→ 不得表述 workload 级 Amdahl 上界。

**5. 三维路由判定**：
- `current source`：compiler-generated scalar C（annotate C 源行 + `wcsmbs/wcschr.c` 核对）
- `implementation existence/reachability`：不存在 RVV 变体；无 dispatch slot → 修复需要"新建实现 + 新建接线"
- `function-level policy`：glibc riscv 已为 byte 字符串函数提供 RVV 变体但 **wcs* 宽字符族完全缺失** → 语义 row 命中；missing-`.S` 四证不满足独立 gate（如实记录）

**6. Implementation-shape proof**：不适用（非 policy-backed missing `.S`）。

**7. Related PRs**（`patterns/rvv_sentinel_scan_copy_implementation.md`，按 pattern 分组）：
`Related PRs：6 条 URL`（9b5e9799b6af、40fb13ac3de0、07122a9d8d45、512f99e71d57、bbc3867f8d8f、c275c424b324）。

## Phase 5 — Verification forecast / 验证预测：__GI___wcschr

- **消失/缩小侧**（锚定 Phase 3(a) 引用行）：`a76d4: lw a5,0(a0)`、`a76d6: beq a5,a1,a76e0`、`a76dc: bnez a5,a76d4` 应从 annotate 消失或显著缩小；83.57% 标量 loop 份额下降。
- **出现侧**（锚定 `patterns/rvv_sentinel_scan_copy_implementation.md` §Verification）：`vle32ff.v`（或 bounded load）、`vmseq`×2、`vor`、`vfirst.m` 出现在新 kernel；IFUNC dispatch 采用。
- **benchmark 覆盖**：wc 在首/中/末/未出现、wc==0、短中长、未对齐、page-boundary。
- **A/B 判据**：同一 workload 重跑 `perf annotate`；对比 main loop 份额与 bench-wcschr duration；核对 IFUNC 采用路径。
- **补采建议**：`perf evlist -v` 确认 precise_ip；libc.so `readelf -A`。

## Phase 6 — Completion check / 完成自检

| # | Check item | Status | Anchor 载荷 |
|---|---|---|---|
| 1 | 承诺声明兑现 | ✅ | `1/1 组；__GI___wcschr` |
| 2 | Phase 1 输出要求 | ✅ | `7 行 baseline；gap 标签：baseline_gap: sampling IP precision、sampling metadata、cache counters、build ISA (libc.so)；本函数 zero v* 为首要 baseline finding` |
| 3 | Phase 3 输出要求 | ✅ | `Class selection trace 8 项；Classes scanned: rows-string-memory.md; rows-asm.md; rows-operator-rvv.md；顶层 finding 1 个（RVV Sentinel Scan/Copy Implementation）；evidence 锚点：45.29 : a76d6: beq a5,a1、31.53 : a76dc: bnez a5,a76d4、6.55 : a76da: addi a0,a0,4；supporting 0；排除 7 条；推导式 1` |
| 4 | Phase 4 输出要求 | ✅ | `已读 pattern: rvv_sentinel_scan_copy_implementation.md；命中 row: RVV Sentinel Scan/Copy Implementation；引用短语：标量 sentinel scan 每次循环只检查一个元素（§Why this is slow）；The fix 含 before/after/correctness/风险/Profile signals；Related PRs：6 条 URL` |
| 5 | 路径合规 | ✅ | `模式 A；include class: rows-string-memory.md; rows-asm.md; rows-operator-rvv.md；finding 来自通过 gate 的 row；missing-.S/integration row 如实记录 gate 不成立` |
| 6 | Phase 5 两侧锚定 | ✅ | `消失侧：a76d6 beq a5,a1（Phase 3 引用行）；出现侧：patterns/rvv_sentinel_scan_copy_implementation.md §Verification` |
| 7 | 契约边界合规 | ✅ | `无实施询问/代码修改/补丁生成；交付止于证据、根因、The fix、验证预测` |

修正记录：无