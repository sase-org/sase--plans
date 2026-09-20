---
tier: tale
title: Render the launch-default pill as its shortest %model spelling
goal:
  The ACE top-bar calm launch-default pill names the launch default with the smallest
  string the `%model` directive would accept (`grok-4.6@high`) instead of
  `PROVIDER(model)@effort` (`GROK(grok-4.6)@high`), while the hover tooltip keeps the
  provider-qualified form.
size: medium
proposed_by: bbugyi200.apollo.0s.f0
create_time: 2026-09-20 07:34:57
status: wip
---

# Plan: Render the launch-default pill as its shortest `%model` spelling

## Problem

`src/sase/ace/tui/widgets/llm_override_indicator.py` renders the calm (dim-cyan)
launch-default pill in the ACE top bar through
`sase.llm_provider.registry.format_provider_model_label`, which always produces
`PROVIDER(model)`. With the effort suffix added in the previous tale, the pill reads
`GROK(grok-4.6)@high`.

The `PROVIDER(...)` wrapper is redundant in this particular slot. The pill answers "what
will a prompt with no `%model` directive get?", so the most useful spelling is the one
the user would type into `%model` to pin that same target: `grok-4.6`. That is both
shorter (the top bar is width-constrained) and directly actionable.

## Goal

The calm launch-default pill renders `<shortest %model value>[@<effort>]`, for example
`grok-4.6@high` (or `grok-4.6` when no effort is configured). When the bare model name
is not by itself a valid, unambiguous `%model` value, the pill falls back to the
explicit `provider/model` spelling, e.g. `codex/o3@high`.

## Background: what `%model` accepts

`sase.llm_provider.registry.resolve_model_provider_with_cursor` resolves a `%model`
value in this order:

1. Alias resolution (`resolve_model_alias_with_effort`) rewrites the value when it names
   a configured or built-in model alias.
2. `prefix/rest` returns `(prefix, rest)` when `prefix` is a registered provider name.
3. `model_to_provider_map()` (plugin-supplied metadata) maps a bare model name to its
   provider.
4. Otherwise the value routes to whatever the default provider happens to be.

So for a resolved `(provider, model)` pair there are exactly two spellings worth
considering: the bare `model`, and the explicit `provider/model`. Case 4 is _not_ an
acceptable way to earn the bare spelling — it depends on the ambient default provider
and would silently change meaning the moment that default moves, so a bare name is only
used when `model_to_provider_map()` positively maps it to this provider.

## Design

### 1. New formatter: `src/sase/llm_provider/model_directive_label.py`

Add a module next to the existing label helpers exposing:

```python
def format_model_directive_label(
    provider: str | None = None,
    model: str | None = None,
) -> str:
    """Return the shortest ``%model`` value that resolves to (provider, model)."""
```

Behavior, in order:

1. When `model` is falsy, delegate to
   `sase.llm_provider.registry.format_provider_model_label(provider, model)` so the
   degenerate provider-only / empty cases keep today's strings (`GROK`, `Agent`).
2. When `provider` is falsy, return `model` unchanged — there is no provider to qualify
   with, and a bare unknown model already routes to the default provider.
3. Otherwise try the bare candidate `model`. Accept it only when **both** hold:
   - `model not in sase.llm_provider.model_alias_config.model_alias_names()` — an alias
     of the same name would shadow the model in `%model`, and an alias may be a rotating
     pool, so the spelling would not denote this concrete target.
   - `resolve_model_provider(model, consume=False) == (provider, model)`.
4. Otherwise return `f"{provider}/{model}"`. This is also the terminal fallback when the
   provider plugin is not registered (so the explicit form would not resolve today
   either) — it is still the spelling a user would type, and it never hides the
   provider.
5. Any exception raised while probing degrades to the step-4 explicit spelling. This
   function feeds a status indicator and must never raise.

`consume=False` is mandatory: the probe must never advance a load-balanced pool's
rotation cursor. The function is a pure read; it takes no routing locks of its own and
adds no caching, because (see below) it is only ever called from the indicator's
off-thread resolution worker at most once per launch-default change token.

Example results, given a stock plugin set:

| provider, model                                       | pill subject                |
| ----------------------------------------------------- | --------------------------- |
| `grok`, `grok-4.6` (mapped)                           | `grok-4.6`                  |
| `claude`, `opus` (mapped)                             | `opus`                      |
| `codex`, `some-unmapped-model`                        | `codex/some-unmapped-model` |
| `claude`, `large` (a model name shadowed by an alias) | `claude/large`              |
| `""`, `mystery-model`                                 | `mystery-model`             |

Export it from `sase/llm_provider/config.py`'s re-export block only if that block is
where the sibling label helpers are already surfaced; otherwise import it directly from
the new module. Do not widen `registry.py`'s public surface unnecessarily — Symvision
flags unused re-exports (`sase/memory/symvision.md`).

### 2. Wire it into the calm default lane only

In `src/sase/ace/tui/widgets/llm_override_indicator.py`:

- Add `directive_label: str | None = None` to `_LaunchDefaultSnapshot`.
- In `_schedule_default_resolution_if_needed().task()` — which already runs off the UI
  thread and already calls `resolve_effective_effort` — compute
  `format_model_directive_label(snapshot.provider, snapshot.model)` and store it on the
  returned `_LaunchDefaultSnapshot`. This keeps every render path reading a pre-resolved
  string, satisfying rules 1, 8, and 10 of `sase/memory/tui_perf.md`. Do **not** call
  the formatter from `refresh()`, `_apply_content()`, or
  `_build_cached_default_content()`.
- Change `_format_default_label` to take the already-computed subject string plus
  effort: `_format_default_label(subject: str, effort: str | None) -> str`, still
  returning `f"{subject}@{effort}"` or `subject`.
- In `_build_cached_default_content()`, use `self._cached_snapshot.directive_label` when
  present, and fall back to `format_provider_model_label(*self._cached_default)` when
  the cached snapshot is missing or carries no label. The two caches are written and
  cleared together today, but the fallback keeps a stale/partial cache from rendering an
  empty pill.
- Update the synchronous `_build_default_content()` path (kept for tests and non-widget
  callers) the same way, inside its existing `try` so a formatter failure still yields
  `_UNAVAILABLE_TEXT` rather than propagating.

### 3. Tooltip keeps `PROVIDER(model)`

`_build_tooltip` and `_format_default_tooltip_label` are unchanged. The pill is the
width-constrained surface and is where compactness pays; the tooltip is the long-form
surface and stays the place that names the provider, so the change loses no information
— it moves the provider one hover away. The round-robin line
(`@large rotates across N models; CLAUDE(opus) @ high is next.`) likewise stays on the
provider-qualified form.

This is a deliberate asymmetry; call it out in the docs edit (step 5) so it reads as a
decision rather than an oversight.

## Non-goals

Everything below deliberately keeps `PROVIDER(model)`:

- The **gold temporary-override pill** rendered by the same widget
  (`_build_override_content` → `build_override_pill`). An override is a target the user
  explicitly chose including its provider, and naming the provider back to them is the
  point; the previous tale left this lane alone for the same reason.
- The violet alias-override pills (`alias_overrides_indicator.py`), Launch Control /
  models-panel rows, agent panels, `sase agent show`, notification toasts, and the Plan
  Review modal badge — every other `format_provider_model_label` caller.
- **Aliases are never used as the pill subject**, even when shorter than the model name.
  `%model:@large` is accepted and is fewer characters, but an alias may be a rotating
  pool or an ordered fallback, so it does not denote the one concrete model the pill is
  reporting. Alias provenance already has its own surface: the tooltip's rotation line
  and `snapshot.referenced_alias`.
- No Rust core work. `sase/memory/rust_core_backend_boundary.md` asks whether another
  frontend would need this behavior to match the TUI; the answer is "eventually", but
  the inputs this formatter reads (`model_to_provider_map()` from plugin metadata, the
  model-alias config, the provider registry) all live in Python in this repo alongside
  the incumbent `format_provider_model_label`, and no `../sase-core` checkout is even
  present in this workspace. Adding a sibling formatter next to the existing one is the
  consistent placement; relocating the whole label/resolution surface to Rust is
  separate, much larger work.

## Tests

### New: `tests/test_model_directive_label.py`

Monkeypatch `model_to_provider_map`, `resolve_model_provider`, and `model_alias_names`
at their use sites in the new module so the cases are hermetic:

1. Bare model returned when the map points the model at this provider.
2. Explicit `provider/model` when the map points the bare name at a _different_
   provider.
3. Explicit `provider/model` when the bare name is unmapped (i.e. would only resolve via
   the ambient default provider).
