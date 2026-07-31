---
tier: tale
title: Keep only the colored type symbol in compact bead rows
goal:
  sase bead list and sase bead show --format compact retain the beautiful, aligned, colored bead-type symbol while
  removing the redundant plan, phase, or task word from every compact row.
proposed_by: bbugyi200.athena.qg.f0
create_time: 2026-07-31 13:05:08
status: done
---

- **PROMPT:** [202607/prompts/bead_list_glyph_only.md](prompts/bead_list_glyph_only.md)

# Plan: Keep only the colored type symbol in compact bead rows

## Problem

The recently added bead-type indicator makes `sase bead list` informative, but spelling out the type makes the default
compact format less compact than it needs to be:

```text
▸ plan  ◐ sase-bv · Attribute beads to the agent that created them
◆ task  ◐ sase-bt · Fix xdist flake in artifact modal copy shortcut
↳ phase ◐ sase-bv.3 · Record the creator on every bead creation path ← sase-bv
```

The desired refinement is to keep the visual identity—the first-column type glyph and its color—while removing the
literal `plan`, `phase`, and `task` words:

```text
▸ ◐ sase-bv · Attribute beads to the agent that created them
◆ ◐ sase-bt · Fix xdist flake in artifact modal copy shortcut
↳ ◐ sase-bv.3 · Record the creator on every bead creation path ← sase-bv
```

This is a public text-contract change, not just deletion of three strings. The current implementation deliberately
measures the combined glyph/word cell, colors that cell, shares it with `show --format compact`, pins it in focused and
golden tests, documents the exact grammar twice, and has two downstream Rust/search follow-up beads whose descriptions
refer to the word-bearing grammar. All of those surfaces must move together.

## Tier

Use a **tale**. One follow-up coding agent can make and verify this small, cohesive presentation-contract refinement in
the current repository. There are no independently landable phases and no reason to coordinate multiple agents.

## Current State

| Concern                  | Location / contract                                                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ | --------------------------------------------------------------------------------------------- |
| Shared type metadata     | `src/sase/bead_type_presentation.py` defines plan `▸`, phase `↳`, task `◆`, and their accents                                                                |
| Compact type-cell helper | `bead_type_cli_cell()` currently returns padded `{glyph} {word}` and colors the entire cell                                                                  |
| Live compact renderer    | `src/sase/bead/cli_query.py` `_render_list_compact()` measures `{glyph} {word}` and renders both `list` and compact `show`                                   |
| Color behavior           | `-c/--color auto                                                                                                                                             | always | never`, `NO_COLOR`, TTY detection, status color, and ID color already work and stay unchanged |
| Focused contract tests   | `tests/test_bead/test_cli_list.py`, `tests/test_bead/test_cli_show.py`, `tests/test_bead_type_presentation.py`, and `tests/test_bead/test_claimed_status.py` |
| Disk-backed CLI contract | Nine compact `.stdout` fixtures under `tests/test_bead/golden/cli/` contain the word-bearing prefix                                                          |
| Human and agent docs     | `docs/beads.md` and source template `src/sase/xprompts/skills/sase_beads.md` describe the exact grammar                                                      |
| Rust boundary guard      | `tests/main/test_bead_fast_path.py` keeps `list` on the Python argparse path because dormant Rust output differs                                             |
| Downstream work          | Ready beads `sase-cc` and `sase-cd` ask Rust `list` and compact `search` to adopt the type column                                                            |

The compact output is Python-only today: `src/sase/main/bead_fast_path.py` explicitly defers `list` and `show` to
argparse. The dormant Rust `handle_list` and Rust-owned compact search renderer remain out of scope, but their ready
follow-up beads must describe the revised glyph-only target.

## Design

### Target grammar

```text
{type_glyph}<pad> {status_glyph} {id} · {title}{ ← parent_id}
```

