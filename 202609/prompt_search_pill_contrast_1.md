---
tier: tale
title: Legible prompt search pill (graphite keycap + gold count chip)
goal:
  The prompt bar's committed search pill and the active search panel's count chip are
  clearly legible and good-looking in every Textual theme, in both truecolor and
  256-color terminals, including base16 palettes that repaint xterm indices 16-21.
size: small
proposed_by: bbugyi200.apollo.0h
create_time: 2026-09-18 06:58:19
status: wip
---

# Prompt search pill: legible "graphite keycap + gold count chip" redesign

## Problem

After a prompt search is accepted (`/` or `?` in prompt NORMAL mode, then `Enter`), the
prompt bar's bottom border shows a two-tone pill such as `?just 4/6`, just left of the
`Ln, Col` readout. In the user's real TUI (default `flexoki` theme) the pill is nearly
unreadable. The `4/6` count renders as **salmon text on a mustard background**, and the
query segment is a dull slate block.

### Root cause (verified by pixel sampling of the user's screenshot)

1. **The TUI is rendering in xterm-256 mode.** Every sampled color matches Rich's
   `Color.downgrade(ColorSystem.EIGHT_BIT)` output exactly. The pill's darkened accent
   `#583984` became index 60 (`#5f5f87`). Warning `#AD8301` became index 136
   (`#af8700`). The accent match highlight `#9B76C8` became index 104 (`#8787d7`).
2. **The count text is literal black.** `_search_readout_colors()` in
   `src/sase/ace/tui/widgets/_prompt_search_readout.py` uses
   `warning.get_contrast_text(1.0)`, which returns pure `#000000`. Rich maps pure black
   to palette index **16**. base16-style terminal palettes repurpose indices 16–21 as
   extra accents; index 16 is their orange. So the "black" digits are painted
   orange/salmon on the mustard background, at roughly 1.9:1 contrast. The same thing
   happens in `textual-dark` and `textual-light`, where the count turns
   orange-on-orange.
3. **The query segment has no contrast budget.** `accent.darken(0.25)` quantizes to a
   muddy slate. It also blends into the accent-colored hint text beside it on the
   border.

Simulating 256-color quantization plus a base16 remap of index 16 reproduces the
screenshot pixel-for-pixel. The same simulation was used to validate the design below.

## Design (decided — implement as specified)

**"Graphite keycap + gold count chip."** The before/after mockup artifact
(`file:explicit:a3aea706a110e058c10fe8ea`; read it with `sase artifact read`) shows it
next to the current pill. The pill has two padded, solid segments. Visual weight goes to
the information the user is hunting for: where they are in the matches.

```
 ?just  4/6      ← full pill (plain text: " ?just  4/6 ")
 4/6             ← count-only fallback on narrow borders (plain: " 4/6 ")
```

| Part              | Text                                    | Background                                                            | Foreground                                   | Weight                              |
| ----------------- | --------------------------------------- | --------------------------------------------------------------------- | -------------------------------------------- | ----------------------------------- |
| Query chip        | `" "` + sigil + query + `" "`           | **raised graphite**: `surface.blend(foreground, 0.20)`                | query: max-contrast ink                      | query bold                          |
| Sigil (`/ ? * #`) | inside query chip                       | same as query chip                                                    | theme `accent`, nudged until ≥ 3.0:1 vs chip | bold                                |
| Count chip        | `" "` + ordinal + `"/"` + total + `" "` | theme `warning`, the same gold as the in-text current-match highlight | max-contrast ink                             | ordinal **bold**, `/total ` regular |

Design rationale (keep these properties when implementing):

- **Legend coupling.** The gold count chip uses the same `warning` background as the
  in-text `search.current` highlight, so "gold = where you are" reads at a glance. The
  accent-colored sigil echoes the accent-colored non-current match highlights.
