---
tier: epic
title: Make the default parallel test suite reliable under host contention
goal: 'A full `just test` / `just check` run stops failing single unrelated nodes
  under xdist and host contention, the flake class is reproducible on demand through
  a committed deterministic harness, and the harness runs as a gate so newly written
  contention-fragile tests fail immediately instead of intercepting an unrelated agent''s
  required check weeks later.

  '
phases:
- id: harness
  title: Deterministic contention model and reproduction lane
  depends_on: []
  size: medium
  description: 'harness: add the committed pytest contention model that pins Textual''s
    process_time idle probe and delays off-loop work, expose it through a run_pytest
    mode plus a Justfile recipe, and record the pre-fix baseline of failing nodes.

    '
- id: settle
  title: Contention-proof settle for Textual pilot pauses
  depends_on:
  - harness
  size: medium
  description: 'settle: replace the CPU-heuristic idle wait behind pilot.pause() with
    a quiescence-signal settle that drains message pumps, Textual workers, and registered
    pump-free tasks, backed by a process-global pump-free task registry.

    '
- id: tui_residual
  title: Residual ACE TUI nodes that settle cannot reach
  depends_on:
  - settle
  size: medium
  description: 'tui_residual: convert the remaining pause-then-assert ACE TUI tests
    that still fail under the contention model into event-driven waits, without weakening
    any assertion.

    '
- id: deadlines
  title: Wall-clock deadline assertions
  depends_on: []
  size: medium
  description: 'deadlines: remove fixed real-time budgets from the prompt-catalog
    heartbeat, stall-watchdog state-machine, and pending-question marker tests by
    asserting causal ordering or driving an injected clock instead.

    '
- id: shared_state
  title: Load-sensitive and order-sensitive non-pump nodes
  depends_on:
  - harness
  size: medium
  description: 'shared_state: diagnose and fix the reported flakes that never touch
    the Textual pump - the xprompt VCS-tag selector identity cache, the TaskMirror
    detached count, the artifact-file facade reclaim and VCS cache nodes, and the
    custom-gate tracked executor.

    '
- id: guard
  title: Gate the harness and soak the full suite
  depends_on:
  - settle
  - tui_residual
  - deadlines
  - shared_state
  size: medium
  description: 'guard: wire the contention lane into CI, confirm the visual contention
    lane is clean, soak the full parallel suite, and close the umbrella bead with
    the measured evidence.'
proposed_by: bbugyi200.athena.tg
create_time: 2026-08-05 18:13:51
status: wip
bead_id: sase-fd
---

- **PROMPT:** [prompts/202608/parallel_suite_contention_reliability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/parallel_suite_contention_reliability.md)
- **BEAD:** [sase-fd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fd/README.md)

# Plan: Make the default parallel test suite reliable under host contention

## Problem

`sase-ct` is the umbrella bead for a flake class that has been reported at least sixteen times by independent agents
(twelve sibling beads were consolidated into it during the 2026-08-05 backlog triage). Every report has the same shape:
a full four-to-28-worker `just test` / `just check` run fails exactly one or two nodes after ~25,800 passes, and the
exact node passes immediately in isolation. The failures land on unrelated agents' required gates, forcing each of them
to re-run and hand-adjudicate a failure they did not cause.

### Root cause, confirmed empirically

Textual's `Pilot.pause()` (`textual/pilot.py`, Textual 8.0.1) delegates to `textual._wait.wait_for_idle(0)`, which
decides the app is idle by comparing `time.process_time()` against wall clock:

```python
if elapsed_time > min_sleep and cpu_elapsed < SLEEP_IDLE:
    break
```

With `min_sleep=0`, one 20 ms probe is enough to break out whenever the process burned less than 1 ms of CPU during that
window. That is precisely the state of a pytest worker on a saturated host, and it is also the state of any app that is
waiting on `asyncio.to_thread` or a Textual thread worker. So `pilot.pause()` returns _earliest_ exactly when the app
most needs settling, and the following `assert` reads state that has not landed yet. This inverted behavior is the
shared mechanism behind the ACE TUI half of the bead.

This was verified during planning with a two-effect model that needs no CPU hog:

1. pin `textual._wait.process_time` to a constant, so the first probe always breaks out while the real 20 ms wall sleep
   (and therefore every genuine Textual timer) is preserved;
2. put a fixed 50 ms delay in front of `asyncio.to_thread`, modelling off-loop work that finishes in microseconds on an
   idle host but not within one probe window under contention.

