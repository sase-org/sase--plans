---
tier: tale
title: Show each bead's type in sase bead list
goal:
  sase bead list renders an aligned, colored, self-describing type indicator for every bead, reusing the shared
  cross-surface bead-type vocabulary the TUI and bead pages already use.
proposed_by: bbugyi200.athena.qg
create_time: 2026-07-31 10:54:09
status: done
---

- **PROMPT:** [prompts/202607/bead_list_type_indicator.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_list_type_indicator.md)

# Plan: Show each bead's type in `sase bead list`

## Problem

`sase bead list` (default `--format compact`) renders every bead as an identical row shape and gives **zero** indication
of the bead's type:

```
◐ sase-bv · Attribute beads to the agent that created them
◐ sase-bt · Fix xdist flake in artifact modal copy shortcut
```

Those two rows look the same, but `sase-bv` is an **epic-tier plan** bead and `sase-bt` is a **task** bead. Type is not
recoverable from the row:

- The status glyph (`◐`) encodes status only.
- The ID does not encode type. Per `docs/beads.md` (Issue Types table), plan and task beads **share** the
  `{prefix}-{counter}` ID format, and plan and phase beads **share** the `{parent_id}.{N}` format. So neither `sase-bt`
  nor `sase-bv.3` reveals its type.

Today the only ways to see type are `--format json`, `--format full` (a 60-dash-separated detail block per bead), or
re-running with `--type` filters. That is a poor experience for the command agents and humans reach for most.

Meanwhile a complete, accessible, cross-surface bead-type vocabulary **already exists** in this repo and is used by the
ACE TUI and generated bead pages — but the CLI never adopted it.

## Current State

| Concern                                                             | Location                                                              |
| ------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Compact row renderer (the thing to change)                          | `src/sase/bead/cli_query.py:232` `_render_list_compact`               |
| `list` handler dispatching to it                                    | `src/sase/bead/cli_query.py:72`                                       |
| `show --format compact` reusing the same renderer                   | `src/sase/bead/cli_query.py:117`                                      |
| `list` argument parser                                              | `src/sase/main/parser_bead_queries.py:76` `register_bead_list_parser` |
| Shared bead-**type** presentation (glyph/accent/label)              | `src/sase/bead_type_presentation.py`                                  |
| Shared bead-**status** presentation (glyph/`cli_style` ANSI/accent) | `src/sase/bead_status_presentation.py`                                |
| Existing bead-CLI color renderer to reuse                           | `src/sase/bead/cli_dep_render.py`                                     |
| `status_icon` helper used by compact rows                           | `src/sase/bead/cli_common.py:336`                                     |

Three facts that shape the design:

1. **The type vocabulary already exists and is shared.** `src/sase/bead_type_presentation.py` defines, per type, a
   `glyph`, an `accent_color`, a `chip_style`, and a `label`: plan `▸` `#FFD700`, phase `↳` `#87D7FF`, task `◆`
   `#D787FF`. It is already consumed by the TUI (`src/sase/ace/tui/widgets/artifacts/plans_rendering.py:262`,
   `src/sase/ace/tui/modals/bead_edit_modal.py:49`) and by generated bead pages (`src/sase/bead_pages/roster.py:40`).
   Only the CLI is missing. This work is therefore **adoption of an existing design system, not invention of a new one**
   — which is what makes it cheap, consistent, and low-risk.

2. **There is already a bead-CLI color precedent to reuse.** `src/sase/bead/cli_dep_render.py` renders almost exactly
   this row shape (`{glyph} {id} · {title}`) with ANSI color, and exports `resolve_color()` (handles
   `auto`/`always`/`never` + `NO_COLOR` + TTY detection) and `styled()`. `sase bead search` and `sase bead dep` already
   expose `-c/--color`; **`sase bead list` does not**. That asymmetry gets fixed here.

3. **`bead list` is Python-only.** `src/sase/main/bead_fast_path.py:33` explicitly defers `list` and `show` to argparse,
   so the Rust core's parallel `handle_list` (`sase-core` `crates/sase_core/src/bead/cli.rs:122`) is not reachable for
   `list` today. Python's `_render_list_compact` is the single live implementation. See **Rust Boundary** below.

## Design

### Target output

