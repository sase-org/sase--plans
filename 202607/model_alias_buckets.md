---
create_time: 2026-07-11 09:11:12
status: done
prompt: 202607/prompts/model_alias_buckets.md
tier: tale
---
# Plan: Model Alias Buckets in the TUI Models Panel (`,m`)

## Goal

Add **model alias buckets** to the ace TUI **Models** panel (opened with `,m`): a way to group a set of related custom
model aliases under one named, single-line **bucket row**. Selecting a bucket row and pressing `l` drills _into_ the
bucket to see its member aliases; `h` drills back _out_ to the main list.

First concrete use-case: fold the `research` custom aliases into a `research` bucket, renaming `research → research_a`,
`research_assist → research_b`, and adding a new `research_c → codex/gpt-5.6-sol`.

The feature must be **intuitive** (mirrors the Agents-tab `h`/`l` fold gesture users already know), **reliable**
(buckets are a _display-only_ grouping — they never change how `@<alias>` resolves), and **beautiful** (a clean
folder-style row that harmonizes with the existing alias-row columns, with rich detail in the reused description strip).

## Guiding design principles

1. **Buckets are presentation, not semantics.** A bucket is purely an organizational grouping in the Models panel. Alias
   _names stay globally unique and flat_, and `%model:@research_b` / `%m:@research_a` resolve exactly as before
   regardless of bucket membership. This keeps resolution, `%model` completion, delegation, and every launch path
   untouched — the safest possible foundation.
2. **Stay on the Python/TUI side of the Rust boundary.** Model aliases are a Python-only concept today (the Rust core
   has no model-alias structs; the only Python→Rust handoff is the `%model` completion catalog, which is unaffected
   because aliases still complete individually). No `sase-core` changes are required.
3. **Additive and backward-compatible config.** A bucket is an optional `bucket:` tag on an existing custom alias
   object. Aliases without a `bucket` render flat exactly as today. No migration is forced on anyone.
4. **Reuse the existing panel machinery.** Keep the single `OptionList` + description strip + footer. Model drill-in/out
   as a "swap the list contents" state change (`_active_bucket`), not a rewrite to a Tree widget. Every existing
   per-alias action (override `o`, clear `x`, edit `e`, reset `r`) keeps working verbatim on the member rows inside a
   bucket.

## Config model & schema

Aliases live under `llm_provider.model_aliases.{builtin,custom}`. Custom aliases are objects requiring `model` and
`description`. We make two additive changes:

**(a) Optional `bucket` tag on a custom alias** — the single source of truth for membership:

```yaml
llm_provider:
  model_aliases:
    custom:
      research_a:
        model: codex/gpt-5.6-sol
        description: "Lead researcher + consolidator."
        bucket: research # <-- new, optional
      research_b:
        model: claude/opus
        description: "Second-opinion researcher."
        bucket: research
      research_c:
        model: codex/gpt-5.6-sol
        description: "Extra researcher lane."
        bucket: research
```

**(b) Optional `buckets` metadata map** (sibling of `builtin`/`custom`) for a human bucket description shown on the
bucket row's description strip:

```yaml
buckets:
  research:
    description: "Research-swarm model roles (lead, second-opinion, extra)."
```

Membership is defined _only_ by the aliases' `bucket:` fields. The `buckets:` map is optional metadata: a bucket with
member aliases but no `buckets` entry still renders (with a derived summary and a gentle "no description" hint); a
`buckets` entry with zero members simply doesn't render (and can be surfaced by doctor as a dangling entry). This
decoupling keeps the minimal path minimal and the feature robust.

Only **user/custom** aliases can carry a bucket; builtin/role/`<provider>_coder` implicit aliases are never bucketed.

### Schema (`src/sase/config/sase.schema.json`)

- Add an optional `bucket` property to the custom-alias object (keep `additionalProperties: false`,
  `required: ["model","description"]` unchanged): `{"type":"string","minLength":1,"maxLength":40}` with a description
  noting it only affects the `,m` panel display.
