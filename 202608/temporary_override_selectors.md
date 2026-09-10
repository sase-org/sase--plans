---
tier: epic
title: Full selector support for temporary model alias overrides
goal: "A temporary model alias override set from the ACE Models panel accepts the same
  `|` round-robin pool and `||` ordered fallback grammar as a persistent alias value,
  stores the selector verbatim, evaluates it at launch (availability-filtered rotation
  for pools, availability-ordered first-winner for fallbacks), and renders its members
  in the Models panel and top-bar pills instead of silently corrupting the model string.

  "
phases:
  - id: state
    title: Selector-aware override state and write path
    depends_on: []
    size: medium
    description:
      "state: add selector fields to TemporaryLLMOverride, bump the override state file
      to v3 with tolerant v2/v1 reads, validate and snapshot selector input on write,
      and namespace override-owned rotation cursors."
  - id: resolve
    title: Selector evaluation in alias resolution and launch lanes
    depends_on:
      - state
    size: medium
    description:
      "resolve: evaluate an override-borne selector inside the alias resolver with
      cycle/nesting fail-closed rules, expose reusable override selector details, and
      route every launch-default lane through the selector-aware path with correct
      rotation consumption."
  - id: view
    title: Display data layer for override-owned selectors
    depends_on:
      - resolve
    size: small
    description:
      "view: teach AliasView which side owns an active selector, derive the effective
      provider/model/effort from the live selection, and keep the completion overlay and
      doctor selection context truthful."
  - id: tui
    title: Models panel input flow, rendering, and top-bar pills
    depends_on:
      - view
    size: medium
    description:
      "tui: accept selector text in the temporary Override flow with pre-write
      validation, render override-owned pools and fallbacks in the row state tag and
      description strip, and make both top-bar override pills honest about selectors."
  - id: docs
    title: Documentation and doc-sync updates
    depends_on:
      - tui
    size: small
    description:
      "docs: retract the config-only selector claim for temporary overrides, document
      the v3 state schema and rotation-cursor namespacing, and update the ACE Models
      panel and pill descriptions."
proposed_by: bbugyi200.athena.sv
create_time: 2026-09-09 20:00:33
status: wip
---

- **PROMPT:**
  [prompts/202608/temporary_override_selectors.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/temporary_override_selectors.md)

# Plan: Full selector support for temporary model alias overrides

## Problem

The Models panel offers two ways to change an alias:

- **Persistent Edit (`e`)** already supports selectors. `models_panel_alias_edit.py:139`
  routes any value containing `|` past the effort picker, validates it with
  `validate_model_alias_selector_value()`
  (`src/sase/llm_provider/model_alias_resolution.py:395`), and writes it verbatim into
  `sase.yml`.
- **Temporary Override (`o`)** does not. `models_panel_override.py:154`
  (`_on_custom_picked`) treats the typed text as a single target. `set_alias_override()`
  then calls `resolve_model_provider_with_effort()`, which never parses selector syntax
  for a free-form value, so `claude/opus || codex/gpt-5.6-sol` is accepted without any
  error and stored as `provider="claude"`, `model="opus || codex/gpt-5.6-sol"`. That
  bogus model string is what a later launch passes to the provider CLI.

Selector parsing today only ever runs on an alias _value_ pulled from configuration
(`_resolve_model_alias_result` in `model_alias_resolution.py:209-247`), never on the
override snapshot, which the module docstring describes as deliberately snapshot-only:
_"A temporary override suspends selector behavior for that alias"_.

The chosen resolution is **full selector support**: store the raw selector on the
override and evaluate it at launch.

## Scope

In scope: temporary per-alias overrides written to `~/.sase/llm_override.json` (the
Models panel `o` flow, plus the `set_alias_override` / `set_alias_override_until` /
`set_temporary_override` API).

Explicitly out of scope, and still single-target after this epic: `%model` directive
values and launch-scoped `%model(alias=...)` overrides. The documentation phase must
narrow, not delete, the existing "selector expressions are config-only" sentence.

## Design decisions

These apply across phases; each phase implements its slice.

1. **The selector is stored raw and evaluated live.** `raw_model` keeps the verbatim
   text the user typed, and the parsed mode plus ordered members are stored alongside
   it. The existing `provider` / `model` / `effort` fields keep their meaning for a
   plain override and, for a selector override, hold the **write-time** selection (a
   peek, no cursor consumption). Any reader that does not understand selectors therefore
   still sees a single valid member rather than a corrupt string.
