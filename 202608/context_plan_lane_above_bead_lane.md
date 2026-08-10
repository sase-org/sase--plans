---
tier: tale
size: small
title: Show the PLAN lane above the BEAD lane in SASE CONTEXT
goal:
  Render the SASE CONTEXT PLAN lane above the BEAD lane in the Agents metadata panel
  whenever both lanes are present, in both the single-agent panel and the aggregated
  clan panel, and keep every derived contract (hint numbering, responsive reflow, tests,
  docs, and PNG goldens) consistent with the new order.
proposed_by: bbugyi200.athena.xi
create_time: 2026-08-10 14:10:52
status: wip
---

# Show the PLAN lane above the BEAD lane in SASE CONTEXT

## Goal

In the ACE Agents metadata panel, the `SASE CONTEXT` section renders its lanes in a
fixed declared order. Today that order starts with `BEAD` and then `PLAN`. The project
owner wants `PLAN` rendered **above** `BEAD` whenever both lanes are present.

The lane order is data, not control flow: it lives in two module-level tuples, and every
downstream behavior (hint numbering, responsive reflow ranges, section navigation
anchors, metadata search order) derives from render order automatically. So the code
change is small; the real work is updating the order contracts that are asserted in
tests, documented in `docs/`, and baked into PNG visual goldens.

## Scope decision (read this first)

The request named "the agent metadata panel". That panel (`AgentPromptPanel`) renders a
`SASE CONTEXT` section in **two** shapes:

1. The single-agent metadata panel, ordered by `CONTEXT_LANE_ORDER`.
2. The aggregated clan document, ordered by `CLAN_CONTEXT_LANE_ORDER`.

This plan changes **both**, so the two shapes of the same panel do not disagree about
where `PLAN` sits relative to `BEAD`. That is a deliberate, small extension of the
literal request; if the reviewer wants the clan document left alone, drop step 2 and its
associated test/golden updates — nothing else in the plan depends on it.

Not in scope: lane colors, lane contents, lane fold IDs, lane presence/suppression rules
(for example, a phase worker still never shows its parent epic as a `PLAN` lane), the
`ARTIFACTS`/`MEMORY`/`SKILLS`/`WORKSPACES`/`COMMITS` relative order, and the plan-header
block feature in `src/sase/sdd/plan_display.py` (a different feature that also has
`BEAD` sections — do not touch it).

No Rust core change is required. Lane ordering here is presentation-only Textual
rendering, which stays on the Python side of the Rust core backend boundary.

## Current behavior

Single-agent panel — `src/sase/ace/tui/widgets/prompt_panel/_agent_context.py`:

- Lines 28-35 declare:

  ```python
  CONTEXT_LANE_ORDER = (
      "BEAD",
      "PLAN",
      "ARTIFACTS",
      "MEMORY",
      "SKILLS",
      "WORKSPACES",
  )
  ```

- `append_agent_context_section()` builds each lane's `Text` in that order (lines
  107-112), skips empty lanes, then appends the surviving lanes in the same order (lines
  145-174).

Clan document — `src/sase/ace/tui/models/_agent_clan_sections.py` lines 41-49 declare:

```python
CLAN_CONTEXT_LANE_ORDER = (
    "BEAD",
    "PLAN",
    "COMMITS",
    "ARTIFACTS",
    "MEMORY",
    "SKILLS",
    "WORKSPACES",
)
```

Both clan lane builders iterate that same tuple, so one edit covers both:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_clan_disk_aggregation.py` line 40
  (accumulator seeding) and line 168 (final lane emission).
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_sections.py` line 609
  (in-memory lane emission).

## Why no other source changes are needed

Verify these while implementing, but expect no edits:

- **Responsive reflow.** `_agent_context.py` lines 169-174 record
  `responsive_ranges["BEAD"]` and `responsive_ranges["PLAN"]` as `(start, end)` offsets
  captured at append time, so the offsets follow whatever order the lanes were appended
  in. `_agent_display_header.py` lines 392-402 collect those ranges and then
  `responsive_sections.sort(key=lambda section: section[0])` before building
  `AgentHeaderRenderable`, whose `__rich_console__` walks the already-sorted sections
  with a monotonically advancing cursor. Order-independent by construction.
- **The `plan_range` return value.** `append_agent_context_section()` still returns the
  `PLAN` lane's range (line 173-175) regardless of position. Keep that contract.
- **Hint numbering.** Hint numbers are assigned inside the lane renderers as the lanes
  are built (lines 57-80), using the shared `HeaderHintState` counter, so hint numbers
  keep ascending in document order after the swap. This _does_ renumber `PLAN` and
  `BEAD` hints relative to each other — that is intended and is the correct behavior
  (see the test update below).
- **Family/clan fold path.** The `fold_level is not None` branch (lines 120-168) derives
  lane IDs and fold levels from `rendered_lanes`, so it follows the tuple too.
