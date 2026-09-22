---
tier: tale
title: Give the top-bar updates badge its own visual language
goal:
  The ACE top-bar updates badge (routine, sase-core rebuild, and agent-CLI states) is
  unmistakable next to every neighboring indicator chip, including under red-green
  color-vision deficiency, and the badge, the ,U Update panel, and the startup update
  toast share one glyph and one palette.
size: medium
proposed_by: bbugyi200.apollo.1h.f0.f0.f0.w2.w0
create_time: 2026-09-22 15:15:57
status: wip
---

# Plan: Give the top-bar updates badge its own visual language

## Product context

The updates badge in ACE's top-bar indicator row (`UpdatesAvailableIndicator`, mounted
as `#updates-indicator`) never had a color of its own. Each of its three states reuses
the exact hex of a chip that sits right next to it:

| Badge state         | Today's background | Collides with (top-bar neighbor)                                                                                       | WCAG contrast vs that neighbor |
| ------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| routine `↑ N`       | `#AF87FF` violet   | alias-override pill `ALIAS_LANE_PALETTE` `#AF87FF` (the **immediate right** neighbor)                                  | 1.00 (identical)               |
| core `↑ N *`        | `#FFAF5F` orange   | monitor gear chip `MONITOR_GEAR_HUE` `#FFAF5F` (the **immediate left** neighbor), provider hard-disable pill `#FFAF5F` | 1.00 (identical)               |
| agent CLI `CLI ↑ N` | `#00D7FF` cyan     | proc gear chip `PROC_GEAR_HUE` `#48CAE4`, provider-priority pill `#87D7FF`                                             | 1.05–1.12                      |

With neighbors present, the chips merge into one strip: `⚙ 1` and `↑ 3 *` read as a
single orange block, and `↑ 3` runs straight into `@large 2h`. The core marker itself is
a bare `*`, a weak glyph that only reads as "core" if you already know what it means.

How this happened: violet was the Updates identity color (it matches the Admin Center
Updates tab). Plan `plan:202607/distinct_update_stash_badges.md` moved the stash badge
to teal to get away from it. The alias-override pill later adopted the same violet. The
core state (plan `plan:202607/updates_badge_core_flair.md`) picked amber, which the
monitor chip also uses. Each choice made sense locally. Together they left the badge
with no color of its own on this row.

The user asked for a better-looking up-arrow (or a better icon), and in particular for a
core-update color that doesn't clash with that row. The goals are intuitive, reliable,
and beautiful.

Design mock (every badge state next to the real neighbor chips, before vs after, in
normal vision and simulated deuteranopia): `file:explicit:388aa002807de649e2ca3bbd`.
Open it with `sase artifact open file:explicit:388aa002807de649e2ca3bbd`.

## Design

### Principle 1 — the badge is the top bar's only _deep_ chip

Every other filled chip on this row is a **bright** background with dark `#1a1a1a` ink:
cyan, orange, violet, yellow, light blue, teal. That part of color space is full. An
exhaustive search of the xterm-256 cube found no bright color that stays at least 30 ΔE
from every neighbor under red-green color-vision deficiency. The best candidates came
out around 25, and they were garish yellows.

So the badge inverts the chip grammar: a **deep moss surface with bright lime ink**. It
is told apart from its neighbors by lightness, not hue, so it stays distinct in every
color-vision condition:

- WCAG 2.1 SC 1.4.11 non-text contrast against every neighbor chip background is at
  least **3.75:1** (vs the violet alias pill). Every old badge background scores
  1.00–1.12.
- CIE76 ΔE is at least 46 against every neighbor, in normal vision and in simulated
  deuteranopia, protanopia, and tritanopia. The bright green first considered
  (`#87D75F`) fell to ΔE 7 against the adjacent orange monitor chip under deuteranopia,
  which is why it was rejected.
