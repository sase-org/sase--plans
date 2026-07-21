---
tier: tale
title: Make the Admin Center update key responsive
goal: 'Pressing lowercase u on a freshly loaded Admin Center Updates tab opens the
  SASE update preview without repeating remote work the tab just completed, while
  stale, failed, or incomplete freshness evidence still falls back to a safe per-checkout
  refresh.

  '
create_time: 2026-07-21 09:19:49
status: done
prompt: 202607/prompts/fast_updates_u_keymap.md
---

# Plan: Make the Admin Center update key responsive

## Context

Lowercase `u` in `PluginsBrowserPane` runs the pane-wide SASE core and plugin update flow. The Updates-tab loader has
already collected the runtime core and plugin inventory, fetched editable checkouts, classified their upstream state,
and returned canonical `fresh_editable_roots`. The dev-update planner already accepts that evidence and skips a second
`git fetch` only for explicitly named roots, while continuing to re-read local checkout state and refreshing every other
root normally.

That fast path is currently wired only to automatic update-on-open. The manual key action deliberately passes no roots,
so it repeats serial network fetches immediately after the tab finishes loading. It also reads and parses the uv-tool
receipt in the action path before the thread worker starts. The confirmation modal's larger incoming-commit preview is
already loaded on its own worker after the modal paints, so it is not the key-to-preview blocker.

## Desired behavior

- A manual `u` pressed soon after a successful online Updates load reuses the loader's successful editable-root refresh
  evidence and performs no duplicate remote fetch for those roots.
- Reuse is bounded to a short, explicit freshness window. Once it expires, or after a failed/offline/incomplete reload,
  the existing planner refreshes each uncovered checkout normally.
- Partial evidence remains partial: a root whose load-time fetch failed, or which was detached, offline, unavailable, or
  otherwise not proven fresh, is never promoted into the reusable set.
- The preview still rebuilds runtime inventory and reclassifies local checkout state, so dirtiness, branch/upstream
  changes, and other local safety blockers that occur after the tab load are detected before confirmation.
- The key handler itself performs only guards, immutable state capture, and worker submission. Receipt I/O, inventory
  collection, git probes, and preview construction all execute on the existing thread worker.
- Automatic update-on-open, managed-only installs, mixed editable/managed installs, confirmation content, tracked
  execution, and post-update restart behavior remain unchanged.

## Implementation

### Retain bounded load freshness in the Updates pane

Extend the pane's in-memory load state in `src/sase/ace/tui/modals/plugins_browser_pane.py` to retain the latest
`PluginsLoadResult.fresh_editable_roots` together with a monotonic completion timestamp. Use a small named TTL for this
UI handoff (consistent with the project's short remote-fetch freshness policy) rather than treating a tab that has
remained open indefinitely as authoritative.

Replace the retained set atomically when a load succeeds. Clear it as soon as a new load begins and leave it empty on
error, cancellation, offline results, or results without successful root evidence. Keep the evidence pane-local; do not
add a persistent cache or broaden the semantics of the shared dev-update planner.

Add a focused helper that returns the retained roots only while the monotonic age is within the TTL. Make the clock/TTL
boundary straightforward to inject or patch in tests. Canonicalization and eligibility remain owned by
`plugins_browser_loading._fresh_editable_roots`, which already admits only successful online states.

### Reuse freshness from manual `u` without weakening planning

Update `src/sase/ace/tui/modals/plugins_browser_sase_update.py` so the manual action captures the pane's currently
reusable roots and forwards them through the existing `already_refreshed_roots` path. Preserve the current per-root
fallback in `sase.dev_update.plan.plan_dev_update`: the planner must still fetch any root absent from the captured set
and must still run its local `classify_git_upstream` checks for every editable checkout.

Unify the manual and automatic paths around the same captured freshness contract where practical, while retaining the
one-shot comprehensive-update request behavior. Avoid caching a completed `DevUpdatePlan`; its local safety decisions
would become stale even when the remote refs remain fresh.

Move `load_receipt_for_summary` into the existing `sase-update-plan` worker and capture only the immutable uv-tool probe
plus freshness set in the action handler. This keeps filesystem parsing off Textual's event loop and ensures the visible
UI remains responsive even when receipt access is slow.

### Regression and performance coverage

Update the focused Updates-pane tests under `tests/ace/tui/` to replace the current assertion that manual updates never
reuse load freshness with the new contract:

- immediate manual `u` forwards all successfully refreshed roots;
- automatic update-on-open continues to forward the same evidence;
- expired evidence, a reload in progress, load failure, and offline/empty evidence forward no roots and therefore
  preserve normal refresh behavior;
- partial evidence is passed unchanged, leaving uncovered roots to the planner's existing fallback;
- receipt loading occurs inside the worker rather than synchronously in the action handler;
- managed-only and mixed editable/managed previews still open and retain their existing commands, blockers, and
  completion behavior.

Keep and, if useful, extend `tests/dev_update/test_plan.py` assertions that fresh roots skip fetch while uncovered roots
fetch exactly once and local classification still runs. Add a deterministic performance regression test using
delayed/failing fake fetches or fetch-call counts: after a successful load, manual `u` must reach the confirmation path
without invoking the redundant remote fetches. Prefer call-count/event ordering over a flaky wall-clock-only threshold;
a small elapsed-time assertion may supplement it if the TUI harness is stable.

## Validation

Run `just install` first as required for an ephemeral workspace, then run the focused dev-update planner and Admin
Center Updates suites, including loading, manual/automatic SASE update, mixed-install, and comprehensive-update tests.
Run `just check` for the repository-wide required verification. If a targeted latency test is added, also run it with
the duplicate fetch stub made deliberately slow to demonstrate that fresh manual `u` is independent of that delay, while
the expired-evidence case still exercises the safe fallback.

Manually verify with an editable installation that opening Admin Center → Updates and pressing `u` immediately after
loading opens the confirmation preview promptly, that `r` refreshes and renews the evidence, and that waiting past the
bounded TTL restores the ordinary remote-refresh behavior without blocking the event loop.

## Risks and boundaries

- Reusing evidence forever could hide newly arrived remote commits, so the handoff must expire and must never include
  failed or offline probes.
- Reusing a whole preview would miss local working-tree changes; retain only remote-fetch evidence and always
  reconstruct/reclassify the plan.
- A load may cover only some editable roots; preserve the planner's explicit per-root fallback instead of using an
  all-or-nothing freshness flag.
- This is presentation-layer orchestration over an existing planner contract. No Rust-core behavior or cross-frontend
  update semantics need to change.