Under that model, `tests/ace/tui/test_agent_bulk_kill_edit.py::test_bulk_waiting_agents_mount_forced_artifact_prompts`
fails deterministically in 0.76 s with exactly the symptom `sase-ek.land` reported on 2026-08-03 — `app.screen` still
`Screen(id='_default')` instead of `ConfirmKillAllModal`. Seven of the thirteen distinct reported nodes reproduce under
the pump-starvation effect (measured with the first, harsher variant across the whole `tests/ace/tui` tree); the three
spot-checked against the calibrated model all reproduce there too. A prototype settle — message pumps drained, pending
Textual workers awaited, registered pump-free tasks awaited — makes them pass again under the same model.

The remaining reported nodes do not touch the pump and need separate diagnosis: they are fixed real-time budgets (a 50
ms `call_soon` heartbeat, 30/80 ms watchdog thresholds, a 10 s marker poll) and process-global state that leaks between
test files, which `--dist=worksteal` regroups nondeterministically across workers on every run.

## Constraints for every phase

- Run `just install` before anything else: these workspaces are ephemeral and may be stale.
- Run `just check` before completing, per the repo's agent instructions.
- **Never weaken an assertion to make a test pass.** Every node named here asserts real product behavior. Making a test
  deterministic means removing its dependence on scheduling luck, not removing its coverage. If a node's assertion turns
  out to describe a genuine product race, fix the product and say so.
- **Do not edit `sase/memory/*.md`, `AGENTS.md`, or any generated provider shim** (`CLAUDE.md`, `GEMINI.md`,
  `OPENCODE.md`, `QWEN.md`). This plan does not carry permission for memory edits. Documentation belongs in
  `CONTRIBUTING.md` or `docs/`.
- Shared backend behavior belongs in the Rust core. If a fix here reaches into task-store or artifact-file core logic,
  open `../sase-core` through the `/sase_repo` skill and change the Rust wire, bindings, and tests there, then update
  the Python adapter.
- The suite gate governs xdist worker tokens from a host-global pool. When sibling workspaces hold the pool, use
  `SASE_TEST_GATE_DISABLED=1` with a modest explicit `-n` for exploratory runs, and never for the final verification
  run.

## Harness

Build the deterministic contention model as committed test infrastructure, because every later phase needs it and
because "reproduce it" has been the blocking step on all twelve consolidated beads.

Add `tests/_contention_model.py` implementing the two effects described above as a pytest plugin:

- pin `textual._wait.process_time` to a constant so `wait_for_idle` breaks out of its first probe, leaving the
  `SLEEP_GRANULARITY` wall sleep intact so real Textual timers still fire;
- wrap `asyncio.to_thread` with a configurable delay (default 50 ms) so off-loop work reliably outlives one probe
  window.

Keep both effects individually switchable, because they diagnose different failures, and make the delay tunable through
an environment variable so a phase agent can find the threshold at which a given node flips.

Wire it in as an opt-in, default-off pytest option registered in `tests/conftest.py` (for example
`--sase-contention-model`), so a plain `just test` is unaffected. Add a `contention` mode to `tools/run_pytest`
alongside `fast` / `slow` / `visual` / `cov`, and a `just test-contention *args` recipe modelled on the existing
`test-visual-contention` recipe, including its habit of recording measured baselines in the recipe comment.

The model also needs a hang guard. A first, over-aggressive variant that shrank the probe sleep to zero produced **170
failing nodes across ~60 files** in `tests/ace/tui` — mostly artifacts, because removing the wall sleep also starves
legitimate Textual timers. The calibrated two-effect model above is far cleaner: **24 failures** over the same tree at
six workers, which is a believable fragility surface. But that run also wedged one node at 99% and never finished, so
give the lane a per-test timeout (`pytest-timeout`, or an equivalent budget) rather than letting a blocked `to_thread`
loop hang the suite. Identifying the wedged node is itself a finding for `tui_residual`.

Record the calibrated baseline node list in the phase bead note along with the exact command. Do not commit the list as
a golden file — it would churn on every unrelated test addition.

Deliberately out of scope for this phase: fixing anything. This phase ships the instrument.

## Settle

Replace the CPU-heuristic wait behind `pilot.pause()` with one that waits on real quiescence signals.

Add `src/sase/ace/testing/settle.py` next to the existing `ace_page.py` test-support module, so plugin repos that drive
ACE (`sase-github`, `sase-telegram`, `sase-nvim`) get the same behavior. It should expose a `settle_app(app, *, budget)`
that loops until all of the following hold, or the wall-clock budget expires:

- the current screen barrier completes (`Pilot._wait_for_screen` posts a callback to every widget in the screen tree and
  waits for all of them; repeat it, because a screen pushed from a callback that had not yet run is not the screen the
  previous barrier walked);
- every message pump reachable from the app and its screen stack has an empty `_message_queue` and no queued
  `_next_callbacks`;
