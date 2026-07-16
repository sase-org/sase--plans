---
tier: tale
title: Merge SASE PLAN into SASE CONTEXT as a PLAN lane
goal: 'The Agents-tab detail panel presents an agent''s plan as the leading PLAN lane
  inside SASE CONTEXT rather than a separate SASE PLAN major section, so every input
  that informs an agent reads as one ranked list, the lane header absorbs the redundant
  tier and phase-count rows, and the plan path keeps a hint number that ascends in
  visual reading order.

  '
create_time: 2026-07-16 17:09:11
status: wip
prompt: 202607/prompts/plan_lane_in_sase_context.md
---

# Plan: Merge SASE PLAN into SASE CONTEXT as a PLAN lane

## Product context

The Agents-tab detail panel currently renders two adjacent gold-underlined major sections that answer the same user
question — _what is this agent working from?_

```
SASE PLAN
Title: Hint-selected agent panel folding
Goal:  Agents-tab users can press ,H, select one or more numbered panel hints ...
Tier:  tale
Path:  sase/repos/plans/202607/hint_selected_agent_panel_folding.md

──────────────────────────────────────────────────

SASE CONTEXT
▸ MEMORY · 2 reads · 1 file · 2 agents
  16:56:32  coder  ◇ tui_perf.md  ↩ frontmatter
            ↳ Need TUI performance and responsiveness guidance before ...
▸ WORKSPACES · 1 open · 1 repo
  16:56:40  coder  ▣ plans  → …/plans
            ↳ Read the user-approved implementation plan for ...
```

They are also physically far apart: `SASE PLAN` renders immediately after `Timestamps:`, while `SASE CONTEXT` renders
near the bottom, after output variables, commits, deltas, artifacts, and workflow variables. A user scanning the panel
reads the plan, scrolls past five unrelated sections, and then reads the rest of the agent's inputs.

The merge is semantically right, not just a layout tidy-up. `SASE CONTEXT` already means _the audited inputs that inform
this agent_. An agent's plan is its primary input — the `WORKSPACES` lane in the screenshot above literally records
opening a repo **in order to read the plan**. Keeping them apart splits one idea across two sections.

After this change the panel reads as a single ranked answer:

```
SASE CONTEXT
▸ PLAN · tale
  Title: Hint-selected agent panel folding
  Goal:  Agents-tab users can press ,H, select one or more numbered panel hints with
         the TUI's familiar single/range syntax, and atomically toggle those panels
         between expanded and collapsed states with stable focus, durable state, and
         polished visual feedback.
  Path:  [7] sase/repos/plans/202607/hint_selected_agent_panel_folding.md

▸ MEMORY · 2 reads · 1 file · 2 agents
  16:56:32  coder  ◇ tui_perf.md  ↩ frontmatter
            ↳ Need TUI performance and responsiveness guidance before ...

▸ WORKSPACES · 1 open · 1 repo
  16:56:40  coder  ▣ plans  → …/plans
            ↳ Read the user-approved implementation plan for ...
```

## Design decisions

These are the choices a reviewer should push back on if they disagree; the rest is mechanical.

### 1. PLAN leads the lane order

Lane order becomes **PLAN → MEMORY → SKILLS → WORKSPACES**. The plan is the most important context an agent has, and
leading with it preserves the prominence the old section earned by rendering first. The order also narrates: _what am I
building_ (PLAN) → _what did I read_ (MEMORY) → _what did I use_ (SKILLS) → _where_ (WORKSPACES).

### 2. PLAN is a descriptive lane, not an event lane — and looks like it

This is the central design fork, and it is worth being explicit about because it is the one place the new lane
deliberately breaks the existing lane contract.

The three existing lanes are **event logs**: each row is `timestamp → role → glyph → primary → ↳ reason`, and
`tests/ace/tui/widgets/test_agent_context.py::test_context_rows_share_columns` enforces that they align on shared
columns. The plan is **not an event**. It has no meaningful moment, no actor, and no "reason" — it is a standing
description with structured fields and an optional phase roadmap.

Two options were considered and rejected:

- **Force the plan into the event-row idiom** (`16:56:04 ◈ <path>` with the goal as the `↳ reason`). This is false
  consistency. It invents a timestamp the plan does not have, discards `Title`/`Tier`, and leaves the phase roadmap
  homeless. The `PLAN | ...` entry in `Timestamps:` is the plan-phase _transition_, not "when the plan was read" —
  reusing it would be a lie in the UI.
- **Leave SASE PLAN where it is.** Rejected: it is what the user asked to change, and the semantic case for the merge is
  sound.

