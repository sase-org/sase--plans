---
tier: tale
title: Frame the top-bar current-project chip in its own project-colored box
goal:
  The `+<project>` chip renders as a self-contained box drawn in that project's accent
  color, visually separated from the LLM provider indicators on its left and the
  notification chips on its right.
size: medium
proposed_by: bbugyi200.athena.06x
create_time: 2026-08-18 18:19:35
status: wip
---

# Plan: Frame the top-bar current-project chip in its own project-colored box

## 1. Problem

`CurrentProjectIndicator` (`src/sase/ace/tui/widgets/current_project_indicator.py`)
renders the current project as bare colored text — `" +"` in `dim <accent>` followed by
the display name in `bold <accent>` — and mounts it between
`#provider-disables-indicator` and `#stashed-prompts-indicator` in
`src/sase/ace/tui/_app_layout.py`.

In the resting state its neighbours are also bare colored text: the default-model
readout on the left (cyan, e.g. `CODEX(gpt-5.6-codex)`) and the notification chips on
the right (`⚑1 ✉18`, gold/teal). Three runs of colored glyphs separated only by single
spaces read as one continuous strip, and the project accent can land near a
notification-tab color (the `sase` accent `#B46817` sits beside the gold `⚑`).
`docs/ace.md` already concedes the symptom: the chip "reads flush against the
default-model indicator".

The chip must become its own object: a box, drawn in the project's color, that the eye
separates from both neighbours before reading any text.

## 2. What "a box" can be on a one-row top bar

`#top-bar` is `height: auto; max-height: 1` (`src/sase/ace/tui/styles.tcss:10`). A
four-sided box needs three rows: a top edge row, the content row, and a bottom edge row.
Every Textual border type (`round`, `solid`, `tall`, `hkey`, …) consumes one row above
and one below, so a real `border:` on this widget costs two permanent rows of app height
for one chip. That trade is not worth it and is not what this plan does.

Two single-row routes to the missing horizontal edges were evaluated and rejected on
evidence:

- **Overline + underline text decorations** (SGR 53 / SGR 4) would draw top and bottom
  edges inside the one cell row. Rejected: Rich's SVG export writes only `underline` and
  `line-through` (`rich/console.py:2384-2387`), so the top edge would be **invisible in
  every PNG golden** — the repo's own visual gate could never see the feature — and SGR
  53 is the least uniformly supported decoration across terminals and multiplexers. An
  edge that silently disappears fails the reliability bar.
- **Underline alone** (bottom edge only, no top) renders everywhere but is asymmetric;
  it reads as underlined text or an open-topped cup, not a box, and collides visually
  with the underline styling this app already uses for links and matched text.

What ships instead is the box reduced to exactly what one row can carry, using Textual's
own one-row-friendly border grammar (`vkey`, i.e. `▏` / `▕`, from
`textual._border.BORDER_CHARS`):

- **Two vertical accent rails** — `▏` (U+258F LEFT ONE EIGHTH BLOCK) and `▕` (U+2595
  RIGHT ONE EIGHTH BLOCK) in the project accent. Their ink sits at the outer edge of
  their own cell, so they read as the left and right _walls_ of a box (a centered `│`
  reads as a separator between neighbours, which is the opposite of the goal), and the
  empty 7/8 of each cell supplies the box's inner padding for free.
- **A plate**: the cells between the rails are painted one surface step above the app
  background, tinted with the project's own accent, so the box has a body in the
  project's color instead of only an outline. The row's own top and bottom are the box's
  remaining edges.

The rails plus the tinted body are what make it a box rather than two dividers, and both
are drawn in the project's color, which is what the request asks for.

## 3. Visual specification

For a resolved project with display name `sase` and accent `#B46817`, the widget's
`Text` is exactly:

```
 ▏+sase▕
```

| Span       | Content          | Style                      |
| ---------- | ---------------- | -------------------------- |
| clearance  | `" "`            | unstyled (app background)  |
| left rail  | `"▏"`            | `<accent> on <plate>`      |
| sigil      | `"+"`            | `dim <accent> on <plate>`  |
| name       | `<display_name>` | `bold <accent> on <plate>` |
| right rail | `"▕"`            | `<accent> on <plate>`      |
| clearance  | `" "`            | unstyled (app background)  |

