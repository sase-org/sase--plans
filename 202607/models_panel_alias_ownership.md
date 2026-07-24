---
tier: tale
title: Separate user-defined and built-in aliases in the ACE Models panel
goal: The ACE Models panel makes alias ownership unmistakable at a glance by splitting
  rows into labelled "Built-in" and "Yours" sections and marking every user-defined
  alias, bucket, and bucket member with a persistent ownership gutter, without reordering
  rows, changing alias resolution, or adding any keybinding.
create_time: 2026-07-24 14:39:07
status: done
---

- **PROMPT:** [202607/prompts/models_panel_alias_ownership.md](prompts/models_panel_alias_ownership.md)

# Plan: Separate user-defined and built-in aliases in the ACE Models panel

## Context and intent

The Models panel (`,m`) lists every model alias in one undifferentiated column stack. The only signal that an alias was
invented by the user rather than shipped by SASE is the small `user` word in the 13-cell kind badge and its tan color —
easy to miss in a dense list, and completely absent for bucket rows, where a user-defined bucket such as `researchers`
renders identically to the built-in `coders` bucket. The distinction matters because the two kinds of row have different
semantics: a built-in alias name is part of SASE's contract and only its _value_ is user-owned, while a user alias
exists only because the user created it.

Make ownership legible through three reinforcing layers, in this priority order:

1. **Structure** — labelled section headers split the list, so ownership is obvious when reading top to bottom.
2. **Locality** — a per-row ownership gutter keeps the signal attached to the row, so it survives scrolling past a
   header, drilling into a bucket, and any future filtering.
3. **Wording** — explicit origin language on bucket rows and the drilled-in title, so nothing depends on color alone.

Deliberately keep the change presentation-only. Do not alter alias resolution, merge precedence, the config schema,
`default_config.yml`, keymaps, or any existing binding; no new keys are introduced and no existing key changes meaning.
Do not reorder rows: user-owned rows already sort last at the top level, so the section boundary falls on an existing
seam and all muscle memory is preserved. Every classification must be derived from the in-memory `AliasView` /
`BucketView` snapshot the panel already builds; render and keystroke paths must not read config, stat, or glob (see
`sase/memory/tui_perf.md`).

Filtering, sorting, or a "show only my aliases" toggle are explicitly out of scope.

## Ownership classification

Ownership is a single, centrally defined predicate, not an ad-hoc test repeated per call site. Extend the display-ready
model in `sase.llm_provider.alias_view`:

- An alias row is user-owned exactly when its `kind` is `user`. Nothing else counts.
- A bucket row is user-owned exactly when its name is not one of `BUILTIN_MODEL_ALIAS_BUCKET_NAMES`. By construction
  only user aliases are placed into non-built-in buckets, so a user bucket is homogeneous.
- Expose the number of user-owned members a bucket contains, so a built-in bucket can advertise the user aliases folded
  into it without being drilled into.

