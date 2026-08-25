---
tier: tale
title: Restrict Ctrl+J/Ctrl+K to real metadata section titles
goal:
  The Agents-tab metadata panel's Ctrl+J / Ctrl+K cycle stops only on rendered ALL-CAPS
  underlined section titles, while numbered roster rows and SASE CONTEXT lane
  sub-headings keep working as fold anchors.
size: medium
proposed_by: bbugyi200.athena.0dt
create_time: 2026-08-25 16:18:36
status: wip
---

# Plan: Restrict Ctrl+J/Ctrl+K to real metadata section titles

## Problem

On the Agents tab, `Ctrl+J` / `Ctrl+K` (`next_agent_metadata_section` /
`prev_agent_metadata_section`) are documented as cycling through _section titles_ of the
metadata panel:

- `docs/ace.md` ("Agents Tab Metadata Panel"): "cycle forward and backward through the
  rendered **titled sections** … Only rendered section titles participate".
- `docs/agent_families.md`: "Use `Ctrl+J` and `Ctrl+K` to move between the visible
  **section headings**."

The implementation does not honor that contract. Two classes of rendered line are marked
as navigation anchors even though they are not section titles:

1. **Numbered member-roster rows.** `_append_numbered_entry()` in
   `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py` (currently line 378) marks
   every roster data row with
   `append_section_heading(text, line, section_id=anchor_id)`, where `anchor_id` is
   `f"{member_anchor_prefix}{entry.presented_name}"`. That makes every row of every
   roster a `Ctrl+J` stop, in all four rosters that share this renderer:
   - `FAMILY SHELLS` — `_agent_display_header.py:208` → `_agent_display_family.py:165`
     (prefix `member:`) — this is the case the project owner reported;
   - clan `MEMBERS` — `_agent_display_clan.py:225` (prefix `member:`);
   - `NEIGHBORS` — `_agent_display_neighbors.py:124` (prefix `neighbor:`);
   - tribe `MEMBERS` — `_agent_display_tribe.py:93` (prefix `tribe:member:`).

   These rows render as data (`--plan · agent · ✓ Done · sonnet · 3m`), not as an
   all-caps underlined title, so each one wrongly steals a jump stop and re-scrolls the
   panel.

2. **Family-container `SASE CONTEXT` lane sub-headings.** In
   `src/sase/ace/tui/widgets/prompt_panel/_agent_context.py`, when
   `fold_level is not None` (family container panels only —
   `family_fold_enabled = agent.is_family_container_row and lane_fold_level is not None`
   in `_agent_display_header.py:130`), the code appends the real `SASE CONTEXT` heading
   **unmarked** (currently lines 195-196) and instead marks each lane sub-heading
   (`BEAD`, `PLAN`, `ARTIFACTS`, `MEMORY`, `GLOSSARY`, `SKILLS`, `WORKSPACES`, currently
   line 236). Those lane labels are styled bold + colored but **not underlined**
   (`COLOR_BEAD_SUBHEADER` and friends in `_agent_context_common.py:20-26`), so they are
   sub-headings, not section titles. The result on a family container is backwards: the
   one true title in that region is unreachable and seven sub-headings are stops.

No other anchor sources violate the rule. Every anchor in the panel is produced by
`append_section_heading()` in `src/sase/ace/tui/widgets/prompt_panel/_helpers.py`, and
each remaining call site passes an ALL-CAPS title styled with an `underline` attribute
(`AGENT PROMPT`, `AGENT REPLY`, `ERRORS`, `OUTPUT VARIABLES`, `WORKFLOW VARIABLES`,
`MONITOR`, `PROC DETAILS`, `SLOW TOOL CALLS`, `QUEUE`, `FAMILY SHELLS`, `NEIGHBORS`,
`MEMBERS`, `INPUTS`, `WORKFLOW DETAILS`, `WORKFLOW STEPS`, `STEP OUTPUT`, `BASH COMMAND`
/ `PYTHON CODE`, `COMMAND`, `LOG TAIL`, `ATTEMPT N REPLY`, `ATTEMPT ERROR`,
`SASE CONTEXT`, clan/tribe section titles).

