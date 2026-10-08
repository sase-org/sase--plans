---
tier: tale
title: Light up in-flight targets in the Agents jump panel
goal:
  Running and starting agents, sessions, and turns stand out in the Agents-tab jump
  panel as lit key caps in every view (collapsed, narrowed, expanded), and the collapsed
  legend keeps them visible first when it overflows.
size: medium
proposed_by: bbugyi200.athena.0xv
create_time: 2026-10-08 09:44:23
status: wip
---

# Plan: Light Up In-Flight Targets In The Agents Jump Panel

## Goal

Make running and starting agents, sessions, and turns stand out in the Agents-tab jump
panel (the sticky footer below the deck panels). These are the agent relation jump
targets users are most likely to press. The result should be intuitive (it means one
thing everywhere), reliable (it derives from data the panel already renders), and
beautiful (calm, static, and in the existing palette).

Today every collapsed legend cell looks the same apart from a small trailing status
glyph: `[01] member01 ▶`. When the collapsed legend overflows (`+9`), the running
targets are often hidden while finished ones fill the two visible rows.

## Design

### 1. One rule: a target is lit when its glyph is ▶ or ◐

A jump target is **in-flight** when its displayed status bucket is in
`IN_FLIGHT_STATUS_BUCKETS` (`{"Running", "Starting"}`,
`src/sase/agent/status_buckets.py`). That is exactly the set of targets whose legend
glyph is `▶` (Running) or `◐` (Starting), so the highlight can never contradict the
glyph the user sees. Container targets (tribe-roster clans and sessions) follow their
displayed aggregate bucket, so they are lit exactly when they show `▶`/`◐`. Dismissed
targets (`role == "dismissed"`) are never lit.

