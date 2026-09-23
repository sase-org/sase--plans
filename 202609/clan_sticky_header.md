---
tier: tale
title: Show the sticky agent header panel for agent clan rows
goal:
  Selecting an agent clan on the Agents tab shows the sticky, collapsible identity
  header panel titled CLAN, with the clan's identity fields moved out of the scrolling
  metadata body.
size: medium
proposed_by: bbugyi200.athena.0ps
create_time: 2026-09-23 08:53:40
status: wip
---

# Plan: Show the sticky agent header panel for agent clan rows

## Context

Epic `sase-16k` added a sticky, collapsible identity header panel (`AgentHeaderPanel`,
`src/sase/ace/tui/widgets/agent_header_panel.py`) above the Agents-tab metadata panel.
Builders "detach" the identity block from the document they render: they return an
`AgentHeaderRenderable` whose `identity_header` slot holds an `IdentityHeader`
(`prompt_panel/_identity_header.py`: `kind_label`, `accent`, `expanded`, `compact`,
`has_hints`). `AgentPromptPanel.update()` calls `find_identity_header(content)` and
publishes the result to `AgentDetail._on_identity_header`, which shows the panel when an
identity is present and hides it when the identity is `None`.

Clans were explicitly left out of that epic, and they are excluded in exactly one place.
`build_header_text()` (`prompt_panel/_agent_display_header.py`, the
`if agent.is_clan_container:` early return near the top) calls
`build_clan_detail_text()` (`prompt_panel/_agent_display_clan.py`) and ignores
`detach_identity`. So every clan document is a plain `Text`, publishes `None`, and the
header panel hides.

Everything else already works for clans without changes:

- Every clan render path already passes
  `detach_identity=getattr(self, "detaches_identity_header", False)` into
  `build_header_text`. That covers `_update_display_impl` in `_agent_display_render.py`
  (debounced full render, also re-run after clan disk-section enrichment),
  `update_header_only` in `_agent_display.py` (immediate j/k path), and
  `_update_clan_display_with_hints` in `_agent_display_hint_render.py` (hint mode,
  wrapped in `CachedRenderable`, which `find_identity_header` already unwraps through
  `.renderable`).
- The widget, visibility sync, the `d` toggle availability, zoom seeding
  (`inline_document_renderable`), and the identity-change scroll reset are all
  kind-agnostic.

This is presentation-only Textual/Rich code, so it stays in this repo. No `sase-core`
change is needed.

## Design

### What the user sees for a selected clan row

```
╭─ CLAN ────────────────────────────────────────────────────────────╮
│  sase-16k · RUNNING ● 2 ◐ 1 ✓ 3                                   │
│  @epic · 6 agents · 1 family · 12m 04s · ▸ 1/3                    │
╰──────────────────────────────────────────────────────── ▾ d more ─╯
┌───────────────────────────────────────────────────────────────────┐
│ ━━ CLAN MEMBERS ━━ … (roster, summary, sections — scrolls)        │
```

- **Title and accent.** The border title is `CLAN` in `_CLAN_IDENTITY_COLOR` (orchid,
  from `_agent_list_styling`), the same color as today's inline kind line. The panel
  dims it for the border the same way it does for every other kind.
- **Expanded (`d`).** Exactly today's inline clan identity lines in their current
  styles, minus the kind line (it is now the title):
  - `Name:`
  - `Tribes:`, only when the clan has tribes
  - `Status:`, with the queued extras and the count chip
  - `Runtime:`
  - `Members:`
  - the `Fold:` line from `append_fold_header_line(..., scale=CLAN_FOLD_SCALE)`

  Putting `Fold:` in the identity matches the family and tribe documents, so `z` presses
  still show feedback.

- **Collapsed (default).** Two fixed, non-wrapping rows that end in `…` when truncated,
  built with the shared `_chips_row` / `_compact_text` conventions:
  - Row 1, who and state: the clan name (`_CLAN_NAME_STYLE`, falling back to
    `agent.display_name` like the `Name:` line), then the status in its
    `_MEMBER_STATUS_STYLES` bucket style, then the count chip
    (`format_agent_count_chip`, omitted when it is empty). The layout mirrors row 1 of
    `build_tribe_compact_lines`.
  - Row 2, composition: up to 3 `@tribe` chips in their tribe identity styles, then a
    dim `+N`; the members summary (`N agents · M families`, same wording as the
    `Members:` line); the runtime (`bold #BCBCBC`); and the fold chip (same glyph and
    color as `_fold_chip`, using `CLAN_FOLD_SCALE`).
