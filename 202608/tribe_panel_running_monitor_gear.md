---
tier: tale
size: small
title: Render the running-monitor gear badge on tribe panel titles
goal:
  Every Agents-tab tribe panel title carries a bold amber `⚙N` badge counting the
  running monitors contained in that tribe, rendered immediately before the existing
  grey settled badge so the panel title mirrors the clan/family container row's
  amber-then-grey lane pair.
proposed_by: bbugyi200.athena.06j.f0
create_time: 2026-08-18 15:47:09
status: wip
---

# Render the running-monitor gear badge on tribe panel titles

## Goal

Commit `ef30e98f2` (plan `202608/tribe_panel_settled_monitor_gear.md`) lifted the grey
`⚙N` **settled**-monitor badge from clan/family container rows onto Agents-tab tribe
panel titles. Its decision 2 deliberately deferred the amber lane: _"Do not add an amber
running badge to the panel title. Out of scope for this tale; the shared counter returns
both lanes, so a later change is a two-line render addition."_ This tale is that
follow-up.

Today a collapsed tribe panel tells you how much monitored work has **finished** inside
it and nothing about what is **still running** — the more urgent of the two facts. Add
the amber lane so a panel title reports the same two-lane partition that a collapsed
container row already reports.

Target rendering — the two badges are the last elements of the title, amber first:

```
@review · 5 [R2 D3] ⚙1 ⚙4
▸ ∴ @research · 12 [R2] ⚙2
⌂ @default · 3 ⚙1 ⚙3
```

## Current behavior (read this before editing)

Almost all the machinery already exists; this is plumbing plus one render block.

- `src/sase/ace/tui/models/agent_family_members.py` already exposes
  `panel_monitor_lane_counts(rows) -> MonitorLaneCounts`, which returns **both**
  `running` and `settled` from one shared `_MonitorLaneTally` traversal (root-inclusive,
  deduped across all roots, cycle-guarded on `id(row)`, deduped on `row.identity`). No
  change is needed in this file.
- `src/sase/ace/tui/actions/agents/_display_panel_titles.py`:
  - `AgentPanelCounts` carries `settled_monitors: int = 0` as its last field and
    deliberately excludes it from `metric_items()`.
  - `agent_panel_counts` computes
    `settled_monitors=panel_monitor_lane_counts(visible_top_level_agents).settled` —
    note it discards `.running` today.
  - `agent_panel_border_title` appends, in order: optional `[hint] ` prefix, `❖ `/`▸ `
    marker, icon + `@tribe` (or `All agents`), `↺`, `▿N` fold-restore,
    ` · <lane_count>`, the `[S1 R2 …]` chip, then the grey `⚙N` badge (currently last).
  - Constants `_PANEL_MONITOR_GLYPH = MONITOR_GLYPH` and
    `_PANEL_MONITOR_SETTLED_STYLE = MONITOR_SETTLED_GLYPH_COLOR` are already defined.
- All five title call sites route through `PanelCollectionMixin._agent_panel_title`
  (`_display_panel_collection.py:105`), and `_display_panel_patches.py:350` builds
  `AgentPanelCounts` via `agent_panel_counts`, so no call site needs touching.
- The row-side amber badge to mirror:
  `src/sase/ace/tui/widgets/_agent_list_render_agent.py:487-497` appends **running
  first, then settled**, using `_MONITOR_COUNT_GLYPH_STYLE = "bold #FFAF5F"` and
  `_MONITOR_SETTLED_COUNT_GLYPH_STYLE = MONITOR_SETTLED_GLYPH_COLOR` from
  `src/sase/ace/tui/widgets/_agent_list_styling.py:93-103`.
- `src/sase/monitor_state.py` owns the hue tokens: `MONITOR_GLYPH = "⚙"`,
  `MONITOR_GLYPH_COLOR = "#FFAF5F"`, `MONITOR_SETTLED_GLYPH_COLOR = "#9E9E9E"`.

## Design decisions

Make these exact choices; each one answers a question a reviewer will ask.

1. **Reuse the lane the existing traversal already returns.** `agent_panel_counts`
   already calls `panel_monitor_lane_counts` once and throws `.running` away. Bind the
   result to a local and read both lanes — do **not** add a second call, a second
   traversal, or a new predicate. Sharing one traversal is what makes the two panel
   counts, and the panel counts and the row counts, unable to disagree. A monitor that
   has not reported a terminal state and has no `stop_time` counts as running, exactly
   as it does on container rows.
