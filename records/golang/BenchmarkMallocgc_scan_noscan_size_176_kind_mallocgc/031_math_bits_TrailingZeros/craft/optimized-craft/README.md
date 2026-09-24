# Go RISC-V Zbb TrailingZeros runtime dispatch

`patch.diff` enables runtime Zbb dispatch for public
`math/bits.TrailingZeros{8,16,32,64}` calls in `rva20u64` binaries.

- Zbb-capable systems execute `CTZ`/`CTZW`.
- Systems without Zbb, and `GODEBUG=cpu.zbb=off`, retain the pure-Go fallback.
- `rva22u64` and newer keep unconditional CTZ lowering because Zbb is mandatory.
- `internal/runtime/sys` keeps its generic rva20 implementation, avoiding the
  runtime hot-path regressions reproduced with the broader v1 patch.

The patch is based on Go commit
`d6eb7bd6aee56dd33e0542b56a507b028692db20` and changes:

1. `src/cmd/compile/internal/ssagen/intrinsics.go`
2. `src/cmd/compile/internal/ssagen/intrinsics_test.go`
3. `test/codegen/mathbits.go`

See `../test-results/validation-report.md` and `final-gate.json` for the
submission Gate and reproducible evidence summary.