2. **Rotation state is namespaced.** An alias such as `cheap` can own a configured pool
   _and_ carry a pool override at the same time. Both must rotate independently, so an
   override-owned pool keys its cursor in `~/.sase/llm_lb.json` under a namespaced owner
   rather than the bare alias name.
3. **Fail closed, exactly like configured selectors.** A selector reached from inside
   another selector, a member that cycles back to the overridden alias, and an
   empty/mixed selector are all invalid. Resolution returns the already-established
   `valid=False` result, which surfaces as the original input.
4. **The keystroke path stays free of new I/O.** Per `sase/memory/tui_perf.md` rules 8
   and 11, selector evaluation on the `%model` completion path may only use
   already-cached lookups — `model_to_provider_map()` and the `lru_cache`d
   `provider_cli_available()` — and the time-gated `peek_active_alias_overrides()` read
   that path already performs. No new disk reads, locks, or subprocesses on a keystroke.

## Selector-aware override state and write path

Files: `src/sase/llm_provider/temporary_override.py`,
`src/sase/llm_provider/load_balancing.py`.

### Dataclass and schema

Add two fields to `TemporaryLLMOverride`:

```python
selector_mode: ModelAliasSelectorMode | None = None
selector_members: tuple[str, ...] = ()
```

Bump `_STATE_VERSION` to `3` and extend `_entry_from_dict()`:

- Absent or `null` `selector_mode` means a plain override; `selector_members` must then
  be absent or empty. This is what every existing v2 entry looks like, so v2 files keep
  loading unchanged.
- A present `selector_mode` must be `"round_robin"` or `"fallback"`, and
  `selector_members` must be a JSON list of at least two non-empty strings (coerced to a
  `tuple[str, ...]`).
- Anything else makes the entry structurally invalid, which the existing loader already
  handles by dropping the entry — keep that behavior, do not raise.

`_extract_raw_entries()` needs no structural change: a v2 file still carries an
`"overrides"` key, so it is accepted, reports `canonical=False`, and is rewritten as v3
on the next self-cleaning read. Keep the v1 flat-object migration.
`_serialize_overrides()` continues to use `asdict()`; the tuple serializes as a JSON
list and round-trips through the `_entry_from_dict()` coercion above.

Forward-compatibility note worth a comment: an older build reading a v3 file ignores the
unknown keys and falls back to the snapshotted `provider` / `model`, which is a degraded
but safe single target — not the corrupt string this epic fixes.

### Write path

In `_write_alias_override()`, before the existing single-target resolution:

1. `selector = parse_model_alias_selector(cleaned)`. Convert `ModelAliasSelectorError`
   into a `ValueError` carrying the parser's message (empty members, mixed `|`/`||`) so
   the Models-panel worker's `Could not set override: <detail>` toast is actionable.
2. When `selector is None`, behavior is exactly as today.
3. When `selector is not None`:
   - Run `validate_model_alias_selector_value(cleaned_alias, cleaned)` and raise
     `ValueError(errors[0])` when it reports anything. This reuses the persistent-edit
     validator, so unknown `@alias` members, cycles back through the overridden alias,
     nested selectors, and depth-limit violations are rejected before anything is
     written. Note the validator walks _configured_ alias values only; a member that
     reaches another alias's active selector override is caught later by the fail-closed
     resolution rule instead of here. Say so in a comment rather than duplicating
     override state into the validator.
   - Snapshot the write-time winner: resolve each member with
     `resolve_model_provider_with_effort(member)` (the default `consume=False`), build
     the availability mask with `resolved_target_is_available()`, then pick with
     `select_model_alias_fallback_member(availability)` or
     `select_model_alias_pool_member(override_cursor_owner(cleaned_alias), selector, availability, consume=False)`.
     Store that member's provider/model/effort in `provider` / `model` / `effort`. Keep
     the existing "provider is `None` → `resolve_effective_default_provider_model()`"
     fallback.
   - Store `selector_mode=selector.mode`, `selector_members=selector.members`, and
     `raw_model=cleaned`.

`set_alias_override()` and `set_alias_override_until()` need no signature change — they
already funnel through `_write_alias_override()`.

### Rotation cursor namespacing

Add to `load_balancing.py`:

```python
#: Cursor-owner namespace for a pool that comes from a temporary override.
OVERRIDE_CURSOR_NAMESPACE = "override:"

def override_cursor_owner(alias: str) -> str:
    """Return the ``llm_lb.json`` cursor owner for an override-owned pool."""
```

