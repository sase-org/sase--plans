---
tier: tale
title: Pager line-number gutter and `;`/`:` go-to-line prompt
goal: Every SASE pager target renders an always-on line-number gutter, and `;`/`:`
  open a distinct one-line go-to-line prompt that shows the valid `1-<N>` range and
  jumps exactly.
size: medium
proposed_by: bbugyi200.athena.05a
status: done
---

# Pager Line-Number Gutter And `;` / `:` Go-To-Line Prompt

## Goal

Give the SASE pager (`src/sase/pager/`) an always-on line-number gutter that works for
every pager target — files, stdin, beads, artifact reads, hint-file views, and every
link-followed document — plus a `;` / `:` go-to-line prompt. The prompt must be visually
distinct at a glance, hide as little document content as possible (one chrome row,
exactly like vim search), and always show the valid range `1-<N>` for the target being
viewed.

## Design Decisions

These are settled; do not re-litigate them during implementation.

1. **The gutter lives in the compose layer.** Line numbers are painted by `compose_body`
   (`src/sase/pager/_layout.py`), the one seam every pager document flows through, so
   every target and entry point gets them with zero adapter changes. Plain (non-TTY /
   `--plain`) output stays number-free — piped text must remain clean.
2. **Numbering restarts per section and counts logical lines.** A section is a distinct
   target (one file, one bead), so its numbering starts at 1. A body ending in `\n` does
   NOT get a phantom trailing line: `"a\n"` is one line, exactly as an editor counts.
   Section transition rules are furniture and take no number.
3. **Gutter anatomy.** Number right-aligned in `W` cells styled `dim`, then a `dim` `│`
   separator, then one space: `f"{n:>W}│ "`. `W` is the digit count of the largest
   section line count in the document (minimum 2), so the left edge is stable across
   sections. Wrapped continuation rows keep the fence but no number: `f"{'':>W}│ "`. The
   separator matters: bodies frequently contain their own numbers (diffs, logs), and the
   fence is what keeps the gutter recognizable at a glance.
4. **Pre-wrapped hanging indent with exact row maps.** The gutter path wraps each
   section itself: split the section's final styled `Text` (after label capsules and
   prepared syntax styling are applied, so rows match what is painted) into logical
   lines, wrap each at `content_width = width - gutter_width`, prefix every visual row
   with the gutter cells, and join into one `no_wrap` `Text` (`overflow="crop"`).
   Because rows are now built rather than estimated, `compose_body` derives exact
   heights, section offsets, and a per-line row map — replacing the render-and-count
   `_measure_section_heights` pass for these sections. Every in-repo `PagerSection`
   constructor passes a `str` body, so the `Text` form (`body_text` / prepared /
   labeled) is always available; keep a defensive no-gutter fallback only for a
   hypothetical non-`Text` renderable.
5. **Compose at the width the body actually paints.** `_ensure_body`
   (`src/sase/pager/_screen_body.py`) currently composes at `scroll.size.width`, but
   `#pager-body` has `padding: 0 1`, so painted content is narrower — today that only
   skews the offset estimates, but with pre-wrapped `no_wrap` rows it would crop. Derive
   the compose width from the body's real content width (scrollable content region minus
   the Static's horizontal padding, with the existing `max(_, 1)` guard) and keep the
   width-change cache key semantics unchanged. This makes `ctrl+n`/`ctrl+p` offsets
   exact where they were estimates.
6. **The search overlay stays gutter-free.** The vim-search corpus contract requires
   character offsets over unmodified text (`search_corpus` / `styled_search_base` in
   `_layout.py`), and the overlay is already an explicitly different, unwrapped view.
   Leaving search exits back to the gutterized body via the existing restore path.