- **Neutral graphite, not a tinted accent.** Tinted backgrounds (for example Textual's
  `accent-muted`) look lovely in truecolor. Rich's per-channel 256 quantizer turns them
  into saturated magenta (`#5f005f`). Neutral grays always land on the 232–255 grayscale
  ramp, so the keycap looks the same in both color modes.
- **Padding on both chips.** The old pill was flush-left and unpadded (`?just ` +
  ` 4/6`). Symmetric one-cell padding makes both segments read as deliberate chips.
- **Hierarchy inside the count.** The bold ordinal plus the regular `/total` reads like
  `4 of 6`.
- **Terminal safety.** Never emit pure black or white ink. Every emitted color must
  downgrade outside xterm-256 indices 16–21.

Colors come from the app's **resolved** CSS variables (`app.theme_variables`), not raw
`Theme` attributes. Several built-in themes leave `background`, `surface`, or
`foreground` as `None` on the `Theme` object: `textual-dark` has no background or
surface, and `textual-light` has no foreground. `theme_variables` always contains the
resolved values; `src/sase/ace/tui/modals/update_panel.py` already reads colors this
way. Textual refreshes CSS (and therefore `theme_variables`) before publishing
`theme_changed_signal`. The existing `SearchHighlightMixin._app_theme_changed` re-notify
therefore repaints the pill with the new palette on theme switches.

Prototype results with the exact rules below (WCAG contrast ratios):

| Theme                                                                 | Query ink / chip            | Sigil / chip   | Count ink / chip            |
| --------------------------------------------------------------------- | --------------------------- | -------------- | --------------------------- |
| flexoki (default)                                                     | `#FFFCF0` on `#494844`, 8.9 | `#AB85D8`, 3.1 | `#100F0F` on `#AC8301`, 5.5 |
| textual-dark                                                          | 7.4                         | 5.0            | `#121212` on `#FEA62B`, 9.5 |
| textual-light                                                         | 7.9                         | 3.3            | `#1F1F1F` on `#FEA62B`, 8.4 |
| all 20 Textual built-ins + pure-black and pure-white synthetic themes | ≥ 4.6                       | ≥ 3.0          | ≥ 4.4                       |

In every case, no emitted color quantizes into indices 16–21.

## Implementation

### 1. `src/sase/ace/tui/widgets/_prompt_search_readout.py`

This is presentation-only styling, so it stays in Python (no Rust core change). Replace
`_search_readout_colors(theme)`, `_theme_color`, and the `_ACCENT_FALLBACK` and
`_WARNING_FALLBACK` constants with a small palette resolver.

- `_FALLBACK_COLORS: dict[str, str]` holds `surface #1E1E1E`, `foreground #E0E0E0`,
  `background #121212`, `accent #6B4FBB`, and `warning #FFA62B`.
- Constants: `_QUERY_CHIP_RAISE = 0.20`, `_INK_MIN_CONTRAST = 4.5`,
  `_SIGIL_MIN_CONTRAST = 3.0`, and `_NEUTRAL_INKS = ("#121212", "#FAFAFA")`. Also add
  `_BASE16_REPURPOSED_INDICES = range(16, 22)`, with a comment explaining that base16
  terminal palettes repurpose those xterm-256 slots as accent colors.
- `_theme_var(variables: Mapping[str, str] | None, name: str) -> textual.color.Color`
  parses `variables[name]`. It falls back to `_FALLBACK_COLORS[name]` when the key is
  missing, the value is unparseable (for example `"auto 87%"`), the color is translucent
  (`a < 1`), or it is an ANSI color (`color.ansi is not None`). The `textual-ansi` theme
  resolves to `ansi_default`, which must not crash.
- `_relative_luminance(color)` and `_contrast_ratio(a, b)` implement WCAG 2.x. Textual
  only offers perceived `brightness`. Do **not** pick ink by `brightness` or
  `get_contrast_text`: for flexoki's gold that heuristic picks cream at 3.5:1 over dark
  ink at 5.5:1.
