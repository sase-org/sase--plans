---
tier: tale
size: medium
title:
  Fix the remaining update-skew auto-restart defects and land epics sase-1j6.10 and
  sase-1j6
goal:
  CI is free of auto-restart-caused failures; deferrals expire, no failure is swallowed
  or touched twice, every decline follows quiet_declines, episode rows carry every
  evidence file and honest copy, the TUI closure drops the epic modules, and epics
  sase-1j6.10 and sase-1j6 are closed with their plan files marked done.
proposed_by: bbugyi200.athena.sase-1j6.10.land
bead: sase-1j6.10
create_time: 2026-10-10 13:48:53
status: done
---

- **PARENT:**
  [202610/finish_update_skew_auto_restart.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_update_skew_auto_restart.md)
- **BEAD:**
  [sase-1j6.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1j6/sase-1j6.10.md)

# Plan: Finish landing update-skew auto-restart (sase-1j6.10, then sase-1j6)

## Context

Epic `sase-1j6.10` (plan `plan:202610/finish_update_skew_auto_restart.md`, a child of
epic `sase-1j6`) closed all seven phases. Its land agent reviewed the source on master
`e5e58ac3d5` and the Master Gate CI logs and found that the epic is not landable yet:

- Two auto-restart test failures turn CI red.
- Several healer and notification defects break goals the plan states explicitly:
  - deferral expiry
  - "never swallow a failure"
  - quiet declines
  - idle ticks
  - files union
  - honest copy

This tale fixes all of it, then performs the landing closeout for `sase-1j6.10` and its
parent plan bead `sase-1j6`. Nothing resumes this landing after you finish. **The
closeout section at the end is part of this tale's required work.**

Before you start:

- Read the epic plans with
  `sase artifact read plan:202610/finish_update_skew_auto_restart.md "<why>"` and
  `sase artifact read plan:202610/update_skew_agent_auto_restart.md "<why>"`.
- Their DECISIONS are final, in particular `quiet_declines = quiet`.
- Read `docs/agent_auto_restart.md`.
- Read the `tui` memory before touching TUI code and `lint_and_test` before finishing.

No sase-core change is needed. The pinned core (`sase-core-revision.txt`) already
supports the reconcile `refresh_files` option (sase-core
`crates/sase_core/src/notifications/wire.rs`, `NotificationReconcileRequestWire`).

### Shared notification rule (used throughout)

- **Silenced row:** its runner dropped a doorbell or wrote `recovery.state = "pending"`.
- **Silenced terminal outcome:** made visible exactly once:
  - escalation copy where this plan or the parent plan specifies one;
  - otherwise `resurface_failure`.
- **Not-silenced terminal outcome:** quiet. It gets only the ledger record, the
  `done.json` `recovery` (when the row exists), and the dim Agents-tab hint.
- **Two exceptions stay loud regardless:**
  - `already_restarted` for a replacement's second failure;
  - the single storm escalation.

## 1. CI failures caused by the epic