## Why the anchors cannot simply be deleted

The same marked spans serve a second consumer. `FoldNavigationMixin` in
`src/sase/ace/tui/actions/navigation/_fold.py` resolves the fold target through
`_current_agent_metadata_section_id()` → `AgentPromptPanel.resolve_section_at_row()`, so
`za` / `zA` fold whatever marked anchor owns the metadata viewport's top row. That is
how per-member overrides (`member:<name>`) and per-lane overrides (`bead`, `plan`, …)
are applied — `docs/ace.md:1137` documents `za` as "Cycle the foldable section **or
numbered member** at the top of the metadata viewport".

So the fix is not "stop marking these rows"; it is "give each marked anchor a **role**"
and let only title-role anchors participate in `Ctrl+J` / `Ctrl+K`, while every anchor
keeps participating in the viewport-top fold lookup.

## Implementation

All paths below are repo-relative. Line numbers are from the pre-change tree and are
hints, not addresses.

### 1. Anchor roles — `src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py`

- Add a public role enum, e.g.:

  ```python
  class PromptPanelSectionRole(Enum):
      """Whether a marked span is a navigable title or only a fold anchor."""

      TITLE = auto()
      FOLD_ONLY = auto()
  ```

- Add `role: PromptPanelSectionRole = PromptPanelSectionRole.TITLE` as the third field
  of the frozen `PromptPanelSectionAnchor` dataclass (defaulting to `TITLE` keeps every
  existing construction site and test valid).
- Add a second meta key next to `SECTION_MARKER_META_KEY`, e.g.
  `SECTION_FOLD_ONLY_META_KEY = "sase_prompt_panel_section_fold_only"`. Only fold-only
  anchors carry it; its absence means `TITLE`. Keeping the existing key as the identity
  key means existing tests that enumerate marked spans by `SECTION_MARKER_META_KEY` keep
  working.
- Change `_segment_section_identity()` to return
  `tuple[str, PromptPanelSectionRole] | None` and have both collection paths
  (`render_strips()` for the Textual visual path and `get_height()` for the Rich
  measurement path) build anchors with the resolved role. Preserve the existing
  first-occurrence `seen` de-duplication by identity.
- **Performance requirement** (`sase/memory/tui_perf.md` rule: render/measure paths must
  stay cheap): `rich.style.Style.meta` unpickles on _every_ property access. The current
  helper already reads `style.meta` twice for a marked segment. Read it **once** into a
  local and take both keys from that dict, so the new role never adds an unpickle:

  ```python
  meta = style.meta if style is not None else None
  if not meta:
      return None
  identity = meta.get(SECTION_MARKER_META_KEY)
  ...
  ```

  Emit both keys from a single `Style(meta={...})` construction (one pickle) rather than
  two `stylize()` calls.

### 2. Mark-time API — `src/sase/ace/tui/widgets/prompt_panel/_helpers.py`

- Keep `append_section_heading(...)` exactly as it is for callers: it produces a `TITLE`
  anchor. Extend its docstring to state that it is for ALL-CAPS underlined section
  titles only, and that a `Ctrl+J` stop is created by calling it.
- Add a sibling for fold-only anchors, e.g.:

  ```python
  def append_fold_anchor(text: Any, line: Text, *, section_id: str) -> None:
      """Append one foldable, non-navigable line and mark its fold anchor.

      Roster rows and SASE CONTEXT lane sub-headings are foldable by ``za``/``zA``
      at the viewport top but are not section titles, so they never become
      ``Ctrl+J``/``Ctrl+K`` stops.
      """
  ```

  It must behave identically to `append_section_heading()` in every other respect —
  append the pre-styled `Text`, mark `[start, start + len(line.plain))`, append exactly
  one `"\n"`, and set `text.end = ""` — because `_append_numbered_entry()` depends on
  that `end` handling. Factor the shared body into the existing private
  `_mark_section_heading()` (add a `fold_only: bool = False` parameter) plus a small
  shared appender rather than duplicating the logic.

