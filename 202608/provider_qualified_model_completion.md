---
tier: tale
title: Complete provider-qualified model names after a provider slash
goal:
  Typing a known provider name plus `/` in a `%model` value (for example `%m:claude/`)
  offers every model that provider owns, in ACE's prompt input and in editors through
  the xprompt LSP, and provider rows make that drill-down discoverable from the plain
  `%model` menu.
size: medium
proposed_by: bbugyi200.athena.012
create_time: 2026-08-14 10:20:37
status: wip
---

# Plan: Complete provider-qualified model names after a provider slash

## Goal

`%model` / `%m` already accepts explicit `provider/model` syntax (`docs/llms.md` §
"Explicit Provider/Model Syntax"), but completion knows nothing about it. After this
plan:

- `%m:claude/` lists `claude/opus`, `claude/sonnet`, `claude/haiku`,
  `claude/claude-haiku-4-5`, `claude/claude-fable-5`.
- `%m:codex/gpt-5` narrows to that provider's matching models.
- `%m:opencode/anthropic/` narrows inside a provider whose own model names contain
  slashes.
- A `claude/` "provider" row appears in the unqualified menu so the drill-down is
  discoverable, and accepting it reopens the menu scoped to that provider.
- The same behavior reaches Neovim and other editors through the Rust xprompt LSP.

## Background: how `%model` completion works today

Read these before editing; the whole feature lives in four seams.

**1. Catalog (Python, authoritative).** `src/sase/xprompt/model_completion.py` builds an
ordered list of `_ModelCompletionEntry` records from the cached LLM metadata payload:

- `_build_static_catalog()` walks providers in autodetect order, skipping any provider
  in `model_picker_hidden_provider_names()` (today: `fakey`), and appends one
  `kind="model"` entry per model where `model_to_provider[model] == provider`. Each
  entry carries `value`, `display`, `description` (provider display name, plus short
  alias and advisory suffixes), and `provider`.
- A second loop adds any `model_to_provider` entry the provider metadata missed.
- Implicit role aliases and user aliases follow, as `kind="implicit_alias"` /
  `kind="user_alias"` entries whose `value` starts with `@`.
- `build_model_completion_catalog()` caches the static list per config token and
  overlays temporary alias overrides.
- `filter_model_completion_entries(entries, partial)` is the only filter: a leading `@`
  gates to alias rows; otherwise an entry matches when its `value` or one of its short
  `aliases` prefix-matches the partial (case-insensitively).

**2. ACE prompt input (Python).** `src/sase/ace/tui/widgets/directive_completion.py:439`
(`_build_model_arg_completion_candidates`) filters the catalog and wraps each entry in a
`CompletionCandidate` carrying `ModelCompletionMetadata` (`directive_completion.py:54`).
Rendering is `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_directives.py`
(a four-column grid: name / kind / target / state), panel labels are
`src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel_labels.py`, and acceptance
is `_accept_file_completion()` in
`src/sase/ace/tui/widgets/_file_completion_accept.py:269`.

**3. Token extraction (Python).** `extract_directive_arg_token_around_cursor()` in
`src/sase/ace/tui/widgets/_directive_completion_tokens.py` already treats `/` as a
`%model` argument character (`_is_model_directive_argument_identifier`), so `%m:claude/`
is captured as a single argument token today — it simply matches nothing. A
**non-leading** `@` in the token already reroutes to effort completion, which is why
`%m:claude/opus@high` keeps working.

**4. LSP catalog + Rust filter.** `model_completion_catalog_payload()` is materialized
to JSON at LSP launch by `_materialize_model_catalog()` in
`src/sase/integrations/xprompt_lsp.py`, and the Rust server re-reads it on every
`%model` completion request. The Rust side lives in the linked `sase-core` repo (open it
with `sase repo open sase-core -r "<reason>"`; never clone or web-fetch it):

- `crates/sase_core/src/editor/completion.rs` → `directive_arg_token()` /
  `detect_directive_context_at_position()` produce the `%model` argument token (already
  slash-tolerant, already splitting a non-leading `@` off to effort).
