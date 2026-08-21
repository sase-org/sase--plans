---
tier: tale
title: Fix toobig_split hard-limit handling
goal:
  Hard-limit findings produce split-file proposals while genuine scanner failures remain
  visible.
size: small
proposed_by: bbugyi200.athena.09c
create_time: 2026-08-21 09:29:39
status: wip
---

# Fix `toobig_split` hard-limit handling

## Context

The configured `toobig_split[sase]` chop succeeds while `toobig --files-only` finds only
informational or warning-level files, but changes to `check_error` as soon as a file
exceeds the hard line limit. `toobig` intentionally returns exit code `1` for a
successful scan containing a hard-limit violation and writes the matching paths to
stdout. The `bugyi-chops` scanner adapter currently treats every nonzero exit code as an
execution failure before consuming stdout, conflating actionable findings with scanner
failures. The same exit code is also used for invalid inputs and filesystem errors,
where `--files-only` produces no path payload, so the fix must retain a fail-closed
distinction.

## Implementation

1. Open the `bbugyi200/bugyi-chops` repository through `sase repo open`, then update
   `src/bugyi_chops/toobig_split.py` so `_scan_files` accepts the scanner's two healthy
   outcomes: exit `0`, and exit `1` accompanied by a non-empty files-only stdout
   payload. Continue raising the existing bounded `scanner failed` diagnostic for every
   other nonzero result, including an empty exit-`1` result, and preserve path
   normalization, repository-boundary checks, ordering, and de-duplication.
2. Extend `tests/test_toobig_split.py` with a realistic regression in which the fake
   scanner emits an oversized path and exits `1`; assert that the chop returns an
   actionable result with the expected proposal and violation presentation. Add or
   retain coverage proving that an empty exit-`1` result and other scanner failures
   remain typed `check_error` results.
3. Clarify the `toobig_split` scanner contract in `README.md`: exit `1` with listed
   paths represents hard-limit findings, while malformed or payload-free failures are
   still rejected.

## Validation

1. Run the focused `tests/test_toobig_split.py` suite.
2. Run the repository's `just check` gate (lint, type checks, tests with coverage, and
   package build).
3. Reproduce the original boundary with the real `toobig` CLI against a fixture that
   contains a file above the hard limit and confirm the updated chop produces a normal
   proposal/report instead of `check_error`; also confirm a missing scan tree remains an
   error.

## Non-goals

- Changing `toobig`'s public exit-code semantics.
- Changing AXE's structured chop-result handling, scheduling, de-duplication, or clan
  launch behavior.
- Editing the SASE `toobig_split` configuration or lowering its thresholds.
