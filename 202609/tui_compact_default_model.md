---
tier: tale
title: Compact `%model` spelling on the TUI launch-default pill
goal:
  The ACE top-bar launch-default pill names the no-%model target with the shortest
  string `%model` would accept, then the existing `@<effort>` suffix, for example
  grok-4.6@high instead of GROK(grok-4.6)@high.
size: small
proposed_by: bbugyi200.apollo.0s.f0
create_time: 2026-09-19 11:43:34
status: wip
---

# Plan: Compact `%model` spelling on the TUI launch-default pill

## Goal

The ACE top-bar launch-default pill currently names the no-`%model` target as
`PROVIDER(model)[@<effort>]`, for example `GROK(grok-4.6)@high`. After this work it must
use the shortest string the `%model` directive would accept for that same
provider/model, then keep the existing `@<effort>` suffix. The example becomes
`grok-4.6@high`.

## Context

The previous tale landed the `@<effort>` suffix on this pill (`ea5fc2b504`).
`LLMOverrideIndicator` (`src/sase/ace/tui/widgets/llm_override_indicator.py`) is the
top-right pill in `#top-bar`. Both of its model-identity states still go through
`format_provider_model_label` (`src/sase/llm_provider/_registry_routing.py`):

| State                                   | Current string                      | Formatter                       |
| --------------------------------------- | ----------------------------------- | ------------------------------- |
| No model override (calm cyan)           | `PROVIDER(model)[@<effort>]`        | `_format_default_label`         |
| Calm tooltip                            | `PROVIDER(model)[ @ <effort>]`      | `_format_default_tooltip_label` |
| Temporary default-model override (gold) | `PROVIDER(model)[@<effort>] <time>` | `_build_override_content`       |
| Gold tooltip target                     | `PROVIDER(model)[ @ <effort>]`      | shared `format_tooltip_target`  |

`format_provider_model_label` is the display badge used across ACE, axe, and CLI
(`CLAUDE(opus)`, `GROK(grok-4.6)`). It is **not** the `%model` grammar.

`%model` / `%m` accept (`docs/xprompt.md`, `resolve_model_provider` in
`src/sase/llm_provider/registry.py`):

1. A configured alias (`@large`).
2. Explicit `provider/model` (`grok/grok-4.6`, `opencode/anthropic/claude-sonnet-4-5`).
3. A bare canonical model name from `model_to_provider_map()` (`grok-4.6`, `opus`,
   `o3`). The first slash is the provider split when the left-hand token is a registered
   provider name.

Completion inserts those same values. Plugin short aliases (`haiku45`, `sonnet45`) are
match/display hints only; they are not `%model` values.

The pill already shows the _resolved_ concrete model, not the raw `@large` alias. This
tale only changes how that concrete `(provider, model)` pair is spelled. Effort
resolution, peek tokens, and Launch Control invalidation stay as landed.

This is a `small` tale: one helper, the one top-bar widget, existing tests, and the two
docs sentences that currently advertise `PROVIDER(model)[@<effort>]`.

## Non-goals

- Do not change `format_provider_model_label`. Agent lists, Launch Control row badges,
  plan-review titles, axe/CLI `Model:` lines, and alias-override tooltips that still
  want `PROVIDER(model)` stay on that helper.
- Do not change Launch Control, `Ctrl+E`, effort resolution, peek tokens, or the 5 s
  tick. The `@<effort>` suffix and its omit-when-unset rule stay as landed.
- Do not invert plugin short aliases (`haiku45`, `sonnet45`). They are not uniquely
  routed `%model` values.
- Do not show the unresolved alias (`@large`) on this pill. It remains the resolved
  concrete target.
- Do not call `resolve_model_provider` (or any alias resolver that can consume a
  round-robin cursor) from the render path. Cached metadata lookups only.
- Do not restyle the pill (dim cyan / gold two-tone / tooltip connective). Only the
  model-identity subject string changes.
- Do not change the violet alias-override pill (`@<alias>[@<effort>] <time>`).

## Design

### Compact `%model` spelling

Add `format_model_directive_label(provider: str, model: str) -> str` next to the
registry façade (`src/sase/llm_provider/registry.py`), not in `_registry_routing.py`
(that module cannot import `model_to_provider_map` without a cycle).