- Also extend the `append_kind_header()` docstring note so the whole rule lives in one
  place: kind identity lines (`FAMILY`, `CLAN`, `TRIBE`, `AGENT SHELL`) stay unmarked
  chrome; roster rows and lane sub-headings are fold-only anchors; only titles are
  navigable.

### 3. Roster rows — `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`

- In `_append_numbered_entry()`, replace
  `append_section_heading(text, line, section_id=anchor_id)` with the new
  `append_fold_anchor(text, line, section_id=anchor_id)`.
- Leave `_append_roster_heading()` on `append_section_heading()`: `FAMILY SHELLS`,
  `NEIGHBORS`, and `MEMBERS` stay navigable titles.
- No change is needed in the four roster call sites; they all flow through this one
  renderer.

### 4. Family `SASE CONTEXT` — `src/sase/ace/tui/widgets/prompt_panel/_agent_context.py`

In `append_agent_context_section()`, the `fold_level is not None` branch:

- Mark the `SASE CONTEXT` heading as the region's title. Today it is written as two bare
  appends (`text.append("SASE CONTEXT", style=_COLOR_HEADER)` then
  `text.append(f" · {len(rendered_lanes)}\n", style="dim")`). Build the heading as a
  `Text` (label styled `_COLOR_HEADER`, then the dim ` · N` count) and route it through
  `append_section_heading(text, heading, section_id="sase-context")`. Watch the line
  ending: `append_section_heading()` appends its own `"\n"`, so drop the `\n` currently
  embedded in the count string — the rendered output must be byte-identical to today.
  Use the literal id `"sase-context"` so it matches
  `_default_section_id("SASE CONTEXT")` used by the non-family branch and any existing
  override key.
- Change the per-lane heading mark (currently
  `append_section_heading(text, heading, section_id=section_id)` inside the
  `family_lane_levels is not None` loop) to `append_fold_anchor(...)`, keeping the same
  `section_id` values (`bead`, `plan`, `artifacts`, `memory-reads`, `glossary-reads`,
  `skill-uses`, `opened-workspaces`) so per-lane fold overrides continue to resolve.
- The non-family branch is already correct (`SASE CONTEXT` marked, lanes unmarked) —
  leave it alone.
- Note the consequence and do **not** try to fix it here: `za` on the new `sase-context`
  title writes an override no renderer reads today, exactly as it already does on
  non-family panels. Making the `SASE CONTEXT` title fold its lanes is out of scope (see
  Non-goals).

### 5. Navigation filtering — `src/sase/ace/tui/widgets/prompt_panel/__init__.py`

- `resolve_section_target()`: filter the cached anchors to
  `role is PromptPanelSectionRole.TITLE` **before** the empty check and before index
  math, so `EMPTY` is returned for a document with anchors but no titles, and so the
  `TOP` waypoint still fires at the first/last _title_. The `NOT_READY` generation/width
  gate is unchanged.
- `resolve_section_at_row()`: unchanged — it must keep walking **all** anchors so `za` /
  `zA` still resolve roster rows and lane sub-headings at the viewport top.
- `_publish_section_layout()`: the "active identity disappeared" reconciliation should
  compare against title anchors only (an active identity is always a title).
- `get_content_height()`: the trailing layout reserve currently uses `anchors[-1].row`.
  It exists so the **final title** can be aligned to the viewport top, so it must now
  use the last `TITLE` anchor's row (and reserve `0` when there are no titles).
  Otherwise a roster whose rows trail the last title reserves more blank space than the
  feature needs.

