# Go RISC-V Zbb TrailingZeros v2 validation

## Disposition

**`PASS_SUBMISSION_READY`**

The narrowed v2 patch is ready to place in a review PR together with this test
package. No commit, push, PR, Gerrit upload, or upstream contact is included in
this offline handoff.

## Source and patch

- Go commit: `d6eb7bd6aee56dd33e0542b56a507b028692db20`
- Patch SHA-256: `686b1f217394833122496b6c1feaf2e59f5d7dfb510a629a66c165de0fb2863e`
- Clean apply check: pass
- `gofmt` and `git diff --check`: pass
- Changed files: compiler intrinsics, intrinsic-table test, and math/bits asmcheck

## Correctness and portability Gate

- Clean latest-master `all.bash`: pass (`ALL TESTS PASSED`)
- Latest-master post-checks: 13/13 zero statuses
- Baseline and candidate targeted tests: pass
- Forced `GODEBUG=cpu.zbb=off`: pass
- RISC-V codegen asmcheck: pass
- amd64/arm64 cross-builds: 4/4 pass
- rva20u64 and rva22u64 probes: pass

## Target performance

Effect is `100 × (candidate / baseline - 1)`; negative is improvement. The
baseline and candidate were derived from the same commit, built with the same
bootstrap/toolchain settings, pinned to CPU 2, and measured for 15 matched
rounds with alternating execution order.

| Benchmark | Median | 95% bootstrap CI | Improved pairs |
|---|---:|---:|---:|
| `TrailingZeros` | -46.136% | [-46.294%, -45.940%] | 15/15 |
| `TrailingZeros8` | -8.405% | [-8.697%, -8.284%] | 15/15 |
| `TrailingZeros16` | -51.624% | [-51.694%, -51.554%] | 15/15 |
| `TrailingZeros32` | -48.713% | [-49.646%, -48.279%] | 15/15 |
| `TrailingZeros64` | -44.645% | [-44.811%, -44.370%] | 15/15 |

All 75 target pairs improved.

## v1 regression guard

The broader v1 patch aliased rva20 `internal/runtime/sys.TrailingZeros*` calls
to a runtime-dispatched intrinsic and reproducibly regressed 13 focused map and
allocator controls. v2 keeps those aliases generic for rva20 and repeats the
same 13 controls:

- 7 classified improvements
- 6 inconclusive
- **0 classified regressions**

These controls establish that the v1 blocker is absent. Their apparent gains
are not claimed as causal patch benefits because the affected runtime paths are
intentionally left generic.

## Machine code

- rva20u64: feature-flag branch plus `CTZ`/`CTZW` fast path and pure-Go call.
- rva22u64: direct `CTZ`/`CTZW`, with no runtime feature branch.
- Forced feature-off execution passes.
- rva20 runtime aliases remain generic; rva22 aliases remain intrinsic.

## Evidence integrity

- Paired-validation evidence: 675 files, remote/local SHA-256 match
- Latest-master evidence: 41 files, remote/local SHA-256 match
- Paired statuses: 225/225 zero
- Latest-master statuses: 13/13 zero

The structured details are in `final-gate.json` and
`performance-summary.json`. Absolute time/op evidence is retained in
`performance-samples.csv`; the target benchmark table is
`benchstat-targets.md`.