Given the already-resolved `(provider, model)`:

1. Let `qualified = f"{provider}/{model}"`.
2. If `model_to_provider_map().get(model) != provider`, return `qualified`. The bare
   name either is unknown or belongs to a different provider, so `%model:<model>` would
   not route here.
3. If `model` contains `/` and `model.split("/", 1)[0]` is in
   `registered_provider_names()`, return `qualified`. Otherwise `%model:<model>` would
   be stolen as an explicit provider prefix. Today `anthropic` is not a provider, so
   OpenCode's `anthropic/claude-sonnet-4-5` stays bare; if that prefix later becomes a
   registered provider, the helper qualifies automatically.
4. Otherwise return `model`.

Do not walk `model_short_alias_map`. Do not add a leading `@` (this pill is never an
alias). Empty provider or model should not occur on this widget; if a caller passes only
one side, fall back to that side rather than inventing `PROVIDER(model)`.

The helper must read only the cached LLM metadata payload (`model_to_provider_map()`,
`registered_provider_names()`). That payload is already warmed by launch-default
resolution and is a dict lookup, not a lock, glob, or plugin invoke. Optional
`model_to_provider=` / `provider_names=` kwargs keep unit tests off the live catalog.

Round-trip examples (effort suffix applied by the existing label helpers, not this
function):

| Resolved `(provider, model)`                        | Compact subject                              | Why                                       |
| --------------------------------------------------- | -------------------------------------------- | ----------------------------------------- |
| `("grok", "grok-4.6")`                              | `grok-4.6`                                   | Unique `model_to_provider` hit            |
| `("claude", "opus")`                                | `opus`                                       | Unique hit                                |
| `("codex", "o3")`                                   | `o3`                                         | Unique hit                                |
| `("opencode", "anthropic/claude-sonnet-4-5")`       | `anthropic/claude-sonnet-4-5`                | Unique hit; `anthropic` is not a provider |
| `("codex", "visual-snapshot-model")`                | `codex/visual-snapshot-model`                | Not in the map                            |
| `("verylongprovider", "extremely-long-model-name")` | `verylongprovider/extremely-long-model-name` | Not in the map                            |

`%model:grok-4.6@high` is then the string a user can paste to get the same launch target
plus the shown effort.

### Visible grammar

Calm default (dim cyan, `_DEFAULT_STYLE` on the whole string, including the suffix):

```text
 grok-4.6
 grok-4.6@high
 opus@none
 opencode/unknown-model@high
```

Gold default-model override (existing two-tone `build_override_pill`):

```text
 grok-4.6 1h2m
 grok-4.6@medium 1h2m
 grok-4.6 ∞
```

Keep the compact `@<level>` connective on the pill. Keep the existing spaced tooltip
connective:

```text
Launch default: grok-4.6
Launch default: grok-4.6 @ high
```

Gold tooltip target in this widget only: `grok-4.6 @ high`, not `GROK(grok-4.6) @ high`.
Implement that at the `LLMOverrideIndicator._build_tooltip` call site. Do **not** change
shared `format_tooltip_target` in `src/sase/ace/tui/widgets/_override_pill.py`;
`alias_overrides_indicator.py` still renders `@medium -> CLAUDE(opus) @ xhigh`.

Configured `none` still renders as `@none`. Unset effort still omits the suffix.
Placeholder `...` and `unavailable` stay suffix-free and model-free.

The gold pill still shows only override-borne effort. Do not paint the global default
effort onto a gold pill whose override carries none.

### Widget wiring

Replace `format_provider_model_label(provider, model)` with
`format_model_directive_label(provider, model)` in:

- `_format_default_label`
- `_format_default_tooltip_label`
- `_build_override_content` (the gold subject)

Leave `_cached_default: tuple[str, str]` as `(provider, model)`. Do not store a
pre-rendered compact string on `_LaunchDefaultSnapshot`; the helper is a cached dict
lookup, and tests already stub provider/model independently of spelling.

Do not resolve, glob, or take override locks on the UI thread or in the 5-second tick
(`tui_perf.md` rules 1, 8, 11).

### Docs

