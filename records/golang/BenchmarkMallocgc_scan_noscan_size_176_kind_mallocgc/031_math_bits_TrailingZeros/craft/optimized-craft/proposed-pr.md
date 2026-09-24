# Proposed change

Use runtime Zbb detection for public `math/bits.TrailingZeros*` calls in
generic `rva20u64` binaries, while preserving the pure-Go fallback and keeping
the Go runtime's own rva20 hot paths on their generic implementation.

## Validation scope

- Paired, alternating-order `math/bits` benchmarks on a Zbb-capable riscv64 host.
- Focused map and allocator controls used to reject the broader v1 patch.
- Forced `GODEBUG=cpu.zbb=off` fallback tests.
- rva20/rva22 disassembly and intrinsic-table tests.
- amd64/arm64 cross-builds and a clean latest-master full regression.

No external submission is included in this offline package.
