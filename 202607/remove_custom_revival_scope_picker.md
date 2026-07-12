---
create_time: 2026-07-11 13:02:09
status: done
prompt: 202607/prompts/remove_custom_revival_scope_picker.md
tier: tale
---
# Plan: Open Custom Revival Search Directly on Recent Dismissed Agents

## Summary

Remove the intermediate project/PR picker from the **Custom revival search...** path in the **Agent Revive** panel.
Selecting that row should open the existing dismissed-agent selection panel directly, let its current background loader
materialize the newest 250 top-level dismissed agents (including each selected workflow parent's child rows), and put
focus in the panel's filter bar. Users can then narrow the loaded rows by agent, project, or PR/ChangeSpec text and
revive one or several agents as they do today.

Keep the established archive-performance behavior: bundle/index work stays off the Textual event loop, the initial
archive query remains bounded to 250 visible parent rows, and `Ctrl+K` can continue loading older 250-row pages when a
user intentionally needs history beyond the initial recent set. This change removes the mandatory scope-selection step;
it does not remove the general-purpose project picker from other workflows or remove dismissed-archive paging.

## Current behavior and relevant contracts

The current route is:

1. `R` on the Agents tab opens `SavedAgentGroupRevivalModal`.
2. Selecting **Custom revival search...** returns `SavedAgentGroupRevivalResult(action="custom_search")`.
3. `AgentReviveFlowMixin._open_custom_revival_search()` opens `ProjectSelectModal(include_all=True)`.
4. The selected `SelectionItem` is passed to `AgentReviveArchiveMixin._show_dismissed_agents_for_scope()`, which scopes
   both the archive-index query and the in-memory rows before opening `DismissedAgentSelectModal`.
5. On mount, that modal loads a 250-parent page through `load_dismissed_bundles_page()` in `asyncio.to_thread()`, then
   supports local filtering, multi-selection, previews, revival, and optional `Ctrl+K` paging.

Important existing invariants to preserve:

- `load_dismissed_bundles_page()` is already newest-first, using the dismissed-bundle SQLite index ordered by
  `COALESCE(start_time, raw_suffix) DESC`. Its limit counts top-level/visible rows and it loads children sharing those
  parents' suffixes so workflow step counts and previews remain complete.
- Same-session dismissed objects are merged with archive-loaded objects and deduplicated by agent identity, ensuring a
  recently dismissed agent remains revivable even if persistence/index reconciliation is still catching up.
- Archive loading and dismissed-projection repair are separate background paths. Removing the picker must not move JSON
  parsing, index maintenance, response-corpus reads, or projection repair onto the Textual event loop.
- The filter already searches display/canonical PR names, named-agent labels, and response text. A project selection,
  however, currently narrows the index before the filter panel opens; once that picker is removed, project identity must
  be added to the filter corpus so users retain that capability locally.
- `selection_scope` on single/batch revival is optional and only enriches audit-log fields. The actual artifact restore,
  dismissed-set update, bundle purge, and refresh logic does not require a scope.

## Desired behavior / acceptance criteria

- Choosing **Custom revival search...** transitions directly from the Agent Revive panel to `DismissedAgentSelectModal`;
  no project, PR/ChangeSpec, home, or ALL picker is shown.
- The dismissed-agent panel retains its loading placeholder while the newest archive page loads and then renders up to
  250 recent top-level dismissed agents without requiring a keystroke. Workflow children remain available for parent
  step counts, previews, and implicit child revival.
- The filter input is focused and can narrow the loaded set by project name (canonical or humanized), PR/ChangeSpec
  name, agent/workflow name, named-agent handle, and the response corpus already supported by the modal.
- Selecting or marking rows still invokes the existing single/batch revival operations. Custom-search audit events no
  longer claim a user-selected scope; agent identities and outcomes remain logged normally.
- `Ctrl+K` paging, marks, current filter text, highlighted identity, preview behavior, loading/error states, and
  dismissed-projection repair continue to work as before.
- The Agent Revive preview copy for the custom row accurately explains that it opens the recent dismissed-agent list and
  that the user should filter there; it no longer describes a project/PR-scoped search.
- Other call sites of `ProjectSelectModal` are unchanged.

## Implementation approach

### 1. Route custom search directly to the dismissed-agent panel

In `src/sase/ace/tui/actions/agents/_revive_flow.py`, simplify `_open_custom_revival_search()` so it calls the archive
panel entry point directly instead of importing/pushing `ProjectSelectModal` and waiting for a `ProjectSelectResult`.
This keeps the saved-group modal's result contract (`action="custom_search"`) intact while removing only the
intermediate screen.

### 2. Make the custom archive entry point unscoped

In `src/sase/ace/tui/actions/agents/_revive_archive.py`:

- Replace the scope-oriented entry point with an unscoped custom-search entry point. Its page-loader closure should call
  `load_dismissed_bundles_page(limit=250, offset=...)` without `cl_name` or `project_name`, merge/deduplicate the
  returned rows with same-session dismissed objects, split visible parents from child rows, and feed those results to
  `DismissedAgentSelectModal`.
- Preserve the current initial-page-on-mount and `Ctrl+K` offset/exhaustion behavior rather than eagerly parsing the
  full archive. Preserve the current ordering behavior unless a focused regression test demonstrates that the unscoped
  transition needs a newest-first presentation adjustment; the archive membership itself must remain the newest page.
- Keep recently dismissed in-memory agents available during the loading state and after the first archive merge. Do not
  introduce a second unbounded archive list merely to replace the old scopes.
- Invoke `_do_revive_agent()` / `_do_revive_agents()` without a fabricated `SelectionItem`; the absence of a scope is
  the truthful audit representation for a filter-based search.
- Remove now-dead picker-specific helpers/imports (scope-to-index arguments, scope filtering, `SelectionItem`, and
  `Path` if no longer needed), while retaining the independently scheduled dismissed-projection repair.

The optional `selection_scope` parameters in the lower-level revival execution methods can remain for compatibility with
any other direct callers; this UI change should not broaden into an unrelated logging API rewrite.

### 3. Preserve project/PR discoverability in the filter bar

In `src/sase/ace/tui/modals/revive_agent_modal.py`, extend the cheap per-agent filter label/corpus with project identity
derived from the agent's project metadata. Include both canonical and user-facing project forms where they differ,
alongside the existing canonical/humanized PR/ChangeSpec name and named-agent fields. This makes typing the project that
a user previously would have selected produce the expected subset, including ordinary PR agents whose row label does not
itself display the project.

Keep filtering local to the rows already loaded. Avoid per-keystroke file I/O: reusable response/project corpus data
should be prepared once per loaded page, and any response-content reads performed while applying a new 250-row page
should stay in the existing background-loading boundary (or be moved there if the refactor exposes that they currently
run on the event loop).

### 4. Update explanatory UI copy

In `src/sase/ace/tui/modals/saved_agent_group_revival_rendering.py`, replace the custom-row preview text that says it
opens an existing project/PR-scoped search. Describe the direct behavior concisely: it loads the most recent 250
dismissed agents and the next panel's filter bar narrows them. Keep the row label and `custom_search` action unchanged.

No style changes should be necessary. The standalone `ProjectSelectModal`, its exports, styles, and tests remain because
other agent/workflow entry points still use it.

## Tests and verification

Update/add focused coverage for these contracts:

- `tests/test_agent_group_revival_routing.py`
  - Replace the test expecting `ProjectSelectModal` with a test proving the custom result pushes
    `DismissedAgentSelectModal` directly.
  - Assert the custom panel's first page loader requests the global dismissed archive with `limit=250`, `offset=0`, then
    advances offsets for existing load-more behavior without passing project/PR scope arguments.
  - Exercise the selection callback for one and multiple rows, confirming it dispatches to the existing revive methods
    without a synthetic scope.
- `tests/ace/tui/modals/test_revive_agent_modal.py`
  - Add filter tests showing project names discriminate agents from different projects even when they share a
    PR/ChangeSpec name, and showing both canonical and humanized project identifiers remain searchable.
  - Retain the existing initial-page rendering, off-thread/error, mark/filter/highlight preservation, and `Ctrl+K`
    paging tests as regressions for the direct unscoped flow.
- `tests/ace/tui/modals/test_saved_agent_group_revival_rendering.py` and jump/behavior tests as appropriate
  - Assert the custom-row preview describes recent-250 loading and in-panel filtering, with no scoped-picker wording.
- `tests/ace/tui/visual/test_ace_png_snapshots_saved_groups.py`
  - Regenerate only the saved-group snapshot(s) whose custom-search preview text intentionally changes, inspect the
    actual/expected/diff artifacts, and accept those goldens with `--sase-update-visual-snapshots` only after confirming
    the layout change is expected.

Run targeted routing/modal tests while iterating, then `just test-visual` for the affected PNG coverage and the required
`just check` after `just install` before completion.

## Edge cases and non-goals

- **No archive rows / failed load:** preserve the existing disabled empty/loading placeholder and error notification;
  the removed picker must not turn an archive failure into an accidental revive attempt.
- **Fewer than 250 rows:** the first page marks itself exhausted and the `Ctrl+K` hint remains absent.
- **More than 250 rows:** the initial searchable set is bounded; older rows remain opt-in through existing paging.
- **Same-session/index overlap:** continue identity-based deduplication so a row appears once and recent not-yet-indexed
  dismissals are not lost.
- **Workflow parents:** continue loading all children for each included parent even though the visible-page limit counts
  parents, not child bundles.
- **Filter semantics:** filtering does not query the entire on-disk archive; it narrows the currently loaded recent page
  (and any explicitly loaded older pages), matching the bounded-performance intent.
- **Saved groups:** recent/saved group loading, deletion, revival, and their separate pagination are unchanged.
- **Other project selection flows:** launching custom agents/workflows and other `ProjectSelectModal` users are out of
  scope.
- **Backend boundary:** no `sase-core` change is needed. This is presentation/routing around an existing Python TUI
  archive API; archive ordering, paging, persistence, and revival domain behavior remain unchanged.