For the current three glyphs, Rich measures each as one terminal cell, so `<pad>` is empty. The renderer should still
derive the maximum glyph width and pad to it rather than hard-code one cell; this protects alignment if the shared glyph
vocabulary changes later or a future glyph has a different Rich width.

### Decision 1 — the symbol alone is the type indicator

Honor the requested compact design exactly: the first field contains only `▸`, `↳`, or `◆`. Do not replace the removed
word with an abbreviation, badge, extra separator, tier label, tooltip escape, or opt-in flag.

The design remains understandable without color:

- The type glyph is always the first field; the status glyph is always the second.
- The type and status vocabularies use disjoint shapes.
- Each type has a distinct silhouette, so `--color never`, `NO_COLOR`, and piped output retain the type signal.
- The type table remains in `docs/beads.md`, and the generated bead skill maps each glyph to its type for agents.

Color continues to reinforce the mapping in terminals, but it is not the only carrier of meaning. This accepts the
intentional tradeoff that first-time readers may need the documented glyph legend in exchange for a substantially
cleaner compact row.

### Decision 2 — preserve measured alignment

Keep `rich.cells.cell_len` in the compact renderer and measure only the shared glyphs:

```python
max(cell_len(bead_type_presentation(value).glyph) for value in BEAD_TYPE_VALUES)
```

Pass that width to `bead_type_cli_cell()`. Retain the equal-cell-width regression test, but make it assert the prefix up
to the status glyph after the word is gone. Do not replace the measured width with `len()` or delete alignment merely
because all three current glyphs measure as one cell on the development terminal.

### Decision 3 — color only the symbol

Revise `bead_type_cli_cell()` from a padded `{glyph} {word}` cell to a padded glyph cell. Apply the type ANSI style and
reset around the glyph itself, then append any width padding outside that ANSI span:

```text
{type_style}{glyph}{reset}{uncolored_padding}
```

This makes `--color always` semantically precise—the symbol is colored; invisible alignment spaces are not—and avoids
future background/style leakage if the type style evolves. Keep the helper name: it still returns the complete padded
CLI column cell, and renaming an otherwise useful shared helper would add churn without clarity.

Unknown types must continue to raise `ValueError`; compact rows receive validated `IssueType` values, and silently
inventing or omitting a glyph would be misleading.

### Decision 4 — preserve every other compact-row behavior

Do not change:

- Status glyphs or their colors.
- Bold-blue ID styling.
- The `·` title separator.
- The optional ` ← parent_id` suffix.
- `-c/--color auto|always|never`, `NO_COLOR`, or TTY behavior.
- Empty/fallback notices.
- `--format full` or `--format json` output.
- The promise that `sase bead show --format compact` is byte-identical to the corresponding list row.

### Decision 5 — do not alter cross-surface chips

`bead_type_chip()` and `BEAD_TYPE_CHIP_WIDTH` serve TUI, bead pages, and other richer surfaces where a glyph-plus-word
chip remains appropriate. Leave those APIs, their width calculation, and their tests unchanged. Only the compact CLI
cell loses its word.

## Implementation

### 1. Narrow the compact CLI type cell

In `src/sase/bead_type_presentation.py`:

- Change `bead_type_cli_cell()` to build its cell from `presentation.glyph` only.
- Keep optional cell-width padding based on `rich.cells.cell_len`.
- When color is enabled, wrap only the glyph in `presentation.cli_style` and the reset code, then append uncolored
  padding.
- Update the docstring from “`{glyph} {word}` cell” to the glyph-only contract.
- Leave `bead_type_chip()`, `BEAD_TYPE_CHIP_WIDTH`, the shared glyphs/colors, normalization, and `__all__` intact.

### 2. Measure and render the glyph-only column

In `src/sase/bead/cli_query.py`:

- Change `_render_list_compact()`'s `type_width` calculation from `cell_len(f"{glyph} {value}")` to `cell_len(glyph)`.
- Keep computing the maximum once per render call and passing it to `bead_type_cli_cell()`.
- Update the nearby alignment comment so it describes glyph width rather than the old glyph/word cell.
- Keep the row assembly and the shared `list`/compact-`show` path otherwise unchanged.

