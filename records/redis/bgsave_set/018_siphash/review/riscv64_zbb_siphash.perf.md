# PR-C Zbb SipHash · 性能数据

**补丁**: `riscv64_zbb_siphash.patch`（PR-C：Zbb `roli` 指令级加速 + U8TO64_LE 非 Zicclsm 对齐装载修正 + 启动期函数指针运行时分派）
**平台**: K3 (Spacemit X100, riscv64, 8 核, timebase 100MHz) · gcc-15.1

## 性能（siphash 延迟相对基线）
| 构建/模式 | 归一化耗时 | 提升 |
|---|---|---|
| base（portable） | 1.000 | — |
| **Zbb 直接 TU** | 0.946 | **-15.2%** |
| **运行时 dispatch（Zbb 机器）** | 0.912 | **-18.3%** |
| dispatch + `SIPHASH_DISABLE_ZBB` | 1.194 | +6.9%（回落 wrapper 开销，功能正常）|

## 正确性
- `tests/unit/riscv-siphash.tcl`（默认与 `SIPHASH_DISABLE_ZBB` 双模式）3/3 PASS ✅
- 位精确差分（HEAD vs 补丁，11 输入 × 双函数）逐字节一致 ✅

## 结论
Zbb 机器 ~**-15%（直接）/-18%（dispatch）** siphash 延迟；非 Zbb 回落仅 +4~7% 单次间接跳转开销。