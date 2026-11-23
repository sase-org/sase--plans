---
tier: tale
title: Fix the five deterministic Master Gate failures on master
goal:
  "Every Master Gate failure that recurs on all recent master runs passes: the
  session-members facade import, the anonymous-workflow hijack canary (a real TUI status
  regression), the lumberjack audit path, the ffmpeg-dependent video-frame tests, and
  the convergence-repro harness. Only the known intermittent flakes that already have
  beads can still fail."
size: small
proposed_by: bbugyi200.athena.1fd
proposed_with: claude/opus
create_time: 2026-11-22 19:06:26
status: wip
---

# Turn Master Gate green: fix the five deterministic failures on master

## Context

Master Gate (`.github/workflows/master-gate.yml`, 8 test shards) has failed on all of
the last 300 master pushes. The failure notification behind this task is run 45076753261
on `bcaa3a8c0e`. The same failures persist through the latest run, 45081096230, on HEAD
`1b9e88513b`. Across 16 recent runs, five failures recur on every run, and each has a
known, precise root cause. All five reproduce locally on HEAD with direct `pytest`.
Fixing them should leave only the known intermittent flakes listed under "Out of scope".

The five failures and their causes:

1. **Collection ImportError, plus a downstream contract-manifest failure.**
   `tests/ace/tui/models/test_agent_session_members.py` is a re-export facade
   (`__test__ = False`). It still imports three tests from
   `test_agent_session_members_current` under their old names, but commit `3dc72ddd4c`
   ("feat(tui): unify failure-aware session rollups") renamed them. The broken import
   fails collection on two shards. It also makes
   `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
   fail with `pytest -m contract --collect-only failed (exit 2)`.
2. **Hijack canary regression, a real product bug.**
   `tests/fakey/test_agent_hijack_canary.py::test_anonymous_tmp_workflow_hijack_continue_same_workspace`
   times out waiting for a TUI row with status `HIJACKED`. The rows read `FAILED`, even
   though `done.json` records `outcome: hijacked`. The failure started with
   `3dc72ddd4c`: run 45051533953 on its parent `e203b61ffd` passed this test, and every
   run since has failed it. The sase-1ux epic notes called this failure "intermittent",
   which is wrong.
3. **Stale audit path** (task bead **sase-1uz**).
   `tests/test_project_display_presentation_audit.py::test_project_metadata_loaders_stay_at_reviewed_tui_worker_boundaries`
   fails with
   `missing=[('src/sase/axe/lumberjack.py', '_run_tick', 'refresh_project_display_snapshot')]`.
   Commit `97a2f821c2` moved `_run_tick` into `src/sase/axe/_lumberjack_tick.py`.
4. **Video tests depend on host ffmpeg** (task bead **sase-1u8**). Five nodes in
   `tests/ace/tui/artifact_file_viewer/test_video_frames.py` fake `subprocess` but not
   `shutil.which`. GitHub runners have no ffmpeg or ffprobe, so production code returns
   "executable not found" before it reaches the fakes.
5. **Convergence harness missing a method.**
   `tests/ace/tui/test_agent_session_status_convergence_repro.py::test_settled_monitor_replay_converges_after_index_upsert`
   raises
   `AttributeError: '_PreparedApplyHarness' object has no attribute '_surface_token_consumer_drifted'`.
   Commit `d1cc8bfd45` split `EventAutoRefreshTokenMixin._surface_token_drifted` so that
   it now calls `self._surface_token_consumer_drifted`. The test's stand-in harness does
   not subclass the mixin, and it delegates only `_last_surface_token` and
   `_agents_token_reload_kind`.

## Changes

### 1. Re-sync the session-members re-export facade

In `tests/ace/tui/models/test_agent_session_members.py`, replace the three stale names
in the `test_agent_session_members_current` import block, and in `__all__`, with the
names `3dc72ddd4c` gave them:

- `test_lane_entries_keep_failed_bucket_when_followup_errored` becomes
  `test_root_rollup_keeps_failed_bucket_when_followup_errored`.
- `test_lane_entries_map_final_launched_monitor_to_running` becomes
  `test_root_rollup_maps_final_launched_monitor_to_running_lane`.
- `test_lane_entries_skip_non_final_failed_monitor_with_launched_followup` becomes
  `test_root_rollup_skips_failed_monitor_handed_to_later_continuation`.

Keep both lists sorted, leave `__test__ = False` in place, and do not otherwise expand
the facade. The facade re-exports only the tests from the original module, so later
additions such as the `test_running_monitor_row_*` tests stay out. Afterwards, check
that every name the facade imports is still present in its source module's `__all__`.
The ImportError proves it does not today.

### 2. Keep a settled hijack status on the root (product fix)

In `src/sase/ace/tui/models/_agent_status_apply.py`, inside `apply_status_overrides`,
`3dc72ddd4c` added a newest-failed-child rule to the settled-newest block:
`newest_is_failure_frontier`, which leads to `_mirror_root_from_child(parent, newest)`.
In the anonymous tmp workflow case, the root row has status `HIJACKED`, loaded from the
`done.json` hijack outcome. Its workflow step child for the same run reads `FAILED`
because the hijack SIGTERM'd the runner (exit 143). The new rule copies that `FAILED`
onto the root.

`src/sase/ace/tui/models/_dedup.py` already states the governing rule in its comment on
`_HIJACK_SETTLED_STATUSES`. The hijack settlement labels (`HIJACKED`, `ESCALATED`,
`CONTINUED`) "must win over that inferred failure or the TUI row reads FAILED instead of
HIJACKED." Apply the same rule here:

- Move the set to a public constant, `HIJACK_SETTLED_STATUSES`, in
  `src/sase/ace/tui/models/agent_status.py` next to `HIJACKED_STATUS`, keeping the
  `_dedup.py` explanatory comment with it. Import it in `_dedup.py`, replacing the
  private `_HIJACK_SETTLED_STATUSES` and all its uses, and in `_agent_status_apply.py`.
  Do not import a private name across modules; symvision flags private misuse.
- Gate only the failure-frontier clause on it:
  `newest_is_failure_frontier = parent.status not in HIJACK_SETTLED_STATUSES and newest_bucket == "Failed" and ...`.
  Do not gate the `is_plan_root` clause or the `row_is_agent_session_turn(newest)`
  clause. A hijack-settled root still mirrors a newer monitor or gate turn, exactly as
  it did before `3dc72ddd4c`. The parent-is-Failed clause cannot match, because these
  statuses do not bucket as Failed.

A temporary local experiment with exactly this guard confirmed the fix: the canary
passed in about 14s, where it had been timing out at 60s.

Add a fast regression unit test to
`tests/ace/tui/models/test_failure_rollup_consistency.py`, using its existing `_row`
helper and `apply_status_overrides`. A plain agent-session root with status `HIJACKED`,
plus a settled `FAILED` child launched after it, keeps `HIJACKED` after
`apply_status_overrides`, and its bucket is not `Failed`. Parametrize over `HIJACKED`,
`ESCALATED`, and `CONTINUED`. Keep the existing positive case, where a plain `DONE` root
mirrors a newest `FAILED` member, untouched; it is
`test_plain_agent_session_root_mirrors_newest_failed_member`.

### 3. Point the project-display audit at the real lumberjack refresh site (closes sase-1uz)

In `tests/_project_display_presentation_audit_metadata.py`, replace
`src/sase/axe/lumberjack.py` with `src/sase/axe/_lumberjack_tick.py` in two places: the
audited flow-file tuple (around line 36) and the `_ALLOWED_METADATA_LOADERS` key
`(..., "_run_tick", "refresh_project_display_snapshot")` (around line 106). Keep the
existing reason string. Do not weaken the boundary assertion.

`src/sase/axe/_lumberjack_shared.py` holds only a Protocol stub, `def _run_tick(...)`,
and contains no loader call, so it needs no entry. Confirm that the facade
`src/sase/axe/lumberjack.py` no longer contains any of the
`_PROJECT_METADATA_LOADER_NAMES` calls. If so, dropping it from the audited inventory is
correct. If it still contains one, keep the facade listed.

### 4. Fake executable discovery in the video-frame tests (closes sase-1u8)

In `tests/ace/tui/artifact_file_viewer/test_video_frames.py`, add a small fixture.
Prefer a fixture in the module, unless `conftest.py` in that directory is the more
natural home. The fixture monkeypatches `shutil.which` so that `ffmpeg` and `ffprobe`
resolve to fake paths such as `/usr/bin/ffmpeg`, and it delegates every other name to
the real `shutil.which` captured before patching. Use it in the five failing tests:

- `test_extract_frame_caches_and_reports_errors`
- `test_scrub_track_cancel_kills_process`
- `test_scrub_track_failed_process_reports_error`
- `test_metadata_service_caches_payload`
- `test_metadata_service_live_fallback_without_cache`

Production checks `shutil.which` through the `shutil` module at
`_viewer_video_extract.py:54`, `_viewer_video_scrub.py:113`,
`_viewer_video_cache.py:187`, and `_viewer_video_media.py:132`. Patching `shutil.which`,
which is what `test_extract_frame_missing_ffmpeg` already does, therefore reaches all
four.

Leave `nice` delegated to the real `which`, so the scrub argv matches the host the way
the existing golden test expects. Do not change `test_extract_frame_missing_ffmpeg` or
the `skipif`-guarded real-ffmpeg test. Make no production change.

### 5. Delegate the new token-drift method in the convergence harness

In `tests/ace/tui/test_agent_session_status_convergence_repro.py`, add the following to
`_PreparedApplyHarness`, next to the existing `_last_surface_token` and
`_agents_token_reload_kind` delegates:

```python
def _surface_token_consumer_drifted(
    self, snapshot: SurfaceTokenSnapshot | None, surface: str
) -> bool:
    return EventAutoRefreshMixin._surface_token_consumer_drifted(self, snapshot, surface)