- `crates/sase_xprompt_lsp/src/server.rs` → `model_completion_list()` is the Rust twin
  of `filter_model_completion_entries`, `model_entry()` parses one catalog entry,
  `load_model_catalog()` reads the JSON.
- `crates/sase_xprompt_lsp/src/lsp_convert.rs` → `model_completion_response()` and
  `model_completion_kind_label()` decorate the items.
- `/` is already an advertised LSP trigger character (`server.rs`,
  `trigger_characters`), so editors will re-request completion the moment the user types
  the slash. No capability change is needed.

Concretely, the current provider/model inventory is: `agy`, `claude`, `codex`, `fakey`
(hidden), `grok`, `muse`, `opencode`, `qwen`. Note that **`opencode` model names already
contain slashes** (`anthropic/claude-sonnet-4-5`, `openai/gpt-5`, …), so `%m:anthropic/`
already prefix-matches those bare model values today. That behavior must not regress.

## Design decisions

Settle these before writing code; the rest of the plan assumes them.

### D1. Derive qualified rows at filter time; ship provider rows as data

Do **not** put a `provider/model` entry for every model into the catalog. That would
double the catalog and flood the unqualified menu (`%m:op` would list both `opus` and
`opencode/openai/gpt-5`).

Instead:

- **Catalog gains one `kind="provider"` entry per visible provider**, with `value` and
  `display` equal to `"<provider>/"` (trailing slash), `provider="<provider>"`, and
  `description` set to the provider's display name (`Claude`, `Antigravity`, …). Emit
  these only for providers that are not hidden and that contributed at least one
  completable model entry.
- **Qualified rows are derived inside the filter** from the existing `kind="model"`
  entries, which already carry `provider`. The derived row is the source entry with
  `value`/`display` prefixed by `"<provider>/"`; `description`, short `aliases`, and
  advisory fields are carried over unchanged.

### D2. The filter rule, implemented identically in Python and Rust

`filter_model_completion_entries` (Python) and `model_completion_list` (Rust) must agree
on this precedence:

1. **Alias gate** — partial starts with `@`: unchanged. Alias rows only; provider rows
   are excluded (they are neither `implicit_alias` nor `user_alias`, so today's
   predicate already excludes them — assert it in a test).
2. **Provider scope** — the partial splits on its **first** `/` into `<head>/<rest>` and
   `<head>` (lowercased) names a provider that has a `kind="provider"` catalog entry:
   return **only** derived qualified rows for that provider's models, in catalog order,
   keeping a model whose name **or** short alias prefix-matches `<rest>`. Provider rows
   and alias rows are excluded. `<rest>` may itself contain slashes
   (`opencode/anthropic/`).
3. **Fallback** — anything else: today's plain prefix match over `value` and `aliases`,
   so `%m:anthropic/` still finds opencode's slash-bearing model names and `%m:cl/` (no
   such provider) still yields nothing.

Rule 2 keys off the presence of a `kind="provider"` entry rather than a separate
provider list, which keeps the wire schema purely additive.

### D3. Version skew is safe in both directions — no schema bump

Keep `MODEL_COMPLETION_CATALOG_SCHEMA_VERSION` at `1`; the change is additive, exactly
as the module docstring already anticipates for alias metadata.

- **New Python catalog + old LSP binary**: the old binary prefix-matches
  `kind="provider"` entries like any other row, so `claude/` still shows and still
  inserts. Drilling in yields nothing — degraded, not broken.
- **New LSP binary + old catalog (no provider entries)**: rule 2 never fires, so
  behavior is exactly today's.

Therefore **the two repos can land in either order**, and neither needs to wait on a
`sase-core` release. Do not add a `sase-core-rs` binding for this; the `sase-core-rs`
pin in `pyproject.toml` would force the Python side to depend on a core release and to
carry a fallback path, for a filter over ~50 in-memory rows.

> Note on the Rust core backend boundary: `%model` completion filtering is arguably
> shared backend behavior, and consolidating both copies into `sase_core` (exposed to
> Python the way `placeholder_completion` already is) would be the boundary-correct end
> state. That consolidation is deliberately **out of scope** here — it is a refactor of
> pre-existing duplication (the `@` alias gate is already implemented twice), it is
> orthogonal to this feature, and it would couple this change to a core release. Propose
> it as a follow-up task bead via `/sase_new_task` after this lands.

