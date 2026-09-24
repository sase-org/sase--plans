---
tier: tale
title: Never fold agent jump targets out of the JUMP panel
goal:
  The expanded Agents-tab JUMP panel always lists and numbers every FAMILY SHELLS and
  NEIGHBORS target, with no fold-driven "+N more neighbors (zz to show more)" tail.
size: medium
proposed_by: bbugyi200.athena.0qg
create_time: 2026-09-23 21:12:28
status: wip
---

# Never Fold Agent Jump Targets Out Of The JUMP Panel

## Problem

On the Agents tab, the numbered `FAMILY SHELLS` and `NEIGHBORS` rosters are detached out
of the detail body and rendered only inside the sticky JUMP panel (toggled with `.`).
When that panel is expanded, the `NEIGHBORS` roster still gets truncated by the lane's
fold level. A user sees, for example, 3 of 11 neighbors followed by
`… +8 more neighbors (zz to show more)`. The hidden neighbors also get no jump digits,
so they cannot be jumped to at all.

The cap comes from `neighbor_entry_limit()` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_neighbors.py`. It maps the lane's
fold-scale position to 3 rows (first position), 10 rows (middle positions), or all rows
(last position). `build_header_text()` in `_agent_display_header.py` repeats the same
calculation to size the shared `MemberJumpNumbering`. That fold ladder was designed back
when the roster was rendered inline in the detail body. Since commit 78f1d71e7 moved
numbered rosters into the JUMP panel, `zz` folds a panel that no longer contains the
roster, and the expanded JUMP panel can never list every target. No other roster is
capped this way: FAMILY SHELLS, clan, and tribe rosters pass no `entry_limit`.

## Desired Behavior

Every lane neighbor is always rendered and numbered, whatever the lane fold level. In
practice:

- The expanded JUMP panel lists every FAMILY SHELLS and NEIGHBORS target, with no
  `… +N more neighbors (zz to show more)` tail.
- Jump digits come from one continuous ladder covering all targets. Numbering therefore
  does not depend on the fold level, and it is identical in the collapsed legend, the
  expanded roster, and the published `MemberJumpMap`. Toggling the panel or pressing
  `zz` never renumbers anything.
- The collapsed JUMP legend is unchanged. `JumpLegendRenderable._collapsed_lines`
  already packs at most two rows and ends with a `+N` overflow cell, so a longer map
  cannot make it grow.
- The existing `… +N also listed under FAMILY SHELLS` tail stays, because it reports
  rows deduplicated into FAMILY SHELLS, not rows hidden by folding.
- The only remaining hidden-row tail is the shared 100-slot numbering capacity
  (`MEMBER_ROSTER_LIMIT`). There it should use the generic default hint
  (`… +N more neighbors (not numbered)`) and never mention `zz`.
- Unchanged on purpose: the lane fold level still drives the NEIGHBORS heading fold
  glyph and each entry's annotation detail (activity, `wait …`, and the full
  timestamps). Those control how much detail each row shows, not which targets exist.

## Implementation

### 1. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_neighbors.py`

- Delete `NEIGHBORS_FIRST_LEVEL_LIMIT`, `NEIGHBORS_MID_LEVEL_LIMIT`, and
  `neighbor_entry_limit()`, and remove them from `__all__`. Drop imports this leaves
  unused (`fold_scale_position`, and possibly `effective_fold_level`).
- In `append_lane_neighbors_section()`, stop computing `neighbors_level` and
  `entry_limit`. Pass no `entry_limit` to `append_member_roster`, so it defaults to
  `None`. When `numbering is None`, size it as
  `MemberJumpNumbering(total=len(entries))`.
- Remove the `hidden_tail_hint="zz to show more"` argument so the capacity-only tail
  uses the default `"not numbered"` hint. Keep `hidden_tail_label="neighbors"`.
- Keep the `panel_level`, `scale`, and `section_fold_overrides` parameters. They still
  feed the heading glyph and each entry's annotation level inside
  `append_member_roster`.
- Update the module docstring if it still describes fold-aware row limits.

### 2. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`

- Remove the `neighbors_level` / `neighbors_limit` computation and the
  `neighbor_entry_limit` import. Set
  `shown_neighbor_count = 0 if lane_neighbors is None else len(lane_neighbors.rows)`.
  This keeps the shared `MemberJumpNumbering` width equal to the digits actually
  rendered: one digit up to 10 targets, two digits above that.
- Drop the `NEIGHBORS_SECTION_ID` import if nothing else in the module uses it.

### 3. Tests

