---
tier: tale
title: Render the agent detail Wait field as aligned, tagged lanes
goal: The Agents-tab metadata panel shows each wait dimension on its own line, led
  by a bracketed category tag in a padded gutter, with values aligned in one column
  and long lists wrapping with a hanging indent instead of reflowing to column 0.
create_time: 2026-07-28 09:26:10
status: wip
---

- **PROMPT:** [202607/prompts/wait_field_lanes.md](prompts/wait_field_lanes.md)

# Redesign the agent detail `Wait:` field as aligned, tagged lanes

## Problem

In the Agents-tab metadata panel, `_append_wait_field`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py:229`) renders every wait dimension as one
run-on line joined by `+`:

```
Wait: sase-ae.1 ✓, sase-ae.2 ✓, sase-ae.3 ▶, sase-ae.4 ✓ + beads: sase-ae.1 ✓, sase-ae.2 ✓,
sase-ae.3 ▶, sase-ae.4 ✓
```

Three defects:

1. **The leading list is unlabeled.** Agent names get no category marker, while beads get a lowercase `beads:` prefix
   that reads as prose rather than a column.
2. **Agent names and bead IDs are indistinguishable.** Both are `sase-ae.N`, so the `+ beads:` boundary is the only
   signal, and it disappears the moment the line soft-wraps.
3. **Soft wrap reflows to column 0.** The header `Text` wraps with no hanging indent, so continuation content lands
   under `Wait:` and reads as a new field.

## Design

One **lane per wait dimension**, each with a bracketed category tag in a padded gutter, values aligned in a single
column:

```
Wait: [agents] sase-ae.1 ✓, sase-ae.2 ✓, sase-ae.3 ▶, sase-ae.4 ✓
      [beads]  sase-ae.1 ✓, sase-ae.2 ✓, sase-ae.3 ▶, sase-ae.4 ✓
```

Four lane tags, emitted in the same order the current `+` chain uses, so this is a pure re-layout with no semantic
reordering: `agents` → `beads` → `time` → `runners`.

### Decisions

- **Always tag, even for a single lane.** `Wait: [time] 5m` is self-describing and keeps one grammar. Consistency is
  what makes the column learnable.
- **Tag gutter width is the widest tag actually present, plus one space.** Only `[agents]` + `[beads]` present →
  width 9. A lone `[time]` → width 7. This stays compact in the common case and perfectly aligned in every case. The
  width changes only when the _set_ of lanes changes, which is a real state change worth seeing.
- **Continuation lines align under the value column, not column 0.** Achieved by rendering the field as a responsive
  section (below), the same mechanism the `BEAD` lane already uses for its wrapped `Description:` value.
- **Tag hue carries the category; values stay uniform.** All tags are `dim`; each reuses a color this codebase already
  attaches to that concept, so nothing new is invented. Values keep `_WAITING_VALUE_STYLE` (`#FF87D7`) across all lanes,
  so the gutter answers "what kind of thing" and the value column stays a single scannable stream.
- **`[runners]` drops its redundant leading noun.** `runners ≤ 0` → `≤ 0`; `runners: 10/10 in use` → `10/10 in use`;
  `runners: cap 11` → `cap 11`. The trailing `· 3 runners still running` clause keeps its noun (it counts a different
  set than the threshold).
- **The `QUEUED` branch is untouched.** It renders a different field (`Queue:`) with a single dimension and returns
  early; lanes would add nothing.

### Rendering mechanism

The field becomes `ResponsiveWaitSection`, following the established `ResponsiveBeadSection` / `ResponsivePlanSection` /
`ResponsiveSlowToolCallsSection` pattern:

- The builder appends the section's `logical_text` into the header `Text` and records `(start, end)` in
  `responsive_ranges` under key `"wait"`.
- `build_header_text` passes the section to `AgentHeaderRenderable`, which swaps the live renderable in at paint time
  while `.plain` keeps the full logical text for tests, search, and inspection.
- `__rich_console__` renders a three-column `Table.grid`: field-label column (`no_wrap`, width `len("Wait: ")`), tag
  column (`no_wrap`, gutter width), value column (`overflow="fold"`). Rich then wraps long values with the continuation
  aligned under the value column.
- No max-width cap. `BEAD` caps at 80 because prose reads badly when wide; comma-separated dependency lists do not, and
  fewer wrapped lines is better here.

`logical_text` must be byte-identical to the unwrapped table: `Wait: ` on row 0, six spaces on later rows, tags padded
to the gutter width, no trailing spaces after values.

## Implementation

### 1. New module `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py`

