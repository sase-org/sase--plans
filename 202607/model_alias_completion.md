---
tier: epic
title: Model aliases in the %model completion menu
goal: 'Typing `%m:` / `%model:` shows model aliases as unmistakable, richly annotated
  rows beneath the concrete model names, each showing what it resolves to and how
  it was configured, and typing `@` after the colon narrows the menu to aliases only
  — in the ACE prompt input and in editors through the xprompt LSP.

  '
phases:
- id: gate
  title: Fix the `@` alias gate in the prompt-input directive grammar
  depends_on: []
  size: small
  description: 'gate: make a leading `@` in a `%model:` value stay in model-argument
    context instead of being read as an `@effort` suffix, matching the already-correct
    sase-core grammar.'
- id: catalog
  title: Enrich the model completion catalog with alias resolution and provenance
  depends_on: []
  size: medium
  description: 'catalog: derive alias rows from the same AliasView data the Models
    panel uses, add a read-only temporary-override peek, split the catalog into a
    config-token-cached static build plus a cheap override overlay, and extend the
    LSP catalog JSON additively.'
- id: rows
  title: Render alias rows in the ACE completion panel
  depends_on:
  - gate
  - catalog
  size: medium
  description: 'rows: add a shared model-alias presentation module used by both the
    Models panel and the completion menu, render `%model` values as an aligned four-column
    grid with alias kind badges and resolution badges, add the contextual panel title/subtitle,
    and warm the catalog off the keystroke path.'
- id: lsp
  title: Surface the alias detail through the xprompt LSP
  depends_on:
  - catalog
  size: medium
  description: 'lsp: consume the new catalog fields in sase-core''s xprompt LSP so
    editors show alias kind, resolution target, provenance, and description, with
    stable model-before-alias ordering.'
- id: polish
  title: Visual snapshots, docs, and help text
  depends_on:
  - rows
  - lsp
  size: small
  description: 'polish: add ACE PNG snapshots for the mixed and alias-only menus,
    and update the xprompt/LLM docs plus the ACE help popup to describe the `@` alias
    gate.'
create_time: 2026-07-29 07:46:18
status: done
bead_id: sase-ao
---

- **PROMPT:** [prompts/202607/model_alias_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/model_alias_completion.md)
- **BEAD:** [sase-ao](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ao/README.md)

# Plan: Model aliases in the `%model` completion menu

## Background

`%model:` / `%m:` completion already includes model aliases. `src/sase/xprompt/model_completion.py` builds one catalog
of `_ModelCompletionEntry` rows — concrete model names first (in provider order), then the implicit role aliases
(`@default`, `@coder`, each `@<provider>_coder`, `@epic_lander`, the five `@<size>_phase_worker` roles, `@smartest` …
`@cheapest`), then user-configured aliases alphabetically. The catalog feeds two surfaces: the ACE prompt input
(`_build_model_arg_completion_candidates` in `src/sase/ace/tui/widgets/directive_completion.py`) and the Rust xprompt
LSP, which reads the JSON written by `_materialize_model_catalog` in `src/sase/integrations/xprompt_lsp.py`.

Three things are missing or broken today:

1. **Alias rows are indistinguishable from model rows.** Every `directive_arg` row renders as `display` in magenta plus
   a dim `description`, so `@default  default model when a prompt has no %model` sits in the same visual grammar as
   `opus  Claude (opus)`. Nothing says "this is an alias", and nothing says what it resolves to or where it came from.
2. **The `@` gate is broken in ACE.** `extract_directive_arg_token_around_cursor`
   (`src/sase/ace/tui/widgets/_directive_completion_tokens.py:109-119`) redirects to the `effort` vocabulary whenever
   `prefix.rfind("@") >= 0`. Typing `%m:@` therefore opens the reasoning-effort menu (`none`, `minimal`, `low`, …)
   instead of the alias list. This contradicts both `split_model_effort` (`src/sase/xprompt/effort.py:52`, which uses
   `at <= 0` so a leading `@` is never an effort suffix) and the sase-core grammar, which already gets this right:
   `crates/sase_core/src/editor/completion.rs:1652-1665` matches `Some(rel_at) if rel_at > 0` under the comment _"A
   leading `@` is the model-alias marker and stays in model completion context"_, with a regression test at
   `crates/sase_core/src/editor/completion.rs:2624`. The parenthesized ACE form (`_extract_model_paren_arg_token`) is
   likewise already correct (`if at_index > 0`). Only the ACE colon form diverged.
