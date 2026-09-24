## Summary

Vectorize the depthwise-conv input padding copy with RVV.

`CPUConvolutionDepthwise::BasicFloatExecution::onResize` pads every input row with `::memcpy(dst, src, src_width * unit * bytes)`. With the fp32 CPU backend that is `src_width * 16` bytes per row, and the loop runs on every inference (it sits inside the `mExecutor` lambda that `onExecute` calls).

This adds `MNNMemcpyBytes_RVV` (e8, LMUL m8, VLEN-agnostic) and calls it when the CPU reports the V extension at runtime:

```cpp
#ifdef MNN_USE_RVV
if (core->supportRVV) {
    MNNMemcpyBytes_RVV(dst, src, src_width * unit * bytes);
} else {
    ::memcpy(dst, src, src_width * unit * bytes);
}
#else
::memcpy(dst, src, src_width * unit * bytes);
#endif // MNN_USE_RVV
```

The kernel is placed in `source/backend/cpu/riscv/rvv/MNNMemcpyBytes.cpp`, not next to the call site. `CPUConvolutionDepthwise.cpp` belongs to the `MNNCPU` object library, which `source/backend/cpu/riscv/CMakeLists.txt` gives `-DMNN_USE_RVV` but not `-march=${MNN_RVV_BASE_MARCH}`; only `MNNRVV` receives the vector ISA flag. Intrinsics therefore cannot be compiled in the generic CPU translation unit, and the call is a cross-TU declaration. This is the arrangement the other `riscv/rvv/` kernels already use: `compute/CommonOptFunction.cpp` declares each `*_RVV` kernel extern and assigns it under `if (core->supportRVV)`, and `compute/ImageProcessFunction.cpp` keeps its generic bodies under `#ifndef MNN_USE_RVV` so the RVV translation unit supplies them.

## Validation

Measured on the target board, not under emulation:

- SpacemiT K3 X100, riscv64, 8 cores, VLEN = 256 bits (`vlenb = 32`)
- GCC 15.2.0 (Bianbu 15.2.0-16ubuntu1bb5), glibc 2.43
- `-march=rv64gcv -mabi=lp64d -O3`, pinned to a single core

**Build integration.** The patch was applied to a real MNN tree and the MNN target built (`cmake .` then `make -j8 MNN`): `libMNN.so` relinks with 0 errors. `nm libMNN.so` shows the kernel is linked in, and disassembling the patched `onResize` shows both branches:

```asm
jal 1edfba <_Z18MNNMemcpyBytes_RVVPvPKvm>   ; RVV path, guarded by beqz a3
jal 22ad0  <memcpy@plt>                      ; fallback path
```

The runtime gate is live on this board: `AT_HWCAP = 0x20112d` has the V bit set, so `gCPUInfo.rvv` is true and `core->supportRVV` selects the RVV path.

The kernel cannot live in the `MNNCPU` translation unit. Compiling the intrinsics with the flags `MNNCPU` actually receives (and an explicit `-march=rv64gc`) fails, which is why the kernel is in `riscv/rvv/`:

```text
error: built-in function '__riscv_vsetvl_e8m8(size)' requires the 'v' ISA extension
```

It only appears to work with the bare `MNNCPU` flags because this board's gcc default `-march` happens to contain v.

**Correctness.** Run against the exact file this patch adds (the test `#includes` it, rather than a retyped copy): 2461 checks, 0 failures, on real silicon. Coverage: the size list {0,1,2,3,4,7,8,15,16,31,32,33,63,64,65,127,128,129,255,256,257,511,512,513,1023,1024,1025,4095,4096,4097,16383,16384,65535,65536,262144}, every size from 0 to 600, 800 randomized sizes below 20,000, and every row width the call site can produce up to `src_width = 1024` — with 512-byte sentinel guard bands on both sides of the destination to catch an underrun or an overrun on the tail chunk.

**Performance.** The gain is concentrated on short rows. Measured on the exact patched call shape, with the destination strided into the padded buffer as in `onResize`, paired ABBA and reported as a sign test over 201 samples (`z > 2` means the win is real, not drift):