- `_terminal_safe(color)`: while
  `rich.color.Color.from_rgb(r, g, b).downgrade(ColorSystem.EIGHT_BIT).number` is in
  `_BASE16_REPURPOSED_INDICES`, apply `color = color.lighten(0.03)`, up to 12 steps. Use
  `from_rgb`, not `parse(hex)`.
- `_ink_for(bg, variables)`: pick whichever of the resolved `foreground` and
  `background` has the higher contrast against `bg`. If that is below
  `_INK_MIN_CONTRAST`, also consider `_NEUTRAL_INKS` and take the maximum. Return
  `_terminal_safe(best)`.
- `_ensure_contrast(color, bg, minimum)`: lighten if `_relative_luminance(bg) < 0.18`,
  otherwise darken, in 0.06 steps (at most 10) until the contrast is at least `minimum`.
- A frozen dataclass `_SearchReadoutPalette` holds four Rich `Style`s: `query` (padding
  and query text, bold), `sigil` (bold), `ordinal` (bold), and `total` (regular). A
  resolver `_search_readout_palette(variables)` builds it:
  - `query_bg = _terminal_safe(surface.blend(foreground, _QUERY_CHIP_RAISE))`
  - `query_fg = _ink_for(query_bg, variables)`
  - `sigil_fg = _terminal_safe(_ensure_contrast(accent, query_bg, _SIGIL_MIN_CONTRAST))`
  - `count_bg = _terminal_safe(warning)`
  - `count_fg = _ink_for(count_bg, variables)`

  Wrap the resolution in `functools.lru_cache(maxsize=8)`, keyed on the tuple of the
  five raw variable strings, because `_render_subtitle` runs on every cursor move while
  a pill is visible. Keep the docstring note that the roles mirror
  `SearchHighlightMixin`'s `search.match` (accent) and `search.current` (warning)
  overlay roles.

- Public formatter signatures: rename the `theme: object | None` keyword to
  `variables: Mapping[str, str] | None` on both `format_search_count_segment` and
  `format_search_readout`.
  - `format_search_count_segment` returns `" {ordinal}"` in the `ordinal` style followed
    by `"/{total} "` in the `total` style. It still returns an empty `Text` when
    `ordinal is None` or `total <= 0`.
  - `format_search_readout` with `include_query=True` returns `" "` (query style), the
    sigil (sigil style), and `_display_query(query) + " "` (query style), then appends
    the count segment. With `include_query=False` it returns only the count segment. The
    resulting plain strings are `" /alpha  2/3 "` and `" 2/3 "`.

### 2. Callers

- `_prompt_input_bar_completion_panel.py`, `_search_readout_pill`: pass
  `variables=self.app.theme_variables`. `_render_subtitle` already measures the pill
  with `cell_len(pill.plain)`, so the degrade order still works with the wider padded
  pill: base truncates, then the pill drops to count-only, then only `Ln, Col` remains.
  Confirm this; do not restructure it.
- `_prompt_input_bar_search.py`, `_search_command_status`: pass
  `variables=self.app.theme_variables` to `format_search_count_segment`, so the active
  search panel's right-edge count shows the same padded gold chip. Leave the
  `no match in this pane · N in stack` and `pattern not found` statuses unchanged.
- `grep` for any other `format_search_readout`, `format_search_count_segment`, or
  `_search_readout_colors` users and update them. Only these two callers exist today.

### 3. Unit tests: `tests/ace/tui/widgets/test_prompt_search_readout.py`

- Replace the `_theme()` `SimpleNamespace` helper with a variables-mapping helper. For
  built-in themes, resolve variables exactly as `App.get_css_variables` does:
  `{**theme.to_color_system().generate(), **theme.variables}`.
- Update the plain-text expectations to `" /alpha  2/3 "` and `" 2/3 "`. Substring
  checks such as `"2/3" in plain` and `"#alpha" in plain` still hold.