### 6. Docs

- `docs/ace.md`, "Agents Tab Metadata Panel" (near the `Ctrl+J` / `Ctrl+K` paragraph):
  state the invariant explicitly — the stops are exactly the rendered ALL-CAPS
  underlined section titles; numbered roster rows and `SASE CONTEXT` lane sub-headings
  are fold anchors only, reachable by `za` / `zA` when they own the viewport's top row.
- `docs/ace.md` fold-chord table row for `za`: keep "or numbered member" (still true)
  and mention the family `SASE CONTEXT` lanes belong to the same fold-anchor class.
- `docs/ace.md` family-panel section: note that a family container's `SASE CONTEXT`
  heading is now the navigable title for that region.
- `docs/agent_families.md`: the `Ctrl+J`/`Ctrl+K` sentences already say "section
  headings"; add the roster-row clarification where the numbered roster is described.
- Optional polish: the help-modal label for these keys in
  `src/sase/ace/tui/modals/help_modal/agents_bindings.py` currently reads "Cycle
  metadata through top"; "Cycle metadata section titles" is more accurate.

## Tests

Add or update tests in this order; the first one is the durable guard the project owner
asked for.

1. **Invariant guard** (new, in
   `tests/ace/tui/widgets/test_prompt_panel_section_navigation.py`): render
   representative documents — a family container, a clan container, a tribe panel, a
   plain sase-agent panel, and a workflow panel (reuse the existing helpers
   `tests/ace/tui/widgets/_agent_display_family_helpers.py::make_family`,
   `build_header_text`, `build_workflow_detail_renderable`) — and assert that for
   **every `TITLE` anchor** the underlined portion of the marked span is non-empty and
   equal to its own `.upper()`. Implementation note: `Text.spans` may carry either a
   `rich.style.Style` or a style string, so normalize with `Style.parse()` before
   checking the `underline` attribute; the marked span legitimately also covers a fold
   glyph and a dim ` · N` suffix, so scope the case check to the underlined sub-spans.
   Assert the complementary half too: roster rows and family lane sub-headings resolve
   as `FOLD_ONLY`.
2. **`resolve_section_target()` skips fold-only anchors** (new unit test): a document
   with title, fold-only, title anchors cycles title → title in both directions, and a
   document with only fold-only anchors resolves to `EMPTY`.
3. **`resolve_section_at_row()` still resolves fold-only anchors** (new unit test) —
   this is the regression guard for `za` / `zA` on a roster row.
4. **`get_content_height()` reserve** (new or extended unit test): reserve is computed
   from the last title, not from a trailing roster row.
5. `tests/ace/tui/widgets/test_prompt_panel_section_navigation.py::test_clan_members_section_is_a_navigation_target`
   — keep the rendered-id list assertion (`member:research.member` is still a marked
   anchor) and extend it so the forward cycle goes `members` → `errors`, never
   `member:research.member`.
6. `tests/ace/tui/test_agents_panel_fold_mounted.py` (~lines 114-140): the `Ctrl+J`
   sequence currently expects `members` → `member:sase-mounted.phase` →
   `member:sase-mounted.land` → `errors`. Update it to `members` → `errors` and keep a
   per-member fold assertion by driving the fold from the viewport-top path instead
   (scroll the `#agent-prompt-scroll` container so the member row is the top row, then
   press `z a` and assert the `member:…` override).
7. `tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py` (~lines
   230-250): the same rework. The `agents_family_panel_member_override_120x40` golden
   must keep covering a per-member override; reach the row by scrolling rather than by
   `Ctrl+J`. Also re-check the "press `Ctrl+J` until the cursor wraps to `None`" loop
   above it, which now needs fewer presses. If the resulting viewport genuinely differs,
   accept the new golden with `--sase-update-visual-snapshots` only after inspecting the
   diff artifacts under `.pytest_cache/sase-visual/`, and run the suite with
   `just test-visual`.