### 3. Update focused behavior tests

Adjust the public contract without weakening its guarantees:

- `tests/test_bead_type_presentation.py`
  - Rename/update the parameterized helper test to require the literal glyph alone in colorless and colored modes.
  - Update the padding test to expect a glyph followed by the requested number of spaces.
  - Add or adapt an assertion proving padding is outside the ANSI reset when `use_color=True` and `width` exceeds the
    glyph width.
  - Retain exact xterm-256 style derivation and unknown-type rejection coverage.
- `tests/test_bead/test_cli_list.py`
  - Replace “glyph and word” assertions with exact glyph-first assertions for all three types.
  - Explicitly assert the obsolete words are absent from the prefix before each status glyph; titles may legitimately
    contain words such as “Plan,” so do not assert absence across the entire row.
  - Retain equal `cell_len` type-column width, exact color-style presence, colorless modes, separator, parent suffix,
    and JSON invariance coverage.
- `tests/test_bead/test_cli_show.py`
  - Update compact prefix assertions to `↳ ` and `▸ `.
  - Retain color-mode and list/show byte-identity coverage.
- `tests/test_bead/test_claimed_status.py`
  - Update the exact claimed-row expectation from `▸ plan  ◎ ...` to `▸ ◎ ...`.
- `tests/main/test_bead_fast_path.py`
  - Keep the defer test; update its docstring to call the live contract a glyph-only type column so the Rust boundary
    explanation stays accurate.

Avoid touching assertions for TUI/page chips such as `◆ task` or `↳ phase`; those are separate, intentionally
self-describing surfaces and are not compact list output.

### 4. Update the disk-backed compact contract

Regenerate or carefully update only the compact rows in these nine golden fixtures under `tests/test_bead/golden/cli/`:

- `list.stdout`
- `list_limit.stdout`
- `list_closed_unlimited.stdout`
- `list_implicit_closed.stdout`
- `list_implicit_closed_limit.stdout`
- `list_closed_default.stdout`
- `show_compact.stdout`
- `list_implicit_closed_filters.stdout`
- `list_implicit_closed_default.stdout`

Preserve fallback notices, row ordering, statuses, IDs, titles, parent suffixes, and final newlines. Full and JSON
goldens must remain byte-for-byte unchanged.

### 5. Synchronize documentation and generated-skill source

In `docs/beads.md`:

- Change the compact grammar and examples to glyph-only prefixes.
- Remove the claim that a displayed type word matches `--type`.
- Keep the Type/Icon legend so readers can decode `▸`, `↳`, and `◆`.
- Explain briefly that the fixed first column is type and the second glyph is status; color is controlled by the
  existing `-c/--color` option.

In `src/sase/xprompts/skills/sase_beads.md`:

- Change the exact grammar to `[type_icon] [status_icon] [id] · [title][ ← parent_id]`.
- Keep the plan/phase/task glyph mapping and status mapping, but remove the type-word sentence.
- Leave the executable command examples unchanged so `test_list_skill_examples_parse_against_cli_contract` continues to
  pin the same CLI examples.

Per the generated-skills memory workflow, edit only the in-repo source template. Use `sase skill init --diff` or
`--dry-run` for a read-only preview if useful, but do **not** deploy generated skills from this workspace. Deployment
happens only after the source commit lands on the canonical branch from a clean tree.

### 6. Keep downstream follow-ups accurate

The two ready task beads created with the original feature now describe downstream adoption of that feature:

- Update `sase-cc` so its explicit Rust `handle_list` target grammar is
  `{type_glyph}<pad> {status_glyph} {id} · {title}{ ← parent_id}` and no longer mentions `{type_word}`.
- Update `sase-cd` to say compact search should adopt the same glyph-only type column as list.