- **Section navigation.** `Ctrl+J`/`Ctrl+K` anchors come from rendered section markers
  in document order (`_section_navigation.py`), so they follow automatically.

## Implementation steps

### 1. Swap the single-agent lane order

In `src/sase/ace/tui/widgets/prompt_panel/_agent_context.py`, change
`CONTEXT_LANE_ORDER` to:

```python
CONTEXT_LANE_ORDER = (
    "PLAN",
    "BEAD",
    "ARTIFACTS",
    "MEMORY",
    "SKILLS",
    "WORKSPACES",
)
```

Add a brief comment above the tuple recording the intent — the authored plan is the
agent's stated intent and leads the narrative, with the bead lane immediately beneath it
— so a future reader does not "fix" the order back. Match the file's existing comment
density; one or two lines is enough.

### 2. Swap the clan lane order

In `src/sase/ace/tui/models/_agent_clan_sections.py`, change `CLAN_CONTEXT_LANE_ORDER`
to lead with `"PLAN"`, then `"BEAD"`, leaving `"COMMITS"`, `"ARTIFACTS"`, `"MEMORY"`,
`"SKILLS"`, `"WORKSPACES"` in their current relative order.

### 3. Update the order contracts in unit tests

`tests/ace/tui/widgets/test_agent_context.py`:

- `_EXPECTED_CONTEXT_LANE_ORDER` (lines 51-58): lead with `"PLAN"`, then `"BEAD"`.
- `test_bead_precedes_plan_without_changing_plan_range_bookkeeping` (~line 247): rename
  to `test_plan_precedes_bead_without_changing_plan_range_bookkeeping` and flip the
  final assertion from `bead_end < plan_start` to `plan_end < bead_start`. Keep the
  `plan_range == responsive_ranges["PLAN"]` and both slice-equality assertions as-is —
  they are the bookkeeping this test exists to protect.
- `test_task_authored_plan_context_renders_bead_then_plan_lane` (~line 267): rename to
  `..._renders_plan_then_bead_lane` and flip the ordering assertion to
  `plain.index("▸ PLAN") < plain.index("▸ BEAD")`. The per-lane content assertions do
  not change.
- `test_context_lane_order_contract_holds_for_every_presence_combination` (~line 390):
  the `rendered_labels == expected_labels` assertion is derived from
  `_EXPECTED_CONTEXT_LANE_ORDER` and needs no edit, but the explicit index assertions
  below it must become:
  - `if enabled["PLAN"]: assert rendered_labels[0] == "PLAN"`
  - `if enabled["BEAD"]: assert rendered_labels.index("BEAD") == (1 if enabled["PLAN"] else 0)`
  - the `ARTIFACTS` index assertion is unchanged (still
    `int(enabled["BEAD"]) + int(enabled["PLAN"])`).
- `test_context_lanes_render_in_parent_context_order` (~line 428): flip the first two
  assertions so the chain reads `SASE CONTEXT` → `▸ PLAN` → `▸ BEAD` → `▸ ARTIFACTS` →
  `▸ MEMORY` → `▸ SKILLS` → `▸ WORKSPACES`.
- `test_context_hint_numbers_follow_display_order` (~line 451): the `PLAN` lane's `Path`
  hint now comes before the `BEAD` lane's `Epic Plan` hint. Expect:
  - `Path: [1]` before `Epic Plan: [2]`, then `[3] abcdef123456`, `[4] src/output.py`,
    `[5] reports/result.md`, `[6] generated_skills.md` (unchanged tail);
  - `hint_state.hint_mappings` with `1` mapped to the plan path
    (`/tmp/workspace/sase/repos/plans/plan.md`) and `2` mapped to the epic path
    (`/tmp/workspace/sase/repos/plans/epic.md`), other entries unchanged;
  - `list(hint_state.commit_views) == [3]` and `hint_state.hint_counter == 7` both
    unchanged.

  Also fix the first assertion in that test, which currently reads
  `plain.index("Size:  medium") < plain.index("Epic Plan: [1]")` — after the swap it
  should compare against the lane rows that are actually adjacent in the new order.
  Prefer asserting the hint sequence explicitly rather than relying on the old `Size` →
  `Epic Plan` adjacency.

`tests/ace/tui/widgets/test_agent_clan_aggregation.py` (~line 234): the
`[lane.label for lane in lanes] == [...]` expectation becomes `"PLAN"`, `"BEAD"`,
`"ARTIFACTS"`, `"MEMORY"`, `"SKILLS"`, `"WORKSPACES"`.

Nothing else in `tests/ace/tui/widgets/` needs an order edit. In particular
`tests/ace/tui/widgets/test_agent_display_clan_context_hints.py` hand-builds its lane
tuple with `PLAN` already first, and
`tests/ace/tui/widgets/_agent_display_clan_helpers.py` builds only `BEAD` and `SKILLS`
lanes — both are already consistent with the new order. Confirm this rather than
assuming; if a test fails, fix it in the same spirit.

