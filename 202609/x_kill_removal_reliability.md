---
tier: epic
title: Reliable Agents-tab x kill and dismiss
goal: 'Pressing `x` on any Agents-tab node (agent, clan container, workflow, monitor,
  proc shell, gate, panel, group, or marked set) removes every affected row at once
  and it never comes back. Every process that belongs to a killed node is verifiably
  terminated, including descendants that left the runner''s process group. This holds
  even if the TUI exits right after the keypress.

  '
phases:
- id: core-wire
  title: Rust cleanup wire for live runners and atomic dismissed index
  depends_on: []
  size: medium
  description: 'core-wire: in sase-core, add runner_is_live to the cleanup target
    wire (schema 5) so a FAILED row with a live runner becomes a kill item. Add an
    atomic, locked, merge-on-write dismissed-index update API. Then update the Python
    wire, the reference planner, the target projection, and the sase-core revision
    pin in the same change.'
- id: row-tombstones
  title: Session removal tombstones honored at every roster publication
  depends_on: []
  size: medium
  description: 'row-tombstones: record a session tombstone for every x-driven removal.
    Honor it in the load compute filter, the Tier-1 merge, apply time (removal-generation
    recheck), fleet reprojection, and refilter, ahead of runner_is_live. Keep the
    local roster copy in sync, fix the revive-modal repair that drops kill identities,
    and stop one failed signal from blocking the removal of every other row.'
- id: tree-kill
  title: Verified process-tree termination in the durable cleanup proc
  depends_on:
  - row-tombstones
  size: medium
  description: 'tree-kill: add a shared terminator that finds an agent''s whole process
    set (process group, session, ppid tree, and inherited launch scratch key). It
    sends SIGTERM, escalates to SIGKILL, and verifies death. The durable persist-cleanup
    proc runs it before it releases workspaces or deletes artifacts. The TUI only
    sends the immediate SIGTERM. Also stop the in-flight guard from dropping whole
    batches.'
- id: member-scope
  title: x stops every member kind instead of skipping it
  depends_on:
  - row-tombstones
  - tree-kill
  size: medium
  description: 'member-scope: focused x on a running monitor, an active proc shell,
    or a pending gate now stops or cancels it and removes the row in one step. Clan,
    panel, group, and marked cleanups include active proc shells and pending gates
    instead of skipping them. Leftover members are named explicitly, and remote clan
    members are resolved correctly.'
- id: additive-dismissals
  title: Additive dismissed-index persistence for every writer
  depends_on:
  - core-wire
  - member-scope
  size: medium
  description: 'additive-dismissals: switch the cleanup transactions and every other
    dismissed-agents writer from full-snapshot saves to the core-wire add/remove API.
    Revive becomes a removal. Concurrent procs, runners, and TUIs can no longer lose
    each other''s dismissals.'
- id: e2e-regression
  title: End-to-end x regression coverage
  depends_on:
  - additive-dismissals
  size: small
  description: 'e2e-regression: drive the real Agents tab through Textual pilot against
    on-disk fixture agents with real process trees. Race an in-flight load, a fleet
    reprojection, and a live runner against x on a clan and on single rows. Assert
    the rows never reappear and every fixture process is dead.'
proposed_by: bbugyi200.athena.0ra
create_time: 2026-09-24 16:28:27
status: wip
bead_id: sase-18d
---

- **PROMPT:** [prompts/202609/x_kill_removal_reliability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/x_kill_removal_reliability.md)
- **BEAD:** [sase-18d](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18d/README.md)

# Plan: Reliable Agents-tab `x` kill and dismiss

## Problem

`x` on the Agents tab (`action_kill_agent`,
`src/sase/ace/tui/actions/agents/_kill_action_flow.py`) removes rows optimistically
(`_apply_killed_agents_in_memory` in `_kill_identity.py`, `_apply_dismissal_in_memory*`
in `_dismiss_memory.py`), then persists through a durable `sase agent persist-cleanup`
proc (`_kill_procs.py`, `_dismissing.py`, `src/sase/ops/commands/_agent_cleanup.py`,
`_kill_transactions.py`, `_kill_persistence.py`). Users see removed agent and clan nodes
come back, linger, or never leave. Separately, processes of killed agents can survive.

## Root causes (confirmed by tracing the code)

Rows that fail to leave or come back:

