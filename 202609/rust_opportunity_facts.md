---
tier: tale
title: Put receipt opportunity facts in the Rust core
goal:
  The Rust core returns a bounded, read-only receipt opportunity report with
  Python-compatible fields and a tested PyO3 binding.
size: medium
proposed_by: bbugyi200.athena.sase-1ah.8.1
bead: sase-1ah.8.1
create_time: 2026-09-26 13:44:28
status: wip
---

- **PARENT:**
  [202609/e4_landing_remainder.md](https://github.com/sase-org/sase--plans/blob/main/202609/e4_landing_remainder.md)
- **BEAD:**
  [sase-1ah.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ah/sase-1ah.8.1.md)

# Put receipt opportunity facts in the Rust core

Implement phase `sase-1ah.8.1` of the approved E4 remainder plan. This is the Rust
report and PyO3 binding only. Phase `sase-1ah.8.2` will replace Python's report
implementation and handle the core revision and released wheel. Keep the existing
`sase tool receipts` JSON envelope and behavior as the parity contract; do not change
the CLI in this phase.

## Contract

Add a versioned `tool_run_receipts_report` request/result wire to
`crates/sase_core/src/tool_run/wire.rs`, with schema version 1 and the same field names
and meanings as `src/sase/tool/receipt_report.py`: window, receipt counts and age/status
items, opportunity groups, repeat runs/duration/hours, top tools, uncomparable runs,
scan truncation, note, and diagnostics. Include the project, days, current time, and
project root in the request so tests are deterministic and the core can resolve
fingerprinted repositories. Reject negative days and invalid schema versions. Preserve
the Python envelope's measurement-only wording: a repeat opportunity is never a covering
receipt and never skips a child run.

Implement the read-only query in `crates/sase_core/src/tool_run/store/`, using
`with_read_store` and handling an absent store or receipt table as the current CLI does.
Read a bounded newest-run window (500), bound receipt and uncomparable output, and keep
Git calls at or below 400 with a per-call timeout. Never migrate, quarantine, or
otherwise write the ledger for this report. Do not return argv, output, raw environment
values, or other secrets.

Implement comparison in Rust within the `tool_run` domain, reusing the existing
fingerprint and receipt proof types where useful. The non-tree key retains tool,
definition digest, extra-args digest, toolchain, environment, and input content
identity. Compare each repository's effective path content at its recorded HEAD plus
dirty/untracked content hashes; different HEADs can be equivalent when their Git blobs
match. Resolve only the current project and configured linked repository checkouts under
the supplied root, validate repo-local paths, cache Git facts, and treat missing
objects, failed Git reads, malformed or incomplete fingerprints, and exhausted budgets
as uncomparable rather than equivalent. Keep output ordering deterministic and calculate
repeat duration from all but the first run in each group.

Expose and register `tool_run_receipts_report` in
`crates/sase_core_py/src/telemetry/mod.rs` following the neighboring receipt bindings.
The binding accepts the request dict, returns a serialized result dict, and has a
round-trip test.

## Verification

Use Rust fixtures that mirror `tests/tool/test_receipt_report.py` for empty store,
same-tree repeats, dirty-tree-to-committed-HEAD equality, incomplete fingerprints, and
output parity. Add focused core cases for missing Git history, missing receipt tables,
retention/window bounds, scan/output/Git-call limits, and read-only ledger bytes. Test
the binding round trip and schema rejection. Run targeted `just test -p sase_core` and
`just test -p sase_core_py` while iterating, format with `just fmt`, then run the
required `sase tool run check` in the changed `sase-core` repo. Do not run `check-full`.

Before closing only `sase-1ah.8.1`, run `sase bead epic-symbols sase-1ah.8.1` and
resolve or re-key every remaining phase symbol. Record a `PROPOSED FOLLOW-UP:` note on
this phase for any independent clean-base failure or discovered out-of-scope issue.
Close this phase with a note stating the tests and report behavior verified. Leave
`sase-1ah.8`, `sase-1ah`, and their plan beads open.
