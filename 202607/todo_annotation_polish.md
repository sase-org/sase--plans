---
tier: tale
title: Refine prompt-bar TODO draft-annotation styling
goal: 'Redesign the ACE prompt bar''s TODO highlighting so draft markers read as one
  cohesive, restrained, legible amber annotation system — a soft marker chip, a quiet
  note body, and a matching title count pill — replacing today''s loud solid blocks
  and ragged half-line washes, while preserving every existing detection, precedence,
  count, and presentation-only guarantee.

  '
create_time: 2026-07-22 08:33:23
status: wip
---

- **PROMPT:** [202607/prompts/todo_annotation_polish.md](prompts/todo_annotation_polish.md)

# Plan: Refine prompt-bar TODO draft-annotation styling

## Context

ACE highlights uppercase `TODO` / `TODO:` / `TODO(owner):` markers in the prompt input bar and shows an aggregated
`TODO N` count in the border title (feature from commit `583925096`, plan `202607/prompt_todo_highlighting.md`). The
feature is **presentation-only**: it never alters the text that is submitted, stashed, opened in `$EDITOR`, or launched.

The behavior is correct, but the **look is poor** and the user has asked for a lead-driven redesign that is _intuitive,
reliable, and beautiful_. The current treatment (see `_todo_highlight.py`) does three things per marker:

1. **Header** `todo.header` — bold, `warning`-contrast text on a **fully saturated `warning` fill** (defaults `#ffcc00`
   dark / `#996300` light).
2. **Body** `todo.body` — the rest of the line gets a **faint warm background wash** (`canvas` blended 14% toward
   `warning`) plus a warm foreground.
3. **Title** — a solid `TODO N` capsule in the same saturated `warning` fill.

### Why it looks bad (design critique)

- **The saturated header fill reads like a highlighter accident or an error**, not an intentional annotation. With
  several markers, the pane is peppered with fluorescent blocks that fight the syntax palette (blue Markdown headers,
  etc.).
- **The body wash produces ragged, uneven-width rectangles** — each TODO line's warm background ends at a different
  column (wherever its text ends), so the pane looks like it has stray selection artifacts or a rendering bug. It also
  **bleeds past intended boundaries**: e.g. `Keep `` `TODO: literal` `` beside the exact command.` washes across the
  closing backtick and the ordinary prose that follows.
- **Two tones per line** (bright block → faint block) look glitchy/unfinished.
- **A bare `TODO` mid-sentence** gets a solid block that looks like a mistake (this is exactly the case in the user's
  reference screenshot).

The sibling overlays (`jinja.*`, `xprompt.*`, `codeblock.*`) are almost entirely **foreground-forward** — colored
keywords, not background fills — which is the house idiom for "semantic markup in the prompt." Only genuinely transient
state (`search.*`, `yank.*`, selection, cursor) uses background fills. TODO is semantic markup, so it should lean
foreground-forward too, with at most a restrained chip on the marker word itself.

## Design goals

- **Restraint** — dramatically less saturated color and far less filled area. The marker should feel like a soft
  label/sticky-note, subordinate to search, selection, yank, and the cursor (it already yields to those — keep that).
- **Consistency / reliability** — no ragged rectangles. Any background must have a clean, bounded extent (the marker
  word only). Everything else is foreground-only, so extent is never ambiguous.
- **Legibility** — the note body is the user's real prompt text and must stay fully readable; the marker chip must have
  solid contrast in **both** dark and light themes.
- **Cohesion** — the inline marker chip and the title count pill must read as the same "system," using the same derived
  colors.

## The redesign

Replace the "bright block + ragged wash" with a **soft amber draft-annotation system**: a muted chip on the marker, a
quiet italic-and-warm note for the rest of the line, and a matching soft count pill in the title.

### 1. Marker chip (`todo.header`)

The marker (`TODO`, `TODO:`, `TODO(owner):` — the whole matched header, which is already cleanly bounded) becomes a
**soft chip**:

- Background: a gentle tint of the canvas toward `warning` (roughly 18–24% dark, 26–32% light — tune against snapshots),
  **not** the saturated fill.
- Foreground: **bold**, warm, and contrast-guaranteed against that tint (see color derivation below).

