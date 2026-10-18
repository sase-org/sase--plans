---
tier: tale
title: Distinct cornflower-blue hue for the bg gear
goal: The bg ⚙ gear renders in cornflower blue
size: small
proposed_by: bbugyi200.athena.11m
proposed_with: claude/opus
create_time: 2026-10-18 18:15:08
status: wip
---

# Give the `bg:` gear a distinct hue: cornflower blue `#5F87FF`

## Problem

The top bar renders `tools: ⚒ N` and `bg: ⚙ N` side by side. The `⚒` chip fill is
`TOOL_RUN_ACCENT = "#87D7FF"` (light sky blue, `src/sase/tool/view_vocabulary.py`) and
the `⚙` chip fill is `PROC_GEAR_HUE = "#48CAE4"` (cyan/teal,
`src/sase/ace/tui/proc_gear_chips.py`). The two chips are nearly the same color:

| Pair (fill vs fill)        | CIELAB ΔE | protan ΔE | deutan ΔE | tritan ΔE | WCAG luminance ratio | HSV hue gap |
| -------------------------- | --------- | --------- | --------- | --------- | -------------------- | ----------- |
| `#48CAE4` vs `#87D7FF` now | 15        | 7.5       | 7.9       | 7.2       | 1.22:1               | 10°         |

(ΔE under color-vision deficiency uses Machado 2009 severity-1.0 simulation.) The same
clash repeats in the Procs tab header, which always renders the `bg` gear chip right
next to the `⚒` chip (`procs_pane_selection.py` `_title_text`).

## Chosen color and why

Use **`#5F87FF`** (xterm-256 color 69, "CornflowerBlue") as the new `PROC_GEAR_HUE`.

1. **It fills the biggest open gap in the top-bar palette.** In perceptual OKLCH hue,
   the top bar's chip/accent colors sit at: red `#FF5F5F` 24°, HITL `#FF8700` 56°,
   monitor orange `#FFAF5F` 64°, amber `#FFC000` 85°, yellow/gold 91–95°, update lime
   `#AFFF87` 136°, tab teal `#00D7AF` 173°, muted teal `#5FAFAF` 196°, current `#48CAE4`
   214°, tools sky `#87D7FF` 231°, alias/beads lavender `#AF87FF` 297°, orchid `#D787FF`
   314°, stash pink `#FF87D7` 341°. With the current teal removed, the widest gap is sky
   (231°) → lavender (297°). `#5F87FF` lands at OKLCH 267°, right in the middle (36°
   from `⚒`, 30° from lavender).
2. **It also separates by lightness.** It is much darker than the sky chip (OKLCH L 0.65
   vs 0.84), and lightness is what survives red-green color-vision deficiency. Against
   `⚒`: ΔE 15 → 57, protan 7.5 → 44, deutan 7.9 → 45, tritan 7.2 → 25, luminance ratio
   1.22 → 2.06:1, HSV hue gap 10° → 25°. It stays far from the other gear lanes that use
   the same `⚙` glyph: monitor orange ΔE 117, update lime ΔE 136, silent/failed red
   ΔE 104.
3. **The palette has no free non-blue slot.** Every other gap is ruled out:
   - Green/mint (130–170°) would clash with the green "updating SASE" gear lane
     (`UPDATE_GEAR_HUE`) and drift back toward teal.
   - Magenta/orchid (275–5° HSV) fails the existing ≥45° stash-chip hue guard in
     `tests/ace/tui/test_top_bar_palette.py`.
   - Vivid violets such as `#AF5FFF` and `#8A6CFF` fail the existing ≥3:1
     updates-surface guard (≈2.8:1).
   - Periwinkle `#8787FF` nearly merges with the alias lavender (ΔE 11.5).
   - A neutral gray would read as "disabled/stale".
4. **It keeps the documented identity.** Code and docs already call this "the blue `bg:`
   gear" / "Blue is the session proc lane" (`proc_gear_chips.py`,
   `docs/contribute/tui-rendering.md`, `docs/reference/tui/procs-tab.md`). The change
   makes that wording accurate (blue, not teal), so no doc edits are needed.
5. **It is an exact xterm-256 cube color**, like its neighbors (`#87D7FF`=117,
   `#FFAF5F`=215, `#AFFF87`=156, `#AF87FF`=141, `#FF87D7`=212). On 256-color terminals
   it renders without quantization. Today `#48CAE4` quantizes to roughly `#5FD7D7`, a
   literal teal.

It passes every existing guard in `tests/ace/tui/test_top_bar_palette.py`:

- dark ink `#1a1a1a` on it: 5.30:1 (needs ≥ 4.5)
- vs updates surface `#244A14`: 3.10:1 (needs ≥ 3.0)
- HSV hue distance from stash `#FF87D7`: 95° (needs ≥ 45°)

As text it reads 5.7:1 on the dark `#121212` background, up from 1.86 → 3.15:1 on light
`#FAFAFA`.

The weak spot: under protanopia it lands close to the alias-override lavender pill (ΔE
5.8). That pill is two groups away (`tools · bg · updates · overrides`), has a different
shape (two-part pill), and shows only when overrides are set, so this is acceptable.

## Scope: one procs identity, everywhere

`PROC_GEAR_HUE` is the procs-lane identity color, not just the top-bar chip. The Procs
filter bar accent already derives from it. Change the constant and route the hard-coded
`#48CAE4` duplicates through it, so the TUI ends up with exactly one procs hue.
Presentation-only change: nothing crosses into `sase-core`, and no config or keymap
values are involved.

## Implementation steps

