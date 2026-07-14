---
create_time: 2026-07-14 10:49:44
status: wip
prompt: 202607/prompts/codeblock_card_highlight.md
tier: tale
---
# Beautiful Code-Block Rendering in the Prompt Input Widget

## Problem

Code-block highlighting in `PromptTextArea` (shipped in commit `a9813bf2`, "feat: highlight literal code in prompts")
renders poorly — most obviously for a fenced block with **no language**, but the root cause degrades _every_ fenced
block. Two pieces of evidence:

- A no-language block (` ``` ` / `foo bar baz` / ` ``` `) shows a faint grey rectangle hugging only the text
  `foo bar baz`, bracketed by ` ``` ` fence marks so dim they read as stray artifacts. There is no cohesive block — just
  a ragged tinted word floating between two nearly-invisible tick rows.
- Even a block _with_ a language (the current `prompt_codeblock_highlight_*` goldens) looks broken: the ` ```python `
  info string floats as an orphaned accent word next to invisible fence ticks, and the content tint hugs each line's
  text with a ragged right edge while the fence lines carry no tint at all.

### Root cause

Highlighting is emitted only as **character-range spans** fed through Textual's `TextArea` syntax map. But Textual pads
every rendered line out to full width with the editor's **base** style — in `TextArea._render_line`:

```python
text_strip = text_strip.extend_cell_length(target_width, line_style)  # line_style = base/cursor-line, never a highlight
```

So a background style attached to a character range can only ever tint the exact characters it covers. Consequences:

1. **No card.** Content lines get a tint only under their glyphs (ragged right edge, varying per line); the ` ``` `
   fence lines get no background at all (their `codeblock.fence` style is foreground-only). The three parts never fuse
   into a rectangle.
2. **Invisible fences.** `codeblock.fence` / `codeblock.delimiter` are `foreground.blend(background, 0.45)` **plus**
   `dim=True`. The double attenuation renders ` ``` ` as barely-there ticks, which also makes the adjacent language tag
   look orphaned.
3. **No test coverage for the worst case.** The visual snapshots only exercise `python`/`bash` blocks; the plain
   no-language block — the ugliest case — is never rendered in a golden, so the regression shipped unseen.

This is a **presentation-only** defect. The launch-time literal-zone semantics (inline + fenced code are inert),
`_fenced_blocks.py`, `_literal_zones.py`, and the Rust core boundary are all correct and unchanged by this work. No
`memory/*` edits and no `sase-core` changes are required or intended.

## Goals

1. **Intuitive** — a fenced code block reads at a glance as one contiguous, escaped "card," language or not. The visual
   weight is the block itself, not the fence syntax.
2. **Reliable** — the card is derived from the same `fenced_block_details` scanner the launch path uses; an unclosed
   fence tints the whole region to EOF and un-tints the instant it is closed; all new rendering is fail-open and honors
   the existing overlay size caps; search / yank / selection / cursor overlays still paint cleanly on top.
3. **Beautiful** — a crisp full-width background band with legible-but-recessive fences, a clear language tag, an inline
   "pill" for backtick spans, and real per-language token colors inside — all theme-adaptive (light + dark).

## Design

### A. Full-width "card" background band (the core fix)

Character-range spans cannot fill a line to its edge, so the block background must be painted at **render time**. There
is an established precedent for this in the same widget: `LineRenderingMixin` (`_line_rendering.py`) already overrides
`_render_line` / `render_line` to rewrite the gutter per line, mapping screen `y` → document line via
`wrapped_document._offset_to_line_info`. The card reuses exactly that pattern.

- **Record the banded lines during `_build_highlight_map`.** From `fenced_block_details(text)`, compute the set of
  document line indices each block spans — from the opening-fence line through the closing-fence line inclusive, or
  through EOF while the fence is unclosed. Store it on the widget (e.g. `self._codeblock_band_lines`). Inline spans are
  **not** banded (they remain mid-line pills, §C). When `_build_highlight_map` bails early (no backticks/tildes, or over
  the `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES` caps), the set is empty, so oversized prompts degrade to plain text.

- **Paint the band in `_render_line`.** Override `_render_line` in `CodeBlockHighlightMixin` (only `_render_line`, not
  `render_line` — both of `LineRenderingMixin`'s paths funnel the final strip through `self._render_line`, so a single
  override applies exactly once and layers on top of the gutter rewrite without double-application). Steps:
  1. `strip = super()._render_line(y)`.
  2. Map `y` → document line index via `wrapped_document._offset_to_line_info`, mirroring `LineRenderingMixin`; any
     `IndexError` / `None` / out-of-bounds returns the strip untouched (fail-open).
  3. If that line index is in `self._codeblock_band_lines`, rebuild the strip so every **non-gutter** cell whose
     background is the editor base background (or the cursor-line background — see §E) is repainted to the card
     background, while cells that already carry an overlay background (selection, search, yank, cursor, matching
     bracket) keep theirs. Gutter cells (the first `gutter_width` cells) are left untouched so line numbers keep their
     color.
  4. The whole method is wrapped fail-open; on any exception it returns the original strip.