### D4. Provider rows sort last

In the unqualified menu, provider rows render as a trailing group, **after** concrete
models and after alias rows. Rationale: many provider names share a first letter with a
common model (`o` → `opus` _and_ `opencode/`; `c` → `claude-fable-5` _and_ `claude/`,
`codex/`), and a provider row sorted first would steal the default selection — and
therefore <kbd>Enter</kbd> — from the model the user was actually typing. Sorting them
last keeps every existing keystroke landing exactly where it does today while still
exposing the drill-down.

### D5. Hidden providers stay hidden

`fakey` is excluded from the catalog today and must get neither a provider row nor a
provider scope: `%m:fakey/` falls through to rule 3 and returns nothing. Typing
`%model:fakey/fakey-large` by hand still resolves — completion coverage and routing are
independent, as `docs/llms.md` already states.

### D6. Accepting a provider row drills in

`claude/` is a prefix, not a value. Accepting it must insert `claude/` and reopen the
menu, mirroring the existing directory drill-down at `_file_completion_accept.py:380`
(`if selected.is_dir and self._completion_kind in ("file", "xprompt_arg_path")`). Mark
provider candidates with `is_dir=True` and extend that branch to cover `directive_arg`,
reopening via `_try_auto_directive_arg_completion()`. Do the same for the lone-candidate
auto-accept in `_file_completion_tab.py:159` so <kbd>Ctrl+T</kbd> on a unique provider
match drills in instead of dead-ending.

### D7. Do not disturb the existing row grid

`_MODEL_KIND_CELL` is 7 cells and every existing kind label fits (`model`, `default`,
`role`, `coder`, `custom`). The provider row must reuse that width — pick a label that
fits in 7 (`models` is the suggested choice) rather than widening the column, which
would shift every existing row and every pinned PNG snapshot.

## Implementation

Work the Python side first; it is self-contained and independently testable.

### Step 1 — catalog: emit provider entries

`src/sase/xprompt/model_completion.py`

1. Track, while `_build_static_catalog()` appends model entries, which providers
   actually contributed at least one entry, along with the display name resolved by
   `_provider_display()`. Keep provider order stable (autodetect order, then sorted
   remainder), matching `_provider_order()`.
2. After the alias entries, append one `kind="provider"` entry per contributing
   provider: `value=display=f"{provider}/"`, `description=<provider display name>`,
   `provider=<provider>`. Appending last realizes D4 without a separate sort. Guard with
   `_is_inline_completable()` for consistency (`claude/` passes the existing regex).
3. Consider carrying the model count on the entry (a new `provider_model_count: int = 0`
   field) so the ACE row can render "5 models" without recounting. If you add it, add it
   to `model_completion_catalog_payload()` too; Rust ignores unknown keys.
4. `_apply_alias_overrides()` needs no change — it indexes only alias kinds — but add a
   regression test proving a provider row survives an active override untouched.

### Step 2 — catalog: the provider-scoped filter

`src/sase/xprompt/model_completion.py`

Implement D2 in `filter_model_completion_entries()`. Suggested shape: a private
`_provider_scope(entries, needle)` returning `tuple[str, str] | None` (provider,
remainder) by splitting the needle on its first `/` and confirming a `kind="provider"`
entry named `f"{head}/"` exists, plus a private `_qualified_entry(entry, provider)`
returning the prefixed `replace(...)` copy. Keep the function pure and total — it is
called on every keystroke and is covered by the existing cache-free tests.

Ordering inside a scope is catalog order, which is the provider's declared
`known_model_names` order — the same order the unqualified menu already uses.

### Step 3 — ACE candidates

`src/sase/ace/tui/widgets/directive_completion.py`

1. Add the provider fields to `ModelCompletionMetadata` (at minimum the
   `kind="provider"` value flows through `kind`; add `provider_model_count` if Step 1.3
   added it).
