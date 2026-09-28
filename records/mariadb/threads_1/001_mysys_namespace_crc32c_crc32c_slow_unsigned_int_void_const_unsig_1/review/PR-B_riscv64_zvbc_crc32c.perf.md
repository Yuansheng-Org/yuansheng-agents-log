# PR-B2 Zvbc 向量 CRC32C（修复后版本）· 性能数据

**补丁**: `PR-B_riscv64_zvbc_crc32c.patch`（内容为 **PR-B2 修复版**：K-lane 向量折叠，vlseg2e64 段加载解交织 + vclmul_vx 常数广播，主循环零标量提取）
**平台**: k3 内部验证机（Spacemit X100）· 8 核/31GB · VLEN=256 · gcc 14.3.0
**对照**: 官方 `crc32c_slow`（slicing-by-4）

## 性能（MB/s）
| len | slow | PR-B2 Zvbc | speedup |
|---|---|---|---|
| 128 B | 289 | 1147 | 4.0x |
| 256 B | 296 | 2105 | 7.1x |
| 1 KiB | 297 | 5466 | 18.5x |
| 4 KiB | 297 | 9116 | 30.7x |
| 64 KiB | 296 | 11398 | **38.7x** |

## 正确性
- 官方 crc32-t.c **36/36** + RFC 3720 + 500 随机 + 100 链式 + 边界/非对齐 ✅
- 与 PR-A（Zbc 标量）同数学（k1..k4 + Barrett）→ **位精确一致**

## 结论
Zvbc 向量路径 ~**4–38.7x**（相对 PR-A 再提升一个量级）；无 Zvbc 时自动降级 Zbc 标量/软件，可独立回滚。本文件对应为 B2 修复版（fix 已合入，非基版 PR-B）。