One rule needs to be stated explicitly because the obvious shortcut is wrong: a **built-in alias mistakenly configured
under `llm_provider.model_aliases.custom`** (today's `is_custom_builtin_shadow`) is _not_ user-owned. It keeps its
built-in classification, stays in the built-in section, keeps its existing gold `!` glyph and warning description, and
never receives the ownership gutter. Ownership is `kind == "user"`; `configured_source == "custom"` is a config-location
fact and must not be used as the ownership test.

Add a pure, Textual-free function to the same module that partitions an ordered Models-panel row list into an ordered
pair of sections (built-in first, then user), each carrying its rows plus the alias and bucket counts needed by its
header. Provide the same split for the members of an open bucket. These functions must be order-preserving: partitioning
the current row list and re-concatenating the sections must reproduce the original list exactly, both at the top level
and inside every bucket. Keeping the partition in the data layer keeps it unit-testable without a mounted app and
reusable by a future CLI or web surface, matching the module's stated purpose.

## Section headers and the row grid

Render section headers as non-selectable `disabled=True` options, matching the established house convention already used
by the xprompt browser, plugins browser, xprompt-location, and agent-run-log modals. Use a reserved id prefix that
cannot be confused with an alias name or the existing `bucket:` id prefix.

The headers must land on the panel's existing column grid rather than floating free. Extend the current
`<kind> <name> <provider/model> <state>` skeleton in `models_panel_rendering.py` with a two-cell leading gutter, and
render a header as a labelled rule that fills the kind, name, and provider/model columns and places its counts in the
state column — so the section counts align vertically with the `configured` / `implicit` / `bucket` state tags below
them. The rule must be computed from the same dynamic provider/model column width the alias rows use, so it can never
drift out of alignment when a wide model badge widens that column.

The intended top level reads:

```
   ── Built-in ─────────────────────────────────────────────  11 aliases · 2 buckets
   default       default         CODEX(gpt-5.6-sol)              implicit
   ▸ bucket      coders          7 aliases                       bucket · 2 yours
   role          epic_creator    CODEX(gpt-5.6-sol)              configured
   ...
▌  ── Yours ────────────────────────────────────────────────  4 aliases · 1 bucket
▌  ▸ bucket      researchers     3 aliases                       bucket
▌  user          phase_worker    CODEX(gpt-5.6-sol)              configured
```

Header counts describe the rows in that section, counting bucket members rather than collapsed bucket rows, and
singularize correctly (`1 alias`, `1 bucket`); omit the bucket clause entirely when a section has none. User aliases
folded into a built-in bucket are counted in the built-in section, because that is where they are displayed — their
ownership is surfaced by the bucket row's chip instead, so no number ever contradicts what the list shows.

When the user has no custom aliases and no custom buckets at all, still render the `Yours` header followed by one dim,
non-selectable hint row that names `llm_provider.model_aliases.custom` as where custom aliases are declared. Keeping the
section constant makes the panel's shape learnable and turns "I have none" into an explicit answer rather than an
ambiguous absence.

## Ownership gutter and origin chips

Every alias and bucket row gains a two-cell leading gutter: a solid left half-block glyph (`▌`, U+258C) in the ownership
accent for a user-owned row, and two blank cells otherwise. Because ownership is carried by a glyph and not only by
color, the signal survives color-blind vision, terminal theme changes, and monochrome captures. The `Yours` section
header carries the same glyph in the same accent, so the header teaches the glyph at the exact point of first use. Reuse
the existing `user` kind color as a single named ownership-accent constant shared by the gutter, the `Yours` header,
user bucket rows, and the chip below — one accent, used consistently, is what makes the result read as designed rather
than decorated.

A built-in bucket that contains user-owned members appends a `· <n> yours` chip in the ownership accent to its state
tag. The chip is always appended last and never replaces an existing chip, so the misplaced-builtin warning and the
active-override count keep their precedence; the row's existing right-edge ellipsis then drops the least important chip
first if a rare three-chip combination exceeds the width budget. A user-owned bucket keeps its current `bucket` state
tag but renders it in the ownership accent instead of the dim gold used for built-in buckets.

The kind badge text itself is unchanged. Ownership is now carried by structure, gutter, and chips, so widening or
rewording that column would only add redundancy.

## Bucket navigation and titles

Drilling into a bucket must answer the ownership question without scrolling back out. The panel title already renders
`Models › <bucket>`; extend it so a user-owned bucket shows the ownership glyph before the name, renders the name in the
ownership accent, and carries a dim `· your bucket` suffix, while a built-in bucket keeps its gold name and carries a
dim `· built-in bucket` suffix.

Inside an open bucket, emit the same two section headers **only when that bucket is mixed** — that is, when it holds
both built-in and user-owned members, which today can happen only for `coders` and `phase_worker`. A homogeneous bucket
gets no headers, because there is nothing to disambiguate and the gutter already marks each row. Headers that appear
exactly when they carry information are what keeps the panel calm.

## Navigation and selection reliability

Headers and the empty-section hint are decoration and must never be actionable:

- Textual's `OptionList` cursor actions skip disabled options, so `j` / `k` / arrows / `Ctrl+N` / `Ctrl+P` require no
  new logic — but this must be covered by a test rather than assumed.
- The panel currently highlights index `0` on mount and falls back to index `0` when restoring a highlight. Both must
  move to the first _enabled_ option, or the panel will open with a header selected and the description strip blank.
- Header and hint ids must not enter the row lookup map, so a highlighted-row query for one resolves to nothing and
  every action (`o`, `x`, `e`, `r`, `l`) degrades to a no-op instead of acting on an unrelated alias.
- Highlight restoration after refresh, override writes, and bucket enter/leave must continue to land on the same alias
  or bucket it does today, now that indices are shifted by header rows.
- Preserve the existing programmatic-highlight guard flag and its synchronous `finally:` reset when adjusting the
  initial-highlight logic; clearing that guard later would reintroduce the `OptionHighlighted` echo race.

## Documentation

Update the **Models Panel** section of `docs/ace.md`: describe the two sections and their counts, the ownership gutter
and where it appears, the `· <n> yours` chip on built-in buckets, the drilled-in title suffixes, the mixed-bucket rule
for in-bucket headers, and the empty-`Yours` hint. Amend the existing paragraph that documents top-level ordering to say
that sectioning groups the existing order without changing it, and amend the misplaced-builtin paragraph to state that
such an alias stays in the built-in section. No help-popup or keymap documentation changes are required, because no
binding is added or altered.

## Coverage and verification

Add data-layer tests in the alias-view test suite for: the ownership predicate on aliases and buckets; a misplaced
builtin staying built-in and un-gutted; user aliases folded into a built-in bucket counting toward that bucket's user
member count and the built-in section's counts; user-bucket and ungrouped-user rows landing in the user section; and the
order-preservation invariant that concatenating the sections reproduces the unsectioned row list exactly, at the top
level and inside buckets.

Add rendering tests for header rule construction and count alignment against the same dynamic provider/model width used
by rows, singular/plural and zero-bucket count wording, gutter presence and absence per row kind, chip append order when
a warning and an active override are also present, and the empty-`Yours` hint.

Add mounted-panel tests for: the first highlight landing on a real alias rather than a header; `j`/`k` traversal
skipping headers and the hint in both directions including wrap-around; headers appearing inside a mixed built-in bucket
and not inside a homogeneous one; the drilled-in title distinguishing user and built-in buckets; and highlight
restoration surviving a refresh and a bucket enter/leave round trip with headers present.

Extend the Models-panel PNG snapshot fixtures with a view set that contains user aliases, a user bucket, and a user
alias folded into a built-in bucket. Regenerate the existing Models-panel goldens whose list content changes, and add
new goldens for the two states this feature introduces: the empty-`Yours` section, and a mixed built-in bucket drilled
in. Inspect the visual artifacts at the production 120x40 size for clipping, confirm the gutter glyph renders under the
pinned Fira Code fixture font, and confirm the header rule and its counts align with the state column.

Before handoff, run `just install`, then the focused alias-view, Models-panel, and Models-panel navigation unit tests,
then the dedicated Models-panel visual snapshot tests, and finally the required `just check` suite. Confirm the final
diff touches no alias-resolution, schema, `default_config.yml`, keymap, or binding code.
