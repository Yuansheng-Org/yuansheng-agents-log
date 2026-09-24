# PR description —— 017 Part A：reduce_with_axis 标量强度削减（独立补丁）

> 补丁文件：`mxnet_017_reduce_with_axis_partA.patch`（+49/−4，单文件
> `3rdparty/mshadow/mshadow/extension/reduce_with_axis.h`，git apply 可直接应用）
> 验证板：SG2044（XuanTie C920v2，VLEN=128），GCC 15.1，CI 真实参数 `-march=rv64gcv -mabi=lp64d -O2 -O3`

## 结论速览

| 项 | 值 |
|---|---|
| 触发路径 | `l2_normalization` kSpatial（`l2_normalization-inl.h:132` 前向 / `:212` 反向，3D `(N,C,HW)` reduce axis=2，`trailing_==1`）—— **真实可达生产路径** |
| 真机收益 | 107.40 ms → **50.33 ms（2.13×）**，ABBA ×3 轮、中位数、核心固定、形状 128×256×512 |
| 子项归因 | 索引解码强度削减（A1）单独 **−27.5%**（77.79 ms）；去 volatile（A2）单独 **±0%**；两者叠加才有 2.14× |
| 数值行为 | **A1 逐位相同**（32,770 样本 md5 一致，0 ULP 差）；**A2 会引入 FMA 收缩**（82.8% 样本变化，全局 max\|abs\| 9.537e-06）——已在代码注释与下文如实声明 |
| 跨架构 | 与架构无关（x86/ARM 同样受益），RVV 归约指令 0 条 |
| 补丁质量 | `git apply --check` 通过（mxnet 仓库根，`3rdparty/mshadow` 为 vendored 目录非 submodule）；`--whitespace=error-all` 通过；LF（0 CR） |

> **提交版实测（最终产物，非中间变体）**：用本补丁产出的头文件直接编译同一基准 TU
> （真实 CI flag，`leaky_relu.cc`）：**107.402 ms → 50.327 / 50.382 ms**，
> 输出 `O[0] = -3.716410160e+00` 与纯 A 变体 **逐位相同**，反汇编 RVV 归约指令 **0 条**。

## 改动内容（3 处，全为可移植标量）

### 1. `ReduceWithAxisStep` —— 加法归约去掉 volatile 内存往返（A2）
```cpp
template<typename DType>
MSHADOW_XINLINE void ReduceWithAxisStep(red::sum, DType &acc, DType val) { acc += val; }
template<typename Reducer, typename DType>
MSHADOW_XINLINE void ReduceWithAxisStep(Reducer, DType &acc, DType val) { Reducer::Reduce(acc, val); }
```
`red::sum::Reduce`（`base.h:987-992`）的 `volatile DType&` 累加器迫使部分和在每步
往返内存；加法归约改用寄存器累加，执行完全相同的 `dst += src` 序列。其余 reducer 保持
原有 `Reducer::Reduce` 契约不变（`volatile` 依然有效）。

### 2. 索引解码强度削减（A1）—— 把 k 循环里的除法改为增量进位
原式 `z = (x*size_+k)*trailing_+y; src_.Eval(z/last_, z%last_)` 每元素做 **2 次整数除法**
（RISC-V 无硬件除法，`div/rem` 是软件例程，慢 ~20–40 周期）。
改为：`base = x*size_*trailing_+y`，`q = trailing_/last_`，`r = trailing_%last_`，
每步 `zdiv += q; zmod += r; if (zmod >= last_) { zmod -= last_; ++zdiv; }`。
整数等价（`index_t` 为有符号 64/32 位，所有量非负，`zmod+r < 2*last_` 一次进位足够）。
**已验证逐位相同。**

### 3. `size_ == 0` 提前返回
循环不执行时避免 `last_ == 0` 的前置除法除零，并保留原语义（mask 路径 0，值路径 init）。

## 实测数据（真机，128×256×512，axis=2，核心 40，ABBA×3，中位数 ms）

| 变体 | 内容 | reduce_axis2_ms | vs 基线 |
|---|---|---|---|
| v0 | 基线 | 107.34（107.33–107.39） | — |
| v1 | 仅 A1（解码） | 77.79（77.78–77.83） | **−27.5%** |
| v2 | 仅 A2（去 volatile） | 107.14 | ±0% |
| v3 | A1+A2（= 本补丁） | 50.15（50.13–50.18） | **−53.3%（2.14×）** |

为什么 A2 单独无效、与 A1 叠加才有 2.14×：volatile 强制每步 store/load，除法也在关键
路径上；两者串在同一关键链上，只去其一，另一个仍是瓶颈。去 volatile 后编译器把
`acc + (lhs*rhs)` 收缩为 `fmadd.s`（0 → 2 条），并允许乱序调度，链长显著缩短。

## 数值行为（A2 必须如实声明）

