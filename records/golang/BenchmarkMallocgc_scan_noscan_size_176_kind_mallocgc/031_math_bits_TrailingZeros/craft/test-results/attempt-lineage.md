# Attempt lineage

Only corrected canonical attempts contribute to the final Gate.

| Attempt | Classification | Use in final Gate |
|---|---|---|
| v1 phase1-r1 | Original target and broad context measurement | Historical evidence |
| focused-r6 | Wrong pointer-regex selection | Excluded |
| focused-r7 | Correct reproduction of all 13 v1 regressions | Design input for v2 |
| optimized-v2-r9 | Used old phase1 baseline binaries | Excluded from final performance claim |
| latest-master-r8 | Inherited `GOMAXPROCS=32`, invalidating cgroup expectations | Harness failure; retained |
| latest-master-r8b | Cleared proxy and `GOMAXPROCS` environment | Canonical full regression |
| optimized-v2-r10 | Missing Git/VERSION metadata in copied trees | Harness failure; retained |
| optimized-v2-r10c | Same-commit Git trees and corrected harness | Canonical paired validation |

Excluded attempts are retained in the full local archive and are not silently
combined with canonical measurements.
