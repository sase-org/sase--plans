---
tier: tale
title: Top-bar icon chips, a pink stash chip, and priority/disabled routing groups
goal:
  The ACE top bar shows gear icons on the procs and monitors chips, a `≡` stash icon on
  a visually distinct orchid-pink prompts chip, and labels provider routing truthfully
  with separate `priority:` and `disabled:` groups rendered from one routing snapshot.
size: medium
proposed_by: bbugyi200.athena.0q9.f2
create_time: 2026-09-23 16:13:10
status: wip
---

# Top-bar follow-up: icon chips, a pink stash chip, and `priority:` / `disabled:` groups

## Goal

This follows up the approved labeled top-bar cluster
(`plan:202609/labeled_top_bar_indicators.md`, landed as
`feat(ace): labeled dot-separated top-bar indicator cluster`). The user asked for three
changes to the right-aligned indicator cluster in sase's TUI top bar:

1. Put the little gear icon back on the procs indicator.
2. Give the prompt stash chip an icon again, but a better one than `❄` (it looks too
   much like the `⚙` gear). Also switch its icon and count to a more visually distinct
   color.
3. Label the provider group `disabled:` instead of `provider:`.

Before and after, busy state, full density:

```text
before: procs:  2  · monitors:  1  · updates:  3  · overrides:  @medium@max ∞  · provider:  CODEX ★ 42m +1  · prompts:  4  · inbox: ⚑1 ?5
after:  procs:  ⚙ 2  · monitors:  ⚙ 1  · updates:  3  · overrides:  @medium@max ∞  · priority:  CODEX ★ 42m  · disabled:  CLAUDE off 42m  · prompts:  ≡ 4  · inbox: ⚑1 ?5
```

This is presentation-only Textual work. It does not touch the Rust core boundary, config
schema, keymaps, or `src/sase/default_config.yml`. The labeled-group grammar, dim `·`
separators, density rule, and click behavior from the previous plan all stay as they
are.

## Design

### 1. Identity icons live inside count chips

The rule: a count chip carries its identity glyph inside its fill, and the dim label
names the group in words. The two are redundant on purpose. When space runs out and the
cluster goes compact, every label drops but the chips stay, so the icon is what still
identifies each group:

```text
full:     procs:  ⚙ 2  · monitors:  ⚙ 1  · … · prompts:  ≡ 4  · inbox: ⚑1 ?5
compact:   ⚙ 2  ·  ⚙ 1  · … ·  ≡ 4  · ⚑1 ?5
```

**Procs and monitors both get the gear back.** The user named procs; bringing monitors
along is a deliberate design call:

- Both groups click through to the Admin Center Procs tab. That tab's header already
  renders these same two chips with `proc_gear_chips.gear_chip` (blue for procs, amber
  for monitors). Reusing that builder makes the top bar and its click target
  byte-identical by construction.
- `⚙` is sase's monitor glyph everywhere else (agent-list rows, tribe panel titles). A
  gear on procs but not on monitors would make the monitor chip the odd one out.
- Hue, not glyph, tells the proc lane from the monitor lane, exactly as in the Procs
  tab.

`updates` stays glyph-free. It was not requested, and it belongs to a different chip
family (the segmented lime-on-moss badge).

### 2. Stash icon: `≡` (a stack)

- **Meaning.** The stash is a stack of set-aside prompt drafts (git-stash for prompts).
  `≡` reads as stacked layers, a pile of prompts.
- **Shape.** Flat horizontal bars are the opposite of the round, radial `⚙` and `❄`, so
  the procs and prompts chips cannot be mistaken for each other, even in compact mode.
- **Reliable rendering.** `≡` (U+2261) is single-cell and has no emoji presentation. It
  is in Fira Code itself, so it renders crisp and bold in the PNG renderer and in nearly
  every terminal font.
- **Alternatives prototyped in the project's own renderer and rejected:**
  - `❐` / `❏` (stacked sheets) render only through the DejaVu fallback. They come out
    thin and read as a checkbox at chip size.
  - Fira draws `☰` (trigram) wider than one cell, so the chip overflows its box.
  - `✎` already means "editing target" in the prompt bar and in plan refs.
  - `▤` already means files and plans in Artifacts.
  - `¶`, `⇊`, and `◩` are the right width but do not read as "stash".
