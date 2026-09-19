---
tier: tale
title: Show default effort on the TUI launch-default pill
goal:
  The ACE top-bar launch-default pill names the no-%model target together with the
  launch-effective default effort, for example GROK(grok-4.6)@high, and omits the suffix
  when that effort is unset.
size: small
proposed_by: bbugyi200.apollo.0s
create_time: 2026-09-19 09:47:51
status: wip
---

# Plan: Show default effort on the TUI launch-default pill

## Goal

The ACE top-bar launch-default pill currently names only the no-`%model` target, for
example `GROK(grok-4.6)`. After this work it must also name the effort a no-`%model`,
no-`%effort` launch will actually receive, for example `GROK(grok-4.6)@high`. When that
effort is unset, keep today's suffix-free label so the calm default stays quiet.

## Context

`LLMOverrideIndicator` (`src/sase/ace/tui/widgets/llm_override_indicator.py`) is the
top-right pill in `#top-bar`. Two visual states already exist:

| State                                   | Current string                      | Notes                                                      |
| --------------------------------------- | ----------------------------------- | ---------------------------------------------------------- |
| No model override (calm cyan)           | `PROVIDER(model)`                   | `_build_cached_default_content` / `_build_default_content` |
| Temporary default-model override (gold) | `PROVIDER(model)[@<effort>] <time>` | `_build_override_content` via `build_override_pill`        |

The gold override pill already appends `@<effort>` when the _model override_ carries one
(`tests/test_llm_override_indicator.py` asserts `CODEX(o3)@medium 1h2m`). The calm
default path never does.

Launch effort for a prompt with no `%effort` / `@effort` is already centralized in
`resolve_effective_effort` (`src/sase/llm_provider/config.py`):

1. Explicit per-prompt `%effort` / `@effort` (not in play for this pill).
2. Alias-borne / selected-member effort on the resolved default model
   (`LaunchModelSettingSnapshot.effort`).
3. Active machine-wide temporary default-effort override
   (`~/.sase/llm_effort_override.json`, `Ctrl+E` in Launch Control).
4. `llm_provider.default_effort`.
5. Nothing — each provider keeps its own default.

The pill should show the value from steps 2–4, and omit the suffix at step 5. That is
the same “what will actually be used” resolution the model half of the pill already
applies: it shows `GROK(grok-4.6)`, not the raw `@large` alias.

`LaunchModelSettingSnapshot` already carries alias-borne `effort`. The off-thread worker
in `_schedule_default_resolution_if_needed` already calls
`build_launch_model_setting_snapshot(DEFAULT_MODEL_FIELD, consume=False)` but drops
`snapshot.effort` on the floor when building `_LaunchDefaultSnapshot`.

The 5-second tick is a _peek_, not a resolve (`tui_perf.md` rules 1, 8, 10, 14).
`peek_launch_default_change_token` (`src/sase/llm_provider/launch_default_peek.py`)
currently watches the config token, pool-rotation file, model-override file,
provider-disable file, and provider-priority route. It does **not** watch
`llm_effort_override.json`, so a `Ctrl+E` override or its expiry would not re-arm the
worker. Config edits to `llm_provider.default_effort` _are_ covered by
`current_config_token()`.

`get_active_effort_override()` is a locking, self-cleaning Rust read. It must not run on
the UI thread or in the timer callback. The established pattern is a lock-free peek
module (`temporary_override_peek.py`, `provider_disable_peek.py`) that stats + parses
and filters `expires_at` against `now`.

Autouse test fixture `_isolate_default_llm_effort` (`tests/_conftest_runtime.py`) stubs
configured and temporary default effort to `None`. Existing calm-default assertions stay
suffix-free unless a test explicitly unstubs them. Visual startup
(`tests/ace/tui/visual/_ace_png_snapshot_startup.py`) stubs the launch snapshot with
`effort=None`; keep that so header goldens do not pick up host effort.

## Non-goals

- Do not restyle the gold temporary-_model_-override pill. It already shows
  override-borne effort. Do not invent a fallback that paints global default effort onto
  a gold pill whose override carries none.
