# PR-A Zbc 标量 CRC64 · 性能数据

**补丁**: `PR-A_riscv64_zbc_crc64.patch`（Zbc clmul/clmulh 折叠 + slice-by-8 尾，运行时 zbc 探测）
**平台**: k3 内部验证机（Spacemit X100）· 8 核/31GB · VLEN=256
**对照**: 基线 crcspeed（slice-by-8）

## 性能（MB/s）
| len | baseline crcspeed | PR-A Zbc | speedup |
|---|---|---|---|
| 256 B | 1084 | ~1800 | **~1.7x** |
| 1 KiB | 1070 | ~2700 | ~2.5x |
| 4 KiB | 1129 | ~2850 | ~2.5x |
| 64 KiB | 1109 | ~2900 | **~2.6x** |

## 正确性
- 官方 crc64Test + 随机/链式/边界 vs slice-by-8 位一致 ✅

## 结论
标量 Zbc CRC64 ~**2.6x**（64KiB）；PR-A 单独合入即完整可用，PR-B 为独立增强。