| 对比 | 不同样本 | max abs | max ULP | max rel |
|---|---|---|---|---|
| v0 vs v1（A1） | **0（逐位相同）** | 0 | 0 | 0 |
| v0 vs v2（A2） | 27,140（82.82%） | 9.537e-06 | 71,296 | 7.786e-03 |
| v0 vs v3（本补丁） | 同 v2 | 9.537e-06 | 71,296 | 7.786e-03 |

A1 是纯索引算术，可证明且实测逐位相同（32,770 样本，md5 一致，0 ULP）。

A2 的差异来自 **FMA 收缩**（GCC `-O2`/`-O3` 默认 `-ffp-contract=fast`）——这是
「编译器把 `acc + (lhs*rhs)` 合并为一条 `fmadd.s`」，**不是累加顺序重排**，每个元素
仍只参与一次加法。误差因此表现为**绝对误差**而非相对误差，按结果量级分桶可见：

| \|基线值\| 区间 | 样本数 | max \|abs\| | max rel | max ULP |
|---|---|---|---|---|
| [1e-10, 1e-3) | 4 | 2.075e-06 | 7.786e-03 | 71,296 |
| [1e-3, 1) | 3,438 | 5.007e-06 | 2.601e-03 | 26,976 |
| **[1, 1e9)** | **23,698** | 9.537e-06 | **4.577e-06** | **47** |

即：**全局最大绝对误差 = 9.537e-06**，且量与结果量级基本无关（512 步 FMA 舍入累积，
≈ √512·ULP(10) 量级，实测吻合）。
- **占 72.3% 的常规量级结果（|x| ≥ 1）相对误差仅 ≤ 47 ULP / 4.6e-06**，属正常舍入水平。
- 出现 7.8e-03 大相对误差的只有 **4 个样本**，其基线值仅 ~2.7e-04——即结果本身接近
  相消（cancellation），此时 1e-5 的绝对扰动被放大成相对误差，属可预期的放大效应，
  而非精度失控。这些点的**绝对误差仍在 2e-06**。

已在代码注释与本文档中如实声明。若下游对该算子有逐位可复现要求，可选只取 A1
（单独 −27.5%，逐位相同），或用 `-ffp-contract=off` 关闭收缩。

## 为什么这是独立补丁（与 RVV 部分的关系）

原补丁（craft 017，+166/−6）还包含一个 `#if defined(__riscv) && defined(__riscv_v)`
的 RVV 单步归约内核（Part B），触发条件苛刻（`red::sum` + `float` + `trailing_==1` +
`BinaryMapExp<op::mul,…>`）。真机实测 **Part B 让同一形状从 50.15 ms 倒退到 ~97 ms**
（慢 ~2×，ABBA ×3 中位数），并额外扰动 93.0% 的样本。**Part B 已从本补丁中剔除**。
标量部分与架构无关，收益面更大、可独立验证，值得单独合入；RVV 内核需另行论证或重写。

## 影响面核查

- `reduce_with_axis<red::sum,false>` 调用点：`softmax_activation-inl.h:122`（axis=1，
  `trailing_>1`，本补丁无收益但无回归）、`l2_normalization-inl.h:114/132/191/212`
  （kChannel axis=1 无收益；**kSpatial axis=2 获 2.14×**）。
- `reduce_with_axis<red::maximum/minimum,true>`（argmax/argmin）调用点
  （`broadcast_reduce_op.h:673`、`np_broadcast_reduce_op.h:482`）：A1 同样受益且逐位相同；
  A2 对 argmax/argmin 走 `ReduceWithAxisStep(Reducer,...)` → 保持 `volatile` 路径，无数值影响。
- mask 路径与值路径都经过本补丁的循环改写，行为等价。
- 编译：真实 CI 命令单 TU 编译 `l2_normalization.cc`、`softmax_activation.cc` 通过；
  GCC 15.1 与基线 GCC（mxnet 2023 时代）均无告警；无新增依赖。
  在**真实 mxnet 树**上应用本补丁后按捕获的真实 CI 命令编译上述两 TU 均 PASS
  （22,636,672 B / 21,643,952 B 目标文件，仅 dmlc 既有 `-Wtemplate-id-cdtor` 告警）。

## 注意：本补丁不包含 `<riscv_vector.h>`

Part A 是**纯标量**改动，不含任何 RVV 代码，因此**没有**引入 `<riscv_vector.h>`。
原 017 补丁里那行 include 位于被剔除的 Part B 守卫内，随 Part B 一起移除。
若同一批次里还有其他依赖"017 顺带引入 `<riscv_vector.h>`"的补丁（本仓库的
`021`（`tensor_cpu-inl.h`）就是这种情况），**它们必须自己补上该 include**；
否则单独应用时仍会编译失败。这是 021 自身的缺陷，不是本补丁的回归。

## 后续建议

1. 若能接受 A2 的 FMA 末位变化（或改用 `-ffp-contract=off` 等开关策略），本补丁可直接合入；
   若必须逐位保持基线输出，可只取 A1（单独 −27.5%，逐位相同）。
2. 给 OpPerf 补一个 `l2_normalization`（kSpatial）用例，让该收益进入验收可见范围。