### 4. Update the visual test's inline ordering assertions

`tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py`, inside
`test_agents_phase_family_bead_and_plan_context_png_snapshot`:

- Line ~482: `assert metadata.index("▸ BEAD") < metadata.index("▸ PLAN")` becomes
  `assert metadata.index("▸ PLAN") < metadata.index("▸ BEAD")`.
- Line ~488: `assert svg_plain.index("BEAD") < svg_plain.index("PLAN")` becomes
  `assert svg_plain.index("PLAN") < svg_plain.index("BEAD")`.

Consider renaming the test to reflect the asserted order (for example
`test_agents_phase_family_plan_and_bead_context_png_snapshot`); if you rename it, leave
the golden's file name alone unless you also rename the `assert_page_png` key, and if
you do rename the key, rename the golden file with `git mv` rather than regenerating a
second copy.

Leave `tests/ace/tui/visual/test_ace_png_snapshots_agents_zoom.py` alone: its assertion
chain starts at `▸ PLAN` and that agent has no `BEAD` lane, so it stays valid.

### 5. Regenerate the affected PNG goldens

Run `just test-visual`. Expect at least
`tests/ace/tui/visual/snapshots/png/agents_phase_bead_and_plan_context_120x40.png` to
fail, since that is the one golden known to show both lanes together. The clan-panel
goldens (`agents_clan_panel_epic_level_2_120x40`, `..._level_3_120x40`,
`..._hints_120x40`, `..._logical_prompt_hints_120x40`) may also shift if their fixture
clan contributes both a `PLAN` and a `BEAD` lane; the hint-mode goldens can additionally
renumber. Let the suite tell you the exact set instead of guessing.

For every failure:

1. Inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/`.
2. Confirm the only differences are the lane swap and any resulting hint renumbering —
   no changed colors, spacing, truncation, or lost rows.
3. Accept with `just test-visual` plus `--sase-update-visual-snapshots` (pass it through
   to pytest), then rerun `just test-visual` clean.

If a diff shows anything beyond the swap and renumbering, stop and investigate — that
would mean a real regression, not a golden refresh.

### 6. Update the documentation

- `docs/ace.md` (~lines 1040-1041): "The lane order is `BEAD`, `PLAN`, `ARTIFACTS`, the
  audited `MEMORY`, `SKILLS`, and `WORKSPACES`, with absent lanes omitted." → lead with
  `PLAN`, then `BEAD`.
- `docs/ace.md` (~lines 3299-3301, in the **SASE CONTEXT / PLAN** bullet): "across every
  combination of present lanes, the full order is `BEAD`, `PLAN`, `ARTIFACTS`, `MEMORY`,
  `SKILLS`, then `WORKSPACES`" → same swap.
- `docs/agent_images.md` (~lines 650-651): "The complete lane order is `PLAN`,
  `ARTIFACTS`, `MEMORY`, `SKILLS`, then `WORKSPACES`". This sentence is already stale —
  it omits `BEAD` entirely. Correct it to `PLAN`, `BEAD`, `ARTIFACTS`, `MEMORY`,
  `SKILLS`, then `WORKSPACES` so all three passages agree.

Then grep the docs once more for any other stated lane order
(`rg -n 'BEAD.*PLAN|PLAN.*ARTIFACTS' docs/`) and fix anything the list above missed. Do
**not** edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) — a grep confirmed no memory
file states this lane order, and memory edits require explicit user permission that this
plan does not grant.

## Verification

```bash
just install          # ephemeral workspace: always install first
just check            # whole-repo lint gates + diff-scoped tests
just test-visual      # PNG snapshot suite (regenerate goldens as described above)
just check-full       # before landing: every lint gate + the full test suite
```

`just check-full` is required here rather than optional: this change touches docs, two
shared order constants, and the visual suite, which is exactly the broadening set the
scoped lane is not meant to cover on its own.

Targeted runs while iterating:

```bash
just test tests/ace/tui/widgets/test_agent_context.py
just test tests/ace/tui/widgets/test_agent_clan_aggregation.py
```

## Definition of done

- `PLAN` renders directly above `BEAD` in `SASE CONTEXT` in the single-agent metadata
  panel, in the metadata zoom view, and in the aggregated clan document, whenever both
  lanes are present.
- A lane that is absent is still omitted entirely; every other lane keeps its current
  relative position.
- `plan_range`, `responsive_ranges`, responsive width reflow, section navigation, and
  metadata search all still behave correctly, with hint numbers ascending in document
  order.
- Unit tests, visual assertions, PNG goldens, and the three doc passages all agree on
  the new order.
- `just check-full` and `just test-visual` pass.