3. **The Models panel already knows everything we want to show, and none of it reaches the menu.**
   `sase/llm_provider/alias_view.py` flattens alias policy, live resolution, and temporary overrides into `AliasView`
   rows; `src/sase/ace/tui/modals/models_panel_rendering.py` renders them as
   `<ownership> <kind> <name> <PROVIDER(model)> <state>` with a `configured` / `implicit → @fallback` /
   `override · 15m left` / `· pool 2/3` state tag.

## Design

### Row anatomy

`%model` argument rows become an aligned four-column grid shared by models and aliases, so the menu reads as one table
instead of two bolted-together lists:

```
┌ %model values ──────────────────────────────────────────────────────────────┐
│ ▸ claude-fable-5    model    Claude                          fable           │
│   opus              model    Claude                                          │
│   gpt-5.6-sol       model    Codex                                           │
│   @default          default  CLAUDE(opus) @ high             implicit        │
│   @coder            role     CODEX(gpt-5.6-sol)              configured      │
│   @claude_coder     coder    CLAUDE(opus)                    implicit → @coder │
│   @scout            custom   CLAUDE(sonnet-4-5)              configured · pool 2/3 │
│   ↓ 14 more…                                                                 │
└─────────────────────────────────── [@] model aliases ───────────────────────┘
```

Columns: **name**, **kind badge**, **target**, **state**.

- Model rows: kind badge `model` (magenta, the existing directive-value color), target = the provider display name
  styled with `provider_name_style()`, state = the short model alias hint (dim) when one exists.
- Alias rows: kind badge from the Models-panel vocabulary — `default` / `role` / `coder` / `custom` — in the
  Models-panel kind colors (`#87D7FF` / `#87D7AF` / `#AFAFFF` / `#D7AF87`), target = the same `PROVIDER(model)` badge
  the Models panel uses plus an ` @ <effort>` suffix when the alias carries one, state = the provenance tag.

The kind-badge column plus the per-kind color is what makes "this is an alias" unmistakable; the `@` is already in the
inserted value, and the ownership gutter is intentionally _not_ repeated (the completion panel already spends two cells
on the `▸` selection marker, and the `custom` badge plus the ownership accent color carry the same signal).

### State (provenance) vocabulary

Mirrors `state_tag()` in `models_panel_rendering.py`, minus the live countdown:

| State                  | Meaning                                                               |
| ---------------------- | --------------------------------------------------------------------- |
| `configured`           | Explicit `llm_provider.model_aliases.builtin`/`.custom` entry         |
| `configured → @ref`    | …whose value references another alias (` @ <effort>` appended if set) |
| `implicit`             | No config entry; resolved from the built-in fallback chain            |
| `implicit → @fallback` | …with the immediate fallback named                                    |
| `override`             | A temporary override is active; the target column shows its target    |
| `… · pool 2/3`         | Round-robin selector; available/total members                         |

The override chip deliberately omits `15m left`. Remaining time would go stale between keystrokes and is already shown
by the top-bar override pills; omitting it also keeps the catalog cache key free of the clock.

### Panel title and subtitle

- `border_title`: `%model values` normally, `model aliases` when the partial starts with `@`.
- `border_subtitle`: when the highlighted row is an alias, its long-form description (or, for a custom alias with none,
  `set llm_provider.model_aliases.custom.<name>.description`, matching the Models panel's missing- description hint).
  Otherwise `[@] model aliases` when the unfiltered catalog has aliases — which teaches the `@` gate at exactly the
  moment it is useful.

### Data flow and performance

`sase/memory/tui_perf.md` rules 8 and 11 govern this: completion is a keystroke path, so it must be read-only, must not
take shared-store locks, and must not re-read files per keypress. `build_alias_views()` currently calls
`get_active_alias_overrides()`, which takes an `flock` and can _write_ (it prunes expired entries and deletes the file).
That must never run inside key handling. The design:

1. `build_alias_views()` grows an optional `overrides` argument. `None` keeps today's self-cleaning load (the Models
   panel path); an explicit mapping is used verbatim.
