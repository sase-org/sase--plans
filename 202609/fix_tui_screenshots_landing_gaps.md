---
tier: tale
title: Close the sase-169 landing gaps in fix-tui-screenshots update mode
goal: A successful capture retry applies the retry's candidates. No dead legacy update-mode
  verify/apply code remains in the check path. Update-mode unfinished-journal conflicts
  exit 2 as the sase-169 contract specifies. Check-mode behavior is unchanged.
size: small
proposed_by: bbugyi200.apollo.sase-169.land
bead: sase-169
status: done
---

- **PARENT:**
  [202609/fix_tui_screenshots_never_fail.md](https://github.com/sase-org/sase--plans/blob/main/202609/fix_tui_screenshots_never_fail.md)
- **BEAD:**
  [sase-169](https://github.com/sase-org/sase--beads/blob/main/pages/sase-169/README.md)

# Close the sase-169 landing gaps in `fix-tui-screenshots` update mode

The land audit of epic sase-169 found three remaining gaps. Epic sase-169 was "Make
fix-tui-screenshots salvage, retry, and warn instead of failing" (plan
`202609/fix_tui_screenshots_never_fail.md`). The epic's commits caused all three gaps,
so they are epic work. The epic stays open until this tale lands.

All code is test tooling under `tests/ace/tui/visual/_visual_maintenance*.py`. It stays
in this repo, not the Rust core.

## 1. A successful capture retry must use the retry's candidates

**Bug (reproduced).** In `_UpdateRun.execute`
(`tests/ace/tui/visual/_visual_maintenance_salvage.py`), the flow runs like this:

1. Pass 1 writes no usable inventory, so `_retry_capture()`
   (`_visual_maintenance_salvage_recovery.py`) reruns the capture into
   `<run_dir>/capture-retry/`.
2. `_retry_capture()` returns the retry's `InventoryReport`.
3. `_ordered_candidates()` still pairs every non-recovered record with
   `self.capture_dir`, which is the pass-1 `capture/` directory.
4. `classify_selected_captures` then cannot find the candidate PNGs. Every golden
   becomes a `golden`/`protocol_error` skip ("missing candidate PNG for ..."). The run
   ends `partial` and applies nothing.

The retry path therefore throws away the one capture it was meant to salvage.

A FakeRunner probe reproduces it. Use `AttemptScript(no_inventory=True, exit_code=1)` as
the only scripted attempt, so the retry falls through to the default all-passing script.
One ACE golden changes. `main([])` exits 0 with `status: partial`, and both goldens are
skipped as `protocol_error`. The target golden is not updated.

**Fix.**

- Track the directory and log that the working inventory came from. A small field on
  `_UpdateRun` is enough, for example an inventory directory that defaults to
  `capture_dir`, plus the matching log key. After a successful retry, set them to the
  retry directory and `capture-retry`.
- Use that directory in `_ordered_candidates`. Also use it everywhere else that resolves
  pass-1 candidate bytes; `_apply_capture_dir` checks `source_dir == self.capture_dir`,
  so it must also accept the retry directory.
- The session-only nonzero-exit warning ("pytest exited N although no visual test failed
  ...") must name the log of the pass whose inventory is used: `capture-retry.log` after
  a retry.
- The manifest's top-level `capture_dir` must name the directory whose inventory was
  used. Its `attempts` list already records both passes.

**Tests.** Add them to `tests/test_fix_tui_screenshots_salvage.py`, using the existing
`FakeRunner` and `AttemptScript` helpers:

- No inventory once, then a clean retry:
  - exit 0 and `status: applied`;
  - the changed golden holds the retry candidate bytes;
  - `skipped == []`;
  - attempt labels are `["capture", "capture-retry", "verify"]`.
- No inventory once, then a retry where one node fails and recovery cures it:
  - candidates come from `recover-1/`;
  - every other candidate comes from `capture-retry/`.
- A retry whose session-only exit is nonzero: the warning names `capture-retry.log`.
- Keep `test_no_inventory_twice_is_fatal` and `test_check_mode_ignores_salvage` green.
  Check mode must not retry.

## 2. Delete the legacy update-mode code the salvage phases left behind

`_run_locked` in `_visual_maintenance_run.py` routes update mode to `run_update`.
`_run_locked_check` is reached only when `request.check` is true. Every
`not request.check` branch inside `_run_locked_check` is therefore dead:

- the old `verify_changes` call;
- the pre-apply manifest;
- `recheck_baseline_or_raise`;
- `apply_changes` and its `KeyboardInterrupt`/`MaintenanceError` terminal-manifest
  handlers;
- the `STATUS_APPLIED` and journal assignments.

The helpers that only this dead code used are now unreferenced. The epic plan said
per-golden agreement _replaces_ `verify_changes`.

**Change.**

- Simplify `_run_locked_check` to the check path only. Keep check-mode behavior
  identical: the same errors, statuses (`drift`/`clean`/`failed`/`interrupted`), exit
  codes, and manifest fields. Here `verify_dir` and the journal are always `None`. Drop
  the now-unused `terminal_manifest` plumbing and the unused imports.
- Delete `verify_changes` from `_visual_maintenance_exec.py`. Delete
  `compare_verification_captures` from `_visual_maintenance_compare.py`, along with its
  re-export and `__all__` entry in the façade `_visual_maintenance.py`. Delete
  `recheck_baseline_or_raise` from `_visual_maintenance_baseline.py` and
  `trusted_captures` from `_visual_maintenance_trust.py`.
- Before deleting each one, `rg` the whole repo (tests, tools, docs, Justfile) to
  confirm nothing else references it. Keep `assert_pruning_safe` and
  `detect_concurrent_edits`: check mode and salvage still use them.
- `_visual_maintenance_verify.py` defines its own `_load_if_present` with
  `# type: ignore[no-untyped-def]`. Give it a real `-> InventoryReport | None`
  annotation and remove the ignore, keeping its exception-tolerant behavior.
  Alternatively, share one typed helper with `_visual_maintenance_salvage_recovery.py`,
  if that keeps the recovery path's current (raising) behavior unchanged.

## 3. Update-mode unfinished-journal conflicts exit 2

The epic contract lists an unfinished-journal conflict under **exit 2**, as an
environment refusal. Today `recover_journal` (`_visual_maintenance_apply.py`) raises a
plain `MaintenanceError`, so `main` returns 3. Phase sase-169.5 documented the
implemented 3 in `_HELP_EPILOG` and asked for the plan and the code to be reconciled.
Follow the contract:

- When update-mode journal recovery finds conflicts, raise an error whose `exit_code` is
  `EXIT_USAGE`. For example, add a `JournalConflictError(UsageError)` in
  `_visual_maintenance_types.py` and raise it from `recover_journal`. Keep the message
  and the journal's `conflict` status write unchanged, so the next run proceeds.
- The check-mode refusal ("unfinished screenshot apply journal found; check mode refuses
  recovery writes") keeps today's exit 3. Check-mode semantics are out of scope.
- Update `_HELP_EPILOG` in `_visual_maintenance_cli.py`: move "unfinished-journal
  conflict" from the exit-3 line to the exit-2 line. Also check `docs/development.md`
  and the Justfile comments above `fix-tui-screenshots`, and fix any exit-code statement
  that becomes inaccurate. Do not edit `CHANGELOG.md` or anything under `sase/memory/`.
  Memory-note updates are tracked separately by task bead sase-16r.
- Tests in `tests/test_fix_tui_screenshots_apply_recovery.py` or
  `tests/test_fix_tui_screenshots.py`:
  - Through `main([])` with a conflicting planned journal under
    `.pytest_cache/sase-visual/runs/`: the result is `EXIT_USAGE`, the golden is
    untouched, and the journal status becomes `conflict`.
  - Through `main(["--check"])` with an unfinished journal: the result is still
    `EXIT_FAILURE`.
  - The existing `pytest.raises(Exception, match="conflicts")` unit test keeps passing.

## Verification

- Run `just fix`, then `sase tool run check`. `just check` is guarded and must run
  through `sase tool run`; do not run `just check-full`. `just check` does not run PNG
  snapshots.
- Run the fast maintenance suites explicitly:
  - `tests/test_fix_tui_screenshots.py`
  - `tests/test_fix_tui_screenshots_apply_basic.py`
  - `tests/test_fix_tui_screenshots_apply_failures.py`
  - `tests/test_fix_tui_screenshots_apply_recovery.py`
  - `tests/test_fix_tui_screenshots_salvage.py`
  - `tests/test_fix_tui_screenshots_verify.py`
  - `tests/test_render_visual_snapshot_failure_report.py`
  - `tests/test_render_visual_snapshot_failure_report_outputs.py`
  - `tests/test_visual_capture_deselection.py`
  - `tests/test_visual_tree_markers.py`
- Prove the real flow still works on one small file. Run
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_link_rail.py`
  and expect exit 0 with status `clean` or `applied`. Then run
  `just fix-tui-screenshots --check -- tests/ace/tui/visual/test_fix_tui_screenshots.py`
  and expect exit 0. Inspect the report if any golden changes, and do not commit
  unrelated golden churn.
- `toobig`, `ruff`, and `mypy` stay clean. The touched modules are well under their
  limits.

## Out of scope

- Fixing individual flaky or failing visual tests (sase-151, sase-160, sase-144,
  sase-14w, sase-15y, sase-119, sase-14b, sase-155).
- Memory notes (sase-16r).
- Check-mode and CI semantics.
- Closing epic sase-169, the symvision pass, and marking its plan file done. The
  sase-169 land agent does those after this tale lands.
