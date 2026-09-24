# Target benchmark table

This benchstat-style table exposes the absolute time/op values that underpin the percentage column.
Each value is the median of 15 paired samples from CPU 2; all individual observations are in `performance-samples.csv`.

| name | baseline time/op | candidate time/op | delta |
|---|---:|---:|---:|
| `BenchmarkTrailingZeros` | 3.882 ns/op | 2.094 ns/op | -46.136% |
| `BenchmarkTrailingZeros16` | 5.018 ns/op | 2.428 ns/op | -51.624% |
| `BenchmarkTrailingZeros32` | 4.393 ns/op | 2.256 ns/op | -48.713% |
| `BenchmarkTrailingZeros64` | 3.882 ns/op | 2.149 ns/op | -44.645% |
| `BenchmarkTrailingZeros8` | 2.679 ns/op | 2.453 ns/op | -8.405% |

The table deliberately reports `ns/op` (the unit emitted by Go benchmark binaries); it is an absolute time/op unit equivalent to `sec/op`.