2. **Order: amber before grey.** `_agent_list_render_agent.py` appends running then
   settled on container rows; the title must match so the leftmost gear is always the
   live lane on every surface. This is the whole reason the badge is inserted _before_
   the existing grey badge rather than appended after it.
3. **Bold amber, not plain amber.** `_agent_list_styling.py`'s own comment states the
   settled badge is non-bold _deliberately_, so hue **and** weight both separate the two
   lanes for low-color terminals and red/green color vision deficiency. A non-bold amber
   title badge would drop that second channel exactly where the two gears are adjacent.
   Use `bold` + `MONITOR_GLYPH_COLOR`.
4. **Hue lives in `sase.monitor_state`; weight lives in the surface.**
   `monitor_state.py` holds colors and glyphs, not Rich style syntax — do **not** add a
   `"bold #FFAF5F"`-shaped constant there. Each surface composes `f"bold {…_COLOR}"`
   itself, which is why step 1 derives the widget-side constants from the token instead
   of moving the composed style into the domain module.
5. **Zero-suppress each lane independently.** `running_monitors == 0` renders nothing —
   no glyph, no separator space. Every existing exact-title assertion (all of which have
   `running_monitors` defaulting to `0`) must keep passing byte-for-byte. Never render
   `⚙0`. The Procs tab header deliberately _does_ render `⚙ 0` (`docs/ace.md:5464`, so a
   missing chip cannot read as "unknown"); that is a different surface with a filled
   chip — do not copy it here.
6. **Position rule: the two badges are the last elements, in fixed order.** Append order
   inside the `counts is not None` block becomes chip → amber → grey, each badge
   preceded by a `" "` in `_PANEL_COUNT_STYLE` (mirroring how the title already
   separates `↺`, `▿N`, and the chip). The chip is zero-suppressing and a
   `Starting`-bucket agent counts toward `lane_count` without landing in any metric, so
   `@chop · 3 ⚙1` (badge directly after the total, no `]`) is reachable and correct.
7. **Keep the amber hue on selected panels.** Same rule as the grey badge: the gear is a
   metric, not chrome, so it never takes the focus accent. Note for the reviewer: three
   warm tones can now share one title — selected chrome `#FFD75F`, the `↺`/`▿N` restore
   markers `bold #D7AF5F`, and this badge `bold #FFAF5F`. Accepted: the restore markers
   sit left of the `·` total and use different glyphs, the `⚙` glyph carries the
   meaning, and the contrast that actually has to survive is amber-vs-grey between the
   two adjacent gears, which is maximal.
8. **Carry it as `running_monitors`, declared immediately before `settled_monitors`.**
   Declaration order should match render order. This is safe: `AgentPanelCounts` has
   only one non-test construction site (`_display_panel_titles.py:88`) and every
   construction in the tree — source and tests — is keyword-based, so no positional
   argument shifts. Both fields default to `0`.
9. **Keep it out of `metric_items()`.** Same reason as `settled_monitors`: monitors are
   not agents, and `sum(metric_items()) == lane_count` is a true invariant asserted in
   `tests/ace/tui/test_agent_panel_titles.py`. Leave `_PANEL_METRIC_LABELS` and
   `_PANEL_METRIC_STYLES` untouched. Consequence, unchanged from today: `counts=None`
   renders no badge, just as it renders no chip.
10. **Fold- and collapse-independent by construction.** Inherited for free — the count
    traverses subtrees from `visible_top_level_agents`, never the rendered panel slice.
    Do not introduce a slice scan.
11. **Do not reuse `src/sase/ace/tui/proc_gear_chips.py`.** `gear_chip()` builds the
    _filled_ `⚙ N` badge with a background, used by the top bar and the Procs tab
    header. This is the bare foreground-hue `⚙N` text badge — a different visual object.
12. **Stay in Python; no Rust core change.** Presentation aggregation over TUI `Agent`
    rows; the one piece of real domain semantics (which monitor states are terminal)
    already lives in `sase.monitor_state.monitor_state_is_terminal`.
