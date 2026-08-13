---
tier: tale
title: Fix duplicate settled-monitor rows in the ACE Agents tab
goal:
  Every settled monitor member renders as exactly one ACE agent row that keeps its
  stop-status label and can resolve its own artifacts directory.
size: medium
proposed_by: bbugyi200.athena.zv
create_time: 2026-08-13 15:22:30
status: wip
---

# Plan: Fix duplicate settled-monitor rows in the ACE Agents tab

## Symptom

In the ACE Agents tab, a settled monitor member (`<lane>--mon`) renders as **two** rows
with the same name, the same stop time, and the same duration:

```
└ ⚡ ⏰ just (MONITORED ✗ 1) ◆ sase-l3.1--mon   15:03:41 · 1m38s
└ ⚡ ⏰ just (FAILED    ✗ 1) ◆ sase-l3.1--mon   15:03:41 · 1m38s
```

Two further consequences of the same defect:

- The family-member panel (`FAMILY MEMBERS`) picks the phantom row, so the detail pane
  shows `--mon · MONITOR · ✗ FAILED` while the selected tree row says `MONITORED` — the
  two views disagree about the same member.
- The correct (`MONITORED`) row cannot resolve its own artifacts directory:
  `Agent.get_artifacts_dir()` returns `None` for it, so artifact/file actions on that
  row have nothing to open. The phantom row is the only one that knows the path.

This is not failure-specific. It reproduces for every settled monitor, including
successful ones (a monitor whose stop status is `EPIC CREATED` also gets a phantom
`FAILED` twin).

## Reproduction and baseline

Run against a real `~/.sase` artifact tree (this is how the defect was confirmed; it
found 14 duplicated `--mon` pairs — every settled monitor on the machine):

```python
from collections import Counter
from sase.ace.tui.models import agent_loader as al

agents = al.load_all_agents()
counts = Counter(
    (a.raw_suffix, a.agent_name)
    for a in agents
    if a.agent_name and a.agent_name.endswith("--mon")
)
print("duplicate --mon pairs:", sum(1 for v in counts.values() if v > 1))
```

The two rows for one settled monitor look like this:

```
AgentType.RUNNING  status=MONITORED  pid=None     workflow=ace-run  project_file=''
AgentType.WORKFLOW status=FAILED     pid=1528509  workflow=run      project_file='.../gh_sase-org__sase.sase'
```

## Root cause

The ACE loader unions several per-marker projections over one artifact snapshot
(`_load_agents_from_artifact_snapshot_sources` in
`src/sase/ace/tui/models/agent_loader.py`) and relies on dedup to collapse the
projections that describe the same artifact directory. A settled monitor defeats both
halves of that contract, for two independent reasons.

### Defect A — a monitor member never leaves a terminal `workflow_state.json`

A monitor member's artifacts directory is created through `create_followup_artifacts()`
(`src/sase/axe/run_agent_helpers_artifacts.py`), which seeds `workflow_state.json` with
`status: "running"`, `appears_as_agent: true`, and `pid: os.getpid()` — the PID of the
_starter_ process, not the detached supervisor.

Nothing ever rewrites that file. A monitor is settled by writing `done.json` in
`_finish_monitor()` (`src/sase/monitor/supervise.py`), `_reconcile_dead_supervisor()`
(`src/sase/monitor/reconcile.py`), or `_teardown_failed_member()`
(`src/sase/monitor/start.py`); none of them touches `workflow_state.json`. Verified on
disk: all 25 monitor artifact dirs from one day carry
`workflow_state.json → status: "running"`.

`load_workflow_states_from_snapshot()`
(`src/sase/ace/tui/models/_loaders/_workflow_snapshot_loaders.py`) therefore sees a
`running` workflow whose PID is dead and applies the stale-workflow heuristic:
`display_status = "FAILED"`. That is the phantom row.

While the monitor is alive the phantom is invisible, because `apply_monitor_meta()`
(`src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py`) overwrites the row's
status with `monitor_start_status` (`MONITORING`) whenever
`agent_meta.json::monitor_state == "running"`. Once the monitor settles, that mask is
gone and the synthesized `FAILED` surfaces.

### Defect B — monitor `done.json` markers omit `project_file`

Every other done marker is built by `build_done_marker()`
(`src/sase/axe/run_agent_markers.py`), which always writes `project_file`. The three
monitor writers hand-roll their marker dict and omit it. So
`_build_done_agent_from_record()` (`src/sase/ace/tui/models/_loaders/_done_loaders.py`)
sets `project_file=""` on the done row.

That breaks two things:

