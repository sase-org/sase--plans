---
tier: tale
title: Stop workspace-claim placeholder STARTING rows from becoming phantoms
goal:
  The Agents tab never shows a `N starting` count for an agent that has no row, and a
  live agent's row never flips between RUNNING and STARTING while it starts up.
size: medium
proposed_by: bbugyi200.athena.zy
create_time: 2026-08-13 16:07:03
status: wip
---

# Plan: Stop workspace-claim placeholder STARTING rows from becoming phantoms

## Background: what the user reported

Two symptoms, reported together with the suspicion that they share a root cause:

1. The Agents-tab headline almost always shows `1 starting`, with no corresponding row
   anywhere in the panel.
2. Agent launches take a while, and during that window rows appear to jump between
   `RUNNING` and `STARTING` repeatedly before finally settling.

**The suspicion is correct.** Both symptoms come from the same defect, described below.
Symptom 1 is fully reproduced and root-caused; symptom 2 is root-caused in the
reconciliation code with one directly observed corroborating signal. The "launches take
a while" part is partly a real, separate cost and is scoped out explicitly (see
Non-goals).

## Root cause

The Agents tab builds rows for a live agent from two independent sources:

- **Liveness claims** — the ProjectSpec `RUNNING` field and home-mode `running.json`.
  `src/sase/ace/tui/models/_loaders/_running_loaders.py` seeds those rows with
  `status="STARTING"` and relies on `enrich_agent_from_meta()` to promote them once
  `agent_meta.json` records `run_started_at`. Its own module docstring says these are
  "liveness claims, not display-status claims".
- **Artifact records** — `workflow_state.json`, `done.json`, and `agent_meta.json` from
  the artifact snapshot, which carry a genuinely observed status.

`"STARTING"` is assigned in exactly three places, all in `_running_loaders.py` (lines
166, 227, 306). Nothing else in the TUI model layer ever produces it. So `STARTING`
means _"the claim source knows nothing about this agent yet"_ — a placeholder. The
pipeline nevertheless treats it as an authoritative status, and that produces both
symptoms.

### Symptom 1 — the permanent phantom `1 starting`

A monitor's workspace claim is written with `workflow == "ace-monitor"`
(`MONITOR_WORKSPACE_CLAIM_WORKFLOW`, `src/sase/monitor/start.py:57`), deliberately
distinct from `ace-run` so the starter's runner-exit cleanup cannot release it. But the
monitor member's artifacts live under `artifacts/ace-run/<timestamp>/`.

`get_artifacts_dir()` (`src/sase/ace/tui/models/artifact_files.py:22`) maps a claim's
workflow label to an artifacts subtree. It handles `ace(run)*`, `axe(fix-hook)`,
`axe(crs)`, `axe(mentor)*`, `mentor(...)`, `mentor`, and `fix_hook`, then falls through
to `workflow_name = workflow`. For `"ace-monitor"` that means it looks for
`artifacts/ace-monitor/<timestamp>/`, which never exists, and returns `None`.

The consequences chain:

- `enrich_agent_from_meta(agent, None)` is a no-op, so the placeholder `STARTING` is
  never promoted — for the monitor's entire lifetime.
- `dedup_running_vs_workflow()` (`src/sase/ace/tui/models/_dedup.py:293`) only absorbs
  claim rows whose workflow is `ace(run)*`, `ace-run`, or `run`, so the monitor claim
  row is never merged into the monitor member's real row.
- `dedup_by_pid()` cannot collapse them either: the claim carries the monitor supervisor
  pid while the member row carries the runner pid.
- `agent_is_rendered_in_agents_panel()` (`src/sase/ace/tui/models/agent_panels.py:104`)
  hides _every_ `STARTING` row from the panel, and
  `AgentPanelIndex.hidden_starting_indices` feeds the headline `N starting` chip
  (`src/sase/ace/tui/actions/agents/_display_detail_info.py:54-61`).

Net effect: for the whole duration of every monitor there is one extra top-level row
stuck at `STARTING` that is invisible, unselectable, and unkillable, and shows up only
as a phantom count.

**Evidence gathered on the reporting host (all read-only):**

