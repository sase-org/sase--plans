---
tier: tale
title: Collapsed agent panel summary pane
goal: 'Selecting a collapsed Agents-tab panel shows a fast, accurate, and polished
  summary of that panel instead of stale metadata and actions for an arbitrary hidden
  agent.

  '
create_time: 2026-07-17 14:35:14
status: wip
prompt: 202607/prompts/collapsed_agent_panel_summary.md
---

# Plan: Collapsed agent panel summary pane

## Context and product outcome

An Agents-tab panel is a real navigation target when it is collapsed, but the right pane currently continues to treat
the panel's backing `current_idx` as an agent selection. That leaks the first or previously selected hidden agent's
metadata, file/tools state, and conditional actions into a screen whose yellow focus border clearly says that the panel
itself is selected.

Make collapsed-panel focus a first-class presentation state. The left pane will retain its compact title-only panel
strip, while the right pane will become a scrollable summary of the focused panel. Expanded panels and collapsed _group
banners inside_ an expanded panel keep their existing behavior; this tale is specifically about whole collapsed tag
panels.

The finished interaction should make three things immediately clear:

1. which panel is selected (`#tag` or `(untagged)`) and that it is collapsed;
2. how its members are distributed across attention, active, waiting, failed, unread, and completed states; and
3. which agents and nested workflow/family members the panel contains, without implying that any one hidden row is
   selected.

## Summary-pane experience

Add a dedicated collapsed-panel summary surface within the existing `AgentDetail` area, using the same semantic colors,
glyphs, labels, and status projection as the Agents list and count chips rather than creating a parallel visual
language.

The pane should contain:

- a restrained eyebrow such as `COLLAPSED AGENT PANEL`, a prominent gold-accented panel label, and a composition line
  that distinguishes total rendered entries, top-level roots, and nested workflow/family members when those counts
  differ;
- a zero-suppressed status overview using the established stopped/running/waiting/ failed/unread/done categories and
  palette, so its numbers agree exactly with the selected panel's title chip and parallel-family projection;
- an `AGENTS` roster grouped in the shared operational priority order (attention and failure before autonomous work and
  completion), preserving stable panel order within a bucket and nesting child entries beneath their parent; and
- compact per-row context drawn only from already-loaded `Agent` fields: display name, exact display status, project or
  ChangeSpec context when useful, model, and compact elapsed/completion time. Marked and unread members should retain
  their familiar accents. Missing optional metadata must simply be omitted, not replaced with noisy placeholders.

Use a full-height `VerticalScroll` with the selected-panel gold as its border accent, generous but terminal-efficient
spacing, aligned status/context columns where width permits, and graceful Rich/Textual wrapping at narrower widths. The
roster remains complete and scrollable for large panels; do not silently truncate members. Avoid an independent timer:
normal cached agent refreshes are the source of truth for status and runtime changes.

When the summary is visible, the Agents info strip should say `view: summary`. Agent-only file/tools indicators and
footer actions (retry, tmux, tag, edit chat, kill one hidden row, and similar) must disappear. Explicit app-wide or
marked-set actions may remain because they do not claim that a hidden agent is selected. Expanding the panel selects its
existing first rendered stop and restores normal agent detail; preserve the user's prior file/tools/info detail-mode
preference rather than resetting it merely because the summary was visited.

## Selection and data model

Introduce one authoritative resolver for the focused collapsed-panel state. It should require all of the following: the
Agents tab is active, a panel collection exists, the focused panel key still exists, and that key is in
`_collapsed_panel_keys`. Build an immutable presentation snapshot from the memoized
`AgentPanelIndex.slice_for(focused_key)` plus the in-memory unread and marked identity sets. Keep label formatting and
aggregation in pure helpers so tagged, untagged, empty/transient, child-heavy, parallel-family, and mixed-status panels
can be unit tested without mounting the app.