This keeps a distinctive "capsule" identity (matching the title pill) but reads as a tasteful label rather than a
fluorescent marker. Because the chip covers only the matched header characters, its edges are always clean. Do **not**
add synthetic padding spaces around the marker (that would change document columns and break the presentation-only
guarantee — padding lives only in the title pill, which is border markup, not document text).

### 2. Note body (`todo.body`)

**Remove the background wash entirely** — this is the single biggest source of ugliness. The rest of the marker's line
becomes **foreground-only**: a _subtle_ warm cast on the existing foreground (roughly 25–35% toward `warning`) rendered
**italic**, so it reads as a hand-written draft note tied to the marker, with no rectangle and no ragged edge.

- The warm foreground cast is the **load-bearing** signal and must keep the text fully readable (mostly `foreground`,
  only a warm hint). Do **not** dim it.
- Italic is an _enhancement_, not the sole signal. If the pinned visual font (Fira Code) does not render a distinct
  italic face in the SVG→PNG pipeline, the warm cast alone must still make the note legibly distinct — see Risks.
- Continue to **append the `todo.body` span** for every marker with a non-empty body, exactly as today (the
  overlay-precedence and byte-column tests depend on the span existing); only its _style_ changes.

**Sanctioned fallback:** if, during snapshot review, the warm+italic note still reads as too busy, leave the body
**completely unstyled** (chip-only marker). The chip alone is enough to mark a TODO. Prefer warm+italic; fall back
cleanly.

### 3. Title count pill

Keep the `TODO N` pill, restyled with the **same chip colors** so the inline markers and the title count read as one
system. If the soft tint reads too faint sitting on the thin border line during snapshot review, give the title pill a
slightly stronger tint than the inline chip (a small dedicated bump), but keep it in the same amber family. The
plain-text `TODO N` content is unchanged.

### 4. Color derivation (`todo_theme_colors`)

Rework the helper to return the new palette, still derived purely from the app theme's `background` (canvas),
`foreground`, and `warning`, with the same dark/light `warning` fallbacks (`#ffcc00` / `#996300`) so themes without an
explicit warning still work. Recommended derivation (final constants tuned against snapshots):

- `chip_bg = canvas.blend(warning, tint)` — `tint ≈ 0.20` dark, `≈ 0.28` light.
- `chip_fg = chip_bg.get_contrast_text(alpha=1.0).blend(warning, ~0.30)` — a contrast-guaranteed text color warmed
  toward amber; **bold**. This automatically yields pale-amber text on the dark chip and deep-amber/brown text on the
  light chip, so legibility holds in both themes.
- `note_fg = foreground.blend(warning, ~0.30)` — subtle warm cast; **italic**, **no background**.

Replace the two body-blend constants (`_TODO_BODY_BACKGROUND_BLEND`, `_TODO_BODY_FOREGROUND_BLEND`) with clearly named
new constants for the chip tint (dark/light) and the two warmth blends. Update the return shape (the old 4-tuple
included a now-removed body background); update both call sites accordingly.

## What stays exactly the same (guarantees & non-goals)

These are contracts, not implementation details — the redesign must not regress any of them:

- **Detection is unchanged.** `_TODO_HEADER_RE`, bounded-uppercase matching (`TODO`, `TODO:`, `TODO(owner):`; never
  `todo`, `TODOS`, `TODO2`, `preTODO`), body-stops-at-line-end-or-next-marker, and the byte/line overlay ceilings all
  stay as-is. No regex or span-scanning changes.
- **Presentation-only.** Submitted, stashed, `$EDITOR`, and launched text remain the literal prompt; the cursor is not
  moved during history/stash restoration.
- **Overlay precedence is unchanged.** TODO still yields to selection, search, yank, and the cursor, and still sits
  above persistent syntax highlighting. The `PromptTextArea` MRO and `_restore_todo_selection` stay. Note that with the
  body background removed, selection-restore now naturally applies only to the chip cells (the guard already skips
  styles without a bgcolor) — verify this path still behaves and tidy its comment if it references the body wash.
