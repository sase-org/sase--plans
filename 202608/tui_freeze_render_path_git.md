---
tier: epic
title: Stop TUI freezes caused by workspace materialization on the render path
goal: 'Navigating and viewing agents in the ACE TUI never blocks the UI thread on
  git clone/fetch/push, SDD store locks, or registry writes; multi-second random freezes
  stop, and any freeze that does occur is captured by the stall watchdog.

  '
phases:
- id: probe
  title: Read-only workspace directory probe API
  depends_on: []
  size: medium
  description: 'probe: add a side-effect-free workspace path resolver to workspace_provider
    so callers can ask where a workspace is without materializing it.'
- id: hints
  title: Convert agent-detail hint rendering to the probe API
  depends_on:
  - probe
  size: medium
  description: 'hints: make the prompt-panel hint and clan/family render paths resolve
    workspace dirs read-only, removing git subprocesses from the UI thread.'
- id: sweep
  title: Audit remaining TUI resolution call sites
  depends_on:
  - probe
  size: medium
  description: 'sweep: find every other keystroke, render, and completion path that
    materializes a workspace and move it to a read-only probe or a tracked background
    task.'
- id: guard
  title: Regression guard against materialization on the UI thread
  depends_on:
  - hints
  - sweep
  size: small
  description: 'guard: add a runtime guard plus tests so a future change cannot reintroduce
    clone/fetch/push work into a render or keystroke path.'
- id: watchdog
  title: Close the stall-watchdog forensics gap
  depends_on: []
  size: small
  description: 'watchdog: explain and fix why a reported multi-second freeze produced
    no record, so future freezes are always diagnosable.'
- id: loader
  title: Reduce agents-tab reload cost
  depends_on: []
  size: large
  description: 'loader: cut the full agents reload disk stage down from multi-second
    so post-launch and auto-refresh updates stop reading as unresponsiveness.'
proposed_by: bbugyi200.athena.rr
create_time: 2026-08-02 09:05:43
status: wip
bead_id: sase-e4
---

- **PROMPT:** [prompts/202608/tui_freeze_render_path_git.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/tui_freeze_render_path_git.md)
- **BEAD:** [sase-e4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-e4/README.md)

# Plan: Stop TUI freezes caused by workspace materialization on the render path

## Problem

The ACE TUI randomly freezes for several seconds at a time while the user is on the Agents tab. The freezes are not
correlated with any single user action and feel arbitrary, which is why they read as "random".

## Diagnosis

### Primary root cause: the render path materializes workspaces

`resolve_agent_workspace_dir()` in `src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py` is a **render-path
helper** (it is `lru_cache`d and called while building detail-panel text). At line 119 it calls
`sase.workspace_provider.get_workspace_directory()`.

Despite the read-only-sounding name, that function is a **materializer**. It routes through
`_plugin_manager.get_workspace_directory()` into the `ws_get_workspace_directory` hook. The bare-git plugin
implementation (`src/sase/workspace_provider/plugins/bare_git_workspace.py:156`) does:

```python
ensure_bare_git_sdd_initialized(
    primary_workspace_dir, commit=True, push=True, raise_on_error=True,
)
return ensure_workspace_checkout(primary_workspace_dir, workspace_num)
```

and `ensure_workspace_checkout()` (`src/sase/workspace_provider/utils.py:368`) in turn performs:

1. `ensure_git_clone_at(...)` — a full `git clone` when the checkout is absent.
2. `_record_managed_workspace(...)` — registry and checkout-marker **writes**.
3. `ensure_workspace_sdd_clone(...)` → `sdd/_store_link.py:_pull_sdd_clone` → a network `git fetch`, wrapped in
   `run_with_git_lock_retry` (which sleeps) and an SDD store write lock.

So painting an agent's detail panel can clone repos, fetch from the network, and even commit and push — **synchronously
on the Textual event loop**. This violates the established TUI rules that keystroke paths stay read-only and
prompt-free, that render paths never stat/glob or run subprocesses, and that the event loop is never blocked on
subprocess work.

The freezes look random because the work only fires when the selected agent's workspace or SDD sidecar actually needs
materializing or its fetch cooldown has expired.

### Evidence

From `~/.sase/logs/tui_stalls.jsonl` (the always-on watchdog), captured main-thread stacks on 2026-08-02 at 06:06:33 and
07:03:47, both `tab=agents`:

```
ace/tui/actions/hints/_files.py:198        _render_agent_hint_document
ace/tui/widgets/agent_detail.py:380        update_display_with_hints
ace/tui/widgets/prompt_panel/_agent_display_hints.py:745  _update_clan_display_with_hints
ace/tui/widgets/prompt_panel/_agent_display_clan.py:389   _clan_hint_workspace
ace/tui/widgets/prompt_panel/_file_path_hints.py:119      resolve_agent_workspace_dir
workspace_provider/_registry.py:288        get_workspace_directory
workspace_provider/_plugin_manager.py:168  get_workspace_directory
workspace_provider/utils.py:375            ensure_workspace_checkout
sdd/store.py:165                           ensure_workspace_sdd_clone
sdd/_store_link.py:319                     _pull_sdd_clone
sdd/_repository_transaction.py:108         integrate_machine_managed_sdd_repository
sdd/_git_contention.py:97                  run_sdd_git_write
git_lock_retry.py:130                      run_with_git_lock_retry
sdd/_git.py:49                             run_sdd_git
```

