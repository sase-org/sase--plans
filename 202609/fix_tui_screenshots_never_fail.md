---
tier: epic
title: Make fix-tui-screenshots salvage, retry, and warn instead of failing
goal: "`just fix-tui-screenshots` in update mode applies every golden it can prove,
  retries the tests and captures that fail or flicker, leaves only the rest untouched
  behind a loud warning, and exits 0. It fails only when it is invoked wrongly, when the
  environment must refuse, or when no capture inventory can be produced at all.
  `--check` stays strict for CI.

  "
phases:
  - id: marker-evidence
    title: Stop unmarked tests from blocking full inventories
    depends_on: []
    size: small
    description: "marker-evidence: make the capture plugin ignore marker-based
      deselection of tests that cannot produce PNG goldens. Mark the stray
      startup-regression module as visual and add a static guard so no test module under
      a visual root misses the marker.

      "
  - id: salvage
    title: Per-node salvage, recovery retries, and partial apply
    depends_on: []
    size: medium
    description: 'salvage: in update mode, trust captures per test node instead of per
      run. Rerun failed or lost nodes a bounded number of times, apply every trusted
      change, and skip the rest with warnings under a new `partial` status. Downgrade
      whole-run evidence problems to "stale removal skipped" and exit 0.

      '
  - id: verify-agreement
    title: Per-golden determinism agreement
    depends_on:
      - salvage
    size: medium
    description: "verify-agreement: replace the all-or-nothing determinism verification
      with per-golden agreement voting over bounded serial re-verification. Apply agreed
      captures and skip unstable goldens with warnings.

      "
  - id: invocation
    title: Lock waiting and worker-count translation
    depends_on:
      - salvage
    size: small
    description: "invocation: wait (bounded) for the checkout-local maintenance lock
      instead of refusing at once. Translate `-n/--numprocesses` selector arguments into
      the governed `SASE_PYTEST_WORKERS` request instead of letting pytest reject them.

      "
  - id: docs
    title: Document the partial-success contract and prove a full run
    depends_on:
      - marker-evidence
      - salvage
      - verify-agreement
      - invocation
    size: small
    description:
      "docs: update the Justfile comments, tool help, and developer docs for the new
      exit contract. Record the needed memory-note changes as a follow-up, then prove
      that a real full update run exits 0 on this host."
proposed_by: bbugyi200.apollo.1i
create_time: 2026-09-22 10:17:55
status: wip
---

- **PROMPT:**
  [prompts/202609/fix_tui_screenshots_never_fail.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fix_tui_screenshots_never_fail.md)

# Why `just fix-tui-screenshots` fails today

Evidence comes from this machine (apollo). It covers 569 retained run manifests under
`.pytest_cache/sase-visual/runs/*/manifest.json` in three sibling agent workspaces
(2026-09-20 to 2026-09-22), the `tool_calls.jsonl` logs of recent `ace-run` agents, and
the related beads.

- Update-mode runs: 318 succeeded and 138 failed with exit 3. Only **1 of 10**
  full-scope update runs succeeded. The failures break down as follows:
  - About 70
    `determinism verification disagreed with the first capture: verify_hash_mismatch:<golden>`.
    These are goldens whose second capture differed under host load (beads sase-144,
    sase-14w, sase-15y, sase-119, sase-14b, sase-155).
  - 42 `visual pytest failed; candidates were retained but goldens were not changed` and
    21 `determinism verification pytest failed`. Some are genuine test failures:
    - `test_selected_gate_shell_output_png_snapshot` has failed deterministically in
      every full run since 2026-09-21 (sase-151).
    - Load-induced timeouts (sase-160).
    - A cluster of tests that were broken on master for a while (sase-13y, sase-13v).
  - **About 30 of those runs had every test pass.** The session-scoped temp-leak guard
    (`tests/_tmp_leak_guard.py`) flipped pytest's exit status to 1. The cause was
    `muse-*` and `sase-codex-usage-*` temp dirs created by _other_ live processes on the
    host (sase-15x).
  - 4 runs exited 5 because the selection matched no tests. One run exited 4 because
    `-n 2` was passed and `tools/run_pytest` rejects `-n` (it wants
    `SASE_PYTEST_WORKERS`).
- Some failures wrote no manifest at all:
  - An agent's earlier foreground run was killed by its tool timeout while the child
    kept running. The retry then hit
    `another fix-tui-screenshots run holds the maintenance lock` (exit 2).