- Add an optional `buckets` map under `model_aliases` (which currently has `additionalProperties: false`, so it must be
  declared): keyed by bucket name, each value an object with an optional `description` (`minLength:1, maxLength:160`,
  `additionalProperties:false`).

### Ordering / sequencing note

Because the schema currently rejects unknown keys (`additionalProperties: false`), the sase-repo schema+code changes
must be **installed before** the chezmoi config gains `bucket:`/`buckets:` (otherwise `sase doctor` and config-edit
validation would flag them). Runtime alias _resolution_ never breaks — unknown keys are ignored by the defensive parser
— so this is a doctor/validation-cleanliness ordering, not a launch-safety one. Land the sase feature first, then apply
the chezmoi migration.

## Data layer (Textual-free, unit-testable)

**`src/sase/llm_provider/config.py`** — add small parsing helpers mirroring the existing
`_custom_model_alias_descriptions()`:

- `_custom_model_alias_buckets() -> dict[str, str]` — alias → bucket name (parse `.bucket`, strip, skip blanks;
  defensive like the model/description parsers).
- `model_alias_bucket(name) -> str | None` — public accessor (meaningful only for custom aliases).
- `model_alias_bucket_description(bucket) -> str | None` — read `model_aliases.buckets.<bucket>.description`.
- `model_alias_bucket_names() -> set[str]` — buckets referenced by any custom alias.

**`src/sase/llm_provider/alias_view.py`** — the panel's data source:

- Add `bucket: str | None = None` to the `AliasView` dataclass; populate it in `build_alias_views()` via
  `model_alias_bucket(name)`. `build_alias_views()` still returns the full flat list (nothing else that consumes it
  changes).
- Add a `BucketView` dataclass: `name`, `description: str | None`, `members: tuple[AliasView, ...]`, plus derived
  helpers — `alias_count`, `override_count` (members with an active temporary override), and a concise `model_summary`
  (dominant member model + `+N` for the count of _other distinct_ models).
- Add `build_models_panel_rows(views=None) -> list[AliasView | BucketView]` that produces the **top-level** display
  order: keep the existing sort for non-user rows (default → role → `<provider>_coder`), then within the user region
  emit **buckets first (alpha), then ungrouped user aliases (alpha)** — a clean "folders then files" layout. Bucketed
  user aliases are folded into their `BucketView` (and thus hidden from the top level).

This keeps all grouping logic pure and covered by `tests/llm_provider/test_alias_view.py`.

## Rendering (`src/sase/ace/tui/modals/models_panel_rendering.py`)

Reuse the existing column skeleton (`<kind cell 13> <name cell 16> <provider/model ≤20> <gap> <state tag>`) so bucket
rows align beautifully with alias rows.

- `render_bucket_row(bucket, *, provider_model_width) -> Text`:
  - **kind cell**: `▸ bucket` — a disclosure triangle + `bucket` label, in a new distinct hue (folder gold, e.g.
    `bold #FFD787`) so buckets pop out from the four kind colors already in use.
  - **name cell**: the bucket name, bold.
  - **provider/model column**: `{n} aliases` (dim), padded to the shared width so the state tag stays aligned.
  - **state tag**: `override · {k} active` (violet) when any member is overridden, else `bucket` (dim gold). This
    surfaces live in-bucket override state on the collapsed row — useful and honest.
- `description_text_for_bucket(bucket) -> Text` (2 lines, fits the fixed 4-row strip): line 1 = the bucket description
  (or a muted "no description — set `llm_provider.model_aliases.buckets.<name>.description`" hint, matching the existing
  missing-description style); line 2 = a compact model breakdown, e.g. `codex/gpt-5.6-sol ×2 · claude/opus ×1`.
- Add a small `description_text_for_row(row)` dispatcher over `AliasView | BucketView` (existing
  `description_text_for_view` stays for alias rows).

