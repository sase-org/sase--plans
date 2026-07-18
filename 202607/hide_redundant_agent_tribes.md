---
tier: tale
title: Hide redundant tribe badges in split agent panels
goal: 'Agent rows omit tribe badges already communicated by their enclosing split
  panel while merged panels and otherwise ambiguous clan rows retain tribe context.

  '
create_time: 2026-07-18 06:25:21
status: done
prompt: 202607/prompts/hide_redundant_agent_tribes.md
---

# Plan: Hide redundant tribe badges in split agent panels

## Context and behavior

The Agents tab already treats tribe labels as panel context for ordinary agent rows: split panels are titled with their
tribe, while the merged `,g` view supplies effective tribe labels directly to each row. Synthetic clan rows bypass that
distinction because the row formatter always renders every value in `Agent.clan_tags`. A single-tribe clan therefore
repeats `@epic` (or another tribe) inside an `@epic` panel, as shown in the referenced screenshot and the
`agents_clan_panel_epic_120x40` visual snapshot.

Keep the underlying agent/clan metadata, panel membership, detail panel, grouping controls, navigation, and folding
semantics unchanged. Make only the row presentation panel-aware:

- In split mode, omit a tribe badge when the enclosing panel title already identifies that same tribe. A single-tribe
  clan in the `@epic` panel should render its clan name and status without a second `@epic` badge.
- In merged mode, retain the effective tribe badges that make rows self-describing after `,g` removes the tribe-specific
  panel titles. Clan rows must continue to show all of their distinct tribes without duplicates.
- Do not suppress useful context merely because split mode is active. In particular, a multi-tribe clan that necessarily
  appears in the untagged panel should retain its `@tribe` badges because no enclosing tribe title represents them.
- Continue showing tribe metadata in the selected clan's detail pane; this request concerns agent rows only.

## Implementation

Thread the enclosing panel's tribe/display context from the full and selective panel refresh paths in
`src/sase/ace/tui/actions/agents/_display_panel_widgets.py` through `AgentList.update_list()` and the
row-building/rendering pipeline. Use that context when composing clan-row badges so the formatter filters only the badge
duplicated by a split panel, while preserving the existing effective-tag labels used by merged mode and the stable
ordering/deduplication of `clan_tags`.

Treat the panel context as an explicit render input rather than mutating `Agent.tag` or `Agent.clan_tags`. Record it in
the per-row render context used by `patch_agent_row()` and include it in the agent render-cache key. This keeps mode
toggles, selective refreshes, and later row patches from reusing a row rendered for a different panel context, while
preserving the established fast paths and avoiding any new I/O or full-refresh mechanism on the UI thread.

Keep the change within the Python TUI presentation boundary. No Rust core or persisted agent metadata changes are
needed.

## Tests and validation

Add focused regression coverage around the formatter/widget boundary and panel wiring:

- A single-tribe clan row omits the matching badge in its split tribe panel.
- The same clan row shows its tribe in merged mode, including after the grouping state changes.
- A multi-tribe clan in the untagged split panel retains all nonredundant tribe badges in deterministic order.
- The row render cache and patch path distinguish split-panel context from merged/untagged context so mode changes and
  selective updates cannot leave stale badges.
- Existing ordinary-agent effective-tag behavior in merged mode remains intact.

Update the intentional `agents_clan_panel_epic_120x40` PNG golden (and any other directly affected clan-row golden) so
the split `@epic` panel title remains visible while its clan row no longer repeats `@epic`. Inspect generated
actual/expected/diff artifacts before accepting the snapshot change.

Run `just install` first as required for an ephemeral workspace, then run the focused Agents-tab model/widget/panel
tests, `just test-visual`, and the repository-wide `just check` gate.