- **Accepted cost.** The agent list marks workflow rows with `≡`. That is a different
  surface. On the top bar, the `prompts:` label, the pink fill, and the tooltip
  (`4 stashed prompts`) remove any ambiguity.

### 3. Stash color: orchid pink `#FF87D7` (xterm 212)

- **Why the teal had to go.** The old teal `#00D7AF` (hue ≈169°) sits about 21° from the
  procs cyan `#48CAE4` (≈190°), so two blue-green filled chips competed. That is much of
  why the stash chip read like a second proc chip.
- **Why pink.** Pink is the one hue family no top-bar chip uses. Its nearest neighbors
  are the violet overrides pill (≈260°, 60° away) and the orange monitors and disabled
  chips (≈30°, 70° away). The procs cyan is about 130° away.
- **Why this shade.** `#FF87D7` is at the same pastel xterm level as the rest of the bar
  (`#87D7FF`, `#AF87FF`, `#FFAF5F`), so it fits the palette. Hot magenta `#FF5FD7`
  shouted, rose `#FF87AF` drifted toward the error red, and `#D787D7` blurred into the
  violet pill.
- **Contrast.** The chip text stays `bold #1a1a1a`, at about 8:1 on `#FF87D7`, which
  passes WCAG AA.
- **Accepted cost.** `#FF87D7` is one of the six auto-palette hues for unnamed
  notification tabs, so an unnamed tag's inbox chip can occasionally render pink text
  beside the stash chip. One is text and the other a filled chip with a different glyph,
  so they still read as separate things.

### 4. `disabled:`, plus a sibling `priority:` group

Renaming the label alone would be wrong. The same widget also renders the provider
**priority** pill (`CODEX ★ priority 42m`). When priority and disables are both active,
it folds the disables into `CODEX ★ 42m +1`. The label `disabled: CODEX ★ priority 42m`
would state something false.

So routing state splits into two fixed-label groups. Their names match the vocabulary
that Launch settings (their click target) already uses in its title line:
`priority: CODEX ★ <time>` and `disabled providers: …`.

| Label      | Widget (id)                                                        | Visible when                  | Body (full density)                                                                                                                                                          |
| ---------- | ------------------------------------------------------------------ | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `priority` | new `ProviderPriorityIndicator` (`#provider-priority-indicator`)   | a provider priority is active | `CODEX ★ 42m`. A non-default availability keeps its state word (`CODEX ★ soft-disabled 42m`, `CODEX ★ unavailable 42m`). Palettes are the same as today.                     |
| `disabled` | `ProviderDisablesIndicator` (`#provider-disables-indicator`, kept) | ≥1 active disable             | The disable pill, unchanged: `CLAUDE off 42m`, `CLAUDE soft 42m`, `CLAUDE +2`. Hard disables are listed first, and the soft palette is used only when every disable is soft. |

- **Pill text.** The word `priority` leaves the pill body because the label now says it.
  `★` stays: it is the icon that still identifies the group in compact mode.
- **No `+N` folding.** When both are active the row reads
  `priority:  CODEX ★ 42m  · disabled:  CLAUDE off 42m`. Each fact gets its own label,
  color, and tooltip.
- **Order:** `… overrides · priority · disabled · prompts · inbox`. Launch routing reads
  from what you pinned (overrides, priority) to what you excluded (disabled). `disabled`
  keeps the old `provider` slot, so the orange pill stays where the eye expects it.
- **Clicks.** Both groups open Launch settings (`open_models_panel`).
- **Tooltips split the same way.** The combined `Provider routing state:` heading goes
  away.
  - The priority tooltip has a `Provider priority:` heading, the existing
    `CODEX - preferred · 42m left` line and its detail line, then
    `Press ,m for Config > Launch.`
  - The disabled tooltip has a `Disabled providers:` heading, the per-provider lines,
    the existing hard/soft explanation lines, then the same `,m` line.

**One snapshot, one timer.** Today `ProviderDisablesIndicator` owns the 30 s routing
poll, and it is what leader mode's `_refresh_launch_indicators` refreshes. It stays the
single driver:

- Each `_apply_content()` takes **one** `peek_provider_routing_context(now)` snapshot.
  It renders its own disables body and tooltip from that snapshot, then passes the same
  context to its `ProviderPriorityIndicator` sibling. It finds the sibling through
  `self.parent` and skips silently when the sibling is unmounted or absent.