13. **Add no new refresh path and no cache.** Same argument as the settled badge
    (decision 14 of the prior tale): `Agent` is a plain dataclass whose
    `runtime_children` / `followup_agents` participate in `__eq__`, so a monitor
    starting or finishing makes its container compare unequal in
    `build_agent_display_diff` → `changed_same_position` → `_try_patch_agent_row`, which
    already rebuilds the panel title; a monitor appearing or disappearing as a top-level
    row is an add/remove diff that rebuilds the panel. Do not touch
    `_refresh_agent_panel_titles` or add a timer.
14. **Leave `src/sase/ace/tui/modals/help_modal/agents_bindings.py` alone.** Its entry
    (line 444) already reads "N running monitors (amber)" — lane-scoped, not
    surface-scoped, so it reads correctly for the title badge too. Deliberate; do not
    "fix" it.

## Implementation

### Step 1 — derive the amber widget styles from the shared hue token

`src/sase/ace/tui/widgets/_agent_list_styling.py` currently hardcodes `#FFAF5F` in three
constants (lines 94-96) while importing `MONITOR_SETTLED_GLYPH_COLOR` for the grey one.
This is the exact analogue of the prior tale's step 1, applied to the amber lane.

- Import `MONITOR_GLYPH_COLOR` alongside the existing
  `from sase.monitor_state import MONITOR_GLYPH, MONITOR_SETTLED_GLYPH_COLOR`.
- Redefine, keeping every existing comment in place:
  - `_MONITOR_GLYPH_STYLE = f"bold {MONITOR_GLYPH_COLOR}"`
  - `_MONITOR_ROW_STYLE = MONITOR_GLYPH_COLOR`
  - `_MONITOR_COUNT_GLYPH_STYLE = f"bold {MONITOR_GLYPH_COLOR}"`

All three resolve to byte-identical strings, so this is a pure no-op for rendering and
for any test asserting the literal `"bold #FFAF5F"`. Do all three rather than only the
badge one: leaving two `#FFAF5F` literals beside one derived constant invites exactly
the drift this step exists to prevent.

Symvision note: module constants are not checked (functions and classes are), so nothing
here needs a pragma, and this tale adds no new public function.

### Step 2 — count and render

`src/sase/ace/tui/actions/agents/_display_panel_titles.py`:

- Extend the existing import to
  `from sase.monitor_state import (MONITOR_GLYPH, MONITOR_GLYPH_COLOR, MONITOR_SETTLED_GLYPH_COLOR)`.
- Add `_PANEL_MONITOR_RUNNING_STYLE = f"bold {MONITOR_GLYPH_COLOR}"` beside the existing
  `_PANEL_MONITOR_SETTLED_STYLE`.
- In `AgentPanelCounts`, add `running_monitors: int = 0` **immediately before**
  `settled_monitors: int = 0`. Update the class docstring so it covers both fields: they
  count the running and finished monitors across the panel's subtrees, partition those
  monitors exactly, and are both intentionally absent from `metric_items()` because
  monitors are not agents and folding them in would break the disjoint-status invariant
  that the metric counts sum to `lane_count`.
- In `agent_panel_counts`, replace the inline
  `panel_monitor_lane_counts(visible_top_level_agents).settled` expression with a local
  bound before the `return`, e.g.
  `monitor_lanes = panel_monitor_lane_counts(visible_top_level_agents)`, then pass
  `running_monitors=monitor_lanes.running` and `settled_monitors=monitor_lanes.settled`.
  One traversal, both lanes (decision 1).
- In `agent_panel_border_title`, inside the existing `if counts is not None:` block,
  insert the amber badge **between** the chip block and the settled-badge block:

  ```python
  if counts.running_monitors:
      title.append(" ", style=_PANEL_COUNT_STYLE)
      title.append(
          f"{_PANEL_MONITOR_GLYPH}{counts.running_monitors}",
          style=_PANEL_MONITOR_RUNNING_STYLE,
      )
  ```

  Leave the existing `if counts.settled_monitors:` block exactly as it is, after this
  one.

No other source file changes.

### Step 3 — docs

Four edits, all of which currently assert the panel title shows _only_ the finished lane
and must stop saying so.

- `docs/ace.md`, badge legend table (~line 1856-1857). The amber row currently reads "N
  running monitors in a family/clan subtree (amber)"; the grey row already reads "…, or
  in a tribe panel title for its whole tribe (grey)". Extend the amber row the same way
  so both rows describe both surfaces.