- **Body.** The scrolling metadata document now starts at the `CLAN MEMBERS` roster,
  followed by the summary, errors, variables, replies, context, slow tools, prompts, and
  the scanning tail, all unchanged. Strip leading newlines with the existing
  `strip_leading_document_chrome`.
- **Hints.** The clan identity lines contain no file paths, so `has_hints` is normally
  false. Still compute it from `hint_state.hint_counter` before and after the identity
  build, the same way the agent path does, so a future hint-bearing identity field
  forces expansion correctly.
- **Everything else unchanged.**
  - Non-detached clan output (the `Z` zoom modal's own panel, the `V` pager, direct
    builder callers, and existing tests) must stay byte-identical.
  - Member-roster jump numbering, fold anchors, and `Ctrl+J`/`Ctrl+K` section targets
    are unchanged. `CLAN` was already unmarked chrome and the first target is still
    `CLAN MEMBERS`.

## Implementation

1. **New module**
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_identity.py`. Keeping this
   code in a new module keeps `_agent_display_clan.py` (about 450 lines) and
   `_identity_header_compact.py` (about 435 lines) under the `toobig` limits.
   - Move the identity field block out of `build_clan_detail_text`, from `Name:` through
     `append_fold_header_line`, into
     `append_clan_identity_fields(text, agent, *, counts, agent_count, family_count, fold_level, now)`.
     It must produce exactly today's lines and styles. Move or import the
     `_FIELD_LABEL_STYLE` and `_MEMBER_STATUS_STYLES` constants as needed and keep a
     single definition of each.
   - Add
     `build_clan_compact_lines(*, agent, counts, agent_count, family_count, fold_level, now) -> Text`,
     which returns the two rows described in Design. Reuse `_chips_row`,
     `_compact_text`, `_fold_chip`, and `_CHIP_SEPARATOR_STYLE` from
     `_identity_header_compact.py`. Import them, or promote them to public helpers if
     symvision objects to cross-module private use (see `sase/memory/symvision.md`). Use
     only in-memory data: agent fields, the already computed `clan_member_counts`
     result, and the member and family totals. No disk I/O.
   - Compute the counts, member totals, and runtime once in `build_clan_detail_text` and
     pass them to both functions, so the compact and expanded forms cannot disagree.
2. **`build_clan_detail_text(..., detach_identity: bool = False) -> Text | AgentHeaderRenderable`**
   (`_agent_display_clan.py`):
   - Not detached: call `append_kind_header(text, "CLAN", _CLAN_IDENTITY_COLOR)` and
     then `append_clan_identity_fields(text, ...)`, keeping output byte-identical to
     today.
   - Detached:
     - Record `hint_state.hint_counter` if `hint_state` is set.
     - Build `identity_text` with `append_clan_identity_fields` and no kind line.
     - Build the body `Text` starting at the roster, reusing the rest of the function
       unchanged. The simplest approach is to append all remaining sections to a `body`
       variable that is the same object as `text` in non-detached mode.
     - Strip leading chrome with `strip_leading_document_chrome`.
     - Return
       `AgentHeaderRenderable(body, (), identity_header=IdentityHeader(kind_label="CLAN", accent=_CLAN_IDENTITY_COLOR, expanded=identity_text, compact=build_clan_compact_lines(...), has_hints=...))`.

     The `_append_tribe_body` split in `_agent_display_tribe.py` is the closest
     precedent.

   - Keep the `member_jump_map_publisher` call and all hint-state side effects in the
     same order, so hint numbering is unchanged.

3. **`build_header_text`** (`_agent_display_header.py`): pass
   `detach_identity=detach_identity` through to `build_clan_detail_text` in the clan
   early return. Update the return annotation or type comments if mypy requires it.
   `AgentHeader` already admits `AgentHeaderRenderable`.
4. **Audit the clan call sites**:
   - Confirm that each of the three clan call sites listed in Context now publishes a
     non-`None` identity in detached mode, including the cheap immediate path, where
     `clan_snapshot` may be only the in-memory snapshot.
   - In `AgentDetail` (`_agent_detail_display.py`, the `agent.is_clan_container` branch
     that calls `_expand_prompt_only()`), confirm the header ends up visible. The
     `_sync_header_visibility()` calls in `_agent_detail_panels.py` should already cover
     this. Add a sync only if a pilot test shows stale visibility.
   - Check that the `,/` metadata search, clan section enrichment re-renders, and fold
     chords (`z` / panel fold) repaint the header, so the fold chip and `Fold:` line
     track the level. They should, because each re-render republishes the identity.
5. **Tests**:
   - `tests/ace/tui/widgets/test_identity_header.py`: replace
     `test_clan_documents_carry_no_identity` with tests that check:
     - A detached clan document carries an identity with title `CLAN` and the orchid
       accent.
     - The body excludes `CLAN`, `Name:`, `Status:`, `Runtime:`, `Members:`, `Tribes:`,
       and `Fold:`, and starts at the `CLAN MEMBERS` roster.
     - The expanded identity's plain text and spans equal the non-detached document's
       identity region, minus the kind line.
     - The non-detached clan document is unchanged, for example against a small expected
       plain-text or line-list fixture built before the refactor.
     - Hint mode: the summary hints are still numbered from 1 in the body, and
       `has_hints` is false.
   - `tests/ace/tui/widgets/test_identity_header_compact.py`: clan compact output is
     exactly two lines. Cover:
     - name, status, and count chip on row 1;
     - tribe chips with the `+N` overflow at more than 3 tribes;
     - the member summary with singular and plural forms;
     - runtime;
     - the fold chip at each clan fold level;
     - a clan with no tribes.
   - `tests/ace/tui/widgets/test_agent_header_panel.py`: invert
     `test_clan_selection_hides_header` into `test_clan_selection_shows_header`. It
     should check that the panel is visible, the border title contains `CLAN`,
     `header_toggle_available()` is true, the panel is collapsed at two rows, and `d` /
     `toggle_header_expanded()` expands it to the full field list and back. Add a check
     that moving between a solo agent and a clan keeps the expanded/collapsed state.
   - Check the clan pilot and section-navigation tests that read clan identity fields
     from `#agent-prompt-panel` and point them at the header panel, or use the
     header-plus-body text helper introduced by `sase-16k` if one exists. Start with:
     - `tests/ace/tui/widgets/test_prompt_panel_section_navigation_targets.py`, whose
       direct builder calls are non-detached and probably unaffected;
     - `tests/ace/tui/test_agents_panel_fold_mounted.py`;
     - `tests/ace/tui/actions/test_view_files_agent_hints.py`;
     - `tests/ace/tui/test_agent_detail_two_phase.py`.

     Also grep for `Members:` and `CLAN` assertions under `tests/ace/tui`.