1. **Timezone-dependent test.**
   `tests/test_agent_auto_restart_scan.py::test_collect_journal_updates_bundle_window`
   fails in CI (`TZ=UTC`) with `['…11:40:00-04:00'] == ['…11:53:00-04:00']`.
   - The test builds `dismissal` with naive `.astimezone()` (host zone).
   - `_collect_journal_updates` interprets `artifacts_timestamp` in its own zone.
   - Make the test independent of host TZ: build every instant in the same zone the
     function uses (check `sase.core.time` / the function's parser), or pin the zone
     with `monkeypatch.setenv("TZ", ...)` + `time.tzset()` and restore it.
   - Verify by running the test with `TZ=UTC` and with `TZ=America/New_York`.
   - Check the neighbouring `_collect_journal_updates` tests the same way.
2. **Temp-dir leaks.** CI's system-temp-leakage guard fails on `launched-*` and
   `failed-bead-*` dirs. Route each of these through pytest temp paths. Dir names must
   stay distinct where they act as dedup stamps.
   - `tests/test_agent_auto_restart_episode_polish.py`:
     - `_seed()` (`tempfile.mkdtemp(prefix=f"launched-{name}-{index}-")`);
     - the two `failed-bead-*` mkdtemps in the reused-name test.
   - `tests/test_agent_auto_restart_episode_notify.py`:
     - `tmp_path_launched_dir()`, used by `_seed_record`;
     - its direct use in the escalation-copy test.
   - Grep `tests/test_agent_auto_restart_*.py` and
     `tests/test_core_agent_auto_restart.py` for any other `tempfile.` use.

## 2. Healer state machine (`src/sase/agent/auto_restart/healer_claim.py`, `healer_flow.py`, `healer_relaunch.py`, `ledger.py`, `sweep.py`, `healer_targets.py`)

1. **Deferrals must expire.**
   - The bug:
     - The stale-deferred branch of `_recover_existing_claim` calls
       `take_over_ledger_claim` → core `reclaim`, which resets `claimed_at`.
     - The expiry check measures from `claimed_at`.
     - So a lineage re-deferred more often than `max_defer_seconds` (the sweep retries
       deferred keys every tick) never expires.
   - The fix: measure expiry from the lineage's **first** claim:
     - the earliest ledger `history[].at`;
     - or a `first_claimed_at` kept in the record's Python extras and preserved across
       takeovers;
     - fall back to `claimed_at`.
   - Test: a failing probe defers, repeated passes every 20 s with
     `max_defer_seconds=30` decline as `deferred_expired` once 30 s have passed since
     the first claim, and a silenced row escalates once.
2. **A dead `launching` claim must never adopt the failed row.**
   - The bug:
     - `_find_replacement_dir` accepts whatever `planned_name` resolves to if its dir
       mtime ≥ `claimed_at`.
     - Before the wipe, that is the failed row itself, whose mtime the `launching`
       recovery write just bumped.
     - So a healer that died before the wipe reports `relaunched`/`adopted` and swallows
       the failure.
   - The fix:
     - Reject any candidate whose resolved path equals the record's
       `failed_artifacts_dir`.
     - When the candidate's `agent_meta.json` carries `auto_restart` provenance, require
       its `ledger_key` to match.
   - Test: execute raises before the wipe; the next pass settles `settled_failed` with
     `launch_aborted` and a visible notice for a silenced row, not `adopted`.
3. **Orphaned ledger records must not keep the job busy.**
   - The bug: `_collect_job_work` marks `actionable` for every `deferred` key and every
     stale `claimed`/`launching` key, even when the failed row is gone (dismissed, wiped
     by execute, or older than the lookback). `run -p` only enumerates doorbells and
     failed rows, so it never resolves them, and the job submits a no-op healer proc on
     every tick.
   - The fix: have the healer pass (`run -p` / `resolve_pending_targets` plus
     `heal_one`) also process ledger-only work by key:
     - A stale `launching` record goes through the adopt-or-settle path (with fix 2.2),
       using the record's own fields when the failed row is gone.
     - A stale `claimed`/`deferred` record whose failed `done.json` no longer exists
       declines quietly. Use reason `row_gone`: no `done.json` write, no notification,
       and delete any doorbell for it.
   - After one healer pass over such records, the next `run_job_tick` must be `idle`.
     Add that test.
4. **Retakes keep their arguments.**
   - The bug: `_recover_existing_claim`'s stale-`claimed` and stale-`deferred` branches
     call `heal_claimed` without `derive_episode`, `silenced`, or the injected
     `classify`/`check_quiescence`/`run_probe`/`plan_restart`/`execute_restart` hooks.
     Tests that inject a classifier silently get the real one on the second pass, and
     `silenced` defaults to `True`.
   - The fix:
     - Thread every argument through.
     - Persist `silenced` in the ledger record's Python extras at first claim (the
       doorbell is deleted at claim, so it cannot be re-derived later).
     - Read it back on every later pass.
