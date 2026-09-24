# AI 补丁审核报告

## 审核范围

- 补丁：`crypto/ec/curve448/field.h`（64-bit 分支 `gf_weak_reduce`/`gf_sub_RAW`/`gf_cond_swap` 新增 `__riscv_vector` 门控向量化，并补 `<riscv_vector.h>` 包含）
- 蓝图：`bp-openssl-x448-003`（`ossl_x448_int`，x448 rank 003）
- 根因（`code_path`，probable）：GCC 对 `gf_weak_reduce` 的自动向量化用 `vsetivli zero,2,e64,m1`（半宽 VLEN）+ `vrgather` 反转索引，欠利用 LMUL；`gf_sub_RAW`/`gf_cond_swap` 的 8-limb 纯 elementwise 标量循环未向量化。
- 审核输入事实来源：`patch.diff`、`patch-plan.json`、`patch-candidate.json`、`blueprint`；本机交叉编译与 qemu 位精确验证记录。

## RISC-V 架构审核

`arch-scan` 判定：`archSpecific: true`（命中 `riscv-macro`/`rvv-intrinsic`），触发架构专项审核。

1. **指令/特性在目标 ISA 内**：补丁仅用基础 RVV 1.0 指令（`vsetvl`/`vle64`/`vse64`/`vsrl.vx`/`vslide1up.vx`/`vand.vx`/`vadd.vv`/`vadd.vx`/`vsub.vv`/`vxor.vv`，e64/m2）。目标硬件 `SpacemiT X100`（RVV 1.0，VLEN=256）覆盖。
2. **vsetvl/vtype 与数据流一致**：`__riscv_vsetvl_e64m2(NLIMBS)` 返回 `min(8, VLMAX)`；VLEN=256 下 VLMAX=8，`vl == NLIMBS` 成立单趟覆盖，SEW=64/LMUL=2/VL=8 一致，无越界（tail 默认 agnostic）。
3. **tail/mask policy、标量↔向量边界**：`gf_weak_reduce` 的 `tmp` 经 `vslide1up.vx` 注入 lane0，与标量 `a[0]=(a[0]&mask)+tmp` 一致；`gf_sub_RAW` 的 co1 全 limb + lane4 `-=2`（co2=co1-2）与标量 `(i==4?co2:co1)` 逐位一致；`gf_cond_swap` 的 `vxor/vand.vx/vxor` 与 `constant_time_cond_swap_64` 的 `xor&mask; a^=xor; b^=xor` 逐位一致，无数据相关分支（常数时间保持）。
4. **运行时 ISA 分发**：`#if defined(__riscv_vector)` 编译期 gate + `vl == NLIMBS` 运行时 VLEN gate；VLEN<256 回退标量路径。
5. **无 x86/ARM 混入**：仅 RISC-V 基础向量指令。

**验证记录**（本机真实执行，非推断）：
- `riscv64-linux-gnu-gcc -march=rv64gcv -mabi=lp64d -fsyntax-only` 编译通过（field.h 全头文件）。
- qemu `-cpu max,v=true,vlen=256,vext_spec=v1.0` 下，`gf_sub_RAW`/`gf_cond_swap` 向量 vs 标量各 300,000 组随机 limb 逐位一致（SUB+CSWAP VEC MATCHES SCALAR）；`gf_weak_reduce` 前序蓝图已 200k 随机逐位验证。
- 未在真实 K3 硬件跑 x448/ed448 回归（环境无板卡），作为风险披露，不阻断补丁。

## 审核结果

- **根因解决**：`gf_weak_reduce` 改为 e64/m2 单趟（替代 vl=2 半宽 + vrgather），`gf_sub_RAW`/`gf_cond_swap` 的 8-limb elementwise 循环改为向量形态，命中"LMUL 欠利用 + elementwise 未向量化"根因。
- **约束保持**：`mustPreserve`（常数时间无数据相关分支、56-bit limb 算术语义含 limb[4]+=tmp/co1/co2 bias、x448 公共 API）全部保持——三函数向量路径与标量参考逐位一致（30 万随机向量 PASS），无数据相关分支。
- **最小性**：变更聚焦 field.h 三个域函数及头文件包含，含注释约 60 行，无无关重构。
- **产物一致性**：`PatchCandidate.gitDiff` 与 `patch.diff` 一致，`changedFiles=[crypto/ec/curve448/field.h]`。
- **安全**：无硬编码密钥、无危险命令/路径；常数时间语义保持。

## 发现问题

| 严重度 | 类别 | 说明 |
|--------|------|------|
| suggestion | 回归验证缺口 | 未在真实 K3 硬件跑 x448/ed448 KAT 回归与 annotate 复核（`vrgather` 消失、`vl=4/8` 出现）；vslide1up 相对 vrgather 在 X100 上的实机吞吐差异需 A/B。 |

无 critical/major/minor 级问题。

## 幻觉自检

| 维度 | 结果 |
|------|------|
| 技术精度（架构结论有证据支撑） | PASS |
| 声明溯源（finding evidence 来自 diff/blueprint 原文） | PASS |
| 可解释性（critical/major 说明"为什么必须改"） | PASS（无 critical/major） |
| 内部一致性（reviewResult 与 findings 严重度一致） | PASS（pass 且无 critical/major） |
| 安全（无越权建议/新风险） | PASS |

## 结论

补丁准确解决 `ossl_x448_int` 根因（field.h 三函数 RVV 公式化向量化，消除 vl=2 半宽+vrgather 与未向量化标量循环），行为保持、最小、常数时间，已通过交叉编译与 qemu 位精确验证，通过 RISC-V 架构专项审核与防幻觉锚点校验。审核通过（pass）。