1. **`src/sase/ace/tui/proc_gear_chips.py`**: set `PROC_GEAR_HUE = "#5F87FF"`. Add a
   short comment above it explaining the choice: cornflower blue (xterm 69), darker and
   bluer than the `⚒` sky `#87D7FF` beside it so the two chips separate by both hue and
   lightness. The module docstring's "Blue is the session proc lane" stays.
2. **`src/sase/ace/tui/modals/procs_pane_render.py`**: in `_render_log_line`, the
   `progress` stream currently uses `style="bold #48CAE4"`. Change it to
   `f"bold {PROC_GEAR_HUE}"`, adding `PROC_GEAR_HUE` to the existing
   `from ..proc_gear_chips import UPDATE_GEAR_HUE` import. Leave `_TOOL_RUN_GLYPH_STYLE`
   alone.
3. **`src/sase/ace/tui/modals/runners_modal.py`**:
   - Import `PROC_GEAR_HUE` from `..proc_gear_chips`. Use a module-level import; if it
     creates an import cycle, use a local import inside the two methods.
   - Replace the four `"#48CAE4"` literals in the Procs section (header, entry, empty
     row, footer) and the running badge style `"bold #1a1a1a on #48CAE4"` with the
     constant (`f"bold #1a1a1a on {PROC_GEAR_HUE}"`).
   - Change the comment `# Procs section (teal)` to `# Procs section (blue)`.
4. **`src/sase/ace/tui/modals/procs_filter_bar.py`**: change the class docstring's
   "accented in the procs teal" to "accented in the procs blue".
   `ACCENT = PROC_GEAR_HUE` already follows the constant.
5. **`src/sase/ace/tui/styles.tcss`**: the
   `ProcsFilterBar .filter-bar-completion > .option-list--option-highlighted` rule has
   `background: #48CAE4;`. Change it to `#5F87FF`; TCSS cannot import the Python
   constant.
6. **Tests: update pinned literals.**
   - `tests/ace/tui/test_proc_gear_chips.py`: `assert PROC_GEAR_HUE == "#5F87FF"`.
   - `tests/ace/tui/widgets/test_proc_indicator.py`: expected style
     `"bold #1a1a1a on #5F87FF"`.
   - `tests/ace/tui/widgets/test_procs_filter_bar.py`:
     `ACCENT == PROC_GEAR_HUE == "#5F87FF"`.
   - `tests/ace/tui/widgets/test_top_bar_group.py`: the three arbitrary probe bodies
     `Text(" 2 ", style="bold #1a1a1a on #48CAE4")` change to `#5F87FF`, so the retired
     teal does not linger. Behavior-neutral.
7. **New regression guard in `tests/ace/tui/test_top_bar_palette.py`.** Add
   `test_proc_gear_stays_distinct_from_tools_chip`, importing `TOOL_RUN_ACCENT` from
   `sase.tool.view_vocabulary` and reusing the file's existing `_contrast_ratio` and
   `_hue_distance` helpers. Assert both:
   - `_contrast_ratio(PROC_GEAR_HUE, TOOL_RUN_ACCENT) >= 1.8`. This is a lightness step,
     the property that keeps adjacent chips apart under color-vision deficiency. Old
     pair 1.22, new 2.06.
   - `_hue_distance(PROC_GEAR_HUE, TOOL_RUN_ACCENT) >= 20.0`. Old 10°, new 25°.

   Use module constants such as `_MIN_TOOLS_PROC_CONTRAST = 1.8` and
   `_MIN_TOOLS_PROC_HUE_DISTANCE = 20.0`, failure messages in the same style as the
   neighboring tests, and a docstring explaining that `tools:` and `bg:` render
   adjacent, so their fills must differ in both lightness and hue. The new test must
   fail against the old `#48CAE4`.

8. **Final sweep:** `git grep -i 48cae4` must return nothing.

## PNG visual goldens

The Procs tab header always draws the `bg` gear chip (dim at zero), and the Procs filter
bar uses the accent. So PNG goldens under `tests/ace/tui/visual/snapshots/png/` will
drift. Expected at least:

- `config_center_procs_tab_*`
- `tool_runs_procs_marker_120x40.png`
- possibly `config_center_home_resume_procs_*`
- any golden whose top bar shows a nonzero `bg:` chip

Regenerating:

- Run `just fix-tui-screenshots` through `/sase_monitor`; it can outrun a turn. Targeted
  selectors after `--` are fine for a first pass, but finish with a full run so no stray
  golden is missed.
- In the follow-up, inspect the retained report and every changed golden. The only
  acceptable differences are procs-hue pixels: the `⚙` gear chip fill or dim gear text,
  the Procs filter-bar accent and completion highlight, progress log lines, and the
  runners-modal Procs section.
- Anything else is a regression to investigate, not accept.
- Read the WARNING block; a `partial` status means some goldens were left untouched.

## Verification

- `sase tool run check` must pass. Do not run `just check-full`.
- Targeted tests:
  - `tests/ace/tui/test_top_bar_palette.py`
  - `tests/ace/tui/test_proc_gear_chips.py`
  - `tests/ace/tui/widgets/test_proc_indicator.py`
  - `tests/ace/tui/widgets/test_procs_filter_bar.py`
  - `tests/ace/tui/widgets/test_top_bar_group.py`
  - `tests/ace/tui/test_procs_pane_header_counts.py`
- Optionally take a live capture of the top bar with a running background proc
  (`sase screenshot`) and confirm the `⚙` chip reads as a clearly darker, bluer chip
  than the `⚒` chip beside it.

## Out of scope

- `TOOL_RUN_ACCENT` and the provider-priority pill both use `#87D7FF`. That overlap is
  pre-existing and not changed here.
- The runners modal's `success`/`error` badge colors (`#2ECC71`/`#E74C3C`) are unrelated
  to the procs identity and stay as they are.