1. `dedup_running_vs_workflow()` (`src/sase/ace/tui/models/_dedup.py`) keys on
   `(project_file, raw_suffix)` — deliberately project-scoped, so two projects that
   launched in the same clock second cannot cross-contaminate. `("", ts)` never matches
   `("/…/gh_sase-org__sase.sase", ts)`, so the two rows never collapse.
2. `get_artifacts_dir()` (`src/sase/ace/tui/models/artifact_files.py`) derives the
   project from `agent.project_file` when a row carries no explicit `artifacts_dir`.
   With `""` it returns `None`, so the settled monitor row loses its artifacts path.

Neither defect alone produces the screenshot: Defect A creates a spurious row and Defect
B stops it from being collapsed.

Note that `tests/ace/tui/models/test_monitor_rows.py` already covers the terminal
monitor projection, but its `DoneMarkerWire` fixture sets
`project_file="/tmp/.sase/projects/sase/sase.sase"` — a field the real writers never
emit. Fixture drift is why this stayed hidden.

## Fix

Four changes. Steps 1 and 2 are the fix proper (and repair already-written artifacts,
since they are loader-side); steps 3 and 4 stop new artifacts from being written wrong.

### Step 1 — a settled monitor projects one row, from its done marker

For an artifact whose `done.json` has `outcome == "monitored"`, the terminal row must
come from the done marker (it owns `status_label`, `monitor_state`, `monitor_exit_code`,
and `response_path`). Its `workflow_state.json` is vestigial launch scaffolding and must
not project a second agent row.

- In `load_workflow_agents_from_snapshot()`
  (`src/sase/ace/tui/models/_loaders/_workflow_snapshot_loaders.py`), skip entries whose
  record has `record.done is not None and record.done.outcome == "monitored"`. The
  record set is already in hand there (`record_by_dir`), so this costs one dict lookup
  per entry and does not change the loader's complexity or add I/O.
- Mirror it in the filesystem twin `load_workflow_agents()`
  (`src/sase/ace/tui/models/_loaders/_workflow_loaders.py`), which reads
  `<artifacts_dir>/done.json` through `load_json_cached` and skips the same case. This
  twin is a fallback/test path, but the twins must not diverge.
- Do **not** put the skip in `load_workflow_states_from_snapshot()` /
  `load_workflow_states()`: those model workflow state, and a monitor member does have a
  `workflow_state.json`. The invariant being enforced is "one agent row per artifact",
  which belongs in the agent projection.
- Running monitors are unaffected: no `done.json` exists yet, so the workflow row still
  carries the live `MONITORING` row exactly as today.

### Step 2 — a done row inherits its record's project file

In `_build_done_agent_from_record()`
(`src/sase/ace/tui/models/_loaders/_done_loaders.py`), fall back to the record's own
project file when the marker omits one: `done.project_file or record.project_file`.
`record.project_file` is the same value the workflow projection already uses, so the
fallback cannot invent a wrong project.

Mirror it in `_load_done_agent_for_dir()` (same module) by deriving the project from the
artifact path — use `parse_agent_artifact_path()`
(`src/sase/core/agent_artifact_paths.py`, which handles both the legacy flat and the
sharded layout) plus `preferred_project_spec_path()`, rather than the hand-rolled
`artifact_dir.parents[2]` indexing already in that function.

This restores `get_artifacts_dir()` for settled monitor rows and makes the row's
project-scoped identity correct for any other consumer keyed on `project_file`.

### Step 3 — monitor done markers record `project_file`

Add `project_file` to the marker dict in all three writers, so new artifacts match
`build_done_marker()`:

- `_finish_monitor()` in `src/sase/monitor/supervise.py` — `project_name` is already
  computed there via `project_name_from_artifacts_dir()`.
- `_reconcile_dead_supervisor()` in `src/sase/monitor/reconcile.py` —
  `record.project_name` is in scope.
- `_teardown_failed_member()` in `src/sase/monitor/start.py` — resolve the project from
  `artifacts_dir` the same way (`project_name_from_artifacts_dir()` in
  `src/sase/monitor/settlement.py`).

Use `get_project_file_path()` (`src/sase/workflows/utils.py`), which resolves through
`preferred_project_spec_path()` and therefore produces the same string the artifact
scanner reports. Skip the field when the project cannot be resolved rather than writing
an empty string. Factor the shared marker construction if the three copies drift too
far, but do not restructure the monitor lifecycle for it.

### Step 4 — settling a monitor finalizes `workflow_state.json`