- A live claim `ws=16 pid=2211985 workflow=ace-monitor ts=20260813153749` existed while
  `load_all_agents()` returned `hidden_starting_indices` containing exactly one entry:
  `('STARTING', AgentType.RUNNING, 'gh_sase-org__sase', 'ace-monitor', '20260813153749')`,
  and `Agent.get_artifacts_dir()` for that row returned `None`.
- Constructing the same row with `workflow="ace(run)-..."` instead resolves the
  artifacts dir and enrichment promotes it to `RUNNING`; with `workflow="ace-monitor"`
  it resolves to `None` and stays `STARTING`.
- A 10-minute poll of `load_agents_from_running_field()` at 0.3 s resolution saw three
  separate `ace-monitor` claims appear (at t=21 s, t=320 s, t=505 s). Every one held
  `status=STARTING, artifacts_dir_found=False` and never changed state. The single
  ordinary agent launched in that window went `STARTING`(no dir) → `STARTING`(dir found)
  → `RUNNING` in 7.4 s.
- 42 monitor members ran on this host today. That launch rate is why the chip is
  essentially always on screen.

### Symptom 2 — the RUNNING/STARTING flap during launch

For claim rows that _do_ merge, `dedup_running_vs_workflow()` ends with:

```python
elif matched.status == "RUNNING" and agent.status != "RUNNING":
    matched.status = agent.status
```

`matched` is the artifact-derived workflow row; `agent` is the claim row. So a
placeholder `STARTING` overwrites a genuinely observed `RUNNING`. Because
`agent_is_rendered_in_agents_panel()` hides `STARTING`, that downgrade does not merely
relabel the row — it removes it from the panel entirely.

The two refresh paths then disagree about whether claim rows exist at all:

- The broad load (`_load_agents_from_all_sources`,
  `src/sase/ace/tui/models/agent_loader.py:257`) calls
  `load_agents_from_running_field()`.
- The exact artifact-delta load (`load_artifact_delta_agents`,
  `src/sase/ace/tui/models/agent_loader.py:493`) deliberately does not — see the comment
  on `_mark_live_artifact_delta_runners`: "Artifact deltas intentionally do not rescan
  ProjectSpec RUNNING claims."

Both patch the same row through the same Tier-1 merge key (`_tier1_merge_key` →
`("artifact-root", WORKFLOW, raw_suffix)` in
`src/sase/ace/tui/actions/agents/_loading_compute_merge.py`). So while `run_started_at`
is still unwritten, a delta refresh renders the row `RUNNING` (visible) and the next
broad refresh renders it `STARTING` (hidden). The TUI runs both continuously —
inotify/countdown-driven deltas (`_poll_starting_agent_transitions`,
`src/sase/ace/tui/actions/agents/_loading_refresh_polling.py:126`) plus 1 s auto-refresh
— so the row appears and disappears repeatedly until the runner records
`run_started_at`.

**Evidence:** rows carrying `agent_type == AgentType.WORKFLOW` **and**
`status == "STARTING"` were observed live (timestamps `20260813150654`,
`20260813153709`). Workflow loaders never emit `STARTING`; the only code path that can
put that value on a `WORKFLOW` row is the `dedup_running_vs_workflow` copy above. That
confirms the downgrade half of the mechanism directly.

The flap window is wide enough to be plainly visible: measured across 924 launches on
this host, launch-timestamp → `run_started_at` is **p50 13.1 s, p75 24.6 s, p90 110 s**
(excluding runs with a dependency wait, which display as `WAITING`/`QUEUED` instead).
The design assumption that `STARTING` is a sub-second transient — which is what
justifies hiding those rows unconditionally — does not hold in practice.

## Design

Three changes, smallest-blast-radius first, plus one defense-in-depth change so this
whole class of bug can never again present as an invisible count.

### 1. A claim's placeholder `STARTING` must never overwrite an observed status

In `dedup_running_vs_workflow()` (`src/sase/ace/tui/models/_dedup.py`), the final
status-resolution branch must skip when the incoming claim row's status is the
`STARTING` placeholder. Keep every other branch as-is: terminal statuses from the claim
row still win, `FAILED` + live `RUNNING` still promotes, and real semantic statuses
(`PLAN`, `WAITING`, `QUESTION`, …) still win over a raw `RUNNING`.