- Do not change Launch Control's default-effort row, `Ctrl+E` cards, or
  effort-edit/override workflows.
- Do not change `format_provider_model_label` itself; many non-TUI callers want
  `PROVIDER(model)` with no suffix.
- Do not resolve, glob, or take override locks on the UI thread or in the 5-second tick.
- Do not show a fake `@provider` / `provider default` suffix when no launch-effective
  effort is configured.

## Design

### Visible grammar

Calm default (dim cyan, unchanged style including the suffix):

```text
 PROVIDER(model)
 PROVIDER(model)@high
```

Match the compact `@<level>` connective the gold pill already uses. Do not introduce a
two-tone calm style; two-tone exists to separate subject from modifiers on the inverted
gold/violet pills.

Tooltip for the calm state, using the existing spaced connective from
`format_tooltip_target`:

```text
Launch default: PROVIDER(model)
Launch default: PROVIDER(model) @ high
```

Keep the round-robin extra line, “No temporary override active.”, and “Press ,m for
Config > Launch.” Canonical vocabulary is `none`, `minimal`, `low`, `medium`, `high`,
`xhigh`, `max`. Configured `none` is a real level and **must** render as `@none`; it is
not the same as unset.

Placeholder `...` and failure `unavailable` stay suffix-free.

### Effort value

In the off-thread worker, after `build_launch_model_setting_snapshot`:

```python
from sase.llm_provider.config import resolve_effective_effort
from sase.xprompt.directives import PromptDirectives

level, _explicit = resolve_effective_effort(
    PromptDirectives(),
    snapshot.effort,
)
```

Store `level` on `_LaunchDefaultSnapshot`. That folds alias-borne effort, the temporary
default-effort override, and `llm_provider.default_effort` through the same function
launches use. `_explicit` is unused (a no-effort prompt is never explicit).

The sync `_build_default_content` test helper must use the same folding so unit tests
and the live widget cannot disagree.

### Change detection

Add a lock-free effort-override peek, then fold it into the existing launch default
token:

1. Python path helper for `~/.sase/llm_effort_override.json` (Rust already names this
   file `EFFORT_OVERRIDE_STATE_FILENAME` in
   `sase/repos/linked/sase-core/crates/sase_core/src/effort_override.rs`).
2. New `src/sase/llm_provider/effort_override_peek.py` modeled on
   `temporary_override_peek.py`: time-gated `os.stat`, parse JSON, never take the Rust
   lock, never prune or rewrite, degrade to “no override” on missing/corrupt state,
   filter `expires_at` against the requested clock on every call.
3. Extend `peek_launch_default_change_token` with
   `_stat_token(effort_override_state_path())` **and** the currently active peeked
   effort (or `None`). The active-effort slot is what makes an expiry without a rewrite
   flip the token, matching `peek_provider_priority_change_token`.

Keep the 0.5s peek floor and the “stat error other than missing → sentinel” behavior.

### Instant Launch Control updates

`_refresh_launch_indicators` today only calls `invalidate_cached_default()` when
`provider_routing_changed` is true. Effort edits call `_mark_changed()` without that
flag, so the pill would wait for the peek token (and can miss the write entirely during
the 0.5s peek cache floor).

Change `_refresh_launch_indicators` so **every** Launch Control mutation invalidates the
cached launch default. That path is user-initiated, not a timer tick, so scheduling one
off-thread worker is allowed. Model-override set/clear still peeks onto the gold pill;
invalidating the calm cache is harmless and makes persistent default-model edits equally
snappy.

## Implementation

1. **Peek + token**
   - Add `effort_override_state_path()` next to the existing effort-override façade
     (`src/sase/llm_provider/effort_override.py`), returning
     `sase_home() / "llm_effort_override.json"`.
   - Add `effort_override_peek.py` with `peek_active_effort_override(now=None)`
     returning `TemporaryEffortOverride | None`.
   - Include the effort-override stat token and the active peeked effort in
     `peek_launch_default_change_token`. Update the module docstring.

