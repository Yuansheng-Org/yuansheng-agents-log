# PR-A Zbc 标量 CRC32C · 性能数据

**补丁**: `PR-A_riscv64_zbc_crc32c.patch`（RISC-V Zbc carry-less multiply 加速 CRC-32C，4-way clmul 折叠 + Barrett）
**平台**: k3 内部验证机（Spacemit X100）· 8 核/31GB · VLEN=256 · gcc 14.3.0
**对照**: 官方 `crc32c_slow`（slicing-by-4）

## 性能（MB/s）
| len | slow | PR-A Zbc | speedup |
|---|---|---|---|
| 128 B | 289 | 1118 | **3.87x** |
| 256 B | 296 | 1082 | 3.65x |
| 1 KiB | 297 | 1123 | 3.79x |
| 4 KiB | 297 | 1073 | 3.62x |
| 64 KiB | 295 | 1069 | **3.63x** |

## 正确性
- 官方 `ctest -R crc32`（unittest/mysys/crc32-t.c）**36/36 全过**（crc32 17 + crc32c 19）
- RFC 3720 `crc32c("123456789")==0xE3069283` ✅
- 0..4096 全长度 + 500 随机 + 100 链式（vs slow）位一致 ✅

## 结论
标量 Zbc 路径 ~**3.6–3.9x**；无 Zbc CPU/非 riscv64 自动回落软件，零行为变化。