**Chosen:** PLAN is a _descriptive_ lane. It wears the same `▸ LABEL · details` header so it reads as a sibling, and its
body keeps the existing responsive field grid indented to two spaces so it lines up under the header exactly like event
rows do. The absence of a timestamp column is the deliberate tell that this lane describes rather than logs.
`test_context_rows_share_columns` keeps covering the three event lanes; the PLAN lane is explicitly exempt, and a
comment in the shared helper should say why so the next reader does not "fix" it.

### 3. The lane header absorbs the `Tier:` and `Phases:` rows

The header becomes the lane's summary line, matching how `▸ MEMORY · 2 reads · 1 file` summarizes its payload:

| Plan state          | Lane header                          |
| ------------------- | ------------------------------------ |
| Tale/plan           | `▸ PLAN · tale`                      |
| Valid authored epic | `▸ PLAN · epic · 3 phases`           |
| Known-invalid epic  | `▸ PLAN · epic · phases unavailable` |
| Unresolved tier     | `▸ PLAN · tier unavailable`          |

This drops two body rows as pure duplication and makes the lane two lines shorter in the common tale case. The tier
keeps its distinct per-tier color (`PLAN_TIER_STYLES`) inside the header, so no signal is lost — the color moves, it
does not disappear. Body rows reduce to **Title, Goal, Path**.

Consequence: `append_context_lane_header` must accept a styled `Text` for `details`, not just a `str`, since the tier
token is colored differently from the surrounding dim separators. Widen the shared helper to `str | Text` rather than
forking a second header function, so all four lanes stay on one code path.

### 4. The plan path's hint number moves, and that is the point

File hints (`[N]`) are allocated in strict render order today — timestamps, then plan, then commits, deltas, artifacts,
slow tools — so that a user reading down the panel sees ascending numbers. Moving the plan into `SASE CONTEXT` must move
its hint allocation with it, or the invariant breaks and the plan shows a number smaller than everything above it.

This requires threading `hint_state` into `append_agent_context_section`, which takes no hint state today. That is new
plumbing, not a rename, and it is the main reason this change is not a two-line move. The plan path remains the only
hint-bearing element of the lane.

### 5. `ResponsivePlanSection` gets simpler, and the splice mechanism is untouched

`_AgentHeaderRenderable` splices the live plan widget into the header via `plan_start`/`plan_end` character offsets so
the plan can reflow at render time (`Table.grid` folding at `PLAN_SECTION_MAX_WIDTH = 80`) while the rest of the header
stays a flat `Text`. **This mechanism is position-agnostic** — it only requires that the plan's logical text remain one
contiguous range. So it survives the move unchanged; only the offsets shift.

`ResponsivePlanSection` itself gets _smaller_: it stops emitting its own major divider and `SASE PLAN` heading (the
context section already emits one divider and one heading), and emits the lane header instead. Its dual `_heading()` /
`_heading(width=...)` divider paths collapse.

The one real constraint: `append_agent_context_section` takes a bare `Text` and cannot report offsets. It must return
the plan's `(start, end)` range (or `None`) so `build_header_text` can construct the renderable. Keep this explicit in
the return type rather than smuggling it through mutable state.

### 6. A plan alone justifies the section

`append_agent_context_section` currently early-returns unless memory/skills/workspaces exist, and its caller in
`build_header_text` guards on the same condition. Both must learn that an associated plan is sufficient. Miss either and
agents with a plan but no audited events lose their plan entirely — the most likely way to ship a regression here.

### 7. Accent color

PLAN takes `bold #AF87FF` (purple), distinct from MEMORY cyan `#5FD7FF`, SKILLS green `#5FD75F`, and WORKSPACES pink
`#FF87D7`. It echoes the existing phase glyph color, so the lane header and its phase roadmap rhyme. The old heading
color (`bold #D7AF5F underline`) is retired from the plan — that gold belongs to major section headings, and PLAN is no
longer one.

## Implementation outline

All of this is presentation-only Rich/Textual rendering and stays in this repo; it does not cross the Rust core
boundary. `agent_associated_plan.py` resolves the plan data and is **not** changed — this plan only moves and reshapes
rendering.

1. **`_agent_context_common.py`** — widen `append_context_lane_header` details to `str | Text`; add the PLAN accent
   color constant. Note in a comment why the PLAN lane has no timestamp column.
2. **`_agent_plan_section.py`** — replace the major-heading emission with the lane header (tier + phase-count summary);
   drop the `Tier:` and `Phases:` rows; indent body rows and the phase roadmap to sit under the lane header; keep the
   responsive grid and its 80-cell fold.
3. **`_agent_context.py`** — accept the optional `ResponsivePlanSection` and `hint_state`; render PLAN first; relax the
   empty guard; return the plan's logical `(start, end)` range.
4. **`_agent_display_header.py`** — remove the plan block after `Timestamps:`; build the `ResponsivePlanSection` and
   allocate its hint at the context-section call site; relax the caller's guard; construct `_AgentHeaderRenderable` from
   the range the context section returns.
