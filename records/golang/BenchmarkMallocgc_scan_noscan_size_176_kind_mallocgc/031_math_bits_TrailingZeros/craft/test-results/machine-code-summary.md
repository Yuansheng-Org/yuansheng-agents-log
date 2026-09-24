# Machine-code and fallback summary

## rva20u64

Each public `math/bits.TrailingZeros*` probe contains:

1. a load of the existing `runtime.riscv64HasZbb` flag;
2. a likely branch to `CTZ` or `CTZW` when Zbb is available;
3. a call to the original pure-Go implementation otherwise.

Combined probe symbol size is 252 bytes:

- `TrailingZeros64`: 58 bytes
- `TrailingZeros32`: 60 bytes
- `TrailingZeros16`: 68 bytes
- `TrailingZeros8`: 66 bytes

The probe and `math/bits` test pass with `GODEBUG=cpu.zbb=off`, proving the
fallback is executable rather than merely present in disassembly.

## rva22u64

Zbb is mandatory, so lowering remains direct:

- `TrailingZeros64`: `CTZ`, `RET`
- `TrailingZeros32`: `CTZW`, `RET`
- `TrailingZeros16`: OR with bit 16, `CTZW`, `RET`
- `TrailingZeros8`: OR with bit 8, `CTZW`, `RET`

Combined probe symbol size is 34 bytes and contains no runtime feature branch.

## Runtime hot-path guard

At rva20, `internal/runtime/sys.TrailingZeros{8,32,64}` is intentionally not
aliased to the runtime-dispatched public intrinsic. At rva22+, the aliases keep
direct intrinsic lowering. Intrinsic-table tests cover this boundary.