```
▸ plan  ◐ sase-bv · Attribute beads to the agent that created them
◆ task  ◐ sase-bt · Fix xdist flake in artifact modal copy shortcut
↳ phase ◐ sase-bv.3 · Record the creator on every bead creation path ← sase-bv
```

Row grammar becomes:

```
{type_glyph} {type_word}<pad> {status_glyph} {id} · {title}{ ← parent_id}
```

The existing suffix (` ← parent_id`) and the `·` separator are unchanged.

### Decision 1 — a glyph **and** a word, not a glyph alone

A bare second glyph (`▸ ◐ sase-bv · …`) would be glyph soup: two adjacent abstract symbols in different meaning-spaces,
each requiring a lookup. Spelling the type out makes the row self-describing on first read, and keeps the indicator
legible when color is off (piped output, `NO_COLOR`, dumb terminals). Color becomes **redundant reinforcement rather
than the sole carrier of meaning** — which is the stated intent of `bead_type_presentation.py` ("Accessible Rich
presentation helpers").

This also resolves a real collision: the type accents and status accents **share colors**. Type `plan` is `#FFD700` and
status `in_progress` is also `#FFD700`; type `phase` is `#87D7FF` and status `open` is also `#87D7FF`. Two adjacent
same-colored glyphs would be genuinely ambiguous. A fixed column position plus a literal word removes the ambiguity
entirely.

### Decision 2 — the word is exactly the `--type` value

The column prints `plan` / `phase` / `task`, the exact values accepted by `--type`. What you read is what you can filter
on. No translation layer, no invented display vocabulary.

### Decision 3 — reuse the accent colors verbatim via xterm-256

Every accent color in both presentation modules is an **exact xterm-256 palette entry**:

| Type  | Accent    | xterm-256 | ANSI SGR         |
| ----- | --------- | --------- | ---------------- |
| plan  | `#FFD700` | 220       | `\x1b[38;5;220m` |
| phase | `#87D7FF` | 117       | `\x1b[38;5;117m` |
| task  | `#D787FF` | 177       | `\x1b[38;5;177m` |

So the CLI can emit colors **pixel-identical** to what the TUI renders, rather than approximating with basic ANSI. Add a
`cli_style` field to `_BeadTypePresentation` mirroring the existing `cli_style` field on `_BeadStatusPresentation`, and
derive it from `accent_color` so the two can never drift.

Row coloring: type glyph + word in the type accent; status glyph keeps `bead_status_presentation(...).cli_style`; ID in
`ANSI_BOLD_BLUE` to match `cli_dep_render.render_issue`; title unstyled.

### Decision 4 — alignment is measured, not assumed

The three type glyphs do **not** share a Unicode width class: `▸` (U+25B8) and `↳` (U+21B3) are East-Asian _Neutral_,
while `◆` (U+25C6) is East-Asian _Ambiguous_ (1 cell in most terminals, 2 in CJK-configured ones). Padding with `len()`
silently assumes 1 cell per character.

Pad the type cell using `rich.cell.cell_len` (Rich's width model, which is also what the TUI renders under — so CLI and
TUI agree by construction) rather than `len()`, and lock it with a test asserting all three rendered type cells have
equal `cell_len`. Document the residual CJK-ambiguous-width caveat rather than pretending it does not exist. Note this
is not a new class of risk: the existing status glyphs already mix widths (`○◎◇◐` Ambiguous, `✓` Neutral).

While in the module, fix `BEAD_TYPE_CHIP_WIDTH` (`src/sase/bead_type_presentation.py:50`), which computes a display
width with `len()` and has the same latent flaw.

### Decision 5 — tier is deliberately **not** folded into the type column

`sase-bv` is `type=plan, tier=epic`. Rendering it as `epic` would be more specific, but it would mean a row labeled
`epic` is returned by `--type plan` and never by any `--type` value — display and filter vocabularies would diverge, and
the CLI would fork the shared three-value vocabulary that `bead_type_presentation.py` guarantees across TUI, bead pages,
and CLI. Tier stays visible through `--tier`, `--format full`, and `--format json`. Recorded as a follow-up, not a
silent omission.

### Decision 6 — default on, with JSON as the stable machine contract

The indicator is on by default (that is the point of the request). `--format json` is untouched, and remains the
supported contract for scripts. The one in-repo consumer of compact output, `smoke/pypi/smoke_check.sh:352`, greps the
**title** with `grep -F "Smoke plan"`, so a new leading column does not affect it — but the plan verifies this rather
than assuming it.

## Implementation

### 1. `src/sase/bead_type_presentation.py`

- Add `cli_style: str` to `_BeadTypePresentation`, derived from `accent_color` (hex → xterm-256 SGR) so the ANSI code
  and the Rich accent cannot drift apart. Mirror the naming already used by `_BeadStatusPresentation.cli_style`.
- Add a `bead_type_cli_cell(value, *, use_color: bool, width: int | None = None) -> str` helper returning the padded
  `{glyph} {word}` cell, padded by `rich.cell.cell_len`.
- Replace `len()` with `rich.cell.cell_len` in `BEAD_TYPE_CHIP_WIDTH`.
- Export the new names in `__all__`.

Keep `_normalize_bead_type`'s existing strictness: unknown types must not be guessed. `_render_list_compact` receives
`Issue` objects whose `issue_type` is a validated `IssueType` enum, so the honest-`unavailable` path used by
`bead_type_chip` is not needed on this row; if a type ever fails to normalize, fail loudly rather than printing a
misleading row.

### 2. `src/sase/bead/cli_query.py`

- Change `_render_list_compact(issues)` to `_render_list_compact(issues, *, use_color: bool)` and emit the new grammar,
  composing `bead_type_cli_cell(...)`, `bead_status_presentation(...).cli_style`, and `styled()` /`ANSI_BOLD_BLUE` from
  `cli_dep_render`.
- Compute the pad width once per call from the shared type-cell width (not per row).
- Pass `resolve_color(getattr(args, "color", "auto"))` from `handle_bead_list` (`cli_query.py:72`) and from
  `handle_bead_show`'s compact branch (`cli_query.py:117`).

`show --format compact` and `list` **must** stay byte-identical — `docs/beads.md:749` promises "`compact` prints the
same single row as `sase bead list`". Sharing the one renderer preserves that automatically.

### 3. `src/sase/main/parser_bead_queries.py`

Add `-c/--color {auto,always,never}` (default `auto`) to `register_bead_list_parser`, with help text copied from the
existing `search` parser (`parser_bead_queries.py:173`) so the two commands read identically. Add the same flag to
`register_bead_show_parser` so its compact branch can be controlled too.

### 4. Tests

New/updated coverage in `tests/test_bead/test_cli_list.py` (plus `tests/test_bead/test_cli_show.py` for the shared row):

- Each type renders its correct glyph + word: plan `▸ plan`, phase `↳ phase`, task `◆ task`.
- Type cells all have equal `rich.cell.cell_len` (the alignment guard from Decision 4).
- `--color never` and `NO_COLOR=1` emit **no** `\x1b[` sequences; `--color always` emits the exact expected
  `38;5;{220,117,177}` codes for the three types.
- Default (`auto`) is colorless under pytest's non-TTY capture.
- The ` ← parent_id` suffix and `·` separator are preserved.
- `list` compact output and `show --format compact` output remain byte-identical (extends the existing intent of
  `test_handle_bead_list_full_reuses_show_rendering` / `test_handle_bead_list_explicit_compact_matches_default`).
- `--format json` output is unchanged.
- The CLI-vs-shared-vocabulary tie: extend `tests/test_bead_type_presentation.py` so `cli_style` is asserted to be
  derived from `accent_color` for every type.

`tests/main/test_bead_fast_path.py:259` (`test_fast_path_defers_list_to_argparse`) already guards the Rust boundary
described below; extend its docstring to state _why_ it now matters.

### 5. Docs

- `docs/beads.md`: update the `sase bead list` section (line ~655) with the new row grammar and a **Type** glyph table
  (`▸` plan, `↳` phase, `◆` task) mirroring the existing Status Lifecycle icon table at line ~118; document `-c/--color`
  in the flag table.
- `src/sase/xprompts/skills/sase_beads.md`: the `### list` section documents the exact compact grammar
  (`[icon] [id] · [title][ ← parent_id]`) and **must** be updated to the new grammar. Note
  `tests/test_bead/test_cli_list.py:17` (`test_list_skill_examples_parse_against_cli_contract`) parses this section and
  pins the exact list of `sase bead list` examples — keep that example list intact, or update the test in the same
  change.
- `src/sase/bead/cli_admin.py:394` onboarding help: no grammar change needed, but re-read it to confirm it makes no
  claim the new column contradicts.

**Do not run `sase skill init`** in this change. Per the generated-skills workflow, the source template under
`src/sase/xprompts/skills/` is edited and committed here; chezmoi deployment happens from a clean, landed tree as a
separate step.

### 6. Verify

- `just install` first (ephemeral workspaces may have stale deps), then `just check`.
- Manually eyeball `sase bead list`, `sase bead list --status closed`, `sase bead list --color always | cat`, and
  `sase bead list | cat` for alignment and for absence of escape codes when piped.
- Confirm `smoke/pypi/smoke_check.sh:352`'s `grep -F "Smoke plan"` still matches (it greps the title, not the row
  prefix).
