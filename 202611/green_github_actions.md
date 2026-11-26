---
tier: epic
title: Green GitHub Actions for sase
goal: "Every red sase GitHub Actions workflow is green on the landed master tip or
  handed to a named owner with evidence, each traced to a root cause, with no raised
  timeouts, weakened assertions, or blindly regenerated goldens.

  "
decisions:
  core_cut:
    ask:
      Propose a gate that dispatches a sase-core release-plz cut to clear Publish's
      CORE_LAG?
    choices:
      gate:
        ci-proof proposes a /sase_gate for the core cut; nothing publishes until you
        approve
      leave: No core cut proposed; Publish and release PR
    default: gate
    why:
      the steward cannot cut core with merge authority off; the gate still needs your
      approval
    answer: gate
phases:
  - id: steward-wire
    title: Send Rust the exact release-asset wire shape
    depends_on: []
    size: small
    description: "steward-wire: drop the transport-only asset id from the release wire,
      read abandoned-observation asset ids from the raw release, fix the fixtures that
      encode the wrong shape, and add real-binding regression coverage.

      "
  - id: bench-bead-path
    title: Make bench_bead.py runnable as a script again
    depends_on: []
    size: xsmall
    description: "bench-bead-path: add the repo-root sys.path bootstrap to
      tests/perf/bench_bead.py and a static guard over every Justfile-invoked perf
      script.

      "
  - id: leak-gate
    title: Clear the four globals that fail the test-cost leak gate
    depends_on: []
    size: medium
    description: "leak-gate: fix the four leaking module globals at their root so the CI
      Telemetry global-leak blocking gate reports zero poisoning changes.

      "
  - id: master-gate-flakes
    title: Root-cause the flakes that turned recent master SHAs red
    depends_on: []
    size: medium
    description: "master-gate-flakes: reproduce and fix the four intermittent Master
      Gate failures at their source, or record evidence-backed follow-ups.

      "
  - id: visual-drift
    title: Rebaseline the November golden drift after proving each change is intended
    depends_on:
      - master-gate-flakes
    size: medium
    description: "visual-drift: regenerate the 19 drifted PNG goldens only after tracing
      each group to an intended change, fixing the retry-chain artifact loss if it is a
      regression, and leaving flickering goldens to their flake owners.

      "
  - id: ci-proof
    title: Prove the workflows green on the landed tip
    depends_on:
      - steward-wire
      - bench-bead-path
      - leak-gate
      - master-gate-flakes
      - visual-drift
    size: small
    description:
      "ci-proof: wait for Master Gate, dispatch and monitor Full CI and an observe-only
      steward tick, triage leftovers, and handle the core_cut decision."
proposed_by: bbugyi200.athena.1ge
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 06:30:31
status: wip
---

- **PROMPT:**
  [prompts/202611/green_github_actions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/green_github_actions.md)

# Green GitHub Actions for sase

## Goal

Every red GitHub Actions workflow on `sase-org/sase` master either goes green or is
handed to a named owner with evidence. Each failure is traced to a root cause. No
timeouts are raised, no assertions weakened, and no goldens copied blindly from CI.

## Diagnosis (from `actstat` and the run logs, 2026-11-26)

Governing decisions: `decisions:ci-two-speed-split` (Master Gate is the per-SHA gate;
Full CI is the scheduled exhaustive lane and a release prerequisite),
`decisions:rust-core-required` (the Rust `*Wire` contract is canonical; the Python
adapter must produce exactly that shape), and `decisions:check-full-is-explicit` (this
plan explicitly instructs `just fix-tui-screenshots` for the visual phase, and never
`just check-full`).