Presentation-only; no I/O, no disk reads (render paths must not stat or glob).

```python
WAIT_SECTION_ID = "wait"
WAIT_FIELD_LABEL = "Wait: "
WAIT_FIELD_LABEL_STYLE = "bold #87D7FF"   # unchanged from today
```

Lane tag styles — each reuses an existing meaning-bearing color:

| tag         | style         | why that color                                                 |
| ----------- | ------------- | -------------------------------------------------------------- |
| `[agents]`  | `dim #AF87FF` | the wait/`Waiting` purple already used for wait context        |
| `[beads]`   | `dim #FFAF00` | the `Bead:` value color in `_append_identity_fields`           |
| `[time]`    | `dim #87D7FF` | the metadata field-label cyan used for timestamps              |
| `[runners]` | `dim #5F87FF` | `QUEUED_STATUS_COLOR`, which already means "runner slot queue" |

Public surface:

- `build_wait_lanes(agent, *, agent_status_buckets, clan_wait_member_statuses, wait_bead_statuses, runner_queue_ahead_count) -> tuple[tuple[str, Text], ...]`
  — pure function returning `(tag, value_text)` pairs in lane order, empty when the agent has no wait state. Move the
  existing per-lane value construction here verbatim except for the `[runners]` noun change; keep
  `_append_wait_status_badge` / `_append_wait_bead_status_badge` behavior (including the `?` unknown-target glyph and
  the `None` status-map degradation) exactly as-is.
- `wait_gutter_width(lanes) -> int` — `max(cell_len(f"[{tag}]") for tag, _ in lanes) + 1`.
- `@dataclass(slots=True) ResponsiveWaitSection` holding `lanes`, with a `logical_text` property and `__rich_console__`
  as described above.

The `(Xm left)` countdown from `wait_remaining_seconds` stays attached to the `[time]` lane value.

The clan expansion (`sase-7g (all clan members · 2/4 done: .f0 ✓ · …)`) stays inside the `[agents]` lane value and now
benefits from hanging-indent wrapping.

### 2. `_agent_display_header_metadata.py`

- `_append_wait_field` keeps the `QUEUED` early-return branch unchanged, then delegates to `build_wait_lanes`. When
  lanes exist it appends `section.logical_text`, records `responsive_ranges[WAIT_SECTION_ID] = (start, len(text))`
  (range **includes** the trailing newline, matching how `_agent_context.py:171` records `BEAD`/`PLAN`), and returns the
  section; otherwise returns `None`.
- `append_agent_metadata_fields` gains a `responsive_ranges: MutableMapping[str, tuple[int, int]] | None = None` keyword
  and returns `tuple[list[tuple[str, str]], ResponsiveWaitSection | None]`. This mirrors
  `append_slow_tool_calls_section`, which already both returns its section and writes into `responsive_ranges`. It has
  exactly one production caller and no direct test callers.

### 3. `_agent_display_header.py`

- Create `responsive_ranges: dict[str, tuple[int, int]] = {}` **before** the `append_agent_metadata_fields` call
  (currently created at line 239, after it) and pass it in; unpack the new tuple return.
- Register the wait section in `responsive_sections` alongside `BEAD` / `PLAN` / `slow-tool-calls`. The existing
  `responsive_sections.sort(key=...)` keeps document order correct even though the wait range precedes the others.

**Consequence to verify, not to fear:** the header now returns `AgentHeaderRenderable` instead of a bare `Text` in many
more cases, including the `cheap=True` first-paint path. Every consumer is already typed against the
`AgentHeader = Text | AgentHeaderRenderable` union (`_agent_display_render.py`, `_agent_display.py`,
`_agent_display_family_render.py`, `_agent_display_step_render.py`, `_agent_display_hints.py`) and uses only `append` /
`append_text` / `stylize` / `plain` / `end`, all of which `AgentHeaderRenderable` forwards. Confirm no consumer calls a
`Text` method the wrapper lacks; add the forwarder if one is found.

Known and accepted limitation, identical to the pre-existing `BEAD` / `PLAN` behavior: styles applied post-hoc to a
range inside a responsive section are not visible, because the section renders from its own data rather than from the
logical text. Nothing currently restyles the wait range.

## Tests

### Update existing expectations