- It looks calm: a dark, glowing chip that says "new versions are ready" without
  competing with the activity chips (procs, monitors, overrides) for attention.
- In 256-color terminals Rich downgrades the surface to xterm 22 (`#005F00`). That is
  still a deep green chip, and lime ink on it is still 6.6:1.

### Principle 2 — hue is identity; state lives inside the chip

The old badge changed its whole background to show state (violet → amber), which is how
it took on its neighbors' colors. The new badge **never changes hue**. The core state
adds a word tag inside the same chip, following the top bar's existing two-part pill
grammar (`_override_pill.py`: primary subject + secondary trailing):

- **Core rebuild = a bright lime inset tag reading `core`** (dark ink on lime, the
  identity accent filled in). It is the loudest part of the badge. The earlier plan
  argued the rebuild cost is "the single most decision-relevant fact about a pending
  update", so it should be. The word needs no legend. It survives a monochrome terminal
  and needs no special font. The tag stands out from its own surface in every
  color-vision condition (ΔE ≈ 70).
- **Agent CLIs = the same moss surface with sage ink.** The badge stays one object (one
  click target that opens one Updates tab). The two domains differ by label and ink
  tone, not by borrowing another hue.

### Principle 3 — a heavier, still-reliable arrow: `⬆`

The glyph keeps the "up = upgrade" direction, because sase already uses that meaning:
`↑` marks update actions and `↓` marks installs (plugins browser, confirm modals). The
VCS log uses `↑ ahead / ↓ behind`, and chat provenance uses `↓ remote`. The other
candidates were rejected:

- down arrows / `⤓` / `⇣`: these mean "install" or "remote" elsewhere in sase. `⤓` and
  `⭳` are also missing from the bundled Fira Code / DejaVu faces, so they would render
  as tofu in the goldens.
- `⟳` / `↻`: read as "in progress", and `↻` already marks the Update panel's Restart-ACE
  row.
- `★`: provider priority. `⚙`: procs and monitors. `▲`: reads as "play" or "raised".
  `⇧`: the Shift key.

`⬆` (U+2B06, UPWARDS BLACK ARROW) is a solid pictogram, so it matches its neighbors
`⚙ ❄ ⚑ ✖` instead of looking like punctuation. It is in both bundled Fira Code
weights, and it is East-Asian-Width Neutral, so it is always one cell. `↑` is Ambiguous
and can render two cells wide in CJK-wide terminals. On this host, 4 font families cover
`⬆`, which is as many or more than cover the row's existing `⚙` (2), `❄` (3), `✖`
(3), and `⚑` (2). The row already relies on that class of glyph, so `⬆` adds no new
rendering risk.

### Palette (single source of truth: `src/sase/ace/tui/widgets/update_accents.py`)

