---
tier: tale
title: Group notification-panel rows into per-tab sections
goal:
  The notification modal renders a tab's rows under ordered, richly styled section
  headers separated by blank lines whenever that tab has a grouping strategy, `S`
  toggles that tab between grouped and newest-first for the rest of the ACE session, and
  the `Beads` tab ships grouped by task-bead type with Due / Cleanup / Other buckets for
  the rows that carry no type.
size: medium
proposed_by: bbugyi200.athena.0ed
---

# Plan: Group notification-panel rows into per-tab sections

## Problem

The notification modal renders one flat, newest-first list per tab
(`src/sase/ace/tui/modals/notification_modal_options.py:142`). That is the right default
for a tab whose rows are homogeneous, and the wrong one for the `Beads` tab, which mixes
four unrelated asks that arrive interleaved by arrival time:

| Gate kind            | `action`           | What it asks                                    |
| -------------------- | ------------------ | ----------------------------------------------- |
| `task_triage`        | `TaskTriage`       | Triage a typed task bead (bug / ci / flake / …) |
| `flag_triage`        | `FlagTriage`       | Decide a due feature flag                       |
| `bead_snooze`        | `BeadSnooze`       | A snoozed bead woke up                          |
| `bead_stale_cleanup` | `BeadStaleCleanup` | Close a roster of stale task beads              |

All four declare `presentation.panel: "beads"` (`src/sase/bead/_task_gate_spec.py:122`,
`_flag_gate_spec.py:114`, `_snooze_gate_spec.py:109`, `_stale_cleanup_gate_spec.py:90`),
so the Rust core's `tab_key_for` routes every one of them to the same tab. A reader
triaging five flaky tests has to re-read every row to find the next flake, because a
stale-cleanup digest and a woken plan bead sit between them for no reason other than
when they were sent.

The modal used to section its rows (commit `e449f633f`, PRIORITY / INBOX / MUTED) and
that machinery was replaced by tabs in `e0be8b456`. The residue is still load-bearing
and is exactly the seam this plan reuses: `HEADER_ID_PREFIX = "hdr:"`
(`notification_modal_constants.py:54`) and the four guards that skip `hdr:`-prefixed
option ids (`notification_modal.py:227,496,509` and
`notification_modal_options.py:184`).

## Goal

- A notification tab with a registered **grouping strategy** renders its rows under
  ordered section headers, one blank line between adjacent sections, headers styled with
  the section's own glyph and accent color and carrying a row count.