- Both groups therefore render from the same snapshot and can never disagree. For
  example, a `soft-disabled` priority can never sit beside a disabled pill rendered from
  an older peek.
- No timer is added, no extra peek runs per tick, and `_leader_mode.py` does not change.
- `ProviderPriorityIndicator` peeks once in its constructor, as the disables widget
  already does, so its content is correct before mount. It has no timer of its own.

## Implementation

Use repo-relative paths. Follow the existing idioms: `TopBarGroup` subclasses, pure
static `_build_content` / `_build_tooltip` builders, `_set_body` dedupe, and tooltip
assignment only on change. Follow the TUI perf rules: no I/O, subprocesses, or new
timers on render, refresh, or resize paths.

1. **`src/sase/ace/tui/widgets/top_bar_group.py`**
   - Replace `filled_count_chip(count, hue)` with
     `icon_count_chip(icon: str, count: int, hue: str) -> Text`. It renders
     `{icon} {count}` in `bold #1a1a1a on {hue}`, or empty `Text("")` when `count <= 0`.
   - Update `__all__` and the module docstring (count chips carry their identity glyph).
2. **`src/sase/ace/tui/proc_gear_chips.py`**
   - `gear_chip`'s filled branch returns `icon_count_chip(_GEAR, count, hue)`. The
     hidden-at-zero branch and the dim-unfilled zero branch are unchanged.
   - Rewrite the module docstring: the top bar and the Procs tab header share this chip
     again.
   - The import cannot cycle: `widgets/__init__.py` uses lazy exports, and
     `top_bar_group` imports no indicator modules.
3. **`src/sase/ace/tui/widgets/proc_indicator.py`**
   - `ProcIndicator._build_content` returns `gear_chip(count, PROC_GEAR_HUE)`.
   - `MonitorIndicator._build_content` returns `gear_chip(count, MONITOR_GEAR_HUE)`.
   - Both keep the default `hide_at_zero=True`. Update the docstrings to `procs: ⚙ N`
     and `monitors: ⚙ N`, noting the chips match the Procs tab header.
4. **`src/sase/ace/tui/widgets/stashed_prompts_indicator.py`**
   - Add module-private `_STASH_GLYPH = "≡"` and change `_STASH_ACCENT` to `"#FF87D7"`.
   - The body becomes `icon_count_chip(_STASH_GLYPH, count, _STASH_ACCENT)`.
   - Rewrite the class docstring with the design rationale: the stack glyph, its shape
     contrast with the gear, and pink as the one unused hue family.
5. **New `src/sase/ace/tui/widgets/provider_priority_indicator.py`**
   - Add `ProviderPriorityIndicator(TopBarGroup)` with `GROUP_LABEL = "priority"` and
     `CLICK_ACTION = "open_models_panel"`.
   - Move the priority statics from `ProviderDisablesIndicator` into it:
     `_priority_availability`, `_priority_state_label`, `_priority_tooltip_label`,
     `_priority_tooltip_detail`, and `_priority_palette`.
   - Add `_build_content(priority, *, priority_availability=None, now=None) -> Text`. It
     is the old `_build_priority_content` without `disable_count`: the trailing text is
     `remaining` for the `priority` state and `f"{state} {remaining}"` otherwise. It
     returns empty `Text("")` when there is no priority or the priority has expired.
   - Add
     `_build_tooltip(priority, *, priority_availability=None, now=None) -> str | None`
     (priority-only, as described in Design §4).
   - Add a public
     `apply_routing_context(context: ProviderRoutingContext, *, now: float | None = None) -> None`.
     It computes the availability, calls `_set_body`, and assigns the tooltip only on
     change.
   - The constructor peeks once through `peek_provider_routing_context()` and applies
     the result. The widget has no `on_mount` timer.
6. **`src/sase/ace/tui/widgets/provider_disables_indicator.py`**
   - Set `GROUP_LABEL = "disabled"`.
   - `_build_content(disables, *, now=None)` renders only the disable pill (the current
     `_build_disable_content` logic). `_build_tooltip(disables, *, now=None)` renders
     the disables-only heading, lines, and footnotes.
   - Delete `_build_routing_content`, `_active_disable_count`,
     `_build_priority_content`, and the priority statics that moved in step 5.
   - `_apply_content(now=None)` peeks once, renders itself, then calls a small
     `_apply_priority_sibling(context, now)`. That helper finds
     `#provider-priority-indicator` among `self.parent`'s children, tolerates absence,
     and calls `apply_routing_context`.
   - Keep `_build_initial_content` (the widget test uses it) and the `_text_signature`
     alias.
   - Refresh the class docstring: this widget owns the one routing poll that drives both
     routing groups.
