---
tier: tale
title: Periodic non-blocking ACE update checks
goal: Keep long-running ACE sessions aware of newly available SASE updates by checking
  every ten minutes until the updates indicator is set, without blocking or overlapping
  work on the TUI event loop.
create_time: 2026-07-16 09:29:15
status: wip
prompt: 202607/prompts/periodic_ace_update_checks.md
---

# Plan: Periodic Non-Blocking ACE Update Checks

## Context

ACE currently starts one best-effort update check after the first paint:

- `StartupLoadsMixin._start_post_mount_background_loads()` calls
  `UpdateToastMixin._schedule_startup_update_toast_check()` once.
- `UpdateToastMixin` runs `get_cached_update_status()` in a Textual worker with `thread=True`, applies the resulting
  count to `UpdatesAvailableIndicator` on the UI thread, and shows the update-available toast at most once per session.
- `get_cached_update_status()` uses the configured `ace.updates.check_ttl_minutes` value (ten minutes by default),
  revalidates cached results against the local installation, recomputes stale results, and preserves a stale snapshot as
  a best-effort fallback when a fresh computation fails.
- The persistent badge exposes its current update count through `UpdatesAvailableIndicator.count`. Dismissing the Admin
  Center already revalidates and updates that count after update-related actions.

The existing worker boundary is important: update checks may perform config and cache I/O, package metadata inspection,
PyPI/GitHub access, git subprocesses, and incoming-commit collection. Per `memory/tui_perf.md`, none of that work may
move onto Textual's event loop. The periodic feature should add only a cheap timer callback to that loop and continue
using the established worker path.

This is a `tale` because it is one focused ACE lifecycle change in the current Python repository. It does not add shared
backend/domain behavior and therefore does not require a coordinated `sase-core` phase.

## Desired Session Semantics

1. Preserve the immediate post-first-paint startup check.
2. Register one Textual interval timer for exactly 600 seconds. Each tick should inspect only in-memory/UI state before
   deciding whether to enqueue work.
3. If the mounted updates indicator already has a positive count, the tick is a no-op: no cache read, package scan,
   subprocess, or network call should run. Keep the lightweight timer alive so checks can resume if an existing flow
   later clears the indicator.
4. If the indicator is clear and no update check is already in flight, enqueue the same update-status worker off-thread.
   If a prior startup or periodic check is still running, skip the tick rather than overlap checks.
5. A periodic discovery should use the existing UI behavior: set the badge and, when the existing toast setting permits
   and the session has not already consumed an update toast, show the update-available toast. Once the badge is set,
   later timer ticks stop scheduling checks.
6. Respect the existing update settings on each worker run. In particular, configurations that disable the indicator
   should not perform recurring update-status work merely to support a surface the user disabled; the existing
   startup-toast behavior remains intact. If both toast and indicator are disabled, retain the current fast exit.
7. Keep the fixed session timer separate from the cache TTL. The timer determines when a long-running ACE session asks
   again; `check_ttl_minutes` continues to determine whether `get_cached_update_status()` may reuse a snapshot. This
   avoids a zero-valued TTL accidentally creating a hot Textual timer while preserving all existing configuration
   compatibility.
8. Treat failures as best effort. Log them at the existing debug level, clear the in-flight guard on every
   success/failure/early-return path, and allow a later interval to retry. Update checking must never break startup or
   normal TUI interaction.

## Implementation

### 1. Separate one-time startup orchestration from reusable check scheduling

Refactor the focused scheduling portion of `src/sase/ace/tui/actions/update_toast.py` so startup and periodic ticks
share a single worker-launch helper while preserving the current update computation and rendering functions.

- Add a named 600-second interval constant rather than embedding a magic value.
- Start the interval once from the existing post-mount/after-first-paint path; do not add a process-wide daemon or
  affect non-ACE commands.
- Give the timer a descriptive Textual name for inspection and diagnostics.
- Keep the existing startup worker group behavior unless a dedicated update group is needed for clarity; do not use
  exclusivity as a substitute for the explicit in-flight guard, because canceling a thread worker does not reliably stop
  its underlying synchronous work.