Preserve their IDs, titles, ready status, and scope. These are metadata corrections to already-filed follow-ups, not new
work items.

## Verification

1. Run `just install` first because this is an ephemeral workspace.
2. Run focused tests for:
   - `tests/test_bead_type_presentation.py`
   - `tests/test_bead/test_cli_list.py`
   - `tests/test_bead/test_cli_show.py`
   - `tests/test_bead/test_claimed_status.py`
   - `tests/test_bead/test_cli_golden.py`
   - `tests/main/test_bead_fast_path.py`
3. Manually inspect representative output:
   - `sase bead list --color never`
   - `sase bead list --color always | cat`
   - `NO_COLOR=1 sase bead list`
   - `sase bead show <plan-id> --format compact --color never`
   - A phase row with `← parent_id`.
4. Confirm all three type glyphs occupy equal measured cells before the status glyph and the status column aligns.
5. Confirm `--color always` emits each type's exact xterm-256 style around the glyph, followed immediately by reset,
   while `--color never`, `NO_COLOR`, and piped `auto` output contain no escapes.
6. Confirm `list --format json`, `list --format full`, and default/full `show` fixtures are unchanged.
7. Run `sase skill init --diff` only as a read-only generated-skill preview if needed; do not deploy.
8. Run the required full `just check` and resolve any failures caused by this change.
9. Re-read `sase-cc` and `sase-cd` in JSON form to verify their descriptions state the glyph-only downstream contract
   and remain `ready`.

## Risks and Mitigations

| Risk                                                       | Mitigation                                                                                                 |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Two adjacent symbols are mistaken for the same concept     | Fixed grammar (type first, status second), disjoint glyph sets, type legend, and exact contract tests      |
| Removing the word makes colorless output undecodable       | Each type keeps a distinct glyph; docs and skill map glyphs; color is reinforcement, not the sole signal   |
| Status column alignment regresses if a glyph width changes | Continue measuring every shared glyph with `cell_len`, pad to the maximum, and retain equal-width coverage |
| ANSI color leaks into padding or the next column           | Wrap only the glyph, append padding after reset, and add an exact colored-padding helper test              |
| Compact `show` drifts from `list`                          | Continue sharing `_render_list_compact()` and retain byte-identity coverage                                |
| Golden updates accidentally alter unrelated output         | Limit updates to nine compact fixtures and verify full/JSON contracts remain unchanged                     |
| Rich-surface chips accidentally lose their words           | Leave `bead_type_chip()` and chip/page/TUI assertions untouched                                            |
| Dormant Rust/search work implements the superseded grammar | Correct the descriptions of existing ready follow-ups `sase-cc` and `sase-cd`                              |

## Out of Scope

- Changing type glyphs, accent colors, status glyphs, or ID styling.
- Removing words from TUI chips, bead pages, full detail output, agent headers, or any other non-compact surface.
- Adding tier (`tale`/`epic`) to compact rows.
- Porting the row to dormant Rust `handle_list`; tracked by `sase-cc`.
- Adding a type indicator to `sase bead search --format compact`; tracked by `sase-cd`.
- Changing `sase bead ready` or `sase bead blocked` output.
- Deploying generated skills into chezmoi before the source change is committed and landed.

## Done When

1. `sase bead list` compact rows begin with only the aligned colored type glyph, followed by the existing status glyph;
   the literal type word is absent.
2. Plan, phase, and task rows render `▸`, `↳`, and `◆` respectively in both colored and colorless output.
3. `sase bead show --format compact` emits the same glyph-only row as list.
4. Color modes, `NO_COLOR`, piped output, separator, title, parent suffix, fallback notices, and row ordering are
   preserved.
5. Full and JSON output are unchanged.
6. Focused tests, nine compact golden fixtures, human docs, and generated-skill source all describe the glyph-only
   grammar.
7. Existing downstream beads `sase-cc` and `sase-cd` describe the glyph-only target and remain ready.
8. `just check` passes.
