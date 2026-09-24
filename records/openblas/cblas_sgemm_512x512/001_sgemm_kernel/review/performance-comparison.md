# 001_sgemm_kernel: baseline vs patched

- Module: SGEMM 16x8 ZVL256B micro-kernel
- Official benchmark: `benchmark/cblas_sgemm.goto`
- Baseline commit: `9e857d0055984e6839fc0aba94b5943ae12ddb5d`
- Patch SHA-256: `98828caa45b85ef42c59e1a964ac62f18817a1ffc5199f6e3da5952cf499a946`
- Apply: PASS
- Build: PASS
- Functional test: PASS; official `make tests` returned 0, with 125/125 utests and 1473/1473 extension tests passing, and no new CBLAS failure.
- Path coverage: target path selected by the matching official benchmark.
- Method: fixed X100 CPU7 at observed 2.2 GHz, single thread, warm-up once and measure three times per size; compare medians.
- Performance gate: 512 median gain >=2%, all three patched 512 runs above the baseline 512 median, and no measured size <=-3%.
- Obvious regression: NO (worst change +3.43%).

## Performance comparison

| Size | Baseline runs (MFlops) | Patched runs (MFlops) | Baseline median | Patched median | Change |
|---:|---|---|---:|---:|---:|
| 256 | 22348.96, 22926.65, 23653.20 | 24058.81, 24302.02, 24102.74 | 22926.65 | 24102.74 | +5.13% |
| 512 | 24957.03, 25201.08, 24892.52 | 26389.07, 26344.61, 26320.82 | 24957.03 | 26344.61 | +5.56% |
| 1024 | 24306.48, 24331.77, 24252.92 | 25139.09, 25205.71, 24974.79 | 24306.48 | 25139.09 | +3.43% |

## Result

`PASS`. All correctness and performance gates passed.
