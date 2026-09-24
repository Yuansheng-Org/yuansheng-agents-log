# AI 补丁审核报告

- reviewId: `pr-bp-mxnet-category-nn-basic-012-r1`
- patchCandidateId: `pc-bp-mxnet-category-nn-basic-012`
- patchPlanId: `pp-bp-mxnet-category-nn-basic-012`
- blueprintId: `bp-mxnet-category-nn-basic-012`
- 审核人: `opencode-comet-reviewer`（独立只读会话）
- 审核日期: 2026-09-21
- 被审对象: `.yuansheng/craft/mxnet/012_MapPlan_sv_saveto_Tensor_cpu_2_float_2_float_BinaryMapExp_plus_d/craft/patch.diff`
- 审核轮次: 1（`review_round: 0` → 首轮）

## 审核范围

四层审核：通用质量、约束保持（逐条核对 `constraints.mustPreserve`）、RISC-V 架构专项（`arch-scan`）、防幻觉锚点（finding 必须锚定 `patch.diff` 原文或蓝图字段）。

改动为单文件单 hunk（`PatchCandidate.gitDiff` 与 `patch.diff` 字节一致，已校验 `True`）：

```
diff --git a/CMakeLists.txt b/CMakeLists.txt
@@ -243,6 +243,20 @@ if(MSVC)
 1 file changed, 14 insertions(+)
```

**锚点区间**：`review-validate` 按 hunk 头的**旧文件**行号区间校验，即 `243..248`；本报告所有 finding 的 `line` 均落在该区间内（纯新增 hunk 下这是唯一可锚定的区间）。

**与 `PatchPlan.changes` 的偏差（需显式声明）**：`pp-bp-mxnet-category-nn-basic-012` 的 `changes` 列出 3 个文件（`src/operator/math_functions-inl.h`、`src/operator/mshadow_op.h`、`3rdparty/mshadow/mshadow/tensor_cpu-inl.h`），而本补丁只改 `CMakeLists.txt`。这不是漏做，而是**蓝图的步骤①首选方案经验证无效后，改用了蓝图同一条行动项中给出的另一个方案**（蓝图原文：「①消除 sqrt 的 errno 保持型 lowering——**核对并修改构建配置使该 TU/目标启用 -fno-math-errno（mxnet 的 CMake 编译选项）**，或 在 src/operator/math_functions-inl.h:44-46 把 float sqrt(float a) 从 return ::sqrtf(a) 改为直接落到硬件语义的 __builtin_sqrtf(a)」）。详见 F1 的实测证据。`src/operator/mshadow_op.h` 与 `tensor_cpu-inl.h` 的条目分别属于步骤①的间接调用点与步骤②，本轮未触及。

## RISC-V 架构审核

`arch-scan` 机器判定：`{"archSpecific": false, "matches": []}`。补丁为 CMake 构建开关，**不含**汇编、inline-asm、RVV intrinsic 或 ISA 特性代码。按规则跳过架构专项审核，仅做通用质量审核；`PatchCandidate.archReviewWarning` 已由工具自动写入。`archReview.status = not-applicable`。

（注：本补丁虽被工具判为非架构相关，但其**效果仅体现在 RISC-V 目标上**——`-fno-math-errno` 移除的正是 `fsqrt.s` 后接 `flt.s`/`fsflags`/`jalr sqrtf@plt` 的 errno 保持序列，该序列的存在性由 rank 012 的 annotate 在 RISC-V 上直接观测到。）

## 审核结果

`reviewResult: pass`（4 条 finding 全为 `suggestion`，无 `critical` / `major`）。

### 约束保持逐条核对（对照蓝图 `constraints.mustPreserve`）