1. **Live-runner liveness overrides dismissal.** Commit `786cdb7f7` (retry status) made
   any row with `runner_is_live=True` bypass the dismissed-set filter:
   - `_loading_compute.py`
     (`if (a.runner_is_live or a.identity not in effective_dismissed) ...`);
   - `is_dismissed()` in `merge_incomplete_load_after_complete_history`
     (`_loading_compute_merge.py`: `if agent.runner_is_live: return False`).

   A killed runner is still alive for a while after `x`: its SIGTERM handler is soft,
   and SIGKILL only arrives about 0.5 s later from a TUI daemon thread. Any load that
   probes the pid in that window revives the row. The cached copy then keeps
   `runner_is_live=True`, so later bounded, delta, and Tier-1 merges carry it forward
   until a complete-history load. The same happens when you dismiss a FAILED row whose
   runner is live (for example in retry backoff): the planner dismisses it without a
   kill because FAILED is a dismissable status, the next load brings it back, and the
   retry keeps running.

2. **Fleet reprojection rebuilds from a stale local roster.** Non-clan kills
   (`_kill_identity.py`) and fast-path dismissals (`_dismiss_memory.py`,
   `_proc_shell_dismiss.py`) filter `_agents` and `_agents_with_children`, but never
   `_agents_local_with_children`; only `_refilter_agents` re-syncs it. Every load apply
   schedules a fleet refresh (`_loading_apply.py`), which the NavigationGate often
   defers until right after the user's j/k plus `x`. That refresh ends in
   `_reproject_agents_from_current_mode` (`_fleet_projection.py`), which rebuilds the
   list from the stale copy, so the removed rows come back.
3. **An in-flight async load overwrites the optimistic removal.**
   `_loading_disk_full.py` (and the delta path) snapshots `set(self._dismissed_agents)`
   and the cached roster, awaits three worker stages, and then
   `_apply_loaded_agents_prepared_inner` assigns the roster wholesale. Nothing re-checks
   removals made during those awaits, and a kill never makes an in-flight load stale.
4. **Clan containers are re-derived from any member left over.** `project_clan_tree`
   synthesizes `(RUNNING, "clan:<clan>", generation)` from the remaining members.
   `_present_bulk_kill_modal` (`_marking_kill.py`) deliberately skips active proc shells
   and waiting gates, remote members go down a separate path, and rows the planner
   silently skips stay. Any one of these keeps the clan node on screen.
5. **Two-step `x`.** Focused `x` on a running monitor or an active proc shell
   (`_monitor_stop_flow.py`) only stops it; the row stays until a second `x`. A pending
   gate refuses with "Gate is waiting for a decision".
6. **Opening the revive modal deletes kill records.** `_repair_dismissed_projection`
   (`_revive_archive.py`) keeps only identities that have a bundle or appear in
   `_dismissed_agent_objects`. Single kills add to neither, so killed agents come back
   after the modal opens. The repair also reassigns `self._dismissed_agents` from a
   worker thread.
7. **One signal failure hides nothing.** `_do_kill_agent`'s `signal_failed` early return
   (`_kill_flow.py`) skips the in-memory removal for every target, including ones that
   already received SIGTERM.
8. **The dismissed index loses updates across processes.**
   - Each persist-cleanup proc saves a full snapshot, and the generation stamp is lost
     across the subprocess boundary (`identities_from_json` returns a plain set), so an
     older snapshot can land last.
   - The Rust save is a plain `fs::write` (sase-core
     `agent_cleanup/execution.rs::save_dismissed_agents_index`).
   - Other writers load, modify, and save the whole file:
     `axe/run_agent_runner_lifecycle.py`, `agent/running.py`, `agent/names/_auto.py`.

   This shows up on restart or in another TUI.

Processes that survive:

9. **Escalation lives in a TUI daemon thread.**
   `request_user_kill(wait=False, background=True)` (`src/sase/agent/user_kill.py`)
   escalates SIGTERM to SIGKILL in a thread that dies with the TUI. Nothing verifies the
   outcome, yet the durable proc still releases the workspace and, for workflows,
   deletes the artifacts.