Update `_get_selected_agent()` to return `None` while this resolver is active. The backing `current_idx` remains an
internal re-expansion anchor, but it is no longer exposed as a user selection. This also makes existing actions and
clean auto-refresh checks fail closed instead of operating on an invisible row. Do not change
`_agents_in_focused_panel()`, since bulk cleanup and other explicitly panel-scoped flows still need the complete panel
slice.

Keep this model in the Python TUI layer: it is a presentation projection of data already supplied by the backend, not
new shared backend/domain behavior.

## Detail lifecycle and reliability

Compose a dedicated summary widget/scroll region alongside the current prompt, file, and tools regions in `AgentDetail`,
with mutually exclusive visibility. Give `AgentDetail` explicit methods for showing a panel snapshot and returning to
agent/empty content instead of reaching into child widgets from app actions.

Both the immediate navigation path and the full/debounced detail path must resolve collapsed-panel focus before
resolving an agent. Summary rendering is synchronous and in-memory: do not hydrate attempt history, inspect files,
resolve aliases, start file/tools/diff workers, or schedule a redundant slow detail phase. Showing a summary must
advance the existing detail generation (and clear the current agent render identity) so any worker started for the
previously selected agent is unable to repaint over the summary. Likewise, late enriched-header messages must be ignored
while panel focus is active.

Route collapse, expand, keyboard panel switching, jump-hint panel selection, mouse/focus changes, list refreshes, and
status/unread/mark refreshes through this same resolver and rendering path. Re-read the current focused key and collapse
set at application time; never apply a snapshot captured before an await. A collapsed sibling panel must not affect the
detail for an agent selected in an expanded panel.

Keep the keystroke path compatible with the TUI performance rules:

- consume the existing panel-index slice instead of rescanning files or stores;
- keep immediate rendering bounded to pure formatting of the focused panel;
- preserve the current detail debouncer for real agent selections and avoid new timer/call-later callbacks; and
- invalidate/repaint only the summary and existing info/footer state when panel membership or status changes, without
  rebuilding unrelated list panels.

## Verification

Add focused unit and integration coverage for:

- tagged and untagged snapshot labels, zero-suppressed status totals, unread and marked accents, stable priority
  ordering, nested child placement, and parallel family count agreement with existing panel-title semantics;
- the selection contract: a focused collapsed panel returns no selected agent, while a collapsed non-focused panel and
  every expanded panel retain normal agent selection;
- collapse, expand, next/previous-panel, jump-hint, and refresh transitions, including restoration of the correct first
  rendered stop and the user's detail mode;
- immediate and full detail routing: summaries spawn no hydration/file/tools work, increment the generation, reject
  stale async agent results, update live summary data, and return cleanly to agent detail;
- info-strip/footer behavior (`view: summary`, no hidden-agent conditional actions) and fail-closed agent actions while
  a panel is selected; and
- large and narrow summary rendering without member truncation or crashes.

Update the existing collapsed-panel interaction visual fixtures and their PNG goldens so the primary collapsed state,
jump-hint state, and panel-fold selection state visibly exercise the new right pane. Strengthen those tests with
semantic assertions for the panel label/status roster and the absence of the former hidden agent metadata, so
correctness is not entrusted to pixels alone.

Before handoff, run `just install`, the targeted model/widget/navigation/detail/ footer tests, and the dedicated
collapsed-panel visual tests. Accept only the intentional PNG changes, inspect the generated images at the standard
120x40 size, then run `just test-visual` and the repository-required `just check`. If the navigation formatting adds
measurable work, compare the existing Agents j/k benchmark or `SASE_TUI_PERF=1` trace and keep p95 key-to-paint within
the documented 16 ms target.

## Non-goals and safeguards

- Do not change persisted panel-fold formats, panel ordering, or collapse/expand keybindings.
- Do not turn a collapsed whole-panel selection into a collapsed in-panel group summary; group-banner behavior can be
  designed separately.
- Do not add disk reads, subprocesses, backend API changes, or independently refreshed state to the summary.
- Do not make destructive actions implicitly panel-wide. A panel-scoped cleanup remains an explicit, confirmed flow;
  ordinary single-agent actions have no target while the summary is selected.