| Workflow                                                       | Symptom                                                                                                                                                                                                                                                                                                                                                                                                    | Root cause                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Status                                                   |
| -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Release steward (every run since 2026-11-12 20:42Z)            | `watcher_broken`: `invalid_release_steward_request: ... unknown field "id", expected one of "name", "kind", "generation"`                                                                                                                                                                                                                                                                                  | `tools/_release_watch_facts.py::_release_wire` adds a transport-only `"id"` to every release asset. Rust `AssetFactsWire` (`deny_unknown_fields`) has only `name/kind/generation` and never had `id`. The bug was latent until the steward's Nov 12 `dispatch_release` made `release.yml` upload the first-ever assets to the v0.19.0 draft release. The test fixtures copied the same wrong shape, and the real-binding round-trip test uses `releases: ready([])`, so CI never exercised it.       | Deterministic. Phase `steward-wire`                      |
| Full CI `perf-floors` (every run)                              | `just bead-perf-smoke`: `ModuleNotFoundError: No module named 'tests'`                                                                                                                                                                                                                                                                                                                                     | `5b79f0f5e3` (2026-11-19) added `from tests.test_bead.legacy_sqlite import db` to `tests/perf/bench_bead.py`. The Justfile runs it as a script (`python tests/perf/bench_bead.py`), so only `tests/perf/` is on `sys.path`. Sibling scripts (`bench_bead_scale.py`, `bench_prompt_search.py`, `bench_core_query.py`) insert the repo root first; this one does not. Reproduced locally.                                                                                                              | Deterministic. Phase `bench-bead-path`                   |
| Full CI `visual-test` (every run)                              | `just fix-tui-screenshots --check` reports reproducible drift: 19 goldens on `8e6818d` (run 45728775430), growing since at least 2026-11-14 (3 → 4 → 8 → 9 → 11 → 12 → 17 → 18 → 19)                                                                                                                                                                                                                       | UI commits landed without regenerating every golden they affect (`just check` never runs PNG snapshots). At least one drift may be a product regression rather than an intended change: `agents_retry_e2e_running_fallback_120x40` now lists 2 artifact files in the fallback agent's SASE CONTEXT where the golden lists 4 (the two `db500a0f…` files are gone). Other groups look intentional, e.g. the Artifacts sub-tab strip moved in the `plan_toast_*` and `startup_update_toast_*` captures. | Phase `visual-drift`                                     |
| Master Gate (single-test flakes on recent SHAs)                | `32b0d45`: `test_agent_archive_column_app.py::test_archive_replaces_the_inbox_column_and_keys_navigate_it`; `8e6818d`: `test_link_follow_planner.py::test_epic_context_selects_and_expands`; `e489558`: `test_refresh_warmup_gc.py::test_auto_refresh_merge_live_hints_and_warmup_release_obsolete_payloads`; `ac3f1d2`: `test_agents_tab_delta_integration.py::test_scoped_prep_is_materially_cheaper[8]` | All pass locally on current master. They fail intermittently on the sharded gate, and each made a master SHA red.                                                                                                                                                                                                                                                                                                                                                                                    | Phase `master-gate-flakes`                               |
| Master Gate / Full CI (already fixed)                          | `tests/completion/test_snapshot.py` drift and `tests/test_render_design_decisions.py` (`decisions:rosters-load-on-demand` link)                                                                                                                                                                                                                                                                            | Fixed on master by `32b0d45` and `ac3f1d2`; verified locally (both files pass).                                                                                                                                                                                                                                                                                                                                                                                                                      | No work. `ci-proof` confirms                             |
| CI Telemetry `test-cost` (every run)                           | `sase global leak detector blocking gate failed`: 41-47 poisoning tests, while all ~63.5k tests pass                                                                                                                                                                                                                                                                                                       | Only four module globals: `sase.config.local_trust_enforcement_flag_on._warned_signatures` (30 tests, mostly `tests/test_config_local_trust_hook_enforcement.py`), `sase.ace.tui.models._projected_record_detail._detail_owner` (9), `sase.config.core._live_config_token_refresh_threads` (7, deck/widget pilot tests), `sase.axe._state_chops._SKIP_STREAKS` (1, `tests/test_axe_routine_tick_metrics.py`). Evidence: artifact `sase-global-leaks-e489558…` of run 45709304226.                    | Deterministic. Phase `leak-gate`                         |
| Publish `sync-release-metadata` (every run since ≥ 2026-11-15) | `CORE_LAG(pin=71944fb…, newest_complete=0.44.2)`, exit 3                                                                                                                                                                                                                                                                                                                                                   | No published `sase-core-rs` release contains master's `sase-core-revision.txt` pin; 0.44.2 (2026-11-09) is the newest. The steward would propose `dispatch_core_cut`, but the Rust policy suppresses it while merge authority is off, and the steward has been `watcher_broken` anyway. Release PR #327's `release-core-floor-smoke` fails for the same reason: 0.44.2 lacks bindings current sase calls.                                                                                            | Operational. Decision `core_cut`, executed in `ci-proof` |
| CI Telemetry `coverage-contexts`, `contention-test`            | A different set of timing-sensitive failures each run (e.g. `test_known_continuation`, `test_countdown_tick_gc`, `test_agents_panel_focus_follows_selection`), plus soak cancellations                                                                                                                                                                                                                     | Load and coverage-overhead flakes with no single cause. The contention soak is already owned by `sase-11b`.                                                                                                                                                                                                                                                                                                                                                                                          | Out of scope (see below)                                 |