5. **Docs** — `docs/ace.md:347-354` and `docs/ace.md:1540-1553` both assert the plan is "the first major section after
   ordinary metadata"; rewrite both for the lane. `README.md:191`'s lane inventory gains PLAN. While there:
   `docs/ace.md:1540-1553` lists the rows as `Goal`/`Tier`/`Path` and already omits `Title` — correct that stale text to
   the new `Title`/`Goal`/`Path` set rather than preserving the error.

## Testing

The ordering tests are contracts that must be **inverted**, not patched:

- `test_agent_display_plan_section.py::test_plan_is_first_major_section_in_maximal_append_flow` asserts PLAN precedes
  `SASE CONTEXT` and seven other sections. Rewrite it to assert PLAN now renders _inside_ `SASE CONTEXT`, still exactly
  once, and ahead of the three event lanes.
- `test_agent_display_plan_section.py::test_section_follows_timestamps_and_precedes_optional_major_sections` and
  `::test_plan_heading_is_immediately_followed_by_title` encode the old position and heading shape.
- `test_agent_display_artifact_metadata.py::test_full_header_renders_artifact_paths_after_deltas` asserts `SASE PLAN`
  precedes `Deltas:`; the plan now follows deltas.
- `test_agent_display_artifact_metadata.py::test_committed_plan_uses_workspace_relative_path_and_hint_mapping` asserts
  the plan path is hint `[1]`. It is `[1]` today only because the plan allocated first; the expected number changes.
  This is the sharpest regression canary — it must keep asserting that `file_hints[N]` resolves to the plan path, with
  `N` matching the plan's new reading-order position.
- `test_agent_context.py::test_context_lanes_render_in_parent_context_order` gains PLAN at the front;
  `::test_lane_subheaders_use_distinct_accent_colors` gains a fourth pairwise-distinct accent.
- `test_prompt_panel_header.py::test_header_renders_workflow_variables_before_agent_context` still holds and now
  transitively covers the plan's new position.

New coverage to add:

- A plan with **no** memory/skills/workspaces still renders `SASE CONTEXT` with only the PLAN lane (design decision 6).
- Event lanes with **no** plan omit the PLAN lane entirely — the negative case matching the existing per-lane omission
  trio.
- The lane header renders each tier/phase state from the decision-3 table, including `phases unavailable` for a
  known-invalid epic and `tier unavailable`.
- The existing fold/wrap tests (`test_agent_display_plan_section.py:417-556`) must keep passing at widths 32 and 120 —
  they are the guard that the responsive splice survived the move. If they regress to hard-wrapping, the `(start, end)`
  range was plumbed wrong.

Visual snapshots — regenerate with `--sase-update-visual-snapshots` after confirming the diffs are the intended reshape:

- `agents_plan_goal_metadata_120x40` and `agents_epic_phase_roadmap_120x40` (both in
  `test_ace_png_snapshots_agents_auto_approve.py`) are the only two snapshots rendering the plan. **Update their wait
  anchors first**, or the tests hang on a string that no longer exists instead of failing cleanly. Each waits on a
  different doomed string: the first on `wait_for_svg_contains(page, "SASE PLAN")`, the second on
  `wait_for_svg_contains(page, "Phases:")` — and this design deletes _both_ (the heading becomes a lane header; the
  phase count moves into it). Their `assert_page_svg_contains` lines assert the same two strings and need the same
  treatment.
- Add a snapshot showing PLAN and the event lanes together in one `SASE CONTEXT`, which no current snapshot covers.
  `agents_context_zoom_modal_120x40` is the natural sibling: its `_zoom_agent` fixture has no plan today, so give the
  new snapshot one.

Close with `just install && just check`, plus `just test-visual` for the snapshot suite.

## Risks

- **Silent hint-number drift.** Any test asserting a literal `[N]` for a post-plan section could shift. The audit found
  the others all seed their own `HeaderHintState`, so they should be immune — but verify rather than assume.
- **Losing the responsive fold.** If the plan's logical text is appended to the context `Text` without the offsets
  reaching `_AgentHeaderRenderable`, the plan still _renders_ — it just silently stops reflowing. Tests at width 32 are
  what catch this; do not skip them.
- **Snapshot churn masking a real regression.** Two PNGs will legitimately change. Inspect the diff artifacts in
  `.pytest_cache/sase-visual/` rather than blanket-accepting.
- **Vertical growth.** `SASE CONTEXT` becomes taller now that it carries the plan. Dropping the `Tier:` and `Phases:`
  rows offsets this; if the section still feels heavy in the 120x40 snapshot, the goal row is the candidate for a future
  clamp — explicitly out of scope here, since today's section wraps the goal in full and silently truncating it would be
  a regression.