2. A new `peek_active_alias_overrides()` in `sase/llm_provider/temporary_override.py` reads the state file **without**
   lock, prune, rewrite, or delete, filters expired entries in memory, memoizes on `(st_mtime_ns, st_size)` behind a
   short monotonic floor, and returns `{}` on any error.
3. `model_completion.py` splits into a **static** catalog memoized on `current_config_token()` (which is time-gated with
   off-thread refresh, so it is safe to call per keystroke) plus a pure **override overlay** that rewrites only the
   handful of overridden alias rows. An override change therefore never forces a full catalog rebuild on the keystroke
   path.
4. The prompt bar warms the static catalog in a background thread on mount, mirroring
   `_warm_vcs_project_completion_catalog`, so the first `%m:` opens instantly.

The LSP payload is built with `overrides={}`: it is a launch-time snapshot written by `_materialize_model_catalog`, so
baking in ephemeral overrides would be misleading. This is a deliberate difference between the two surfaces and is
documented in the module docstring.

### Wire compatibility

`MODEL_COMPLETION_CATALOG_SCHEMA_VERSION` **stays at 1** and the new fields are added additively. The Rust loader
rejects any `schema_version != 1` outright (`load_model_catalog`), so bumping it would blank the `%model` menu for
anyone on an older `sase-core-rs` until they upgraded. `model_entry` reads named keys and ignores unknown ones, so
additive-under-v1 is compatible in both directions: an old LSP ignores the new fields, and a new LSP reading a stale
catalog sees empty new fields and renders exactly as it does today.

### Non-goals

- Reordering aliases. The existing order (default, coder, the `@<provider>_coder` group, epic/phase roles, capability
  roles, then user aliases) intentionally groups the coder aliases together and is kept as-is.
- The bare `alias=` keyword completion inside `%model(...)` (`_build_model_alias_key_completion_candidates`). Those are
  alias _keys_, not values, and keep their current rendering.
- Effort completion after `=` in `%model(coder=opus@…)`. Today that offers nothing; making it symmetric with the colon
  form is a separate, unrelated grammar change.
- Rewriting the LSP catalog when a temporary override changes.

---

## Fix the `@` alias gate in the prompt-input directive grammar

**File:** `src/sase/ace/tui/widgets/_directive_completion_tokens.py`

In the colon branch of `extract_directive_arg_token_around_cursor`, change the `%model` effort redirect from
`if at_index >= 0:` to `if at_index > 0:`, and update the surrounding comment to state the rule explicitly: a leading
`@` is the model-alias marker and stays in model-argument context; only a non-leading `@` opens effort completion. This
is the exact rule already implemented and tested in `crates/sase_core/src/editor/completion.rs`, and it matches
`split_model_effort`'s `at <= 0` guard, so after this change ACE, sase-core, and the directive parser agree.

Resulting behavior:

| Input            | Context before   | Context after               |
| ---------------- | ---------------- | --------------------------- |
| `%m:@`           | `effort` `""`    | `model` `"@"`               |
| `%m:@def`        | `effort` `"def"` | `model` `"@def"`            |
| `%m:opus@`       | `effort` `""`    | `effort` `""` (unchanged)   |
| `%m:opus@xh`     | `effort` `"xh"`  | `effort` `"xh"` (unchanged) |
| `%m:@default@hi` | `effort` `"hi"`  | `effort` `"hi"` (unchanged) |

Because concrete model names can never begin with `@` (`_INLINE_MODEL_VALUE_RE` permits `@` but no registered or
configured model name starts with one) and every alias entry's value does, the existing prefix filter in
`filter_model_completion_entries` already yields an alias-only menu for a `@…` partial. The `catalog` phase makes that
explicit rather than incidental.

**Tests** — extend `tests/ace/tui/widgets/test_directive_arg_extraction.py`:

- Rename/extend `test_directive_arg_extraction_redirects_model_at_suffix_to_effort` so it keeps asserting the
  `%model:opus@` / `%model:opus@xh` behavior.