- Watch for Symvision unused-symbol lint on any newly exported helper — every new public name added here must actually
  be consumed.

## Rust Boundary

`CLAUDE.md`'s Rust core boundary rule says cross-frontend backend behavior belongs in `../sase-core`. This change stays
in Python deliberately:

- `sase bead list` compact rendering is **terminal presentation**, and the Python renderer is the only live one —
  `src/sase/main/bead_fast_path.py:33` hard-defers `list` to argparse.
- The Rust core carries a dormant near-duplicate (`crates/sase_core/src/bead/cli.rs:122` `handle_list`) that emits the
  old grammar and would silently regress the feature if the fast path were ever widened to include `list`.

The plan therefore treats `test_fast_path_defers_list_to_argparse` as the guard rail and **files a follow-up task bead**
to port the type column into the Rust `handle_list` (and to align Rust's duplicated `status_icon`/ANSI helpers with the
shared Python presentation modules). Doing that port here would turn a single-repo tale into a cross-repo change for a
code path no user currently reaches.

## Out Of Scope (deliberate, with follow-ups)

- **`sase bead search --format compact`** — visually inconsistent with `list` after this change, but it is
  **Rust-owned** (`crates/sase_core/src/bead/cli.rs:1750` `render_search_compact`); Python's `_render_search_compact` is
  only a fallback. Cross-repo. File a follow-up bead.
