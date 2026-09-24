# Validation artifacts

This directory contains the compact evidence package for the adjacent
`optimized-craft/patch.diff`.

- `validation-report.md`: human-readable Gate report.
- `final-gate.json`: machine-readable submission Gate.
- `performance-summary.json`: all 18 paired conditions, absolute `ns/op`
  summaries, and bootstrap intervals.
- `performance-samples.csv`: all 270 matched baseline/candidate `ns/op`
  observations used by the summary (18 conditions × 15 pairs).
- `benchstat-targets.md`: reviewer-facing target benchmark table with absolute
  baseline and candidate time/op columns.
- `latest-master-status.tsv`: 13 latest-master correctness/build statuses.
- `paired-validation-status.tsv`: 225 paired-validation statuses.
- `machine-code-summary.md`: rva20/rva22 lowering and fallback audit.
- `attempt-lineage.md`: canonical and excluded attempts.
- `reproduction.md`: commands and measurement design.
- `SHA256SUMS`: package integrity manifest.

Raw logs and binaries remain in the local validation archive; they are omitted
from this compact review package. The complete numerical benchmark evidence is
included here, so percentage deltas can be independently checked.
