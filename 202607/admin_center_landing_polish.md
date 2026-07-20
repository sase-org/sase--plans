---
tier: tale
title: Beautiful SASE Admin Center landing page
goal: 'The Admin Center landing page becomes a visually polished, optically balanced
  hero view — one centered axis, a framed sections card with hover-highlighted clickable
  rows, and a responsive compact layout — without giving up any of the fast, pure,
  zero-IO open-path guarantees.

  '
create_time: 2026-07-20 13:02:37
status: wip
prompt: 202607/prompts/admin_center_landing_polish.md
---

# Plan: Beautiful SASE Admin Center landing page

## Context and motivation

The home-first Admin Center landing shipped recently (`605fbebd1`) and is fast, but the rendered page is visually weak.
The current 120x40 and 100x24 PNG goldens (`tests/ace/tui/visual/snapshots/png/config_center_home_*.png`) show these
concrete defects:

1. **Orphaned lead.** "Choose a section" renders in the shared caption slot directly under the tab strip, ten-plus rows
   away from the menu it introduces. The header and the body read as two unrelated fragments.
2. **Dead void.** At 120x40 the content block floats in a featureless gap below the divider with nothing framing it; the
   page looks unfinished rather than intentionally spacious.
3. **Mixed alignment axes.** The orientation and instruction lines are text-centered across the full panel width while
   the fixed-width menu block sits flush left, so the eye has to jump between two different horizontal centers.
4. **No grouping or breathing room.** Four separate text bands (orientation, instruction, seven dense menu rows, footer)
   stack with uneven spacing; the footer hint is crammed directly under the last row.
5. **Weak affordances.** The single reversed digit chips are cramped, and the seven large menu rows — the visually
   dominant targets on the page — do nothing when clicked; mouse users must find the small tab strip instead.

This plan is a presentation-layer redesign of that landing view. All navigation semantics, canonical tab descriptions,
lazy pane loading, and performance guarantees from the previous plan remain in force.

## Design specification

### Page structure

The header keeps its identity: aurora gradient title, gradient underline, and the numbered clickable tab strip with no
active tab while home is shown. Two changes re-anchor the hierarchy:

- The caption slot under the tab strip (which shows `› <description>` for a real tab) shows the muted orientation
  sentence — "Configure, observe, and maintain SASE from one place." — while home is visible. It reads as the panel
  subtitle, adjacent to the header it belongs to.
- The lead, menu, and hints all move into one centered hero group in the body, on a single shared vertical axis:
  1. **Lead** — "Choose a section", bold, rendered with the same per-character aurora gradient as the title so the body
     visually rhymes with the header identity.
  2. **Sections card** — an auto-width, horizontally centered card with a `round` border in a quiet gray (the house card
     style; e.g. the `#5F5F87` / `#444444` family) so the seven tab accent colors remain the color story. The card
     contains exactly one row per catalog spec:
     - a three-cell key chip `1` in `bold reverse <accent>`;
     - a two-space gutter, then the tab label in bold accent, padded to a common column width;
     - a gutter, then the canonical description in the existing muted body color (`#C6C6C6`).
  3. **Hint line** — one consolidated, muted, centered line below the card replacing today's separate bold instruction
     band and footer band, e.g.: "Press 1-7 or click a section · Tab/Shift+Tab cycle · q/Esc close". Fold the previous
     `_HOME_INSTRUCTION` and `_HOME_FOOTER` copy into this single line (exact final wording may be tuned during
     implementation, but it must mention digits, clicking, Tab cycling, and q/Esc close).

The group is vertically centered in the body region with a slight upward optical bias (content centered one to two rows
above geometric middle reads better than true geometric centering). Generous-but-deliberate whitespace above and below
the card is the design at 120x40; the card frame is what turns that whitespace from "void" into "composition".

### Interactive rows

Each of the seven rows becomes its own click-target widget (not merely lines inside one Static):

- Clicking anywhere on a row opens that tab through the exact same `_schedule_switch` path used by digits, Tab cycling,
  and the tab strip, inheriting the navigation lock and idempotency.
- Rows get a Textual `:hover` style — a subtle background boost and a brightened description — so the page communicates
  "these are buttons" the moment the pointer touches them. This is the first `:hover` usage in `styles.tcss`; keep it
  subtle and consistent with the dark surface.
- Rows remain keyboard-transparent: `can_focus = False` everywhere on the landing, digits and modal-priority bindings
  keep working immediately, and nothing on home can steal focus.

### Responsive compact mode

The landing container toggles a `-compact` CSS class from its own synchronous `on_resize` handler (same pure pattern
`PanelTabStrip` already uses for width-compaction; no timers, no workers). Thresholds are chosen so 120x40 renders the
roomy layout and 100x24 the compact one:

- **Roomy** (tall/wide bodies): full card border, interior padding, blank-line gaps between the lead, card, and hint
  line.
- **Compact** (short or narrow bodies, e.g. 100x24): drop the card border and interior padding, tighten the key chip to
  its single-character form, and collapse the gaps as needed. The full content — lead, all seven rows with complete
  descriptions, hint line — must remain visible with no horizontal clipping, no overlap with the header, and no
  scrollbar at 100x24.

Width arithmetic to respect (95% panel width, thick border, `1 2` padding): the roomy row (3-cell chip + gutters +
10-char label column + 73-char longest description + card border and padding) fits the ~108-column inner width at 120
cols but not the ~89-column inner width at 100 cols — which is exactly why compact mode sheds the card chrome instead of
truncating copy. The body height at 100x24 is roughly 11 rows, so compact must budget lead + seven rows + hint plus at
most two gap rows; shed the gaps first if the fit is off by one.

### What does not change

- The immutable `_TAB_SPECS` catalog stays the single source of truth for number, label, accent, and description; the
  redesigned rows must derive from it so copy and navigation cannot drift.
- Canonical tab descriptions, tab order, `1`-`7` bindings, Tab/Shift+Tab semantics, out-of-range digit swallowing,
  `q`/`Esc` close, direct-entry `initial_tab` behavior, and generic-reopen- shows-home behavior are all untouched.
- The landing render path stays pure and bounded: no filesystem access, config lookups, workers, timers, subprocesses,
  or async callbacks. Resize handling must be synchronous and cheap per the TUI performance rules (no work on the pump
  beyond class toggling and a repaint).
- The lazy pane lifecycle, pane caching, and failure-recovery paths are untouched.

## Implementation notes

- `src/sase/ace/tui/modals/config_center_modal.py`: rework `_AdminCenterLanding` into the hero group (lead Static, a
  card container composed of seven row widgets, hint Static). Replace `_landing_menu_text()` with per-row rendering
  derived from `_CenterTabSpec`. Add the row widget class with `on_click` posting/scheduling the tab switch and
  `can_focus = False`. Update `_sync_header` so the home caption is the orientation sentence; retire
  `_home_lead_text()`'s caption-slot role (the gradient lead moves into the landing body). Update the `_HOME_*`
  constants to match the new copy set (lead, orientation, single hint line).
- `src/sase/ace/tui/styles.tcss`: replace the `#admin-center-home-*` block with the hero-group layout (centered
  alignment, card border/padding, row and `:hover` styles, `-compact` variants).
- Keep IDs stable where reasonable (`admin-center-home`, menu/row IDs may change to reflect the new structure) and
  update the tests that import the `_HOME_*` constants and query these IDs.
- If the row-widget class or any new helper trips Symvision (as `_CenterTabSpec` did last time), follow the Symvision
  memory guidance rather than ad-hoc pragmas.
- `docs/ace.md` / `docs/configuration.md`: the behavior descriptions remain accurate; adjust only if either quotes
  landing copy that changed.

## Verification

Run `just install` first (ephemeral workspace), then:

1. **Behavioral tests** (`tests/ace/tui/test_config_center_tabs.py`): update structure/copy assertions for the new hero
   layout; keep the structural home guarantees (no concrete panes, no loaders/workers/timers, home on generic reopen).
   Add coverage for:
   - clicking a landing row mounts exactly that pane and focuses it (pilot click on the row);
   - the landing exposes no focusable widgets and digits still work immediately;
   - the compact class toggles with viewport size (drive resize through the pilot at roomy and compact geometries).
2. **Visual snapshots**: re-capture `config_center_home_120x40` and `config_center_home_100x24` via `just test-visual`
   with `--sase-update-visual-snapshots`, then **visually inspect the rendered PNGs** — not just equality — for: single
   shared axis, optical centering, card framing, chip/label/description alignment, accent contrast on the dark surface,
   full copy visibility, and the compact fit at 100x24 (no clipping, no scrollbar, no header overlap). Iterate on the
   design until the page genuinely looks composed; do not accept the first render that merely passes. Other Admin Center
   goldens must remain byte-stable (this change must not touch real-tab rendering).
3. **Performance**: run the existing slow benchmark (`pytest -s -m slow tests/ace/tui/bench_admin_center_open.py`) and
   confirm open latency is in the same band as the current home view (the hero group adds a handful of static widgets;
   there is no data-scaled or IO work to regress). Report the numbers.
4. **Full checks**: the complete visual suite, then the required `just check`.
