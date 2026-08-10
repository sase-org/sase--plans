---
tier: tale
title: Show bead size in every sase bead list format
goal:
  "`sase bead list` reveals each bead's size in a way that fits the requested
  `--format`: an aligned, color-accented size column in `compact`, an honest
  stored-vs-defaulted `Size:` line in `full`, and an always-present `size` key in
  `json`."
size: medium
proposed_by: bbugyi200.athena.wz
create_time: 2026-08-10 09:40:56
status: wip
---

# Plan: Show bead size in every `sase bead list` format

## Problem

Size is a first-class bead attribute — it routes the worker model
(`@<size>_phase_worker`) and decides whether the work gets a plan-first handoff — but
`sase bead list` hides or misreports it, and the three formats disagree with each other:

- **`compact`** (the default, and the surface humans and agents read most) never shows
  size at all. `src/sase/bead/cli_query.py:409` (`_render_list_compact`) renders
  `{type} {status} {id} · {title}{badges}{ ← parent}{ ⧖ age}` with no size anywhere.
- **`full`** prints a size for every phase and task, but fabricates one when the bead
  has none: `src/sase/bead/cli_detail.py:525` (`_phase_size_value`) returns `"small"`
  for an unsized bead, so `Size: small` is printed identically whether the value was
  stored or invented. This is documented current behavior in `docs/beads.md` (the "Phase
  and task detail views always print a size" paragraph), so it must be changed
  deliberately rather than quietly.
- **`json`** omits the `size` key entirely when the bead is unsized
  (`src/sase/bead/cli_detail_json.py:90-91`), even though every sibling wire schema
  already emits `null` for the same field — `ref_to_wire_dict`
  (`src/sase/bead/cli_detail_json.py:135`), `sase bead work`
  (`src/sase/bead/work.py:242`), and the mobile bridge
  (`src/sase/integrations/_mobile_helper_beads.py:266`). Consumers therefore need two
  code paths for one field depending on which bead command produced the JSON.

## Design

Three decisions, one per format. They are already made; implement them as written rather
than re-deriving them.

### 1. `compact`: a third gutter column carrying a T-shirt size token

The compact row grows one aligned column between the status glyph and the bead ID:

```
{type_glyph} {status_glyph} {size_token} {id} · {title}{badges}{ ← parent}{  ⧖ age}
```

```
▸ ◐    sase-ho · Artifact reference xprompts  ⧖ 1d
↳ ◐  S sase-ho.5 · Prove the end-to-end contract ← sase-ho  ⧖ 1d
↳ ○  M sase-hq.3 · Build project-aware glossary catalogs ← sase-hq  ⧖ 1d
↳ ○ XL sase-hq.6 · Migrate SASE's glossary ← sase-hq  ⧖ 1d
◆ ○  L sase-hr · Refresh stale beads sidecar README  ⧖ 1d
```

Rules, in priority order:

1. **Tokens are the canonical T-shirt abbreviations**: `xsmall → XS`, `small → S`,
   `medium → M`, `large → L`, `xlarge → XL`. Uppercase, because `L` versus `l`/`1` must
   never be ambiguous in a terminal font, and because these five tokens are read
   instantly without a legend.
2. **Right-aligned in a fixed 2-cell column**, so the ID column stays vertically aligned
   no matter which sizes appear. Width is _measured_ from the token table
   (`max(cell_len(token))`), never hardcoded, matching how `_render_list_compact`
   already measures the type column at `cli_query.py:411-413`.
3. **Colored with the existing size palette.** Reuse `PHASE_SIZE_ACCENTS` from
   `src/sase/phase_size_presentation.py`, which already backs both the ACE size chips
   and `sase bead show`'s `Size:` line, so a size never has two different hues on two
   surfaces. Color obeys `-c/--color` exactly like the type glyph, status glyph, and ID
   already do; with color off the letters alone still carry the full meaning.
4. **Never fabricate.** A token appears only when `issue.size is not None`. A bead
   without a stored size gets a blank cell — the compact row makes no claim it cannot
   support. (Plan beads can never carry a size at all; see `Issue.validate` in
   `src/sase/bead/model.py:259-260`.)
5. **The column collapses when nothing in the listing is sized.** If no bead in the
   rendered batch has a stored size, the column — and its separating space — is omitted
   entirely, so `sase bead list --type plan` and legacy sizeless stores print exactly
   the bytes they print today. This is ordinary table behavior (drop an empty column)
   and it is deterministic for a given result set.

Why a gutter column and not a `[M]` badge next to `[+2]`/`[↺1]`, and not a trailing cell
next to `⧖ 1d`: bracket badges in this row grammar mean _counts_, and both alternatives
float with the variable-width title, so sizes could not be scanned down the page. The
gutter is the only position where a reader can compare 40 rows' sizes in one saccade.
Placing it _after_ the status glyph rather than before it also leaves both existing
columns exactly where users and `docs/beads.md` say they are.

Why letters rather than a bar ramp (`▁▃▅▇█`): the ramp is prettier in isolation but `▅`
versus `▇` is not reliably distinguishable at a glance, and agents read this surface too
(`/sase_new_task` runs `sase bead list --type task --since 1w --status all` while
triaging duplicates). `XS/S/M/L/XL` is unambiguous for both audiences, and the color
accent still supplies the at-a-glance ordinal cue.

### 2. `full`: keep printing a size, but mark the defaulted one

`--format full` renders `render_issue_detail`, so this also changes
`sase bead show --format full`; that is intended — the two must not disagree about the
same bead.

- A **stored** size renders exactly as today: `Size: medium`.
- An **unstored** size renders `Size: small (default)`, with the `small` in the usual
  size accent and the ` (default)` suffix in the palette's placeholder (dim) style, the
  same treatment `(none)`/`(unrecorded)` already get in this renderer.
- The same suffix applies to the `CHILDREN → PHASES` rows built at
  `src/sase/bead/cli_detail.py:221-229`.

This preserves the documented "phase and task detail views always print a size"
guarantee (the `small` fallback is _real_ — an unsized bead genuinely launches through
`@small_phase_worker`, see `src/sase/bead/work.py:471`) while ending the current
inability to tell a chosen `small` from an absent one.

The literal marker text lives in `src/sase/phase_size_presentation.py` as a single
exported constant, so the one other surface that shows this fallback (ACE's bead detail
pane, `src/sase/ace/tui/widgets/artifacts/beads_detail.py:78`) can adopt it later
without re-deciding the wording. Do not change ACE in this plan.

### 3. `json`: `size` is always present

`issue_to_wire_dict` emits `"size": null` for an unsized bead instead of dropping the
key. This makes the issue schema agree with the ref schema in the same file and with
every other bead JSON surface, and lets `jq '.results[].size'` work on every row.

Do **not** add a derived `effective_size` field; `null` plus the documented "unsized
beads launch as small" rule is enough, and a second size field would be one more thing
to keep consistent.

### Scope

In scope, because they share the compact row grammar this change defines and would
otherwise become the only bead rows that hide a size:

- `sase bead list` (`compact`, `full`, `json`)
- `sase bead show --format compact` (literally calls `_render_list_compact`)
- `sase bead search --format compact` (`_render_search_compact`, same grammar)
- `sase bead ready` and `sase bead blocked` rows (plain, uncolored variants of the same
  row; size is decision-relevant precisely when triaging a ready task)

Explicitly out of scope:

- No new or changed CLI options. In particular, no `--size` filter for `sase bead list`;
  ACE's `size:` filter token already covers filtering.
- ACE/TUI rendering, generated bead pages, the mobile bridge, and `sase bead work`
  output are unchanged.
- No edits to `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). The user has not granted
  memory-edit permission for this work. `docs/` is not memory and is updated normally.

## Implementation

### Step 1 — Extend the shared size presentation module

`src/sase/phase_size_presentation.py` is the single source of truth for size
presentation; add the CLI-row tier next to the existing Rich chip and ANSI accent
helpers.

Add and export via `__all__` (Symvision requires every new public symbol to be exported
and used):

- `PHASE_SIZE_ABBREVIATIONS: dict[PhaseSizeValue, str]` — `XS`, `S`, `M`, `L`, `XL`.
- `PHASE_SIZE_TOKEN_WIDTH: int` —
  `max(cell_len(token) for token in PHASE_SIZE_ABBREVIATIONS.values())`, measured with
  `rich.cells.cell_len` rather than `len`, mirroring `BEAD_TYPE_CHIP_WIDTH`.
- `PHASE_SIZE_DEFAULT_MARKER: str` — `"(default)"`, the suffix `--format full` appends
  to a defaulted size.
- `phase_size_cli_token(value, *, use_color, width=PHASE_SIZE_TOKEN_WIDTH) -> str` —
  returns the right-aligned, optionally ANSI-accented token cell. `None` returns
  `" " * width` (an unsized bead is a legitimate, silent case). Any _other_ unrecognized
  value raises `ValueError`, matching the deliberate contract of `bead_type_cli_cell` in
  `src/sase/bead_type_presentation.py`: rows are built from a validated `PhaseSize`, so
  a normalization failure there is a bug and must fail loudly instead of printing a
  misleading row. Build the ANSI code from
  `xterm256_foreground_style(PHASE_SIZE_ACCENTS[normalized])` and reset with
  `sase.ansi_style.ANSI_RESET`, exactly as `phase_size_cli_style` and
  `bead_type_cli_cell` do.

### Step 2 — Compact rows

In `src/sase/bead/cli_query.py`:

- Add a module-private helper that returns the size column width for a batch:
  `PHASE_SIZE_TOKEN_WIDTH` when any issue in the batch has `size is not None`, else `0`.
- `_render_list_compact` (`cli_query.py:409`): compute the width once for the batch, and
  when it is non-zero insert
  `f"{phase_size_cli_token(issue.size, use_color=..., width=...)} "` between the status
  glyph and the bead ID. When it is zero, emit nothing — not even a space.
- `_render_search_compact` (`cli_query.py:469`): same treatment, computing the width
  from `[match.issue for match in matches]`.
- `handle_bead_ready` (`cli_query.py:346`) and `handle_bead_blocked`
  (`cli_query.py:363`): same cell with `use_color=False`, since neither command exposes
  `--color`. Keep their existing row grammar otherwise.
- Leave `_row_badges` untouched: `[+N]`/`[↺N]` remain count badges and stay where they
  are.

`sase bead show --format compact` needs no change — it already routes through
`_render_list_compact`, so a single sized bead prints its column and an unsized one
prints none.

### Step 3 — `--format full` default marker

In `src/sase/bead/cli_detail.py`:

- Replace `_phase_size_value` (`cli_detail.py:525`) with a helper that returns both the
  size string _and_ whether it was defaulted, or keep it and add a sibling predicate —
  either shape is fine as long as call sites stop being unable to tell the two apart.
- Apply the marker at the `Size:` line (`cli_detail.py:126-131`) and the
  `CHILDREN → PHASES` rows (`cli_detail.py:221-229`): the size value keeps
  `phase_size_cli_style`, and a defaulted size appends
  `f" {palette.placeholder(PHASE_SIZE_DEFAULT_MARKER)}"`.
- Both plain and rich styles must remain byte-identical after SGR stripping, per the
  `--format full` styling contract documented in `docs/beads.md`.

### Step 4 — JSON schema

In `src/sase/bead/cli_detail_json.py`, replace the conditional `payload["size"]`
assignment (lines 90-91) with an unconditional
`"size": issue.size.value if issue.size else None` inside the payload literal, placed
next to the other identity fields rather than appended after the dict is built.

### Step 5 — Help text

In `src/sase/main/parser_bead_queries.py`, `register_bead_list_parser`
(`parser_bead_queries.py:78`): document the new column in the parser `description` and
add a legend line to the `epilog`, for example:

```
Size column: XS xsmall · S small · M medium · L large · XL xlarge
(shown only when a listed bead has a stored size)
```

Keep the epilog's existing example block and `DATE grammar` line, and keep options
sorted alphabetically per the CLI rules memory. No new options are added.

### Step 6 — Docs

In `docs/beads.md`:

- The `### sase bead list` section (the "Compact rows lead with an aligned, colored type
  indicator" block): update the row-grammar snippet and the example rows to include the
  size column, extend the paragraph that currently ends "The fixed first column is the
  bead type; the second glyph is status" to describe the third column, its collapse
  rule, and its `--color` behavior, and add a size legend table next to the existing
  type table.
- The `sase bead show` prose paragraph that reads "Phase and task detail views always
  print a size: they use the stored value when present and `small` when it is absent":
  keep the guarantee, and state that a defaulted size is now marked `(default)`.
- The JSON paragraph in the same section: state that `size` is always present on the
  issue object and is `null` for a bead with no stored size.
- If `sase bead ready` / `sase bead blocked` row shapes are described anywhere in this
  file, update them to match.

`docs/configuration.md` needs no change: its `sase bead list` table documents flags, and
this plan adds none.

## Verification

```bash
just install                      # ephemeral workspace: dependencies may be stale
just check                        # whole-repo lint gates + diff-scoped tests
just test-scoped                  # if iterating on tests alone
```

Run `just check-full` before landing, because this change touches shared presentation
code (`phase_size_presentation.py`) that several TUI and CLI surfaces import.

Manual confirmation on a real store:

```bash
sase bead list --color always | head -20        # column present, aligned, colored
sase bead list --color never  | head -20        # tokens still readable
sase bead list --type plan                      # column collapses; bytes unchanged
sase bead list --format json  | jq '.results[].size'
sase bead show <sized-id> --format compact
sase bead show <unsized-phase-id> --format full # Size: small (default)
```

## Tests

Extend the existing suites rather than adding a new module, except where noted.

`tests/test_phase_size_presentation.py`:

- Every `PHASE_SIZE_VALUES` entry has an abbreviation, and the abbreviations are unique.
- `phase_size_cli_token` returns cells of equal `cell_len` for every size (the alignment
  contract), right-aligned.
- `use_color=True` wraps the token in the accent derived from `PHASE_SIZE_ACCENTS` and
  resets; `use_color=False` emits no ESC bytes and the two forms are identical after SGR
  stripping.
- `None` yields a blank cell of the same width; an unknown value raises `ValueError`.

`tests/test_bead/test_cli_list.py` (mirroring the existing type-column tests at
`test_list_compact_renders_type_glyph_only_per_type` and
`test_list_compact_type_cells_share_equal_cell_width`):

- A listing containing sizes renders the expected token per size, and every row's prefix
  through the ID column has identical cell width.
- A listing with no sized bead renders byte-identically to the pre-change output (the
  collapse rule).
- A partially sized listing pads unsized rows so the ID column stays aligned.
- `--color never` / `NO_COLOR` suppress the size ANSI just like the existing color tests
  assert for type and status.
- JSON: `size` is present and `null` for an unsized bead, and equals the stored value
  otherwise.
- One cross-format coherence test: for the same fixture bead, `compact` shows the token,
  `full` shows the matching `Size:` value, and `json` carries the matching `size` — and
  for an unsized bead, `compact` shows no token, `full` shows the `(default)` marker,
  and `json` shows `null`.

`tests/test_bead/test_cli_search.py` (or the existing search test module) and the
`ready`/`blocked` tests: one row-level assertion each that a sized bead's row carries
its token.

`tests/test_bead/test_cli_golden.py` goldens under `tests/test_bead/golden/cli/`:

- The golden fixture stores under `tests/test_bead/golden/stores/` contain **no** sizes
  today, so every compact golden (`list.stdout`, `list_closed_*.stdout`,
  `list_implicit_closed*.stdout`, `search*.stdout`, `blocked.stdout`, …) must remain
  byte-identical — that is the collapse rule's regression proof. Do not edit them, and
  do not add sizes to the shared fixture stores just to exercise the column; use
  purpose-built beads in `test_cli_list.py` instead.
- `list_full.stdout`, `show_full`-style goldens, and any other full-format golden gain
  the ` (default)` suffix wherever they currently print `Size: small` for an unsized
  bead.
- `list_json.stdout`, `show_json.stdout`, `show_phase_json.stdout`, and any other
  issue-schema golden gain `"size": null` on issue objects that currently omit it.
  (`ref` objects already have it.)

Existing assertions that must keep passing unchanged: `test_cli_show_json.py`'s
`test_search_json_keeps_phase_size_in_machine_output` and the unresolved-ref payload
tests, and `tests/test_bead_time_surface_coverage.py`'s compact-row tests (the created
cell must survive the new column).

## Risks and mitigations

- **Compact output is not a machine contract, but the goldens pin it.** The ID column
  shifts right by three cells for sized listings, which breaks naive whitespace-index
  parsing of compact rows. JSON is the machine surface and is the documented one; the
  golden diff review is the checkpoint that this shift is intentional and nowhere else.
- **Shared presentation module blast radius.** `phase_size_presentation.py` is imported
  by ACE widgets, the SDD plan display, and the clan summary script. The change is
  purely additive (new constants and one new function), so those callers are unaffected
  — `just check-full` confirms it.
- **`(default)` marker changes documented behavior.** Handled by updating the
  `docs/beads.md` paragraph that documents the current fallback in the same change.