- no Textual worker is `PENDING` or `RUNNING`;
- no registered pump-free task is still pending.

Finding the pump-free tasks needs a source change. `spawn_pump_free_task` in `src/sase/ace/tui/util/pump_tasks.py`
currently only records tasks on the owner object, and owners are not always reachable from the DOM, and the thirteen
distinct `registry_attr` names make per-registry draining unattractive. Add a process-global `WeakSet` of live tasks
(pruned by the existing done callback) so the settle can enumerate them without walking the widget tree. All 41 spawn
sites are bounded coroutines today — the one `while True` among them, the Admin Center tab save loop, drains its pending
queue and breaks — so a blanket drain is safe; still bound each wait so a future long-running task cannot hang the
suite, and consider an explicit `transient=False` opt-out for any task that is intentionally unbounded.

Install it through an autouse fixture in `tests/conftest.py` that overrides `Pilot.pause` for `delay is None`,
preserving the explicit-delay path and the trailing `self.app.screen._on_timer_update()`. Route `AcePage.pause` and the
`AcePage.wait_for` / `expect_*` poll loops through the same helper. Overriding `Pilot.pause` globally is the point:
there are hundreds of call sites, and the override protects tests nobody has written yet.

The budget must be generous (five seconds is a reasonable start) but the settle must return immediately when nothing is
pending, so an idle-host `just test` does not get slower. Measure and report the wall-clock delta on the full
`just test` run; a noticeable regression means the loop is waiting when it should not be.

Risk to watch: a settle that waits for workers could mask a genuine responsiveness bug in a test that meant `pause()` as
"one tick". Tests that assert non-blocking behavior use explicit deadlines rather than bare `pause()`, so this should
not bite, but re-read any test whose meaning changes.

Acceptance: the harness baseline from the previous phase drops substantially, `just test` runtime is unchanged, and no
assertion anywhere was weakened.

## Tui_residual

Re-run the harness after `settle` lands and fix what still fails. Against the calibrated model the pre-fix surface is 24
nodes plus one that wedges, so this phase is bounded; re-baseline first rather than assuming that exact set. (For
reference, a prototype settle cut the over-aggressive zero-probe baseline from 170 to 107, but most of that residue was
model artifact, not fragility.)

The reported nodes that must end up green under the contention model:

- `tests/ace/tui/modals/test_artifact_files_modal_copy.py` — the whole copy family. Only two of its thirteen tests call
  the file's own `_drain_clipboard_tasks` helper; the rest, including
  `test_artifact_file_modal_Y_anchors_path_recovered_from_agent_meta_json` (sase-ct) and
  `test_artifact_file_modal_copy_palette_copies_every_single_representation` (sase-cu +1), assert on `copied` after
  `pilot.pause()` and sometimes after the app has already shut down. Once settle drains pump-free clipboard tasks the
  local helper becomes redundant; remove it rather than leaving two mechanisms.
- `tests/ace/tui/test_agent_bulk_kill_edit.py::test_bulk_waiting_agents_mount_forced_artifact_prompts` — the confirm
  modal is pushed from an `asyncio.to_thread` inside a Textual worker (`schedule_relaunch_prompt_resolution`), so it
  needs the worker-awaiting arm of settle.
- `tests/ace/tui/test_agent_metadata_search.py` — `test_inline_metadata_search_commit_repeat_q_and_passthrough` and
  `test_inline_metadata_search_reverse_key_override`.
- `tests/ace/tui/widgets/test_prompt_at_prefix_completion.py::TestAtPrefixIntegration::test_at_prefix_directory_drilldown`.
- `tests/ace/tui/test_plugin_action_confirm_modal.py::test_plugin_action_modal_scrolls_overflowing_preview` — asserts
  exact `scroll_y` after `press` + `pause`; check whether scroll animation, not settling, owns the residue, and disable
  animation for the assertion if so.
- `tests/ace/tui/widgets/test_prompt_xprompt_highlight.py::test_xprompt_highlight_overlay_marks_spans_and_registers_styles`
  — reported by `sase-em.land`; it does not reproduce under the pump model, so find its actual dependency (theme
  registration ordering is the first place to look) rather than adding a pause.

For everything else the harness surfaces, prefer converting `pause(); assert X` into an event-driven wait on `X` over
adding more pauses. Where a test is fragile because the _product_ offers no signal that the work landed, adding that
signal is the better fix.

## Deadlines

Three reported nodes encode a fixed real-time budget and fail when the host cannot meet it. None of them reproduced
under CPU oversubscription during planning (24 spinners pinned to a shared core, 4-6 repeats each), so treat
reproduction as best-effort and fix the design rather than chasing the race.