- **Count aggregation is unchanged.** `TODO N` still counts every marker across the whole prompt stack — active pane,
  compact inactive panes, and markers outside the viewport — and the pill disappears immediately when the last marker is
  edited away.
- **Theming.** Colors still follow the active dark/light theme and re-register on theme switch and for the Jinja overlay
  theme.
- **Out of scope:** no new keymaps, no TODO navigation/jump, no changes to detection scope, and no changes to the
  prompt-stack layout. This change is purely the visual treatment of an existing, correct feature.

## Files to change

Source (`src/sase/ace/tui/widgets/`):

- **`_todo_highlight.py`** — rework `todo_theme_colors` (new palette + return shape + constants); restyle `todo.header`
  to the soft chip (bold `chip_fg` on `chip_bg`) and `todo.body` to italic `note_fg` with **no** bgcolor in
  `_register_todo_text_area_theme`; keep appending both spans in `_build_highlight_map`; confirm/adjust
  `_restore_todo_selection` for the now-absent body background.
- **`prompt_input_bar.py`** — update `_todo_chip_markup` to the restyled soft pill and update its
  `todo_theme_colors(...)` unpacking to the new return shape.

Docs:

- **`docs/ace.md`** (the three TODO paragraphs before `### Prompt Stacks`) — rewrite to describe the soft marker chip,
  the quiet italic note body, and the matching count pill; drop the "softly warms the remainder of that line" wording.
  Keep the presentation-only / yields-to-search-selection-yank-cursor guarantees.

Tests (`tests/ace/tui/`):

- **`widgets/test_prompt_todo_highlight.py`** — `test_todo_overlay_registers_theme_styles_and_updates_on_theme_change`
  asserts `todo.body.bgcolor is not None`; update it to the new shape (`todo.body` now has **no** bgcolor and **is
  italic**; `todo.header` keeps a bgcolor and bold). Update any `todo_theme_colors(...)` index expectations to the new
  return shape. `test_todo_background_yields_to_selection_search_yank_and_cursor` targets cell 0 (a chip cell that keeps
  a bg) and should still hold — verify and adjust only if needed.
- **`widgets/test_prompt_todo_title.py`** — plain-text `TODO N` assertions and the restyle-after-theme-switch test
  remain valid; adjust only if the pill markup shape assertions change.
- **`visual/test_ace_png_snapshots_prompt_stack.py`** — same fixtures/asserts (`TODO 4`, `TODO 3`, inactive count 2,
  cursor row); **regenerate** the three golden PNGs (`prompt_todo_restored_dark_120x40`,
  `prompt_todo_restored_light_120x40`, `prompt_todo_stack_120x40`) with `--sase-update-visual-snapshots` after
  implementing, and eyeball the actual/expected/diff artifacts to confirm the new look is genuinely better.

## Testing & validation

- `just install` (ephemeral workspace may have stale deps), then `just check` (ruff + mypy) and `just test` (includes
  the visual suite).
- Regenerate only the three TODO goldens with `just test-visual --sase-update-visual-snapshots`; inspect
  `.pytest_cache/sase-visual/` artifacts to confirm: soft chips (no fluorescent blocks), clean italic notes (no ragged
  rectangles), a cohesive title pill, and solid legibility in **both** dark and light themes.
- Manually run `sase ace` and open a prompt containing a bare `TODO`, a `TODO:`, a `TODO(owner):`, and a multi-pane
  `---` stack, in both themes, to sanity-check live rendering, the count pill, and that selection/search/yank/cursor
  still override the chip.

## Risks & mitigations

- **Light-theme chip contrast** (amber text on pale-amber tint) — mitigated by deriving `chip_fg` from
  `chip_bg.get_contrast_text()` warmed toward `warning`; verify in the light golden.
- **Italic may not render distinctly** in the Fira Code SVG→PNG pipeline — the warm foreground cast is the load-bearing
  signal, so the note stays legibly distinct even if italic is dropped; the unstyled-body fallback is available if
  warm+italic reads as too busy.
- **Title pill faint on the border line** — bump the title tint slightly if the count is hard to read, staying in the
  amber family.
- **Snapshot regeneration hygiene** — regenerate only the three intended goldens and confirm no unrelated visual diffs
  sneak in.