Rules:

- Plain text is `f" {LEFT}+{display_name}{RIGHT} "` — width `len(display_name) + 5`, two
  cells wider than today's `len + 3`.
- The two clearance spaces are **outside** the plate and carry no background, so the box
  never touches a neighbouring pill's fill or a notification chip.
- No inner padding spaces: the one-eighth rails already leave 7/8 of a cell of air on
  their inner side. Adding spaces was prototyped and floats the walls too far from the
  name.
- Rails are neither `bold` nor `dim`: full accent keeps them crisp on terminals that
  ignore or over-apply `dim`, and one-eighth of a cell of ink is already visually light.
- The sigil stays `dim` and the name stays `bold`, preserving today's hierarchy and the
  chip's tie to the `+` launch picker.
- The hidden states are unchanged and must stay **zero width**: `indicator: false` or an
  unresolved project still returns `Text("")` — no rails, no plate, no clearance.
- Tooltip text, the click action (`start_custom_agent`), the mount position, the 5s peek
  cadence, and the off-thread resolve are all unchanged.

Prototyped through the pinned snapshot renderer against both bare-text neighbours and
active gold/violet override pills: the outlined chip stays legible as a distinct object
in both cases, and its outline-vs-fill contrast with the provider pills keeps it from
joining that run.

## 4. Plate color math

The plate is `background` blended `0.12` toward the accent, computed with
`textual.color.Color.blend` — the same "derive a sibling color by blending toward a
theme color" pattern as `derive_argument_color` in
`src/sase/xprompt/highlight_theme.py`.

`0.12` is not arbitrary. Measured across all 18 `PROJECT_ACCENTS`:

| Reference background            | plate vs background | accent on plate |
| ------------------------------- | ------------------- | --------------- |
| `#100F0F` (pinned flexoki bg)   | 1.11 – 1.12         | 4.00 – 4.05     |
| `#121212` (stock dark)          | 1.11 – 1.13         | 3.88 – 3.96     |
| `#1E1E1E` (stock dark surface)  | 1.13 – 1.15         | 3.41 – 3.48     |
| `#E0E0E0` (stock light)         | 1.13 – 1.14         | 2.82 – 2.86     |
| `#D8D8D8` (stock light surface) | 1.12 – 1.14         | 2.63 – 2.67     |

The pinned theme's own `$surface` (`#1C1B1A`) sits at contrast **1.11** over its
`$background` (`#100F0F`). So at `0.12` the chip is lifted by _exactly one surface step_
— the app's own elevation unit — and that step is hued with the project accent. It reads
as raised material, never as a highlight. Accent-on-plate retains 0.873–0.901 of the
accent's contrast against the bare background on every reference surface, so the body
color costs at most ~13% of the name's legibility, and stays above the palette's own 3.3
dark floor everywhere ACE actually runs.

ACE pins one theme (`self.theme = ACE_THEME_NAME` in
`src/sase/ace/tui/actions/_state_init_runtime.py:66`, flexoki), but the plate is derived
from the live theme background rather than a hardcoded hex, so it stays correct if that
pin ever changes.

## 5. Implementation

### 5.1 `src/sase/ace/tui/project_styles.py` — plate derivation

Add the blend constant and one public helper beside `project_accent`:

```python
# The chip plate is one surface step above the app background, tinted with the
# project's own accent. 0.12 puts plate-vs-background at contrast 1.11-1.12 for
# every palette entry -- the same step the pinned theme's $surface (#1C1B1A)
# takes over its $background (#100F0F) -- so the chip reads as raised material,
# not a highlight, while accent-on-plate keeps >= 87% of the accent's contrast
# against the bare background.
PROJECT_CHIP_PLATE_BLEND = 0.12


@functools.lru_cache(maxsize=256)
def project_chip_plate(accent: str, *, background: str) -> str:
    """Return the plate color behind a project chip drawn on ``background``."""
```

- Implement with
  `Color.parse(background).blend(Color.parse(accent), PROJECT_CHIP_PLATE_BLEND).hex`
  (`from textual.color import Color`); the module is under `ace/tui`, so the Textual
  import is in-layer.
- `lru_cache` keeps this off the render path's cost budget (`tui_perf.md` rule 8): pure
  in-memory math, no I/O, memoized like `_project_accent_map_cached`.