- `tests/ace/tui/widgets/test_agent_display_waiting_warning.py` — add a `_wait_block` helper that returns the `Wait: `
  line plus every following line beginning with six spaces, and rewrite the wait assertions:
  - `Wait: dep ✓` → `Wait: [agents] dep ✓` (and each parametrized glyph)
  - `Wait: ghost_deploy ?` → `Wait: [agents] ghost_deploy ?`
  - `Wait: coder ✓, ghost_deploy ?, reviewer ✗` → `Wait: [agents] coder ✓, ghost_deploy ?, reviewer ✗`
  - `Wait: ghost_deploy` (no status map) → `Wait: [agents] ghost_deploy`
  - `Wait: coder ✓ + beads: sase-87.2, sase-87.3` → `Wait: [agents] coder ✓` / `      [beads]  sase-87.2, sase-87.3`
  - `Wait: coder ✓, deploy ▶ + 5m` → `Wait: [agents] coder ✓, deploy ▶` / `      [time]   5m`
  - `Wait: coder ✓ + until …` → `[agents]` lane plus a `[time]` lane starting `until ` and ending ` left)`
  - both clan-expansion cases → same text under `[agents]`, with the `+ 5m` case moving to a `[time]` lane
  - `Wait: archived-clan ?` → `Wait: [agents] archived-clan ?`
- `tests/ace/tui/widgets/test_agent_display_wait_bead_statuses.py` — bead-only waits become `Wait: [beads] sase-9r.2 ✓`
  (gutter width 8, single lane); `Wait: coder` → `Wait: [agents] coder`.
- `tests/ace/tui/widgets/test_agent_list_status_indicators.py` — update `Wait: dep + 5m` (two places), `Wait: 5m (`,
  `Wait: parent + 2m`, `Wait: until `, and the two runner-slot assertions to
  `Wait: [runners] ≤ 0 (drain barrier) · 3 runners still running · queue #2 of 2` and
  `Wait: [runners] 10/10 in use · queue #2 of 3 · priority 20`. Leave every `format_agent_option` list-row assertion
  untouched — list rows are not part of this change.
- `tests/ace/tui/widgets/test_agent_queue_section.py` — `Wait: runners ≤ 0 (drain barrier)` →
  `Wait: [runners] ≤ 0 (drain barrier)`.

### New tests

Add `tests/ace/tui/widgets/test_agent_wait_section.py`:

1. **Gutter alignment** — with agents + beads lanes, both value columns start at the same character offset, and the
   continuation lines start with exactly six spaces.
2. **Gutter width tracks present lanes** — agents+beads yields width 9; a lone `[time]` lane yields width 7 (no
   over-padding).
3. **Lane order is agents → beads → time → runners** — an agent with all four wait dimensions renders exactly four lanes
   in that order.
4. **Rendered wrap keeps the hanging indent** — render the section through a `rich.console.Console` at a narrow width
   (e.g. 60) with a long dependency list and assert every continuation line is blank through the value column, i.e. the
   wrapped text never starts at column 0.
5. **`logical_text` equals the unwrapped render** — rendering at a width wide enough to avoid wrapping produces the same
   lines as `logical_text`.
6. **Tag styles are per-category** — the `[beads]` tag span carries `dim #FFAF00` and the `[agents]` tag span carries
   `dim #AF87FF`.
7. **No wait state renders no lanes and no `Wait:` label** — `build_wait_lanes` returns `()` and the header omits the
   field.
8. **`QUEUED` still renders the single-line `Queue:` field** and no `Wait:` label.
9. **`responsive_ranges["wait"]` covers exactly the logical wait block**, including its trailing newline, and slicing
   the header at that range round-trips (prefix + block + suffix reconstructs `.plain`).

### Visual snapshots

`tests/ace/tui/visual/test_ace_png_snapshots_agents_zoom.py` renders the wait detail in
`agents_waiting_missing_target_row_120x40` and `agents_waiting_unknown_zoom_modal_120x40`. Both goldens change.

Run `just test-visual --sase-update-visual-snapshots`, then **inspect the regenerated PNGs** in
`tests/ace/tui/visual/snapshots/png/` and confirm the lanes are tagged and aligned before accepting them. Keep the
existing `assert_page_svg_contains` assertions and add `[agents]` to the missing-target case.

## Docs

Update the **Wait state** bullet in `docs/ace.md` (around line 2353): describe the tagged lanes, the four tags, the
aligned gutter, and the hanging-indent wrap. Keep the existing statements about badges, the `?` unknown marker, the
WAITING list-row marker, and the separate `Queue:` line — none of those behaviors change.

## Verification

```bash
just install
just check
```

Then re-render the two visual goldens and eyeball them as described above.

## Out of scope

- The compact single-line wait text on Agents-tab **list rows** (`_agent_list_render_agent.py`) — a row has one line and
  no room for a gutter.
- The `Queue:` field and the runner-queue ladder section.
- `WaitModal` prefill text and the `%wait` xprompt grammar.