- Add a test asserting `%m:@`, `%m:@def`, and `%model:@claude_coder` all classify as `model` with the leading `@`
  included in the partial and in the replaced span (so accepting a candidate replaces `@def`, not `def`).
- Add a test asserting `%model:@default@hi` still classifies as `effort` with partial `hi`.

Also add a case to `tests/ace/tui/widgets/test_directive_arg_completion.py` asserting that
`build_directive_arg_completion_candidates("model", "@")` returns only `@`-prefixed insertions.

---

## Enrich the model completion catalog with alias resolution and provenance

### Read-only override peek

**File:** `src/sase/llm_provider/temporary_override.py`

Add `peek_active_alias_overrides(now: float | None = None) -> dict[str, TemporaryLLMOverride]`:

- Never acquires `_locked_state()`, never writes, never deletes — a corrupt or unreadable file yields `{}`.
- `os.stat`s `_state_path()`; a missing file yields `{}`.
- Reuses `_extract_raw_entries` / `_entry_from_dict` for parsing, and drops expired entries in memory.
- Memoizes the parsed result on `(st_mtime_ns, st_size)`, behind a short monotonic floor (≈0.5 s) so fast typing does at
  most one `stat` per floor window. Expiry filtering is applied on every call against the current clock, so the memo
  never hands back an expired override.
- Docstring states the contract: this is the _display_ read for keystroke paths; `get_active_alias_overrides` remains
  the authoritative, self-cleaning read for the Models panel and launch paths.

Export it from `sase/llm_provider/__init__.py` next to `get_active_alias_overrides`.

### Injectable overrides in the alias view

**File:** `src/sase/llm_provider/alias_view.py`

Give `build_alias_views` an `overrides: Mapping[str, TemporaryLLMOverride] | None = None` keyword. `None` preserves
today's `get_active_alias_overrides(now)` call; an explicit mapping (including `{}`) is used verbatim. No other behavior
changes.

### Catalog entry fields

**File:** `src/sase/xprompt/model_completion.py`

Extend `_ModelCompletionEntry` with flat, JSON-serializable fields (empty string / `0` for model rows):

| Field              | Type  | Source                                                              |
| ------------------ | ----- | ------------------------------------------------------------------- |
| `alias_kind`       | `str` | `AliasView.kind` — `default` / `role` / `provider_coder` / `user`   |
| `target_provider`  | `str` | `AliasView.provider` (`""` when the default provider applies)       |
| `target_model`     | `str` | `AliasView.model`                                                   |
| `target_effort`    | `str` | `AliasView.effort`                                                  |
| `provenance`       | `str` | `override` / `configured` / `implicit`                              |
| `reference`        | `str` | `AliasView.references` or `AliasView.implicit_fallback` (bare name) |
| `reference_effort` | `str` | `AliasView.reference_effort`                                        |
| `selector_mode`    | `str` | `AliasView.selector_mode`                                           |
| `pool_available`   | `int` | count of available `selector_members`                               |
| `pool_total`       | `int` | `len(selector_members)`                                             |
| `config_source`    | `str` | `AliasView.configured_source` — `builtin` / `custom`                |
| `bucket`           | `str` | `AliasView.bucket`                                                  |

`kind` keeps its existing `model` / `implicit_alias` / `user_alias` values so the Rust reader and existing assertions
stay valid; `alias_kind` carries the finer classification.

`description` for alias rows becomes the **long-form human description** — `model_alias_description(name)` for built-in
roles and configured customs, a generated `"<Provider> coder follow-up model."` for `@<provider>_coder`, and `""` for a
custom alias with no configured description. The old `"alias for <target>"` text is superseded by the structured target
fields. As a consequence, `_LEADING_IMPLICIT_ALIASES` / `_TRAILING_IMPLICIT_ALIASES` collapse to ordering-only name
tuples and their duplicated terse descriptions are deleted, so `ROLE_ALIAS_DESCRIPTIONS` in
`sase/llm_provider/model_alias_policy.py` becomes the single source. Model-row `description` is unchanged
(`"<Provider display> (<short alias>)"`).

### Build, cache, and overlay

Restructure the builder into:

