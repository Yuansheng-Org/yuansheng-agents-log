# PR description —— 063：GELU(erf) 反向 `gelu_erf_grad` 的 RVV 快速路径

> 补丁文件：`patch.diff`（+182/−1，两文件：
> `src/operator/mshadow_op.h`（+168/−1）、`src/operator/leaky_relu-inl.h`（+14/−0））
> 验证板：SG2044（XuanTie C920v2，64 核，RVV 1.0，**VLEN=128**），GCC 15.1.0，
> CI 真实参数 `-march=rv64gcv -mabi=lp64d -O2 -g -fno-omit-frame-pointer -std=gnu++17 -fopenmp`
> 基线：`apache/mxnet` master `b84609d3fc73d20929c114eab95faaa56e6c5ede`（2023-01-26）

## 结论速览

| 项 | 值 |
|---|---|
| 触发路径 | `LeakyReLU(kGELU_ERF)` 反向（`leaky_relu-inl.h:407`）—— 该算子**全仓库唯一**调用点 |
| 真机收益 | 标量 172.62 ms → RVV **32.91 ms = 5.25×**（保守区间 5.2×～7.0×，标量侧读数波动大） |
| 正确性 | RVV 与标量对 double 参考的 `maxAbs` **逐项完全相同**；相对误差 5.5e-07 |
| 上游测试 | `tests/python/unittest/test_operator.py:636` 的 `test_gelu`（`rtol=1e-3, atol=1e-5`）**违规 0/0**，余量 ×8390 / ×168 |
| 附带修复 | 消除 `erf_grad` 的 **per-element fp64 往返**：`erf_grad::Map<float>` 双精度指令 **3 → 0 条**（实测反汇编） |
| 守卫 | 全部 RVV 代码在 `#if defined(__riscv) && defined(__riscv_v)` 内；非 RISC-V 平台完全惰性 |
| 唯一缺口 | 现有 OpPerf 用例表里没有 GELU/erf，因此**收益不可被现行验收覆盖**（详见「验收可见性」） |

## 改动内容

### 1. `erf_grad` 的 float 显式特化（**无守卫，所有平台生效**）

原定义是一行宏：
```cpp
MXNET_UNARY_MATH_OP(erf_grad, 2.0 / math::sqrt(PI) * math::exp(-(a * a)));
```
改为显式 struct + `float` 特化：
```cpp
struct erf_grad : public mxnet_op::tunable {
  template <typename DType>
  MSHADOW_XINLINE static DType Map(DType a) {
    return DType(2.0 / math::sqrt(PI) * math::exp(-(a * a)));   // 泛型：保持原 double 路径
  }
};

template <>
MSHADOW_XINLINE float erf_grad::Map<float>(float a) {
  return 2.0f / math::sqrt(PI) * math::exp(-(a * a));           // float：整链回到 float32
}
```

**根因核实（不是猜测）**：
- `PI` 在 `mshadow_op.h:65` 是 **`const float`**（`__constant__` 版在 `:59`），**不是 double**；
- `math::sqrt` 由 `MXNET_UNARY_MATH_FUNC(sqrt)`（`math_functions-inl.h:91`）生成，有
  `float sqrt(float)` / `double sqrt(double)` 两个重载 ⇒ `math::sqrt(PI)` **返回 float**；
- 所以那条 **double 提升来自原来的 `2.0` 双精度字面量**（`double / float → double`，
  再把 `exp` 也拖进 double）。

**真机 A/B 反汇编（直引真实 `mshadow_op.h`，`-O2 -march=rv64gcv`）**：

| `probe::call`（即 `erf_grad::Map<float>`） | 双精度指令数 | 反汇编 |
|---|---|---|
| 基线 | **3** | `fcvt.d.s` → `fmul.d` → `fcvt.s.d` |
| 本补丁 | **0** | 仅 `fmul.s` |

即：补丁注释所述"消除 per-element `fcvt.d.s`/`fmul.d`/`fcvt.s.d` 往返"**完全成立**。

### 2. `gelu_erf_grad` 的 RVV float32 快速路径（有守卫）

新增 `namespace gelu_erf_grad_rvv`，含三部分：

- **`exp_f32m1(x, vl)`** —— 自带 range-reduced 向量指数，**不调用 libm**：
  Cephes 风格 `x = k*ln2_hi + k*ln2_lo + r`（`k = round(x*log2e)`，`|r| ≤ ln2/2`），
  6 次多项式求 `e^r`，再用 `2^k` 精确缩放。精度约 **1 ULP of `expf`**（正常范围）；
  `x ≤ -87.33654475f`（含 `-inf`）flush 到 0（libm 那里本是次正规/0，绝对差 < 2^-126）；
  NaN 经 `vmfne(x,x)` 正确传播。
- **`gelu_erf_grad_block_f32m1`** —— 一个连续块的完整链：
  `t = a/√2` → `exp(-t²)` → `erf_grad(t) = (2/√π)·exp(-t²)` →
  `inner = b/a + (0.5f·a·eg)/√2` → `out = g·inner`，全程 `e32m1`。
- **`gelu_erf_grad_f32`** —— 固定 VL 主循环 + 运行时 VL 尾巴（**不会 over-read / over-write**）。

### 3. 分发钩子（无 ODR 风险）

```cpp
template <typename xpu, typename DType>
struct gelu_erf_grad_vec {           // 泛型：报告"不支持"
  static inline bool Launch(...) { return false; }
};
#if defined(__riscv) && defined(__riscv_v)
template <>
struct gelu_erf_grad_vec<mshadow::cpu, float> {   // 特化：跑 RVV 并返回 true
  static inline bool Launch(...) { gelu_erf_grad_rvv::gelu_erf_grad_f32(...); return true; }
};
#endif
```