| 蓝图 mustPreserve 条目 | 核对结论 |
|---|---|
| 数值语义：sqrt 对 a>0、a==0、a==-0.0、a<0、+inf、NaN 的结果必须与参考实现一致（RISC-V `fsqrt.s` 对负有限输入产生 canonical qNaN 并置 NV flag） | **满足**。`-fno-math-errno` 只删除 errno 保持序列，不改变 `fsqrt.s` 这条运算指令本身。已实测 22,874,381 例位级比对（含特殊值 ±0/±Inf/NaN/±NAN、子正规数 0x00000000–0x00200000 全枚举、正数 0x3f000000–0x3f800000 全枚举、负数 0xbf800000–0xc0000000 全枚举、4,000,000 组随机位模式），受保护版与 `-fno-math-errno` 版结果**逐位相同**。NV flag 语义亦一致：受保护序列在 `flt.s` 前用 `fsflags` 恢复标志，因此 `fsqrt.s` 置起的 NV 得以保留；去掉守卫后 `fsqrt.s` 仍置 NV。 |
| errno 合同：只有在证明算子路径不读取 errno 后，才可移除 errno 保持型 lowering；否则必须保留显式 slow path | **满足（已审计）**。全仓 `errno` 使用点：`src/storage/pooled_storage_manager.h:210`（`strerror(errno)`，在 `shm_*`/`mmap` 类系统调用后）、`src/storage/cpu_shared_storage_manager.h:168,179,184,188,206,211,217`（同型）、`src/io/image_io.cc:221,226`（std::ifstream 后）、`src/initialize.cc:52`（仅 `#include <cerrno>`）、`3rdparty/mshadow-ps/thread.h:131,188,193,198,203`（信号量系统调用后）。**没有任何一处是在数学函数调用之后读取 errno**，故算子路径不依赖数学函数的 errno 合同。 |
| 禁止使用 `-ffast-math` 等会改变 NaN/Inf 语义或重结合浮点加法的编译开关替代本项修复 | **满足**。选用的是 `-fno-math-errno`，与 `-ffast-math` 无关：前者只豁免 errno 设置（GCC 文档明确以 `sqrt` 为唯一单指令示例），不启用重结合、不启用 `-ffinite-math-only`、不改变 NaN/Inf 语义；后者会隐含 `-ffinite-math-only -funsafe-math-optimizations -fno-signed-zeros -fno-trapping-math`。 |
| 表达式求值语义：plus/div/mul/minus 的逐元素定义与广播（Broadcast1DExp 同列重复）语义不得改变 | **未触及**。补丁未改动任何算子定义或表达式代码。 |
| tail 与边界：shape[1] 不假设为 VLMAX 整倍数，shape[1]==0 行为不变；广播源列索引在 run 边界处取值不变 | **未触及**。补丁不引入向量代码，亦不改写索引计算。 |
| 公共 API 不变（MapPlan / MXNET_UNARY_MATH_FUNC / MXNET_UNARY_MATH_OP 宏接口不变） | **满足**。补丁未改动任何头文件或宏。 |

### 本补丁的验证证据（供追溯）

**A. 反汇编证据（rank 012 annotate，本补丁要消除的对象）**

```
 0.00% 11b420a frflags    a2
14.41% 11b420e flt.s      a5,fa0,fa2
13.55% 11b4212 fsflags    a2
16.09% 11b4218 fsqrt.s    fa0,fa0
 0.00% 11b428e jalr       -1450(ra) # 59cce0 <sqrtf@plt>
```

errno 保持序列（`frflags` / `flt.s` / `fsflags` / `bnez` / `jalr sqrtf@plt`）合计约占该函数样本的 **27.96%**（14.41% + 13.55%），**高于 `fsqrt.s` 自身的 16.09%**——即守卫比平方根本身更贵。这确认了蓝图诊断的根因。

**B. 方案有效性实测：`__builtin_sqrtf` 无效，`-fno-math-errno` 有效**

x86-64 `gcc -O2`（默认 `-fmath-errno`）下三种写法的代码生成：

```
sqrtf(a)                  -> pxor/ucomiss/ja + sqrtss + ret + jmp sqrtf@PLT   （有守卫）
__builtin_sqrtf(a)        -> pxor/ucomiss/ja + sqrtss + ret + jmp sqrtf@PLT   （有守卫，与上式完全相同）
(float)__builtin_sqrt((double)a) -> 同上
sqrtf(a) @ -fno-math-errno -> sqrtss + ret                                    （守卫消失）
```

`__builtin_sqrtf` 与 `sqrtf` 的指令序列**逐条相同** → 蓝图步骤①的备选方案「改为 `__builtin_sqrtf(a)`」是**编译器空操作**，不能消除守卫；只有 `-fno-math-errno` 能消除。这直接决定了本补丁的实现路径。

**C. 被否决的替代实现（记录以免重复尝试）**

| 方案 | 实测结论 |
|---|---|
| `__attribute__((optimize("no-math-errno")))` 加在 `math::sqrt` 包装函数上（可作用域精确） | **双重失败**：① 守卫**仍然存在**（实测 `sq_attr` 仍含 `pxor/ucomiss/ja/jmp sqrtf@PLT`）；② 该属性使函数**不再被内联**（`use_attr` 编译为 `jmp sq_attr`，`map_attr` 每元素 `call sq_attr`），逐元素引入函数调用，比原守卫更差。已否决。 |
| 在 `math_functions-inl.h` 用 RISC-V `fsqrt.s` 内联汇编特化 | 可作用域精确且语义等价（`fsqrt.s` 正是 mustPreserve 指定的参考语义），但本环境**无 RISC-V 汇编器**，asm 字符串无法验证；且把架构相关内联汇编引入通用数学包装头会显著提高维护成本。作为备选保留，本轮不采用。 |

**D. 数值等价性实测（见「约束保持」表首行）**

```
sqrtf bit-identity guarded vs -fno-math-errno: cases=22874381 diffs=0 -> BIT-IDENTICAL
```

**E. CMake 结构性检查**

改动文件在编辑前后的 `if/foreach/function/macro/while` 与对应 `endif/...` 嵌套深度均为 0（`HEAD` 与工作树一致），插入的 14 行内含 `if(MXNET_COMPILER_HAS_NO_MATH_ERRNO)` / `endif()` 一对，配对正确；新增代码位于 `else()`（非 MSVC 分支）内、`include(CheckCXXCompilerFlag)`（第 244 行）之后，`check_cxx_compiler_flag` 与 `if()` 语法均为标准用法。

