---
tier: tale
size: medium
title: Drain header enrichment before config-refresh teardown
goal:
  Eliminate the late header-worker config-refresh restart behind sase-1vy.1 while
  preserving a strict leak detector and responsive TUI shutdown.
proposed_by: bbugyi200.athena.sase-1vy.1
proposed_with: codex/gpt-6-astra
bead: sase-1vy.1
create_time: 2026-11-28 03:23:50
status: wip
---

- **PARENT:**
  [202611/backlog_epic_2.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_2.md)
- **BEAD:**
  [sase-1vy.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vy/sase-1vy.1.md)

# Drain header enrichment before config-refresh teardown

Complete phase `sase-1vy.1` of `plan:202611/backlog_epic_2.md`. This is a single-agent
implementation tale: one identified producer lifecycle defect, its test isolation
integration, and focused regression coverage. The phase had no earlier-work notes when
inspected. Do not start another backlog epic.

## Evidence and scope

Read `sase-1vy.1` and `sase-1vg`, including their notes, before implementation. The
latest report is CI Telemetry run 46071761552: its tests and cost budget passed, but
four pilot nodes left `sase.config.core._live_config_token_refresh_threads` behind. The
previous fix, `b0f947b91d`, is already present: the leak detector drains the config
revalidator after all fixture teardown and before its after-snapshot. Repeating that
change is insufficient.

Investigation on base `4010ecd89a` found a concrete producer:

- An instrumented run of
  `tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_new_subject_spread_starts_at_top`
  passed with zero poisoning. Its config-refresh start came from
  `sase-agent-detail-header_0`, through `_bead_touches_loader`,
  `AgentIdentitySnapshot.current`, `get_agent_owner_config_snapshot`, and
  `current_config_token`. This passing run identifies a real caller; it does not
  establish that every reported CI failure has that cause.
- `_AgentDetailHeaderEnrichmentService.shutdown()` marks the service closed, drops
  `_active` and `_executor`, and invokes
  `executor.shutdown(wait=False, cancel_futures=True)`. `_start()` discards the
  submitted future. Already-running synchronous work can therefore outlive both shutdown
  and the config drain without a retained completion handle.
- A deterministic, event-controlled probe confirmed the gap: block a service's
  `_build_detail_header_summary` before a call to `current_config_token`, shut down the
  service, drain config, then release the resolver. Immediately after shutdown and
  drain, `service.is_running` is false and the config worker slot is empty; after
  release, a live `sase-config-token-refresh` exists. The probe stubbed token
  computation and joined both executors/workers in `finally`.
- Bare Textual `App` pilot tests also create this app-owned service, while the explicit
  app shutdown hooks live in ACE's `actions/lifecycle.py`. Prompt-panel unmount only
  cancels its selection work. Root test isolation currently knows how to stop proc
  observers but has no equivalent header-service drain.

Fix actual producer lifetime and cleanup order. Preserve detection of genuinely live
refresh threads, including unregistered threads found by enumeration. Do not exempt the
thread name, downgrade poison to warming, disable background refresh, increase sleeps as
the fix, or add a detector-only suppression interval.

## Governing decisions

- `decisions:rust-core-required`: this change concerns Python executor ownership,
  Textual callback delivery, and pytest isolation. Keep it in that adapter layer; do not
  change configuration/domain semantics or duplicate Rust behavior.
- `decisions:check-full-is-explicit` and `decisions:guarded-recipes`: use the targeted
  cost lane and `sase tool run check`. The parent explicitly forbids `just check-full`
  for this work.
- `decisions:triage-annotates-does-not-change-exit-codes`: preserve failed check exit
  codes; an unrelated failure needs independent clean-base evidence.
- `decisions:host-owned-completion` and
  `decisions:handoffs-end-through-host-completion`: finalizers own commits; long
  verification commands use `/sase_monitor` and staged handoffs end through
  `/sase_final`.

No memory changes are authorized or required. Read `ace.md`, `tui.md`, `tui_perf.md`,
and `lint_and_test.md` through `/sase_memory_read` before editing their domains. Read
`tools.md` only if a tools change becomes necessary; the existing runner should suffice.

## Implementation

1. **Turn the reproduced schedule into a regression.** Extend
   `tests/ace/tui/widgets/test_agent_display_header_worker.py` or a focused sibling with
   a blocked resolver that reads config after cancellation/shutdown. Use
   `threading.Event` barriers, real worker completion, and bounded deadlines.
   Demonstrate the old code permits the late config worker; the repaired cleanup must
   wait for the resolver to finish before the final config drain. Keep unrelated real
   filesystem/config work stubbed and release every barrier in `finally`. Do not depend
   on load, sleep timing, or the full CI run to reproduce.