- **Single source of truth for the card color.** The band reads its color from the registered `codeblock.content`
  style's `bgcolor` (§F), so the render path and the theme registration never diverge.

Because the band is painted beneath the overlays and only replaces base/cursor-line backgrounds, the existing layering
requirement ("code-block spans append first; every other overlay paints above") is preserved — now for the background
plane as well as the foreground spans.

The per-character `codeblock.content` span continues to be emitted as a cheap fallback tint (identical color to the
band), so a block still reads as literal even on any line the band path declines to touch. The band is authoritative for
full-width coverage; the span is defense-in-depth.

### B. Legible, recessive fences + a real language tag

- Drop `dim=True` from `codeblock.fence` and `codeblock.delimiter`; lighten the blend (e.g.
  `foreground.blend( background, ~0.35)`) so ` ``` ` is unmistakably a fence yet stays quiet against the card.
- Keep `codeblock.lang` as accent + italic. Against a visible fence and a solid card, ` ```python ` now reads as a
  proper fence-with-language instead of an orphaned word.

### C. Inline code "pill"

Inline spans are single-line and mid-line, so they stay a character-range background (no band). Polish:

- Extend the `codeblock.inline` tint to cover the **entire** span including the backtick delimiters, so
  `` `sase bead` `` renders as one contiguous pill rather than a box with the ticks hanging outside it.
- Paint the recessive fence color (§B) over the delimiter cells on top of the pill tint.
- Keep the inline tint a touch stronger than the block tint so short spans stay perceptible.

### D. No-language blocks are first-class

With A–C, a block with no or unknown info string is beautiful by construction: full-width card, legible ` ``` ` fences,
plain theme-foreground text (no injection). This is precisely the case the user flagged, and it now needs a golden
(§Testing).

### E. Cursor-line composition

`highlight_cursor_line` is `True` in insert mode, so when the cursor sits on a banded line the whole line arrives with
the cursor-line background. Treat the cursor-line background as fillable (repaint it to the card) so the card stays
solid across the cursor row; the cursor cell itself keeps `cursor_style`. This keeps the block visually intact while
editing inside it.

### F. Theme-adaptive style registration (tuning)

Re-tune the values registered in `_register_codeblock_text_area_theme`, still derived from `app.current_theme` so light
and dark both work:

- `codeblock.content` / `codeblock.inline`: `blend(background → foreground)` card tints, strong enough to read as a
  crisp rectangle now that it fills full width (inline slightly stronger). Exact percentages tuned against the
  regenerated snapshots.
- `codeblock.fence` / `codeblock.delimiter`: recessive fence color, no `dim` (§B).
- `codeblock.lang`: accent + italic (unchanged).
- No foreground on content/inline: theme foreground and injected token colors show through the card.

### Overlay suppression (already correct — keep)

`xprompt_inspect` / `alt_inspect` / `jinja_inspect` already exclude literal zones, so xprompt / directive / separator /
skill / alt / jinja styling never appears inside code (live widget and read-only previews). This design does not change
that; it only changes how the code zone itself is drawn.

## Testing

- **Widget tests** (`tests/ace/tui/widgets/test_prompt_codeblock_highlight.py`): band-line set computed correctly for a
  closed block, an unclosed-to-EOF block, multiple blocks, indented and `~~~` fences, and empty when there is no fence;
  the rendered strip for a banded line carries the card background across the full content width while the gutter is
  untouched; an active selection / search cell on a banded line keeps its overlay background; `codeblock.fence` is no
  longer `dim`; the inline pill tint spans the delimiters. Preserve/extend the existing assertions (fence/lang/content/
  inline/delimiter styles registered, python injection tokens present, MRO ordering, unclosed→closed transition,
  oversized-block fallback).
- **Visual snapshots** (`tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py`): add a **no-language** fenced
  block to the solo case (dark + light) — the regression case — and regenerate the four existing
  `prompt_codeblock_highlight_*` goldens, which now show cohesive cards. Pin with the existing `just test-visual`
  workflow; local runs use exact pixel equality, CI allows the small renderer-drift tolerance.
- **Performance** (`tests/perf/bench_tui_trace.py`): the render band is O(cells in a visible banded line) and the
  band-set build is O(blocks); neither adds keystroke-time scanning cost. Keep the existing code-heavy tokenizer p95
  under the 16 ms line.
- Run `just install` then `just check` before completion, per repo rules (remember ephemeral-workspace dependency
  refresh).

## Non-goals / follow-ups

- No change to launch-time literal-zone semantics, `_fenced_blocks.py`, `_literal_zones.py`, or `sase-core` — this is
  presentation-only.
- No `memory/*.md` edits.
- Read-only Rich/pygments preview paths (stash/history) are left as-is; giving them card parity with the live widget is
  a possible later polish, not part of this change.