- `docs/ace.md`, **Tribe Side Panels** (~lines 1462-1468). This paragraph currently says
  the title "can end with a grey `⚙N` badge after the metric chip … and it shows only
  the finished lane: running monitors are not shown on the panel title, only on
  container rows." Rewrite it to say the title can end with an amber `⚙N` badge for
  running monitors followed by a grey `⚙N` badge for finished ones, in that order, after
  the metric chip (or after the total when the chip is empty); that the two counts
  partition the tribe's monitors exactly; that each badge is fold- and
  collapse-independent and omitted when its own count is zero; and that both keep their
  semantic hue on a selected panel while the brackets and metric letters take the focus
  accent. Delete the "running monitors are not shown on the panel title" sentence
  outright.
- `docs/ace.md`, monitor-row paragraph (~lines 1872-1877). Its last sentence currently
  reads "The tribe panel title aggregates the finished lane across the whole tribe, so a
  fully collapsed panel still reports how much monitored work has already completed
  inside it." Change it to say the title aggregates **both** lanes across the whole
  tribe, so a fully collapsed panel still reports both what is still running and how
  much has already completed.
- `docs/monitors.md`, **In the ACE TUI** (~lines 296-300). Same correction to the
  matching sentence: the tribe panel title aggregates both lanes across the whole tribe,
  so a fully collapsed panel still reports running and completed monitored work.

## Tests

### `tests/ace/tui/test_agent_panel_titles.py`

Follow the file's existing conventions: keyword `AgentPanelCounts(...)` construction,
exact `title.plain` assertions, and the `_assert_title_span` /
`_assert_title_range_style` helpers from `._agent_panel_title_helpers`. Import
`MONITOR_GLYPH_COLOR` alongside the existing `MONITOR_SETTLED_GLYPH_COLOR` import and
assert against the token, never a hex literal.

1. Both lanes with a chip:
   `AgentPanelCounts(running=1, waiting=2, running_monitors=1, settled_monitors=2)` →
   `"@chop · 3 [R1 W2] ⚙1 ⚙2"`. This is the ordering test — amber first.
2. Running lane only, with a chip: `"@chop · 3 [R1] ⚙1"`.
3. Running lane only, empty chip (all metrics zero, nonzero total): `"@chop · 3 ⚙1"`.
4. `running_monitors=0` with `settled_monitors=2` still renders exactly
   `"@chop · 3 [R1] ⚙2"` — the byte-for-byte regression guard for inserting a block
   ahead of the settled badge.
5. Styles in the both-lanes case: the `⚙1` span is `f"bold {MONITOR_GLYPH_COLOR}"`, the
   `⚙2` span is still `MONITOR_SETTLED_GLYPH_COLOR`, and the separator space before each
   is `_PANEL_COUNT_STYLE`. Locate the two badges by index rather than by searching for
   `"⚙"` (both badges start with the same glyph).
6. `selected=True` → `"❖ @chop · 3 [R1] ⚙1 ⚙2"`, with the bracket/metric letter taking
   `_PANEL_SELECTED_CHROME_STYLE` while `⚙1` stays bold amber and `⚙2` stays grey.
7. `collapsed=True` → `"▸ @chop · 3 [R1 W2] ⚙1 ⚙2"` — the collapsed panel is the point
   of the feature.
8. `merge_tribe_panels=True` → `"All agents · 3 ⚙1 ⚙2"`.
9. `agent_panel_counts` is fold-independent for the running lane: build a family
   container whose `runtime_children` / `followup_agents` hold a `state="running"`
   monitor, pass **only** the container (no monitor row in the list, simulating a
   collapsed fold), and assert `running_monitors == 1` and `settled_monitors == 0`.
10. Extend `test_agent_panel_counts_does_not_double_count_clan_and_family_rows`: give
    the same family root a second, `running` monitor and assert
    `(counts.running_monitors, counts.settled_monitors) == (1, 1)` with both the clan
    container and the family root in the slice. Proves the shared dedupe covers both
    lanes, not just the settled one.
11. Extend `test_panel_settled_monitors_matches_sum_of_container_row_badges` (rename it
    to `test_panel_monitor_lanes_match_sum_of_container_row_badges`) so it also compares
    `counts.running_monitors` against the sum of `monitor_lane_counts(root).running`
    over the same containers. Its fixture already includes a running monitor, so this is
    the cross-surface drift guard for both lanes.