- `_build_static_catalog()` — today's builder plus alias enrichment, calling `build_alias_views(overrides={})` once and
  indexing the result by name. Existing entry order is preserved exactly: alias rows are looked up from that index,
  never re-sorted by the Models-panel `_sort_key`. Enrichment is wrapped so a registry/config failure degrades to
  today's plain alias rows (empty new fields) rather than dropping alias entries.
- A single-slot memo keyed on `current_config_token()`, replacing the unbounded, never-invalidated `_CATALOG_CACHE`.
  Config edits now refresh the menu without a restart.
- `apply_alias_overrides(entries, overrides)` — pure, `O(len(overrides))`; for each overridden alias, rewrites
  `target_provider` / `target_model` / `target_effort` from the `TemporaryLLMOverride` and sets
  `provenance = "override"`, clearing `reference` / `reference_effort` / pool fields.
- `build_model_completion_catalog(*, use_cache=True, overrides=None)` — `None` means "no overlay" so existing callers
  are unchanged; the ACE path passes `peek_active_alias_overrides()`.

### Alias-only filtering

Make the `@` gate explicit in `filter_model_completion_entries`: when `partial` starts with `@`, restrict to entries
whose `kind` is an alias kind and prefix-match on both `value` and `"@" + bare alias`. Otherwise keep today's
value/alias-hint prefix match. Document that alias rows still surface for a bare partial (`de` matches `@default`
through its bare-name hint) but always after the model rows, per the catalog's order.

### Payload

`model_completion_catalog_payload()` serializes the new fields alongside the existing ones and keeps
`schema_version: 1`. Update the constant's docstring to record the additive-compatibility contract described in **Wire
compatibility** above, so a future reader does not "fix" it by bumping the version.

### Tests

- `tests/test_xprompt_model_completion.py`: update the existing order/description assertions; add coverage for alias
  enrichment (kind, target, provenance, reference, config source), for the round-robin pool counts, for the `@`
  alias-only filter, for the override overlay, and for graceful degradation when `build_alias_views` raises.
- New `tests/llm_provider/test_alias_override_peek.py`: peek returns `{}` for a missing/corrupt file, never creates or
  deletes the state file (assert the file is byte-identical after a peek of a file containing an expired entry), drops
  expired entries, and re-reads after an mtime change.
- `tests/llm_provider/test_alias_view.py`: `build_alias_views(overrides={})` performs no override load (monkeypatch
  `get_active_alias_overrides` to raise) and an injected mapping wins.
- `tests/main/test_lsp_handler.py`: the materialized payload still carries `schema_version == 1` and now includes the
  new keys.

---

## Render alias rows in the ACE completion panel

### Shared presentation module

**New file:** `src/sase/ace/tui/model_alias_styles.py`

Owns the single alias-presentation vocabulary used by both surfaces:

- `MODEL_ALIAS_KIND_LABELS` (`default` / `role` / `coder` / `custom`), `MODEL_ALIAS_KIND_STYLES`, `OWNERSHIP_ACCENT`,
  and `alias_kind_label(kind)`.
- Pure `rich.text.Text` builders over **plain scalars** (not `AliasView`), so the completion path never imports the
  modals layer: `append_effort_suffix(text, effort)`, `append_alias_reference(text, reference, effort)`,
  `append_pool_chip(text, available, total)`, and
  `alias_state_text(provenance, reference, reference_effort, pool_available, pool_total)`.
- `provider_model_text(provider, model, effort)` returning the themed `PROVIDER(model) @ effort` badge as a measurable,
  truncatable `Text` (moved from `models_panel_rendering._provider_model_text`).

`src/sase/ace/tui/modals/models_panel_rendering.py` is refactored to import from this module. Its public surface
(`kind_label`, `state_tag`, `OWNERSHIP_ACCENT`, `provider_model_column_width`, `render_alias_row`, …) and its rendered
output are unchanged — the override countdown chip stays panel-local, since only the panel has a `now`. Existing
Models-panel unit tests and PNG snapshots must stay green with no golden updates; treat any Models-panel snapshot diff
as a bug in the refactor, not an intentional change.

### Completion metadata and rows

**File:** `src/sase/ace/tui/widgets/directive_completion.py`