- `docs/ace.md` Launch Control paragraphs that currently say the calm pill and the gold
  default-model override render `PROVIDER(model)[@<effort>]`: describe the compact
  `%model` subject (`grok-4.6`, or `provider/model` when the bare name would not route)
  plus the existing optional `@<effort>` suffix. Update the round-robin tooltip example
  the same way.
- `docs/llms.md` Reasoning Effort sentence that currently says the top-bar pill shows
  `PROVIDER(model)[@<effort>]`: same compact grammar.
- `CHANGELOG.md`: `feat(ace)` entry for the compact spelling.

Do not rewrite Launch Control row badges, agent-list `PROVIDER(model)` copy, or the
xprompt completion docs.

### Visual hygiene

Visual startup (`tests/ace/tui/visual/_ace_png_snapshot_startup.py`) stubs
`provider="codex"`, `model="visual-snapshot-model"`, `effort=None`. That name is not in
`model_to_provider_map()`, so the header pill becomes `codex/visual-snapshot-model`
instead of `CODEX(visual-snapshot-model)`. Keep `effort=None`. If a targeted visual run
shows a header golden diff, inspect the `just fix-tui-screenshots` report before
treating the new bytes as approved. No dedicated PNG of the compact spelling is required
when header goldens do not move.

## Implementation

1. **Helper**
   - Add `format_model_directive_label` in `src/sase/llm_provider/registry.py` with the
     round-trip rules above and optional map kwargs for tests.
   - Re-export it from the registry public surface if `__all__` / façade listings
     include `format_provider_model_label`.

2. **Widget**
   - Import the helper in `llm_override_indicator.py`.
   - `_format_default_label` / `_format_default_tooltip_label`: compact subject, same
     `@effort` / ` @ effort` connectives as today.
   - `_build_override_content`: compact subject into `build_override_pill`.
   - Override tooltip: compact subject plus the existing spaced effort connective,
     without editing shared `format_tooltip_target`.

3. **Docs + changelog** as above.

## Tests

Extend existing files. Add a focused unit file for the helper next to other registry
tests if `tests/test_llm_provider_registry.py` is not the natural home.

Helper (`format_model_directive_label`), with injected maps:

- Unique mapped model returns the bare name (`grok`/`grok-4.6` → `grok-4.6`).
- Unmapped model returns `provider/model`.
- Mapped model whose first slash token is a registered provider returns
  `provider/model`.
- Mapped OpenCode nested name whose first slash token is **not** a provider returns the
  nested name (`anthropic/claude-sonnet-4-5`).
- Short aliases are not consulted (a map that includes `haiku45` as a hint still returns
  `claude-haiku-4-5` when that is the resolved model).

`tests/test_llm_override_indicator.py`:

- Calm default with configured `high` renders `o3@high` (not `CODEX(o3)@high`) in dim
  cyan.
- Alias-borne / temporary-effort / `@none` / unset-effort cases keep their current
  precedence and suffix rules, with compact subjects (`o3@medium`, `o3@none`, `o3`).
- Tooltip `Launch default:` line and round-robin extra line use `opus` / `opus @ high`.
- Cached-default content uses the compact subject.
- Gold override cases become `o3 1h2m`, `o3@medium 1h2m`, `o3 ∞`. Do not add default
  effort onto a no-effort gold override.
- Long unmapped labels render the qualified `provider/model` form fully, both calm and
  gold.
- Override tooltip for this widget uses `opus @ xhigh`, not `CLAUDE(opus) @ xhigh`.
- `__init__` / `refresh` still must not call `build_launch_model_setting_snapshot` or
  `resolve_effective_effort` on the UI thread. The new helper may run on the UI thread
  because it is a cached dict lookup; do not call `resolve_model_provider` from it.

Leave `tests/test_alias_overrides_indicator.py` asserting `CLAUDE(opus)`.

## Verification

While iterating:

```bash
pytest tests/test_llm_override_indicator.py \
       tests/test_llm_provider_registry.py \
       tests/test_alias_overrides_indicator.py
```

Include the helper's test path if it lands in a different file. Then `just fix` and
`just check`. If the scoped lane or a header-bearing visual test reports a PNG diff, run
a targeted `just fix-tui-screenshots -- <selectors>` and inspect the report; do not land
unreviewed golden updates.
