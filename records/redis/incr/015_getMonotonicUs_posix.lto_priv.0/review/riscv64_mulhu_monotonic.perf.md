# PR-D mulhu Monotonic（mtime 换算）· 性能数据

**补丁**: `riscv64_mulhu_monotonic.patch`（PR-D：`getMonotonicUs_riscv` 的 `mtime / ticksPerUs` 除法 → `mulhu` 倒数乘法，`recip = ceil(2^64/tps)` 一次性计算）
**平台**: K3 (Spacemit X100, riscv64, timebase 100MHz) / sg2044-board (C920) · gcc-15.1

## 性能
- mtime→us 换算路径（微基准）：**~+5.3%**
- 抖动：与精确除法 **≤1us 偏差**（64 位全范围验证）
- 适用场景：`getMonotonicUs` 每次调用均执行该换算（无硬件除法器核如 X100 用软件除法多周期）

## 正确性
- `test_mulhu.c` 全范围（4 时基 × 1M 全序采样 + 随机 + 边界）✅
- 官方 `unit/expire` 单测 + 全量 `make test` ✅

## 结论
消除每次时钟读取的除法长依赖链，换算路径 +5.3%；数学语义与精确除法一致（偏差 ≤1us）。