- Symvision: `project_chip_plate` is public and gains its one non-test consumer in the
  same change (§5.2). Do **not** add an `--epic-symbol` entry and do not land the helper
  without its caller — that is precisely the failure mode the `sase-pw` epic notes
  record repeatedly.

### 5.2 `src/sase/ace/tui/widgets/current_project_indicator.py` — render the frame

- Add private module constants with codepoint comments:
  `_FRAME_LEFT = "▏"  # U+258F LEFT ONE EIGHTH BLOCK` and
  `_FRAME_RIGHT = "▕"  # U+2595 RIGHT ONE EIGHTH BLOCK`.
- Extend `_build_content` to `(project, *, accent, plate, indicator)` and emit the six
  spans from §3. Keep it a `@staticmethod` so the existing unit tests and the visual
  fixture can drive it directly.
- Resolve the plate on the UI thread in `_apply_content`, from the cached snapshot's
  accent plus the live theme background:

  ```python
  plate = project_chip_plate(accent, background=self._theme_background()) if accent else ""
  ```

- Add `_theme_background(self) -> str`: guarded
  `getattr(self.app, "current_theme", None)` when attached (the established pattern —
  see `prompt_input_bar._current_theme` and `confirm_dialog._resolve_current_theme`),
  returning `theme.background` when it is a non-empty string. Fall back to
  `BUILTIN_THEMES[ACE_THEME_NAME].background` resolved through a small
  `functools.cache`d private helper with function-scope imports, mirroring
  `highlight_theme()`; do not hardcode a hex in the widget.
- The plate is computed at paint time, so a theme change is picked up on the next
  `_apply_content` (at worst one 5s tick). No new timers, workers, or disk reads.
- No `styles.tcss` change: painting the plate through span styles (rather than a CSS
  `background` on the widget) is what keeps the hidden state at zero width instead of a
  two-cell colored gap.

### 5.3 `docs/ace.md`

- **Current project** section (~line 3414): replace the "reads flush against the
  default-model indicator" sentence with the framed description — the chip is a boxed
  `+<project>`, walls and body both drawn from the project's unique accent, deliberately
  set apart from the model indicators on its left and the notification chips on its
  right.
- **Current Project Indicator** section (~line 3469): same correction, keeping the mount
  position, the click-to-open-picker behaviour, and the `ace.current_project.indicator`
  pointer.
- No configuration change: `ace.current_project.indicator` already turns the chip off,
  and the frame is not a user-tunable style. Leave `default_config.yml`,
  `sase.schema.json`, and `docs/configuration.md` untouched.

## 6. Tests

### 6.1 `tests/ace/tui/test_current_project_indicator.py`

- Update `test_resolved_project_renders_display_name_with_accent`: assert
  `text.plain == " ▏+sase▕ "` and that the spans carry `f"{accent} on {plate}"` (both
  rails), `f"dim {accent} on {plate}"` (`+`), and `f"bold {accent} on {plate}"` (name),
  with `plate` obtained from `project_chip_plate(accent, background=...)` rather than a
  literal hex.
- Extend `test_unresolved_and_disabled_render_empty` so it also asserts that neither
  frame glyph appears in the hidden states (guards the zero-width invariant).
- Add a test that `_theme_background` falls back to the pinned theme background when the
  widget is unattached or `app.current_theme` exposes no usable background, and that
  `_apply_content` still renders in that case.

### 6.2 `tests/ace/tui/test_project_styles.py`

Add plate coverage, reusing the WCAG helper shape already established in
`tests/ace/tui/test_artifacts_provider_palette.py` (`_relative_luminance`,
`_contrast_ratio`), over the five reference surfaces `#100F0F`, `#121212`, `#1E1E1E`,
`#E0E0E0`, `#D8D8D8`:

- `project_chip_plate` is deterministic, is distinct per accent, and never returns the
  background unchanged.
- **One surface step**: `contrast(plate, background)` ∈ `[1.05, 1.25]` for every accent
  on every reference surface (measured 1.11–1.15) — the plate can neither vanish nor
  become a loud fill.
- **Legibility retention**:
  `contrast(accent, plate) >= 0.85 * contrast(accent, background)` for every accent on
  every reference surface (measured ≥ 0.873).
