---
tier: tale
title: Usage-limit disables fall back to collected usage-window resets
goal:
  When a usage-limit error carries no reset hint, the provider disable expiry comes from
  the usage collectors' window reset data (clamped and corroborated), falling back to
  the flat disable_seconds only when no usable window data exists.
size: medium
proposed_by: bbugyi200.athena.0ix.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ix.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ix.f0.md)
- **COMMITS:**
  - [12f01fb](https://github.com/sase-org/sase/commit/12f01fbc1c7ad45b520391f35df963554f8f94f0)
    — feat(llm-provider): fall back to collected usage-window data for disable duration

# Usage-Limit Disable Duration: Fall Back To Collected Usage-Window Resets

## Problem

When a provider error matches a usage-limit pattern, `detect_usage_limit()` in
`src/sase/llm_provider/usage_limit_config.py` resolves the disable duration in exactly
two steps:

1. If `honor_reset_hint` is on and `parse_reset_hint()` finds a reset instant in the
   error text, that instant (clamped to `min/max_disable_seconds`) becomes `expires_at`.
2. Otherwise the flat per-provider or global `disable_seconds` is used.

Grok Build never emits a reset instant in any limit message (see the comment in
`GrokPlugin.llm_default_usage_limit_config()` in `src/sase/llm_provider/grok.py`), so
every Grok usage-limit event lands on its flat built-in `disable_seconds=172800` (48h).
That is what produced the observed "disabled for 2 days" outcome even though the
account's billing window may reset much sooner (or later).

Meanwhile the recently landed LLM provider usage collectors already record, per
provider, public usage windows that include the window reset instant:
`load_provider_usage()` (Rust-backed, `src/sase/llm_provider/usage/_facade.py`) returns
a snapshot whose `providers[]` entries carry `windows[]` with `resets_at`,
`used_percent`, `remaining_percent`, `freshness`, `reset_passed`, and `applicability`.
The Grok collector (`src/sase/llm_provider/usage/grok.py`) fills `resets_at` from the
billing period end.

**Goal:** when the error text has no parseable reset hint, fall back to the collected
usage-window data to compute the disable expiry, and only then to the flat
`disable_seconds`. Precedence: error-text reset hint > usage-window reset > flat config
default.

## Design Decisions

- **Where the logic lives.** The whole usage-limit detection stack
  (`usage_limit_config*.py`, `usage_limit_disable.py`) is Python in this repo and is the
  only writer of `source="usage_limit"` disables, so the fallback is Python glue beside
  it. It must reuse the existing Rust domain logic for window selection
  (`provider_usage_summarize_for_model` binding and the snapshot's per-provider
  `summary`), not reimplement applicability/freshness/limiting-window policy in Python.
  This respects the Rust-core boundary: selection semantics stay in `sase-core`; only
  the "read `resets_at` of the limiting window and validate it is in the future" glue is
  new Python. No `sase-core` change is required.
- **Window selection.** Use the model-scoped scoped summary when the failing model id is
  known (`provider_usage_summarize_for_model(windows, model_id)`), else the provider
  entry's own account-scope `summary` field from the snapshot. Both already restrict to
  fresh, applicable, not-reset-passed windows and report `limiting_window_keys` (the
  minimum-remaining windows). Map those keys back to the provider's `windows[]` and
  collect finite `resets_at` values strictly greater than `now`; use the **maximum**
  such value (if several windows tie as limiting, the provider is only usable once all
  of them reset). If no usable value: fall through to the flat default.
- **Corroboration gate.** Only trust a window reset when the collected data actually
  corroborates exhaustion: require the scoped summary's
  `used_percent >= DEFAULT_USAGE_CRITICAL_PERCENT` (90.0, from
  `src/sase/llm_provider/usage/constants.py`). A fresh snapshot showing plenty of
  headroom means the limit error is about a dimension the collector does not see (e.g. a
  credit balance); in that case the flat default is the safer duration. Use a
  module-level constant, not new config.
- **Clamping.** Treat the window-derived duration exactly like a parsed reset hint:
  clamp `resets_at - now` to `[min_disable_seconds, max_disable_seconds]`. A Grok
  monthly period ending 25 days out therefore yields the 7d cap, after which expiry
  re-probes and, if still limited, a fresh disable is written. Document this in the code
  comment that currently says the min/max bounds only apply to reset hints.
- **Config surface.** Mirror `honor_reset_hint`: add a global
  `llm_provider.usage_limit.honor_usage_windows: bool = true` to `UsageLimitSettings`
  and a per-provider `honor_usage_windows: bool | null = null` override to
  `ProviderUsageLimitConfig` (key-presence merge semantics identical to
  `honor_reset_hint`). This is a durable user choice, so it is a config field — per the
  feature-flag guidance, no feature flag is warranted; the flat-default branch remains
  reachable as the natural fallback, not as a deprecated path.
- **Provenance.** Add `reset_source: str | None = None` (values `"provider_hint"`,
  `"usage_window"`, or `None`) to `UsageLimitDetection`, with a default so existing
  constructions (e.g. the drain-notify reconstruction) keep working. `used_reset_hint`
  keeps meaning strictly "parsed from the error text" and stays `False` on the
  usage-window path.
- **Failure isolation.** The store read and summarize call must be wrapped so any
  exception (missing state file, Rust binding error, malformed snapshot) silently falls
  through to the flat default — mirroring how `handle_possible_usage_limit` never lets
  detection problems mask the provider error. The store read is one local file read
  through the Rust binding, on the error path only, after a pattern match — no
  performance concern.

## Implementation

### 1. New helper module `src/sase/llm_provider/usage_limit_window_reset.py`

Follows the existing `usage_limit_config_*` sibling-module split. One public function:

```python
def usage_window_expires_at(
    provider: str,
    model: str | None,
    *,
    now: float,
) -> float | None:
    """Best-effort disable expiry from collected usage-window data, else None."""
```

Behavior:

- `load_provider_usage(now=now)`; find the entry in `read.snapshot["providers"]` (a
  list) whose `"provider"` equals `provider`. None → return None.
- Resolve the scoped summary: when `model` is truthy, strip any `provider/` prefix from
  the model target (reuse `model_id_from_target` from `sase.llm_provider.usage.hints` or
  equivalent inline split) and call
  `provider_usage_summarize_for_model(windows, model_id)`; when `model` is None, use the
  provider entry's `"summary"` value. None → return None.
- Corroboration: `summary["used_percent"] >= DEFAULT_USAGE_CRITICAL_PERCENT`, else
  return None.
- Map `summary["limiting_window_keys"]` to windows by `"key"`; collect finite
  `resets_at > now`; return the max, else None.
- Entire body under `try/except Exception: return None` (log at debug), so detection
  never breaks on store or binding problems.

### 2. Wire into `detect_usage_limit()` (`usage_limit_config.py`)

- Add keyword-only `model: str | None = None` parameter to `detect_usage_limit()` and
  `find_usage_limit_detection_for_error()`, threaded through by callers below.
- Resolve `honor_usage_windows` the same way `honor_reset_hint` is resolved
  (per-provider override, else global).
- After the reset-hint branch, when no hint produced an expiry and `honor_usage_windows`
  is true: call `usage_window_expires_at(provider, model, now=resolved_now)`; on a
  value, clamp the duration to `[min_disable_seconds, max_disable_seconds]`, set
  `disable_seconds`, `expires_at`, and `reset_source="usage_window"` (leave
  `reset_hint=None`, `used_reset_hint=False`).
- Set `reset_source="provider_hint"` on the existing hint branch.
- Update the comment that says min/max bounds apply only to reset hints: they now bound
  both externally-derived durations (hint and usage window); the flat `disable_seconds`
  remains used as configured.

### 3. Config types and parsing

- `usage_limit_config_types.py`: add `honor_usage_windows: bool = True` to
  `UsageLimitSettings`; add `honor_usage_windows: bool | None = None` to
  `ProviderUsageLimitConfig`; add `reset_source: str | None = None` to
  `UsageLimitDetection`.
- `usage_limit_config.py`: parse the global key in `get_usage_limit_settings()`; carry
  the per-provider key through `_clone_config`, `_config_from_user_dict`, and
  `_merge_with_built_in` with the same key-presence override semantics as
  `honor_reset_hint`.

### 4. Callers and payloads

- `usage_limit_disable.py` `_handle_possible_usage_limit()`: pass `model=model` to
  `detect_usage_limit()`. Add `"reset_source": detection.reset_source` to the drain proc
  payload in `_submit_drain()`, and include `reset_source` in the auto-disable
  `logger.info` line beside `used_reset_hint`.
- `src/sase/ops/commands/_agent_drain_notify.py`: pass
  `reset_source=_optional_str(trigger.get("reset_source"))` when reconstructing
  `UsageLimitDetection`.
- `src/sase/axe/run_agent_exec_retry.py`: pass `model=ctx.agent_model` where
  `handle_possible_usage_limit` is already called (no change needed if already passed);
  leave the classification-only `find_usage_limit_detection_for_error` call sites with
  the default `model=None`.

### 5. Notification wording (`src/sase/notifications/senders.py`)

In `notify_provider_usage_limit_disabled()`, extend the expiry sentence:

- `used_reset_hint` → existing "Re-enables at {t}, as reported by the provider."
- `reset_source == "usage_window"` → "Re-enables at {t}, based on collected usage data."
- otherwise → existing plain "Re-enables at {t}."

### 6. Docs and defaults

- `src/sase/default_config.yml`: extend the commented `usage_limit` block with
  `honor_usage_windows: true` (global) and `honor_usage_windows: null` (per-provider),
  with a one-line comment describing the hint > window > flat precedence.
- `docs/configuration.md` (`llm_provider.usage_limit` section, ~line 1802): add the two
  new keys to the YAML example and the settings table; amend the `disable_seconds` and
  `providers.grok.disable_seconds` row text to mention the usage-window fallback that
  now usually preempts the flat value when collector data corroborates the limit.
- `src/sase/llm_provider/grok.py`: update the `llm_default_usage_limit_config()` comment
  — `disable_seconds` is now the last resort behind the usage-window fallback, and stays
  load-bearing when the collector has no fresh corroborating window (e.g. logged-out
  CLI, API mode, stale snapshot).

## Tests

Extend `tests/test_llm_provider_usage_limit_detect.py` (or add a sibling
`tests/test_llm_provider_usage_limit_window_reset.py`) using the existing
`load_merged_config`/`_built_in_defaults` patch style plus
`tests/_usage_view_helpers.usage_window` to build provider snapshot entries, patching
`load_provider_usage` where the helper reads it:

1. No hint + corroborated fresh window with future `resets_at` → `expires_at` equals the
   clamped window reset, `reset_source == "usage_window"`, `used_reset_hint is False`.
2. Error-text hint present → hint wins; window data ignored;
   `reset_source == "provider_hint"`.
3. `resets_at` missing or `<= now` on all limiting windows → flat fallback,
   `reset_source is None`.
4. Summary absent (stale/reset-passed/inapplicable windows) → flat fallback.
5. Corroboration failure (`used_percent` below the critical threshold) → flat fallback.
6. `load_provider_usage` raising → flat fallback, no exception escapes.
7. Global `honor_usage_windows: false` and per-provider override in both directions.
8. Clamping: far-future reset (e.g. 25d) clamps to `max_disable_seconds`; near reset
   floors at `min_disable_seconds`.
9. Model scoping: model-specific limiting window chosen for the failing model;
   account-scope summary used when `model is None`; ties across limiting windows use the
   max `resets_at`.
10. Drain payload roundtrip: `_submit_drain` includes `reset_source` and
    `send_usage_limit_drain_notification` reconstructs it (extend
    `tests/test_ops_agent_drain_notify.py`).
11. Notification wording for the `usage_window` source (alongside the existing sender
    tests for `notify_provider_usage_limit_disabled`).

Existing suites that must stay green: `tests/test_llm_provider_usage_limit_detect.py`,
`tests/test_llm_provider_usage_limit_config.py`,
`tests/test_llm_provider_usage_limit_reset_hint.py`,
`tests/test_llm_provider_usage_limit_disable.py`,
`tests/test_ops_agent_drain_notify.py`, and the registry-metadata tests (the new
`ProviderUsageLimitConfig` field flows into `default_usage_limit_config` metadata
dumps).

## Verification

Run `just check` (the agent-default two-speed verification gate) and the focused suites
above. Follow the `lint_and_test` reference memory before finishing, as usual for
changes to tracked files.

## Out Of Scope

- Re-arming or shortening an already-written disable when the post-limit usage refresh
  (`trigger_usage_refresh_after_limit_event`) lands fresher window data; the
  at-detection fallback covers the common case because collection is periodic and window
  boundaries are known before exhaustion.
- Any `sase-core` Rust change: existing bindings already expose the needed selection
  logic.
- Changing Grok's built-in 48h flat default or the global 24h default.
