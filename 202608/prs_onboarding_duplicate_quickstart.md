---
tier: tale
title: Fix the duplicated PRs-tab onboarding panel
goal:
  The PRs empty state renders exactly one centered onboarding panel whose keycap table
  stays aligned.
size: small
proposed_by: bbugyi200.athena.wx
create_time: 2026-08-10 09:18:02
status: wip
---

# Fix the duplicated PRs-tab onboarding panel

## Problem

On the **PRs** sub-tab of the **Artifacts** tab, when no Patch matches the current query
(or no Patches exist at all), the empty-state onboarding panel renders **twice**:

1. A first copy, correctly centered inside the detail container.
2. A second copy directly below it, left-aligned and flush against column 0 of the
   terminal, with its own scrollbar.

Because the two copies split the available vertical space (`1fr` each), both are
clipped: neither shows the whole "Start here" card, and the footer line ("Your agents'
PRs appear here as they work.") is never visible.

A second, independent defect is visible in the same view: in the "Start here" card, the
`#` row's description ("Open the SASE Admin Center: configure sase, install plugins, run
updates.") overflows the card's content width and wraps, and the wrapped remainder
("updates.") lands at **column 0** instead of aligning under the description column,
visually breaking the two-column keycap table.

Both defects are reproducible at `master` (`354d8c19f`). The Agents-tab onboarding is
_not_ affected by defect A and only marginally by defect B.

## Root cause

### Defect A — a legacy alias was implemented as a second mounted widget

`src/sase/ace/tui/widgets/artifacts/panes.py` (`ArtifactsPrsPane.compose`) yields
**two** `TabQuickStart` widgets into `#detail-container`:

```python
yield TabQuickStart(
    tab="artifacts",
    id="patch-quickstart-panel",
    classes="hidden",
)
yield TabQuickStart(
    tab="artifacts",
    selector_prefix="changespec",  # legacy compatibility alias
    id="changespec-quickstart-panel",  # legacy compatibility alias
    classes="hidden",
)
```

`PatchOnboardingMixin._sync_patches_onboarding`
(`src/sase/ace/tui/actions/patch/_onboarding.py`) then builds a `quickstarts` list
containing both panels and **removes the `hidden` class from both**, so both become
visible at the same time.

This is why the second copy is unstyled and left-aligned: `src/sase/ace/tui/styles.tcss`
only styles the canonical id —

```css
#patch-quickstart-panel {
  width: 100%;
  height: 1fr;
  padding: 1 2;
  align-horizontal: center;
  scrollbar-gutter: stable;
}
```

There is **no rule for `#changespec-quickstart-panel`**, so it falls back to the
`VerticalScroll` defaults: no `padding`, no `align-horizontal: center`, and a default
`1fr` height that steals half the container. Its `max-width: 90` children therefore
start at column 0 and the panel stacks below the styled one — exactly what the
screenshot shows.

The legacy ChangeSpec→Patch renames elsewhere in this codebase are handled the right
way: as **selector remaps in the test harness**, not as extra mounted widgets. See
`_LEGACY_SELECTOR_ALIASES` in `src/sase/ace/testing/ace_page.py`, which already maps
`#changespecs-view` → `#artifacts-view` and maps `#patch-quickstart-panel` to itself.
Nothing in `src/` or `tests/` references `#changespec-quickstart-panel` other than the
two sites above — it is a duplicate with no consumers.

The Agents tab confirms the intended shape: `src/sase/ace/tui/_app_layout.py` mounts
exactly one `TabQuickStart` (`#agent-quickstart-panel`), and its golden PNG shows a
single, correctly centered panel.

### Defect B — card rows wrap without a hanging indent

`TabQuickStart._build_card` / `_append_key_row` in
`src/sase/ace/tui/widgets/tab_quickstart.py` emit each row as
`<right-aligned keycaps><2 spaces><description>\n` into a single `rich.text.Text`, with
no wrapping of its own. Rich then soft-wraps at the widget width, and the continuation
starts at column 0 because there is no hanging indent.

The card's content width is fixed by CSS: `.tab-quickstart-card` is `max-width: 90` with
`border: round` (2 cols) and `padding: 0 1` (2 cols), so **86 columns** of content at
any terminal ≥ ~95 columns wide.

`column_width` is the width of the widest keycap group. On the **artifacts** tab the
`1 2 3 4 5` row makes it **19**, so the `#` row needs `19 + 2 + 73 = 94` columns — 8
over budget. On the **agents** tab there is no digits row, `column_width` is 11, and the
same description needs exactly `11 + 2 + 73 = 86` — it just fits, which is why only the
PRs card looks broken today.

Verified by rendering the card standalone at width 86; every other row is ≤ 80 columns
and only the `#` row measures 94.

### Defect C (minor, same function) — trailing blank line inside the card

`_append_key_row` appends `"\n"` after **every** row including the last, so the card
`Text` ends with a newline and Rich renders an empty final line inside the rounded
border. This is visible as an extra gap above the card's bottom border in both the
Agents and PRs onboarding goldens.

### Why the visual snapshot suite did not catch any of this

`tests/ace/tui/visual/test_ace_png_snapshots_changespecs_onboarding.py` covers exactly
these two states, but its assertions are `assert_page_svg_contains(...)` substring
checks, which pass happily when content appears twice. The accepted golden PNGs —
`tests/ace/tui/visual/snapshots/png/changespecs_onboarding_120x40.png` and
`changespecs_onboarding_no_match_120x40.png` — **already contain the duplicated panel
and the broken wrap**. The suite is currently ratifying the defect, so the goldens must
be re-recorded as part of this fix.

## Implementation

### 1. Delete the duplicate quickstart panel

In `src/sase/ace/tui/widgets/artifacts/panes.py`, `ArtifactsPrsPane.compose`: remove the
second `TabQuickStart(...)` yield (`#changespec-quickstart-panel`). Keep the canonical
`#patch-quickstart-panel` yield unchanged.

In `src/sase/ace/tui/actions/patch/_onboarding.py`,
`PatchOnboardingMixin._sync_patches_onboarding`: remove the `quickstarts` list and the
`try/except` that looks up `#changespec-quickstart-panel`. Operate directly on the
single `quickstart` widget for the `set_widget_hidden`, `set_no_match_context`,
`set_keymap_registry`, and `refresh_content` calls.

**Preserve legacy selector compatibility in the right place** instead: add
`"#changespec-quickstart-panel": "#patch-quickstart-panel"` to
`_LEGACY_SELECTOR_ALIASES` in `src/sase/ace/testing/ace_page.py`, matching the existing
`#changespecs-view` entry.

**Out of scope — do not touch:** the hidden `#changespecs-view` `Static` in
`src/sase/ace/tui/widgets/artifacts/view.py` and the `views` list handling in
`_sync_patches_onboarding` that applies `-onboarding-active` to it. That widget is empty
and hidden, renders nothing, and removing it is unrelated cleanup.

If `selector_prefix` on `TabQuickStart` ends up with no remaining callers after this
change, leave the parameter in place (`render_content` still accepts it and the legacy
`patches`/`changespecs` tab literals still flow through it); do not widen this change
into a `TabQuickStart` API refactor. If a lint gate such as Symvision flags a now-unused
symbol, prefer deleting only what the gate names.

### 2. Give card rows a hanging indent

In `src/sase/ace/tui/widgets/tab_quickstart.py`:

- Add module constants that document where the width comes from, e.g.:

  ```python
  # Keep in sync with `.tab-quickstart-card` in styles.tcss: max-width 90 minus
  # the round border (2) and `padding: 0 1` (2).
  _CARD_MAX_WIDTH = 90
  _CARD_CONTENT_WIDTH = _CARD_MAX_WIDTH - 4
  _MIN_DESCRIPTION_WIDTH = 24
  ```

- In `_append_key_row`, wrap `description` with `textwrap.wrap` at
  `max(_MIN_DESCRIPTION_WIDTH, _CARD_CONTENT_WIDTH - (column_width + 2))`, emit the
  first wrapped line after the keycaps as today, and prefix every continuation line with
  `" " * (column_width + 2)` so it aligns under the first description character.
  Preserve the existing right-alignment padding of the keycap column.

- Fix defect C: build the card so rows are separated by `"\n"` without a trailing
  newline (e.g. only append the separator before rows after the first), so no empty line
  renders inside the card border.

Keep `render_content`'s return type as `dict[str, Text]`. Do **not** convert the card to
a `rich.table.Table`: `tests/ace/tui/widgets/test_tab_quickstart.py` reads
`sections[selector].plain`, and a `Table` would break every one of those assertions.

Optionally tighten the `#` row's copy so it fits on one line at the default keymap
(needs ≤ 65 characters when `column_width` is 19). This is a judgment call, not a
requirement — the hanging indent already makes the wrapped form read correctly.

**Accepted limitation:** on terminals narrower than ~95 columns the card shrinks below
90 and Rich re-wraps the pre-wrapped lines back to column 0. Making the card
width-adaptive (measuring the mounted widget and re-rendering on resize) is deliberately
out of scope; note it in the commit message if you think it is worth a follow-up task
bead.

### 3. Tests

Add to `tests/ace/tui/test_changespecs_onboarding.py`:

- A test asserting the PRs empty state mounts exactly **one** quickstart panel — e.g.
  `len(page.query_widget("TabQuickStart"))` inside `#artifacts-view` is 1, or assert
  `page.app.query("#changespec-quickstart-panel")` is empty. This is the direct
  regression guard for defect A; make it fail against current `master` before you fix
  it.

Add to `tests/ace/tui/widgets/test_tab_quickstart.py`:

- Every line of the rendered card (both `tab="agents"` and `tab="patches"`) is
  `<= _CARD_CONTENT_WIDTH` columns, including under a keymap registry configured with
  deliberately long key names so `column_width` grows.
- A row whose description must wrap produces continuation lines indented to the
  description column (assert the continuation line starts with the expected number of
  leading spaces and that no non-blank card line starts at column 0 other than a keycap
  group at full `column_width`).
- The card text does not end with a newline (defect C).

### 4. Re-record the visual goldens

`_build_card` feeds both onboarding surfaces, so more than the PRs goldens will move.
After the code changes:

```bash
just test-visual                                   # observe which goldens fail
just test-visual --sase-update-visual-snapshots    # accept, after reviewing diffs
```

Inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/` before
accepting anything. Expect at least these to change:

- `tests/ace/tui/visual/snapshots/png/changespecs_onboarding_120x40.png`
- `tests/ace/tui/visual/snapshots/png/changespecs_onboarding_no_match_120x40.png`
- `tests/ace/tui/visual/snapshots/png/agents_onboarding_120x40.png` (trailing-blank-line
  removal, plus the `#` row if you tightened its copy)
- `tests/ace/tui/visual/snapshots/png/agents_onboarding_no_plugins_120x40.png` (same
  reason)

**Open the two re-recorded PRs goldens and confirm by eye** that they now show a single,
centered onboarding panel with an intact keycap table — do not accept them blind. The
whole point of this change is that the previous goldens baked in the bug.

Also strengthen the visual tests themselves so a future duplicate cannot be re-accepted
silently: in `tests/ace/tui/visual/test_ace_png_snapshots_changespecs_onboarding.py`,
add a non-substring assertion (for example, that the hero copy appears exactly once in
the captured SVG, or reuse the "exactly one quickstart panel" widget assertion) so the
suite fails on duplication rather than passing on `in`.

## Verification

```bash
just install     # workspaces are ephemeral; dependencies may be stale
just check       # whole-repo lint gates + diff-scoped tests
just test-visual # PNG snapshot suite (excluded from `just test`)
```

Because this change touches shared TUI rendering used by two tabs, finish with:

```bash
just check-full
```

Manual confirmation (optional but recommended): run `sase ace`, open **Artifacts →
PRs**, type a query that matches nothing, and confirm a single centered onboarding panel
with the full "Start here" card and the footer line all visible.

## Done when

- The PRs empty state renders exactly one onboarding panel, centered, with the whole
  card and the footer visible.
- No card row's wrapped continuation starts at column 0; the keycap table stays aligned.
- No blank line renders inside the "Start here" card border.
- `#changespec-quickstart-panel` no longer exists as a mounted widget, and the legacy
  selector still resolves through `_LEGACY_SELECTOR_ALIASES`.
- New regression tests fail on current `master` and pass after the fix.
- Re-recorded goldens visually confirmed; `just check-full` and `just test-visual` both
  pass.
