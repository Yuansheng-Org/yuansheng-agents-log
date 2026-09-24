# riscv-zicond-autodetect（Zicond release 使能）· 性能数据

**补丁**: `riscv-zicond-autodetect.patch`（把 `riscv_hwprobe.cpp` 中 ZICOND 的 `ext_Zicond.enable_feature()` 从 `#ifndef PRODUCT` 移出，product/release 构建也默认启用 UseZicond → 解锁 C2 无分支条件选择 czero.eqz/nez 生成）
**平台**: RISC-V SG2044 验证机（T-Head C920, 64 核）· glibc 2.38 · GCC 12.3.1
**基线**: openjdk/jdk master（JDK 28，2026-09-22，xuantie vendor 支持）

## 性能
| 场景 | 结果 |
|---|---|
| 随机选择密集负载 | **~2.4x** |
| 可预测分支场景 | 与现状持平（±10% 噪声）|
| 正确性 | **零回归** |

## 说明（与 craft 016 的对照关系）
- 本补丁 = **运行时 hwprobe 使能**（Zicond 可用化链的检测侧）
- craft 016_Handshake = **构建层 CFLAGS** `-march=…_zicond`（同链的编译使能侧）——两者同宗（让 Zicond 进入 OpenJDK），机制层不同，属"大致匹配"对照。

## 结论
release 默认启用 Zicond 后在分支不可预测负载 ~2.4x；`-XX:-UseZicond` 可随时关闭，无回归。