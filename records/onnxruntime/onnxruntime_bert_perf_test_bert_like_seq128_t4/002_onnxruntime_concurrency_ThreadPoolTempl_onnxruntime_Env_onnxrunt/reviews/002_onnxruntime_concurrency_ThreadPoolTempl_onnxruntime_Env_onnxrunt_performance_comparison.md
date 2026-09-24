# [MLAS] Add RVV fused activation fast path on riscv64

## Motivation

MlasActivation currently processes the element-wise float32 activation
path through the generic four-element MLAS vector abstraction. On
riscv64 this leaves the scalable RVV width unused for fused bias and
activation processing.

On SpacemiT K3 (X100, VLEN=256), the existing generic activation path
does not contain RVV instructions. Compiling the same four-element
implementation with -march=rv64gcv also does not recover the missing
performance.

Add a dedicated RVV path so supported fused activations can process up
to 32 float32 elements per iteration with e32m4 on VLEN=256 hardware,
while retaining the existing generic implementation as the fallback.

## Changes

* Add an internal MLAS activation override for riscv64 RVV.
* Register the override only after the existing runtime V-extension
  check succeeds, preserving Linux HWCAP detection and
  ORT_MLAS_RISCV_FORCE_SCALAR behavior.
* Vectorize Identity with bias, ReLU, LeakyReLU, Clip, HardSigmoid, and
  HardSwish using dynamic RVV vector lengths.
* Fuse bias addition and activation between a single load/store pair.
* Keep Identity without bias as a no-op.
* Preserve NaN values and signed zero for clamp-style activations by
  using compare-and-select operations instead of RVV min/max
  instructions.
* Preserve the generic LeakyReLU distinction between the four-element
  vector body and scalar tail for signed-zero behavior.
* Fall back to the generic HardSigmoid path for non-finite alpha or beta
  parameters.
* Keep Tanh and Logistic on their existing MLAS paths.
* Use the generic implementation for rows shorter than 32 elements.
  This is a conservative crossover selected from measurements on K3,
  rather than an architectural requirement.
* Add fused-activation tests covering public dispatch, scalar fallback,
  matrix layout, vector tails, guard-page boundaries, floating-point
  edge cases, and unsupported activation fallback.

No public API is changed.

## Performance

Measured on a SpacemiT K3 / X100 with VLEN=256, one pinned CPU
(CPU0), GCC 14.3.0, C++20, Release/O3, and seven alternating runs per
case. Reported values are medians.

For Bias+ReLU with M=64, N=3136, ldc=3139:

```
                     latency
```

generic rv64gc        303.17 us
generic rv64gcv       563.54 us
RVV fast path          99.07 us

The RVV implementation is 3.06x faster than the normal generic
baseline and 5.69x faster than the same four-element generic path
compiled with rv64gcv.

Single-thread MLAS Conv+ReLU measurements, with convolution and GEMM
settings unchanged and only the activation implementation switched:

C/F/H/K        generic       RVV        speedup
3/16/56/3      422.62 us    346.00 us    1.221x
16/32/56/3    2915.10 us   2570.88 us    1.134x
64/64/28/1     533.22 us    450.11 us    1.185x
1/16/112/1     432.32 us    180.32 us    2.397x

All Conv+ReLU comparisons had a maximum absolute output error of zero.

For M=1 and N<32, retaining the generic path limits the measured
difference from the additional dispatch check to approximately
-2.18 ns to +1.36 ns on K3.

## Testing

Tested on SpacemiT K3 with GCC 14.3.0 using the repository MLAS CMake
configuration:

* RVV enabled: 9 fused-activation tests passed.
* ORT_MLAS_RISCV_FORCE_SCALAR=1: 6 tests passed and 3 RVV-specific
  tests were skipped as expected.
* Separate onnxruntime_USE_RVV=OFF build: 5 tests passed.
* macOS ARM64 activation build: 4 tests passed.
* git diff --check passed.
* New test sources pass the repository clang-format 20.1.8 formatting.

The tests cover M=0/1/3, key N values from 0 through 129, with and
without bias, contiguous and padded rows, unaligned inputs, protected
tail pages, random inputs, NaN, +/-Inf, +/-0, denormals, NaN payloads,
Identity bit patterns, non-finite parameters, and fallback behavior.

A physical riscv64 system without the V extension and systems with
VLEN=128 or VLEN=512 were not available for validation. The
ORT_MLAS_RISCV_FORCE_SCALAR result therefore validates the runtime
fallback path but is not presented as testing on non-RVV hardware.

Full ONNX Runtime model-level validation was not performed; the
integration performance measurements above exercise the MLAS
Conv+ReLU path directly.