At the same three settle points, rewrite the member's `workflow_state.json` `status` to
`"completed"` (preserving every other field, notably `appears_as_agent` and `context`),
and refresh the artifact index through
`update_agent_artifact_index_for_marker_mutation()` as the marker writers already do.
Skip silently when the file is absent or unreadable — settlement must never fail because
of it.

`"completed"` describes the _member's_ lifecycle; the monitored command's outcome is
carried by `monitor_state` / `monitor_exit_code` / `status_label`, which is what the
Agents tab renders. Steps 1 and 2 already make the Agents tab correct, so this step is
about not leaving a permanently non-terminal marker on disk for other
`workflow_state.json` consumers (e.g. `src/sase/agents/cli_tribe.py`,
`src/sase/agent/running_listing.py`) to trip over.

## Rejected alternative (do not implement)

The apparently smaller fix — only give the done row a project file (Step 2) and let
`dedup_running_vs_workflow()` collapse the pair — was measured and is wrong. It does
collapse all duplicates, but the survivor is the WORKFLOW row, and the merge's status
rules keep the workflow status: the rows then render `FAILED` for a monitor whose stop
status was `MONITORED`, and `FAILED`/`RUNNING` for monitors whose stop status was
`EPIC CREATED`. Making the merge prefer the monitor label does not rescue it either,
because `apply_status_overrides()` runs after dedup and rewrites the surviving WORKFLOW
row's status again. Keeping the done row as the survivor (Step 1) avoids that whole
pipeline and preserves the row that already renders correctly today.

## Tests

- `tests/ace/tui/models/test_monitor_rows.py`
  - A record with `workflow_state` `status="running"`, `appears_as_agent=True`, and a
    dead PID **plus** a `done` marker with `outcome="monitored"` projects **no**
    workflow agent row (`load_workflow_agents_from_snapshot`), and exactly one done row
    carrying the stop label.
  - A record whose `DoneMarkerWire` omits `project_file` (the shape the real writers
    emit) yields a done row whose `project_file` equals `record.project_file`.
  - Regression fixture parity: keep or add a case where the marker _does_ carry
    `project_file` so the explicit value still wins.
  - A running monitor (no done marker) still projects its workflow row as `MONITORING` —
    guards against over-broad suppression.
- Loader-level regression, following the `_scan_artifacts_for_loader` patch pattern in
  `tests/_agent_loader_helpers.py` / `tests/test_agent_loader_dedup_merge.py`:
  `load_all_agents()` over a snapshot containing one settled-monitor record returns
  exactly one row, with the stop-status label, the monitor bucket, and a resolvable
  `get_artifacts_dir()`.
- `tests/monitor/test_monitor_supervise.py` (and the start/reconcile equivalents):
  settling writes `project_file` into `done.json` and leaves `workflow_state.json` at a
  terminal status with `appears_as_agent` preserved.

## Verification

1. `just install` first — workspaces are ephemeral and may have stale dependencies.
2. `just check` (whole-repo lint gates + diff-scoped tests). Hand it to `/sase_monitor`
   if it runs long.
3. `just check-full` through `/sase_monitor` before landing, since this touches the
   monitor lifecycle writers and the ACE loader.
4. Empirical check against the real artifact tree: the duplicate-pair count from the
   reproduction snippet must go from 14 (or whatever the current baseline is on the
   machine) to 0, and spot-checked settled monitors must keep their labels — a failed
   `just check` monitor stays `MONITORED` with bucket `Failed`, an epic-creation monitor
   stays `EPIC CREATED` with bucket `Done`, and both resolve `get_artifacts_dir()`.

## Notes and non-goals

- **Rust core boundary.** The duplication is a Python-side projection artifact: the Rust
  scanner already emits exactly one record per artifact directory, and the non-TUI
  consumers that build one row per record (`src/sase/agent/running_listing.py`,
  `src/sase/integrations/_agent_list_entry_builder.py`) do not show duplicates. The fix
  therefore stays in the Python loaders and in the existing Python monitor lifecycle
  writers; no wire/schema change is needed.
- **Performance.** Step 1 adds one dict/`None` check per workflow entry inside a loop
  that already iterates the same records, and reads no new files; Step 2 is a fallback
  expression. No render-path or startup work is added.
- **Legacy artifacts.** Steps 1 and 2 are loader-side, so already-written monitor
  artifacts stop duplicating immediately, with no migration or backfill.
- **Out of scope.** The stale-workflow heuristic itself (`running` + dead PID →
  `FAILED`) is correct for real workflows and is left alone. Writing the supervisor's
  PID (rather than the starter's) into the monitor member's `workflow_state.json` is a
  separate liveness question owned by the monitor reconcile path and is not part of this
  plan.