5. **One notification policy for every decline path.** Apply the shared rule above to:
   - the skip-rule branch at the top of `heal_claimed`, which currently always escalates
     or resurfaces (keep `already_restarted` loud);
   - the `deferred_expired` branch (always escalates today);
   - `execute_failed` in `relaunch_claimed`;
   - `_adopt_or_settle`'s `launch_aborted` (both always resurface today);
   - the re-check skip and `wipe_reaches_others` declines in `relaunch_claimed`. These
     never notify today, which leaves silenced failures invisible; they must also delete
     the doorbell.

   Prefer routing them all through `decline_heal` (or one shared helper). Add a test
   matrix: silenced vs not silenced × skip-rule, deferred-expired, execute-failed,
   launch-aborted, wipe-reaches-others.

6. **Small correctness fixes:**
   - `heal_one`'s candidate pre-check must treat an exception from the candidate rule as
     "not a candidate", matching enumeration in `healer_targets.py`.
   - Doorbell targets must resolve their project exactly as scan rows do, so one failed
     row can never map to two ledger keys. Today `project_for_done` falls back to
     `parents[2]`, while scan rows use the record's `project_name`.
   - `_disabled_resurface_stamp` must key on the full artifacts path (for example a
     short hash of it), not the dir name alone, so same-named rows in two projects don't
     collide.
   - `_settle_launched_records` must pass `at=timestamp_for(now)` to
     `advance_ledger_record`.
   - Remove the unreachable branch after the post-probe `defer` return in `heal_claimed`
     (the `verdict.mode != "relaunch"` block that `_terminal_verdict` already covers).
     First confirm it is unreachable.
7. **Idle cost** (unmet sweep-safety requirement).
   - `run_job_tick` calls `_settle_launched_records`, which reads every ledger record
     uncached on every tick.
   - Use the mtime-gated record cache instead, and only open `done.json` for `launched`
     records. First confirm that the cache invalidates on record rewrites (atomic
     replace changes the directory mtime).
   - Add the planned test: count filesystem calls (`os.stat`/`open`/`scandir`) for an
     idle `run_job_tick` that is not a full sweep, and bound them to a small constant.

## 3. Notifications and the episode report (`_notifications.py`, `_notify_report.py`, `_healer_common.py`, `storm.py`, `sweep.py`, `src/sase/core/notification_store_wire.py`, `src/sase/notifications/store.py`)

1. **Files union (unmet episode-polish requirement).**
   - `_refresh_row_content` merges `files`, but the reconcile request never sets
     `refresh_files`, so Rust drops the change.
   - Add `refresh_files: bool = False` to Python `NotificationReconcileRequestWire`.
     Serialize it **only when true**, so older cores never see an unknown field.
   - Add a `refresh_files` keyword to
     `sase.notifications.store.reconcile_notification_rows`.
   - Pass `refresh_files=True` from `_refresh_row_content`.
   - Test: two relaunches in one episode leave both agents' `error_report.md` and
     evidence files on the row, with timestamp and delivery cursors unchanged.
2. **Episode-row refresh must not clobber storm rows.**
   - `refresh_episode_rows` matches by the prefix `agent-auto-restart:{episode}`, which
     also matches storm keys (`…:{agent}:storm:{stamp}`) and rewrites their notes.
   - Match only the exact episode key or its rollover `#N` suffix (see
     `_resolve_dedup_key`).
   - Test: a settlement tick leaves a storm row's notes unchanged.