7. **`src/sase/ace/tui/widgets/top_bar.py`**
   - `compose()` yields eight groups. Insert
     `ProviderPriorityIndicator(id="provider-priority-indicator")` between
     `AliasOverridesIndicator` and `ProviderDisablesIndicator`.
   - Replace the three copies of the `by_id` dict and the hard-coded `7` with one
     module-level `_TOP_BAR_GROUP_IDS: tuple[str, ...]` in compose order. Add one
     `_ordered_groups()` helper and use it from `sync_top_bar_groups`, `full_cells`, and
     `compact_cells`. The count check becomes `len(_TOP_BAR_GROUP_IDS)`.
   - Fix docstrings that say "seven".
8. **Exports and CSS**
   - Add `ProviderPriorityIndicator` to the `widgets/__init__.py` / `__init__.pyi` lazy
     exports only if something imports it through the package; `top_bar.py` imports it
     by module path.
   - Keep the new constants module-private so symvision stays clean.
   - No CSS change is expected: the existing `TopBarGroup` rule styles the new group.
     Confirm it needs no per-id rule.

## Tests

Before finishing, read `lint_and_test.md`, `tui.md`, `tui_perf.md`, and
`tui_screenshot.md` with `/sase_memory_read`.

- **`tests/ace/tui/widgets/test_top_bar_group.py`**
  - Replace the `filled_count_chip` test with `icon_count_chip`: plain `⚙ 2`, the fill
    style, and hidden output at zero and at negative counts.
  - Widen the separator-visibility exhaustive test to eight groups (all 2^8 subsets) and
    refresh its sample group texts.
- **`tests/ace/tui/widgets/test_proc_indicator.py`**
  - Each body equals `gear_chip(n, hue)`.
  - Full text reads `procs:  ⚙ 2 ` and `monitors:  ⚙ 1 `.
  - Both are hidden at 0.
- **`tests/test_stashed_prompts_indicator.py`**: full text `prompts:  ≡ 4 `, style
  `bold #1a1a1a on #FF87D7`, hidden at 0.
- **Provider tests**
  - Scope `tests/test_provider_disables_indicator.py`,
    `tests/test_provider_disables_indicator_widget.py`, and
    `tests/_provider_disables_indicator_helpers.py` to disables only, with the
    `disabled` label.
  - Move the priority cases into a new `tests/test_provider_priority_indicator.py`:
    - `CODEX ★ 1h2m`
    - soft-disabled
    - hard-disabled unavailable
    - missing-CLI unavailable
    - an expired priority hides the group
    - the tooltip heading and lines
    - no `+N` in any priority body
- **Driver test (mounted).**
  - With a priority and a disable both active, both groups are visible with one
    separator between them.
  - Count calls to a monkeypatched `peek_provider_routing_context`: exactly one peek per
    `_apply_content`, and both groups reflect that one context.
  - Clearing the priority and refreshing hides the priority group and its separator.
- **`tests/ace/tui/test_top_bar_palette.py`**
  - Build the procs and monitors neighbors with `gear_chip`, and add the priority pill.
  - Add `test_stash_chip_hue_stays_distinct_from_every_neighbor`: the circular HSV hue
    distance between the stash chip background and every neighbor chip background
    (including the updates surface) must be ≥ 45°. The old teal fails this guard at
    about 21° from the procs cyan, so the guard encodes the user's complaint.
  - Add an AA text-contrast check (≥ 4.5:1) for the stash chip.
- **`tests/ace/tui/test_top_bar_order.py`**: `EXPECTED_TOP_BAR_CLUSTER_ORDER` gains
  `provider-priority-indicator` right before `provider-disables-indicator`. Update the
  comment to eight groups.
- **`tests/ace/tui/test_top_bar_indicators.py`**
  - Add an active priority to the busy state and assert all eight labels.
  - If the busy row no longer fits the current wide size, widen it after measuring.
  - In compact, assert that the labels drop while `⚙`, `≡`, and `★` remain.
  - Assert that clicking `priority` and `disabled` runs `open_models_panel`.
