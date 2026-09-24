# Reproduction notes

## Environment

- Layer: remote RISC-V host
- Architecture: `riscv64`
- Kernel: `6.12.32-0.0.0.0.riscv64`
- Go commit: `d6eb7bd6aee56dd33e0542b56a507b028692db20`
- Bootstrap: Go 1.26.7 for linux/riscv64
- Build profile: `GORISCV64=rva20u64`
- Benchmark CPU: 2

## Full regression

Apply `../optimized-craft/patch.diff` to the recorded commit, clear inherited
proxy and `GOMAXPROCS` variables, then run:

```sh
cd src
GOROOT_BOOTSTRAP=/path/to/go1.26.7 ./all.bash
```

After `all.bash`, run the forced fallback test, RISC-V math/bits asmcheck,
amd64/arm64 cross-builds, and rva20/rva22 probes. Exact return codes are in
`latest-master-status.tsv`.

## Paired performance

Build baseline and candidate from separate copies of the same Git commit. The
candidate differs only by the patch. Generate `math/bits` and `runtime` test
binaries, pin each invocation to CPU 2, and run 15 rounds. Odd rounds execute
baseline first; even rounds execute candidate first. Target benchtime is 750ms
and focused controls use 1s.

For each matched pair:

```text
effect_pct = 100 * (candidate_ns_per_op / baseline_ns_per_op - 1)
```

Classify a condition as improvement/regression only when its deterministic
20,000-sample bootstrap median interval lies wholly below/above zero.
`performance-summary.json` contains both the normalized effects and absolute
per-condition `ns/op` summaries. The 270 individual matched observations are
in `performance-samples.csv`; `benchstat-targets.md` presents the target
benchmark medians with baseline and candidate time/op columns.
