---
tier: tale
title: Green Master Gate and Full CI so ci_watch releases sase v0.17.2
goal:
  "Master Gate is green on the master tip and Full CI can conclude green, so the
  ci_watch chop merges release PR #299 and the Publish workflow ships sase v0.17.2 to
  PyPI."
size: medium
proposed_by: bbugyi200.apollo.02
create_time: 2026-09-15 10:07:51
status: wip
---

# Green Master Gate + Full CI so ci_watch releases sase v0.17.2

## Problem

GitHub Actions for `sase-org/sase` is red on two independent fronts, and both block the
v0.17.2 PyPI release. The `ci_watch` chop (athena) merges the already-open
release-please PR **#299** ("chore(master): release 0.17.2") only when **both** hold:

1. `gating_workflows`: the master tip's **Master Gate** run is green.
2. `heavy_max_age_hours: 6`: a **Full CI** run newer than 6 hours concluded green.

After #299 merges, the scheduled Publish workflow (cron `17 */3 * * *`) tags and
publishes v0.17.2 to PyPI. No manual commit/merge/publish is ever performed by an agent
(completion is host-owned).

### Front 1 — Master Gate red on every push since `3d32832d0` (diagnosed, certain)

CI builds the `sase_core_rs` wheel from the git SHA pinned in `sase-core-revision.txt`
(currently `3566872b4916123fedf100b7c5684c701085655c`, = sase-core v0.34.28). Local dev
instead builds the Rust core from the linked sase-core checkout at HEAD, which is why
these commits passed locally but redden CI:

- `b2a10778e` (fix(managed-tmp)) requires managed-tmp reap wire schema **v3**
  (`src/sase/core/managed_tmp_reaper.py` raises `RuntimeError: ... expected 3, got 2` —
  ~40 tests across 6 shards).
- `3d32832d0` (fix(pager)) requires the Rust binding to raise `ValueError` for
  `~`/`~/...` document source paths (sase-core commit `0de08a6`) — 6 more failures
  (`tests/pager/test_resolve_paths.py`, `tests/pager/test_copy_owned.py`,
  `tests/pager/test_rendered_link_contract.py`,
  `tests/artifact_refs/test_document_source_resolution.py`, and
  `tests/test_run_agent_runner_scratch_cleanup.py`).