7. **Prompt anatomy.** A new one-line command strip `#pager-goto-command` sits in the
   same band as `#pager-search-command` (below the body, above the footer rule), hidden
   by default with the existing `.hidden` class — opening it costs one row of body
   height and covers nothing. Content, left to right:
   - sigil `:` in `bold #FFD75F` — gold, rhyming with the jump-label capsule yellow
     (this is a jump verb) and clearly distinct from search's spring-green `/`;
   - typed digits in `white`, then a block cursor (`" "` styled `reverse`), matching
     `render_search_command_line`'s idiom;
   - right-aligned, `dim`: the valid range as `line 1-<N>` where `<N>` is the current
     section's logical line count; when the document has more than one section, prefix
     the current section's glyph and title (truncated first when narrow) so it is
     obvious which target the range describes;
   - live validation: while the entered number is `0` or `> N`, restyle the digits
     `bold #FF5F5F` and swap the right side to `out of range · 1-<N>` in the same
     warning tone (search's "pattern not found" red).
8. **Prompt interaction.** `;` and `:` (Textual key names `semicolon`, `colon` —
   verified against this repo's Textual version) both trigger `action_goto_line` via one
   `Binding("semicolon,colon", ...)`. Neither key collides with label hints
   (alphanumeric only) nor with `PAGER_RESERVED_JUMP_COMMAND_KEYS` (hint-alphabet
   exclusions only — no allocator change). While the prompt is open, `on_key` routes to
   the goto handler FIRST (before the search controller and label matcher) and consumes
   every key so nothing falls through to scrolling, search, labels, or the `escape`
   close binding:
   - digits append; `backspace` deletes, and on empty input cancels (vim idiom);
   - `escape` cancels; `enter` on empty input cancels quietly;
   - `enter` on a valid number jumps; on an out-of-range number it notifies
     (`severity="warning"`) and keeps the prompt open for correction — never clamp;
   - every other key is consumed as a no-op (a stray letter must not dismiss the prompt
     or scroll the body). If the current section has no lines (empty document), `;`
     notifies "Nothing to jump to." and does not open the prompt. Interplay with search
     is emergent and correct: typing-mode search keeps consuming `;`/`:` as pattern
     characters; committed search exits on the passthrough key, then the binding opens
     the prompt.
9. **Jump semantics.** The prompt targets the current section (viewport-top section,
   `_current_section_index()`), and Enter scrolls so the target line's first visual row
   is the top body row — the same idiom as `ctrl+n` section jumps. As landing feedback,
   the jumped line's gutter number is emphasized (`bold` + the section's accent from
   `section_accent`) and persists as a "last jump" mark until the next jump or document
   navigation. The jump triggers one body recompose (same cost class as the existing
   `_repaint_label_state` rebuild on every label keypress).
10. **Conventions.** The footer legend is unchanged: goto is always available, and the
    house rule keeps always-available verbs (like `j`/`k`) out of the footer and in the
    `?` sheet, which gains a `; or :` → "Jump to a line (1-N)" row. Pager bindings are
    hard-coded in `PagerScreen.BINDINGS` (there is no `keymaps.pager` section in
    `src/sase/default_config.yml`), so no keymap config changes are needed — checked
    deliberately per the gotchas core memory. No config knob and no feature flag: the
    gutter is unconditional for every target, per the request; a toggle can be a
    follow-up task if ever wanted.

## Implementation Steps

### 1. Gutter primitives — new `src/sase/pager/_gutter.py`

Pure, Textual-free module (mirrors `_chrome.py` / `_layout.py` style):

- `gutter_width(max_line_count: int) -> int` — `max(len(str(max_line_count)), 2) + 2`
  (number cells + `│` + space).
- A `GutterSection` result dataclass: the gutterized `no_wrap` rows `Text`, the row
  count, and `line_rows: tuple[int, ...]` mapping logical line index → first visual row
  _within the section_.
- `apply_gutter(text: Text, *, content_width: int, number_width: int, emphasis_line: int | None = None, accent: str | None = None) -> GutterSection`
  — splits into logical lines (drop the single phantom line a trailing `\n` produces),
  wraps each logical line at `content_width` with Rich's own wrapping so style spans
  survive, prefixes gutter cells per design decision 3, and applies the emphasis style
  to the marked line's number. Handle wide (CJK) characters via Rich's cell math — never
  `len()`.

### 2. Compose integration — `src/sase/pager/_layout.py`

- `compose_body` computes the document-wide `number_width` from section line counts,
  gutterizes each section's final styled `Text` (post-labels, post-prepared-syntax: feed
  the exact renderable `_section_renderable` produces today), and assembles parts:
  full-width section rules (unchanged) + gutterized section rows.
- Heights and `section_offsets` come from gutter row counts (exact); drop the
  render-based `_measure_section_heights` pass for gutterized sections.
- Extend `ComposedBody` with `section_line_counts: tuple[int, ...]` and
  `section_line_rows: tuple[tuple[int, ...], ...]` holding _absolute_ document rows
  (rule rows already accounted for), so jump call sites have no off-by-one math.
- New `compose_body` keyword(s) for the goto mark (section index, line number) and its
  accent; thread from the screen.
- `search_corpus` / `styled_search_base` stay untouched (decision 6).

### 3. Screen wiring — `src/sase/pager/_screen_body.py`, `_screen_actions.py`

- `_ensure_body` composes at the body's real painted width (decision 5) and passes the
  goto mark. Check whether `PagerBody.get_content_height` (`_screen_widgets.py`) is
  still needed once rows are pre-wrapped; simplify only if the standalone-geometry
  behavior it protects is covered by tests.