- **`sase bead ready`** (`cli_query.py:191`) — lists task beads exclusively, so a type column would be constant noise.
- **`sase bead blocked`** (`cli_query.py:205`) — mixed types, would benefit; separate small change.
- **Tier in the type column** — see Decision 5.
- Changing the shared glyphs or accent colors: they are pinned by ACE PNG visual snapshots
  (`tests/ace/tui/visual/snapshots/png/`) and by `docs/beads.md:996`.

## Risks

| Risk                                                                       | Mitigation                                                                                           |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Compact output is a de-facto contract for some agent/script                | JSON untouched; only in-repo consumer greps titles; verified explicitly                              |
| `◆` renders 2 cells in CJK-width terminals, shifting task rows by 1 column | `cell_len`-based padding + equal-width test; caveat documented; same class as existing status glyphs |
| Rust `handle_list` drifts further from Python                              | Existing defer test is the guard; follow-up bead filed                                               |
| Generated skill source edited but not deployed                             | Deployment is intentionally a separate post-land step                                                |
| New exported helpers trip Symvision unused-symbol lint                     | Called out in the verify step                                                                        |

## Done When

1. `sase bead list` shows an aligned, colored, self-describing type indicator for plan, phase, and task beads.
2. `sase bead show --format compact` emits a byte-identical row.
3. `sase bead list -c/--color {auto,always,never}` works and honors `NO_COLOR` + TTY detection, matching `search`/`dep`.
4. Piped/`NO_COLOR` output contains no escape sequences and stays greppable.
5. `--format json` is unchanged.
6. `docs/beads.md` and `src/sase/xprompts/skills/sase_beads.md` describe the new grammar and the type glyph table.
7. `just check` passes.
8. Follow-up beads filed for the Rust `handle_list` port and `search --format compact` consistency.
