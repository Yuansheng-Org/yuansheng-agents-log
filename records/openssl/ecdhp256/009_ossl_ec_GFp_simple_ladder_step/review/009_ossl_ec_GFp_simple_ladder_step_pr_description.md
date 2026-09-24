# [RISC-V] Configurations: enable `ec_nistp_64_gcc_128` for `linux64-riscv64`

## Summary

`ec_nistp_64_gcc_128` is globally default-disabled in `Configure`, so on a default
`linux64-riscv64` build the specialised constant-time NIST curve implementations are
never compiled and NIST P-256/P-384/P-521 fall through to the generic Montgomery
ladder. This patch re-enables the feature for the `linux64-riscv64` target **only**,
using Configure's per-target `enable => [ ... ]` mechanism. It is a one-line,
build-configuration-only change: no source change, no ABI change, no new code.

## Motivation

`Configure` registers the feature as default-disabled:

```perl
"ec_nistp_64_gcc_128" => "default",      # Configure:639
```

and `crypto/ec/build.info` gates the specialised implementations on it:

```perl
IF[{- !$disabled{'ec_nistp_64_gcc_128'} -}]     # build.info:92
  $COMMON=$COMMON ecp_nistp224.c ecp_nistp256.c ecp_nistp384.c ecp_nistp521.c ecp_nistputil.c
ENDIF
```

(A second gate at `build.info:48` covers only the ppc64 assembly variants
`ecp_nistp384-ppc64.s` / `ecp_nistp521-ppc64.s`; the C implementations above are
the ones relevant to riscv64.)

So on a default `linux64-riscv64` build `ecp_nistp*.c` is not compiled and
P-256/P-384/P-521 are served by `EC_GFp_mont_method` (`ossl_ec_GFp_simple_ladder_step`).
The specialised implementations use `__uint128_t` field arithmetic and are
substantially faster.

The default-disable is deliberate and correct for its original audience: MSVC
pre-VS2019 and 32-bit targets do not define `INT128_MAX`, and `ecp_nistp256.c` uses
it as the compile-time safety gate:

```c
#ifndef INT128_MAX                         /* ecp_nistp256.c:49 */
#error "Your compiler doesn't appear to support 128-bit integer types"
#endif
```

riscv64 (LP64D) is **not** in that audience: GCC and Clang define both `__uint128_t`
and `INT128_MAX` on every 64-bit target, and on the RISC-V toolchains in practice
(GCC 13+) this has held for years. The feature is therefore unconditionally safe on
`linux64-riscv64` — but because the default-disable is global, the target never
re-enables it.

## Implementation

One line inside the existing `linux64-riscv64` entry in
`Configurations/10-main.conf`:

```perl
"linux64-riscv64" => {
    inherit_from     => [ "linux-generic64"],
    perlasm_scheme   => "linux64",
    asm_arch         => 'riscv64',
    enable           => [ 'ec_nistp_64_gcc_128' ],   # +1
},
```

This follows the established upstream pattern for re-enabling a default-disabled
feature on a single target: `Configurations/50-nonstop.conf:23` uses
`enable => ['egd']` for exactly this purpose.

MSVC and 32-bit targets still see the feature as default-disabled and continue to
skip `ecp_nistp*.c` at build time, so the "MSVC / 32-bit safe default" policy is
left intact.

### Mechanism verification (end to end, on riscv64)

- `Configure:1501` iterates `$target{enable}` and deletes the entry from `%disabled`
  only when its current reason is exactly the string `"default"` (the test is on the
  following line). `ec_nistp_64_gcc_128` is registered with precisely that string
  (`Configure:639`), so the match holds.
- With this patch applied and the stock configure line
  (`./Configure linux64-riscv64 --strict-warnings enable-fips`, **no** `enable-*`
  argument), the generated Makefile's `OPTIONS` no longer contains
  `no-ec_nistp_64_gcc_128`, and `ecp_nistp256.c` enters the `SOURCE`/`OBJ` lists.
- **Library byte-equivalence:** the resulting `libcrypto` is byte-identical
  (md5 `034c2ec8b35d0bc5c56d4fa640a17bee`) to one built with an explicit
  `enable-ec_nistp_64_gcc_128` on the command line, confirming the target-embedded
  mechanism is equivalent to the command-line override.
- **User opt-out still works:** adding `disable-ec_nistp_64_gcc_128` to the configure
  line re-disables the feature (`OPTIONS` regains `no-ec_nistp_64_gcc_128` and
  `ecp_nistp256.o` leaves the build). Command-line processing
  (`Configure:775-1246`) runs before the `$target{enable}` loop, so the disable
  reason becomes `"user"` rather than `"default"` and the delete does not fire.

## Performance

Real-board benchmark on SpacemiT X100 (K3, 8 cores, RVV 1.0, VLEN=256, GCC 15.2.0,
Bianbu 4.0.6), taskset-pinned single core, ABBA-paired with library swap, 4-second
sampling, 6 samples per arm:

| `ecdhp256` (ops/s) | Baseline | Patched |
|---|---:|---:|
| all 6 samples | 1084.5, 1084.2, 1081.5, 1084.0, 1083.8, 1083.8 | 3672.9, 3648.6, 3678.8, 3640.8, 3639.5, 3688.0 |
| median | 1084.0 | **3660.7** |