10. **Only the runner's process group is signalled.** Several things escape `killpg`:
    - inline `sase tool run` children (`start_new_session=not has_owner`);
    - runner helpers (`noninteractive_subprocess.py`,
      `finalizers/bounded_subprocess.py`, `_linked_repo_workspaces.py`);
    - provider-leaked session leaders.

    A ppid-walk reaper exists (`src/sase/llm_provider/_subprocess_reap.py`) but the
    user-kill path never uses it. When the recorded pid is not a group leader, `killpg`
    raises `ProcessLookupError`, which is reported as `already_stopped`.

11. **The in-flight guard drops persistence.** `_kill_procs.py` and `_dismissing.py`
    skip the whole batch's persistence when any identity overlaps an in-flight proc.

## Design decisions

- **Session tombstones outrank liveness.** An identity the user removed with `x` in this
  TUI session stays hidden, whatever `runner_is_live` says. The retry protection from
  `786cdb7f7` still applies to _stale, not-this-session_ dismissals.
- **Success-terminal rows are not killed.** DONE, PLAN DONE, TALE DONE, PLAN COMMITTED,
  EPIC CREATED, PLAN REJECTED, and STOPPED rows whose runner is still alive are
  dismissed and tombstoned but not signalled. Their runner is running host-owned
  finalizers ([[decisions/host-owned-completion]]), which exit on their own (the
  provider stall watchdog covers hangs). A FAILED row with a live runner _is_ killed,
  because it may be in retry backoff.
- **The durable proc owns escalation and verification.** The TUI sends only the
  immediate SIGTERM, then removes rows optimistically. The persist-cleanup proc runs out
  of process, so it survives the TUI: it waits, escalates, sweeps, and verifies. It
  releases workspaces and deletes artifacts only after confirming death.
- **Where each piece lives.** The planner decision (which rows to kill versus dismiss)
  stays in the Rust `sase_core` planner, per the core-boundary rule. OS process
  mechanics stay in the existing shared, UI-free `sase.agent.user_kill` module, which
  the CLI and the TUI already share. Porting termination itself to `sase_core` is out of
  scope.
- **No feature flag.** Every phase fixes a bug and lands complete. There is no
  unfinished user-reaching behavior to gate.

## Phase `core-wire`: Rust cleanup wire for live runners and atomic dismissed index

Open the linked repo with `sase repo open sase-core` and work in the printed path. Read
its `AGENTS.md`.

In sase-core:

1. `crates/sase_core/src/agent_cleanup/wire.rs`:
   - add `runner_is_live: bool` (serde default `false`) to `AgentCleanupTargetWire`;
   - bump `AGENT_CLEANUP_WIRE_SCHEMA_VERSION` to 5.
2. `planner.rs` (`plan_agent_cleanup`): a target with `status == "FAILED"`, a pid, and
   `runner_is_live` is **killable and not dismissable**.
   - `KILL_AND_DISMISS` emits a kill item through `classify_kill_kind`, both for direct
     selection and for the parallel-member cascade.
   - `DISMISS_COMPLETED` skips it as `not_dismissable` with detail `runner_live`.
   - All other dismissable statuses keep today's dismiss semantics, even when live.
   - Add unit tests for:
     - FAILED+live → kill;
     - FAILED+not-live → dismiss;
     - DONE+live → dismiss;
     - the parallel cascade;
     - the schema-4 rejection.
3. `execution.rs`:
   - make `save_dismissed_agents_index` atomic (temp file in the same directory, fsync,
     rename);
   - add
     `update_dismissed_agents_index(path, additions, removals) -> Result<Vec<Identity>>`.
     It holds an exclusive advisory lock on a sibling `<file>.lock`, reads the current
     entries, applies the removals and then the additions, writes atomically, and
     returns the resulting set.
   - A missing file counts as empty. An unparsable file is renamed to
     `<file>.corrupt-<unix_ts>`, then treated as empty. Never silently overwrite it.
   - Accept both the legacy list-of-triples and the dict entry shapes that
     `load_dismissed_agents` accepts.
   - Tests: many threads, and ideally forked processes, adding disjoint identities
     concurrently must lose none; removals and additions compose; the corrupt file is
     preserved.
4. Expose the new function through the `sase_core_py` binding, following the existing
   `save_dismissed_agents_index` binding and the binding wire-version conventions.
5. Run `sase tool run check` inside the sase-core checkout.

In sase:

6. `src/sase/core/agent_cleanup_wire.py`:
   - set `AGENT_CLEANUP_WIRE_SCHEMA_VERSION = 5`;
   - add `runner_is_live: bool = False` to `AgentCleanupTargetWire`;
   - make sure `cleanup_targets_for_core` and `coerce_cleanup_target` carry it.
7. `src/sase/core/agent_cleanup_targets.py`: project
   `runner_is_live=bool(getattr(agent, "runner_is_live", False))`.
8. `src/sase/core/agent_cleanup_python.py` (reference/fallback planner): mirror the
   FAILED+live rule exactly, and extend the Rust/Python parity tests.
9. `src/sase/core/agent_cleanup_execution.py`: add `try_update_dismissed_agents_index`
   next to `try_save_dismissed_agents_index`. In `src/sase/ace/dismissed_agents.py` and
   `dismissed_agents_state.py`, add `add_dismissed_agents(identities)` and
   `remove_dismissed_agents(identities)` (returning the resulting set). Leave the
   callers unchanged; `additive-dismissals` adopts them.
10. Move the pin in `sase-core-revision.txt` past the sase-core commit
    (`just ratchet-core-revision`; see "The CI source revision pin" in
    `docs/rust_backend.md`). Adjust the `pyproject.toml` `sase-core-rs` range if the
    docs require it.

The focused-row `x` on a FAILED+live row now gets a kill item. It therefore goes through
`ConfirmKillModal` and `_do_kill_agent` with no TUI change.

## Phase `row-tombstones`: Session removal tombstones honored at every roster publication

Add `src/sase/ace/tui/actions/agents/_removal_tombstones.py`, a small UI-free helper
plus a mixin:

- **State:** a session `_explicit_removals` set of `AgentIdentity`, derived indexes
  (suffixes by cl_name), and `_agents_removal_generation: int`. Initialize them in
  `src/sase/ace/tui/actions/_state_init_agents.py`.
- **`record_explicit_removals(identities)`:** adds the identities and bumps the
  generation. It runs on the UI thread.
- **`is_explicitly_removed(agent, snapshot)`:** matches on the exact identity. It also
  matches the same `raw_suffix` with the same `cl_name`, or a row `cl_name` of
  `"unknown"`, regardless of `agent_type`. This mirrors the existing
  `dismissed_cl_suffixes` logic, so a killed WORKFLOW row that reappears as RUNNING
  after dedup is still caught.
- **`filter_explicitly_removed(agents)`:** filters, then re-runs `project_clan_tree`, so
  containers with no remaining members vanish.
- **`clear_explicit_removals(identities)`:** used by revive.

Record tombstones synchronously in **every** `x`-driven removal:

- `_do_kill_agent` and `_do_bulk_kill_agents` (`_kill_flow.py`);
- the single and bulk dismissal paths (`_dismissing.py`, `_dismiss_memory.py`);
- proc-shell dismissal (`_proc_shell_dismiss.py`);
- anything else that calls `_apply_killed_agents_in_memory` or the dismissal in-memory
  helpers.

The simplest robust hook is to record inside those two shared helpers. Revive paths
(`_revive_execution.py`, `_revive_archive.py`) clear tombstones for revived identities.

Honor tombstones at every publication point, **before** any `runner_is_live` exemption:

1. `compute_apply_loaded_agents` (`_loading_compute.py`): add a frozenset tombstone
   snapshot plus the generation to the worker snapshot (`_make_prepared_apply_snapshot`
   / `PreparedApplySnapshot`). Tombstoned rows are filtered out, and routed into
   `dismissed_agent_objects` so revive still sees them.
2. `merge_incomplete_load_after_complete_history` (`_loading_compute_merge.py`): check
   tombstones in `is_dismissed` before the `runner_is_live` early return.
3. Apply time (`_apply_loaded_agents_prepared_inner`, `_loading_apply.py`): if
   `_agents_removal_generation` moved since the worker snapshot, or the current
   `_dismissed_agents` gained identities, filter
   `boundary.fold.local_unfiltered_agents`. If anything was dropped, re-derive the
   unfiltered, visible, and fold counts through the existing `project_and_fold_rosters`
   path already used for `roster_moved`. Do not add a new refresh path (tui_perf rule
   5); this branch only runs when a removal raced the load.
4. `_reproject_agents_from_current_mode` (`_fleet_projection.py`): filter
   `_local_base_for_current_projection()` through the tombstones.