8. `tests/ace/tui/widgets/test_member_roster.py` (~lines 190-235, 320-335) and
   `tests/ace/tui/widgets/test_agent_display_tribe_roster.py` (~lines 295-315): these
   assert entry anchor ids from marked spans. Keep the id assertions and add the role
   assertion (fold-only).
9. **Family `SASE CONTEXT`** (new): on a family container document, `sase-context` is a
   `TITLE` anchor and each lane id (`bead`, `plan`, `artifacts`, `memory-reads`,
   `glossary-reads`, `skill-uses`, `opened-workspaces`) is `FOLD_ONLY`; on a non-family
   agent document, `sase-context` remains the only marked id (the existing assertion in
   `tests/ace/tui/widgets/test_agent_context_lanes.py:209` must still pass unchanged).
   Also assert the family `SASE CONTEXT` line's rendered text is unchanged by the
   re-marking (no doubled or missing newline).

## Verification

- `just install` first if this workspace has not been used recently.
- `just check` for the lint gates plus the diff-scoped test lane. Note this change
  touches shared render helpers, so also run `just check-full` through `/sase_monitor`
  with the `TESTING` / `TESTED` status pair before landing.
- `just test-visual` for the PNG snapshot suite (`tests/ace/tui/visual/`), since the
  family-panel goldens are in scope.
- Manual smoke check in `sase ace`: select a family container row, press `Ctrl+J`
  repeatedly, and confirm the stops are only `FAMILY SHELLS`, `NEIGHBORS`,
  `SASE CONTEXT`, `SLOW TOOL CALLS`, `AGENT XPROMPT` / `AGENT PROMPT` / `AGENT REPLY`,
  etc. — never an individual shell row and never a `BEAD` / `PLAN` lane label. Then
  scroll so a shell row is the top row and confirm `za` still folds just that row.

## Non-goals

- **No new keybindings.** See the risk below; if the owner wants a dedicated way to walk
  fold anchors, that is a separate change.
- **No feature flag.** Per `sase/memory/sase_flags.md`, flags exist for unproven
  user-reaching behavior or for keeping a deprecated branch reachable. This is a bug fix
  that aligns behavior with the already-documented contract; the old branch does not
  need to stay reachable, and an agent should not create a `beta` flag outside an epic.
- **Kind headers stay chrome.** `FAMILY`, `CLAN`, `TRIBE`, and `AGENT SHELL` identity
  lines are ALL-CAPS and underlined but are deliberately _not_ navigable
  (`append_kind_header()`, documented at `docs/ace.md:4138` and `:4142`). The rule being
  enforced here is one-directional — only titles may be jumped to — so leave them alone.
- **Making the family `SASE CONTEXT` title fold its lanes** is a behavior addition, not
  part of this fix.

## Risks and trade-offs

- **Per-entry and per-lane fold overrides get harder to reach.** Today the ergonomic
  path is `Ctrl+J` onto a roster row (which top-aligns it) followed by `za`. After this
  change the only way to put such a row at the viewport top is scrolling (`ctrl+f` /
  `ctrl+b`, half/full page), which is coarse. The behavior is preserved and documented,
  but the ergonomics regress. If that proves annoying in practice, the natural follow-up
  is a fold-mode sub-key pair (for example `z j` / `z k`) that walks fold anchors within
  the active section and top-aligns them — it costs no new top-level keybinding because
  fold mode already owns a sub-key namespace with an `agents` dict in the keymap
  registry. Do not implement it as part of this plan; raise it with the project owner.
- **Visual snapshot churn.** The family-panel PNG goldens are the most likely source of
  incidental failures; treat any pixel diff as something to inspect, not to accept
  blindly.
- **Two collection paths.** `SectionTrackingVisual` collects anchors in both
  `render_strips()` and `get_height()`. A role added to only one path produces anchors
  whose role depends on which measurement ran; change both together.