The `:` separator is not part of the `@alias` grammar, so a namespaced owner cannot
collide with a real alias name. Deliberately do **not** prune the cursor entry when an
override is cleared or expires: `select_model_alias_pool_member()` already resets to
member zero on a fingerprint mismatch, which is the same way an edited configured pool
behaves today. Record that reasoning in a comment so a later reader does not add pruning
under a different lock.

### Tests

Extend `tests/llm_provider/test_temporary_override.py` and
`test_temporary_override_phase2.py`:

- Writing `a | b` and `a || b` stores mode, members, verbatim `raw_model`, and a
  snapshot equal to the peeked winner.
- A v2 file on disk loads unchanged and is rewritten as v3 on the next authoritative
  read; a v1 flat object still migrates to `overrides.default`.
- Entries with a bogus `selector_mode`, a one-member list, a non-list, or non-string
  members are dropped rather than raising, and the file self-cleans as it does for other
  malformed entries.
- Empty (`a ||`), mixed (`a | b || c`), unknown-`@alias`, self-cycling, and
  nested-selector input raises `ValueError` with the parser's or validator's message.
- `override_cursor_owner("cheap")` does not collide with the configured `cheap` pool
  cursor: a configured pool and an override pool on the same alias advance independently
  in `llm_lb.json`.

## Selector evaluation in alias resolution and launch lanes

Files: `src/sase/llm_provider/model_alias_resolution.py`,
`src/sase/llm_provider/registry.py`, `src/sase/llm_provider/temporary_override.py`,
`src/sase/llm_provider/_invoke.py`.

### Resolver

Inside `_resolve_model_alias_result.resolve()`, the override branch currently returns
`f"{override.provider}/{override.model}"` immediately
(`model_alias_resolution.py:189-201`). Replace that early return with a dispatch:

- No `selector_mode` → today's snapshot return, unchanged.
- With a `selector_mode`:
  - If `selector_owner is not None`, `return fail()` — an override selector reached from
    inside another selector is a nested selector, matching the existing
    configured-selector rule at line 224.
  - If `bare in seen`, `return fail()`; otherwise `seen.add(bare)` **before** resolving
    members. Today `seen.add(bare)` happens only on the configured-target path, so
    without this a member `@<the overridden alias>` would loop.
  - Resolve every member with the existing recursive call shape:
    `resolve(member, seen=set(seen), steps=steps + 1, selector_owner=bare, inherited_effort=effort)`.
    If any member is invalid, `return fail()`.
  - Build the availability mask through the same
    `config.__dict__.get("_resolved_target_is_available", ...)` indirection the
    configured path uses, so existing monkeypatch points keep working.
  - Select with `select_model_alias_fallback_member(availability)`, or
    `select_model_alias_pool_member(override_cursor_owner(bare), ModelAliasSelector(mode=..., members=...), availability, consume=consume)`
    for a pool. Threading `consume` is what makes an authoritative launch advance the
    cursor exactly once.
  - Return the chosen member's result. Its `selector_alias` is `bare`, which is what
    makes `resolve_model_provider_with_effort()` keep an explicit provider prefix for an
    unregistered provider (`registry.py:336`) — the same diagnostic behavior configured
    selectors get.

Factor the member-resolution / availability / selection block shared with the configured
path into one private helper rather than copying it.

### Reusable selector details

`model_alias_selector_details(name)` (line 344) builds display details from the
_configured_ value only. Refactor its body into a shared
`_selector_details(owner, selector, cursor_owner)` and add a sibling:

```python
def override_selector_details(alias: str, override: TemporaryLLMOverride) -> _ModelAliasSelectorDetails | None
```

which returns `None` for a plain override and otherwise peeks (`consume=False`) the
override's members exactly as the configured variant does. Also add:

```python
def override_effective_target(alias: str, override: TemporaryLLMOverride) -> tuple[str | None, str, str | None]
```

returning `(provider, model, effort)` — the live selection for a selector override, and
the stored snapshot for a plain one. This is the single helper every display surface
calls, so the panel and the pills cannot drift apart. Export both from
`sase/llm_provider/__init__.py` alongside the existing selector exports.

### Launch-default lanes

Four places short-circuit on a `default` override and return its snapshot directly. Each
must delegate to the selector-aware resolver **only when the override carries a
selector**, keeping the direct snapshot return for plain overrides so a
stored-but-now-unregistered provider keeps behaving exactly as it does today:

- `temporary_override.resolve_effective_default_provider_model()` (line 653) and
  `resolve_effective_default_provider_model_with_effort()` (line 687) — delegate to
  `resolve_model_provider[_with_effort]("@default", launch_overrides, consume=consume)`.
- `registry.resolve_default_alias_provider_model()` (line 364) and
  `resolve_default_alias_provider_model_with_effort()` (line 399) — same delegation.
- `registry.get_default_provider_name()` (line 489) — return the live-selected member's
  provider, falling back to `get_configured_default_provider_name()` when that member
  has no explicit provider.
- `_invoke.py:209` — the `elif active is not None:` branch assigns `active.provider` /
  `active.model` / `active.effort` directly. For a selector override it must instead
  call
  `resolve_effective_default_provider_model_with_effort(model_tier, model_alias_overrides, consume=True)`,
  matching the sibling branches. Rotation must advance exactly once per launch: verify
  no other call on the same launch path also passes `consume=True` for the same alias.

### Tests

Extend `tests/llm_provider/test_alias_override_resolution.py`,
`test_temporary_override_resolution.py`, `test_registry_resolution.py`, and
`test_load_balanced_aliases.py`:

- A pool override on `@coder` rotates across members on successive `consume=True`
  resolutions and skips unavailable members; a peek (`consume=False`) never advances the
  cursor.
- A fallback override picks the first available member, is stateless, and preserves
  member zero when nothing is available.
- A pool override on an alias that also owns a configured pool leaves the configured
  cursor untouched and vice versa.
- An override selector whose member is `@<the overridden alias>`, whose member reaches
  an alias owning a configured selector, or which is reached from inside another
  selector, fails closed to the resolver's input.
- A `default` pool override drives
  `resolve_effective_default_provider_model_with_effort()` and a real
  `invoke_agent()`-shaped launch, advancing the rotation exactly once per launch.
- Per-member trailing effort (`claude/opus@high | codex/gpt-5.6-sol@low`) is carried by
  the selected member, and an explicit outer `@coder@xhigh` reference still wins.

## Display data layer for override-owned selectors

Files: `src/sase/llm_provider/alias_view.py`, `src/sase/xprompt/model_completion.py`,
`src/sase/doctor/checks_providers.py`.

### AliasView

`build_alias_views()` currently always asks `model_alias_selector_details(name)` — a
config-only lookup — and lets an active override win the effective provider/model
(`alias_view.py:456-467`). Add a field so rendering can tell the two apart:

```python
selector_source: Literal["configured", "override"] | None = None
```

Selection logic per row:

- Active override with a selector →
  `selector = override_selector_details(name, override)`,
  `selector_source = "override"`, and provider/model/effort from that selector's
  selected member.
- Active override without a selector → unchanged: the configured selector details still
  populate `selector_mode` / `selector_members` (that is what drives the existing
  "suspended by override" rendering), `selector_source` is `"configured"`, and
  provider/model/effort come from the override snapshot.
- No override → unchanged, `selector_source` is `"configured"` when a selector exists.

Keep `build_alias_views(overrides=...)` working: the injected mapping must feed the
override selector details too, so tests can pin state without touching disk.

Update `tests/_models_panel_helpers.py::make_alias_view` and
`tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py` with the new keyword
so fixtures stay expressive.

### Completion overlay

`_apply_alias_overrides()` in `model_completion.py:200` runs on the ACE `%model`
keystroke path and currently forces `selector_mode=""`, `pool_available=0`,
`pool_total=0` on an overridden alias — the wire encoding of "an override suspends the
selector". For a selector override it should instead populate `selector_mode`,
`pool_available`, `pool_total`, and `target_provider` / `target_model` / `target_effort`
from `override_selector_details()`.

This is allowed on the keystroke path because every lookup it needs is already cached
process-wide (`model_to_provider_map()`, `provider_cli_available()`), the override map
itself comes from the time-gated `peek_active_alias_overrides()` this path already
calls, and the work is bounded by active-override count × members. Confirm with a bench
(`pytest -s -m slow tests/ace/tui/bench_tui_jk.py`) that p95 key-to-paint does not
regress, and state in a comment why this resolution is safe here.

