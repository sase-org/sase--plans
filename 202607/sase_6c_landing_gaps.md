---
tier: tale
title: Finish and land the ACE TUI responsiveness epic
goal: 'Remove the two remaining pump-bound slow callbacks, restore a green merged
  test gate, and close sase-6c with its post-close symvision and plan-state cleanup
  complete.

  '
create_time: 2026-07-16 12:10:52
status: wip
prompt: 202607/prompts/sase_6c_landing_gaps.md
---

# Plan: Finish and land the ACE TUI responsiveness epic

## Context

Epic `sase-6c` landed four contiguous implementation commits on `master` and all five child beads are closed. The land
audit confirmed that `HEAD` matches `origin/master` and that no non-epic commits landed after the first epic commit, so
there is no separate post-start integration range to merge. The audit did, however, find two incomplete pump
integrations:

1. `src/sase/ace/tui/actions/agents/_index_maintenance.py` resumes maintenance after a stale-schema rebuild with
   `call_later(self._run_artifact_index_maintenance)`. The normal scheduling and navigation-deferral paths were
   converted to the shared pump-free task helper, but this callback was introduced by phase `sase-6c.3` and missed by
   the later `sase-6c.1` conversion. The merged test `test_scheduler_defers_while_schema_index_is_bypassed` already
   expects the pump-free spawner; `just check` currently ends with this sole failure after 17,653 passing tests.
2. `src/sase/ace/tui/widgets/artifacts/bugs.py` still passes its local async `_runner` to `self.app.call_later`. That
   coroutine awaits `collect_bug_snapshot(...)` through `asyncio.to_thread`, so Textual's serial app message pump waits
   for an issue-provider load. This automatic pane load predates the epic, but it violates the epic-wide invariant that
   callbacks awaiting disk, subprocess, network, or worker-thread work never execute on the message pump. Existing
   app-level Bugs actions were converted, while the pane loader was overlooked.

The config freshness throttle, token-keyed model-alias cache, stale-index metadata check and bounded startup bypass,
pre-existing diff-badge metadata cache, update-status revalidate-only mode, independent 60-minute recompute cadence,
task teardown cancellation, and the other surveyed refresh coalescers were all verified in source and focused tests.
Preserve those behaviors.

## Phase 1: Remove the remaining pump-bound callbacks

- In the stale-schema maintenance resume path, route the already-coalesced request through
  `_spawn_artifact_index_maintenance_task()` rather than `call_later`. Preserve `_artifact_index_maintenance_running`,
  pending-request, force-merge, navigation-gate, and follow-up semantics; do not add another scheduling path.
- Run the Bugs pane's automatic load as a retained pump-free task using the shared helper and the app-level lifecycle
  registry. Preserve its generation checks, cache, loading/pending flags, last-request-wins force handling, and current
  project/filter/activity revalidation after the await. If task creation cannot start, leave the pane retryable rather
  than stranding `_loading`.
- Re-sweep ACE TUI `call_later`, `call_after_refresh`, `set_timer`, and `set_interval` callbacks for async functions or
  closures that await slow work. Convert any same-class callback found. Keep the intentional controlled-exit flush on
  pump ordering and purely synchronous UI/task-staging callbacks as they are, with comments explaining non-obvious
  exceptions.

Before changing TUI responsiveness code, review the required `tui_perf.md` long-memory note through `sase memory read`
and follow its coalescing, post-await state recapture, and teardown rules.

## Phase 2: Prove the integrated behavior

- Keep the stale-schema scheduler regression green and add focused Bugs-pane coverage demonstrating that a slow
  automatic snapshot load does not occupy the Textual message pump. Cover burst coalescing so a request arriving during
  the load produces at most one trailing load with the latest force state.
- Run the focused pump, auto-refresh, live-hint, stale-index, Bugs-pane, config cache/model-alias, update-cache/toast,
  and configuration-schema tests.
- Run `just check` from the installed workspace. Treat any failure as remaining epic work; fix and re-run until the full
  gate is green. Do not accept the prior phase summaries as a substitute for the merged-tree result.

## Phase 3: Land the original epic

This is the final phase and must run only after Phases 1-2 are complete and the merged-tree gate is green.

1. Close the original epic with `sase bead close sase-6c` and verify its child beads remain closed.
2. After closing, run `just symvision` if that recipe is available. Because the epic-symbol allowance expires at close,
   review `symvision.md` through `sase memory read` before fixing any report, then remove stale `sase-6c` allowance
   entries and genuinely unused code. Re-run `just symvision` until it passes.
3. If post-close cleanup changed source or configuration, run `just check` again and require a green result.
4. Finally, set `status: done` in the frontmatter of the original epic plan at
   `${SASE_SDD_PLANS_DIR}/202607/tui_pump_stalls_and_startup.md`. Use the plans sidecar path resolved through
   `sase repo open plans`; do not edit a copied or sibling-workspace plan.

## Risks and safeguards

- Detached tasks interleave where pump callbacks serialized. Reuse existing running/pending/generation guards and
  re-read current pane/tab/filter state after awaits before painting.
- A task-spawn failure must release the corresponding scheduled/loading guard, or subsequent refreshes will be
  suppressed for the session.
- Closing the bead changes symvision's active epic whitelist. Do not run or fix the post-close symvision result before
  the close, and do not mark the original plan done until code cleanup and validation are complete.