### 2. Add cheap eligibility and overlap guards

Maintain explicit per-app state for whether an automatic update check is in flight (and, if needed for one-time
registration or typing, the interval timer). Initialize it with the rest of ACE state in
`src/sase/ace/tui/actions/_state_init.py` and expose the corresponding type declaration through the startup/update mixin
composition as appropriate.

The interval callback must remain constant-time on the event loop:

- resolve the already-mounted `UpdatesAvailableIndicator` and read `count`;
- return immediately when `count > 0`;
- return immediately while the update worker is in flight;
- otherwise mark the check in flight and enqueue the existing function with `thread=True`.

If the widget is unexpectedly unavailable or worker scheduling fails, degrade silently using the module's existing
logging style. Reset the guard when scheduling fails and marshal normal worker completion back to the UI thread before
changing app/widget state.

### 3. Reuse the complete existing status and notification pipeline

Keep `get_cached_update_status()` as the sole automatic status source and pass the parsed `check_ttl_seconds` exactly as
startup does today. Do not duplicate package/version logic in the TUI and do not bypass snapshot revalidation.

Apply successful periodic results through the current helpers so all surfaces stay consistent:

- update the top-bar count when the indicator is enabled;
- retain the one-toast-per-session `_update_toast_shown` guard;
- build incoming-commit sections only when a toast will actually be shown;
- perform that commit collection inside the same background worker, never in a timer or UI callback.

Avoid a second expensive status computation for a single tick. A no-update or failed result leaves the indicator clear
and becomes eligible for the next ten-minute attempt.

### 4. Align user-facing configuration descriptions

Update the relevant descriptions in `src/sase/config/sase.schema.json` and nearby update-toast docstrings/comments so
they describe automatic ACE-session checks rather than claiming the status/toast path is exclusively startup-only. Keep
existing keys, defaults, and backward compatibility unchanged; no config migration or new cadence option is needed.

## Tests

Extend `tests/ace/tui/test_update_toast.py` and, where lifecycle registration is best asserted, the focused startup-load
tests. Cover at least:

- startup still launches the immediate off-thread check;
- exactly one named interval is registered at 600 seconds;
- a periodic tick with a positive indicator count schedules no worker;
- a clear indicator schedules the worker with `thread=True`;
- an in-flight startup/periodic check suppresses duplicate scheduling;
- worker completion, early return, exception, and scheduling failure all release the in-flight guard so a future tick
  can retry;
- a periodic update result sets the badge and shows the existing toast once;
- subsequent ticks stop doing status work after the badge is set;
- indicator-disabled configuration retains existing startup behavior but skips recurring status computation;
- configured TTL seconds are still passed to `get_cached_update_status()` and are not used as an unsafe zero-length
  interval;
- direct mixin harnesses and full `AcePage` startup continue to work without waiting ten real minutes (capture/invoke
  the timer callback deterministically).

No visual snapshot change should be necessary because the badge and toast rendering do not change. If implementation
changes rendered output, update a snapshot only after confirming the visual change is intentional.

Run focused verification first:

```bash
pytest tests/ace/tui/test_update_toast.py \
  tests/ace/tui/test_startup_stopwatch_live_update.py
```

Then follow the repository requirement for implementation changes:

```bash
just install
just check
```

Do not modify memory files or generated provider instruction shims to resolve a check failure without explicit user
permission.

## Acceptance Criteria

- A long-running ACE session performs an immediate startup check and becomes eligible for another automatic check every
  ten minutes.
- A positive updates-indicator count prevents the periodic tick from launching any update worker or doing
  I/O/network/subprocess work.
- At most one automatic update check is active at a time, and failures do not permanently suppress later retries.
- Newly discovered updates use the existing badge and one-shot toast behavior.
- All expensive operations remain off the Textual event loop, and the timer callback is limited to in-memory gating plus
  worker submission.
- Existing update cache TTL/configuration, stale-snapshot fallback, manual Admin Center refresh/revalidation, and
  non-ACE behavior remain compatible.
- Focused tests and the required full repository checks pass.
