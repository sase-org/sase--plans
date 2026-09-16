---
tier: tale
title: Fix stale pending gate-shell rows surviving the Agents-tab incomplete-load merge
goal:
  A long-running sase tui converges to a settled gate shell's terminal status (TALE
  DONE) on its own, without a restart.
size: medium
proposed_by: bbugyi200.athena.0lo
create_time: 2026-09-15 21:08:36
status: wip
---

# Fix stale pending gate-shell rows surviving the Agents-tab incomplete-load merge

## Goal

A long-running `sase tui` must converge to a settled gate shell's terminal status on its
own. Today a gate-shell row cached while the gate was pending can survive every
subsequent refresh with its stale `TALE`/`pending` projection until the TUI is
restarted, so the agent family renders `TALE` (awaiting review, with a live "settling
due in …" countdown) hours after the gate was answered and the family's code agent
finished `TALE DONE`.

## Symptom and confirmed diagnosis

Observed live: family `0ll` stuck at `TALE` in the Running/awaiting group while its
`--code` shell already showed `TALE DONE`. Every durable store was correct and had been
for ~2 hours:

- the gate member's `agent_meta.json` / `done.json` said `gate_state: "answered"`,
  `status_label: "TALE APPROVED"`, `outcome: "gated"`;
- the `~/.sase/agent_artifact_index.sqlite` row (columns and `record_json`, including
  `agent_meta.family_shell.state = "answered"`) was fresh;
- `sase gate list` showed the gate settled;
- a fresh out-of-process run of `load_agents_from_disk_with_state(...)` +
  `apply_status_overrides(...)` produced gate `TALE APPROVED` / root `TALE DONE` for
  every load flavor (unbounded Tier 1, bounded viewport prefix, with the live committed
  query). So restarting the TUI does fix the display; the staleness lives only in the
  running process's in-memory row pool.

Root cause chain (each step verified in code and, for step 3, reproduced by executing
the real merge function):

1. The Agents tab keeps a cached row pool across reloads. Every bounded viewport Tier-1
   load (`load_state.bounded_prefix and load_state.has_more`) and every exact
   artifact-delta load is _patched over_ that pool by
   `merge_incomplete_load_after_complete_history` in
   `src/sase/ace/tui/actions/agents/_loading_compute_merge.py`. Only a complete-history
   load replaces the pool wholesale, which in practice only happens at startup.
   Auto-refresh, the 5-minute sanity reconcile, and the tier1-index-revalidate reconcile
   all route through this merge (confirmed via `~/.sase/logs/tui_agent_loads.jsonl`:
   frequent `load_kind=full` entries whose `apply` stage ran, yet the stale row
   persisted).
2. A gate shell's row shape legitimately changes between pendency and settlement: while
   pending, its artifact dir has only `agent_meta.json` (no `workflow_state.json`, no
   `done.json`); after settlement those markers exist. The loader's classification
   (`agent_type`, workflow shape) can therefore differ between the cached pending row
   and the incoming settled row, which changes `_tier1_merge_key` (the key embeds
   `agent_type`). On a key mismatch, the merge falls back to the stable key
   `("artifact-row", project_file, raw_suffix)` gated by `_stable_replacement_wins`.
3. `_stable_replacement_wins` refuses the replacement:
   - `_terminal_artifact_projection_replaces` requires
     `agent_status_bucket(incoming) in {"Done", "Failed"}`, but a settled gate's
     `TALE APPROVED` status deliberately buckets as `Running`
     (`HANDOFF_SETTLED_STATUSES` in `src/sase/agent/status_buckets.py`) so a family
     never files under Done while its follow-up agent is still coming up.
   - The artifact-delta arm requires `cached.agent_type == incoming.agent_type`, which
     the pendency-to-settlement transition broke.

   Executed repro against the real merge: cached full list with the gate row reverted to
   pending shape, incoming = fresh bounded load. Same-type variant heals (key match).
   Type-changed variant keeps the cached `status='TALE', gate_state='pending'` row and
   drops the incoming settled row.

4. `apply_status_overrides` (`src/sase/ace/tui/models/_agent_status_apply.py`) then
   mirrors the still-"in-flight" pending gate row onto the family root, so the root
   shows `TALE`, buckets as awaiting input, and the detail pane renders a live "settling
   due in …" countdown from the stale pending state.
5. The initial trigger was a single missed or raced watcher delta for the gate dir
   during its ~90-second settle window. The 5-minute sanity reconcile exists exactly to
   heal missed events, but its loads are bounded (`has_more`) and so also route through
   the broken merge — one missed event became permanent until restart.

## Fix

A pure predicate change on the existing merge path. No new refresh code path, no new
disk reads, no unconditional list rebuilds (tui_perf rules: route refreshes through
existing paths, prefer selective updates, keep idle ticks cheap). This is TUI
presentation-layer row reconciliation in Python; no `sase-core` change.

1. In `src/sase/ace/tui/actions/agents/_loading_compute_merge.py`, add a
   shell-settlement arm to `_stable_replacement_wins` (or extend
   `_terminal_artifact_projection_replaces`): for a stable-key match (same
   `project_file` + `raw_suffix`, neither row a workflow step), the incoming row
   replaces the cached row when the incoming row carries terminal shell state and the
   cached row does not:
   - gate shells: `incoming.gate_id` is set and
     `gate_state_is_terminal(incoming.gate_state)` (`src/sase/gate_shell/state.py`)
     while the cached row's `gate_state` is non-terminal. Do NOT require bucket
     `Done`/`Failed` and do NOT require equal `agent_type` for this arm.
   - monitor shells: mirror the same rule with the monitor-state terminality helper if
     one exists (check `src/sase/ace/tui/models/_loaders/` `_meta_enrichment_monitor.py`
     and the monitor shell state module). If settled monitor stop statuses already
     bucket as `Done`, the existing terminal arm covers them — decide with a test, not
     an assumption.
2. Keep the guard's original purpose intact: a placeholder suffix shadow (an incoming
   row with no shell state and no done marker) must still not clobber a richer cached
   row. Scope the new arm strictly to rows that genuinely carry shell state on the
   incoming side.