- Keep and adapt `test_search_readout_count_background_uses_theme_warning`: the count
  chip's bgcolor equals the `warning` variable. Also keep and adapt
  `test_query_and_count_backgrounds_differ_when_theme_reuses_colors`, where accent
  equals warning (as in textual-dark).
- New **regression test for the reported bug.** With resolved flexoki variables, neither
  count style's foreground is `#000000`, the count foreground equals flexoki's
  background `#100F0F`, and its EIGHT_BIT downgrade is not index 16.
- New parametrized test over every `textual.theme.BUILTIN_THEMES` entry, plus synthetic
  pure-black and pure-white themes. Every foreground and background in every palette
  style downgrades outside 16–21.
- New parametrized contrast test over the same set. Require query ink vs chip ≥ 4.5,
  count ink vs chip ≥ 4.0, and sigil vs chip ≥ 3.0. Solarized's orange caps the best
  possible count ink at about 4.4, hence 4.0.
- New test: the ordinal span is bold and the `/total` span is not.
- `_theme_var` fallback test: an empty mapping, `"auto 87%"`, a translucent value, and
  `ansi_default` all resolve to the fallbacks without raising.

### 4. PNG visual snapshots: `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`

- Add a third parametrization to `test_prompt_search_count_pill_png_snapshot`:
  `("flexoki", "prompt_search_count_pill_flexoki_120x40", "ACE prompt input - committed search count pill, flexoki theme")`.
  Flexoki is the real default (`ACE_THEME_NAME`), and it is where the bug showed.
- Regenerate the intentionally changed goldens with
  `just test-visual -- --sase-update-visual-snapshots` (or the equivalent `-k`
  selection): `prompt_search_count_pill_dark_120x40.png`,
  `prompt_search_count_pill_light_120x40.png`, the new
  `prompt_search_count_pill_flexoki_120x40.png`, and
  `prompt_search_highlight_120x40.png` (its search panel shows the count chip).
- **Open and inspect each regenerated PNG** before accepting it. Also check the
  `.pytest_cache/sase-visual/` diffs to confirm that only the pill or count chip pixels
  changed. Then run the full `just test-visual` without the update flag to confirm that
  no other golden moved.

### 5. Docs: `docs/ace.md`, "Prompt Search" section

Replace the sentence "The query segment uses the same accent family as non-current match
highlights, and the count segment uses the same warning color as the current-match
highlight." Describe the new pill:

- a graphite query chip whose sigil takes the accent color;
- a solid count chip in the theme warning color (the same gold as the current-match
  highlight), with a bold current ordinal;
- ink colors chosen for maximum contrast from the theme's own foreground and background;
- no colors that 256-color terminals with base16 palettes repaint.

Keep the rest of the section as is; the narrow-border behavior is unchanged.

## Verification

1. `just fix`, then `just check`. Run `just install` first if the workspace's venv is
   stale.
2. Run `just test-visual` as described in step 4, inspecting every changed PNG.
3. Take a live capture in the real default theme. With `sase screenshot --keep`, open
   the prompt input, type a multi-line prompt containing a word three or more times,
   press `Esc`, then `?word` `Enter` `n`. Recapture the target and inspect the bottom
   border. The pill must read `?word  2/3` as a graphite chip beside a gold chip, with
   dark digits clearly legible. Also check the active-search panel (before `Enter`),
   whose right edge must show the padded gold count chip.

## Out of scope

- In-text `search.match` and `search.current` highlight styles (`_search_highlight.py`)
  are unchanged.
- The hard-coded colors in `render_search_command_line` are unchanged.
- The user's terminal session is rendering in 256-color mode, probably because
  `COLORTERM=truecolor` is not reaching the TUI through ssh/tmux. That quantizes every
  theme color. This plan makes the pill robust either way. Forwarding
  `COLORTERM=truecolor` (and enabling tmux RGB via `terminal-features ',*:RGB'`) would
  restore full flexoki fidelity everywhere. That is an environment change, not part of
  this plan.