| Name                    | Value     | Role                                                                                                                                                           |
| ----------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `UPDATE_GLYPH`          | `⬆`      | badge glyph, Update panel title + chips, startup-toast title + section headers                                                                                 |
| `UPDATES_SURFACE`       | `#244A14` | deep moss chip background (badge only)                                                                                                                         |
| `UPDATES_ACCENT`        | `#AFFF87` | lime identity ink: badge glyph/count, core-tag fill, Update panel SASE row, startup-toast accent                                                               |
| `AGENT_CLI_ACCENT`      | `#AFD7AF` | sage ink: badge CLI segment, Update panel providers row, startup-toast CLI lines                                                                               |
| `CORE_TAG_LABEL`        | `core`    | the inset tag's word                                                                                                                                           |
| `CORE_TAG_INK`          | `#1a1a1a` | dark ink on the lime tag (the top bar's existing chip-ink convention)                                                                                          |
| `UPDATE_CAUTION_ACCENT` | `#FFAF5F` | Update panel caution chrome only — capital apply-now keys, `⚡ apply now` hint, stale subtitle/chip, restart row. **Value unchanged**; it gets an honest name. |

Contrast: lime on moss is 8.46:1, sage on moss is 6.38:1, and dark ink on the lime tag
is 14.48:1. All meet WCAG AA for text. Lime text on the panel's dark background is about
13.9:1.

`CORE_UPDATE_ACCENT` is **removed**. It was doing two unrelated jobs: the core badge
fill and the Update panel's caution chrome. Its caution uses move to
`UPDATE_CAUTION_ACCENT`. Its core use is replaced by the tag. Do not keep a
compatibility alias; update every importer.

Add one shared builder to `update_accents.py` so every surface renders the tag the same
way:

```python
def build_core_tag() -> Text:
    """The inset ``core`` tag marking a pending sase-core Rust rebuild."""
    return Text(f" {CORE_TAG_LABEL} ", style=f"bold {CORE_TAG_INK} on {UPDATES_ACCENT}")
```

Rewrite the module docstring so it explains the design: a shared update language across
the badge, the `,U` Update panel, and the startup toast; the only deep chip on the top
bar; hue as identity with state carried by the tag; and why `⬆`. Update `__all__`.

### Exact badge grammar (`UpdatesAvailableIndicator._build_content`)

| State          | Call                                | `.plain`                   | Spans                                                         |
| -------------- | ----------------------------------- | -------------------------- | ------------------------------------------------------------- |
| hidden         | `(0)`                               | `""`                       | —                                                             |
| routine        | `(3)`                               | `" ⬆ 3 "`                 | `bold #AFFF87 on #244A14`                                     |
| core rebuild   | `(3, core=True)`                    | `" ⬆ 3  core "`           | `" ⬆ 3 "` identity, then `build_core_tag()`                  |
| agent CLI only | `(0, agent_cli_count=2)`            | `" CLI ⬆ 2 "`             | `bold #AFD7AF on #244A14`                                     |
| mixed          | `(3, agent_cli_count=2)`            | `" ⬆ 3  CLI ⬆ 2 "`       | identity segment, then CLI segment (each has its own padding) |
| mixed + core   | `(3, core=True, agent_cli_count=2)` | `" ⬆ 3  core  CLI ⬆ 2 "` | identity, tag, CLI                                            |

- Delete `_CORE_UPDATE_GLYPH = "*"`. The CLI segment always has its own leading pad. The
  old flush `CLI…` join (`prefix = "" if count > 0`) goes away, so each segment reads as
  its own part of the chip.
- The tag only appears when `count > 0` (this is already true: `set_available` clamps
  `core = bool(core and count > 0)`; keep that).
- Leave the tooltip text, `set_available` semantics, the equal-state early return, the
  click behavior, and all properties unchanged. Update the class docstring to describe
  the grammar.
- Width cost: routine and CLI-only are unchanged (5 and 9 cells). Core is +4 cells and
  mixed + core is +5 cells. That is an accepted trade-off for a self-explaining word
  instead of `*`.

### Update panel (`,U`) — same object, same colors

The panel's SASE and providers rows correspond to the badge's two segments, so they use
the same language:

- `src/sase/ace/tui/update_panel_state.py`
  - Add `core_rebuild: bool = False` to `UpdateOptionChip` (frozen/slots dataclass; the
    default keeps existing keyword constructors working). Thread it through `_row(...)`
    and `_chip(...)` as a keyword.
  - `_sase_row`: the accent is **always** `UPDATES_ACCENT`. Remove the
    `has_core_update → CORE_UPDATE_ACCENT` swap. Set
    `core_rebuild = kind == "available" and status.has_core_update`.
  - `_everything_row`:
    `core_rebuild = kind == "available" and sase_row.chip.core_rebuild`.
  - `_providers_row`: `AGENT_CLI_ACCENT` (the new sage value).
  - `_restart_row`: `UPDATE_CAUTION_ACCENT` (same orange as today), glyph `↻` unchanged.
  - Chip text becomes `f"{UPDATE_GLYPH} {count} available"` → `"⬆ 4 available"` through
    the shared constant; no local literal.
- `src/sase/ace/tui/modals/update_panel.py`
  - Replace every `CORE_UPDATE_ACCENT` (uppercase key letter, stale border subtitle,
    stale chip style, the `E S P` / `⚡ apply now · no prompt` hints) with
    `UPDATE_CAUTION_ACCENT`. These render exactly as before.
  - `_row_prompt`: after building `chip`, if `row.chip.core_rebuild`, append `" "` and
    then `build_core_tag()` **before** the right-alignment `gap` is computed, so the row
    stays aligned.
  - The title becomes `⬆ Update` through `UPDATE_GLYPH`.

### Startup "Updates available" toast — the badge's short-lived twin

`src/sase/ace/tui/actions/_update_toast_message.py` currently has its own `↑` and
`#00D7FF` literals, and takes its accent from
`center_tab_accent("updates") or "#AF87FF"`. Once the badge changes, the toast's title
(which already uses the badge glyph path) would disagree with its own body. Fix this:

- Delete the local `_UPDATE_GLYPH` and `_AGENT_CLI_ACCENT`. Import `UPDATE_GLYPH`,
  `UPDATES_ACCENT`, and `AGENT_CLI_ACCENT` from `..widgets.update_accents`.
- `accent = UPDATES_ACCENT` (section headers `⬆ sase …`, the `N updates` headline, and
  `,U`). CLI lines use `AGENT_CLI_ACCENT`. Drop the `center_tab_accent` import if
  nothing else in the module uses it.
- `src/sase/ace/tui/actions/update_toast.py`: build `_TOAST_TITLE` from
  `update_accents.UPDATE_GLYPH` and stop re-exporting `_UPDATE_GLYPH` from
  `_update_toast_message`. Grep for importers of `update_toast._UPDATE_GLYPH` first.
  Only the re-export line exists today.

### Deliberately out of scope

- The **post-update receipt toast** (`post_update_toast.py`, `✓ SASE updated` with
  `↑ label — N commits`). It is a completion receipt with its own success styling, not
  an availability signal.
- The **Admin Center Updates tab accent** (`#AF87FF` in `config_center_catalog.py`) and
  `agent_onboarding.py`'s navigation-colored "Updates" text. That tab strip has its own
  7-color palette, where green already belongs to the Procs tab. The top bar and the tab
  strip keep separate palettes. The badge still opens the tab on click, and the tooltip
  names it.
- Row markers and CLI output: `sase.plugins.render_common` / `render_catalog`,
  `sase.main.update_render`, the plugins browser `↑` markers, and the confirm-modal `↑`
  icons. These are item-level "has an update" markers in dense text, where the thin
  arrow reads well.
- Other top-bar pairs that already share a color (monitor chip and provider hard-disable
  pill are both `#FFAF5F`). This plan only frees the updates badge.
- No config keys, keymaps, or `default_config.yml` changes. No update-detection,
  counting, or cache behavior changes. No `sase-core` changes: this is purely
  presentation code, which belongs in this repo under the Rust-core boundary rule.

## Implementation steps

1. **`src/sase/ace/tui/widgets/update_accents.py`**: new palette, docstring,
   `build_core_tag()`, `__all__` (see the palette table). Remove `CORE_UPDATE_ACCENT`.
2. **`src/sase/ace/tui/widgets/updates_indicator.py`**: implement the badge grammar
   table above. Import the new names. Delete `_CORE_UPDATE_GLYPH`. Update the
   docstrings.
3. **`src/sase/ace/tui/update_panel_state.py`** and
   **`src/sase/ace/tui/modals/update_panel.py`**: the Update panel changes above.
4. **`src/sase/ace/tui/actions/_update_toast_message.py`** and
   **`src/sase/ace/tui/actions/update_toast.py`**: the toast changes above.
5. **Docs**: rewrite the badge paragraph in `docs/configuration.md` (currently "The
   persistent top-bar badge uses separate joined segments: purple `↑ N` … amber `↑ N *`
   … cyan `CLI ↑ N` …"). New content: a deep moss chip with lime `⬆ N` for SASE/plugin
   updates; a bright lime `core` tag when sase-core needs a Rust rebuild; a sage
   `CLI ⬆ N` segment for agent CLIs; the chip is the top bar's only dark chip so it
   never blends with neighbors; the tooltip and click behavior as before. Grep `docs/`
   for other descriptions of the badge, Update panel, or startup-toast glyph or colors
   (`↑ N`, `↑ Update`, `↑ Updates available`, "purple"/"amber"/"cyan" next to updates)
   and update only the ones that describe these three surfaces.
6. Run
   `rg -n "CORE_UPDATE_ACCENT|_CORE_UPDATE_GLYPH|\"↑ Updates available\"" src tests docs`.
   Nothing should remain. Also re-grep `#AF87FF`/`#00D7FF` in the three surfaces'
   modules and tests.

## Tests

- **`tests/test_updates_indicator.py`**: update every pinned `.plain` string to the
  grammar table. Replace hex-literal asserts with the `update_accents` constants: the
  identity span is `bold {UPDATES_ACCENT} on {UPDATES_SURFACE}`, the CLI span is
  `bold {AGENT_CLI_ACCENT} on {UPDATES_SURFACE}`, and the tag span equals
  `build_core_tag()`'s style. Add cases for: the tag appears only with `core=True` and
  `count > 0`; `set_available(0, core=True, agent_cli_count=2)` shows no tag;
  `set_available(2)` → `set_available(2, core=True)` still re-renders (the existing
  early-return test) and now contains `core`. Keep the no-I/O render test and update its
  expected string to `" ⬆ 2  core  CLI ⬆ 1 "`.
- **New `tests/ace/tui/test_top_bar_palette.py`**: a guard so the neighbor-color
  regression can't come back. Use local `_relative_luminance` / `_contrast_ratio`
  helpers, following `tests/ace/tui/test_artifacts_provider_palette.py`.
  - Collect the neighbors' **rendered** backgrounds from their real builders:
    `gear_chip(1, PROC_GEAR_HUE)`, `gear_chip(1, MONITOR_GEAR_HUE)`
    (`sase.ace.tui.proc_gear_chips`), `StashedPromptsIndicator._build_content(1)`, and
    `build_override_pill(...)` for `ALIAS_LANE_PALETTE`, `PROVIDER_DISABLE_PALETTE`,
    `PROVIDER_SOFT_DISABLE_PALETTE`, and `PROVIDER_PRIORITY_PALETTE`
    (`widgets/_override_pill.py`). Read `bgcolor` from each span's parsed `Style`.
  - Render `UpdatesAvailableIndicator._build_content(3, core=True, agent_cli_count=2)`.
    Assert that the chip **surface** background has WCAG contrast `>= 3.0` against every
    neighbor background. Cite SC 1.4.11 in the docstring and note that luminance
    contrast is what keeps the chip distinct under red-green color-vision deficiency.
    Expected minimum today is 3.75.
  - Assert every foreground/background pair in the badge's spans has contrast `>= 4.5`
    (WCAG AA text).
  - Add a clear failure message naming the offending neighbor.
- **`tests/ace/tui/test_update_panel_state.py`**: the SASE row accent is
  `UPDATES_ACCENT` with and without a core update. `chip.core_rebuild` is set on the
  SASE and Everything rows only when `has_core_update` and the kind is `available`, and
  it is never set on providers or restart. The restart row accent is
  `UPDATE_CAUTION_ACCENT`. Chip texts use `⬆`.
- **`tests/ace/tui/test_update_panel.py`**: switch the `CORE_UPDATE_ACCENT` asserts
  (capital-key styles, stale subtitle) to `UPDATE_CAUTION_ACCENT`. Update the `↑`
  strings, including the `"⬆ Update"` border title. Add a test that a
  `core_rebuild=True` row prompt contains the tag with `build_core_tag()`'s style and
  that the chip remains right-aligned (its last cell ends at the same column as a
  non-core row's chip).
- **`tests/ace/tui/test_update_toast_message.py`** and
  **`tests/ace/tui/test_update_toast_startup.py`**: update the `↑` strings to `⬆`
  (`"⬆ Updates available"`, `"⬆ sase"`, `"⬆ CLI Claude Code"`). Assert the lime
  accent and the sage CLI accent through the constants.
- **Visual tests**:
  - `tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`: update each
    `wait_for_state` plain string and the docstrings ("purple", "amber", "cyan" → the
    new language).
  - `tests/ace/tui/visual/test_ace_png_snapshots_update_panel.py`: update the imports
    and fixture accents (`CORE_UPDATE_ACCENT` → `UPDATES_ACCENT` for the SASE row, and
    set `core_rebuild=True` on the SASE/Everything fixture chips where the fixture
    models a core update).
  - **Add one neighbor-context PNG snapshot** to the updates-indicator visual file:
    `updates_indicator_with_neighbors_120x40`, showing the badge in the mixed + core
    state next to live proc and monitor gear chips (`ProcIndicator.set_count`,
    `MonitorIndicator.set_count`) and an alias-override pill (monkeypatch
    `alias_overrides_indicator.get_active_alias_overrides` the way
    `tests/ace/tui/test_top_bar_order.py` does, then refresh the widget). This is the
    exact scenario the user reported. If background polling resets the proc or monitor
    counts in the harness, gate on them with `wait_for_state`. If they still can't be
    held steady, drop only those two chips and keep the alias pill neighbor. Record
    which option you chose.

## Verification

1. `just install` (the ephemeral workspace venv may be stale).
2. `just fix` (or at least `just fmt`), then `sase tool run check` (or `just check`). It
   must pass, including symvision for the removed and added exports.
3. PNG goldens (follow the `lint_and_test` and `tui_screenshot` memories): run
   `just fix-tui-screenshots -- <selectors>` for
   `tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`,
   `tests/ace/tui/visual/test_ace_png_snapshots_update_panel.py`, and
   `tests/ace/tui/visual/test_ace_png_snapshots_update_toast.py`, through
   `/sase_monitor` if it runs long. Then run the full check form (`just test-visual`)
   through `/sase_monitor` to find any other golden that shows the badge, the Update
   panel, or the startup toast (for example `update_pinned_stash_preview_120x40`), and
   update those through targeted selectors.
4. **Inspect the report and every changed golden. Generating a golden is not approving
   it.** Expected changes: the badge (moss chip, lime `⬆`, lime `core` tag, sage CLI
   segment), the Update panel (`⬆` title and chips, lime SASE row with the `core` tag,
   sage providers row, orange caution chrome unchanged), and the startup toast (`⬆`,
   lime/sage). Any other pixel change is a regression to investigate, not to accept.
   Confirm by eye that `⬆` is one cell wide and that no chip boundary merges with a
   neighbor.
5. Optional live check: `sase screenshot -o /tmp/updates_badge.png` on a checkout with
   pending updates. Live PNGs are evidence, not goldens.

## Acceptance criteria

- In every state, the badge background differs from every top-bar neighbor chip by at
  least 3:1 luminance contrast, enforced by the new guard test.
- A core rebuild is shown by a readable `core` tag rather than a color swap or `*`. The
  badge hue never changes with state.
- The badge, Update panel, and startup toast use one glyph (`⬆`) and one palette from
  `update_accents.py`, with no duplicated literals left in those modules.
- The Update panel's caution chrome looks exactly as it does today.
- `just check` passes. The updated goldens were inspected and contain only the intended
  changes.