5. `_refilter_agents` (`_loading_filter.py`): filter the local source through the
   tombstones.

Also:

- `_apply_killed_agents_in_memory` (`_kill_identity.py`) and the dismissal in-memory
  helpers (`_dismiss_memory.py`, including `_apply_dismissal_in_memory_fast_finish`, and
  `_proc_shell_dismiss.py`) call `_sync_agents_local_source_from_current()` after
  mutating, so no stale local copy survives even without tombstones.
- `_repair_dismissed_projection` (`_revive_archive.py`):
  - compute on the worker, but apply the `_dismissed_agents` mutation on the UI thread
    (`call_from_thread`);
  - never drop an identity that is tombstoned, or whose artifacts or bundle still exist
    as a kill record. It may only _add_ identities found in bundles, plus drop entries
    whose agent no longer exists anywhere.
  - Keep the save off-thread.
- `_do_kill_agent`: collect per-target signal results. Successfully signalled targets
  are removed, tombstoned, and persisted. Only failed targets stay visible, with an
  error toast naming them. `on_settled` semantics stay unchanged.
- Keep all new work O(rows) with set lookups and no I/O on the UI thread (tui_perf rules
  1, 5, and 6).

Tests (extend the existing Agents-tab kill, dismiss, load, and fleet tests; find them
with
`rg -l "_apply_killed_agents_in_memory|_reproject_agents_from_current_mode|merge_incomplete_load_after_complete_history" tests`):

- a kill followed by `_reproject_agents_from_current_mode(source="fleet_refresh")` never
  resurrects the row (single, non-clan);
- a load prepared before the kill and applied after it does not resurrect the row;
- a `runner_is_live=True` row that was tombstoned stays hidden through compute and
  through the Tier-1 merge;
- an untombstoned, stale-dismissed live retry is still visible (keeps the `786cdb7f7`
  regression tests green);
- revive clears the tombstone;
- the revive repair keeps killed identities;
- a clan container disappears once every member is tombstoned;
- a partial signal failure removes the signalled targets.

## Phase `tree-kill`: Verified process-tree termination in the durable cleanup proc

1. In `src/sase/agent/user_kill.py` (the shared, UI-free kill primitive), add
   `terminate_agent_processes(pid, *, artifacts_dir, grace_seconds, ...) -> AgentTerminationResult`.

   **Discovery** reads `/proc` directly; no `ps` subprocess on hot paths. A process is a
   target if any of these holds:
   - its pgid is the runner pid;
   - its session id (sid) is the runner pid (session members outlive the leader);
   - it is a ppid descendant of the runner;
   - its `/proc/<pid>/environ` carries `SASE_LAUNCH_SCRATCH_KEY=<recorded key>`. Reuse
     or extract the environ probe from `src/sase/axe/run_agent_runner_scratch.py`. Other
     agents launched from this one get their own scratch key, so they never match.

   Pin every target with `process_identity_token`, and signal only while that identity
   still matches. Shield the current process and its ancestors. Discovered processes
   that are registered SASE supervisors (a proc supervisor or command whose pid or pgid
   matches a proc-store record, or a monitor supervisor) are stopped through their
   canonical stop (`stop_proc_shell` / `stop_monitor`), so store records settle instead
   of being orphaned as "running".

   **Sequence:**
   - verify the runner identity (existing `_target_identity_is_verified`);
   - discover the targets;
   - send SIGTERM to the group and to each pinned pid;
   - poll until the grace period ends. The default grace must be at least
     `TERM_ESCALATE_SECONDS + 1` (the `sase tool run` wrapper's own escalation window,
     `src/sase/tool/executor_process.py`) so wrappers can clean up their children first;
   - rediscover (catching late children), then SIGKILL the survivors;
   - verify, and record the result, including a `survivors` list, in the
     `.sase_user_kill_pending` marker.

   **Non-leader pid:** when `os.getpgid(pid) != pid`, never report `already_stopped`
   from a failed `killpg`. Signal the pid and its tree instead. After a
   `ProcessLookupError`, re-check with `os.kill(pid, 0)`.

2. Record the launch scratch key at launch. The runner writes `agent_meta.json` with
   `process_identity` (`src/sase/axe/run_agent_markers.py`); add
   `launch_scratch_key = os.environ.get(SASE_LAUNCH_SCRATCH_KEY_ENV)` there.
   `terminate_agent_processes` reads it next to the process identity. Older agents
   without it fall back to pgid, sid, and ppid discovery.