6. **Docs** (`docs/ace.md`, "Agents Tab Metadata Panel"):
   - Update the `CLAN / MEMBERS` bullet: the orchid `CLAN` kind label is now the header
     panel title, the identity fields (Name, Tribes, Status, Runtime, Members, Fold)
     live in the header panel, and the body starts at `CLAN MEMBERS`.
   - In the `Header panel` bullet, add `CLAN` to the list of kind titles. Change "hidden
     for clan rows and 'No agent selected'" to "hidden only for 'No agent selected'".
     Describe the clan's collapsed rows briefly, or say that they mirror the tribe
     layout.
7. **Visual verification**. Read `sase/memory/tui.md` first with `/sase_memory_read`.
   - Capture a live screenshot with a clan row selected, both collapsed and after `d`,
     and check the title color, alignment with the body, and truncation.
   - Then run `just fix-tui-screenshots` through `/sase_monitor`. The clan-panel goldens
     will change, including
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py`,
     `test_ace_png_snapshots_agents_clans.py`, and the clan collapse snapshots. The
     `assert_page_svg_contains(page, "CLAN")` checks should still pass because `CLAN` is
     in the border title.
   - Inspect every created, removed, and updated golden group per the golden-maintenance
     rules. Only clan-selected snapshots should change. Any non-clan golden diff is a
     regression to fix, not accept.

## Verification

- Run `just check` via `sase tool run check`, per the `lint_and_test` memory note.
- Run the targeted tests:
  - `tests/ace/tui/widgets/test_identity_header*.py`
  - `tests/ace/tui/widgets/test_agent_header_panel.py`
  - the clan pilot and visual tests listed above
- The golden refresh and review from step 7 is complete.