Add a frozen `ModelCompletionMetadata` dataclass carrying the entry's scalars (`value`, `kind`, `alias_kind`,
`provider`, `provider_display`, `short_alias`, `target_provider`, `target_model`, `target_effort`, `provenance`,
`reference`, `reference_effort`, `pool_available`, `pool_total`, `description`, `config_source`).
`_build_model_arg_completion_candidates` attaches it in place of the generic `DirectiveArgCompletionMetadata`, so
**only** `%model` argument rows change; every other directive argument (`%effort`, `%auto`, `%wait` keywords,
`clan_keyword`, `id_keyword`, `model_alias_key`) keeps its current rendering. It also passes
`peek_active_alias_overrides()` into `build_model_completion_catalog`.

`model_or_alias_key` (the first `%model(...)` slot) reuses the same enriched model candidates it already composes, so
paren and colon forms look identical.

**File:** `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows.py`

- `append_model_completion_row(content, candidate, is_selected, widths)` renders the four-column grid, dispatched from
  `append_directive_arg_completion_row` when the metadata is `ModelCompletionMetadata`.
- `model_completion_column_widths(visible)` computes the name and target column widths across the visible window (name
  capped at 30 cells, target capped at 34), mirroring `vcs_project_label_width`. Widths use Rich cell length, and each
  cell is truncated with `…` rather than allowed to wrap.
- Alias rows style the name with the kind color and bold when selected; model rows keep magenta.
- A row whose alias enrichment is empty (degraded catalog) falls back to `name  description`, so a metadata failure can
  never blank the menu.

**File:** `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`

- Compute the model column widths when the visible rows carry `ModelCompletionMetadata`, alongside the existing
  `vcs_*_width` computations.
- `border_title`: `model aliases` when `token.startswith("@")`, else `%model values`, replacing the generic
  `directive values` for this kind only.
- `border_subtitle`: the highlighted alias's description, the missing-description hint for a custom alias, or the
  `[@] model aliases` gate hint. Truncate to the panel's inner width.

### Warming

**File:** `src/sase/ace/tui/widgets/_file_completion_base.py`

Add `_warm_model_completion_catalog()` following `_warm_vcs_project_completion_catalog` exactly: guard flag,
`get_prompt_completion_settings` capability gate (so lightweight test harnesses skip it), and
`run_worker(..., name="prompt-model-catalog", thread=True)` calling `build_model_completion_catalog()`. Call it from
both warm sites: `PromptInputBar.on_mount` (`prompt_input_bar.py`) and the stack re-render path in
`_prompt_input_bar_stack_rendering.py`.

### Tests

- `tests/ace/tui/widgets/test_directive_arg_completion.py`: `%model` candidates carry `ModelCompletionMetadata`; alias
  candidates carry the resolved target and provenance; `@` yields only alias candidates; other directive arguments still
  carry `DirectiveArgCompletionMetadata`.
- New `tests/ace/tui/widgets/test_model_completion_rows.py`: assert the rendered plain text and styles for a model row,
  an implicit alias with a reference, a configured alias, a pooled alias, and an overridden alias; assert column
  alignment across a mixed visible window and `…` truncation of an over-long name.
- New `tests/ace/tui/test_model_completion_panel_titles.py`: title/subtitle for mixed vs alias-only, for an alias with
  and without a description.
- A keystroke-path regression test that monkeypatches `get_active_alias_overrides` and `_locked_state` to raise, then
  builds `%model` candidates successfully — proving the completion path never takes the override lock.

---

## Surface the alias detail through the xprompt LSP

Open the sibling repo with `/sase_repo` (`sase repo open sase-core`) and work only in the printed checkout.

**File:** `crates/sase_xprompt_lsp/src/server.rs`

- Extend `ModelCompletionEntry` and `model_entry` with the new optional fields, defaulting to empty strings / `0` so a
  stale v1 catalog still loads. Keep the `schema_version != 1` rejection as-is.
- In `model_completion_list`, build:
  - `detail` — `PROVIDER(model) @ effort` for aliases (falling back to the provider name for models), so the resolution
    target is visible in the inline completion list.
  - `documentation` — a short markdown block: the description, the provenance line (`configured` / `implicit → @coder` /
    `override`), the config path (`llm_provider.model_aliases.<source>.<name>`) when `config_source` is set, the bucket
    when set, and the pool availability when `selector_mode` is `round_robin`.
  - `kind` — keep passing the catalog `kind` through so the converter can distinguish models from aliases.