- `S` toggles the **active tab** between `grouped` and `recent` (flat, newest-first —
  today's exact render). The choice is per tab and lives for the ACE process.
- Grouped is the default for any tab that has a strategy; `recent` is the default
  everywhere else, and remains the only mode there.
- The `Beads` tab ships with the `bead_type` strategy: one section per task-bead type
  present, in catalog order, then `Due`, `Cleanup`, and `Other` for rows that carry no
  type.
- `ace.notification_tabs.<tab>.grouping` selects or disables a tab's strategy.
- Nothing about row content, tab membership, marks, jump mode, dismiss/mute/snooze
  replacement selection, or `R` changes.

## Non-goals (deliberately excluded)

Read this list before widening scope; each item was checked and left alone.

- **No Rust core change.** `sase/memory/rust_core_backend_boundary.md` asks whether
  another frontend would need the behavior to match the TUI. Tab _membership_ is in
  `sase-core` (`crates/sase_core/src/notifications/tabs.rs`) because three surfaces —
  panel, top-bar indicator, mobile snapshot — must agree on which bucket owns a row.
  Sections have exactly one consumer: this OptionList. They are the same class of
  presentation as the tab strip's compact mode, the auto-color palette, and the priority
  marks, all of which already live in Python
  (`src/sase/ace/tui/widgets/notification_tab_style.py`). Keep the strategy functions
  pure and I/O-free anyway, so a second frontend could lift them into Rust unchanged. A
  new core binding would also cost a `sase-core-rs` release and a pin bump
  (`pyproject.toml:46`) for a widget-layout concern.
- **No feature flag.** Per `sase/memory/sase_flags.md`, a flag is for behavior that is
  not ready or for a deprecated branch callers must migrate off. `recent` is a mode the
  user is meant to choose forever, which that note names explicitly as a config field,
  not a flag.
- **No new keymap scope.** Every notification-modal key is a literal in
  `NotificationModal.BINDINGS` (`notification_modal.py:64`); there is no
  `ace.keymaps.notifications` section to extend. Adding one for a single binding is a
  larger, separate change.
- **No per-user custom sections.** A strategy is a registered built-in selected by name.
  Config picks _which_ strategy a tab uses, not what its sections are.
- **Only one real strategy ships.** `bead_type` for `beads`. Do not add speculative
  `sender` / `action` / `tag` strategies; the registry is the seam, and a second
  strategy should arrive with a second real use case.
- **Row labels are unchanged.** The per-row type chip stays even though the section
  header repeats its glyph: identical rows in both modes keeps the diff, the goldens,
  and the reader's eye stable.
- **The tab strip is unchanged.** No mode marker next to the tab name; the presence of
  headers is the affordance, and the strip is already carrying icon, label, count, and a
  priority mark.
- **The Snoozed and Muted tabs stay flat.** They hold rows from every panel; a
  by-origin-tab strategy for them is a separate idea.
- **Do not add `[flag]` / `[cleanup]` entries to `ACTION_BADGES`.** `FlagTriage` and
  `BeadStaleCleanup` are genuinely missing from `notification_modal_constants.py:8-24`,
  which is a real (small) gap — file it as a task bead with `/sase_new_task` instead of
  folding it in here.

## Design decisions

### Strategy model

A strategy maps one `Notification` to one `NotificationSection`; the caller groups and
orders. Both records are frozen dataclasses with no I/O beyond the memoized task-type
registry:

```python
@dataclass(frozen=True)
class NotificationSection:
    key: str        # unique within a render: "type:flake", "kind:due"
    label: str      # "Flaky test" — rendered upper-cased
    glyph: str      # "≈"
    color: str      # "#00D7D7"
    order: tuple[int, int, str]   # ascending sort key
```

`order` is a tuple so a strategy can express "typed sections in catalog order, then the
fallback buckets" without a hand-maintained integer ladder.

### The `bead_type` strategy

Every bead gate freezes its task-type presentation into `action_data` at creation time
(`notification_gates/service.py:355-360` writes `gate_chip_glyph` / `gate_chip_label` /
`gate_chip_color`, and `task_type_gate_chip` sets `label` to the **slug**). So the
section is derived from data the row already carries:

1. `chip = gate_chip_from_action_data(n.action_data)` — the zero-I/O render-path reader
   that never raises (`notification_gates/presentation.py:171`).
2. If a chip exists, `slug = chip.label`. When `slug` is in
   `get_task_type_registry().by_slug`, take `label` / `glyph` / `accent_color` from
   `task_type_presentation(slug)` and order by the slug's index in `registry.records`.
   This is the same guard-then-resolve shape
   `notification_tab_style._task_type_tab_glyph_and_color` already uses on the tab-strip
   render path, and it is why an unresolvable slug must **not** call
   `task_type_presentation` (it would degrade every unknown string to the `?` glyph).
   When the slug is unknown, fall back to the chip's own glyph / label / color and sort
   after every known type.
3. With no chip, bucket by `action_data["request_kind"]`, which
   `notification_gates/service.py:340-346` stamps on every gate-backed notification:
   - `bead_snooze` → `Due` (`⏰`, `#FFAF00`)
   - `bead_stale_cleanup` → `Cleanup` (`🧹`, `#5FAFAF`)
   - anything else → `Other` (`◈`, `#AF87FF`, the tab's own color)

Ordering follows from `order`: `(0, catalog_index, label)` for known types,
`(0, len(records), label)` for unknown slugs, then `(1, 0, "")` Due, `(2, 0, "")`
Cleanup, `(3, 0, "")` Other. With the shipped catalog that reads: **Bug, CI failure,
Feature, Feature flag, Flaky test, GitHub, Memory, Due, Cleanup, Other**.

Grouping by the _bead's_ type rather than by gate kind is the deliberate call: a woken
`BeadSnooze` for a bug bead belongs next to the bug awaiting triage, because they are
about the same work. The row's own `[snooze]` / `[test]` badge already says which ask it
is. Only rows with no type at all fall back to a gate-kind bucket, which is what keeps a
woken plan or epic bead out of a misleading typed section.

### Configuration

Reuse the block that already exists for per-tab presentation:

```yaml
ace:
  notification_tabs:
    beads:
      color: "#AF87FF"
      icon: "◈"
      priority: 0
      grouping: bead_type # a strategy id, or "recent" to default this tab to flat
```

Resolution, highest first: the configured `grouping`; the built-in default for the tab
(`{"beads": "bead_type"}`); otherwise no strategy. `grouping: recent` and an unknown id
both resolve to no strategy — the same tolerate-and-degrade contract `color` and `icon`
already have. Parse it inside `_ConfiguredTabStyle` so a render still pays exactly one
token-cached config read (rule 8 of `sase/memory/tui_perf.md`), and ship the value
explicitly in `default_config.yml` _and_ keep the built-in default, mirroring the
existing "these keep the indicator styled if that block is emptied" comment
(`notification_tab_style.py:52-55`).

The JSON Schema block at `src/sase/config/sase.schema.json:1198` sets
`additionalProperties: false`, so `grouping` must be added there or a config carrying it
fails validation. Give it an `enum` of `["", "recent", "bead_type"]` — the config hub
and the nvim schema integration both get completion out of it — and pin the enum to the
registry with a parity test so a new strategy cannot be added without updating it.

### The `S` binding

Free in the modal (used:
`escape q j k ↑ ↓ ^n ^p x d y n e V Y [ ] R M m s ' ^d ^u g G`). Capital letters in this
modal are already the tab-scoped operations — `R` read tab, `M` mute, `V` view, `Y` copy
path — and grouping is tab-scoped, so `S` for **S**ections fits the pattern and is
self-describing. `g` would have matched the Statistics pane's `cycle_group`, but `g`/`G`
are taken here by detail-pane scrolling; `o` was rejected as "open"-shaped next to
`Enter`. Bind `"S"` alone, as `V`/`Y`/`R`/`M` do (only `G` carries a `shift+g` twin, for
historical reasons).

On a tab with no strategy, `S` is an explicit no-op with a toast rather than a silent
one. On a tab with one, it toasts the new mode (`Beads · grouped by type` /
`Beads · newest first`) and re-highlights the same notification, not the same row index.

### Session scope

Per-tab modes live in one dict created in `init_runtime_state`, exactly like
`AdminCenterSessionState` (`_state_init_runtime.py:56-60`) after commit `a3b69bd85`. It
dies with the ACE process, so a restart returns to the configured default, and nothing
new is written to disk. `NotificationModal` is constructed in exactly one place
(`actions/agents/_notification_modal_flow.py:216`), so the parameter is optional and
tests that construct the modal bare get their own empty map.

### Header shape

Follow the Models-panel precedent (`modals/models_panel_rendering_layout.py:105-126`):
disabled `Option`s for headers and spacers, so Textual's `find_next_enabled` cursor
navigation skips them with no extra code (verified in
`textual/widgets/_option_list.py:980-1010`, textual 8.0.1).

```
▎⨯ BUG ────────────────────────  3
  * ✦ ⨯ [test] sase-cy — Add mobile bridge retry badge   4m ago  #bug
  …
                                              ← disabled blank spacer
▎≈ FLAKY TEST ─────────────────  2
```

The rule is padded to a **fixed** 32-cell field, not the widget width: the counts line
up across sections, and no resize handler or rebuild-on-resize is needed. Compute the
padding with `rich.cells.cell_len` (`⏰` and `🧹` are two cells) and emit a single space
instead of a negative rule when a label overruns the budget.

Option ids keep the existing prefix — `hdr:sec:<config_key>:<section.key>` and
`hdr:gap:<config_key>:<section.key>` — so all four `startswith(HEADER_ID_PREFIX)` guards
keep working untouched. That is the single biggest reliability lever in this change:
selection, dismissal, and jump mode are already header-aware.

## Implementation

### Part 1 — the section engine

1. Add `src/sase/ace/tui/modals/notification_sections.py`:
   - `NotificationSection` and `NotificationSectionStrategy` (frozen dataclasses; the
     strategy holds `id`, `display_name`, and `section_for(Notification)`).
   - `NOTIFICATION_SECTION_STRATEGIES: dict[str, NotificationSectionStrategy]` and
     `DEFAULT_TAB_STRATEGY_IDS = {"beads": "bead_type"}`.
   - `RECENT_STRATEGY_ID = "recent"` (the reserved "no sections" id; never a registry
     member).
   - `resolve_tab_section_strategy(config_key: str) -> NotificationSectionStrategy | None`
     implementing the precedence in _Configuration_ above.
   - `group_notifications(rows, strategy) -> list[tuple[NotificationSection, list[T]]]`
     — a stable partition that preserves each group's incoming (already sorted) row
     order and sorts groups by `section.order`. Generic over the `(index, notification)`
     pairs the modal passes.
2. Add `src/sase/ace/tui/modals/notification_sections_bead.py` with the `bead_type`
   strategy from _The `bead_type` strategy_ above, and register it from
   `notification_sections.py`. Keep the module import-cycle-free: it may import
   `sase.notification_gates.presentation`, `sase.task_type_presentation`, and
   `sase.task_types.registry`, but nothing from `sase.ace.tui.modals`.
3. Add `src/sase/ace/tui/modals/notification_section_render.py`:
   `render_notification_section_header(section, count) -> Text` and
   `render_notification_section_spacer() -> Text`, per _Header shape_. Both are pure and
   take no widget.

### Part 2 — config plumbing

4. `src/sase/ace/tui/widgets/notification_tab_style.py`:
   - Add `grouping: str = ""` to `_ConfiguredTabStyle` (line 128) and to
     `_EMPTY_TAB_STYLE` (line 136); parse it in `_configured_tab_styles_for_token`
     (line 376) with a `_sanitize_grouping` that accepts a stripped `^[a-z][a-z0-9_]*$`
     string and returns `""` for anything else.
   - Add `resolve_notification_tab_grouping(config_key: str) -> str` returning the
     configured value or `""`, and export it.
   - Extract `notification_tab_config_key_for_tag(tag: str | None) -> str` from
     `_notification_tab_config_key` (line 157) — same `__snoozed__` → `snoozed`,
     `__muted__` → `muted`, `None` → `general` mapping — and make the existing function
     a one-line delegate. `notification_sections` and the modal both need the key from a
     bare tag.
   - Update the module docstring: it now resolves grouping alongside color, icon, and
     priority.
5. `src/sase/default_config.yml:222`: add `grouping: bead_type` to the `beads` entry and
   extend the comment block above `notification_tabs` (lines 209-214) to describe it.
6. `src/sase/config/sase.schema.json:1220` (beside `priority`): add
   ```json
   "grouping": {
     "type": "string",
     "enum": ["", "recent", "bead_type"],
     "description": "Row-grouping strategy for this tab's notification list: a strategy id such as \"bead_type\" renders ordered sections with headers, \"recent\" renders one flat newest-first list, and \"\" uses the built-in default for the tab. Toggle at runtime with S in the notification panel.",
     "default": ""
   }
   ```

### Part 3 — modal rendering

7. `src/sase/ace/tui/modals/notification_modal_options.py`:
   - Split the tab filter + activity sort out of `_create_notification_options` (lines
     142-177) into `_active_tab_rows(self) -> list[tuple[int, Notification]]`, unchanged
     in behavior.
   - `_create_notification_options` asks the host for the effective strategy
     (`self._active_section_strategy()`, Part 4). `None` → today's flat list, built by
     the same loop, byte-identical.
   - Otherwise: `group_notifications(...)`, then for each group emit a spacer option
     (skipped before the first group), a header option, and the group's row options.
     Header and spacer options are `disabled=True` with `hdr:`-prefixed ids per _Header
     shape_.
   - `_visual_notification_index_order` (line 184) needs no change — it already drops
     disabled and id-less options — but confirm it, because it is what dismiss
     replacement selection and jump order are built on.
   - Leave `_create_sectioned_options` (line 178) as the delegating alias it already is.
8. `src/sase/ace/tui/modals/notification_modal_constants.py`: add `S: sections` to
   `DEFAULT_HINT_TEXT`, `QUESTION_HINT_TEXT`, and `GATE_HINT_TEXT`, next to `[]: tags`.

### Part 4 — mode state and the toggle

9. Add `src/sase/ace/tui/modals/notification_section_modes.py` with a small mutable
   `NotificationSectionModes` holding `dict[str, str]` keyed by config key, with
   `mode_for(config_key)` (explicit choice, else `"grouped"` when a strategy resolves,
   else `"recent"`), `toggle(config_key) -> str`, and `strategy_for(config_key)`
   returning the strategy only when the effective mode is `"grouped"`.
10. `src/sase/ace/tui/modals/notification_modal.py`:
    - `__init__` (line 93) gains `section_modes: NotificationSectionModes | None = None`
      and stores `section_modes or NotificationSectionModes()`.
    - Add `_active_section_strategy()` →
      `self._section_modes.strategy_for( notification_tab_config_key_for_tag(self._active_notification_tag))`.
    - Add `("S", "toggle_sections", "Sections")` to `BINDINGS` (line 64), after
      `("R", "read_tab", …)`.
    - Add `action_toggle_sections`: resolve the active tab's config key; when
      `resolve_tab_section_strategy` returns `None`, `self.notify` an informational "No
      sections for <Tab>" and return. Otherwise capture the highlighted notification's
      **id** (not index) via `_get_highlighted_notification`, toggle, then
      `self._rebuild_list(highlight_index=self._visible_notification_index_for_id( captured_id))`
      and toast the new mode. Both helpers already exist (lines 424-437 and 198).
    - No change to `_switch_notification_tag_tab` (line 402) or
      `_clear_tab_scoped_state` (line 418): the mode map is keyed by tab, so switching
      tabs already shows that tab's own mode, and the mode is deliberately _not_
      tab-scoped state to clear.
11. `src/sase/ace/tui/actions/_state_init_runtime.py` (beside the Admin Center block at
    lines 52-60): `self._notification_section_modes = NotificationSectionModes()`, with
    a comment saying it is per-process by design.
12. `src/sase/ace/tui/actions/agents/_notification_modal_flow.py:216`: pass
    `section_modes=getattr(self, "_notification_section_modes", None)`. The `getattr`
    keeps the several tests that drive this flow against a stub app working.

### Part 5 — tests

13. New `tests/test_notification_sections.py` — the pure engine:
    - `bead_type` sections for one notification of each kind: typed `TaskTriage` (chip →
      `Flaky test`, `≈`, `#00D7D7`), `FlagTriage` (chip slug `flag` → `Feature flag`),
      untyped `BeadSnooze` → `Due`, `BeadStaleCleanup` → `Cleanup`, a `panel: beads` row
      with neither chip nor known `request_kind` → `Other`.
    - An unknown chip slug keeps the chip's own glyph/label/color and sorts after every
      known type — and does **not** pick up the `?` degraded presentation.
    - Malformed chip `action_data` (non-mapping, junk color, oversized label) never
      raises and falls through to a bucket.
    - Section order matches the catalog order, then Due, Cleanup, Other.
    - `group_notifications` preserves incoming row order inside a group and drops empty
      sections entirely.
    - `resolve_tab_section_strategy`: built-in default for `beads`; `grouping: recent` →
      `None`; unknown id → `None`; a strategy id set on a tab that has no built-in
      default → that strategy.
14. New `tests/test_notification_modal_section_toggle.py` — the modal:
    - The beads tab renders headers by default; every header/spacer option is `disabled`
      and `hdr:`-prefixed; exactly one blank spacer sits between adjacent sections and
      none precedes the first.
    - `_visual_notification_index_order()` on a grouped tab returns only notification
      indexes, in section-then-recency order.
    - `action_toggle_sections` flips to flat, produces output identical to
      `_create_notification_options` with the strategy forced off, and keeps the same
      notification highlighted across the toggle.
    - Toggling the beads tab does not change the general tab's mode, and vice versa.
    - `action_toggle_sections` on a tab with no strategy leaves the options untouched
      and notifies.
    - Jump hints (`'`) land on notification rows only, in grouped visual order.
15. Extend `tests/test_notification_tab_style.py` for `grouping` parsing: a valid id, an
    empty string, a non-string, and a junk value all round-trip through
    `resolve_notification_tab_grouping`; and `notification_tab_config_key_for_tag`
    covers `None` / `__snoozed__` / `__muted__` / a plain tag.
16. Extend `tests/test_config_schema_notification_tabs.py`:
    `{"beads": {"grouping": "bead_type"}}` and `{"beads": {"grouping": "recent"}}`
    validate; `{"beads": {"grouping": "nope"}}` and a non-string are rejected. Add the
    parity test that the schema enum equals
    `{"", RECENT_STRATEGY_ID} | set(NOTIFICATION_SECTION_STRATEGIES)`.
17. Audit the beads-tab tests that assert option ids or counts and update only what the
    grouped default actually moves: `tests/test_notification_modal_mark_and_tabs.py`
    (lines ~188-202), `tests/test_notification_modal_read_tab.py:107-124`,
    `tests/test_notification_modal_tab_routing.py:51-64,259-272`. Most assert tab
    membership rather than row order and should need nothing; where a test does depend
    on flat beads rows, prefer forcing `recent` on the modal's mode map over rewriting
    the expectations.
18. `tests/test_notification_modal_sections.py` covers the _General_ tab, which has no
    strategy, so its flat expectations must still hold unchanged — treat any failure
    there as a regression in the no-strategy path, not a test to update.

### Part 6 — visual goldens and docs

19. `tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py`: the two beads
    goldens (`notification_beads_tab_120x40`, `notification_beads_typed_gates_120x40`)
    now render grouped and must be rebaselined with
    `just test-visual --sase-update-visual-snapshots`. **Look at the new PNGs before
    accepting them** — this is the one place the design's beauty is actually verified:
    check alignment of the count column, that the accent bar reads as a header, and that
    no rule overruns at 120 cells. Add one new case that presses `S` and captures the
    flat render (`notification_beads_recent_120x40`), so both modes have a golden.
20. `docs/notifications.md`: add `S` to the Modal Keybindings table (line ~39, beside
    `[` / `]`), and add a **Sections** subsection under _Tabs and Ordering_ (line 72)
    covering: which tabs have a strategy, the `Beads` section order, that rows inside a
    section keep the activity ordering documented at line 113, that the mode is per tab
    and per ACE process, and that `recent` is byte-identical to the old render.
21. `docs/configuration.md`: add `grouping` to the `ace.notification_tabs` field table
    (line ~893) and a paragraph after the priority prose (line ~931) naming the shipped
    strategy, the `recent` opt-out, and the `S` runtime toggle.

## Verification

- `just install` first if this workspace is cold, then `just check`.
- `just check-full` through `/sase_monitor` before landing: this touches the config
  schema, `default_config.yml`, and a shared TUI modal, all of which sit outside a
  diff-scoped selection.
- `just test-visual` after the rebaseline, and a second run to confirm the new goldens
  are stable rather than renderer-dependent.
- `sase doctor -C config.notification_tabs` stays `OK` — the check only reads `icon`
  (`doctor/checks_config_notification_tabs.py:20-40`) and must not start warning because
  a new key appeared.
- Manual smoke in `sase ace`: press `i`, go to `Beads`. Sections must appear in catalog
  order with one blank line between them and the counts aligned. Press `S` — the toast
  must name the new mode, the same row must stay highlighted, and the list must go flat.
  Press `S` again to return. Press `]` to another tab and `S` there: a tab with no
  strategy must toast and change nothing. `j`/`k` must never land on a header. Press `'`
  and confirm hints skip headers. Dismiss the last row of a section and confirm the
  header disappears with it and the highlight lands on a live row.
- Set `ace.notification_tabs.beads.grouping: recent` in the user config, restart ACE,
  and confirm the Beads tab opens flat and `S` still turns sections on for the session.

## Risks

- **The visual goldens are the main churn.** Two existing PNGs move. If the diff shows
  anything beyond the new headers and spacers — shifted rows, a changed chip, a clipped
  tab strip — stop and report it rather than accepting the golden.
- **`request_kind` is the discriminator for untyped rows.** It is stamped by
  `notification_gates/service.py:340-346` for every gate-backed notification, but a
  hand-written or pre-gate notification declaring `panel: "beads"` has none. That is
  exactly why `Other` exists; do not tighten it into an exception.
- **Do not call `task_type_presentation` on an unresolved slug.** It returns a degraded
  `?` / grey presentation for _any_ string, which would turn a typo'd slug into a
  plausible-looking section. Guard on `registry.by_slug` first, as
  `notification_tab_style._task_type_tab_glyph_and_color` does.
- **Keep the flat path byte-identical.** Every existing test of the flat render, and the
  `recent` mode users fall back to, depend on it. Build both modes through the same row
  loop rather than forking the renderer.
- **Header ids must keep the `hdr:` prefix.** Four call sites outside the option builder
  identify non-row options by that prefix alone. A new prefix would silently make
  headers selectable and `int()`-parsed.