**F. 未做的验证（如实声明）**

- **本环境无 `cmake` 可执行文件**（`/nix/store` 仅有 `*-cmake-*.drv` 未构建产物，系统无 `cmake`，Python 无 `cmake` 模块、无 `pip`），因此**未能实际配置/构建**以确认 `check_cxx_compiler_flag` 检出结果、`CMAKE_C_FLAGS` 拼接与 `-fno-math-errno` 实际出现在编译命令行。E 项仅为静态结构检查。
- **无法编译 mxnet**（`3rdparty/dmlc-core`/`dlpack`/`onednn` 为空；无 RISC-V 工具链），故未做端到端编译与 `perf annotate` 回归。
- B/D 的实测在 x86-64 上完成。依据：`-fmath-errno` 的守卫插入是 GCC 中端行为（三种写法在 x86 上生成同构的「比较 + 分支 + libm 调用」序列，与 RISC-V annotate 观测到的 `fsflags`/`flt.s`/`jalr sqrtf@plt` 结构一致），且 `-fno-math-errno` 的效果由 GCC 文档以 `sqrt` 为唯一单指令示例明确规定。但这仍是**跨目标外推**，必须在 RISC-V 上复测确认。
- 收益未实测：蓝图 `benefitUpperbound` 0.8388 为 `cpu-clock` + `local period` 语义下本函数内局部样本份额，非 workload 级加速比。

## 发现问题

| id | 位置 | 严重度 | 类别 | 摘要 |
|---|---|---|---|---|
| F1 | `CMakeLists.txt:246` | suggestion | 蓝图方案有效性 | 蓝图首选备选（`__builtin_sqrtf`）实测无效，已改用同一行动项中的 `-fno-math-errno` |
| F2 | `CMakeLists.txt:247` | suggestion | 影响范围 | 开关作用于所有非 MSVC 编译单元的全部单指令数学函数，比诊断热点更广 |
| F3 | `CMakeLists.txt:248` | suggestion | 验证完备性 | 无 cmake 可执行文件，构建配置改动未能实际验证；x86 结论属跨目标外推 |
| F4 | `CMakeLists.txt:245` | suggestion | 根因覆盖完整性 | 步骤②（`MapPlan` RVV 快速路径）未实施 |

详见同目录 `opencode-patch-review.json` 的 `findings[].evidence` / `suggestion`。

## 幻觉自检

| 维度 | 结果 | 说明 |
|---|---|---|
| 技术精度 | PASS | 未声称任何 RVV 指令已落地；`-fno-math-errno` 与 `-ffast-math` 的区别、`fsqrt.s` 语义、`__builtin_sqrtf` 无效性均有实测或文档依据。 |
| 声明溯源 | PASS | finding 的 `file`/`line` 均锚定 `patch.diff` 的 hunk（区间 243..248）；蓝图引文逐字取自 `bp-mxnet-category-nn-basic-012` 的 `diagnosis.recommendedFirstAction`、`constraints.mustPreserve` 与 blueprint 的 errno 合同条款；annotate 百分比逐字取自 rank 012 annotate 文件。 |
| 可解释性 | PASS | 「守卫 27.96% > fsqrt.s 16.09%」的机制、`-fno-math-errno` 为何既能消除守卫又不改数值、为何 `optimize` 属性双重失败，均可复现。 |
| 内部一致性 | PASS | `reviewResult: pass` 与 findings 严重度一致（全 `suggestion`）；`archReview.status: not-applicable` 与 `arch-scan` 输出一致；各 id 与产物文件一致；已显式声明与 `PatchPlan.changes` 的偏差。 |
| 安全 | PASS | 未改函数语义、未改浮点结果（22,874,381 例位级一致）、未改 NaN/Inf 语义、未改任何宏或公共 API、未引入 `-ffast-math`；新增开关带编译器能力检查，不支持的编译器自动跳过。 |

## 结论

**通过（pass）**。补丁实施蓝图步骤①的「构建配置」分支，用带能力检查的 `-fno-math-errno` 消除 `sqrt` 的 errno 保持型 lowering，逐条满足 `constraints.mustPreserve`，并以三重证据支撑：annotate 显示守卫占 27.96%（高于 `fsqrt.s` 的 16.09%）、`__builtin_sqrtf` 无效率的实测对照、22,874,381 例位级一致的数值等价性。

四条 finding 均为 `suggestion`。其中 **F1 最重要**：蓝图步骤①给出的首选备选（`__builtin_sqrtf`）经实测与 `sqrtf` 生成完全相同的受保护序列，属编译器空操作；本补丁据此改走同一行动项中的构建开关路径，并把与 `PatchPlan.changes` 的偏差显式记录在报告中。**F3 为本次最大的验证缺口**：环境内没有可用的 cmake，构建配置改动未经实际配置验证，且数值结论基于 x86 外推，须在 RISC-V 上以 `-fno-math-errno` 重建 `libmxnet.so` 后确认。