3. TUI immediate stage (`_kill_processes.py`): call
   `request_user_kill(..., wait=False, background=False)`, which only writes the marker
   and sends the SIGTERM. If `_submit_cleanup_proc` rejects the persistence proc, start
   the existing background-escalation thread as a fallback, so there is never a path
   with no escalation.
4. Durable stage (`_kill_transactions.py` / `_kill_persistence.py`, reached from
   `src/sase/ops/commands/_agent_cleanup.py`). Order matters:
   - first persist the dismissal and notification side effects (so other TUIs and
     restarts see it);
   - then run `terminate_agent_processes` for every kill item with a pid (not monitors);
   - **only after verified death** release workspace claims and delete workflow
     artifacts.

   **Safety net:** for every dismissed identity whose status is not success-terminal
   (see Design decisions) and whose recorded pid and process identity are still live,
   also run `terminate_agent_processes`.

   **Survivors:** the transaction reports failure with the survivor pids (severity
   error, via the existing `_CLEANUP_ERROR_RECOVERY` path), keeps the workspace claim,
   and leaves the rows dismissed.

5. The in-flight guards (`_kill_procs.py`, `_dismissing.py`) must never drop a batch.
   Exclude only the identities that are already in flight (or submit everything and rely
   on per-identity proc `concurrency_keys` to serialize). Every removed identity still
   gets persisted and terminated.
6. CLI parity: `sase agent kill` (`src/sase/agent/running.py`) and any other
   `request_user_kill(wait=True)` callers (`rg -n "request_user_kill" src`) use
   `terminate_agent_processes`.
7. Tests (real processes, `tmp_path`, short graces through parameters, no sleeps longer
   than needed), using a fake runner started with `start_new_session=True` that spawns:
   - a same-group child;
   - a child in its own process group, still in the runner's session;
   - a `setsid` grandchild that inherits the scratch key and is reparented to init after
     the runner dies;
   - a child that ignores SIGTERM.

   Assert that all of them are dead, that shielded pids are untouched, that an
   identity-mismatched pid is never signalled, and that a non-leader pid is handled.
   Also assert that the durable transaction releases the workspace only after
   verification and reports survivors otherwise.

## Phase `member-scope`: x stops every member kind instead of skipping it

1. Focused `x` (`action_kill_agent`):
   - **running monitor:** confirm with `ConfirmStopMonitorModal`, then route through the
     planner kill path (`_plan_focused_agent_cleanup` gives a `monitor` kill item) and
     `_do_kill_agent`. The durable proc stops it via `execute_monitor_stop_intents` and
     the row is removed at once;
   - **active proc shell:** confirm with `ConfirmKillProcShellModal`, then remove and
     tombstone optimistically. The durable cleanup proc stops it via `stop_proc_shell`
     and records the proc-shell dismissal (same store as `_dismiss_proc_shell_rows`);
   - **pending gate** (`gate_state == "pending"`): confirm "Cancel gate", then remove
     and tombstone optimistically. The durable proc calls `cancel_gate_shell`
     (`src/sase/gate_shell/cancel.py`). If cancel returns the record unchanged because
     the decision is already executing, the proc reports it and un-tombstones via the
     existing error-recovery refresh;
   - gates that are `settling` (already executing a decision) keep the current warning.
2. Clan, panel, group, and marked cleanup (`_present_bulk_kill_modal` in
   `_marking_kill.py`, `_do_bulk_kill_agents`): active proc shells and pending gates
   become "Kill N proc shells" and "Cancel N gates" lines instead of "Skipping ...".
   They are removed optimistically with the rest and executed in the same durable bulk
   transaction (add `proc_stop_ids` and `gate_cancel_ids` to the payload). After
   planning, any selected member not covered by a kill, dismiss, proc-stop, or
   gate-cancel item is listed in the modal ("N rows cannot be cleaned up: <reason>"), so
   a clan that stays visible is always explained.
3. Remote rows: `_agent_by_identity` (`_remote_lifecycle.py`) searches
   `_agents_with_children`, not just the visible `_agents`, so folded remote clan
   members are no longer dropped. Leave remote-row removal semantics unchanged (see
   Non-goals).