2. **Snapshot + render**
   - Add `effort: str | None` to `_LaunchDefaultSnapshot` (default `None` so existing
     test constructors keep working).
   - Populate it in the worker from `resolve_effective_effort` as above. Never call
     `resolve_effective_effort` / `get_active_effort_override` from `__init__`,
     `refresh`, `_apply_content`, or `_build_cached_default_content`.
   - `_build_cached_default_content` and `_build_default_content`: append `@<effort>` to
     `format_provider_model_label(...)` when effort is set. Keep `_DEFAULT_STYLE` on the
     whole string.
   - Tooltip: append ` @ <effort>` to the `Launch default:` line when set. Round-robin
     copy should use the same labeled default (with suffix) that the pill shows.

3. **Invalidate on Launch writes**
   - In `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`,
     `_refresh_launch_indicators` always `invalidate_cached_default()` on
     `#llm-override-indicator`, not only when `provider_routing_changed`.

4. **Docs**
   - `docs/ace.md` (the paragraph that currently says the calm pill names
     `PROVIDER(model)`): document `PROVIDER(model)[@<effort>]`, that the suffix is the
     launch-effective default for a no-`%model`/no-`%effort` prompt, and that it is
     omitted when that value is unset.
   - `docs/llms.md` Reasoning Effort section: one sentence that the top-bar
     launch-default pill shows the same launch-effective default.
   - `CHANGELOG.md`: `feat(ace)` entry for the visible suffix.

5. **Visual hygiene**
   - Keep the visual-startup launch snapshot at `effort=None`.
   - Do not read `effective_default_effort_snapshot()` on the UI thread in visual
     startup; the worker path plus the autouse effort isolate is enough. If a new stub
     is needed to keep host config from leaking, stub `resolve_effective_effort` (or the
     peek) in `_ace_png_snapshot_startup.py` to return `None`.
   - No dedicated PNG of the suffix is required unless a targeted visual run shows a
     header golden actually changed. If one does, inspect the `just fix-tui-screenshots`
     report before treating the new bytes as approved.

## Tests

Extend existing files; do not add a parallel test module unless the peek file truly
cannot live beside `tests/llm_provider/test_launch_default_peek.py`.

`tests/test_llm_override_indicator.py`:

- Calm default with configured `high` renders `CODEX(...)@high` in dim cyan (unstub
  `_get_default_effort` for that test).
- Alias-borne snapshot effort wins over configured default effort (same precedence as
  `resolve_effective_effort`).
- Temporary default-effort override wins over configured default effort and loses to
  alias-borne effort.
- Unset effort still renders the suffix-free label.
- Configured `none` renders `@none`.
- Tooltip includes ` @ high` on the `Launch default:` line; round-robin extra line keeps
  the suffixed label.
- Worker success commits `effort` onto `_LaunchDefaultSnapshot`.
- `__init__` / `refresh` still must not call `resolve_effective_effort` or
  `build_launch_model_setting_snapshot` on the UI thread.
- Existing gold-pill cases (`CODEX(o3)@medium 1h2m`, no-effort override) stay unchanged.

`tests/llm_provider/test_launch_default_peek.py` (and a focused peek unit file if the
new module needs one):

- Token changes when `llm_effort_override.json` is created/replaced.
- Token changes when a peeked override expires without a rewrite (clock bump, same file
  bytes), matching the provider-priority expiry test.
- Missing effort-override file still yields a stable token.
- Peek never takes the effort-override lock and never deletes the file.

`tests/test_models_panel_leader_mode.py`:

- `_refresh_launch_indicators()` without `provider_routing_changed` still invalidates
  the cached launch default.

## Verification

While iterating:

```bash
pytest tests/test_llm_override_indicator.py \
       tests/llm_provider/test_launch_default_peek.py \
       tests/test_models_panel_leader_mode.py
```

Then `just fix` and `just check`. If the scoped lane or a header-bearing visual test
reports a PNG diff, run a targeted `just fix-tui-screenshots -- <selectors>` and inspect
the report; do not land unreviewed golden updates.
