---
tier: tale
title: Configurable ACE update-check interval
goal: Make the ten-minute ACE periodic update-check cadence a safe user-overridable
  SASE configuration default while preserving non-blocking timer behavior and independent
  cache TTL semantics.
create_time: 2026-07-16 09:47:17
status: wip
prompt: 202607/prompts/configurable_ace_update_check_interval.md
---

# Plan: Configurable ACE Update-Check Interval

## Context

The periodic ACE update-check implementation currently registers its Textual timer from
`src/sase/ace/tui/actions/update_toast.py` with the private constant `_AUTOMATIC_UPDATE_CHECK_INTERVAL_SECONDS = 600.0`.
The default configuration and public schema describe the cadence as fixed, and the lifecycle tests assert the literal
600-second value. Consequently, users cannot override the cadence in their own `sase.yml` today.

ACE already exposes `ace.updates.check_ttl_minutes`, but that field controls cached update-status freshness rather than
the session timer. Keep these controls separate: the interval determines how often a long-running ACE session becomes
eligible to enqueue a check, while the TTL determines whether the background status computation can reuse a snapshot. In
particular, a zero TTL must continue to mean “do not reuse a cached status” without creating a zero-length timer.

The timer is registered after first paint, and its callback runs on Textual's event loop. Per the TUI performance
requirements, configuration file I/O and parsing must not be added to timer registration or timer ticks. ACE already
loads merged configuration during app-state initialization before mount, so the new interval should be resolved from
that existing in-memory configuration and stored as app state for later timer registration.

This is a `tale`: it is a focused configuration and ACE lifecycle refinement in the current Python repository, with no
shared backend behavior or cross-repository phase.

## Configuration Contract

Add `ace.updates.check_interval_minutes` as the user-facing cadence field.

- Set its bundled default to `10`, preserving current behavior for users who do not override it.
- Define it in `src/sase/config/sase.schema.json` as a numeric value strictly greater than zero. Describe it as the
  interval between eligibility attempts in a running ACE session, not as a guarantee that expensive work runs on every
  tick: the positive-indicator and in-flight guards may still suppress work.
- Update the `check_ttl_minutes` description so it refers to the separately configured session cadence instead of a
  fixed ten-minute cadence.
- Treat the value as startup configuration: resolve it once for each `AceApp` instance, and make user edits effective on
  the next ACE launch. Do not add live timer replacement or multiple concurrent timers.
- Convert minutes to seconds at the configuration boundary. Missing, malformed, boolean, non-finite, zero, or negative
  runtime values must defensively fall back to the ten-minute default even if schema validation was bypassed. Use a
  strict positive-finite parser for this field rather than changing the existing TTL coercion, because zero remains a
  valid TTL.

An override should therefore have the following shape:

```yaml
ace:
  updates:
    check_interval_minutes: 30
```

## Implementation

### 1. Resolve the cadence during existing app configuration initialization

Centralize the default and conversion logic near the update-check configuration helpers in
`src/sase/ace/tui/actions/update_toast.py`. Provide a pure resolver that accepts the already-loaded `ace.updates`
mapping and returns a validated interval in seconds.

In `src/sase/ace/tui/actions/_state_init_late.py`, reuse the `ace_cfg` mapping obtained from the existing
`load_merged_config()` call to resolve the interval. Store the result on the app as
`_automatic_update_check_interval_seconds`. Add the corresponding initialization/type declarations alongside
`_automatic_update_check_timer` and `_automatic_update_check_in_flight` in the state/startup mixins. This must not add
another merged-config load or any mount-time/event-loop file access.

Keep direct mixin test harnesses robust by allowing timer registration to fall back to the same 600-second default when
the app-state attribute is absent. The real `AceApp` path should always initialize it explicitly.

### 2. Register the existing timer with the resolved interval

Change `UpdateToastMixin._start_periodic_update_checks()` to pass the stored interval to `set_interval()` instead of the
hard-coded 600-second constant. Preserve all existing lifecycle behavior:

- register exactly one named `automatic-update-check` timer per app;
- keep the immediate post-first-paint startup check;
- keep timer ticks limited to mounted indicator state, the in-flight guard, and worker submission;
- retain the positive-indicator short circuit, overlap suppression, off-thread status/commit work, UI-thread result
  application, and best-effort guard release;
- continue passing `check_ttl_seconds` only to `get_cached_update_status()` and never derive the timer interval from it.

Do not move the cadence into the Rust core: timer ownership and scheduling are presentation-only Textual lifecycle
behavior.

### 3. Align public descriptions and focused coverage

Update nearby comments/docstrings that call the interval fixed or hard-coded. Ensure the schema-backed Config Center can
surface the new field automatically without adding a dedicated UI control or visual change.

Extend `tests/ace/tui/test_update_toast.py`, the focused startup lifecycle tests, and schema tests where useful to
cover:

- no override still registers one timer at 600 seconds;
- a fractional or integer `check_interval_minutes` override is converted to seconds and used by timer registration;
- missing and defensively invalid values fall back to 600 seconds, while the public schema rejects zero/negative values;
- direct mixin harnesses without initialized app state retain the default;
- a zero/custom `check_ttl_minutes` changes only the cache TTL passed to `get_cached_update_status()` and does not
  change the configured timer cadence;
- timer naming, one-time registration, immediate startup worker scheduling, positive-indicator gating, and overlap
  behavior remain unchanged;
- the timer path performs no merged-config load.

No PNG snapshot change should be needed because the feature changes configuration and scheduling only.

## Verification

Run focused tests first:

```bash
pytest tests/ace/tui/test_update_toast.py \
  tests/ace/tui/test_startup_stopwatch_live_update.py \
  tests/test_config_schema.py
```

Then follow the repository requirement for source/config changes:

```bash
just install
just check
```

Do not modify memory files, `AGENTS.md`, or generated provider instruction shims to resolve failures without explicit
user permission.

## Acceptance Criteria

- `ace.updates.check_interval_minutes` is a documented, schema-valid user configuration field with a default of ten
  minutes.
- An override controls the single periodic ACE timer after conversion to seconds, while absent/invalid runtime values
  safely preserve the 600-second behavior.
- `check_ttl_minutes` remains independent and retains zero-TTL semantics.
- No config I/O or expensive update work runs on the Textual event loop; timer ticks keep their existing constant-time
  gates and background-worker boundary.
- Immediate startup checks, update badge/toast behavior, overlap suppression, failure recovery, and one-timer lifecycle
  semantics remain intact.
- Focused tests and the full required repository checks pass.