- Preserve the existing alias/value `filter_text` behavior so typing `def` still matches `@default`, and typing `@`
  still yields aliases only (already correct — add a regression test).

**File:** `crates/sase_xprompt_lsp/src/lsp_convert.rs`

- Add a `%model`-specific response builder alongside `vcs_project_completion_response`: map `kind` to a
  `CompletionItemKind` (`VALUE` for models, `ENUM_MEMBER` for aliases), set `label_details.description` to the alias
  kind badge (`default` / `role` / `coder` / `custom` / `model`), and set `sort_text` to `"<group>:<index:04>"` with
  group `0` for models and `1` for aliases so editors preserve the catalog's model-before-alias order instead of
  re-sorting alphabetically.
- Route `%model` through it from `completion_list_for_context`'s `DirectiveArgument` arm.

**Tests** (`cargo test -p sase_core -p sase_xprompt_lsp`, plus `cargo fmt` and `cargo clippy`):

- A catalog fixture with the new fields produces the expected `detail`, `documentation`, `label_details`, `sort_text`,
  and item kinds.
- A v1 fixture _without_ the new fields still produces items (no panic, no empty menu).
- `%model:@` returns alias items only; `%model:opus@` still returns the effort vocabulary.
- Keep the existing `crates/sase_core/src/editor/completion.rs` leading-`@` test; add a `%model:@` case if the current
  one only covers `%model:@oth`.

Follow the sase-core `AGENTS.md` release rules: do not hand-edit crate versions; use Conventional Commits so release-plz
derives them. The Python side needs no `sase-core-rs` window change because the catalog stays at `schema_version: 1` and
the new fields are additive in both directions.

---

## Visual snapshots, docs, and help text

### PNG snapshots

**New file:** `tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py`, following
`test_ace_png_snapshots_prompt_target_completion.py`: mount a `PromptInputBar`, call `show_file_completions` with
hand-built `CompletionCandidate` rows carrying `ModelCompletionMetadata`, and assert two goldens at 120x40:

- `prompt_model_completion_mixed_120x40` — models above aliases, one alias of each kind (`default`, `role`, `coder`,
  `custom`), one pooled alias, one overridden alias, with a model row highlighted so the `[@] model aliases` subtitle is
  visible.
- `prompt_model_completion_aliases_120x40` — the `@` alias-only menu with an alias highlighted so the description
  subtitle is visible.

Use fixed, fake provider/model values (as `_ace_models_panel_png_snapshot_fixtures.py` does) so the goldens do not
depend on installed provider CLIs. Generate the goldens with `--sase-update-visual-snapshots` and verify with a clean
`just test-visual` run.

### Docs

- `docs/xprompt.md` — extend the completion paragraph at line ~1198 ("ACE and the xprompt LSP complete `%model:` / `%m:`
  values…"): document that model aliases are listed beneath concrete model names with their resolved `PROVIDER(model)`,
  kind, and provenance; that typing `@` right after the colon narrows the menu to aliases; and that the LSP catalog is a
  launch-time snapshot that does not reflect temporary overrides.
- `docs/llms.md` — cross-link the alias completion menu from the model-alias section and point at the Models panel
  (leader `,m`) as the authoritative editor for alias targets and overrides.
- `CHANGELOG.md` is release-please-generated; do not hand-edit it.

### Help popup

`src/sase/ace/tui/modals/help_modal/binding_common.py:29` currently reads
`("%model: / %auto: / %effort:", "Auto-open directive values")`. Per `src/sase/ace/CLAUDE.md`, update the help popup for
the changed behavior — add or amend a line covering `%model:@` → model aliases only. Respect the 57-character box width
and the 32-character keybinding-description cap documented there.

### Verification

`just install` first (workspace virtualenvs are ephemeral), then `just check` and `just test-visual` from the sase
workspace, and `cargo fmt --check`, `cargo clippy`, `cargo test` from the sase-core checkout opened via `/sase_repo`.