- **Stale expectations.** Grep `tests/` for `provider: `, `filled_count_chip`, stash
  `00D7AF`, combined `+1` priority pills, and `Provider routing state` to catch the
  remaining call sites.
- **Visual suites**
  - `tests/ace/tui/visual/test_ace_png_snapshots_top_bar_indicators.py`: add a priority
    to the busy fixture. The full golden must show all eight labeled groups. If they no
    longer fit at 200 columns, rename the golden to the narrowest width where they fit
    and let the update run remove the old one. The compact golden stays at 120x40.
  - The provider goldens in `test_ace_png_snapshots_alias_overrides_indicator.py`
    (`provider_priority_indicator_combined_120x40` now shows both groups) and the
    prompt-stash goldens in `test_ace_png_snapshots_prompt_stash.py` will change. Review
    each one.

## Docs

- **`docs/ace.md`**
  - **"Top-Bar Indicators"** (around line 4721): list the eight labels in order and add
    the icon-chip rule (`⚙` procs and monitors, `≡` prompts, `★` priority; icons stay in
    compact mode). Priority and disabled both open Launch settings.
  - **"Proc Indicator" / "Monitor Indicator"**: describe `procs: ⚙ N` (blue gear chip)
    and `monitors: ⚙ N` (amber gear chip), the same chips the Procs tab header shows.
    Keep the headings so anchors still resolve.
  - **Provider routing paragraph** (around line 3989): rewrite it for the `priority:`
    and `disabled:` groups. Drop the "single priority-led pill with the disable count"
    sentence and describe each group's hover content.
  - **Prompt stash** (around line 6382): `prompts: ≡ N` pink chip.
- Grep `docs/` for other top-bar `provider:`, teal-stash, or combined-pill mentions.
- Run `just fmt`.

## Verification

1. `sase tool run check` (the agent-default `just check`) must pass. Report failures in
   files this change does not touch rather than fixing them.
2. **PNG goldens.** Run `just fix-tui-screenshots` (the full form, through
   `/sase_monitor` if it outlasts the turn), then inspect the retained report as
   `tui_screenshot.md` requires.
   - **Row-local acceptance rule:** an updated golden may differ only in the top-bar
     row. Any diff outside that row is a regression to fix, not approve.
   - Procs tab header goldens must come out unchanged, because `gear_chip` output is
     unchanged. A diff there means the delegation changed the chip.
   - Review every created or removed golden, and expand each update group.
3. **Live check.** Run `sase screenshot` at a wide size (around 220x30) and a narrow
   size (around 90x30), stashing a prompt first (`^S`) so the `prompts:` group shows.
   Inspect the PNGs:
   - the gear and stack glyphs sit crisply inside their fills;
   - the pink chip reads as clearly distinct from the procs cyan;
   - the narrow capture is compact and keeps its icons.

## Acceptance criteria

- The procs and monitors bodies are the Procs tab's own `gear_chip` output
  (`procs:  ⚙ 2 `, `monitors:  ⚙ 1 `).
- The stash group renders `prompts:  ≡ N ` on a `#FF87D7` fill with `bold #1a1a1a` text.
  No `❄` remains anywhere.
- The provider-disable group is labeled `disabled`. An active priority renders in its
  own `priority` group (`CODEX ★ 42m`, plus a state word only for soft-disabled or
  unavailable) placed right before `disabled`. No label ever sits over a pill that
  contradicts it, and no priority body carries a `+N`.
- Both routing groups render from one peek per refresh tick. No timer is added.
- Compact density keeps every icon. Separators never lead, trail, or double across all
  2^8 visibility combinations.
- Existing widget ids and setter APIs are unchanged, and `_leader_mode.py` needs no
  change.
- `just check` passes, goldens are regenerated and inspected under the row-local rule,
  and the docs describe the icons, the stash color, and the `priority:` / `disabled:`
  split.

## Out of scope

- Adding a glyph back to `updates`, and any change to `inbox`, `overrides`, the status
  row, or the usage header.
- Restyling the prompt stash picker modal. It uses the theme `$primary` and violet
  shortcut chips and does not share the badge accent.
- Configurable icons, colors, or labels.
- Changing routing semantics, the order of disables within the pill, or the 30 s poll
  cadence.