- **Absolute floor**: `contrast(accent, plate) >= 3.3` on `#100F0F`, `#121212`, and
  `#1E1E1E` — the dark floor the accent palette already documents for itself.

### 6.3 `tests/ace/tui/visual/test_chip_frame_glyphs.py` (new)

A mechanical tofu audit for the two frame glyphs, reusing
`tests/ace/tui/visual/_glyph_audit.py` exactly as `test_tab_icon_glyphs.py` does
(`pytestmark = pytest.mark.visual`):

- every codepoint in `_FRAME_LEFT` / `_FRAME_RIGHT` is covered by a bundled font;
- each rasterizes to ink (`render_ink(glyph) != render_ink(" ")`).

Both glyphs were verified present in `FiraCode-Regular`, `FiraCode-Bold`, and
`DejaVuSans`; the audit is what keeps a future glyph swap from silently shipping a
`.notdef` box.

### 6.4 Visual golden

- Regenerate `tests/ace/tui/visual/snapshots/png/current_project_indicator_120x40.png`
  via `just test-visual` with `--sase-update-visual-snapshots` (requires
  `just install-visual`; the renderer fingerprint in `renderer_env.json` must match).
- **Open the regenerated PNG and look at it** before accepting: the rails must be crisp
  full-height strokes at the chip's outer edges, the plate a visible but quiet warm
  lift, and clear background must separate the box from the model readout and from `⚑1`.
- Re-run the whole visual suite and confirm no other golden moved. Every other top-bar
  golden has an empty chip, so any additional diff means the hidden state stopped being
  zero width — fix that rather than accepting the golden.

## 7. Verification

1. `just install` (ephemeral workspaces may have drifted deps).
2. `just check` — whole-repo lint gates plus the diff-scoped test lane. Pay attention to
   `_lint-symvision`: `project_chip_plate` must pass on its real consumer, with no
   `--epic-symbol` entry added to the `Justfile`.
3. `just test-visual` for the PNG suite (excluded from `just test`), including the
   golden review above.
4. `just check-full` before landing, run **only** through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …` with a `--next` action), never
   inline.

## 8. Alternatives considered and rejected

- **Real three-row Textual border** (`border: round <accent>`): the only literal
  four-sided box, but it forces `#top-bar` from 1 row to 3, costing every ACE tab two
  rows of vertical space permanently for one chip.
- **Overline/underline edges**: see §2 — invisible to the PNG golden corpus, uneven
  terminal support.
- **Filled pill** (accent background, `#1a1a1a` text) in the `_override_pill.py`
  grammar: handsome and already precedented, but it puts the project chip in the _same_
  shape category as the LLM override / alias / provider-disable pills, which is exactly
  the separation the request is asking for. Outline-vs-fill is the strongest
  differentiator available on one row, and it keeps the project accent as foreground,
  where the palette's luminance was tuned.
- **Half-block caps** (`▐…▌`) or **heavy verticals** (`┃`): prototyped; both are
  visually heavy, and `┃` renders shorter than the cell in Fira Code, leaving ragged
  gaps.
- **Rails hugging the text** (`▕+sase▏`): prototyped; reads as cramped brackets
  squeezing the name rather than as a container.
- **CSS `border-left`/`border-right: vkey <accent>` plus `background: $surface`**:
  closest to native Textual grammar, but a CSS border and background keep painting when
  the chip is hidden, turning today's zero-width empty state into a two-cell artifact,
  and the border color would need imperative per-project mutation anyway.
- **Hover lift on the plate**: attractive, but no other top-bar indicator has hover
  feedback, and the chip already answers hover with a tooltip. Uniformity wins.

## 9. Non-goals and follow-ups

- **Not** changing what the current project _is_, how it resolves, what it seeds, the
  tooltip copy, the click target, or the mount order.
- **Not** adding a config knob for the frame style; `ace.current_project.indicator`
  remains the only switch.
- **Not** adding truncation for long display names — the chip grows by two cells, and
  top bar overflow behaviour is unchanged from today.
- Possible follow-up (out of scope): reuse the same framed grammar for the
  current-project row in the `+` launch picker and the Glossary project ring, so one
  visual grammar means "current project" everywhere.