| Shape         | Row bytes | libc (GB/s) | RVV (GB/s) | Speedup | Verdict        |
| ------------- | --------: | ----------: | ---------: | ------: | -------------- |
| tiny 3x3      |        48 |        2.75 |       3.12 |  1.130x | z = +13.1      |
| ResNet 7      |       112 |        4.29 |       7.30 |  1.666x | z = +13.9      |
| ResNet 14     |       224 |       10.35 |      12.65 |  1.251x | z = +13.8      |
| ResNet 28     |       448 |        5.86 |       5.80 |  1.003x | z = +1.9 (tie) |
| ResNet 56     |       896 |        6.34 |       6.38 |  1.006x | z = +4.2       |
| MobileNet 112 |      1792 |       12.15 |      12.37 |  1.018x | z = +7.7       |
| MobileNet 224 |      3584 |       11.23 |      11.27 |  1.012x | z = +3.5       |
| YOLO 640      |     10240 |        3.87 |       3.88 |  1.000x | z = +0.2 (tie) |

A size sweep over `rowBytes = src_width * 16` locates the crossover, reproducibly across runs:

| Source width | Row bytes | Speedup     | Notes                               |
| ------------ | --------- | ----------- | ----------------------------------- |
| 1–7          | 16–112    | 1.43x–2.18x | glibc takes its scalar word path    |
| 8–14         | 128–224   | ~1.2x–1.9x  | Variable between runs               |
| 18–896       | 288–14k   | 1.00x–1.05x | Both sides use equivalent RVV loops |

The reason is visible in the disassembly. On this libc, `memcpy` is an IFUNC whose runtime target starts with a size test:

```asm
8c350:  li     a3,128
8c356:  bltu   a2,a3,8c372   ; size < 128 -> 8-byte word loop
8c35a:  vsetvli a3,a2,e8,m8,ta,ma
8c35e:  vle8.v v0,(a1)
8c366:  vse8.v v0,(a0)
```

So for `rowBytes >= 128` glibc already runs the same `e8,m8` loop this patch adds, and the change comes out at ~1.0x. Below 128 bytes — `src_width < 8`, i.e. the low-resolution stages of a vision backbone, where a depthwise input row is 16-112 bytes — glibc drops to a scalar word loop while the kernel still uses RVV, and that is where the 1.4-2.2x sits.

## Scope

Touches two files: `source/backend/cpu/CPUConvolutionDepthwise.cpp` (one declaration, one dispatch) and a new `source/backend/cpu/riscv/rvv/MNNMemcpyBytes.cpp`. No existing kernel is modified or replaced, and the non-RVV build is unchanged (`#else` keeps bare `::memcpy`). The affected `::memcpy` call is the only one added to this path; upstream still uses `::memcpy` there, so there is no duplicate implementation.

## Notes

- The benefit depends on the C library. `memcpy` gained an RVV implementation in glibc only recently (the patch series landed in the 2.4x timeframe). A build linked against an older libc — 2.39, for example — still has a purely scalar `memcpy`, and there the gain applies at every row size rather than only below 128 bytes. The numbers above are the conservative case: they were taken against a libc that already vectorizes.
- No end-to-end model speedup is claimed. The padding copy is one part of one op; this patch addresses the copy in isolation. The figures above are the copy measured directly.
- The loop re-issues `vsetvli` every iteration rather than hoisting `vsetvlmax` with a separate tail. That matches the other `riscv/rvv/` kernels.
- `MNNRVV` sources are collected by `FILE(GLOB .../rvv/*.cpp)`, so the new file needs no `CMakeLists.txt` change. Note that `GLOB` is used without `CONFIGURE_DEPENDS`, so an existing build directory needs one `cmake .` re-run before the new file is picked up; a fresh build directory is unaffected.
- Dispatch is gated by `core->supportRVV`, set from `gCPUInfo.rvv`, so an RVV-enabled build running on a board without the V extension stays on `::memcpy` and cannot take an illegal-instruction fault.

## Co-authors

- ihb2032 <hebome@foxmail.com>
- lyd1992 <liuyudong@iscas.ac.cn>
- Yuansheng <yuansheng@isrc.iscas.ac.cn>


## Module

CPU (RISC-V RVV): depthwise convolution input padding copy

## Type

- [ ] Feature
- [ ] Bugfix
- [x] Perf
- [ ] Refact
- [ ] Style
- [ ] Doc
- [ ] Test
- [ ] Chore

## Checklist

- [x] Commit message follows `[Module:Type] Description` format
- [x] Code compiles without errors
- [x] Tested on relevant platform(s)
- [x] No unrelated format or style changes included