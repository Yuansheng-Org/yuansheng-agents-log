# PR-B Zvbc 向量 CRC64 · 性能数据

**补丁**: `PR-B_riscv64_zvbc_crc64.patch`（4 lanes × 128-bit，vlseg2e64 段加载 + vclmul_vx 常数广播，主循环零标量提取；修订后版本 commit `642d9bb`，2026-09-24）
**平台**: k3 内部验证机（Spacemit X100）· 8 核/31GB · VLEN=256
**对照**: 基线 crcspeed（slice-by-8）；依赖 PR-A 基础设施

## 性能（MB/s）
| len | baseline crcspeed | PR-A+PR-B (Zvbc) | speedup | PR-B/PR-A |
|---|---|---|---|---|
| 256 B | 1084 | 2415 | **2.23x** | 1.3x |
| 1 KiB | 1070 | 5818 | 5.44x | 2.2x |
| 4 KiB | 1129 | 8885 | 7.87x | 3.1x |
| 64 KiB | 1109 | 10376 | **9.36x** | 3.6x |

## 正确性
- 官方 crc64Test --crc + 随机/链式/边界 位一致 ✅

## 结论
Zvbc 向量 ~**9.4x**（64KiB），相对标量 Zbc 再 +3.6x；无 Zvbc 硬件回落 Zbc 标量，可独立回滚。