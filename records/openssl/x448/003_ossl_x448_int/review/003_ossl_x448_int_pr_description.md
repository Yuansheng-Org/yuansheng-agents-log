# [RISC-V] Curve448: vectorize field helpers with RVV (`e64/m4`)

## Summary

Curve448 field arithmetic dominates `x448` and Ed448. On RV64 each field element is
`NLIMBS == 8` 64-bit limbs, which maps exactly onto one RISC-V vector register group
at `e64/m4`. This patch adds RVV fast paths for the three helpers called from the
innermost loops — `gf_sub_RAW()`, `gf_weak_reduce()` and `gf_cond_swap()` — and keeps
the scalar loop as the fallback.

## Motivation

The compiler currently lowers these fixed-trip loops to a narrow, unvectorized shape
(`vsetivli zero,2,e64,m1` + `vrgather`). Since `NLIMBS == 8` already matches the
`e64/m4` register group on any legal RVV implementation (the ISA requires
`VLEN >= 128`, so `VLMAX == 8` for `e64/m4`), one `vsetvl` can cover the entire limb
array. The helpers sit in the hot path of `ossl_x448_int()` and the Edwards point
arithmetic, so this is where the cycles are.

## Implementation

One file (`crypto/ec/curve448/field.h`), three guarded fast paths:

- `gf_sub_RAW()` — load both operands with `vle64.v`, subtract with `vsub.vv`, add
  the `co1` bias with `vadd.vx`; the `co2 == co1 - 2` tweak is applied to
  `limb[NLIMBS/2]` in scalar code, then `gf_weak_reduce()` runs as before.
- `gf_weak_reduce()` — derive the carry-out vector with `vsrl.vx` (shift by 56) and
  fold it back one lane up with `vslide1up.vx`, which injects `tmp` into lane 0
  exactly as the scalar loop does; mask and add with `vand.vx` / `vadd.vv`.
- `gf_cond_swap()` — branch-free xor / mask / xor using `vxor.vv` and `vand.vx`.

Design notes:

- **`e64/m4`, not `e64/m2`.** The earlier `e64m2` formulation could never reach the
  vector body on VLEN=128, where `VLMAX` is only 4 < `NLIMBS == 8`, so the
  `vl == NLIMBS` guard always failed and execution fell back to scalar. `e64/m4`
  makes the path reachable on both VLEN=128 and VLEN=256.
- **Guarded by `vl == NLIMBS`.** If a shorter vector were ever returned, the original
  scalar loop still runs; nothing assumes a particular VLEN beyond the ISA minimum.
- **Constant-time property preserved.** `gf_cond_swap()` remains branch-free: no
  control flow and no memory address depends on the swap mask.
- **Gated on `__riscv_vector`, not `__riscv_v`.** The fast paths are compiled in only
  when the compiler defines `__riscv_vector` (the RVV intrinsic interface macro), so
  a non-vector or older toolchain keeps the scalar code verbatim rather than silently
  falling through an undefined macro. No `-march` change is required: the intrinsics
  interface is what the code is written against.

## Performance

SG2044, riscv64, VLEN=128. Baseline (`-nopatch`) vs. this patch, same YuanshengCI
runner (`container-v4`), image digest
`sha256:4d92def334be5ad02440683a7075904a80327fd99b0f3dc52c303c6a5bfcdf1c`,
kernel `Linux 6.12.32-0.0.0.0.riscv64`, `x448` case `op:ecdh,algo:x448,iter:3000`,
`perf record` at `cpu-clock` 99 Hz, 0 lost samples.

| Case | Baseline | Patched | Duration | Throughput |
|---|---:|---:|---:|---:|
| `x448` | 3.624422 s | 3.378005 s | **-6.799%** | 827.718 -> 888.098 ops/s (**+7.295%**) |
| `ed448-sign` | 2.690212 s | 2.620834 s | **-2.579%** | 1858.590 -> 1907.790 ops/s (**+2.647%**) |
| `ed448-verify` | 1.453125 s | 1.355136 s | **-6.743%** | 1376.344 -> 1475.867 ops/s (**+7.231%**) |
| `x25519` | 1.260010 s | 1.249644 s | -0.823% | 3968.221 -> 4001.139 ops/s (+0.830%) |

All three workloads that use the Curve448 field helpers move in the same direction,
while `x25519`, which does not reach them, is essentially flat — the target
correlation supports the helpers as the cause rather than general machine drift.

Target hotspot (`perf report --no-children`): `ossl_x448_int` self overhead falls
from **13.30% to 5.37%** (estimated self time 0.482 s -> 0.181 s), consistent with
the patch also being inlined partly into its callers.

Run-to-run control: across the 51 non-target cases the paired duration change has
median **+0.218%** (P10 -1.511%, P90 +3.923%, MAD 0.603%). `x448` and
`ed448-verify` sit clearly outside that band; `ed448-sign` is smaller but same-signed
and larger than the median.

**Path-reachability evidence.** `perf annotate` on the patched `ossl_x448_int` shows
`vsetivli ...,8,e64,m4` together with non-zero-sampled `vsub.vv`, `vadd.vx`,
`vsrl.vx`, `vslide1up.vx`, `vxor.vv` and `vand.vx`, i.e. the `e64/m4` path executes
in the hot path rather than existing only in an unreachable branch.

