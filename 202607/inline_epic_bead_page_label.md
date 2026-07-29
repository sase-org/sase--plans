---
tier: tale
title: Put the epic bead page label and its URL on one line
goal:
  Epic clan summaries render `Page:` and the complete hosted bead page URL as one logical line, so a wide detail pane
  shows the whole row aligned with its neighbours and a narrow one reflows the URL without SASE ever breaking it.
create_time: 2026-07-29 10:02:52
status: wip
---

- **PROMPT:** [202607/prompts/inline_epic_bead_page_label.md](prompts/inline_epic_bead_page_label.md)

# Plan

## Why

Commit `9e5eadc6` stopped SASE from hard-wrapping the epic bead page URL, which was the right fix. But it paid for the
uninterrupted URL by moving the `Page:` label onto its own line and starting the address at column zero:

```text
   Path: plans:202607/axe_chop_reports.md
   Bead: sase-ar
   Page:
https://github.com/sase-org/sase--beads/blob/main/pages/sase-ar/README.md
```

That break was not necessary. It is charged against `_SUMMARY_WIDTH = 76` — the fixed budget
`src/sase/scripts/sase_clan_summary_epic.py` uses when it authors the launch-time summary — not against the pane the
user actually looks at.

Measured from the reference screenshot (`20260729_095215.png`, 2870×1626, cell width 17.63 px):

| quantity                                | value                               |
| --------------------------------------- | ----------------------------------- |
| terminal                                | 163 columns                         |
| `#agent-list-container`                 | 60 columns (fixed in `styles.tcss`) |
| `#agent-prompt-scroll` border + padding | 2 + 4                               |
| stable scrollbar gutter                 | 2                                   |
| **detail-pane content width**           | **≈ 96 cells** (`terminal − 68`)    |
| bead page URL                           | 73 cells                            |
| `"   Page: "` + URL                     | **82 cells**                        |

The screenshot confirms the geometry directly: the URL row begins at content column 0 and ends about 27 cells short of
the scrollbar. So the label and the address fit on one row with roughly 14 cells to spare, and the current output
sacrifices the field alignment that every neighbouring row (`Title:`, `Goal:`, `Path:`, `Bead:`) maintains — for
nothing.

The deeper point is that the 76-cell authoring budget is not the display width and never was. The page URL is already a
deliberate, documented exception to that budget, so the 9-cell label prefix costs the URL nothing that has not already
been spent. And because the summary is persisted as Rich markup and re-parsed by the ACE panel, Rich's own word wrap
gives the responsive behaviour for free — verified end to end through the real author → `serialize_lines` →
`Text.from_markup` → reflow pipeline:

| detail-pane width | rendered result                                            |
| ----------------- | ---------------------------------------------------------- |
| 96 (the user's)   | `   Page: https://…/sase-ar/README.md` on one row          |
| 82                | one row                                                    |
| 81                | `  Page:` then the whole URL flush-left — today's output   |
| 52 (120-col ACE)  | `  Page:` then the URL folded at column 0 — today's output |

At every width the fragments rejoin to the URL byte-for-byte, and no fragment after the label carries leading
whitespace. There is therefore no width at which this change is worse than today's rendering, and a wide band where it
is better.

## User experience

Plan-backed summary — the page row rejoins the label column and reads as one family with `Path:` and `Bead:`:

```text
◆ EPIC sase-ar
  Title: AXE Chop Reports
   Goal: Selecting a chop on the ACE AXE tab shows a beautiful, colored, …
   Path: plans:202607/axe_chop_reports.md
   Bead: sase-ar
   Page: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ar/README.md
```

Bead-backed fallback — same treatment in its 6-cell reference column:

```text
Plan: 202607/axe_chop_reports.md
      /home/bryan/…/plans/202607/axe_chop_reports.md
Page: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ar/README.md
```

When the panel is too narrow for the composed row, Rich moves the entire URL to the following row, flush-left, and folds
it there:

```text
   Page:
https://github.com/sase-org/sase--beads/blob/main/pa
ges/sase-ar/README.md
```

That is exactly the current output, so nothing regresses on a narrow terminal or in the `120x40` visual goldens.

## Design decisions

1. **Compose the label and URL into one logical `Text` line; let Rich decide where it breaks.** The whole benefit comes
   from moving the decision from author time (where the width is unknown and guessed at 76) to render time (where
   Textual knows the pane). SASE still authors zero whitespace inside the URL, which is the invariant `9e5eadc6`
   established and this plan preserves verbatim.

2. **Do not reintroduce any preferred-break or wrap helper for the page value.** No `_render_field_lines`, no
   `Text.wrap`, no basename hint. The page line is built by direct append and handed on untouched.

3. **Restore `BEAD_PAGE_ROW_LABEL` to the field-shaped `"   Page: "`.** Nine cells, so
   `cell_len(BEAD_PAGE_ROW_LABEL) == PLAN_FIELD_LABEL_WIDTH` and the URL starts in the same column as every other field
   value. This is a revert of the label-only form introduced by `9e5eadc6`, not a new shape.

4. **Keep the page line as the sole intentional over-width exception.** It may exceed the renderer's `width` argument
   and `_SUMMARY_WIDTH`; every other line keeps its existing width contract. That exception already exists today — the
   current tests deliberately use a URL longer than 76 cells — so this plan widens the exception by the label prefix
   only, not in kind.

5. **Stop marking the styled URL `no_wrap=True, overflow="ignore"`.** Those flags were added to protect a _standalone_
   URL line; they do not survive markup serialization, so the TUI never sees them, and if the helper's `Text` were ever
   rendered directly they would now do the wrong thing — overflowing the pane instead of folding at column 0.
   `bead_page_url_text` stays the single styling authority (prefix in `COLOR_PLAN_PATH`, filename in bold
   `COLOR_PLAN_PATH_BASENAME`) and leaves wrapping policy to whoever composes the line.

6. **Do not make the authored width responsive.** Reading the live pane width at launch is impossible — the summary is
   frozen when the clan starts and the panel may be any size afterwards. One logical line plus Rich reflow is the
   correct answer precisely because it needs no width knowledge at author time.

7. **Do not touch layout, CSS, or the Agents split.** The fix needs no extra cells; widening the detail pane to buy 9
   cells would shrink the fixed 60-cell agent list for every other view.

8. **Preserve every resolution and omission semantic.** Stored plan provenance still beats live lookup, an unresolved
   page still renders nothing, renderers stay filesystem-free, and a page block dropped by the 30 KiB budget is still
   reported as `"bead page link"`.

## Rust core boundary

No `sase-core` change. Hosted URL resolution, plan provenance targets, and launch-time summary persistence are all
unchanged; this is presentation composition inside the existing Python/Rich summary adapters.

## Changes

### 1. `src/sase/sdd/_plan_display_rendering.py`

- Change `BEAD_PAGE_ROW_LABEL` from `"   Page:"` back to `"   Page: "`.
- In `bead_page_url_text`, construct the value as a plain `Text()` — drop `overflow="ignore"` and `no_wrap=True`. The
  span logic (directory prefix including the final `/` in `COLOR_PLAN_PATH`, filename in `COLOR_PLAN_PATH_BASENAME`,
  whole value in the basename style when there is no `/`) is unchanged.
- Add a module-private `_bead_page_line(url: str) -> Text` that returns one composed line:
  `Text(BEAD_PAGE_ROW_LABEL, style=COLOR_PLAN_SUMMARY)` followed by `append_text(bead_page_url_text(url))`. Leave its
  `no_wrap`/`overflow` unset so the console default (word wrap, fold) applies.
- Replace both two-element insertions in `render_plan_document` — the one after a `BEAD` provenance row and the one
  after the final provenance row when no `BEAD` row exists — with a single `_bead_page_line(bead_page_url)` append.
- Note in `render_plan_document`'s docstring that the optional page line is the one line permitted to exceed `width`,
  and that it is emitted unwrapped so the viewport can reflow it without breaking the URL.
- `__all__` is unchanged (`BEAD_PAGE_ROW_LABEL` and `bead_page_url_text` both stay exported; the new helper is private).

### 2. `src/sase/scripts/sase_clan_summary_epic.py`

- In `_bead_page_block_lines`, emit the optional leading blank separator followed by **one** line:
  `Text("Page: ", style="bold dim")` with `append_text(bead_page_url_text(page_url))`. Do not apply `_MUTED_STYLE` over
  the composed line — that would dim the URL's address styling; the label carries `"bold dim"` on its own, matching the
  neighbouring `Plan:` row.
- Keep the block a single `_DocumentBlock(..., omission_kind="page")` so truncation stays atomic and `_omission_line`
  keeps reporting `"bead page link"`.
- Nothing else in this file changes: `_plan_bead_page_url`, `_resolve_bead_page_url`, `main()`, the byte budget, and all
  fallback/stderr behaviour stay as they are.

### 3. `docs/beads.md`

Rewrite the trailing sentence of the bead-pages paragraph (currently "Epic clan summaries keep the literal URL on one
uninterrupted logical line …"). The new text should say that the summary places the label and the complete URL on one
logical line, that SASE inserts no break or whitespace inside the URL, and that a panel too narrow for the composed row
moves the whole address to the next row flush-left — so terminal URL matchers and copy/paste always see the complete
target. Do not promise any particular terminal width.

## Tests

### `tests/test_plan_display.py`

- `test_render_plan_document_places_page_after_bead_row`: the single line after `  Bead:` now has
  `plain == BEAD_PAGE_ROW_LABEL + page_url`. Assert it starts with `"   Page: https://"`, that the URL appears exactly
  once across all lines, that its `cell_len` exceeds the requested `width=48`, and that every _other_ line is `<= 48`.
- `test_render_plan_document_places_page_after_final_non_bead_provenance`: `lines[-2]` is still the `Agents:` row and
  `lines[-1].plain == BEAD_PAGE_ROW_LABEL + page_url`.
- Keep the no-`bead_page_url` regression test asserting byte-identical output.
- Add a reflow test over the composed page line. Wrap it with `overflow="fold", no_wrap=False` at widths `96`, `82`,
  `81`, and `52` and assert: at `82` and above it stays one fragment; below that the first fragment is the label and no
  later fragment begins with whitespace; and at every width, stripping the label prefix from the joined fragments
  reproduces the URL exactly.
- Rename `test_bead_page_url_text_handles_url_shapes_and_preserves_overflow` to drop the overflow clause. Keep the
  trailing-slash, bare-value, and multibyte-filename span assertions, and replace the overflow assertions with
  `no_wrap is None and overflow is None` — the helper defers wrapping policy to its caller.

### `tests/test_bead/test_clan_summary_epic_plan_script.py`

- `test_plan_summary_renders_recorded_bead_page_after_bead_row`: assert
  `lines[bead_index + 1] == "   Page: " + page_url`. Keep the deliberately long fixture URL, assert that page line's
  `cell_len > 76`, and keep `all(cell_len(line) <= 76 …)` for every other line. Keep the style probes at the `https://`
  and `README.md` offsets (`COLOR_PLAN_PATH` / `COLOR_PLAN_PATH_BASENAME`).
- `test_plan_summary_resolves_bare_bead_provenance_live`: same single-line assertion after `   Bead: sase-older`.
- Leave the no-`BEAD`, resolver-failure, no-bead-store-access, and byte-budget tests untouched.

### `tests/test_bead/test_clan_summary_epic_bead_script.py`

- `test_epic_summary_places_bead_page_after_plan_reference` and
  `test_epic_summary_separates_page_region_without_plan_reference`: locate the line starting with `"Page: "`, assert it
  equals `"Page: " + page_url`, that it occurs once, that it follows the plan block (or a blank separator when there is
  no plan), and that every other line is `<= 76`.
- `test_bead_page_url_reflows_without_inserting_whitespace`: widen its scope to the composed line rather than the bare
  URL — wrap `"Page: " + styled URL` at width 24, assert more than one fragment, that the first is the label, that no
  fragment after it starts with whitespace, and that joining the post-label fragments reproduces the URL exactly.
- The 1000-phase truncation test is unaffected: the page block is still dropped and still reported.

### `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py`

The three epic goldens render at `120x40`, which gives the detail pane 52 content cells — narrower than the URL — so
Rich should produce the same two visual rows as today (the label row gains only a trailing space, which `Text.wrap`'s
`rstrip_end` leaves invisible).

Therefore: run `just test-visual` **without** `--sase-update-visual-snapshots` first and expect
`agents_clan_panel_epic_120x40`, `..._level_2_120x40`, and `..._level_3_120x40` to pass unchanged. Only if a real pixel
delta appears should the goldens be regenerated, and any regenerated PNG must be opened and checked for a flush-left
continuation and a legible bold `README.md`. Keep `assert_page_svg_contains(page, "Page:")` and the existing no-I/O
fixture URL.

## Verification

```bash
just install
.venv/bin/pytest -q \
  tests/test_plan_display.py \
  tests/test_bead/test_clan_summary_epic_plan_script.py \
  tests/test_bead/test_clan_summary_epic_bead_script.py
just test-visual
just check
```

Then verify against a real epic (`sase-ar` has a hosted page):

1. `sase bead pages url sase-ar` prints the authoritative URL.
2. Render both the plan-backed and bead-backed summaries; each must contain that URL exactly once, on one logical line
   that also carries its label (`  Page:` / `Page: `).
3. Wrap that logical line at width 96 — the user's measured pane — and confirm it stays a single physical row; wrap it
   at 52 and confirm the label row plus fold-at-column-zero continuation, with the fragments rejoining to the exact URL.
4. In `sase ace`, select the epic clan on a wide terminal: the row must read `   Page: https://…/README.md` aligned
   under `Bead:`. Check the `Z` metadata zoom too.
5. In Kitty, `ctrl+shift+e` must still offer the complete hosted page as one target (manual, terminal-only).

If aggregate `just check` reports the known unrelated drift (stale generated provider skills in chezmoi, the
`model_alias_completion` sidecar plan's missing prompt backlink), record it separately and still run every
repository-local formatting, lint, type, structural, unit, and visual stage to completion.

## Non-goals

- Making `_SUMMARY_WIDTH` responsive to the live panel, or re-authoring summaries after launch.
- Changing hosted-link resolution, GitHub URL shapes, or the `<project>--beads` page layout.
- Adding page rows to tale/ordinary plan clans, the ACE PLAN lane, `sase plan show`, or per-phase beads.
- Changing the Agents-tab column ratio, scrollbar gutter, padding, keymaps, fold behaviour, or global wrapping CSS.
- Adding OSC-8 hyperlinks, a URL-opening action, or terminal-specific escape handling.
- Reintroducing preferred-break wrapping, ellipsization, or `no_wrap` cropping for the URL.