## Related active work (do not duplicate; the lander records corroboration)

- `sase-1q2` (ship v0.19.0 through the release steward) and its release predecessors:
  `steward-wire` removes the steward's `watcher_broken` blocker.
- `sase-1jz` (green Master Gate and Full CI with host-independent TUI goldens): its
  phases target October drift. The November drift handled by `visual-drift` is new.
  `agents_session_panel_turns_gate_*` and `agents_final_rows_80x40` (flake bead
  `sase-1nx`) are already associated with that epic.
- `sase-j7` (process-global state leaking between tests) owns the leak class that
  `leak-gate` touches; `sase-t6` concerns `_live_config_token_refresh_threads`
  snapshots.
- `sase-11b` owns right-sizing the contention soak.

## Phases

### Phase `steward-wire`: Send Rust the exact release-asset wire shape

- **Size:** small
- **Depends on:** none

The steward request must match Rust's `ReleaseStewardRequestWire` exactly. GitHub asset
ids are Python-side transport detail and stay out of the wire.

1. In `tools/_release_watch_facts.py::_release_wire`, stop adding `"id"` to each asset
   dict. Keep validating the raw asset id there if you want early transport errors, but
   do not emit it.
2. `_fresh_train_abandoned(tag, tag_fact, release_wire)` is the only reader of a wire
   asset `id` (around line 1544). Change it to take the raw release record
   (`release_raw`, already unpacked beside `release_wire` in `_train_context`). Iterate
   the raw `assets` with the existing `_list` / `_mapping` / `_required_str` /
   `_integer` helpers, match names with `_OBSERVATION_RE`, and read the id from the raw
   row. Update the call site (around line 1848) and its `release_wire is not None`
   guard. Re-grep `tools/_release_watch*.py` for any other `asset.get("id")` /
   `asset["id"]` read on wire assets. `_release_watch_actions.py` re-reads raw releases
   through `_release_for_tag` and is unaffected.
3. Fix the fixtures that encode the wrong wire shape: `_asset_rows` and `extra_assets`
   in `tests/_release_watch_facts_contract_cases.py`, and `wire_assets` in
   `tests/_release_watch_train_start_helpers.py`. Wire assets lose `id`; raw rows keep
   it.
4. Add regression coverage that would have caught this:
   - A unit test that `_release_wire` emits assets with exactly the keys
     `{"name", "kind", "generation"}`. Use a raw release carrying a
     `candidate-manifest.json`, a `candidate-gen1-qualification-run…-attempt1.json`
     observation, and an unrelated asset.
   - A real-binding test next to `test_real_binding_round_trip_when_installed` in
     `tests/test_release_watch_decide.py`. Use the same `importorskip` / missing-binding
     skip guards. Take `releasable_request()` and replace `snapshot.releases` with a
     ready list holding a `_release_wire(...)` output for a draft release with assets.
     Assert `release_steward_decide` returns a result instead of raising `ValueError`.
     Building the wire through `_release_wire`, not a hand-written dict, is the point.
5. Verify: run the `tests/*release_watch*` and
   `tests/test_github_actions_release_steward.py` suites, then `just check`. If `gh` is
   authenticated, also run the observe-only tick
   `.venv/bin/python tools/release_watch --tick --json` (never `--live`). It must no
   longer report `invalid_release_steward_request`. Any other primary state is
   information for `ci-proof`, not a failure of this phase.

### Phase `bench-bead-path`: Make bench_bead.py runnable as a script again

- **Size:** xsmall
- **Depends on:** none

1. At the top of `tests/perf/bench_bead.py`, before any `tests.` import, add the same
   repo-root `sys.path` bootstrap used by `tests/perf/bench_prompt_search.py`
   (`Path(__file__).resolve().parents[2]`, inserted only if absent, with `# noqa: E402`
   on the imports after it).
2. Guard the whole class. Add (or extend, if one exists) a cheap static test. For every
   `tests/perf/*.py` that the `Justfile` runs by path
   (`{{ venv_bin }}/python tests/perf/<name>.py`) and that has a top-level `tests.`
   import, it parses the module's AST and asserts a repo-root `sys.path.insert` comes
   before that import. Today only `bench_bead.py` would fail it.