2. In `_build_model_arg_completion_candidates()`, set `is_dir=True` on candidates whose
   entry kind is `"provider"` (D6). Everything else already flows through the generic
   mapping.
3. Check the `shared_extension` computation still behaves: for `%m:claude/` every
   insertion starts with `claude/`, so `os.path.commonprefix` yields no extension past
   the partial, and Tab opens the menu instead of inserting. Add a test pinning that.
4. `_model_provider_display()` special-cases `kind != "model"` by returning `""`; make
   sure a provider row still reaches the renderer with the display name it needs (either
   relax that helper for `"provider"` or read `description` in the renderer).

### Step 4 — ACE rendering and labels

1. `_prompt_input_bar_completion_rows_directives.py`: add an explicit
   `kind == "provider"` branch in `append_model_completion_row()` before the alias
   branch — otherwise `_model_completion_is_degraded_alias()` catches it and renders a
   bare two-column fallback. Style the name with
   `provider_name_style(metadata.provider)`, use a ≤7-cell kind label (D7), and put the
   provider display name (optionally with the model count) in the target cell. Include
   provider rows in `model_completion_column_widths()` so the grid stays aligned.
2. `_prompt_input_bar_completion_panel_labels.py`:
   - `completion_panel_title()` — when the token is provider-scoped, title the panel for
     that provider (for example `claude/ models`) instead of `%model values`. The token
     is already a parameter.
   - `model_completion_subtitle()` — give the provider row a useful subtitle (for
     example `[⏎] show <provider> models`) rather than falling through to the empty
     string.

### Step 5 — ACE acceptance drill-down

`src/sase/ace/tui/widgets/_file_completion_accept.py` and
`src/sase/ace/tui/widgets/_file_completion_tab.py`: implement D6. Verify the reopened
menu is scoped (the token is now `claude/`, so rule 2 fires) and that
`_clear_file_completion()` is still reached when the reopen finds nothing.

### Step 6 — Rust LSP parity

In the `sase-core` checkout obtained via `sase repo open sase-core`:

1. `crates/sase_xprompt_lsp/src/server.rs` — implement D2 in `model_completion_list()`.
   The parsed `ModelCompletionEntry` already carries `kind` and `provider`, so no
   `model_entry()` change is required unless you added `provider_model_count` and want
   to surface it.
2. `crates/sase_xprompt_lsp/src/lsp_convert.rs` — teach `model_completion_kind_label()`
   to return `"provider"` for the new kind so the item's `label_details.description` is
   not mislabeled `"model"`. Also fix the `sort_text` grouping in
   `model_completion_response()`: it is currently `group = u8::from(is_alias)`, so
   provider rows (not aliases) would land in group `0` alongside the models and sort
   _before_ the aliases, diverging from ACE's D4 order. Give provider rows their own
   group `2` so the LSP order is models → aliases → providers, matching ACE. Pin it with
   a test.
3. No change to `crates/sase_core/src/editor/completion.rs` is expected: the argument
   token already tolerates `/` and already splits a non-leading `@` to effort. Confirm
   with a test rather than assuming.
4. Run the crate's own gates (`just` recipes in the `sase-core` checkout; `cargo fmt` +
   `cargo clippy` + `cargo test` at minimum).

### Step 7 — docs

- `docs/llms.md` — under "Explicit Provider/Model Syntax", document that the completion
  menu offers provider rows and expands `provider/` into that provider's models; extend
  the completion-menu paragraph near the alias vocabulary section
  (`The same alias vocabulary appears in the %model: / %m: completion menu…`) to mention
  the provider scope; keep the existing note that hidden providers (`fakey`) stay out of
  the menu while remaining routable.
- `docs/xprompt.md` — update the `%model` completion row anatomy that `docs/llms.md`
  links to (the "ACE and the xprompt LSP complete `%model:` / `%m:` values…" paragraph
  and the alias-row paragraph beneath it), adding the provider row and the
  provider-scoped drill-down.
- `docs/editor.md` — the "Directive completion" row of the LSP feature table should
  mention provider-scoped model values.