- A latent blocker for every full run:
  - Commit 13467a570 added `tests/ace/tui/visual/test_ace_png_snapshot_startup.py`
    without `pytestmark = pytest.mark.visual`. The visual lane (`-m visual`) deselects
    its three tests, and the capture plugin records them as `deselected_visual_node:*`.
    A requested full inventory can therefore never be proven: the runner raises
    `requested a full visual inventory but completeness evidence is missing`.
  - This error is currently masked because sase-151 fails first.
  - The module also runs in no lane at all, because the fast lanes ignore the visual
    directories.
- How agents coped:
  - Rerunning, `-k`/`--deselect` exclusions, per-file batching, and
    `SASE_TMP_LEAK_GUARD_DISABLED=1`.
  - One agent hand-wrote a `/tmp/sweep.sh` that copied candidates into goldens, and
    another looped for about 15 hours.
  - After a failed run the summary still printed `counts: created=0 updated=0 ...`. At
    least one agent read that as "nothing to update" and committed.

The root cause is policy, not a single bug. The runner built by epic sase-12z
(`tests/ace/tui/visual/_visual_maintenance_run.py::_run_locked`) is deliberately
all-or-nothing:

- Any nonzero child exit, incomplete inventory, protocol error, verification mismatch,
  or concurrent golden edit raises `MaintenanceError`, and nothing is applied.
- A zero-test selection is an error.
- Verification is "a bounded second pass, not retry-until-green".

The user now wants the opposite contract for update mode. The command should basically
never fail: it should figure out a way to update the goldens it needs, or succeed with a
warning.

# Contract after this epic

Safety properties that stay:

- A golden is never written from a node that failed, or from a capture that was not
  reproduced.
- A golden is never deleted as stale without complete evidence.
- Every apply is journaled and rolled back on error.
- Check mode never writes.
- CI refusal and renderer/platform refusal stay in place.

What changes is the unit of failure: from the run to the test node or golden.

| Update-mode situation                                                 | Behavior after this epic                                                                                                 |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Some nodes fail, error, or are lost with a worker                     | Rerun only those nodes (bounded). Apply everything trusted. Skip unrecovered nodes (`test_failed`). Exit 0 `partial`     |
| pytest exits nonzero but no visual node failed (e.g. temp-leak guard) | Apply normally. Print a warning naming the capture log. Status `applied`/`clean`, exit 0                                 |
| A candidate is not reproduced by verification                         | Serial re-verification (bounded), then majority agreement. Unresolved goldens are skipped (`unstable`). Exit 0 `partial` |
| Full inventory cannot be proven (failed nodes, protocol errors, etc.) | Apply creates and updates. Skip stale removal and report no stale entries. Warn with the reason. Exit 0 `partial`        |
| A golden changed on disk during the run                               | Leave that path alone (`concurrent_edit`). Apply the rest. Exit 0 `partial`                                              |
| Selection matched no tests (pytest exit 5)                            | Warn `selection matched no visual tests; nothing was checked or updated`. Status `clean`, exit 0                         |
| `-n N` / `--numprocesses N` after `--`                                | Translate to `SASE_PYTEST_WORKERS=N` for the governed runner                                                             |
| Another run in this checkout holds the lock                           | Wait (bounded, with periodic notices), then run                                                                          |

Update mode exits nonzero only in these cases:

- **Exit 2:** CLI usage errors, pytest usage errors (child exit 4 with nothing
  executed), CI update refusal, non-Linux or renderer refusal, lock-wait timeout, or an
  unfinished-journal conflict.
- **Exit 3:**
  - No usable capture inventory after one full retry of the capture pass (a crash, or
    collection interrupted before any test ran).
  - An apply I/O failure, after rollback.
  - An interrupt.