The `▸` disclosure + description-strip reuse is the heart of the "beautiful": the collapsed row is a crisp single line,
and the rich detail (human description + model mix) appears in the strip on highlight.

## Panel behavior & navigation (`src/sase/ace/tui/modals/models_panel.py`)

Introduce `self._active_bucket: str | None = None` (top level vs. drilled-in) and a `self._row_by_id` map so a
highlighted option resolves to either an `AliasView` or a `BucketView`.

- **Row ids**: alias rows keep `id=<alias name>`; bucket rows use `id="bucket:<name>"` (namespaced so it can never
  collide with an alias name).
- **`_build_options()`**:
  - Top level (`_active_bucket is None`): `build_models_panel_rows()` → render alias rows with `render_alias_row` and
    bucket rows with `render_bucket_row`.
  - Drilled-in: render only that bucket's member `AliasView`s as ordinary alias rows (so every per-alias action and
    column looks/behaves identically to the flat panel).
  - Provider/model column width is computed from the alias rows currently displayed.
- **Drill-in / drill-out**:
  - Add bindings `l` / `right` → `action_enter_bucket`, `h` / `left` → `action_leave_bucket`.
  - `action_enter_bucket`: only at top level and only when the highlighted row is a `BucketView` → set `_active_bucket`,
    rebuild, highlight the first member, update the title breadcrumb + footer. Otherwise a no-op (e.g. `l` on a plain
    alias does nothing).
  - `action_leave_bucket`: when drilled-in → clear `_active_bucket`, rebuild, re-highlight the `bucket:<name>` row we
    came from (cursor-snap so `h` never lands on a vanished row). At top level, no-op.
  - `enter` (`on_option_list_option_selected`): if a `BucketView` is highlighted → drill in; else keep the existing
    behavior (open the override picker).
- **Guard existing actions**: `action_override` / `action_clear` / `action_edit` / `action_reset` operate on an alias.
  When a `BucketView` is highlighted (only possible at top level), show a gentle notify ("Press `l`/`enter` to open this
  bucket") and return. Inside a bucket every row is an alias, so all four work unchanged.
- **Title breadcrumb** (`#models-panel-title`): `Models` at top level; `Models › research` when drilled in (dim `›` +
  gold bucket name).
- **Context-aware footer** (`_footer_markup`), updated on highlight and on drill:
  - Bucket row highlighted (top level): `l/enter=Open · j/k=Navigate · esc=Close`.
  - Alias row at top level: the existing `o/x/e/r` footer.
  - Inside a bucket: the existing `o/x/e/r` footer plus `h=Back`.
- **Refresh reliability**: `_refresh_rows` preserves the highlighted row across rebuilds for both alias and bucket ids.
  If an in-bucket `r` (reset) deletes the last member, auto-leave to the top level so the panel never sits on an empty
  bucket. A persistent `e` edit keeps the `bucket:` sibling (the edit targets the `.model` subkey), so a member never
  silently leaves its bucket.

No `default_config.yml` app-keymap change is needed: the panel's `h`/`l` are modal-local `BINDINGS` (like the existing
`o`/`x`/`e`/`r`), not global ace keymaps.

## Documentation

- Extend the commented `model_aliases` example in `src/sase/default_config.yml` to show a `bucket:` field and a
  `buckets:` block.