3. **Honest fallback copy.** Add one helper that returns `sase update 9fd8a08` for a
   known episode and `a sase update` otherwise, and use it in every "after …" phrase:
   - episode titles;
   - `publish_relaunch`;
   - the `already_restarted` escalation (it currently uses a raw `removeprefix`, so it
     can say "after sase update unknown" or "after sase update a sase update");
   - the storm detail.

   Other copy and key rules:
   - With no relaunched records, an episode title must not say "Restarted 0 agents";
     fall back to the agent being published.
   - A relaunch with no known episode must not create the dedup key
     `agent-auto-restart:unknown`. Use a key derived from the agent and its artifacts
     stamp, and keep `episode_id` consistent between the ledger record and the row (both
     `None` or both the same id).
   - Tests: no title, note, or key contains `unknown`, `sase@`, or
     `sase update a sase update`, including the no-episode path.

4. **Storm copy.** Match the parent plan exactly:
   `Auto-restart paused: N agents broke within one update — this looks like a real bug, not an update race.`
   - N counts the agents that broke: the already-launched records plus the current
     target.
   - Do not include the raw episode id in the title.
   - Apply the same fix to the 30-minute variant's wording.
5. **Settlement outside the tick stays live.** The report file and episode rows must be
   refreshed (same path as `_settle_launched_records`) when a record settles inside a
   healer pass:
   - `_handle_replacement_failure`;
   - `_adopt_or_settle`;
   - `execute_failed`.

   Make the report's Now cell (`_replacement_outcome`) consult the ledger state first:
   `settled_failed` reads FAILED and `settled_ok` reads DONE, even when the
   replacement's `done.json` is gone.

6. **One outcome set.** `_FAILED_REPLACEMENT_OUTCOMES` is duplicated in `sweep.py` and
   `_notify_report.py`. Keep one definition (for example in `constants.py`) and import
   it in both.
7. **The settlement test drives the tick.**
   `tests/test_agent_auto_restart_episode_polish.py`'s settlement test calls
   `refresh_episode_report` by hand. Drive `_settle_launched_records` / `run_job_tick`
   instead, and assert that the report file and the row snapshot settle with the cursor
   unchanged.

## 4. TUI, runner, and UX fixes

1. **Import budget** (`tests/ace/tui/test_app_import_budget.py`, cap `< 3513`).
   - `tools/tui_import_closure -d 70c51adbdd` shows these epic modules in the
     `import sase.ace.tui.app` closure:
     - `sase.agent.auto_restart`
     - `sase.agent.auto_restart.constants`
     - `sase.agent.auto_restart.ux`
     - `sase.axe.runner_lifecycle_phase`
   - Remove all four from the closure:
     - `src/sase/ace/tui/models/_loaders/_done_snapshot_loaders.py` and
       `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`: stop importing
       `ux` at module scope. Use use-site imports, or move the tiny `RESTARTING_*`
       constants to a module already in the closure.
     - `src/sase/ace/tui/widgets/update_accents.py` imports `UPDATE_RECOVERY_GLYPH` from
       `auto_restart.constants`. Give the glyph one definition in a leaf module already
       in the closure and importable by non-TUI code (for example next to the status
       glyphs in `sase.agent.status_buckets`), and make
       `sase.agent.auto_restart.constants` re-export it.
     - `src/sase/llm_provider/_invoke.py`: make the `runner_lifecycle_phase` import
       function-local. The runner already imports it at boot (`run_agent_runner.py`), so
       the runner path stays a `sys.modules` hit.
   - Re-measure. The expected count is 3515. The remaining +5 (`notification_gates.*`,
     `running_field._occupant`) belongs to `sase-1ic`. If the test is still red only for
     that reason, say so in the close note.
2. **Evidence `v` hint is broken.**
   - `ux.apply_provenance_to_agent` appends the bare evidence **directory** to
     `extra_files`. The default `v` view then fails with `[Errno 21] Is a directory`,
     and `error_report.md` is no longer reachable.
   - Register `<evidence_dir>/error_report.md` plus the bundle's individual files. Take
     the file names from the provenance record: have `healer_relaunch` record the
     bundle's file list in `agent_meta.auto_restart` if it does not already. Render
     paths must still never stat the bundle.
   - Test that no directory is registered.