**Delta +237.7%; the two distributions do not overlap.** The gain comes from
switching P-256 from the generic `EC_GFp_mont_method` to
`EC_GFp_nistp256_method`, i.e. `__uint128_t`-based constant-time field arithmetic.
This is ~3.4x for the most common ECDH curve in TLS 1.3 (secp256r1).

The measurement was taken on a `libcrypto` built by applying **this patch alone** and
running the stock configure line above (no `enable-*` argument), so it measures the
target-embedded `enable => [...]` mechanism this PR introduces, not a command-line
override.

Other algorithms are unaffected by this change (the feature only selects EC field
implementations): `rsa2048` sign, `x25519` and `ed25519` show no systematic movement
outside run-to-run noise.

SG2042 (VLEN=128) data is not yet available.

## Why not `-march`

An earlier revision of this change also added:

```perl
cflags => add("-march=rv64gcv_zba_zbb_zbc_zbs"),
```

That line has been **dropped**, for two independent reasons:

1. **It contradicts an explicit upstream policy.** `10-main.conf:670` documents that
   `-march` is intentionally absent from target descriptions:

   > Note that -march is not among compiler options in linux-armv4 target
   > description. Not specifying one is intentional to give you choice to:
   > a) rely on your compiler default by not specifying one; b) specify your target
   > platform explicitly for optimal performance ...; c) build "universal" binary ...

   No target in `10-main.conf` sets `-march` in `cflags`. Pinning one would break (a)
   and (c), and would make `linux64-riscv64` the only target in the file that forces
   an ISA level.

2. **It has no effect on the feature this PR enables.** The upstream RISC-V vector
   code paths (`chacha_riscv.c`, `sha_riscv.c`, `sm3_riscv.c`, `md5_riscv.c`)
   dispatch through the runtime capability macros `RISCV_HAS_*()`, which read
   `OPENSSL_riscvcap_P` populated by `riscvcap.c` at run time. Those perlasm
   implementations are already assembled into a default `linux64-riscv64` build —
   verifiable with `nm`:

   ```
   $ nm libcrypto.so | grep ChaCha20_ctr32_v
   000000000012ee10 t ChaCha20_ctr32_v_zbb_zvkb
   000000000012f348 t ChaCha20_ctr32_v_zbb
   ```

   `ec_nistp_64_gcc_128` itself depends only on `__uint128_t` / `INT128_MAX`, which
   plain `-march=rv64gc` already provides (`__SIZEOF_INT128__ == 16`).

The superseded `-march` variant is retained in the record for traceability only and
is **not** part of this PR.

## User-visible effect

For users building with `./Configure linux64-riscv64`:

- NIST P-256/P-384/P-521 are served by `EC_GFp_nistp*_method` (constant-time,
  `__uint128_t`-based) instead of the generic Montgomery ladder.
- All other targets, including MSVC and 32-bit, are unchanged.

Optional opt-out without code changes:

```
$ ./Configure linux64-riscv64 disable-ec_nistp_64_gcc_128
```

## Risk

Low.

- Only changes *which source files are compiled* for `linux64-riscv64`; no source
  change, no ABI change.
- `INT128_MAX` remains the compile-time safety gate. On a hypothetical riscv64
  toolchain that lacks it, the build fails at the existing `#error` rather than
  producing broken code.
- FIPS: `ec_nistp_64_gcc_128` is FIPS-compliant when enabled; this PR only re-enables
  it on a target where it is unconditionally safe.

## Upstream overlap

No existing open PR touches `linux64-riscv64` in `Configurations/10-main.conf`, nor is
there an open PR re-enabling `ec_nistp_64_gcc_128` for a 64-bit GCC target. PRs
#32607, #31715, #31182, #30787, #32575, #32583, #31082, #32835, #32847 (all riscv64
vector or assembly work) are orthogonal and do not modify Configure targets.

## Attribution

Change derived from the Yuansheng trace/craft pipeline
(`craft/openssl/009_ossl_ec_GFp_simple_ladder_step`,
blueprint `bp-openssl-ecdhp256-009`); the original craft edit removed the global
default-disable in `Configure`, which was redesigned here as a target-scoped
`enable` to avoid affecting MSVC / 32-bit. Build-mechanism and real-board
verification performed independently. Author/sign-off to be filled in by the
submitter.

---

### Diff

```diff
diff --git a/Configurations/10-main.conf b/Configurations/10-main.conf
--- a/Configurations/10-main.conf
+++ b/Configurations/10-main.conf
@@ -756,6 +756,7 @@
         inherit_from     => [ "linux-generic64"],
         perlasm_scheme   => "linux64",
         asm_arch         => 'riscv64',
+        enable           => [ 'ec_nistp_64_gcc_128' ],
     },
 
     "linux32-riscv32" => {
```

Base: OpenSSL master `0dee42e672dbcc5a6bd277786e04bbd45f2de386` (2026-09-21);
`git apply --check` passes.

### Superseded variant (traceability only, not submitted)

```text
Configurations/10-main.conf  sha256 7c68f8247d6c0c268259d1eb68135d342077b2fb023699bb42539dcad1d1103b
  (adds the -march cflags line; dropped -- see "Why not -march")
Submitted form                sha256 237dac9794d00e41c55e384be1f976893148254a5df565d96574ceff5570b585
```