- If the `?` help modal or any ace help surface documents Models-panel internals, add the `l`/`h` bucket drill keys
  (verify per the ace CLAUDE.md help-sync rule; the panel's own footer is the primary doc surface).
- Optional doctor enhancement (`src/sase/doctor/checks_config_model_aliases.py`): warn on a `buckets:` metadata entry
  that has **no member aliases** (dangling bucket description), mirroring the existing dangling-`@alias` check.
  Low-risk, improves reliability signal.

## Testing

- **`tests/llm_provider/test_alias_view.py`**: `AliasView.bucket` population; `BucketView` derived fields
  (`alias_count`, `override_count`, `model_summary`); `build_models_panel_rows` grouping + "folders-then-files"
  ordering; ungrouped aliases untouched; a `buckets` entry with no members renders nothing.
- **Config parsing tests** (wherever `get_custom_model_aliases` / descriptions are tested): `bucket` parsing,
  blank/missing handling, and `model_alias_bucket_description`.
- **Schema test**: a fixture with `bucket:` + a `buckets:` block validates; an unknown extra key still fails.
- **`tests/test_models_panel.py` / `tests/test_models_panel_keymaps.py`**: `l`/`enter` drills into a bucket; `h` drills
  back and re-highlights the source bucket; `o/x/e/r` on a bucket row is a guarded no-op with a notify; title
  breadcrumb + footer update per context; auto-leave when a bucket empties; description strip shows the bucket
  description + model breakdown.
- **PNG visual snapshots** (`tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py`, goldens under
  `tests/ace/tui/visual/snapshots/png/`): add a top-level snapshot showing a bucket row among alias rows, and a
  drilled-in snapshot showing the breadcrumb + member rows. This locks in the "beautiful" bar and follows the repo's
  visual-snapshot discipline.
- Run `just check` (after `just install`) plus `just test-visual`.

## First use-case migration (chezmoi repo)

These edits live in Bryan's chezmoi repo (his global SASE config source); access it via the standard linked-repo
mechanism (`sase workspace open -p chezmoi …`). Apply **after** the sase feature above is installed.

**`home/dot_config/sase/sase.yml`** — under `llm_provider.model_aliases`:

- Rename `custom.research` → `custom.research_a` (keep `model: codex/gpt-5.6-sol` and its description), add
  `bucket: research`.
- Rename `custom.research_assist` → `custom.research_b` (keep `model: claude/opus` and its description), add
  `bucket: research`.
- Add `custom.research_c` → `model: codex/gpt-5.6-sol` with a short description and `bucket: research`.
- Add a `buckets.research.description` (e.g. "Research-swarm model roles: lead, second-opinion, extra").

**`home/dot_xprompts/research_swarm.md`** — update the three alias references to the renamed aliases:

- `%model:@research` → `%model:@research_a` (segment 1, the lead researcher `research.@.cdx`).
- `%m:@research_assist` → `%m:@research_b` (segment 2, the second researcher `research.@.cld`).
- `%m:@research` → `%m:@research_a` (segment 3, the consolidator `research.@.final`).
- Segment 4 (`research.@.image`) keeps its hardcoded `%model:codex/gpt-5.6-sol`.

`research_c` is _defined and bucketed_ per the request but not wired into the swarm (the swarm stays two researchers +
consolidator); it's available for manual use / future swarm expansion.

**Apply & verify**: keep the chezmoi keep-sorted blocks valid, run `chezmoi apply`, then confirm `sase doctor` passes,
`%model:@research_a/@research_b/@research_c` all resolve, and the `,m` panel shows a `research` bucket row that drills
into the three aliases with `l`/`h`.

## Explicit non-goals (v1)

- **Bucket-wide override/edit** (applying `o`/`e` to every member at once). Deferred to keep v1 reliable (no
  partial-failure / mixed-state semantics); the collapsed row already _surfaces_ member override state.
- **Buckets on builtin/role aliases**, **nested buckets**, and any change to alias **resolution/completion**.
- **Rust core changes** — none required.

## Rollout summary / ordering

1. Land the sase-repo changes: schema → `config.py` helpers → `alias_view.py` (`bucket`, `BucketView`,
   `build_models_panel_rows`) → rendering → panel navigation → docs → tests (incl. visual snapshots). Run `just check` +
   `just test-visual`.
2. Apply the chezmoi migration (rename to `research_a`/`research_b`, add `research_c`, add the `research` bucket +
   description, update `research_swarm.md`), then `chezmoi apply` and verify end-to-end in `,m`.