4. Explicit `provider/model` when the bare name collides with a configured or built-in
   alias name, even though `resolve_model_provider` would round-trip it.
5. Explicit `provider/model` when the probe raises.
6. Falsy provider returns the bare model; falsy model delegates to
   `format_provider_model_label` (assert `GROK` for provider-only and `Agent` for
   neither).
7. Assert the probe is called with `consume=False`, so a future refactor cannot silently
   start advancing the pool cursor from a display path.

### Updated: `tests/test_llm_override_indicator.py`

Roughly six pill assertions expect `" CLAUDE(sonnet) "` / `" CLAUDE(opus) "` (around
lines 181, 208, 369, 450). Give `_prepare_indicator` / the `_snapshot` helper a way to
supply the directive label, and update those to the new subject. Tooltip assertions
(lines ~274, ~296, ~536, ~557, ~659) must stay exactly as they are — that is the
regression guard for step 3.

Add one focused test asserting both halves at once: with `(claude, opus)` resolving to a
bare `opus` and effort `high`, the pill plain text is `" opus@high "` while
`_build_tooltip(None)` still starts with `Launch default: CLAUDE(opus) @ high`.

Add one test that a cached default with `directive_label=None` still renders
`" CLAUDE(opus) "` (the step-2 fallback).

### Checked, and updated if it asserts pill text

`tests/test_launch_default_indicator_pool_rotation.py` — confirm whether it asserts
rendered pill strings and update accordingly.

### Visual snapshots — expect broad, mechanical golden churn

`tests/ace/tui/visual/_ace_png_snapshot_startup.py` stubs
`build_launch_model_setting_snapshot` to `provider="codex"`,
`model="visual-snapshot-model"`. Add a matching monkeypatch of
`llm_override_indicator.format_model_directive_label` returning
`"codex/visual-snapshot-model"` — the same value the real formatter would produce for an
unmapped model, but pinned so goldens do not depend on which provider plugins happen to
be installed on the capturing machine. Place it beside the existing
`peek_launch_default_change_token` / `peek_active_temporary_override` stubs.

The pill subject changes from `CODEX(visual-snapshot-model)` (26 chars) to
`codex/visual-snapshot-model` (27), so the top bar shifts by a column and **most ACE PNG
goldens will change**. Per `sase/memory/lint_and_test.md`: `just check` does not run PNG
snapshots, so run `just fix-tui-screenshots` explicitly, and run it through
`/sase_monitor` with the `TESTING` / `TESTED` status pair with an increased timeout — a
full run routinely outruns one agent turn. Then inspect `latest-report.json` under
`.pytest_cache/sase-visual/`: every creation/removal, and each update group's
representative plus members. Generation is not approval — the only expected difference
in each diff is the top-bar pill text and the one-column shift it causes. Anything else
in a diff is a real finding, not a rubber stamp.

## Docs

- `docs/ace.md` (~line 4065): the calm default pill paragraph currently says
  `PROVIDER(model)[@<effort>]`. Restate it as "the smallest `%model` value that would
  pin this exact target — a bare model name such as `grok-4.6` when the model
  unambiguously names its provider, otherwise the explicit `codex/o3` form — plus the
  optional `[@<effort>]` suffix". Keep the hover line at
  `<alias> rotates across N models; PROVIDER(model)[@<effort>] is next` (~line 4075) and
  add one sentence noting the tooltip deliberately keeps the provider-qualified form.
  Leave the gold-pill line (~4044) untouched.
- `docs/llms.md` (~line 1673): same substitution in the "top-bar launch-default pill
  shows the same launch-effective default as `PROVIDER(model)[@<effort>]`" sentence.
- No hand-edited changelog entry; the changelog is generated from the conventional
  commit message.

## Verification

1. `just install` first — this workspace clone may have sat unused while pinned deps
   moved.
2. `just fix` (or at minimum `just fmt`) inline before handing anything to a monitor.
3. `sase tool run check` for the lint gates plus the diff-scoped test lane. Do **not**
   run `just check-full`; it is not requested here.
4. `just fix-tui-screenshots` through `/sase_monitor` as described above, with report
   inspection before finalization.
5. Sanity-check the real rendering by launching the TUI and confirming the top-right
   pill reads as the bare model name for the machine's configured default, and that
   hovering still names the provider.
