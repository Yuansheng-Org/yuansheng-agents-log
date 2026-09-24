# PR-1（序号 76）性能数据 · 非 MSVC 构建下启用 `-fno-math-errno`

**补丁**：`PR1_012.diff`（1340 B，md5 `2f218fbb757c79c2d4e0586b468c58a7`，**1 文件 / 1 hunk**，LF）
**对应单条补丁**：`mxnet_012.diff`（清单序号 **76**，组 E）
**改动**：`CMakeLists.txt`，+14 行
**依赖**：无 —— 可独立提交（与 PR-2 / PR-3 无关）
**数据来源**：`mxnet_PR描述_提交材料.md` § PR-1

---

## 一、测量条件

| 项 | 值 |
|---|---|
| 板卡 | **SG2044**，64 核，rv64gcv |
| 编译器 | GCC 15.1 |
| 负载 | `sqrt`，4 Mi `float32` |
| 轮次 | 200 轮 |
| 方法 | ABBA 配对、核心固定（pinned core）、取中位数 |
| 用例来源 | OpPerf `category-unary` 算子清单**显式含 `sqrt`** ⇒ 非合成微基准 |

---

## 二、性能数据

| 用例 | 基线 | 加 `-fno-math-errno` | 比值 |
|---|---:|---:|---:|
| **`sqrt`** | 107.068 ms | **6.379 ms** | **16.79×** |
| `fma`（对照） | 16.961 ms | 17.053 ms | 噪声带内 |
| `exp`（对照） | 114.597 ms | 114.783 ms | 未变 |

**`exp` 对照的意义**：它同样是 libm 调用，但**没有单指令硬件 lowering**，因此读数不动。
⇒ 收益来源不是"关掉了 errno 检查"这件事本身，而是"**硬件路径变为可达 + 外层循环被向量化**"。

---

## 三、生成代码（机制证据）

整 TU 汇编（同一源码 ±flag）：

| | insns | `vfsqrt.v` | `fsqrt.s` | `frflags` | `flt.s` | `fsflags` | 调到 libm |
|---|---:|---:|---:|---:|---:|---:|---:|
| base | 1104 | 0 | 2 | 3 | 3 | 3 | 2 |
| **+flag** | 1089 | **2** | 0 | **0** | **0** | **0** | **0** |

即：无 flag 时 GCC 为每个元素生成 `frflags / flt.s / fsflags` + 条件跳转 + `call sqrtf`
的 errno 菱形；有 flag 时变成：

```
vsetvli; vle32.v; vfsqrt.v; vse32.v
```

errno 守卫在逐元素算子内核里跑一次，其成本高于 `sqrt` 本身。

### 3.1 负载关键点（必须成立，否则整个补丁是空的）

补丁锚点在 `CMakeLists.txt:245`（`set(CMAKE_C_FLAGS ... -Wall)`）**之后**、
`:256`（`set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_C_FLAGS}")`）**之前**
⇒ C++ 编译单元**确实**拿到 `-fno-math-errno`。已实测证明（源码两处 `CMakeLists.txt` 行号）。

### 3.2 早期 per-sample 画像（佐证 errno 菱形占比）

`perf annotate` 于 `category-nn-basic` rank 012：`flt.s` 14.4% + `fsflags` 13.6% vs `fsqrt.s` 16.1%
⇒ 守卫开销与 `sqrt` 本体同量级。

---

## 四、正确性 / 数值语义

| 检查项 | 结果 |
|---|---|
| 逐位一致性 | 同源码加 / 不加该 flag，结果**逐位相同** |
| 摘要 | 4110 个输出（含 ±0、±Inf、NaN、denorm、FLT_MAX / FLT_MIN）原始比特的 FNV-1a = `559eab036eb5f154`，两种构建相同；NaN 计数相同（**4**） |
| 唯一行为变化 | 已文档化的那一条：负输入不再置 `errno`（33 个 `EDOM` → 0） |
| 浮点异常 | `fenv` 的 `FE_INVALID` 两种构建**均仍照常置位** |
| 与 `-ffast-math` 的区别 | **不**重结合浮点运算、**不**改变 NaN / Inf 语义 |

---

## 五、收益结论与复核要点

| 项 | 结果 |
|---|---|
| 是否 RISC-V 专属 | **否**。x86 / aarch64 在无 `-ffast-math` 时同样受 errno 约束；RISC-V 上收益特别大，是因为它**同时解锁了 RVV 向量化** |
| 收益 | **16.79×**（`sqrt`，N = 4 Mi） |
| 数值影响 | **逐位相同**；唯一变化是负输入不再置 `errno=EDOM` |
| 仓库内 errno 依赖 | 全仓 **10 处** `errno` 使用全部跟在 syscall / stdio 之后（`shm_open`/`munmap`/文件 IO），**无一处读数学函数的 errno** |
| CI 覆盖 | ✅ `sqrt` 在 OpPerf `category-unary` 算子清单里（`unary_operators.py` 显式含 `'sqrt'`），且 `category-unary` 在 CI 默认全类别列表**第 1 行** |
| 风险 | 低（有 `check_cxx_compiler_flag` 守卫；编译器不支持该选项时不受影响） |

---

## 六、数值口径

- 源文档复核要点记 **16.79×**（本节采纳）；同文档正文另记 `16.8×`，属**三位有效数字的写法差异**，非数据冲突。
  由 `107.068 / 6.379 = 16.784`（若用更未舍入的原始中位数则为 16.79×）。
- 对照行比值：`16.961/17.053 = 0.995`（fma，噪声带）、`114.597/114.783 = 0.998`（exp，无变化）
  —— 两者均**不是**收益，**不可**当作"该 flag 对其他算子有益"的证据。