调用点（`leaky_relu-inl.h:407` 之前）：
```cpp
if ((req[leakyrelu::kData] == kWriteTo || req[leakyrelu::kData] == kWriteInplace) &&
    mshadow_op::gelu_erf_grad_vec<xpu, DType>::Launch(s, gdata.size(0)*gdata.size(1)*gdata.size(2),
                                                      gdata.dptr_, grad.dptr_, data.dptr_, output.dptr_)) {
  break;
}
```

- **不特化 `mxnet_op::Kernel`** —— 只用一个独立钩子模板，因此**不产生类模板特化的 ODR 问题**
  （对比 019 v1 曾把 `Kernel` 偏特化放在 `activation-inl.h` 而踩到 IFNDR）。
- 钩子只做**纯存储**语义，故仅 `kWriteTo`/`kWriteInplace` 走向量；`kAddTo` 会回退标量路径
  （LeakyReLU 反向实际用的是 `kWriteTo`，影响面小 —— 已在补丁注释中声明）。
- 其他 dtype 由泛型模板返回 `false`，原 `MXNET_ASSIGN_REQ_SWITCH` 标量内核照旧执行。

## 实测数据

**性能**（真机，16 MiB float32，3 输入 1 输出；核心固定；5 次预热 + 25 次取中位数）：

| 变体 | 中位数 | 吞吐 |
|---|---|---|
| 标量 `gelu_erf_grad::Map` 循环 | **172.618 / 229.437 ms**（波动 ~33%） | 0.39 GB/s |
| RVV `gelu_erf_grad_f32` | **32.906 / 33.001 ms**（稳定） | 2.04 GB/s |

⇒ **5.25×**（保守值；标量上界下至 5.2×，RVV 稳定侧可达 7.0×）。
最大绝对误差 `2.980e-08`，最大相对误差 `5.486e-07`。

**正确性 / 精度（逐项闭合）**：

| 检查项 | 结果 |
|---|---|
| 上游自带测试 `test_gelu` | ✅ `maxAbs=5.960464e-08`、`maxRel=1.191884e-07`，**违规 0/0**，余量 **×8390（rtol）/ ×168（atol）** |
| RVV vs 标量 vs double 参考的 `maxAbs` | ✅ **逐项完全相同**（4 组：1.342259e-07 / 1.342259e-07；3.168575e-07 / 3.168575e-07；1.669792e-03 / 1.669792e-03；2.151335e-08 / 2.151335e-08） |
| 无守卫的 `erf_grad::Map<float>` 数值影响 | ≤1 ULP：400,012 样本中 53,390（13.35%）不同，**全部恰好 −1 ULP，从无 +1**；`\|x\| ≥ 10` 全同 |
| 大 ULP 值 | 下溢假象：所有 `maxULP ≥ 1024` 样本落在 `x ≤ −87.33654475f` 下溢区（12,039 个样本 ≤ 1e-38）；`maxULP = 2147483643` 是 0 与次正规数的比特距离，绝对误差极小 |

## 调用点与宏替换核查（无漏接）

- **`gelu_erf_grad` 全仓库只有 1 个使用者**：`src/operator/leaky_relu-inl.h:407`
  （`LeakyReLU(kGELU_ERF)` 反向，`op_with_req<backward_grad_tuned<gelu_erf_grad>, Req>`）。
  `activation-inl.h` 内没有任何 gelu 分派 ⇒ **没有漏接的调用点**，钩子正装在该处。
  （另在 `operator_tune.cc:415` 有 `IMPLEMENT_BINARY_WORKLOAD_BWD` 注册，只影响调优表。）
- **`MXNET_UNARY_MATH_OP` 替换无副作用**：该宏展开就是
  `struct name : mxnet_op::tunable { template<typename DType> MSHADOW_XINLINE static DType Map(DType a) { return DType(expr); } }`
  （`mshadow_op.h:75-81`），与本补丁手写的 struct **结构完全一致** ⇒ 不存在丢失 half/bf16
  特化的问题（该算子本来也没有 half/bf16 特化）。
- **`erf_grad` 另有一个使用者**：`src/operator/tensor/elemwise_unary_op_basic.cc:1025`
  注册 `_backward_erf`（`ElemwiseBinaryOp::Compute<cpu, unary_bwd<mshadow_op::erf_grad>>`）。
  该路径同样吃到 float 特化的收益（去掉 per-element fp64 往返），且已在
  「无守卫 float 特化」精度核对中一并覆盖。

## 编译验证

- **单补丁即可编译**（无隐式依赖）：真实 CI 命令编译 `src/operator/leaky_relu.cc` **通过**
  （rc=0，0 error；向量指令 +99）。
- `operator_tune.cc` 经 `-fsyntax-only` 验证 **0 error**（`-O2` 全量编译因该 TU 产生 300 MB
  目标文件而超时，属资源问题非编译失败）。
- 与同批次的 61 PartA、65v2 三条补丁**可依次干净应用并共同编译**（见同目录验证记录）。

## 验收可见性（唯一缺口，需上游/验收方决策）

补丁只作用于 `LeakyReLU(kGELU_ERF)` 反向与 `erf` 反向。**现行 10 个 OpPerf 用例表里没有
GELU / erf 算子**（`activation-relu` 对应的是 `kReLU`，不是 `kGELU_ERF`），
因此这条 5.25× 的收益**无法被当前验收流程覆盖**。

建议二选一：
1. 在 OpPerf 里补一个 `leaky_relu`（`act_type="gelu"`）+ `run_backward=True` 的用例，
   使该收益进入验收可见范围；
2. 或明确接受"收益存在但不在用例表内"，按内核级证据（本文档）合入。

