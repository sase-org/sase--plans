---
tier: tale
title: Move the lane NEIGHBORS section above SASE CONTEXT
goal: 'On an Agents-tab lane panel the numbered NEIGHBORS roster renders directly
  below WORKFLOW VARIABLES and immediately above SASE CONTEXT, with family/neighbor
  digit numbering, fold behavior, and every other section unchanged.

  '
create_time: 2026-07-25 13:44:00
status: done
---

- **PROMPT:** [202607/prompts/neighbors_above_sase_context.md](prompts/neighbors_above_sase_context.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-99.land.f3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-99.land.f3/README.md)
  - [bbugyi200.athena.sase-99.land.f3--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-99.land.f3.md#member-code)
- **COMMITS:**
  - [c917bc0](https://github.com/sase-org/sase/commit/c917bc04d341885ec11fdf4285288cc846fa4469) — feat(ace): move the lane NEIGHBORS section above SASE CONTEXT

# Move the lane `NEIGHBORS` section above `SASE CONTEXT`

## 1. Goal

On an Agents-tab lane panel, render the numbered `NEIGHBORS` roster directly beneath `WORKFLOW VARIABLES` and
immediately above `SASE CONTEXT`, instead of at the very bottom of the metadata region below `ERROR`.

This deliberately supersedes decision **D3** of the sase-99 epic plan
(`sase/repos/plans/202607/lane_neighbors_section.md`, "Placement: last section of the metadata region"), which argued
neighbors are navigational context and therefore must not push `BEAD` / `PLAN` / `SASE CONTEXT` down. The user has
decided the opposite trade: lane neighbors are the panel's primary navigation affordance (they own digits `0`-`9`), so
they should be reachable without scrolling past the context, slow-call, and error sections.

## 2. Current state

`build_header_text()` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py` renders a non-clan agent panel
in this document order:

1. agent metadata fields (`append_agent_metadata_fields`, line 131)
2. family only: `Fold: N/M` line + `FAMILY MEMBERS` roster (lines 144-157)
3. legacy parallel members (line 159)
4. `OUTPUT VARIABLES` (line 161)
5. major divider + `WORKFLOW VARIABLES` (lines 175-198)
6. `SASE CONTEXT` (lines 203-225, via `append_agent_context_section`)
7. `SLOW TOOL CALLS` (lines 227-245)
8. `ERROR` (lines 247-276)
9. **`NEIGHBORS`** (lines 278-289, via `append_lane_neighbors_section`)
10. merged member-jump-map publish (lines 291-298)
11. trailing `─` × 50 divider separating metadata from the prompt/reply body (lines 309-311)

The only render call site is `build_header_text()`. The three wiring paths (`_agent_display_render.py:229`,
`_agent_display.py:226`, `_agent_display_hints.py:170`) just resolve the projection and pass `lane_neighbors=` through,
so they need no change.

## 3. Design decisions

**D1 — Insert between `WORKFLOW VARIABLES` and `SASE CONTEXT`.** This is the literal instruction and the minimal edit:
move the existing block, change nothing about its content, fold behavior, limits, or jump map.

**D2 — Family/neighbor numbering order is unchanged.** `document_numbering` (`MemberJumpNumbering`, line 126) allocates
digits in render order. `FAMILY MEMBERS` renders at line 150, well above the new insertion point, so family members keep
digits `0..n-1` and neighbors keep `n..`. The published jump map, its shared digit width, and
`_handle_member_jump_key`'s two-digit detection are all unaffected. **Any implementation that changes the published
digit sequence for a family lane is wrong.**

**D3 — Rejected alternative: place `NEIGHBORS` directly under `FAMILY MEMBERS`.** That would make the two rosters that
share one digit ladder visually adjacent, which reads better on a family lane. It was rejected because it does not match
the user's instruction, it would push `OUTPUT VARIABLES` / `WORKFLOW VARIABLES` down on _every_ lane panel (a second,
unrequested reordering), and on the common single-agent lane — which has no `FAMILY MEMBERS` — the two placements are
identical anyway.

**D4 — Move the jump-map publish block with it.** After the move, the neighbors render block and the
`merged_member_jump_map` publish block (lines 291-298) become non-adjacent for no reason. Move the publish block so it
sits immediately after the neighbors append. It reads only `family_map` (assigned at line 150) and `neighbors_map`, both
of which are set by that point, so this is behavior-preserving. Publishing must still happen exactly once per document.

**D5 — Dividers need no change.** `append_lane_neighbors_section` emits its own leading `\n` + `━` × 50 accent rule via
`_append_roster_heading`, and `append_agent_context_section` emits its own `\n` + `─` × 50 + `\n` major divider
immediately before the `SASE CONTEXT` heading. Both are self-contained, so the existing
`assert_dim_divider_before(header, "SASE CONTEXT\n")` assertions keep passing unchanged. Do **not** add or remove any
divider.

**D6 — `responsive_ranges` stay correct for free.** `append_agent_context_section` records the `BEAD` / `PLAN`
`(start, end)` character offsets against `header_text` at append time. Because `NEIGHBORS` is now already in the buffer
when those offsets are captured, `AgentHeaderRenderable.__rich_console__` still slices the right ranges. No offset
arithmetic is needed.

## 4. Implementation

### 4.1 `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`

Move the block currently at lines 278-298 (the `neighbors_map = None` / `if lane_neighbors is not None:` append **and**
the `if member_jump_map_publisher is not None and (...)` publish) so it runs immediately after the `if meta_fields:`
block ends (currently line 198) and before the `bead_section: ResponsiveBeadSection | None = None` declaration
(currently line 200).

Nothing inside the moved block changes: same `append_lane_neighbors_section(...)` keyword arguments, same
`merged_member_jump_map(agent.identity, family_map, neighbors_map)` call. `neighbors_level` / `neighbors_limit` /
`shown_neighbor_count` / `document_numbering` (lines 107-128) stay exactly where they are — they must keep running
before the family roster so the shared numbering total is known up front.

Verify after the edit that `family_map` is still assigned before the moved publish block reads it (it is: line 150 vs.
the new position after line 198).

### 4.2 `tests/ace/tui/widgets/test_agent_display_neighbors.py`

Rename `test_header_places_neighbors_after_error_and_before_metadata_divider` to
`test_header_places_neighbors_below_workflow_variables_and_above_sase_context` and rewrite its assertions. The existing
version passes `cheap=True`, which suppresses `SASE CONTEXT` entirely, so it cannot express the new contract. The
replacement must render a panel that contains **all four** of `WORKFLOW VARIABLES`, `NEIGHBORS`, `SASE CONTEXT`, and
`ERROR` at once and assert:

```
plain.index("WORKFLOW VARIABLES") < plain.index("NEIGHBORS")
plain.index("NEIGHBORS") < plain.index("SASE CONTEXT")
plain.index("SASE CONTEXT") < plain.index("ERROR")
plain.index("NEIGHBORS") < plain.rindex("─" * 50)
```

To get those sections to co-occur:

- `SASE CONTEXT`: pass `cheap=False` (the default) and `summary=DetailHeaderSummary(associated_plan=...)`. Reuse the
  `plan_summary` helper from `tests/ace/tui/widgets/_agent_display_plan_helpers.py`, as
  `test_agent_display_plan_section.py:52` does.
- `WORKFLOW VARIABLES`: set `agent.step_output = {"meta_foo": "bar"}`, as `test_prompt_panel_header.py:237` does.
- `ERROR`: keep the existing `status="FAILED"`, `error_message="boom"` lane agent.

Keep the existing published-jump-map assertions (`len(published) == 1` and the target identity list) in the rewritten
test — the move must not change them.

Leave `test_family_header_publishes_one_contiguous_shared_width_jump_map` and
`test_neighbors_are_absent_for_empty_projection_and_non_lane_rows` unchanged; both must still pass as written, which is
what proves D2.

### 4.3 Visual snapshots

The three existing lane-neighbor goldens must be re-verified, not blindly re-accepted:

- `agents_lane_neighbors_section_first_level_160x50.png`
- `agents_lane_neighbors_section_expanded_160x50.png`
- `agents_family_lane_neighbors_160x50.png`

None of their fixtures renders `WORKFLOW VARIABLES` or `SASE CONTEXT`, so all three are expected to stay
**byte-identical**. Run `just test-visual` and confirm they pass with no `--sase-update-visual-snapshots`. If any of
them does differ, stop and investigate — a diff there means the move changed something it should not have.

Because those goldens contain no `SASE CONTEXT`, the new order currently has no visual proof. Add one new snapshot to
`tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py`:

- New test `test_agents_lane_neighbors_above_sase_context_png_snapshot`, golden
  `agents_lane_neighbors_above_context_160x50`, size `(160, 50)`.
- Build the fixture from the existing `_single_lane_neighbor_agents()` shape, but give the selected `visual.lane.plan`
  agent a plan so `SASE CONTEXT` renders. `_lane_agent()` already forwards `**overrides` to `Agent(...)`, so pass
  `plan_path=`, `sdd_plan_path=`, `plan_committed=True`, `plan_action="tale"`, and `workspace_dir=str(tmp_path)`. Follow
  `test_agents_sase_plan_metadata_png_snapshot` in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_auto_approve.py:324` for the exact `tmp_path` plan-file setup
  (write a small `tier: tale` frontmatter plan at `sase/repos/plans/202607/<name>.md` under `tmp_path`).
- Assert via `assert_page_svg_contains` that both `NEIGHBORS` and `SASE CONTEXT` are present, and assert on the SVG
  plain text that `NEIGHBORS` appears before `SASE CONTEXT`, mirroring the index-comparison style already used at
  `test_ace_png_snapshots_agents_auto_approve.py:311`.
- Pin time with `pin_agents_visual_now(monkeypatch, _LANE_NOW)` so the golden is deterministic.
- Generate the golden with `just test-visual --sase-update-visual-snapshots`, then **open the produced PNG and confirm
  by eye** that `NEIGHBORS` sits above `SASE CONTEXT` before committing it.

### 4.4 Documentation

`docs/ace.md`, "Lane Neighbors Section":

- First sentence currently reads "Every agent lane panel **ends its metadata region with** a numbered `NEIGHBORS`
  roster." Reword so it no longer claims the section is last (e.g. "carries a numbered `NEIGHBORS` roster in its
  metadata region").
- Replace the placement sentence "The section sits after `ERROR` and immediately before the divider that separates
  metadata from the prompt/reply body, so it never pushes `BEAD`, `PLAN`, or `SASE CONTEXT` down." with the new
  placement and its new rationale: it sits directly below `WORKFLOW VARIABLES` and immediately above `SASE CONTEXT`, so
  a lane's numbered neighbors stay reachable without scrolling past the context, slow-call, and error sections.

`docs/agent_families.md:272`: "A family root is an agent lane, so its panel also **ends with** a `NEIGHBORS` section
listing the lane's ancestors, descendants, and hood neighbors below `FAMILY MEMBERS`." — reword to state that the
section sits below `FAMILY MEMBERS` and above `SASE CONTEXT`.

Leave the rest of both files alone. `docs/ace.md:419` (continuous digit ladder), `docs/ace.md:528-531` (fold scale and
heading glyph), and the help-modal string `"Neighbors modal (see NEIGHBORS)"`
(`src/sase/ace/tui/modals/help_modal/agents_bindings.py:49`) are all placement-agnostic and stay as they are. No keymap,
footer, or `src/sase/default_config.yml` change is involved, so the "help popup must stay in sync" rule in
`src/sase/ace/CLAUDE.md` is satisfied by the doc edits above.

Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or any generated provider instruction shim.

## 5. Out of scope

- Any change to which neighbors are listed, their grouping, ordering, labels, dismissed handling, or the
  `… +N also listed under FAMILY MEMBERS` suppression tail.
- Any change to the fold ladder (3 / 10 / all by scale position) or to `neighbor_entry_limit`.
- The `~` neighbor modal, the info-panel `neighbors:` badge, and digit-jump/revive behavior.
- Moving `FAMILY MEMBERS`, `OUTPUT VARIABLES`, `WORKFLOW VARIABLES`, `SLOW TOOL CALLS`, or `ERROR` relative to one
  another.

## 6. Validation

```bash
just install
just test-visual
just check
```

Targeted runs while iterating:

```bash
pytest tests/ace/tui/widgets/test_agent_display_neighbors.py \
       tests/ace/tui/widgets/test_summary_fold_contracts.py \
       tests/ace/tui/widgets/test_prompt_panel_header.py \
       tests/ace/tui/widgets/test_agent_display_plan_section.py \
       tests/ace/tui/widgets/test_agent_display_commit_metadata.py
pytest tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py
```

`test_summary_fold_contracts.py` is expected to pass **unmodified**. Its `_render_lane` helper uses `cheap=True`, so no
`SASE CONTEXT` divider appears between the `NEIGHBORS` heading and the trailing metadata divider, and its
`_section_body` slicing (which stops at the first `─`/`━` rule after the heading) still captures the same body. If that
file needs edits, re-read it before changing anything — the likely cause is an unintended behavior change, not a stale
test.

## 7. Acceptance criteria

- On a single-agent lane panel with a plan and workflow variables, the metadata region reads
  `… → WORKFLOW VARIABLES → NEIGHBORS → SASE CONTEXT → SLOW TOOL CALLS → ERROR → ────` divider.
- On a family lane panel, `FAMILY MEMBERS` still precedes `NEIGHBORS`, and the published jump map still numbers family
  members first and neighbors second, contiguously, with one shared digit width.
- The three pre-existing lane-neighbor PNG goldens are unchanged; one new golden shows `NEIGHBORS` above `SASE CONTEXT`.
- `docs/ace.md` and `docs/agent_families.md` no longer describe `NEIGHBORS` as the last metadata section.
- `just check` and `just test-visual` are green.