3. **The refresh firewall test must really guard both late imports**
   (`tests/test_run_agent_runner_refresh_macros.py`). The subprocess:
   - must drop `PYTEST_VERSION` as well as `PYTEST_CURRENT_TEST`, so `state_write_guard`
     does not detect pytest;
   - must pass `local_macros`, so the `serialize_local_macros` →
     `get_sase_managed_tmpdir` → `sase.core.managed_tmp_roots` path runs.

   Confirm the test fails if the `managed_tmp_roots` warm-up is removed from
   `run_agent_runner_refresh.py`, then restore it.

4. **Glyph everywhere.** Replace each remaining literal `↻` that means update recovery
   with `UPDATE_RECOVERY_GLYPH`:
   - `auto_restart/_notifications.py`
   - `auto_restart/_notify_report.py`
   - `auto_restart/sweep.py`
   - `src/sase/agents/cli_auto_restart.py`
   - `src/sase/main/update_render.py`

   Leave unrelated retry/reveal glyphs alone. Don't pull new modules into the TUI
   closure (see 4.1).

5. **Nits:**
   - Fix the misattributed mutex-group comment in `tests/completion/test_build.py`: the
     21st group is the `sase agent auto-restart run` target group.
   - Fix the `src/sase/axe/runner_lifecycle_phase.py` module docstring. It claims
     function-local `sase.*` imports add "no deferred import surface", which the
     module's own function-local imports contradict.

## 5. Test coverage gaps

- **Non-skew rows** (`tests/test_agent_auto_restart_sweep_safety.py`): the "non-skew row
  untouched" tests use a day-old row, so the age check rejects it before the text
  prefilter runs. Add a **recent** provider-429 row and assert no ledger record, no
  `done.json` change, and no notification through both `run -p` and `run_job_tick`.
- **Probe mapping** (`tests/test_agent_auto_restart_probe_quiescence.py`): the
  flat-plugin and first src-layout tests pass a `target_module` equal to the expected
  answer, so they pass even if `_path_to_module` returns `None`. Assert the
  frame-derived module names independently.
- **Provenance** (`tests/test_agent_auto_restart_healer.py`): cover `_attach_provenance`
  filling `from_rev`/`to_rev` from W1 identity revisions and
  `culprit_commit`/`culprit_subject` from W3 `file_proof`, not only from the refresh log
  line.
- **Incident replay** (`tests/test_agent_auto_restart_incident_replay.py`):
  - Let the replacement's second failure happen while its record is still `launched`, so
    the real `launched → settled_failed` path runs, and assert that ledger state.
  - Count every notification row the replay creates (not only `sender == SENDER`), so
    "one episode row and one information toast" is really proven.
  - Keep the rule that no test in the replay calls `_escalate`, `escalate_healer`, or
    `refresh_episode_report` by hand.

## 6. Docs

Update `docs/agent_auto_restart.md`:

- Deferral expiry is measured from the lineage's first claim.
- Ledger-only orphan records resolve as `row_gone` / `launch_aborted`.
- The unified silenced-vs-quiet decline policy, with its two loud exceptions.
- The storm copy.
- The `v` hint lists the evidence files.

Remove the two claims the land review found false: "deferral does not loop forever" was
untrue before fix 2.1, and "not-silenced rows get no second resurface/escalation" was
untrue before fix 2.5. Keep them only if your fixes now make them true.

## 7. Verification

- Run the auto-restart suites:
  - `tests/test_agent_auto_restart_*.py`
  - `tests/test_core_agent_auto_restart.py`
  - `tests/test_agent_artifact_marker_mutation_audit.py`
  - `tests/test_agent_artifact_marker_path_passing_audit.py`
- Run them once with `TZ=UTC` and once with the host zone, and point `TMPDIR` at a
  scratch dir to confirm nothing leaks.