`model_completion_catalog_payload()` — the launch-time JSON snapshot consumed by the
Rust `sase_xprompt_lsp` crate — is built with `overrides={}` and therefore never sees
temporary overrides. Its wire shape is unchanged and no Rust change is required; verify
that before touching anything in `sase-core`.

### Doctor

`checks_providers.selection_context()` (line 103) reports the active `default` override
as `{"reason": "temporary_override", "provider": ..., "model": ...}`. For a selector
override, add `"selector": override.raw_model` and
`"selector_mode": override.selector_mode`, and report the live-resolved provider/model
through `override_effective_target()` so `sase doctor` does not claim a stale member.
Update `tests/doctor/test_checks_providers.py`.

### Tests

Extend `tests/llm_provider/test_alias_view_overrides.py` and
`tests/llm_provider/test_alias_view.py`: an override pool row reports
`selector_source="override"`, exposes its members with availability and the live
`selected` flag, and its effective badge matches the selected member; a plain override
over a configured pool still reports `selector_source="configured"` with the configured
members.

## Models panel input flow, rendering, and top-bar pills

Files: `src/sase/ace/tui/modals/models_panel_override.py`,
`src/sase/ace/tui/modals/models_panel_rendering.py`,
`src/sase/ace/tui/widgets/_override_pill.py`,
`src/sase/ace/tui/widgets/llm_override_indicator.py`,
`src/sase/ace/tui/widgets/alias_overrides_indicator.py`.

### Override input flow

Mirror the persistent-edit flow (`models_panel_alias_edit.py:132-155`) in
`_on_custom_picked()`:

- When the typed value contains `|`, skip `alias_reference_rejection()` (which only
  understands a single `@alias`) and skip the effort picker entirely — selectors carry
  per-member effort. Validate with
  `validate_model_alias_selector_value(self._pending_alias, raw_model)` and, on any
  error, notify `Cannot override @<alias>: <first error>` with `severity="warning"` and
  abort, exactly as `_open_model_edit_preview()` does. Otherwise set
  `_pending_raw_model` and go straight to `_open_duration_picker()`.
- The `ModelPickerModal` itself needs no change: selectors only arrive through
  `Custom...`, the same as Edit.
- Update the override flow's `CustomModelInputModal` copy (currently
  `models_panel_override.py:143-149`) to advertise selectors, matching the Edit wording:
  hint
  `Single values may end in @effort; selectors keep per-member effort: A@low | B@high`,
  placeholder `e.g. claude/opus || codex/gpt-5.6-sol`.
- In `_on_override_worker()`, the success toast builds `label` from
  `format_provider_model_label(...)`. For a selector override, use the verbatim selector
  plus its mode instead — for example
  `@coder override: fallback claude/opus || codex/gpt-5.6-sol for 1h` — so the toast
  does not imply a single pinned target.

Keep the write-side validation from the `state` phase as the defensive backstop; the
modal check exists to fail fast before the duration picker.

### Row rendering

In `models_panel_rendering.py`:

- `state_tag()` returns early with the override chip whenever an override is active.
  When `view.selector_source == "override"` and `view.selector_mode == "round_robin"`,
  also `append_pool_chip(...)` with the member availability counts, so an override pool
  reads `override · 15m left · pool 2/3`.
- `description_text_for_view()` computes `suspended = view.override is not None`. Change
  it to `suspended = view.override is not None and view.selector_source != "override"`,
  so an override-owned selector renders live — undimmed members and a `→` on the current
  selection — while a plain override over a configured selector keeps today's suspended
  rendering.
- Prefix the strip label with `override ` when the selector is override-owned:
  `override pool: …` / `override fallback: …` against the existing `pool: …` /
  `fallback: …`.

### Top-bar pills

`_override_pill.format_tooltip_target()` takes only the dataclass and cannot resolve
live. Give it the alias name (or an already-resolved target) so both lanes can describe
a selector.

- **Gold `default` pill** (`llm_override_indicator.py`). The pill subject must stay a
  concrete `PROVIDER(model)`. `_schedule_default_resolution_if_needed()` currently
  returns early whenever any override is active; relax that so a _selector_ default
  override also schedules the existing background worker, which already calls
  `resolve_effective_default_provider_model()` (peek-only, `consume=False`). Render the
  stored snapshot until the worker result lands, then the cached live winner. This keeps
  cold provider resolution off the UI thread, per `tui_perf.md` rules 1 and 2. Its
  tooltip names the selector mode and the verbatim expression.