Terminology: call these targets **in-flight** or **lit**, never "live". The glossary
already uses "live" to mean "currently valid" jump targets ("the jump panel ... lists
every live agent relation jump target"), and reusing the word would be confusing.

### 2. The lit pill: the number chip becomes a lit key

An in-flight cell turns its number chip into a two-tone **key cap**. The accent chip
stays exactly as it is (section accent, `bold black on <accent>`), and directly after it
a softly tinted **pill** carries the name and glyph:

```
unlit:  [01] member01 ▶          (unchanged from today)
lit:    [01]▓ member01 ▶ ▓       (▓ = tint background; the chip and pill touch)
```

- In the collapsed and narrowed legend cell, the pill is the separator space after the
  chip, then the label, a space, the status glyph, and one trailing pad space. All of
  these carry the tint background. Because the pill starts on the separator space, the
  chip and pill join into one key: "this lit key is the one to press".
- Label style inside the pill: `bold #FFD700` (the house agent-name gold, bolded) on the
  tint. Glyph style: its existing `member_status_style(bucket)` color on the same tint.
- Tints are the bucket's status color blended about 22% over the detail column's
  background (`#101010`), defined as named constants with that derivation in a comment:
  - Running (`#FFD700`): `#453C0D` (dark olive gold)
  - Starting (`#87D7FF`): `#2A3C45` (dark slate blue)
- Unlit cells (Done, Failed, Stopped, Waiting, Queued, dismissed) render **byte-for-byte
  as today**. The contrast comes from lighting the in-flight targets, not from dimming
  the others. Failed and needs-input targets stay fully legible.
- The section accent chip colors are unchanged, so the title's color legend
  (`JUMP · CLAN MEMBERS 00–13`) still maps chip colors to sections.

### 3. The same pill in every jump-panel view

The lit pill appears in every view of the jump panel, so pressing `.` or typing a first
digit never changes how an in-flight target looks:

- **Collapsed legend** (`JumpLegendRenderable` collapsed mode).
- **Narrowed legend** (after the first digit of a two-key jump).
- **Expanded roster** (`.`): numbered roster rows built by
  `append_member_roster`/`_append_numbered_entry` in
  `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`. For a lit row, the
  separator space after the `01` chip, any marked or unread prefix glyphs, and the label
  carry the tint, plus one trailing tinted pad. The label is bold. The rest of the row
  (`· kind · ▶ RUNNING · model · duration …`) is unchanged; its status is already gold.
  Unnumbered child rows are never lit because they are not jump targets.

### 4. In-flight-first packing in the collapsed legend

The collapsed legend shows at most two packed rows. When every target fits, nothing
changes: all targets show in number order. When the legend overflows, it picks which
targets to show by priority instead of always showing the lowest numbers:

1. In-flight targets first, lowest number first.
2. Then the remaining targets, lowest number first, until the slots are full.
3. The chosen cells are then **displayed in ascending number order**, so the legend
   still reads as one ladder, matching the narrowed and expanded views. Gaps in the
   numbers show that targets are hidden, and the number chips keep each target
   unambiguous.

The existing budget and column search (`_collapsed_lines`, `_largest_collapsed_columns`)
must evaluate the prioritized selection for each candidate column count, because the
chosen cells determine the widths. Label truncation budgets and the uniqueness check
stay computed over all targets, as they are today.

The narrow fallback (`best is None`, a single clamped cell) shows the first in-flight
target when one exists, else target 0.

### 5. The overflow chip reports hidden in-flight targets

The overflow chip stays `+N` (dim) for the number of hidden targets. When `M > 0`
in-flight targets are still hidden (only possible when there are more in-flight targets
than slots), it gains a small lit pill: `+9 ▶2`, with `▶2` rendered `bold #FFD700` on
the Running tint, using the Running tint even when some hidden targets are Starting.
This tells the user that more running targets are behind `.` without adding any panel
chrome.

### 6. Deliberately unchanged

- The panel title, border, border accent, subtitle (`▴ . more` / `▾ . less` /
  `esc cancel`), and visibility rules.
- Numbering, the `MemberJumpMap`, jump/revive semantics, and stale-jump cancellation.
  This is presentation only. Per the Rust core boundary note, it stays in this repo; no
  `sase-core` change is needed.
- No animation (no spinner, pulse, or shimmer). Nothing else on the Agents tab animates,
  and motion in a footer people glance at all day is distracting. A timer would also add
  repaint churn on the jump panel (TUI perf rules: idle ticks skip unchanged surfaces)
  and make PNG goldens nondeterministic. A static lit pill gives the "this one is alive"
  signal without motion.
- No new config option or feature flag: this is a pure presentation improvement with no
  backward-compatibility branch to keep.

### 7. Reliability

- Lit state comes only from `_MemberJumpTarget.status_bucket` (legend) and the entry's
  `effective_bucket or status_bucket_for_values(status)` (roster). These are the same
  values that already drive the glyph and the status text, so the pill updates on the
  same refresh that updates the glyph. There is no new data path, I/O, or cache.
- `JumpLegendRenderable.plain` already includes each target's `status_bucket`, so the
  panel's digest changes, and the panel repaints, exactly when a target flips between
  lit and unlit. Unchanged maps still skip repaint. Add a regression test for this.
- The packing work is O(targets) per candidate column count (targets ≤ 100), and it is
  memoized per width by the existing `_layout_cache`. No render-path I/O.

## Implementation

### Step 1: Shared in-flight helpers (single source of truth)

Create `src/sase/ace/tui/widgets/prompt_panel/_member_in_flight.py`. It lives in
`prompt_panel` so both the roster and the legend can import it without a cycle. Keep
`_member_roster.py` (currently 605 lines) well under the 700-line `toobig` tier. The
module exports:

- `is_in_flight_jump_bucket(bucket: str | None, *, dismissed: bool = False) -> bool`,
  backed by `IN_FLIGHT_STATUS_BUCKETS`.
- `in_flight_tint(bucket: str) -> str`, the tint constants described above (Running tint
  as the fallback).
- `in_flight_label_style(bucket: str) -> str` → `"bold #FFD700 on <tint>"`.
- `in_flight_glyph_style(bucket: str) -> str` →
  `f"{member_status_style(bucket)} on <tint>"`. Avoid an import cycle with
  `_member_roster.member_status_style`: either move the status-style table into this
  module and re-export it from `_member_roster`, or pass the base style in.
- A pill-padding helper if it keeps the call sites tidy.

### Step 2: Legend rendering (`src/sase/ace/tui/widgets/_agent_jump_legend.py`)

- `_build_cell`: when the target is in-flight and not dismissed, render the lit pill
  (tinted separator, bold label on tint, tinted space and glyph, trailing tinted pad).
  Otherwise output is identical to today.
- Add a pure helper, for example `_collapsed_selection(targets, shown) -> list[int]`,
  that implements the in-flight-first priority and returns ascending indices. Use it in
  both `_largest_collapsed_columns` and `_collapsed_lines` in place of `range(shown)`.
  Pass the per-target accent by the original index (`_target_accent(jump_map, index)`),
  not by slot position.
- Overflow chip: build it with a helper that appends the lit `▶M` pill when hidden
  in-flight targets remain. Use the same helper in both functions so the fitting width
  matches the rendered width.
- Narrow fallback: prefer the first in-flight target.
- `_narrowed_lines`: unchanged apart from `_build_cell` now lighting in-flight cells.

### Step 3: Expanded roster (`src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`)

- In `_append_numbered_entry`, compute the entry bucket (as the jump target does) and
  whether it is lit (`not entry.is_dismissed` and in-flight).
- For lit entries, tint the separator space after the chip, and pass a pill style into
  `_append_member_fields` (a new keyword, default `None`). That function then applies
  the tint background to the marked/unread prefix glyphs and the bold label, and appends
  one trailing tinted pad before ` · kind`. Child rows always pass `None`.

### Step 4: Unit tests

In `tests/ace/tui/widgets/test_agent_jump_legend.py`, extend the `_entries` helper with
a per-label status (or bucket) parameter. Today every fixture is `RUNNING`, so existing
exact-text expectations will shift by the pill pad. Update those deliberately rather
than loosening them. Add tests for:

- A lit cell's label and glyph spans carry the Running tint background. A Starting
  target uses the Starting tint. Done, Failed, Stopped, and Waiting cells have no
  background spans and the same plain text as before.
- Dismissed targets are never lit, even if their bucket is Running.
- In-flight-first packing: 14 alternating Done/Running targets at a width that fits six
  slots show the lowest in-flight numbers in ascending order plus `+N ▶M`. With fewer
  in-flight targets than slots, the remaining slots fill with the lowest-numbered
  others, still ascending. When everything fits, the order and contents equal plain
  number order.
- The overflow chip shows `▶M` only when `M > 0`. The rendered line width never exceeds
  the available width at any of the existing `WIDTHS`.
- The narrowed mode lights in-flight candidates.
- Digest/repaint: `JumpLegendRenderable(...).plain` (and the panel's `show_jump_map`
  digest) changes when one target flips Running → Done and stays equal for an identical
  map.
- The narrow fallback prefers the first in-flight target.

In `tests/ace/tui/widgets/test_member_roster.py`, a numbered in-flight entry's label
carries the pill style and trailing pad. A non-in-flight entry, a dismissed entry, and
child rows carry none. Marked and unread prefixes on a lit entry sit inside the pill.

Keep new test files and helpers under the `toobig` tiers. Split into a new
`test_agent_jump_legend_in_flight.py` if the legend test file would grow past about 450
lines.

### Step 5: Visual goldens

- Add `test_jump_panel_collapsed_in_flight_mix_png_snapshot` to
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py`. Use a 14-member
  clan mixing `STARTING`, a few `RUNNING`, `DONE`, `FAILED`, `QUESTION`, and `WAITING`
  members, with fewer in-flight targets than collapsed slots. Interleave the in-flight
  members past the first slots so the golden shows both tints, the fill behavior, and
  unlit neighbors. Golden name: `agents_jump_panel_collapsed_in_flight_mix_120x40`.
  Assert in the test (via the panel's legend) that the shown numbers are the expected
  prioritized set.
- The existing `agents_jump_panel_collapsed_two_digit_overflow_120x40` fixture (7
  running of 14, alternating) becomes the showcase of in-flight-first packing plus the
  `+N ▶M` chip. Update its assertions to check that the running members are the ones
  shown.
- Regenerate the goldens. First run a targeted pass:
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py`.
  Then run a full `just fix-tui-screenshots` pass, because many other Agents goldens
  (clans, neighbors, session panels, tribes, monitors) show a jump panel with running
  targets. Run these through `/sase_monitor` with `TESTING` / `TESTED`, and give the
  full run a generous timeout. Inspect the retained report and every golden change
  before finalizing. The only expected differences are lit pills on in-flight targets
  and the changed collapsed selection and overflow chip. Treat any other difference as a
  bug. Check for a `partial` status per the lint-and-test note.

### Step 6: Docs

Update `docs/ace.md` in the agent-jump paragraph (near "Every live numbered target lives
in the sticky jump panel…") and the jump-panel `.` key description if helpful. Add a
short paragraph:

> In-flight targets, whose glyph is `▶` (running) or `◐` (starting), are lit: their
> number chip runs into a softly tinted pill holding the bold name and glyph. This
> appears in the collapsed legend, the narrowed candidates, and the expanded roster
> alike. When the collapsed legend cannot show every target, it fills its slots with
> in-flight targets first (still in number order), then the lowest remaining numbers.
> The `+N` overflow count gains a lit `▶M` when `M` in-flight targets remain hidden
> behind `.`.

Grep `docs/` for other descriptions of the collapsed legend (for example
`docs/agent_families.md`) and keep them consistent. No memory-note change is needed. The
glossary defines jump targets, not their styling. No keymap or
`src/sase/default_config.yml` change.

## Verification

- `sase tool run check` (run `just fix` first) passes.
- The targeted and full visual golden runs are clean or applied, with every changed
  golden inspected as described above.
- Optional live sanity check with `sase screenshot` on a clan with running members: the
  collapsed legend shows lit key caps for running members, and `.` shows the same pills
  in the expanded roster.

## Out Of Scope

- Distinct treatment for needs-user targets (Stopped: plan review or question) or failed
  targets. The same `_member_in_flight` pattern could later host an "attention" style,
  but this plan lights only in-flight targets.
- Changing the agent-list rows, the identity header, or any other number-chip surface
  outside the jump panel.