2. **Retain and expose actual producer completion.** In
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_worker.py`, retain the
   submitted future or an equivalent explicit worker-completion handle until its
   synchronous work has exited. Cancellation, UI completion, and physical worker
   completion are different states. Make shutdown idempotent, reject new submissions
   after close, cancel pending work, suppress late publication, and retain enough state
   for a later bounded drain even if shutdown already ran. Preserve one active compute
   and latest-pending selection semantics.

   Provide a narrow quiesce/drain operation for test lifecycle cleanup. Track created
   services without retaining dead apps indefinitely, for example with a weak registry
   that also covers bare `App` instances and direct service tests. Registration and
   completion must tolerate a job finishing immediately after submission. A timed-out
   drain must retain ownership and report failure rather than clear a still-live future
   and pretend success.

   Avoid deadlocking on `app.call_from_thread`: a worker may already be waiting for a UI
   callback when teardown starts. Ensure completion does not require a stopped or
   blocked message loop. If callback delivery needs adjustment, use nonblocking UI
   scheduling for these void callbacks, retain closed/job/selection checks on delivery,
   and verify live incremental publication still works. Keep production unmount
   nonblocking; never join a resolver synchronously on the Textual event loop.

3. **Drain producers before invalidating config and taking snapshots.** Integrate the
   service drain into `tests/_conftest_runtime.py` before config and derived cache
   resets. Resolve the service module only when already loaded, following the existing
   optional TUI cleanup pattern so non-TUI tests do not import the application. Cover
   the final cleanup boundary in `tests/_global_state_leak_detector.py` as needed, using
   the same producer-first ordering rather than another independent cleanup
   implementation.

   Include services created by bare widget-pilot apps, not just `AcePage`. Preserve
   module-scoped/shared-app usability: per-test quiescence must not permanently close a
   service needed by the next test. Either leave a drained live service reusable or
   safely recreate a retired service on its next valid request; a shutting-down app must
   not be resurrected. Ensure cleanup runs before fixture patches and temporary config
   paths are restored. Keep the final config-worker drain after all admitted header work
   actually finishes. Genuine producer timeout must fail visibly even if no refresh
   thread has started yet; detector report-only versus blocking behavior must stay
   intact.

4. **Protect lifecycle behavior with focused tests.** Cover repeated shutdown and drain,
   an already-closed service with work still in flight, timeout ownership, dropped
   pending work, no late UI paint, and reuse by a subsequent test using a shared app.
   Retain the current A/B/A single-flight and resolver-error tests. Add a pytest
   integration regression that exercises a real bare pilot/header service and the normal
   final drain/snapshot order with the blocking leak detector enabled. Hold the resolver
   at the identified late-read boundary to prove ordering rather than hoping for a
   spontaneous flake.

   Continue running the detector's existing deliberately-live-thread and
   deliberate-poison integration tests: the repair must not hide those failures.
   `sase-t6` is a separate, already-fixed detector self-test timeout and should not be
   reopened or rewritten as part of this fix.

## Verification and completion

First run focused header-worker, config teardown/isolation/token, and leak-detector
regressions. Then run the cost lane over every historically reported pilot file plus the
new regression. The existing runner enables both global-leak flags:

```sh
.venv/bin/python tools/run_pytest cost \
  tests/ace/tui/widgets/test_agent_display_header_worker.py \
  tests/test_config_cache_teardown.py \
  tests/test_config_cache_isolation.py \
  tests/test_config_cache_token.py \
  tests/test_global_state_leak_detector.py \
  tests/ace/tui/widgets/decks/test_deck_spread_pilot.py \
  tests/ace/tui/widgets/test_agent_header_panel_preview.py \
  tests/ace/tui/widgets/decks/test_deck_chrome.py \
  tests/ace/tui/widgets/decks/test_deck_context_landing.py \
  tests/ace/tui/widgets/test_vim_normal_key_containment.py \
  tests/ace/tui/test_tribe_panel_flicker.py \
  tests/ace/tui/test_tribe_panel_flicker_debounced.py
```

Add any new regression file to this selection. Inspect the JSON leak report for zero
relevant poisoning and zero detector errors; passing pytest assertions alone are
insufficient. Do not compare a targeted cost recording against the full-suite budget. If
another listed producer still recreates the same thread, capture its creation stack and
repair its corresponding lifecycle within this scope before claiming the phase is
complete. Do not broaden into unrelated cache cleanup.

Run `just fix`, then `sase tool run check`. Use `/sase_monitor` for commands that will
outlast a turn and wait for the staging command itself to exit. Record command results
and the check run ID. Snapshot regeneration is unnecessary unless rendered output
actually changes; if it does, follow the TUI screenshot memory.

The latest user instruction authorizes closing **only `sase-1vy.1`**, overriding the
phase-description instruction to also close `sase-1vg`. Leave `sase-1vg` and the parent
epic for the land agent; put the fix and verification evidence on this phase so the land
agent can reconcile the source task. Do not create task beads. Record unrelated
discoveries with `sase bead note sase-1vy.1 'PROPOSED FOLLOW-UP: <summary — evidence>'`.
For a failure that reproduces identically on a clean base, record that evidence and any
existing task ID, then close the completed phase as instructed.

Before closing, run `sase bead epic-symbols sase-1vy.1`. Resolve every remaining entry
or re-key its Justfile exception to a still-open owner under the symvision rules. Close
only this phase with
`sase bead close sase-1vy.1 --note '<fix, deterministic regression, cost-lane result, check run and any independently confirmed baseline failures>'`.
Finish through `/sase_final`; do not close any ancestor or plan a third backlog round.