3. Do not touch `HANDOFF_SETTLED_STATUSES` or gate-member bucketing: settled handoff
   statuses bucketing as `Running` is load-bearing for family grouping (approving a plan
   must not file the family under Done while the follow-up agent starts).
4. Optional, only if tests show remaining gaps: for the same stable key, allow an
   incoming row that carries a done marker (persisted `finished_at`/stop time) to
   replace a cached row that has none. Skip this if it breaks the placeholder-shadow
   guarantees.

## Tests

Extend `tests/test_agents_tab_incomplete_merge.py` and
`tests/test_agents_tab_artifact_delta_merge.py`, reusing
`tests/_agents_tab_incomplete_merge_helpers.py`:

1. Cached pending gate-shell row whose `agent_type` differs from the incoming settled
   row (the reproduced shape): after a bounded Tier-1 merge
   (`bounded_prefix=True, has_more=True`), the merged list holds exactly one row for
   that artifact dir with terminal gate state (`gate_state="answered"`, status
   `TALE APPROVED`), and after merge normalization the family root reads `TALE DONE`.
2. The same scenario through an artifact-delta load state
   (`artifact_source == "artifact_delta"`).
3. Same-type pending-to-settled replacement still works (key-match path, regression
   guard).
4. Existing placeholder-shadow tests keep passing: an incoming shadow row without shell
   state or done marker still loses to the cached row.
5. If a monitor-shell arm is added, one mirrored monitor test.

## Verification

- Read `sase/memory/lint_and_test.md` and run the checks it mandates before finishing
  (at minimum `just check`).
- Run the targeted test modules above.
- No perf-sensitive path gains I/O: the change is a predicate on rows already in memory.
  Confirm no new stat/glob/JSON reads were introduced in the merge module.

## Acceptance

- The reproduced type-changed pending-to-settled scenario converges to the settled row
  through the merge without a TUI restart.
- No regression in the placeholder-shadow, dedup, or same-type merge tests.
- `just check` passes.

## Out of scope

- Why the watcher missed the original settle delta (the sanity reconcile plus this fix
  make any such miss self-heal within one bounded reload).
- Telemetry for merge-refusal counts.
- The already-stale row in the currently running TUI session: a restart clears it; no
  data repair is needed anywhere.