- **Violet non-`default` pill** (`alias_overrides_indicator.py`). The pill body is
  already `@<alias>`, so it needs no target resolution. Its tooltip should render the
  selector expression and mode rather than a single target — for example
  `@coder -> fallback: claude/opus || codex/gpt-5.6-sol - 15m left` — and must not claim
  a current winner, since resolving one per tick would add UI-thread work for no real
  benefit. The Models panel remains the authoritative live view.

### Tests

- `tests/test_models_panel_override_flows.py`: a selector typed into `Custom...` skips
  the effort picker and reaches the duration picker; an invalid selector notifies and
  never opens it; the success toast names the selector.
- `tests/test_models_panel_alias_rendering.py` /
  `tests/test_models_panel_descriptions.py`: override pool and fallback rows render an
  undimmed member list with `→`, an override pool row carries the availability chip, and
  a plain override over a configured pool still renders suspended.
- `tests/test_llm_override_indicator.py` and `tests/test_alias_overrides_indicator.py`:
  pill and tooltip content for both selector modes, including the pre-worker snapshot
  rendering for the gold pill.
- Add one PNG snapshot for an override-owned pool row alongside the existing
  `pool_effort_views(suspended=...)` fixture, accepting it with
  `--sase-update-visual-snapshots`.

## Documentation and doc-sync updates

Files: `docs/llms.md`, `docs/ace.md`, and `docs/configuration.md` if it repeats the
config-only claim.

- `docs/llms.md`, selector section (~line 660): the sentence _"Selector expressions are
  config-only: `%model` values, launch-scoped alias overrides, and temporary overrides
  remain single targets"_ must be narrowed to `%model` values and launch-scoped alias
  overrides. Replace the following sentence about an override bypassing the alias's
  selector with the new rule: a **plain** override suspends the alias's configured
  selector for its lifetime, while a **selector** override replaces it and rotates or
  falls back on its own independent state.
- `docs/llms.md`, Temporary Model Overrides section (~line 999): describe selector
  overrides, per-member effort, the independent rotation cursor, and the fail-closed
  rules (no nesting, no cycle back through the overridden alias).
- `docs/llms.md`, State File section (~line 1055): bump the example to `"version": 3`,
  add a selector example, add `selector_mode` and `selector_members` to the field table,
  and extend the migration note — v2 entries without selector fields read as plain
  overrides, and an older build reading a v3 file degrades to the snapshotted single
  member.
- `docs/ace.md`, Models Panel (~line 2044 and the Temporary overrides section ~line
  2125): document that `Custom...` in the Override flow accepts selectors and skips the
  effort ladder, the `override pool:` / `override fallback:` strip labels, the
  availability chip on an override pool row, that "suspended" now applies only to a
  plain override over a configured selector, and the two pills' selector behavior.
- Update the `@cheap` / `@cheaper` / `@cheapest` wording in both files, which currently
  says an override on those aliases "suspends their independent load-balanced rotations"
  without qualification.
- Re-run any doc-sync test that covers these files
  (`tests/llm_provider/test_model_alias_defaults_docs_sync.py` and the docs checks under
  `tests/docs/`, if present).

## Cross-cutting verification

Run `just install` first (ephemeral workspace), then `just check`. Also run
`just test-visual` if the `tui` phase adds or updates a PNG snapshot.

Manual smoke worth doing once, in the `tui` phase: set `@coder` to
`claude/opus | codex/gpt-5.6-sol` for `15m` from the Models panel, confirm the row shows
`override · pool 2/2`, launch two agents, and confirm they land on different members
while the configured `@cheap` pool cursor in `~/.sase/llm_lb.json` is untouched.

## Backend boundary

`sase/memory/rust_core_backend_boundary.md` puts shared backend behavior in the linked
`sase-core` repo (open it with `sase repo open sase-core -r "<reason>"`, never by
guessing a path). `crates/sase_core` has no model-alias resolution, no selector parsing,
and no LLM-override state — the whole subsystem lives in Python under
`src/sase/llm_provider/`. The only Rust consumer is `crates/sase_xprompt_lsp`, which
renders hover text from the Python-produced model-completion JSON (`selector_mode`,
`pool_available`, `pool_total`) and is fed a snapshot built with `overrides={}`, so
temporary overrides never reach it.

This epic therefore stays in Python and must keep the model-completion wire shape
unchanged so no Rust change is needed. Do not port alias resolution to Rust as part of
this work; if a phase finds itself wanting to, stop and file a task bead instead.