Both behaviors exist at sase-core HEAD **`c5992b1cf363d77b4f952dd96042af9de500dd2e`**
(release v0.34.33; wire v3 came in `c657ee5`, first released in v0.34.31; the sudo
bindings used by `7a1a1ca3e` also exist there). The pin was never ratcheted because the
**Core Pin Ratchet** workflow (`.github/workflows/core-pin-ratchet.yml`) is itself
broken: apply-mode `tools/ratchet_core_revision` exits **2 by documented contract**
("pending, report-only, or was applied"), and the workflow runs it bare under
`set -euo pipefail`, so the step dies after "applied" and before
`git push`/`gh pr create` (see run 34967060505: "... applied" then "Process completed
with exit code 2"). Contrast: publish.yml's window-ratchet step handles exit 2
correctly.

### Front 2 — Full CI red persistently (all four jobs diagnosed)

Both 2026-09-15 scheduled runs (34930887719, 34965385668) failed the same four jobs on
the same head SHA `7420b8298`:

- **visual-test** — the Sep 14 golden rebaseline commit `d0a849df7` committed 4 goldens
  CI can never reproduce:
  - `agents_retry_e2e_countdown_120x40`, `agents_retry_e2e_running_fallback_120x40`,
    `agents_retry_e2e_completed_chain_120x40`: the agent detail panel now renders an
    ARTIFACTS section listing **raw absolute artifact paths** (regen machine's pytest
    tmp root vs CI's) and `agent-delta-<ts>-<id>-<12hex>.json` filenames whose 12-hex
    suffix is **random per run** (hence pixel counts varying 2199–2307 etc. across
    runs). `FakeyRetryHarness.normalize_visual_timestamps`
    (`tests/fakey/harness.py:294`) normalizes epochs/PIDs/repo-root but **not** artifact
    file paths or delta-filename hashes. The countdown itself is frozen via
    monkeypatched `time.time` — timing is not the cause.
  - `config_center_logs_tab_toasts_120x40`: golden captured a stray "Agent Load Timing
    294B · now" log source leaked from the regen machine's state ("6 active" vs CI's
    deterministic "5 active"/empty — identical 2625-pixel diff in both runs).
- **perf-floors** — the "Phase 7E regression floor" step (`just phase7-perf-check` →
  `tests/perf/phase7_check_regression.py`, ceilings in
  `tests/perf/baselines/phase7_regression_floor.json`) fails hairline, on different
  anchors per run (0.05–0.16% over). Margins were consumed by the sase-core
  0.32.61→0.34.0 bump (notification-store 5k load median jumped ~21%, 15.1ms→18.3ms, in
  the 09-10→09-11 window whose only relevant change is the core wheel) and Sep 12–13
  agent-scan commits (`3c1185c28`, `93f3d5891`, `5beda061f`; scan_facade 243–245ms → up
  to 285ms observed). Observed evidence:
  - `notification_store.synthetic_5k.notification_store_5k_load_snapshot`: ceiling
    18,466.00us (1.40x × 13,190us); observed up to 21,760us.
  - `scan_agent_artifacts.synthetic_6p_200pp.scan_facade`: ceiling 281,973.47us (2.35x ×
    119,988.71us); observed up to 285,100us.
  - `evaluate_query_many.synthetic_1000_specs.persistent_query_keystroke`: ceiling
    193.44us (2.90x × 66.70us); observed 193.54us.
- **test (3.13)** — not a hang: the 3.13 leg runs the heavier `test-cost` lane
  (Justfile) and now needs ~2h against the matrix's `timeout-minutes: 90`
  (`.github/workflows/ci.yml`, `test` job). It passed at 1h13m on 08-30 and timed out
  **on the same SHA** the next scheduled run; every run since dies at exactly 90m
  (~73–74% of the suite). The 3.12 leg finishes in ~54m, 3.14 in ~34m.
- **contention-test** — has **never passed** (born red when the scheduled lane was
  created on 08-27; skipped on `workflow_dispatch` via
  `if: github.event_name == 'schedule'`). It runs the full 41.7k-item suite ×3 repeats
  with 26 workers pinned to 2 CPUs (`taskset -c 0,1`,
  `SASE_CONTENTION_REPEAT=3 just test-contention`): collection alone takes ~31m and the
  hosted runner receives a shutdown signal ~37m in; even without the kill, ~2.7h cannot
  fit the 90m timeout. The Justfile documents this lane as "an opt-in diagnostic ...
  deliberately unreachable from `just check`".

## Changes

All paths are repo-relative in the sase repo checkout. Do not modify sase-core; the
needed core changes are already released.

### 1. Bump the pinned sase-core revision

Replace the contents of `sase-core-revision.txt` with:

```
c5992b1cf363d77b4f952dd96042af9de500dd2e
```

(sase-core master HEAD, release v0.34.33.)

### 2. Fix the Core Pin Ratchet workflow's apply step

In `.github/workflows/core-pin-ratchet.yml`, the bare apply invocation
`python3 tools/ratchet_core_revision` must tolerate the tool's documented exit code 2
(applied), mirroring publish.yml's window-ratchet pattern, e.g.:

```bash
apply_status=0
python3 tools/ratchet_core_revision || apply_status=$?
if [ "$apply_status" -ne 0 ] && [ "$apply_status" -ne 2 ]; then
  exit "$apply_status"
fi
```

Keep every string the contract test
`tests/test_github_actions_ci_master_gate.py::test_core_pin_ratchet_uses_the_shared_tool_and_names_the_pin_file`
asserts (`tools/ratchet_core_revision --check`, `tools/ratchet_core_revision`,
`sase-core-revision.txt`, `gh pr create`). Extend that test (or add a sibling contract
test) to pin the regression: the apply-mode invocation must capture its exit status
rather than run bare under `set -e` (e.g. assert
`"python3 tools/ratchet_core_revision || apply_status=$?" in run_text`).

### 3. Ratchet the PyPI core floor and lockfile on master

Run `just ratchet-core-window` (tool exits 2 when it applies — that is success) then
`uv lock`. This bumps the `sase-core-rs` floor in `pyproject.toml` (currently
`>=0.34.28,<0.35.0`) to the newest complete PyPI release (expected 0.34.33) and
refreshes `uv.lock` (currently locking 0.34.28). Without this, master's declared floor
cannot actually run the code (wire v3 needs ≥0.34.31), and release PR #299's
`release-core-floor-smoke` job — which installs sase at the exact declared floor — would
stay red. Commit whatever floor the tool selects; do not hand-pick a version.

### 4. Make the four visual goldens reproducible, then regenerate them

1. In `tests/fakey/harness.py` (`FakeyRetryHarness.normalize_visual_timestamps`, ~line
   294): extend normalization so the agent-detail ARTIFACTS lane renders
   machine-independent text — rewrite artifact file absolute paths (pytest tmp roots) to
   a stable placeholder using the same pattern as the existing
   `repo_root → /workspace/sase` rewrite, and normalize the random trailing 12-hex
   suffix in `agent-delta-<timestamp>-<id>-<12hex>.json` filenames to a fixed token.
   Keep the normalization in the harness (test-support), not in product code
   (`src/sase/ace/tui/widgets/prompt_panel/_agent_artifacts_lane.py` renders verbatim
   and should keep doing so).
2. In `tests/ace/tui/visual/test_ace_png_snapshots_config_center_logs.py`: pin the
   Logs-tab source state explicitly so a stray host-state log (e.g. a leftover Agent
   Load Timing log in the test SASE_HOME) cannot leak into a future regen; the snapshot
   must render identically on a fresh machine and in CI.
3. Regenerate exactly the four affected goldens with the pinned renderer env:
   `just install-visual`, then targeted
   `just test-visual tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py -- --sase-update-visual-snapshots`
   and the config-center-logs equivalent. The update fixture refuses to run outside the
   pinned renderer environment on canonical Linux; if this machine's fingerprint is
   refused, perform the regen on a canonical Linux host instead — do not loosen the
   fingerprint check or the zero-tolerance diff defaults in
   `tests/ace/tui/visual/png_diff.py`.
4. Run the full `just test-visual` suite and require it green before finishing.

### 5. Recalibrate the three hovering Phase 7E anchors

In `tests/perf/baselines/phase7_regression_floor.json`, following the file's own
documented recalibration precedent (each override carries a note citing its evidence),
set per-anchor ceilings with headroom above the worst observed CI values:

- `notification_store.synthetic_5k.notification_store_5k_load_snapshot`: per-anchor
  factor ~1.75x (≈23.1ms ceiling; worst observed 21.76ms), note citing the sase-core
  0.34 step (+21% load-median between runs 34479600173 and 34594086333).
- `scan_agent_artifacts.synthetic_6p_200pp.scan_facade`: 2.35x → ~2.50x (≈300ms; worst
  observed 285.1ms), note citing the Sep 12–13 agent-scan commits.
- `evaluate_query_many.synthetic_1000_specs.persistent_query_keystroke`: 2.90x → ~3.05x
  (≈203us; observed 193.54us vs 193.44us ceiling).

Keep the global 1.40x factor unchanged. Do not delete anchors.

### 6. Raise the ci.yml test-matrix timeout

In `.github/workflows/ci.yml` `test` job: `timeout-minutes: 90` → `150` (the 3.13
`test-cost` leg extrapolates to ~2h). Update
`tests/test_github_actions_ci_workflow.py::test_test_job_timeout_allows_slow_3_12_leg`
(assertion `== 90`) and rename/redocument it to reflect that the 3.13 cost leg is the
slowest.

### 7. Take the contention soak out of the gating path

In `.github/workflows/ci.yml` `contention-test` job: add `continue-on-error: true` with
a comment stating it is an observed-only diagnostic soak (per the Justfile's own
description) that has never fit a hosted 2-CPU runner, so it must not veto the heavy
lane's green that `ci_watch` gates releases on. Keep the job, its schedule `if:`, and
its artifact upload. The heavy-lane contract test
(`test_heavy_lane_jobs_are_defined_once_in_the_reusable_workflow`) only asserts the job
exists in ci.yml and is unaffected; still, grep the workflow contract tests for new
assertions worth adding (a small contract test pinning `continue-on-error` on this job
is welcome).

### 8. File follow-up task beads

Use the `/sase_new_task` skill for each (do not skip the dedup/triage flow):

1. sase-core 0.34.x made `notification_store_5k_load_snapshot` ~21% slower — investigate
   in sase-core whether that cost is intentional/reducible.
2. `artifacts_stitches_persistent_filter_80x24` PNG snapshot fails intermittently with
   an identical 146865-pixel diff (bimodal, order-dependent state pollution under xdist;
   first red 09-03 run 33757602516, most recently red 09-14 run 34907572118; not covered
   by the 09-14 rebaseline). It can re-redden Full CI at any tick.
3. Re-budget or shard the 3.13 `test-cost` lane (~2h and growing) instead of leaning on
   the raised 150m timeout.
4. Right-size the contention soak (curated flake-owning subset and/or
   `SASE_CONTENTION_REPEAT=1`, or a larger runner) so it can become gating again, or
   delete it from the scheduled lane deliberately.

## Verification

1. `just install` first (the workspace venv builds `sase_core_rs` from the linked
   sase-core checkout; make sure that checkout is at
   `c5992b1cf363d77b4f952dd96042af9de500dd2e` — open it with the `/sase_repo` skill,
   which fast-forwards it).
2. `just check` for the whole-repo lint gates + scoped lane.
3. This change broadens the effective test surface (core pin bump), so also run
   `just check-full` — **only** through the `/sase_monitor` skill (`TESTING` / `TESTED`
   status pair), never inline.
4. `just test-visual` fully green (see change 4).
5. Contract tests to re-run explicitly if the scoped lane does not select them:
   `tests/test_github_actions_ci_master_gate.py`,
   `tests/test_github_actions_ci_workflow.py`, `tests/test_contract_manifest.py`.

## Landing and release path (context, not agent actions)

Land through the normal host-owned completion flow (no manual git/gh mutations).
Afterward the pipeline is automatic: Master Gate runs per-SHA on the fix commit; Full CI
runs on the next 2-hour cron tick (`17 */2 * * *`) — a maintainer may also
`workflow_dispatch` Full CI to refresh the heavy signal sooner (dispatch runs skip
contention-test by design); once the tip's Master Gate is green and a green Full CI run
is under 6 hours old, `ci_watch` (athena, 5-minute ticks, `merge_enabled: true`) merges
release PR #299, and the next Publish cron run (`17 */3 * * *`) publishes v0.17.2 to
PyPI. The publish-time "Reconcile release metadata" step will find the floor already
ratcheted by change 3 and no-op.

## Risks / contingencies

- The 0.34.28→0.34.33 core bump may shift Phase 7E anchors again in CI; the new ceilings
  carry 5–8% headroom above worst observed values, and any further hairline trip should
  be fixed by nudging that one anchor per the same documented precedent — not by raising
  the global factor.
- If the visual-update fixture refuses this machine's renderer fingerprint, regenerate
  on a canonical Linux host; the harness normalization changes make the goldens
  machine-independent, so where they are regenerated stops mattering afterward.
- If `just ratchet-core-window` selects a version other than 0.34.33 (e.g. a newer
  release lands meanwhile, or 0.34.33's wheel set is still incomplete), trust the tool's
  selection — it only picks complete releases.
- The pre-existing `artifacts_stitches_persistent_filter_80x24` flake (bead 2) can still
  redden an individual Full CI run; if the first post-fix run trips on it, the next
  2-hour tick retries and the 6-hour freshness window absorbs the delay.