## Correctness

- Functional suite: 398 tests, **375 PASS / 23 SKIP / 0 FAIL** — identical for
  baseline and patched. Both functional CSVs share SHA256
  `1E7F9DD4EC62907E5E6FAE56B35679F8CE62D860FBB1DED00454BAA3DD10243C` and compare
  line-for-line with zero differences.
- Benchmarks: both arms 55 `PASSED` / 3 `FAILED`; the three shared failures
  (`aes-128-xts`, `aes-256-xts`, `cmac-aes-128-cbc`) are pre-existing.
- PMU `cpu_cycle` / `instruction` and some cache counters show known platform
  counting/scaling anomalies on this board; absolute values are not used as
  evidence. Conclusions rest on `duration_time`, `task_clock`, throughput, the
  non-target paired distribution, target hotspot share, RVV instruction sampling,
  and functional parity.

## Upstream status

The same hotspot is already addressed upstream by **PR #32847**
("riscv: vectorize Curve448 field helpers with RVV", open; head branch
`x448-field-e64m4-riscv64-rvv`). This record is the perf-verified artifact for the
**`e64m4` variant on SG2044 (VLEN=128)**; the upstream PR has since evolved beyond
that single-width form, and the differences matter:

| | this record | upstream PR #32847 (current head) |
|---|---|---|
| width | fixed `e64/m4` | `vsetvli` probe: `e64/m2` when `VLMAX(e64,m2) >= 8` (VLEN >= 256), else `e64/m4` |
| gate | runtime `vl == NLIMBS` inside each helper | compile-time `__riscv_v_min_vlen <= 128` |
| emission | RVV intrinsics inline in `field.h` | out-of-line perlasm helpers `ossl_gf_sub_RAW_rvv` / `ossl_gf_weak_reduce_rvv` / `ossl_gf_cond_swap_rvv` in `crypto/ec/asm/curve448-riscv64.pl` |
| files | `crypto/ec/curve448/field.h` only | `crypto/ec/asm/curve448-riscv64.pl`, `crypto/ec/build.info`, `crypto/ec/curve448/field.h`, `crypto/perlasm/riscv.pm` |

The upstream PR body carries the same SG2044 table reproduced above, but the
implementation now selects the register width per build because the fixed `e64m4`
form does **not** win everywhere. Upstream documents that at VLEN = 256 the vector
path costs 7.5% of `x448` cycles (13% when forced to `e64/m4`), which is exactly why
the probe exists — at VLEN >= 256 the whole field element already fits `e64/m2`,
which is also the width GCC picks when auto-vectorizing the scalar loops.

Two consequences for this record:

1. The fixed-`e64m4` form here should be read as the VLEN=128 configuration, not as
   a VLEN-independent win. The measurement below is SG2044 (VLEN=128) and stands on
   its own for that board.
2. The `e64m2` limitation described above ("could not reach the vector body on
   VLEN=128") and the VLEN=256 regression are two halves of the same fact: `m2` is
   right at VLEN >= 256, `m4` is right at VLEN == 128.

Please review the upstream PR for the submission-form discussion; this record is
kept as the perf-verified `e64m4` artifact.

## Risk

Low.

- Each fast path is guarded by `vl == NLIMBS`, with the scalar loop retained as
  fallback, so a narrower vector cannot produce wrong results.
- `gf_cond_swap()` stays branch-free and address-independent with respect to the
  swap mask.
- The change is confined to one header; no API or ABI change.

## Follow-up / not verified here

- Results come from one aggregate run per arm. At least 5 interleaved repetitions
  are recommended to confirm the ~7% gain is stable.
- `gf_cond_swap()` is constant-time sensitive; generated code should get a
  dedicated side-channel / constant-time review.
- Retest on a VLEN=256 part to confirm `e64/m4` still wins and does not add register
  pressure.

## Attribution

Patch derived from the Yuansheng trace/craft pipeline
(`craft/openssl/003_ossl_x448_int`, blueprint `bp-openssl-x448-003`); correctness and
performance verification on real hardware performed independently. Author/sign-off to
be filled in by the submitter.

---

### Diff

```text
diff --git a/crypto/ec/curve448/field.h b/crypto/ec/curve448/field.h
index 850244ee55..9d1a4b2d49 100644
--- a/crypto/ec/curve448/field.h
+++ b/crypto/ec/curve448/field.h
@@ -20,4 +20,8 @@
+#if defined(__riscv_vector)
+# include <riscv_vector.h>
+#endif
@@ -143,6 +147,21 @@ void gf_sub_RAW(gf out, const gf a, const gf b)
@@ -160,6 +179,29 @@ void gf_weak_reduce(gf a)
@@ -252,6 +294,21 @@ static ossl_inline void gf_cond_swap(gf x, gf_s *RESTRICT y, mask_t swap)
```

Blob hashes as recorded in the patch header: `850244ee55` (pre-image) → `9d1a4b2d49`
(post-image). These are the hashes of the worktree the patch was generated in; against
upstream `master` at the time of writing `crypto/ec/curve448/field.h` hashes to
`9c3b8a3f166d2f6ca707007fe3e66f619ceee8a1`. The patch applies cleanly to upstream
`master` regardless — the header hashes are informational, not a base requirement.