This alone makes the broad and delta refresh paths agree on the row's status for the
whole pre-`run_started_at` window, which removes the flap.

Add a short comment stating _why_: `STARTING` on a claim row is "no information yet",
not an observation, and the artifact record is strictly better information.

### 2. Resolve monitor claims to their real `ace-run` artifacts dir

In `get_artifacts_dir()` (`src/sase/ace/tui/models/artifact_files.py`), add a case for
the monitor claim workflow so it maps to the `ace-run` workflow dir, exactly like
`ace(run)*` does.

Do **not** `import sase.monitor.start` from this module — it is on the TUI's hot load
path and that module pulls in the whole monitor stack. Move
`MONITOR_WORKSPACE_CLAIM_WORKFLOW` into a small leaf module (e.g.
`src/sase/monitor/claims.py`) that has no heavy imports, re-export it from
`sase/monitor/start.py` so existing importers (`src/sase/monitor/settlement.py`,
`src/sase/ace/scheduler/stale_running_cleanup.py`, `start.py`'s own `__all__`) keep
working unchanged, and import the constant from the leaf module in the loader. Verify
with `just lint` that no import cycle is introduced.

Ordering inside `enrich_agent_from_meta()` is already correct for this: the
`run_started_at` promotion (`_meta_enrichment_filesystem.py:240-251`) runs before
`apply_monitor_meta()` (line 380), whose `apply_monitor_running` guard
(`_meta_enrichment_common.py:75`) refuses to promote a row that is still `STARTING`. So
once the artifacts dir resolves, a running monitor's claim row correctly becomes its
`monitor_start_status` (default `MONITORING`) instead of `STARTING`. Confirm this rather
than assuming it.

### 3. Let the monitor claim row merge away

Extend the workflow allow-list in `dedup_running_vs_workflow()` to also accept the
monitor claim workflow, so a monitor's claim row merges into the monitor member's
artifact-derived row instead of surviving as a second top-level row. Use the same leaf
constant as change 2.

After changes 2 and 3, a live monitor contributes exactly one row, with a real monitor
status, and contributes nothing to the `starting` count.

### 4. Defense in depth: stop hiding a `STARTING` row forever

`agent_is_rendered_in_agents_panel()` currently hides every `STARTING` row
unconditionally. That is what turned both defects above into _invisible_ problems rather
than visibly wrong rows. Bound it: hide a `STARTING` row only while it is plausibly
transient, and render it normally once it is older than a grace window.

- Add a module-level constant in `src/sase/ace/tui/models/agent_panels.py` (suggested
  `STARTING_ROW_HIDE_GRACE_SECONDS = 120.0`), chosen to sit above the measured p90 of
  110 s so ordinary launches are unaffected.
- Hide only when `agent.start_time` is present and within the window. A row with no
  `start_time`, or one older than the window, renders as a normal selectable row.
- Keep the headline accounting consistent: `hidden_starting_indices` must continue to
  count only the rows that are actually hidden, so `AgentPanelIndex.top_level_total`
  still equals rendered + hidden and the headline total does not double-count. Check
  `top_level_total`, `non_child_total`, and `_agent_info_metrics()` together.
- Comparing against "now" makes this time-dependent. Thread an injectable `now`
  parameter (defaulting to `datetime.now(get_timezone())`) through
  `agent_is_rendered_in_agents_panel()` / `build_agent_panel_index()` so tests are
  deterministic, rather than freezing wall-clock in tests.

This means any future unmapped claim label surfaces as a real, selectable, killable row
instead of a phantom count.

## Files expected to change

- `src/sase/ace/tui/models/_dedup.py` — changes 1 and 3.
- `src/sase/ace/tui/models/artifact_files.py` — change 2.
- `src/sase/monitor/claims.py` (new) and `src/sase/monitor/start.py` — leaf constant
  plus re-export.
- `src/sase/ace/tui/models/agent_panels.py` — change 4.
- `src/sase/ace/tui/models/agent_panel_index.py` and
  `src/sase/ace/tui/actions/agents/_display_detail_info.py` — only if change 4's `now`
  threading or count accounting requires it.

## Tests

Add regression coverage next to the existing suites
(`tests/ace/tui/models/test_agent_panels.py`,
`tests/ace/tui/models/test_agent_panel_index.py`,
`tests/test_agent_loader_dedup_cross_project_collision.py`,
`tests/ace/tui/test_starting_agent_poll.py`):

1. **Monitor claim resolves.** A claim row with `workflow="ace-monitor"` and a 14-digit
   `raw_suffix` whose `ace-run/<ts>/` dir exists resolves through `get_artifacts_dir()`
   to that dir, and enrichment leaves it non-`STARTING`.
2. **Constant is pinned.** A test asserts the loader's monitor mapping and
   `MONITOR_WORKSPACE_CLAIM_WORKFLOW` cannot drift apart.
3. **No phantom row.** Given a monitor claim plus the monitor member's `ace-run` record,
   the loader yields exactly one row for that timestamp and
   `build_agent_panel_index(...).hidden_starting_indices` is empty.
4. **No placeholder downgrade.** `dedup_running_vs_workflow()` with a `RUNNING` workflow
   row and a `STARTING` claim row for the same `(project_file, raw_suffix)` keeps
   `RUNNING`. Assert the surviving branches still work: a `DONE`/`FAILED` claim row
   still wins, and a semantic status such as `PLAN` still overrides a bare `RUNNING`.
5. **Anti-flap invariant.** For a live agent whose `agent_meta.json` has no
   `run_started_at` but whose `workflow_state.json` says running, the broad load and the
   artifact-delta load produce the _same_ status for that row. This is the direct
   regression test for symptom 2; write it so it would fail on today's code.
6. **Grace window.** A `STARTING` row whose `start_time` is inside the window is hidden
   and counted in `hidden_starting_indices`; one older than the window is rendered,
   selectable, and not counted as hidden. Assert the headline total is unchanged in both
   cases.

## Verification

- `just install` first (workspaces are ephemeral and dependencies may have moved).
- `just check` for the whole-repo lint gates plus the diff-scoped test lane.
- Because this touches the shared agent-loader/dedup path that many suites exercise,
  finish with `just check-full` through `/sase_monitor`
  (`sase monitor start --command 'just check-full' …` with a `--next` action) rather
  than inline.
- Manual confirmation on the reporting host, read-only: while a monitor is live
  (`sase monitor` lanes are started constantly by agents), `load_all_agents()` +
  `build_agent_panel_index(...)` must report an empty `hidden_starting_indices` for the
  monitor's timestamp, where today it reports exactly one entry. Stub
  `sase.ace.tui.models._loaders._running_loaders.release_workspace` to a no-op when
  doing this so the probe cannot mutate the user's ProjectSpec files.

## Non-goals

- **Actual launch latency.** The measured launch→`run_started_at` window (p50 13.1 s) is
  real work, not a display bug, and this plan does not try to shrink it. One specific
  suspect was noticed while diagnosing and is deliberately left alone here:
  `_try_claim_runner_slot()` in `src/sase/axe/run_agent_wait_slots.py` runs a full
  `scan_agent_artifacts()` while holding the host-wide `runner_slots.lock`, and every
  queued runner repeats that every 2 s. With ~25 concurrent agents and thousands of
  artifact dirs that lock is heavily contended. It deserves its own measurement and its
  own task bead; do not fold it into this change.
- **Broad-load cost.** A full `load_all_agents()` was measured at roughly 5 s on this
  host. Out of scope here.
- **Pinned dead-pid claims.** A pinned claim whose pid has become a zombie is retained
  by `cleanup_stale_running_entries()` while its artifacts exist. That is the documented
  held-workspace behavior and it does _not_ produce a `STARTING` row
  (`_release_stale_running_claim` skips pinned claims and the loader then drops the
  row). Confirmed not a contributor; leave it alone.
- No changes to the runner's `record_run_started_at()` timing or to when claims are
  written. The placeholder-then-promote design is fine; only its reconciliation is
  wrong.