Check mode (`--check`, `just test-visual`, CI's `visual-test` job) keeps today's strict
semantics and exit codes. Only the marker-evidence fix changes what it considers
complete.

Manifest additions:

- `status` gains `partial`: something the run should have handled was left untouched.
- Top-level `warnings: [str]` and `skipped: [record]`.
  - Each skipped record has `kind` (`node` or `golden`), `node_id` and/or `path`, a
    `reason` slug (`test_failed`, `unstable`, `verify_failed`, `owner_mismatch`,
    `concurrent_edit`, `protocol_error`), a short `detail`, and evidence paths (logs,
    per-attempt candidate PNGs).
- `attempts`: one entry per pytest pass, with label, run id, capture dir, log, requested
  workers, and child exit code.
- `pruning_skipped_reason`.

Keep existing fields and kinds. Consumers include
`tools/render_visual_snapshot_failure_report`, `latest-report.json`, and the tests.

## Code map (all test tooling, so it stays in this repo, not the Rust core)

- `tools/fix_tui_screenshots`: thin entry point into
  `tests/ace/tui/visual/_visual_maintenance.py` (the façade).
- `_visual_maintenance_run.py`: `run_maintenance` / `_run_locked` orchestration, 435
  lines.
- `_visual_maintenance_exec.py`: `run_governed_visual_pytest`, `verify_changes`,
  `assert_pruning_safe`.
- `_visual_maintenance_compare.py`: `classify_captures`,
  `compare_verification_captures`, and the exact comparator.
- `_visual_maintenance_cli.py`: `parse_command`, `resolve_scope`,
  `validate_pytest_args`, and the help epilog.
- `_visual_maintenance_lock.py`: `MaintenanceLock` (non-blocking `flock`).
- `_visual_maintenance_baseline.py`: `detect_concurrent_edits`,
  `recheck_baseline_or_raise`.
- `_visual_maintenance_manifest.py`: `build_manifest`, `failure_manifest`,
  `print_summary`, report publishing.
- `_visual_maintenance_types.py`: exit codes, statuses, `MaintenanceHooks`,
  `MaintenanceRequest`, `VisualPytestFn`.
- Capture protocol:
  - `tests/_visual_capture_plugin.py` (per-worker `WorkerSessionRecord`,
    `pytest_deselected`, `pytest_runtest_logreport`).
  - `tests/ace/tui/visual/_visual_capture_inventory.py` (`evaluate_inventory` reasons,
    `full_inventory`, `complete`).
  - `_visual_capture_records.py`.
- Tests:
  - `tests/test_fix_tui_screenshots.py` (CLI and guards).
  - `tests/test_fix_tui_screenshots_apply.py` (outcome matrix via
    `tests/_fix_tui_screenshots_helpers.py::FakeRunner` / `silent_hooks`).
  - `tests/ace/tui/visual/test_fix_tui_screenshots.py` (visual lane).
  - `tests/test_visual_capture_*.py`.
  - `tests/test_render_visual_snapshot_failure_report.py`.
- `toobig` limits files to 700/850/1000 lines. Put new logic in new focused modules, for
  example `_visual_maintenance_salvage.py` and `_visual_maintenance_verify.py`, rather
  than growing `_run_locked`. Keep `symvision` clean.

# Phase marker-evidence

1. Add `pytestmark = pytest.mark.visual` to
   `tests/ace/tui/visual/test_ace_png_snapshot_startup.py` so its sase-14q regression
   tests run in the visual lane. Confirm they pass there.
2. In `tests/_visual_capture_plugin.py::pytest_deselected`, stop recording a deselected
   item as unexpected visual deselection when both of these hold:
   - it has no `visual` marker (`item.get_closest_marker("visual") is None`);
   - it does not request `ace_png_visual` or `pager_png_visual` (`item.fixturenames`).

   Such an item cannot produce a golden, and the original design already says that
   expected exclusion of non-visual tests is not incomplete inventory. Keep recording
   deselection of marked or PNG-fixture items exactly as today, so a mis-marked snapshot
   test still blocks pruning instead of getting its golden deleted.

3. Add a fast-lane structural test outside the visual directories, e.g.
   `tests/test_visual_tree_markers.py`. It statically parses (no imports) every
   `test_*.py` under `tests/ace/tui/visual/` and `tests/pager/visual/` and requires a
   module-level `pytestmark` that includes `pytest.mark.visual`. Name the offending file
   in the failure.
4. Tests in `tests/test_visual_capture_*.py`:
   - An unmarked, fixture-free deselected node no longer produces
     `deselected_visual_node:*` and does not block `full_inventory`.
   - A deselected node that is marked visual or uses a PNG fixture still does.
   - Use the existing small xdist fixture-project pattern where the current tests
     already do.

This phase is independent of the others. It also fixes check mode: a clean full
`--check` run can be complete again.

# Phase salvage

1. **Runner seam.**
   - Add an optional `workers: int | None` to `VisualPytestFn`, `run_pytest`, and
     `run_governed_visual_pytest`.
   - When it is set, run the child with `SASE_PYTEST_WORKERS=<n>` in its environment
     (copy `os.environ`; do not mutate it). `None` preserves today's governed default.
   - Extend `FakeRunner` so tests can script per-attempt outcomes: failed, errored, or
     lost nodes; a session-only nonzero exit; no inventory; and per-attempt PNG bytes
     per node.
2. **Node trust (new module).** Derive from a loaded `InventoryReport`:
   - _Trusted_ nodes: executed, not failed, not errored in setup or teardown, not
     xfailed/xpassed, and reported by a worker whose session completed.
   - _Recover_ set: failed ∪ errored ∪ collected-but-unaccounted visual nodes. This
     includes nodes on lost or incomplete workers, whose `session.json` only has
     collection data.
   - Skipped and xfailed nodes keep today's semantics.
   - Only captures whose `node_id` is trusted are candidates. Partial captures from a
     failed node are never applied.
   - If the child or session exit is nonzero but nothing is in the recover set, record a
     warning. For example:
     `pytest exited 1 although no visual test failed (often the temp-leak guard); see <capture.log>`.
     Proceed normally, and do not disable the leak guard.
3. **Recovery passes.**
   - Run at most 2 attempts. Each reruns only the node IDs still in the recover set,
     never the caller's selectors, through the same governed runner. Write to
     `<run_dir>/recover-<n>/` with `recover-<n>.log` and run id `<run_id>-recover-<n>`.
   - Use `workers=1` when 25 or fewer nodes remain (serial, which is what cures
     load-induced timeouts). Otherwise use the governed default (`None`).
   - A node that becomes trusted takes its candidates from that attempt.
   - Nodes still failing after the attempts become `skipped` records (`kind: node`,
     `reason: test_failed`). Include the attempt count, log paths, and, when it can be
     extracted from the log, the pytest `FAILED ... - <message>` line. Their existing
     goldens stay untouched.
4. **Whole-run failures.** These apply when pass 1 wrote no `inventory.json` or executed
   zero visual nodes:
   - Child exit 5 in update mode: status `clean`, exit 0, with the warning
     `selection matched no visual tests; nothing was checked or updated` plus the
     selectors.
   - Child exit 4: exit 2, pointing at the pytest usage error in the log.
   - Anything else: rerun pass 1 once with the same scope and arguments. If it still
     yields nothing usable, exit 3 with status `failed` and both log paths. This is the
     one deliberately fatal outcome, because nothing was captured to salvage.
   - Check mode keeps today's errors for all of these.
5. **Classification.**
   - Classify the union of trusted pass-1 captures and recovered captures with the
     existing exact comparator. Either build a merged inventory view or refactor
     `classify_captures` to accept the selected records; keep the check-mode call path
     behaviorally identical.
   - Protocol errors in update mode:
     - Drop the captures for implicated canonical paths (for example duplicate
       ownership) as `golden`/`protocol_error` skips.
     - Errors that cannot be attributed to a path disable pruning and add a warning.
6. **Stale pruning.**
   - Prune only when all of the following hold:
     - pass 1 was requested `full`;
     - after recovery, every collected visual node is trusted or legitimately
       skipped/xfailed under today's rules;
     - pass 1's remaining completeness blockers are limited to reasons that recovery
       cured (`failed_node`, `error_node`, `unaccounted_node`, `incomplete_worker`,
       `lost_worker`, `missing_worker`, `session_exitstatus`);
     - there are no protocol errors;
     - `assert_pruning_safe`'s both-roots rule holds for the trusted captures.
   - Otherwise skip stale removal, report no stale entries, and set
     `pruning_skipped_reason` and a warning. A requested full run in that state is
     `partial`.
   - Replace the current `raise`s for "full inventory ... evidence is missing" and
     "capture inventory is incomplete" in update mode. Keep them in check mode.
7. **Concurrent edits.** In update mode, replace `recheck_baseline_or_raise` with
   filtering. Changes whose path appears in `detect_concurrent_edits` become
   `concurrent_edit` skips and are removed from the apply set (including stale
   deletions). Apply the rest.
8. **Status and exit.**
   - `partial` when any `skipped` record exists, or when a requested full run skipped
     pruning. Otherwise `applied` or `clean`.
   - Update mode returns 0 for `clean`, `applied`, and `partial`.
   - Add `STATUS_PARTIAL` to the types module.
   - `verify_changes` is untouched in this phase; verify-agreement replaces it.
9. **Summary and report.**
   - When `warnings` or `skipped` are non-empty, `print_summary` prints a `WARNING:`
     block before the counts. It lists each skip (reason, node or path, attempts,
     evidence path) and the pruning-skipped reason. It ends with a line saying those
     goldens were left unchanged and are not known to be current.
   - For `failed`/`refused` runs, print the error first and state
     `no goldens were changed` instead of a bare zero-count line.
   - Teach `tools/render_visual_snapshot_failure_report` to render `partial` and a "Not
     updated" section with evidence links. Keep legacy `failure.json` input working.
     `latest-report.json` carries the new status.
10. **Tests.** Use `tests/test_fix_tui_screenshots_apply.py` and the helpers. Rewrite
    these tests to the new contract:
    - `test_pytest_failure_does_not_apply`: passing nodes are applied, the failing node
      is skipped, exit 0 `partial`.
    - `test_zero_tests_are_an_error`: update exits 0 with a warning; check is still an
      error.
    - `test_concurrent_edit_refuses_apply`: only the conflicting path is skipped.

    Add tests for:
    - recovery succeeding on attempt 2 and applying that attempt's candidates;
    - recovery exhausted;
    - a session-only nonzero exit (applied, with a warning);
    - lost-worker nodes being recovered;
    - a full run with an unrecovered failure (no deletion, `partial`);
    - a full run fully cured by recovery (stale removal still happens);
    - no inventory twice (exit 3) and exit 4 (exit 2);
    - `workers=1` being passed for small recover sets;
    - check mode unchanged for each case above.

    Keep these tests runnable without the visual extra.

# Phase verify-agreement

1. Replace `verify_changes` with per-golden agreement in a new module, and call it from
   the orchestration introduced by salvage.
   - Each created or updated candidate starts with one sample: its first trusted capture
     hash (pass 1 or recovery).
   - **Verify attempt 1** is today's pass: owner node IDs, governed default workers,
     `<run_dir>/verify/`. Add samples only from nodes trusted under the salvage rules,
     so session-only exits (the leak guard) no longer matter.
   - A golden is _decided_ the first time any hash reaches two samples.
   - Undecided goldens have a hash mismatch, a missing capture, an owner that failed or
     was lost, or a key-set difference. Rerun only their owner nodes with `workers=1`
     into `verify-2/` and `verify-3/`, for at most 2 extra attempts.
   - Decided with hash H:
     - Use the candidate PNG/SVG bytes from the attempt that produced H.
     - Re-classify against the run-start baseline with the existing exact comparator.
     - If H is pixel-equal to the baseline golden, the result is `unchanged`: no write
       and no mtime change.
   - Undecided after the attempts:
     - `unstable` (distinct hashes, sample count, per-attempt candidate paths) when
       samples disagree.
     - `verify_failed` when the owner never passed verification.
     - The golden is left untouched.
   - `verify_owner_mismatch` becomes an `owner_mismatch` skip.
   - Goldens that appear only in a verify attempt, and were not in the change set,
     produce a warning only.
2. Record each verify attempt in the manifest `attempts` list. Keep all attempt
   directories and logs in the run dir for diagnosis, and show `unstable` skips in the
   report with their candidates side by side.
3. Cost is bounded:
   - Re-verification reruns only the owners of undecided goldens.
   - There are at most 3 verify attempts in total.
   - Stale removal needs no verification (unchanged).
   - Check mode still runs no verification pass.
4. Tests:
   - First verify agrees: applied.
   - First verify flickers, then agrees: applied, including the case where the agreed
     bytes come from a verify attempt rather than pass 1.
   - Agreement equals the baseline: unchanged, file bytes and mtime untouched.
   - Never agrees: that golden is skipped `unstable`, other goldens are applied, exit 0
     `partial`.
   - A verify node fails and then passes serially.
   - A verify session-only nonzero exit is ignored.
   - Owner mismatch is skipped.
   - Serial `workers=1` is requested for re-verification.
   - Replace `test_failed_verification_aborts_update`.

# Phase invocation

1. **Lock wait.**
   - `exclusive_maintenance_lock` polls `flock(LOCK_EX | LOCK_NB)` until it acquires the
     lock or a bound expires. The default bound is 2 hours: full runs on this host take
     12–55 minutes, plus verification.
   - It prints one notice when waiting starts (holder pid, run id, started_at from the
     lock file) and at most one per minute after that.
   - On timeout it raises `OverlappingRunError` (exit 2) naming the holder, without
     touching the holder's scratch.
   - `flock` is already released when the holder dies, so no stale-lock reclaim is
     needed.
   - Add seams (hooks or parameters for timeout, sleep, and clock) so tests stay fast.
   - Update `test_overlapping_run_refuses_without_overwriting_scratch` to cover both
     "waits, then proceeds after release" and "times out with exit 2".
2. **Worker-count translation.**
   - In `parse_command`, recognize `-n N`, `-nN`, `--numprocesses N`, and
     `--numprocesses=N` among the pytest arguments.
   - Remove them from `pytest_args`, and store an integer on a new
     `MaintenanceRequest.workers`. Pass it to the capture pass and to any pass that does
     not force `workers=1`.
   - `auto`/`logical` are dropped with a printed note (governed default). Any other
     non-positive or non-integer value is a `UsageError`.
   - `resolve_scope` must no longer see the value token. Today `-n 2` without a path
     silently makes the scope `targeted`, because `2` looks like a selector.
   - A `-n` inside `PYTEST_ADDOPTS` is rejected up front with a `UsageError` that names
     `SASE_PYTEST_WORKERS`, instead of a late pytest exit 4.
   - Add CLI tests in `tests/test_fix_tui_screenshots.py`.
   - Keep the parser's public options unchanged (`-c/--check`, `-h/--help`), so
     `test_parser_only_exposes_check_and_help` still holds.

# Phase docs

1. Update the Justfile comments above `fix-tui-screenshots`, `update-visual-snapshots`,
   and `check-full`. Update `_HELP_EPILOG` in `_visual_maintenance_cli.py` with the
   narrowed exit codes and the `partial` status. Adjust
   `test_help_lists_sorted_check_flag_and_exit_codes`.
2. Rewrite the screenshot-maintenance paragraphs in `docs/development.md` (the
   `just fix-tui-screenshots is the canonical maintenance command` section and the exit
   code paragraph) to describe:
   - salvage, recovery, and agreement;
   - the `partial` status and the WARNING block;
   - that stale removal needs complete evidence;
   - that `--check` stays strict.

   Adjust `README.md` only if its sentence becomes inaccurate. Do not edit
   `CHANGELOG.md`; it is generated by release-please.

3. Do not edit `sase/memory/` notes. The user did not request memory changes for this
   plan. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead so the land agent can
   route a `memory` task bead. The note should say that the "PNG Snapshot Tests" section
   of `lint_and_test.md` and the "Golden Maintenance" section of `tui_screenshot.md`
   should state:
   - update mode exits 0 with status `partial` when it leaves goldens untouched;
   - agents must read the WARNING block or the manifest `skipped` list and must not
     treat those goldens as current;
   - `-n N` is accepted;
   - `--check` remains strict.
4. **Acceptance run.**
   - Run a real full `just fix-tui-screenshots` through `/sase_monitor` with the
     `TESTING`/`TESTED` pair and a generous timeout (at least 90 minutes).
   - Expect exit 0. If sase-151 is still open, the status is `partial`, with
     `test_selected_gate_shell_output_png_snapshot` skipped as `test_failed` after its
     recovery attempts.
   - With marker-evidence landed and sase-151 as the only unrecovered failure, pruning
     is still skipped with that reason. If sase-151 has been fixed, expect `applied` or
     `clean` with a complete full inventory.
   - Inspect the retained report and every golden change, as the existing workflow
     requires. Commit genuinely unrelated golden refreshes with the
     `UNRELATED_SCREENSHOT_UPDATES=<reason>` trailer the finalizer already documents.

# Verification for every phase

- Run `just check` (prefer `sase tool run check`) after `just fix`. `just check` does
  not run PNG snapshots, so also run the phase's visual-lane tests explicitly and
  targeted. For example, run
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_fix_tui_screenshots.py` once
  salvage lands, or the marked startup module for marker-evidence, through
  `/sase_monitor` if they run long.
- salvage and verify-agreement: prove the real flow on one small file as well. Run
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel_gate.py`.
  While sase-151 is open, this should exit 0 `partial`: the gate node is skipped after 2
  recovery attempts and the file's other goldens are applied or unchanged. Skip this
  proof if sase-151 has since been fixed.
- Do not run `just check-full` (it is explicit-only).

# Out of scope

- Fixing individual failing or flaky visual tests and their root causes (sase-151,
  sase-160, sase-144, sase-14w, sase-15y, sase-119, sase-14b, sase-155) and the
  codex-usage temp leak (sase-15x). After this epic they degrade to warnings, and their
  beads stay open.
- Check-mode and CI semantics beyond the marker-evidence fix.
- Salvage after collection errors (`--continue-on-collection-errors`). Cross-workspace
  host load. Suite-gate queueing. Monitor output-retention gaps. Agents that run full
  updates in the foreground instead of through `/sase_monitor`.
