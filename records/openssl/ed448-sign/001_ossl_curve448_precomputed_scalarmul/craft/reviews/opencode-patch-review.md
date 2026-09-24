# AI 补丁审核报告

## 审核范围

- 补丁：`include/internal/constant_time.h`（`constant_time_lookup` 新增 `__riscv_vector` 门控的向量化 masked-select OR 累加，并补 `<riscv_vector.h>` 包含）
- 蓝图：`bp-openssl-ed448-sign-001`（`ossl_curve448_precomputed_scalarmul`，ed448-sign rank 001）
- 根因（`code_path`，probable）：`constant_time_lookup` 内层 j-loop 以逐字节标量实现 masked select + OR 累加（每字节 8 条指令、3 次内存操作），硬件与 build 均支持 RVV 1.0，但该循环未被编译器自动向量化（autovec 覆盖缺口）。
- 审核输入事实来源：`patch.diff`、`patch-plan.json`、`patch-candidate.json`、`blueprint`；本机交叉编译与 qemu 位精确验证记录。

## RISC-V 架构审核

`arch-scan` 判定：`archSpecific: true`（命中 `rvv-intrinsic` 与 `riscv-macro`），触发架构专项审核。

1. **指令/特性在目标 ISA 内**：补丁仅使用基础 RVV 1.0 指令（`vsetvl`/`vle8`/`vand.vx`/`vor.vv`/`vse8`，e8/m8）。目标硬件 `SpacemiT X100`（RVV 1.0，VLEN=256）覆盖。
2. **vsetvl/vtype 与数据流一致**：`__riscv_vsetvl_e8m8(rowsize - j)` 返回 `min(remaining, VLMAX)`；e8/m8 VLMAX=128（VLEN=128）/256（VLEN=256），strip-mining 以 `j += vl` 精确推进，无越界读/写（tail 默认 agnostic，仅写有效元素）。
3. **tail/mask policy、标量↔向量边界**：`mask`（0x00/0xFF）为行内不变量，经 `vand.vx` 标量操作数应用，与标量 `constant_time_select_8(mask, x, 0)` 的 `mask & x` 逐字节一致；`vor.vv` 累加与标量 `out[j] |= ...` 一致。
4. **运行时 ISA 分发**：`#if defined(__riscv_vector)` 编译期 gate + `vl > 0` 运行时 gate；无 vector 时回退标量循环，VLEN 无关正确性。
5. **无 x86/ARM 混入**：仅 RISC-V 基础向量指令。

**验证记录**（本机真实执行，非推断）：
- `riscv64-linux-gnu-gcc -march=rv64gcv -mabi=lp64d -O2 -static` 编译通过。
- qemu `-cpu max,v=true,vlen=256,vext_spec=v1.0` 下，20,000 组随机测试（rowsize 1..256、numrows 1..20、idx 含下溢 0..numrows+1）向量 vs 标量 `constant_time_lookup` 逐字节一致（CT_LOOKUP VEC MATCHES SCALAR）。
- 未在真实 K3 硬件跑 ed448-sign/ed448-verify 回归（环境无板卡），作为风险披露，不阻断补丁。

## 审核结果

- **根因解决**：`constant_time_lookup` 内层循环改为 RVV e8/m8 向量化 masked-select OR 累加，将逐字节标量（8 指令/字节、3 内存访问/字节）压缩为向量组处理，命中"autovec 覆盖缺口"根因。
- **约束保持**：`mustPreserve`（常数时间不变量、`out` 先 memset 后逐行 OR 累加语义、rowsize=192 无越界、table/out 无别名假设）全部保持——向量路径与标量参考逐字节一致（20k 随机向量 PASS），mask 全 0/全 1/随机交错均一致，无数据相关分支。
- **最小性**：变更聚焦 `constant_time_lookup` 单函数及头文件包含，含注释约 26 行，无无关重构。
- **产物一致性**：`PatchCandidate.gitDiff` 与 `patch.diff` 一致，`changedFiles=[include/internal/constant_time.h]`。
- **安全**：无硬编码密钥、无危险命令/路径；向量 select 为常量时间。

## 发现问题

| 严重度 | 类别 | 说明 |
|--------|------|------|
| suggestion | blueprint 数据质量 | blueprint `rootCause.candidateFiles` 指向 `include/crypto/constant_time.h`，实际 `constant_time_lookup` 定义于 `include/internal/constant_time.h`。本次补丁已正确落在 `include/internal/constant_time.h`，建议 Trace 侧修正 candidateFiles。 |

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

补丁准确解决 `ossl_curve448_precomputed_scalarmul` 根因（`constant_time_lookup` 向量化 masked-select OR 累加），行为保持、最小、常数时间，已通过交叉编译与 qemu 位精确验证，通过 RISC-V 架构专项审核与防幻觉锚点校验。审核通过（pass）。