3. Verify: `just bead-perf-smoke` succeeds locally (it is the exact Full CI step that
   failed). Run the new guard test, then `just check`.

### Phase `leak-gate`: Clear the four globals that fail the test-cost leak gate

- **Size:** medium
- **Depends on:** none

The CI Telemetry `test-cost` job fails only on the global-leak blocking gate. Fix each
leak at its root instead of exempting the name from the detector:

- `sase.config.local_trust_enforcement_flag_on._warned_signatures` (30 tests): a
  warn-once registry. Give tests a supported reset (e.g. a public reset helper the
  suite's isolation fixture calls, or an autouse fixture scoped to the module's tests),
  so no test inherits another's warned state.
- `sase.ace.tui.models._projected_record_detail._detail_owner` (9 tests, e.g.
  `tests/test_agent_group_revival_e2e.py`): a weakref owner map left pointing at a live
  `Agent` after the app closes. Find the lifecycle path that should invalidate ownership
  on teardown (see `_invalidate_agents_detail_ownership` in
  `src/sase/ace/tui/actions/agents/_archive_view_lifecycle.py`). Fix it in product code
  if app shutdown skips it, or in the test harness if only tests bypass shutdown.
- `sase.config.core._live_config_token_refresh_threads` (7 deck/widget pilot tests):
  those tests leave a config-token refresh thread running. Stop it deterministically at
  app or test teardown. Check `sase-t6` and `tui_perf.md` guidance before touching
  refresh threads.
- `sase.axe._state_chops._SKIP_STREAKS` (1 test): reset or isolate it in that test's
  scope.

Find how the `test-cost` recipe enables the detector and its blocking gate (Justfile
plus `tests/_global_state_leaks/`). Reproduce on the named test files only, never the
two-hour full lane. Prove zero poisoning changes for these files after the fix. Run
`just check`. Record remaining leaks you could not attribute as `PROPOSED FOLLOW-UP:`
notes on this phase bead.

### Phase `master-gate-flakes`: Root-cause the flakes that turned recent master SHAs red

- **Size:** medium
- **Depends on:** none

Targets, with evidence runs:

- `tests/ace/tui/test_agent_archive_column_app.py::test_archive_replaces_the_inbox_column_and_keys_navigate_it`
  (`assert None is ArchiveList(...)`, Master Gate run 45744253917, shard 5)
- `tests/ace/tui/test_link_follow_planner.py::test_epic_context_selects_and_expands`
  (`'id:alpha-open' == 'id:alpha-1 id:alpha-1.*'`, run 45719508101, shard 3)
- `tests/ace/tui/test_refresh_warmup_gc.py::test_auto_refresh_merge_live_hints_and_warmup_release_obsolete_payloads`
  (leftover `...WorkerResult` payload, run 45708567744, shard 2; also CI Telemetry)
- `tests/test_agents_tab_delta_integration.py::test_scoped_prep_is_materially_cheaper[8]`
  (wall-clock ratio `560.8 < 481.2 * 0.85`, run 45731715520, shard 2)

For each:

1. Check for an existing owner first (`sase bead list -T flake`, plus related epics such
   as `sase-w3` link-follow reliability and `sase-1ky` archive banner repaint).
2. Reproduce under load: the shard's neighbors, `-n` parallelism, repeated runs.
3. Fix the race or ordering dependency at its source, e.g. wait on the semantic state
   the assertion needs, or remove a shared-state dependency. Do not raise timeouts, add
   retries, or loosen assertions. For the wall-clock ratio test, prefer measuring a
   host-independent quantity (operation or call counts the scoped path avoids) or a
   robust statistic over repeated samples. The test must still fail a real regression
   where scoped prep is no cheaper.
4. If one cannot be root-caused in this phase, leave it untouched and record a
   `PROPOSED FOLLOW-UP:` note with the evidence instead of a speculative change.

`tests/tool/test_admission_processes.py::test_nested_agent_check_keeps_parent_and_never_queues`
and `tests/test_config_cache_token.py::test_getter_never_starts_thread_after_warmup`
also failed in Full CI on PR #319 and in CI Telemetry. Fix them only if they share a
root cause with a target above; otherwise record them as proposed follow-ups. Finish
with `just check`.

### Phase `visual-drift`: Rebaseline the November golden drift after proving each change is intended

- **Size:** medium
- **Depends on:** master-gate-flakes