- Add any new `done.json` or marker read/write sites to the marker mutation and
  path-passing audit allowlists.
- Run `just fix` (or `just fmt`), then `sase tool run check`. Do **not** run
  `just check-full`. These failures are not this tale's; name each in the close note:
  - `sase-1by`: the `session_root_tab` path-passing audit entry.
  - `sase-1jc`: the `test_node_finder_snapshot.py` re-export ImportError and
    `test_contract_manifest_matches_marker_selection`.
  - `sase-1jj`: the bob highlights dry run.
  - `sase-1ic`: the non-epic import-budget remainder.
  - Known flakes `sase-1gh`, `sase-1jk`, `sase-1jl`.
- Treat any other failure as yours.

## 8. Closeout (required; do this in the same turn after verification passes)

### 8.1 Close epic `sase-1j6.10`

1. Run `sase bead epic-symbols sase-1j6.10`. Resolve every listed `--epic-symbol` entry:
   wire it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy (`sase memory read symvision.md -r "<why>"`). Re-key an entry
   only to a still-open bead that needs it. Do not add new `--epic-symbol` rows keyed to
   `sase-1j6.10` or `sase-1j6`.
2. Close the epic. The close note must name:
   - the fixes;
   - the tests run;
   - the `sase tool run check` id and the failures attributed elsewhere;
   - the TUI import count after 4.1.

   ```bash
   sase bead close sase-1j6.10 --note "<what you verified>"
   ```

   - If the close is rejected for leftover `--epic-symbol` entries, finish that cleanup
     and close again.
   - Never use `--force` merely to make the close succeed, and never use `--force` to
     advance a successful nested landing.

3. Run `just symvision` (through `sase tool run` if the recipe is guarded) and confirm
   it is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the `PLAN`
   path printed by `sase bead read sase-1j6.10 -r "Need the plan path for closeout"`
   (`plan:202610/finish_update_skew_auto_restart.md`, currently `status: wip`).

### 8.2 Land the parent plan bead `sase-1j6`

`sase-1j6.10`'s `parent_bead` is `sase-1j6`, a plan bead (tier epic) with no parent of
its own. Its land agent (`sase-1j6.land`) stopped and handed the remaining work to
`sase-1j6.10`.

1. Run `sase bead read sase-1j6 -r "Recheck the parent epic before closing it"`. Review:
   - its two landing notes;
   - every descendant (phases `sase-1j6.1`–`sase-1j6.9` and child epic `sase-1j6.10`,
     all of which must now be closed) and their notes;
   - its linked plan `plan:202610/update_skew_agent_auto_restart.md`, through
     `sase artifact read`.
2. Recheck the ten defects named in `sase-1j6`'s LAND VERIFICATION note. Each must now
   be fixed in source by `sase-1j6.10` and this tale.
3. Check post-child drift: commits on master since this tale's base. Confirm nothing
   newly conflicts with the feature.
4. If everything is complete:
   - Run `sase bead epic-symbols sase-1j6` and retire any leftover entries the same way.
   - Close the bead:

     ```bash
     sase bead close sase-1j6 --note "<what you rechecked>"
     ```

   - Run `just symvision` again.
   - Set `status: done` in `sase-1j6`'s plan file (its `PLAN` path from
     `sase bead read`).

   `sase-1j6` has no parent, so stop there.

5. If anything in `sase-1j6` is incomplete or ambiguous:
   - do not close it;
   - record a note on it describing the blocker
     (`sase bead note sase-1j6 "LAND BLOCKER: ..."`);
   - report the blocker in your final response.

### 8.3 Release follow-up to report (not this tale's action)

Master imports auto-restart bindings that the published `sase-core-rs` (0.37.2) lacks.
Publishing them needs the open sase-core release-plz PR #325 (v0.38.0), and merging it
is a human action. Mention this in the final response.