4. Update any user docs that describe `x` semantics
   (`rg -n "kill_agent|Skipping" docs`). The keybinding does not change, so
   `src/sase/default_config.yml` is untouched.
5. Tests:
   - focused `x` removes running-monitor, active-proc-shell, and pending-gate rows in
     one step;
   - a clan kill with a running proc shell and a pending gate removes the clan container
     and submits the stop and cancel intents;
   - the unexplained-leftover line appears;
   - folded remote members resolve.

   Modal text changes may move PNG goldens. Read the `tui_screenshot` memory and update
   goldens only through the documented flow.

## Phase `additive-dismissals`: Additive dismissed-index persistence for every writer

1. The persist-cleanup transactions (`_kill_transactions.py`, `_kill_persistence.py`,
   `_dismissing.py`, `src/sase/ops/commands/_agent_cleanup.py`) persist
   `add_dismissed_agents(<this batch's identities>)` instead of
   `save_dismissed_agents(<full snapshot>)`. The payload carries the batch identities
   (`added_identities`). The full `dismissed_identities` snapshot stays only for the
   artifact-index sync, if still needed.
2. Revive and un-dismiss (`_revive_execution.py`, `_revive_archive.py`, `_marking.py`)
   use `remove_dismissed_agents`. Runner and axe writers
   (`axe/run_agent_runner_lifecycle.py`, `agent/running.py`, `agent/names/_auto.py`,
   `agents_sync/purge_local_state.py`, `_loading_apply.py`, `_dismiss_memory.py`) use
   add or remove as their intent requires. Audit every remaining caller with
   `rg -n "save_dismissed_agents\(" src`.
3. Delete the in-process generation-stamp machinery in
   `src/sase/ace/dismissed_agents.py` (`snapshot_dismissed_agents` generations,
   `_last_saved_dismissed_generation`) once no full-snapshot writer needs it. Keep
   `snapshot_dismissed_agents` only if it is still used for pure copying.
4. `sync_dismissed_agent_artifact_index` consumes the set returned by the update call,
   so the index matches what is on disk.
5. Tests: two persist-cleanup payloads applied in either order leave the union on disk;
   a runner-lifecycle write racing a TUI dismissal loses neither; revive removes only
   its identities.

## Phase `e2e-regression`: End-to-end x regression coverage

Add a Textual-pilot integration test module for the Agents tab. Reuse the existing ACE
pilot fixtures (`rg -l "run_test\(" tests/ace/tui | head`). It uses on-disk fixture
agents in a temp `SASE_HOME` with real fixture process trees (the `tree-kill` fake
runner), and runs the durable cleanup payload in-process through
`apply_cleanup_payload_for_result`. Scenarios:

- `x` on a clan container while a load is between prep and apply, then a fleet refresh
  lands. The clan and all its member rows are gone after every subsequent refresh,
  including a forced complete-history load.
- `x` on a FAILED row whose runner is live (retry backoff): a kill confirmation, the row
  gone, the process tree dead, and no resurrection.
- `x` on a DONE row with a live finalizing runner: the row is gone and stays gone, and
  the runner is **not** signalled.
- `x` on a running agent, then the TUI app exits immediately: after the durable payload
  runs, every fixture process is dead and the workspace claim is released.
- Restart simulation: a fresh app instance loading the same on-disk state shows none of
  the removed rows.

Mark the tests appropriately if they are slow (follow the existing markers).

## Verification (every phase)

- Read the `lint_and_test` memory. Run `sase tool run check` in sase, and run it inside
  the sase-core checkout for `core-wire`. Do not run `check-full` unless explicitly
  instructed.
- Read the `tui` and `tui_perf` memories before touching TUI load, apply, refresh, or
  render code.

## Non-goals and follow-ups

- **Remote fleet rows.** Their `x` stop/removal semantics stay host-owned (the
  "stopping" overlay). Making remote rows disappear on `x` needs a federation-side
  design; propose it as a follow-up task in the phase notes.
- **Late successors.** A gate that was already `settling` when its clan was killed may
  still launch a successor, which appears as a new row. Killing late successors needs a
  clan-level kill intent honored by gate settlement; record it as a proposed follow-up.
- **Porting to `sase_core`.** Moving process termination itself into `sase_core` is a
  separate decision.