This phase is explicitly instructed to run `just fix-tui-screenshots`. Read
`tui_screenshot.md` and the PNG section of `lint_and_test.md` first. Run full visual
commands through `/sase_monitor` (`TESTING` / `TESTED`) with a long timeout.

Drifted goldens in Full CI run 45728775430 (artifacts `visual-failure-report.html` and
`ace-visual-first-attempt-diffs`): `agents_clan_panel_needs_attention_120x40`,
`agents_final_rows_80x40`, `agents_fleet_remote_tribe_sessions_120x40`,
`agents_retry_e2e_completed_chain_120x40`, `agents_retry_e2e_running_fallback_120x40`,
`agents_running_clan_runtime_collapsed_120x40`, `agents_session_gate_output_120x40`,
`agents_session_panel_needs_attention_160x50`, `agents_session_panel_turns_gate_120x40`,
`agents_session_panel_turns_gate_90x40`, `agents_stranded_blocked_detail_120x40`,
`agents_stranded_detail_header_120x40`, `input_collection_modal_120x40`,
`input_collection_modal_error_120x40`, `plan_toast_epic_120x40`,
`plan_toast_tale_120x40`, `plan_toast_tale_decisions_120x40`,
`startup_update_toast_120x40`, `startup_update_toast_grouped_commits_120x40`.

1. On the integrated tip, run `just fix-tui-screenshots`. Inspect the report: every
   update group's representative and members, plus every creation or removal.
2. Trace each group to the commit that changed its rendering (`git log` / `git log -S`
   over the source behind the changed region since 2026-11-12) and confirm the change
   was intended.
3. `agents_retry_e2e_running_fallback_120x40` and
   `agents_retry_e2e_completed_chain_120x40` need a decision. The fallback agent's SASE
   CONTEXT went from 4 artifact files to 2 (the `db500a0f…` files vanished). Find the
   responsible change. If it intentionally scoped artifacts per attempt, accept the
   golden. If it is a regression (artifacts dropped from the retry/fallback chain), fix
   the product code with a non-visual regression test, then regenerate.
4. Some goldens flicker between Full CI runs instead of drifting steadily:
   `agents_session_panel_turns_gate_90x40`, `agents_final_rows_80x40` (`sase-1nx`),
   `agents_fleet_remote_tribe_sessions_120x40`,
   `agents_session_panel_needs_attention_160x50`,
   `agents_clan_panel_needs_attention_120x40`. Accept an update for one only if repeated
   targeted `just fix-tui-screenshots --check -- <selector>` runs reproduce it exactly.
   Otherwise restore the golden and record a `PROPOSED FOLLOW-UP:` naming the owning
   flake bead or epic.
5. Finish with a clean `just fix-tui-screenshots --check` (or `just test-visual`) and
   `just check`.

### Phase `ci-proof`: Prove the workflows green on the landed tip

- **Size:** small
- **Depends on:** steward-wire, bench-bead-path, leak-gate, master-gate-flakes,
  visual-drift

1. Wait (through `/sase_monitor`) for Master Gate on the tip that contains every phase.
   Then dispatch and monitor Full CI (`gh workflow run full.yml`).
2. Dispatch the steward in observe-only mode
   (`gh workflow run release-steward.yml -f live=false`) and confirm the tick no longer
   reports `watcher_broken` for an invalid request. Record its primary state.
3. Triage any remaining red job against existing owners and this plan's proposed
   follow-ups. Record same-SHA run ids as evidence in this phase's notes.

> [!decision] core_cut = gate
>
> If sase-core master CI is green, propose a `/sase_gate` that runs
> `gh workflow run release-plz.yml -R sase-org/sase-core -f dry_run=false`. Explain that
> it publishes a `sase-core-rs` release containing pin `sase-core-revision.txt`, which
> clears Publish's `CORE_LAG` and release PR #327's `release-core-floor-smoke`. Do not
> dispatch it any other way.

> [!decision] core_cut = leave
>
> Do not propose a core cut. Report in the final notes that Publish and release PR #327
> stay red until a sase-core release containing the pin is published.

## Out of scope

- CI Telemetry `coverage-contexts` and `contention-test`: a different set of
  timing-sensitive failures every run under coverage or soak load. The contention soak
  is owned by `sase-11b`. Fixing them one at a time needs its own triage epic.
- Editing `sase-core`: the Rust wire is correct; only the Python adapter was wrong.
- PR CI failures on open PRs (#319 core-pin ratchet, #327 release PR): they inherit
  master's failures above and are not fixed separately.