A second variant of the same stack ends in `workspace_provider/utils.py:242 ensure_git_clone_at`. Recorded stall
durations on this path range from 1.5 s hitches to 5.3–8.0 s full stalls.

Corroborating logs:

- `~/.sase/logs/tui.log` contains 59 `Failed to pull workspace SDD clone ...` warnings emitted **by the TUI process
  itself**, plus `Slow SDD store write lock acquisition: waited 144.518s ... during sdd.clone.transaction`. The TUI is
  contending for SDD git locks against the concurrently running agents.
- `~/.sase/logs/tui_git_ops.jsonl` records `sdd.clone.remote` operations taking 5–24 s and `sdd.clone.fetch` around 0.5
  s.

### Secondary contributor: slow agents reload

`~/.sase/logs/tui_agent_loads.jsonl` holds 3,049 `tui_agent_load_slow` records since 2026-07-17 (the record is only
written when a stage exceeds 2.0 s). The `disk` stage measures p50 2.62 s, p95 9.45 s, max 166.7 s, every day for weeks.

This one does **not** block the UI thread, and that was verified rather than assumed:

- Both refresh entrypoints (`actions/agents/_loading_refresh.py:171` `_spawn_agents_refresh_task` and
  `actions/event_refresh/_auto_refresh.py:89`) correctly use `spawn_pump_free_task`, so the load is off the loop and off
  the message pump.
- Profiling the loader standalone showed 0.75–1.47 s, dominated by a single `sase_core_rs.query_agent_artifact_index`
  call (0.361 s) plus ~14.3 k `lstat` calls.
- Running that loader in a worker thread while a 5 ms heartbeat thread emulated the UI loop produced a max UI-thread gap
  of **79 ms** — so the Rust binding releases the GIL correctly and does not starve the main thread.

It still matters: a launch at 08:45:40 followed by its post-launch reload finishing at 08:45:46 leaves roughly six
seconds where the newly launched agent is not yet visible. That reads as unresponsiveness even though the pump is alive.

### Observability gap

The freeze the user reported at approximately 08:45 produced **no record at all** in `tui_stalls.jsonl`. The watchdog
thread was confirmed alive in the running process, and no `SASE_TUI_STALL_*` disable variable was set. Every stall
record in the file belongs to earlier, now-dead processes. A freeze that leaves no forensic trace is itself a defect
worth closing.

## Scope note

`ensure_workspace_checkout()` is the shared choke point for every workspace plugin, so fixing it covers both the
bare-git and the GitHub provider paths. The GitHub provider lives in the linked `sase-github` repo; if it defines its
own `ws_get_workspace_directory` hook, it needs a matching read-only hook implementation. Open that repo with the
`/sase_repo` skill before touching it.

Per the Rust core backend boundary rule, consider whether workspace path resolution semantics belong in `sase_core`.
Path _computation_ is a good candidate; the TUI-side wiring stays here.

---

## Phase: probe

Add a genuinely read-only way to ask where a workspace lives.

- Add a `ws_probe_workspace_directory` hookspec alongside `ws_get_workspace_directory` in
  `src/sase/workspace_provider/_hookspec.py`, and a `probe_workspace_directory()` entrypoint in
  `src/sase/workspace_provider/_registry.py`.
- Implement it for the bare-git plugin using `WorkspaceStore.resolve()` (`src/sase/workspace_provider/store.py:330`),
  which is already pure path computation and performs no I/O. The probe must:
  - never clone, fetch, push, or commit;
  - never write the workspace registry or a checkout marker;
  - never initialize or pull an SDD sidecar;
  - never take an SDD store lock;
  - perform at most a single `os.path.isdir()` existence check and return `None` when the checkout is not materialized.
- Document the split explicitly in both docstrings: `get_workspace_directory` **materializes**,
  `probe_workspace_directory` **observes**. The current name is the trap that caused this bug.
- Unit tests: assert the probe returns the same path the materializing call would return for an already-materialized
  workspace, returns `None` for an absent one, and — importantly — assert it issues no subprocess. Patch the git runner
  and assert zero calls.

## Phase: hints

Move the agent-detail render path onto the probe.

- Change `resolve_agent_workspace_dir()` (`src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py:79`) to call
  `probe_workspace_directory()` instead of `get_workspace_directory()`. It already falls back to the explicit
  `workspace_dir` argument and already returns `None` gracefully, so an unmaterialized workspace degrades to the
  existing fallback rather than to a stall.