12. Extend `test_panel_counts_use_lanes_for_total_and_statuses` with
    `assert counts.running_monitors == 0` (its only monitor is `completed`), keeping the
    existing `sum(metric_items()) == lane_count` assertion intact — the disjoint-status
    invariant must still hold with the new field present.

### `tests/ace/tui/test_agent_panel_title_refresh.py`

13. Add `test_running_monitor_badge_is_panel_scoped`, modeled on the existing
    `test_settled_monitor_badge_is_panel_scoped`: give the `@apple` family **two**
    monitors — one `monitor_state="running"` and one `monitor_state="completed"` — and
    assert the `@apple` panel title is `"@apple · 1 [R1] ⚙1 ⚙1"` while `@banana` stays
    `"@banana · 1 [R1]"`. This is the end-to-end proof that ordering and panel scoping
    both hold through `_refresh_panel_widgets`, not just through the pure title builder.

### Visual snapshots (`just test-visual`)

14. `tests/ace/tui/visual/snapshots/png/agents_settled_monitor_lane_badge_120x40.png`
    **must** be regenerated with `--sase-update-visual-snapshots`. Its fixture
    (`settled_monitor_family_agents`: `mon1` running, `mon2` completed, `mon3` failed,
    `mon4` stopped) gains the amber badge. In
    `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py`, update the exact
    title assertion from `"⌂ @default · 1 [R1] ⚙3"` to `"⌂ @default · 1 [R1] ⚙1 ⚙3"`;
    the existing `assert_page_svg_contains(page, "⚙1")` / `"⚙3"` lines still hold.
    Inspect the diff under `.pytest_cache/sase-visual/` and confirm the only change is
    the new title badge.
15. `tests/ace/tui/visual/snapshots/png/agents_family_conversation_monitor_120x40.png`
    **must not** move: its only monitor is `completed`, so `running_monitors == 0` and
    its title is unchanged. Its `assert "⚙1" in …border_title` assertion still passes on
    the grey badge. If this golden moves, stop — the zero-suppression rule is broken.
16. No other agents-tab visual fixture contains an `is_monitor` row: the config-center
    procs helper's `_monitor_visual_agent` sets `monitor_id` but neither
    `agent_family_role` nor `role_suffix`, so `Agent.is_monitor` is `False` and it
    contributes to neither lane. If any golden other than the one in item 14 moves, stop
    and investigate rather than accepting it.

Keep the `settled_monitor_lane_badge` fixture, test, and golden **names** as they are.
The fixture always held one running and three finished monitors; renaming a golden for a
superset assertion is pure diff noise.

## Verification

- `just install` first (ephemeral workspaces drift on dependencies).
- `just check` for the whole-repo lint gates and the diff-scoped test lane.
- `just test-visual` to regenerate, then re-verify. Confirm the blast radius with
  `git status --short tests/ace/tui/visual/snapshots/png/` — exactly one file may
  appear.
- `just check-full` before landing (this touches a shared render path and regenerates a
  golden). Run it **only** through `/sase_monitor`
  (`sase monitor start --command 'just check-full' …`) with a `--next` action — never
  inline.
- Spot-check `tests/ace/tui/test_agent_left_panel_width.py`: a rendered badge grows the
  title by three cells and titles participate in panel width negotiation
  (`AgentList.update_border_title` → `_refresh_requested_width`). Those fixtures have no
  monitor rows, so they must be unaffected; a moved width assertion means the badge is
  rendering where it should not.
- Perf: no new traversal is introduced — `agent_panel_counts` already ran
  `panel_monitor_lane_counts` once per panel per title build and simply discarded
  `.running`. This tale must not change that cost. If
  `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` shows a regression, the traversal
  was accidentally duplicated; fix that rather than adding a cache.

## Out of scope

- Any change to `metric_items()`, `_PANEL_METRIC_LABELS`, or `_PANEL_METRIC_STYLES`.
- `agent_family_members.py` — `panel_monitor_lane_counts` already returns both lanes and
  needs no edit.
- Container-row, monitor-row, and clan-row rendering; the Procs tab; the top-bar gear
  chips; and `proc_gear_chips.gear_chip`.
- The selected-panel `TRIBE` header in the prompt panel, its `Composition` field, and
  the `Z` tribe metadata zoom document.
- `sase stats` and tribe/clan summaries, which by documented design do not count
  monitors as agents.
- Renaming the `settled_monitor_lane_badge` fixture, test, or golden.
