---
tier: tale
title: Deflake the Config Center atomic-save test
goal:
  The Config Center atomic-save test deterministically proves same-directory atomic
  replacement in the full parallel suite.
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.2
bead: sase-ns.6.2
create_time: 2026-08-16 21:07:42
status: done
---

- **PROMPT:**
  [prompts/202608/config_center_atomic_save_deflake.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/config_center_atomic_save_deflake.md)
- **PARENT:** [202608/task_backlog_top5.md](task_backlog_top5.md)
- **BEAD:**
  [sase-ns.6.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.2.md)

# Deflake the Config Center Atomic-Save Test

## Goal

Make
`tests/ace/tui/test_config_center_state.py::test_save_atomically_replaces_existing_state`
deterministic in the full parallel suite while retaining direct proof that saving uses
one same-directory atomic replacement, preserves the old destination until replacement,
publishes the new payload, and removes its temporary file.

## Root Cause and Approach

The test currently patches `config_center_state.os.replace`. Because
`config_center_state.os` is the shared Python `os` module, that patch replaces
`os.replace` process-wide for the duration of the test. A background writer left active
on the same pytest worker can therefore enter the test callback, adding an unrelated
replacement or tripping its path/content assertions. This explains why the node fails
only in full parallel lanes and passes immediately in isolation.

Introduce a narrow, module-local replace seam in `config_center_state` whose production
implementation remains `os.replace`. Patch that seam in the atomic-success and
replace-failure tests instead of mutating the shared `os` module. Keep the existing
success-test assertions intact: exactly one call, source and destination in the state
directory, the prior destination contents visible at replacement time, the expected
destination path, the new on-disk payload, and no leftover temporary file. Keep the
failure test's destination-preservation and cleanup assertions intact as well.

## Implementation

1. Add a private module-local wrapper or bound callable for the final `os.replace`
   operation in `src/sase/ace/tui/modals/config_center_state.py`, and route the save
   through it without changing the tempfile, flush, fsync, cleanup, or exception
   behavior.
2. Update both replacement-intercepting tests in
   `tests/ace/tui/test_config_center_state.py` to patch only that local seam. Add a
   focused regression assertion demonstrating that the process-wide `os.replace`
   function remains untouched while the save is intercepted.
3. Remove this node's exact entry (and its now-empty ownership comment if applicable)
   from `tests/reproducible_flake_baseline.txt` after the deterministic fix is proven,
   preserving all other baseline entries and concurrent phase edits.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the focused Config Center state test module, including the success and failure
   atomic-replace cases.
3. Exercise the node repeatedly under the contention harness to stress the narrow seam;
   treat this as diagnostic support rather than the required whole-suite evidence.
4. Run `just check` and resolve all failures caused by this phase.
5. Run `just check-full` through `/sase_monitor` with a continuation action, and require
   the target node to pass in the whole parallel lane. If an unrelated pre-existing
   failure appears, record it as a `PROPOSED FOLLOW-UP:` note on `sase-ns.6.2` rather
   than creating a bead.
6. Confirm the final diff retains every atomic-replacement assertion, removes only this
   node from the reproducible-flake baseline, and contains no unrelated changes. Close
   only phase bead `sase-ns.6.2` with a note naming the root cause and verification; do
   not change or close `sase-md`, the parent epic, or any ancestor bead.
