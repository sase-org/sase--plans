---
tier: tale
title: Make the visible TUI surface win the startup window
goal:
  The initially visible TUI surface loads without deferrable startup contention, while
  every hidden surface and maintenance task still starts within a bounded delay.
size: medium
proposed_by: bbugyi200.athena.sase-132.2
bead: sase-132.2
status: done
---

- **PARENT:**
  [202609/tui_startup_regression.md](https://github.com/sase-org/sase--plans/blob/main/202609/tui_startup_regression.md)
- **BEAD:**
  [sase-132.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-132/sase-132.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-132.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-132.2.md)
- **COMMITS:**
  - [59c82a3](https://github.com/sase-org/sase/commit/59c82a36e85f282c10aaaeb51e5d30656686161c)
    — feat(tui): prioritize visible startup surface

# Make the visible TUI surface win the startup window

Implement phase bead `sase-132.2` from the approved
`plan:202609/tui_startup_regression.md` design. The baseline phase is already on this
tree (`8319c2240`) and its live athena capture found default-Agents startups at
7.18/7.85/11.11 seconds while the same busy-host bounded loader benchmark was 1.76
seconds p50. It also attributed one startup load as 4.98 seconds in
`agents.load_from_disk.dismissed_snapshot` out of 6.34 seconds total, so the sequencer
must remove competing startup work before assuming that substage needs new loader logic.

Before editing, re-read `sase/memory/tui_perf.md` and `sase/memory/tui.md` through
`sase memory read`, and inspect the current versions of the files below because sibling
phases may have landed. Preserve the established pump-free/coalesced refresh paths and
do not move shared loader behavior into Python: this phase is startup ordering and
presentation glue, not a loader/index redesign.

## Implementation

1. Add one explicit, idempotent startup coordinator around
   `StartupLoadsMixin._start_post_mount_background_loads` in
   `src/sase/ace/tui/actions/_startup_loads.py`, with its state initialized in the
   existing startup state modules and cleaned up on teardown as needed. Snapshot/use
   `_startup_initial_tab`, start the initially visible asynchronous surface first
   (`agents` by default, `axe` when requested), and release the deferred class exactly
   once when that visible surface becomes ready. Arm a short bounded fallback timer (a
   few seconds) whose synchronous callback only launches existing pump-free/worker
   paths; if the visible loader wedges or scheduling fails, the hidden surfaces and
   maintenance still start. Cancel or harmlessly retire the fallback after a normal
   release. For the synchronously composed Artifacts surface, release after first paint
   (while keeping any data needed for its initial usable view in the visible class).

2. Make the classification explicit in code instead of launching the present block
   concurrently:
   - Keep immediate: the stall watchdog/lifecycle protection already installed by
     `on_mount`, artifact and prompt-source watchers whose early events must not be
     missed, notification-state seeding/poll setup, and small state that the selected
     visible surface genuinely needs (including Agents fold state for an initially
     visible Agents tab and the initial Artifacts patch state when Artifacts is
     selected).
   - Visible surface: only the selected tab's first meaningful load and render. Keep the
     stale-agent-index bounded-first-paint behavior and all existing refresh
     pending/on-complete guards intact.
   - Deferred until visible-ready or fallback: the hidden surface load (notably axe init
     on the default Agents startup), relation/link-index construction, prompt catalog
     warm, update-toast check, usage refresh, dismissed proc-shell prune,
     dismissed-index sync/maintenance, proc and agent-monitor reconcile, and post-roster
     bead/family-preview/live-hint/diff-badge warmups. Split the current mount-state
     worker if necessary so unread notification state is not held behind unrelated
     prompt-stash, Patch, saved-selection, startup-query, or development-version reads.
     Preserve `_mount_state_loads_done` as an accurate completion signal for the whole
     mount-state load.

   Route every item through its existing scheduler/coalescing guard; do not add another
   refresh implementation. A user switching tabs during startup may request a hidden
   surface early through the established tab-switch path, but the coordinator must still
   avoid duplicate first loads and must eventually converge all surfaces.

3. Use the existing visible-ready source of truth, but fix its checkpointing if needed:
   `visible_ready_seconds` must be stamped when the initially visible surface is ready,
   not delayed until both Agents and axe are ready. Have that transition end the trace
   startup window/stopwatch and release deferred work. Continue recording
   `all_surfaces_ready_seconds` only after both asynchronous surfaces settle, and do not
   change the meaning or names of existing telemetry fields. Fallback release must not
   falsely mark the visible surface ready.

4. Remove duplicated startup work observed by the baseline:
   - Move the eager mount `_schedule_link_index_refresh(source="mount")` behind the
     coordinator and rely on `LinkSubjectMixin`'s loading/pending guard so
     `relations.index.patches` is built once per startup even if selection/rail
     availability asks for it at the same time.
   - In the initial roster apply, avoid calling `_update_agents_info_panel()` once per
     row patched by the incremental batch and again at batch completion. Add a narrowly
     scoped option/helper so standalone row patches still refresh the info panel, while
     the batched loop suppresses per-row refreshes and performs one final update.
   - Keep the initial Agent detail/prompt panel in loading/placeholder state until the
     coherent roster and selection have been applied. Cancel/suppress any pre-roster
     debounced full render and schedule one normal debounced full detail render
     afterward; preserve immediate highlight/header behavior for real post-startup
     navigation (tui_perf rule 7).

5. Add focused regression tests, adapting the current contrary concurrency test in
   `tests/ace/tui/test_startup_stopwatch_live_update.py` and the dismissed startup/index
   tests rather than layering incompatible expectations:
   - With a fake slow default Agents loader, axe and every deferred-class task stay
     unstarted until Agents ready, then all run; no task starts twice.
   - With visible-ready never arriving, the fallback starts every deferred task without
     marking visible-ready and without starving `all_surfaces_ready` once the loads
     later finish.
   - Initial axe and Artifacts modes choose the correct visible prerequisite, and a
     startup tab switch does not duplicate either load.
   - Relation/link indexing is scheduled/built once per startup despite a concurrent
     rail/selection request.
   - A multi-row initial roster apply performs one info-panel refresh and no full
     AgentDetail/AgentPromptPanel render before roster application, followed by one
     coherent debounced render.
   - Startup telemetry records visible-ready at the visible transition while
     all-surfaces-ready remains gated on both loads. Keep the baseline substage,
     startup-window, stale-index, and dismissed-index tests green.

## Verification and completion

Run the narrow startup/TUI tests first, including at least:

```bash
.venv/bin/pytest -q \
  tests/ace/tui/test_startup_stopwatch_live_update.py \
  tests/ace/tui/actions/test_startup_telemetry.py \
  tests/ace/tui/actions/test_startup_observability.py \
  tests/ace/tui/test_agent_index_schema_startup.py \
  tests/ace/tui/test_dismissed_index_startup_sync.py
```

Add any new focused test module to that invocation. Then follow
`sase/memory/lint_and_test.md`: run `just fix` (or at minimum `just fmt`) and
`just check`, using `/sase_monitor` if a check becomes long-running.

Use the documented `tests/perf/capture_tui_startup.py` / `docs/perf_runbook.md` recipe
on athena at the exact deployed SHA. Compare at least three traced default-Agents starts
against the same-day `bench_agent_load_tiering.py --sase-home ~/.sase` p50, verify no
deferred span begins before visible-ready (unless tagged as fallback), ensure
`relations.index.patches` occurs once, and confirm every deferred surface reaches
`all_surfaces_ready` without starvation. Target startup `agents.load_from_disk` within
1.5x of the same-day standalone `production_bounded` p50 and a visible-ready median
improvement of at least two seconds; if host load prevents decisive live acceptance,
record the exact SHA, host state, measurements, and an attributed `PROPOSED FOLLOW-UP:`
note rather than weakening the target.

Before closing, run `sase bead epic-symbols sase-132.2` and resolve every entry or
re-key its Justfile annotation to `sase-132` or a still-open later phase. Do not create
task beads and do not close the parent epic. Record any discovered out-of-scope work
with `sase bead note sase-132.2 'PROPOSED FOLLOW-UP: <summary — detail>'`, then close
only this phase with
`sase bead close sase-132.2 --note "<tests, checks, deployed SHA, and telemetry verified>"`.