- `_row_for_document_line` returns the exact row from `ComposedBody.section_line_rows`
  (generalize to a section-aware helper; the existing link-follow `scroll_line` call
  sites keep their section-0 behavior).
- `_occurrence_row` (`_labels.py`) estimates rows for label window scoping at full width
  today; pass the content width (width minus gutter) so window-scoped label bands stay
  aligned with painted rows.
- `_navigate_to_document` and `action_refresh` clear goto state: close the prompt, clear
  the last-jump mark.

### 4. Goto prompt — new `src/sase/pager/_screen_goto.py` + `_chrome.py` renderer

- Pure renderer
  `goto_command_line(*, digits: str, line_count: int, section_title: str | None, section_kind: str | None, width: int) -> Text`
  in `_chrome.py`, implementing decision 7 (unit-testable without Textual, like
  `render_search_command_line`).
- `PagerGotoMixin` owning `_goto_active: bool`, `_goto_digits: str`, and the last-jump
  mark state; methods: `action_goto_line`, a `handle_goto_key(event) -> bool` consumed
  from `PagerScreen.on_key` before the search controller, the enter/cancel/jump logic of
  decisions 8–9, and strip show/hide/update helpers.
- `PagerScreen` (`screen.py`): add the mixin, the
  `Binding("semicolon,colon", "goto_line", "Go to Line")`, the `#pager-goto-command`
  Static (hidden) in `compose`, and the `on_key` routing order goto → search → labels.
- `_styles.py`: CSS for `#pager-goto-command` (height 1, padding 0 1, like the search
  command strip).
- `_help.py`: add the `; or :` row to `_BINDING_ROWS` — mind the positional `rows[5:5]`
  insertions for the conditional link/section rows when choosing the spot (place the new
  row after "g / G").

### 5. Tests

- `tests/pager/test_gutter.py` (new): widths (min 2, growth at 100/1000 lines), hanging
  continuation rows, `line_rows` exactness under wrapping, trailing-newline line
  counting, emphasis styling, wide-character wrapping.
- `tests/pager/test_layout.py`: `ComposedBody.section_line_counts` / `section_line_rows`
  / offsets exactness for multi-section documents with wrapped lines; gutter presence in
  the composed renderable.
- `tests/pager/test_chrome.py`: `goto_command_line` states — idle, typing, invalid (red
  restyle + "out of range"), multi-section context, narrow-width truncation order
  (section title truncates before the range disappears).
- `tests/pager/test_app.py` (Pilot, existing patterns): `;` and `:` both open the
  prompt; digits + enter scrolls `scroll_y` to the exact expected row and sets the mark;
  digits while the prompt is open do not activate labels; `escape` cancels without
  closing the pager; enter out-of-range keeps the prompt open; `backspace` on empty
  cancels; empty document notifies without opening; `/` while the prompt is open is
  consumed.
- `tests/pager/test_help.py` coverage lives in `_help` tests — assert the new row.
- Visual PNG goldens: the gutter changes existing pager snapshots. Run
  `just test-visual`, inspect `.pytest_cache/sase-visual/` diffs, and accept intentional
  changes with `--sase-update-visual-snapshots`. Add new goldens in
  `tests/pager/visual/test_app_png_snapshots.py`: gutter on a multi-section document,
  the goto prompt in typing state, and the invalid state.

## Performance Guardrails (tui_perf)

- Composition still happens only on width change, label repaint, syntax publish, and now
  once per executed jump — never per scroll or per prompt keystroke. Prompt keystrokes
  update only the one-line strip.
- The gutter wrap pass replaces the render-and-count measuring pass it supersedes — same
  cost class, one pass per compose, no I/O, no stat/glob in render paths.
- Keep `spawn_pump_free_task` / syntax preparation flows untouched; prepared syntax text
  still lands through `_publish_syntax_update` → recompose.

## Verification

- `just check` before finishing (run `just install` first if the workspace venv is
  stale). Scoped selection should pick up `tests/pager/`.
- `just test-visual` for the PNG suite; regenerate intentionally-changed goldens and
  eyeball every diff artifact.
- `just check-full` is the landing gate and monitor-only — do not run it inline.
- Manual smoke: `sase pager <some file>` and `sase pager -` with piped stdin in a real
  terminal — gutter visible, `;` prompt opens with the right range, `:42` jumps, invalid
  input refuses, search still works and restores the gutter on exit.

## Out Of Scope

- A config knob / toggle key for hiding the gutter (follow-up task if wanted).
- Gutter numbers inside the search overlay (corpus offset contract; decision 6).
- Cross-section jumps from one prompt (the prompt targets the current section).
- Line numbers in plain/piped output.