- `tests/ace/tui/widgets/test_agent_display_neighbors.py`
  - Delete `test_neighbor_entry_limit_uses_lane_scale_position`.
  - Rewrite `test_neighbors_section_renders_fold_ladder_and_truthful_count` so it
    asserts that, at every level and scale in its parametrization, all 12 rows render
    and get numbers (`len(jump_map.targets) == 12`), the heading still reads
    `NEIGHBORS · 12`, and `more neighbors` / `zz` never appear.
  - Add a focused test showing that a projection too large for the numbering capacity
    (or a small explicit `MemberJumpNumbering(capacity=…)`) renders the `(not numbered)`
    tail, never the `zz` hint.
- `tests/ace/tui/widgets/test_summary_fold_lane_neighbors.py`
  - Replace the `neighbor_entry_limit` import. `_shown_neighbor_count` becomes
    `_LANE_NEIGHBOR_TOTAL` at every level. Rename and rewrite
    `test_lane_neighbor_rows_follow_the_positional_ladder` to assert that every level
    renders all 12 numbered rows with no `more neighbors` tail.
  - `test_lane_document_digits_match_the_published_jump_map` should then expect
    two-digit numbering at every level, since leading members plus 12 exceeds 10. Assert
    that numbering is identical across all fold levels.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py`
  - In `test_agents_lane_neighbors_section_fold_levels_png_snapshots`, the first-level
    jump map now holds all five neighbors (`list("01234")`). Assert that
    `more neighbors` is absent. After `zz`, assert that the jump map numbering is
    unchanged (fold no longer affects targets) while the lane fold level still changes.
    Keep pressing `4` to jump to `visual.other.bench`. Update the test's docstring and
    snapshot titles so they describe detail folding rather than row truncation.
- `tests/ace/tui/widgets/test_member_roster.py` passes its own `hidden_tail_hint` to the
  generic roster and needs no change. Leave it alone unless it breaks.
- Grep `tests/` for any other `more neighbors`, `zz to show more`,
  `neighbor_entry_limit`, or `NEIGHBORS_*_LIMIT` references and update them.

### 4. PNG goldens

The neighbor goldens under `tests/ace/tui/visual/snapshots/png/` will change, most
likely `agents_lane_neighbors_section_first_level_160x50`,
`agents_lane_neighbors_section_expanded_160x50`,
`agents_lane_neighbors_above_context_160x50`, `agents_family_lane_neighbors_160x50`,
`agents_neighbor_jump_expanded_panel_120x40`, and
`updates_indicator_with_neighbors_120x40`. Regenerate them with
`just fix-tui-screenshots` (targeted selectors after `--`), run through `/sase_monitor`
as the TUI screenshot memory prescribes. Inspect every golden change in the retained
report before finalizing. The expected differences are extra neighbor rows and digits in
the JUMP legend and roster, plus removed `… +N more neighbors (zz to show more)` tails.
Anything else is a regression to investigate.

### 5. Docs

- `docs/ace.md`, section "Sase Agent Neighbors Section" (the paragraph starting "The row
  count follows the sase agent's fold scale by position…"): replace it with the new
  behavior. Every neighbor always renders and gets a digit whatever the fold level. The
  fold level only changes the heading glyph and each row's annotation detail. Numbering
  is stable across fold levels and JUMP-panel toggles. The heading counts every
  neighbor. The FAMILY SHELLS dedup tail is unchanged.
- `docs/agent_families.md` (the sentence "The family's two-level scale drives the
  section too: level 1 shows the first three neighbors plus a hidden-count tail, and
  level 2 shows all of them."): replace it with a statement that every neighbor is
  always listed and numbered after the family members.

## Verification

- Targeted pytest runs, repeated until green:
  `tests/ace/tui/widgets/test_agent_display_neighbors.py`,
  `tests/ace/tui/widgets/test_summary_fold_lane_neighbors.py`,
  `tests/ace/tui/widgets/test_member_roster.py`, and the neighbor visual snapshot module
  through the TUI screenshot workflow.
- Read the `lint_and_test` memory and run `just check` as it directs. Symvision must not
  flag the removed exports, and nothing may keep importing them.
- Optional manual check: on a sase agent with more than 3 neighbors, open the Agents tab
  and press `.`. The expanded JUMP panel should list every neighbor with a digit and no
  `zz to show more` tail. Pressing `zz` should change only row annotations, never which
  digits exist.

## Out Of Scope

- Changing collapsed-legend packing, the `MEMBER_ROSTER_LIMIT` capacity, or the FAMILY
  SHELLS dedup rules.
- Any change to the Rust core. This is presentation-only roster rendering in the Python
  TUI.