- Verify the render-path callers keep working: `_agent_display_clan.py` (lines 389, 412, 435),
  `_agent_display_hint_render.py:138`, `_agent_display_family_render.py:239`, and `_agent_clan_aggregation.py` (lines
  288, 344).
- Keep the existing bounded `lru_cache` and `clear_file_hint_resolution_caches()` behavior intact.
- Add a regression test that renders an agent detail panel for an agent whose workspace is not materialized and asserts
  that no git subprocess runs and no SDD clone is attempted.

## Phase: sweep

The same trap may exist on other interactive paths.

- Audit every TUI call site that reaches `get_workspace_directory` or `resolve_agent_workspace_dir`, including
  `ace/tui/actions/agents/_panel_tmux.py` (lines 162–178), `ace/revert_agent_resolution.py:48`,
  `ace/tui/actions/agent_workflow/_ref_resolution.py:72`, and `ace/tui/thinking/session_resolver.py:74`.
- For each, classify it as render/keystroke (must use the probe) or user-initiated action (may materialize, but must run
  as a tracked background task via `_submit_tracked_task()` / `_submit_background_task()` in
  `ace/tui/actions/task_actions.py`, never inline on the loop).
- Convert accordingly. Where an action legitimately materializes, give it optimistic UI first and surface failure as a
  toast rather than a freeze.
- Re-sweep `call_later`, `call_after_refresh`, `set_timer`, and `set_interval` callbacks for any slow await introduced
  by the conversion.

## Phase: guard

Make the regression impossible to reintroduce silently.

- Add a debug-mode guard that raises (or logs loudly) when `ensure_workspace_checkout()`,
  `ensure_workspace_sdd_clone()`, or the SDD git runner is entered on the Textual UI thread. A thread-identity check set
  at app startup is sufficient; keep it cheap enough to leave on.
- Add a test that exercises the agent-detail render and hint paths with the git runner patched to fail loudly, asserting
  the paths complete without invoking it.
- Confirm the guard fires on a deliberately reintroduced regression, so the test is proven to have teeth.

## Phase: watchdog

Explain and close the forensics gap.

- Determine why the reported freeze produced no record. Check specifically whether `_pause_depth` in
  `src/sase/ace/tui/util/_stall_watchdog_monitor.py` can leak above zero: an `app.suspend()` that publishes
  `app_suspend_signal` but never publishes `app_resume_signal` (an exception inside the suspend block, or a suspend
  raised from a path that bypasses `suspend_for_external_tool`) would pause detection permanently for the rest of the
  session. There are roughly 20 direct `app.suspend()` / `self.suspend()` call sites outside the helper.
- If that is the cause, make pause strictly scoped (context manager with a `finally:` resume) and/or add a maximum pause
  duration after which the watchdog self-resumes.
- Emit a low-frequency `tui_watchdog_heartbeat` record including the current pause depth, so a silent window is
  distinguishable from a healthy one.
- Verify by lowering `SASE_TUI_STALL_*` thresholds and confirming an induced freeze is captured both before and after a
  suspend cycle.

## Phase: loader

Reduce the cost of a full agents reload. This phase should produce its own plan before implementing.

Measured starting point on the reporter's machine: `disk` stage p50 2.62 s, p95 9.45 s, max 166.7 s across 3,049
recorded slow loads; 142 agents loaded.

Known cost drivers to investigate:

- `sase_core_rs.query_agent_artifact_index` costs 0.361 s against an ~80 MB `agent_artifact_index.sqlite`. Consider
  narrowing the query, incremental reads, or pushing projection into `sase_core`.
- The dismissed-agents set holds **20,694** identities and is copied (`set(self._dismissed_agents)`) on every load and
  passed through the worker boundary. Consider pruning, compacting, or sharing an immutable snapshot.
- Roughly 14.3 k `lstat` calls and 23.9 k `Path` constructions per load suggest per-agent filesystem probing that could
  be cached by mtime.
- The 166.7 s outlier and the 144.5 s SDD store write-lock wait indicate the loader is also blocked by lock contention
  with concurrently running agents; bound those waits and degrade gracefully.

Treat the existing tiered/artifact-delta fast paths as the intended design and extend them rather than adding a new
refresh path. Per the Rust core backend boundary rule, projection and index-query work likely belongs in `sase_core`.

## Verification

- Reproduce before the fix by selecting an agent whose workspace is not materialized and confirming a multi-second stall
  lands in `~/.sase/logs/tui_stalls.jsonl` with the stack above.
- After the fix, confirm the same navigation produces no stall record and no new `sdd.clone.*` entries in
  `~/.sase/logs/tui_git_ops.jsonl` attributable to the TUI process.
- Confirm `~/.sase/logs/tui.log` stops accumulating `Failed to pull workspace SDD clone` warnings from the TUI process.
- Run `just check`. Capture j/k latency with `SASE_TUI_PERF=1` and confirm p95 stays under 16 ms on the Agents tab.