- Do not hand-edit `CHANGELOG.md`; release-please owns it.

## Tests

Match each file's existing conventions.

**`tests/test_xprompt_model_completion.py`**

- The catalog contains one `kind="provider"` entry per visible provider, with the
  trailing slash, ordered after models and aliases.
- Hidden providers get no provider entry (assert `fakey/` is absent), and `%m:fakey/`
  filters to nothing.
- `filter_model_completion_entries(catalog, "claude/")` returns exactly claude's models
  as `claude/<model>`, in the provider's declared order, and no alias or provider rows.
- `"claude/op"` narrows to `claude/opus`; a short-alias prefix inside a scope matches
  too.
- `"opencode/anthropic/"` returns opencode's `anthropic/*` models as
  `opencode/anthropic/*` — the first-slash split, not a greedy one.
- **Regression:** `"anthropic/"` (not a provider) still returns opencode's bare
  `anthropic/*` model values via the fallback rule.
- `"@"` still returns alias rows only, with no provider rows.
- Uppercase input (`"Claude/"`) resolves the scope and yields canonical lowercase
  insertions.
- `model_completion_catalog_payload()` round-trips provider entries.
- An active temporary alias override leaves provider rows untouched.

**`tests/ace/tui/widgets/test_directive_arg_completion.py`**

- `%m:claude/` and `%model:claude/` produce qualified candidates; the paren forms
  `%model(claude/…)` and `%model(opus, alias=claude/…)` do too.
- Provider candidates carry `is_dir=True`.
- `%m:claude/opus@` still routes to effort completion.
- No shared extension is offered for a provider-scoped partial.

**`tests/ace/tui/widgets/test_model_completion_rows.py`** — provider row renders in the
four-column grid with the provider color and a ≤7-cell kind label;
`model_completion_column_widths()` accounts for it.

**`tests/ace/tui/test_model_completion_panel_titles.py`** — provider-scoped title and
the provider row's subtitle.

**ACE acceptance** — a widget-level test that accepting a provider row inserts `claude/`
and leaves the completion menu open and scoped (extend the existing prompt-input
completion tests rather than creating a new module if one fits).

**Visual** — the existing `prompt_model_completion_*` PNG snapshots build synthetic
candidate lists, so they should not shift. Adding one new snapshot for a provider-scoped
menu is worthwhile; regenerate with `just test-visual --sase-update-visual-snapshots`
and inspect `.pytest_cache/sase-visual/` artifacts.

**Rust (`sase-core`)** — extend the `model_completion_list` tests in
`crates/sase_xprompt_lsp/src/server.rs`: a catalog fixture with provider entries;
`%model:claude/` returns qualified items; `%model:anthropic/` keeps the fallback;
`%model:@` excludes provider rows; an old-shaped catalog with no provider entries
behaves exactly as before (the D3 skew guarantee); and a `lsp_convert.rs` test for the
`"provider"` kind label.

## Verification

1. `just install` first — workspace virtualenvs go stale.
2. `just check` while iterating.
3. `just check-full` before landing, and because it routinely outruns a single turn, run
   it through `/sase_monitor`
   (`sase monitor start --command 'just check-full' … --next …`), never inline.
4. `just test-visual` if you touched or added a PNG snapshot.
5. In the `sase-core` checkout, run that repo's Rust gates.
6. Manual smoke in ACE: open the prompt input, type `%m:`, confirm provider rows sit at
   the bottom and the existing menu is otherwise unchanged; type `claude/` and confirm
   the scoped list; accept a provider row and confirm the menu reopens scoped; confirm
   `%m:opus` still lands on `opus`.

## Out of scope

- Consolidating the two filter implementations into `sase_core` (see D3); propose it as
  a task bead via `/sase_new_task` once this lands.
- Provider-name completion anywhere other than `%model` values — the ACE Models panel /
  model picker, and any CLI `--model` shell completion, are untouched.
- Fuzzy or substring matching. This stays prefix-matching, consistent with the rest of
  `%model` completion.
- Any change to `%model` resolution, routing, or the `provider/model` grammar itself.
  This plan only teaches completion about grammar that already works.