```

This method needs `_axe_consumes_detail_parts` only for the `axe` surface, and this test
uses only `agents`. If the test still raises a missing-attribute error, delegate that
method the same way. Do not change production code for this item.

## Verification

Run the following tests with direct `.venv/bin/pytest -q -p no:cacheprovider`. Each must
pass:

- `tests/ace/tui/models/test_agent_session_members.py`,
  `tests/ace/tui/models/test_agent_session_members_current.py`, and
  `tests/test_contract_manifest.py`. Collection must succeed, and
  `pytest -m contract --collect-only` must exit 0.
- `tests/fakey/test_agent_hijack_canary.py`, the whole file.
- `tests/ace/tui/models/test_failure_rollup_consistency.py`, with the new parametrized
  regression test, plus `tests/ace/tui/models/`, so that no other status rollup test
  changes behavior.
- `tests/test_project_display_presentation_audit.py`.
- `tests/ace/tui/test_agent_session_status_convergence_repro.py`.
- The CI no-ffmpeg condition. On athena, which has ffmpeg, the following command
  reproduces exactly the five CI failures before the fix and must pass after it:
  `d=$(mktemp -d); timeout 200 env PATH="$d" ./.venv/bin/python -m pytest -q -p no:cacheprovider tests/ace/tui/artifact_file_viewer/test_video_frames.py; rmdir "$d"`.
  Also run the file normally, with ffmpeg on PATH.

Then run `sase tool run check`, as `lint_and_test.md` requires.

After the fixes land, close the two task beads with verification notes:
`sase bead close sase-1uz --note "..."` and `sase bead close sase-1u8 --note "..."`.
Each note names the test(s) and the commands above. Items 1, 2 and 5 have no beads, and
none need to be created.

## Out of scope: intermittent flakes that already have their own beads

These failures still appear on some Master Gate runs, so the gate can still go red
occasionally after this plan lands. Leave them to their own beads; do not touch them
here.

- **sase-1v2**:
  `tests/ace/tui/test_refresh_warmup_gc.py::test_auto_refresh_merge_live_hints_and_warmup_release_obsolete_payloads`
  sometimes leaks one `FinalizerInventoryWorkerResult` into the GC measurement. This
  happened in about 5 of the last 22 runs, including the bcaa3a8 run.
- **sase-1ty**: `tests/ace/tui/modals/test_preview_panel_video_modal.py` double-loads
  frame 0 under xdist (`[0.0, 0.0] == [0.0]`).
- Sporadic one-offs that did not recur in the latest runs:
  `test_typed_clan_kill_after_pause_stays_cancelled`,
  `test_admitted_run_executes_its_child_once`, `test_chops_run_concurrently`, and the
  link-follow tests. The terminology-guard failure was fixed by `4a058c8e98`. The
  `test_surface_tokens.py` failure was fixed by `1b9e88513b`.