- `tests/ace/tui/test_prompt_catalog.py::test_catalog_loading_worker_does_not_block_event_loop` (sase-e5) gives a
  `call_soon` heartbeat 50 ms while every xdist worker is saturated. The guarantee worth keeping is that catalog
  construction never runs on the Textual event loop. Assert that causally — the heartbeat must fire while the build
  thread is still blocked inside `build_prompt_catalog_snapshot`, which is already gated by a `threading.Event` in the
  test — and give the wait a generous timeout.
- `tests/ace/tui/util/test_stall_watchdog.py::test_watchdog_keeps_hitch_and_stall_state_machines_independent` (sase-cg,
  three independent recurrences) drives a 30 ms hitch threshold and an 80 ms stall threshold with a real
  `time.sleep(0.14)`. Drive the watchdog through an injectable time source so the hitch-before-stall ordering and the
  one-of-each event counts are exact rather than probable.
- `tests/test_axe_run_agent_helpers_questions.py::test_pending_question_marker_deleted_on_kill` (sase-eo) spawns a
  helper thread that polls up to 10 s for the marker file before triggering the kill; if the flow moves on first,
  `marker_payload` is `None` and the assertion fails. Replace the poll-and-hope handshake with an explicit
  synchronization point between the flow and the helper thread.

## Shared_state

The remaining reported nodes never touch the Textual pump. Their common context is that `--dist=worksteal` (the default
in `tools/run_pytest`) regroups test files onto workers differently on every run, so which process-global state a test
inherits is nondeterministic even though pytest collection order is not. Diagnose each one and fix the isolation defect,
not the assertion.

- `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name`
  and `::test_vcs_tag_directory_key_spelling_also_resolves` (sase-cw, plus +1s from `sase-el.land`, `toobig-1h`, and the
  `commit_publication` split). Both tests are fully synchronous, so timing is not the mechanism.
  `sase/xprompt/project_identity.py` holds two `lru_cache`d projections (`_identity_registry`,
  `_canonical_xprompt_project`) cleared only by the file's own `_identity_registry_reset` fixture; any other test in the
  same worker that populates them with a different project inventory poisons this pair. Confirm by running a poisoning
  file and this file in one process, then make the invalidation robust rather than fixture-local.
- `tests/ace/tui/test_task_mirror.py::test_mirror_counts_global_detached_and_this_sessions_command_tasks` (sase-e1, plus
  a `t0` recurrence). The reported symptom is `counts == []`, meaning `_refresh_detached_count` computed a count equal
  to the mirror's existing `_detached_count` and returned before invoking the callback. `read_tasks` goes through the
  Rust binding `read_tasks_snapshot`; establish whether a stale snapshot or a background refresh racing the explicit
  call owns the failure. If the defect is in the snapshot read, it is core backend work in `../sase-core` (open it
  through `/sase_repo`).
- `tests/artifact_file_facade/test_reclaim.py::test_clean_pushed_file_reclaims_resolves_and_is_idempotent` and
  `tests/artifact_file_facade/test_vcs.py::test_materialize_artifact_file_uses_verified_content_cache` (sase-eg,
  reproduced independently by `toobig-1i` and the `sase-ez` land audit, each time as exactly this pair). Both drive real
  git through the Rust core. The pair always failing together is a strong hint of a shared cache or index, not two
  coincidences; start there.
- `tests/ace/tui/test_notification_custom_gate.py::test_tracked_executor_reports_terminal_and_extra_commands_live`
  (sase-f5) asserts an exact ordered `reporter.phases` list from a tracked executor that runs gate options as
  subprocesses.

## Guard

Turn the harness into a standing gate so this class cannot silently return.

- Add a contention job to `.github/workflows/ci.yml` next to the existing `test` and `visual-test` jobs, running the new
  lane over the ACE TUI tests. Keep it out of the local `just check` path so the required local gate does not get
  slower; agents already run `just check` on every change and it is the gate this flake class has been intercepting.
- Confirm the visual lane separately: run `just test-visual-contention` and verify
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots`
  is clean. That node is sase-cb's, closed as fixed but carrying three later recurrences, and the default `just test`
  lane includes the visual suite. Fix it here if it still drifts.
- Document the lane and the "do not assert after a bare pause" rule in `CONTRIBUTING.md` — not in any memory file.
- Soak: three consecutive full `just check` runs at the default worker count with no single-node failures, plus a clean
  contention lane. Record worker counts, pass totals, and timings; the consolidated beads consistently reported ~25,800
  passes, so a comparable total is the signal that the run was real.
- Close `sase-ct` with that evidence. Note in the close reason which consolidated siblings the fixes cover by node, so a
  future recurrence can be attributed precisely.
