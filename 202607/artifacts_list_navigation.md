---
tier: tale
title: Fast navigation for non-PR Artifacts lists
goal: 'Commits, Bugs, and Plans support consistent first/last, fixed-distance, and
  apostrophe-hint navigation over their left-panel entries without changing the existing
  PRs navigation behavior or degrading TUI responsiveness.

  '
create_time: 2026-07-17 08:46:59
status: done
prompt: 202607/prompts/artifacts_list_navigation.md
---

# Plan: Fast navigation for non-PR Artifacts lists

## Context and boundaries

The Artifacts tab retains the historical top-level `changespecs` identity. Its non-PR action allowlist currently blocks
the app's existing `scroll_to_top`, `scroll_to_bottom`, `scroll_detail_*`, `scroll_prompt_*`, and `jump_to_entry`
actions, while the implementations behind those actions assume a PR detail panel or another top-level tab. Commits,
Bugs, and Plans already own independent `OptionList`-style selections and app-level `j`/`k` actions, but their row
structures differ: Commits inserts disabled day headings, Plans inserts disabled section and empty-state rows, and Bugs
contains a second focusable links list that must not become the target of artifact-entry navigation.

This is presentation-only Textual work in the Python TUI. It does not require a Rust-core or wire-format change. Reuse
the existing configurable app actions and their keys, so custom keymaps continue to work and
`src/sase/default_config.yml` does not need new fields or changed defaults. The PRs sub-tab must retain its current
detail scrolling, grouped-row navigation, and apostrophe history.

## Required interaction semantics

When Artifacts is visible and the active sub-tab is Commits, Bugs, or Plans, route the existing configured actions to
the active left-hand entry list:

| Default key | Existing action      | Non-PR Artifacts behavior                        |
| ----------- | -------------------- | ------------------------------------------------ |
| `g`         | `scroll_to_top`      | Select the first entry                           |
| `G`         | `scroll_to_bottom`   | Select the last entry                            |
| `Ctrl+D`    | `scroll_detail_down` | Move down 10 entries                             |
| `Ctrl+U`    | `scroll_detail_up`   | Move up 10 entries                               |
| `Ctrl+F`    | `scroll_prompt_down` | Move down 5 entries                              |
| `Ctrl+B`    | `scroll_prompt_up`   | Move up 5 entries                                |
| `'`         | `jump_to_entry`      | Enter the standard one-key hint mode for entries |

An "entry" is a currently visible, selectable commit, bug, proposal, epic, expanded phase, or archived plan. Disabled
day/section/empty-state rows do not receive hints and do not count toward the 5- or 10-entry distance. Fixed-distance
movement clamps at the first or last entry instead of wrapping; empty lists are silent no-ops. Every successful movement
focuses the primary left list and lets its existing immediate-highlight/debounced-detail path update the right panel.
The Bugs linked-work list remains independently navigable through its existing bindings.

## Design and implementation

1. Introduce a small shared navigation contract for the three non-PR Artifacts panes. It should expose selectable
   targets in current visual order, the selected target, selection by target/relative selectable position, and a way to
   apply or clear transient jump hints. Keep target identities stable across ordinary repaints: use repository plus full
   SHA for commits, scoped issue identity for bugs, and the existing project-aware row identity for plans. Resolve
   positions from the current target list at the moment of the keypress so disabled rows, expansion changes, filters,
   and refreshed ordering cannot make an index stale.

2. Adapt each pane without duplicating navigation policy. Commits should map between commit targets and its timeline
   options while skipping day banners; Bugs should operate only on `BugIssueList`; Plans should derive targets from its
   `_rows`/option IDs and skip disabled headings. Preserve current selection by stable identity whenever options are
   rebuilt. Continue to guard programmatic `OptionList.highlighted` assignments synchronously so they do not echo as
   user navigation, and continue to debounce only detail rendering, never the highlight itself.

3. Add one app-level non-PR Artifacts dispatcher and allow the seven existing actions above through
   `NON_PRS_ARTIFACT_ACTIONS`. The global action handlers should branch to that dispatcher only for Commits, Bugs, and
   Plans, and delegate to their current behavior for PRs, Agents, and AXE. Route first/last and fixed-distance moves
   through the existing Artifacts activity gate and key-to-paint instrumentation, with useful per-sub-tab action labels.
   Do not add disk access, collection, subprocess work, awaited callbacks, or option reconstruction to ordinary movement
   paths.

4. Extend the existing entry-jump machinery with a non-PR Artifacts branch rather than creating an unrelated key-event
   mode. Build hints with the shared jump alphabet in current selectable visual order, preserve case-sensitive dispatch
   through `normalize_jump_key`, and render the usual compact hint marker into each entry prompt. Entering mode must
   preserve the highlight, show the shared `JUMP` footer with `' first` or `' back`, and rebuild only the in-memory
   options needed to add hints. A hint key selects but does not activate the entry; Escape or an invalid key exits
   cleanly and restores normal rows and footer state.

   Keep apostrophe history isolated by Artifacts sub-tab and store stable target identities rather than raw option
   positions. With no valid history, pressing apostrophe again while hints are visible selects the first entry; after a
   hint jump, the same `''` interaction returns to the most recent valid origin. Drop stale history targets lazily when
   filters, scope, refreshes, or plan expansion remove them. Switching top-level tabs or Artifacts sub-tabs, changing
   scope, or replacing the active row model must cancel transient hint display without leaking markers or a `JUMP`
   footer into the destination pane. Keep the existing PR/Agents/AXE fast-back and forward-jump behavior unchanged; this
   tale's required new surface is the apostrophe interaction requested for the three non-PR lists.

5. Update the Artifacts help content and `docs/ace.md` to distinguish PR detail scrolling from non-PR list selection.
   Document configurable first/last, 10-entry, 5-entry, and apostrophe first/back behavior in the Commits, Bugs, and
   Plans sections without adding these global actions to the panes' normal conditional footer. The transient shared
   footer remains the discoverability surface while jump mode is active.

## Verification

- Add focused pane/unit coverage for selectable-target ordering and stable selection. Include commits spanning multiple
  day banners, plans with disabled section rows and expanded phases, and bugs with a populated linked-work list.
- Add pilot coverage on all three sub-tabs using enough entries to prove exact `g`/`G`, +/-10, and +/-5 behavior,
  boundary clamping, disabled-row skipping, focus restoration, empty-list no-ops, and unchanged detail selection. Verify
  the non-PR dispatcher is selected only for Commits, Bugs, and Plans; PRs, Agents, and AXE must retain their existing
  semantics. Include a configured keymap override to prove routing follows action names rather than hard-coded default
  keys.
- Cover apostrophe mode on all three row renderers: visual-order hint assignment, hint selection without activation,
  uppercase hints, `''` first/back behavior, per-sub-tab history isolation, stale-target pruning, Escape/invalid-key
  cleanup, footer restoration, and cleanup during sub-tab/scope/data changes. Assert that disabled headings and the Bugs
  links list never receive entry hints.
- Update help/key-display assertions and add or extend representative PNG snapshots for the transient hint state while
  confirming ordinary Artifacts snapshots remain unchanged.
- Extend the existing Artifacts navigation performance benchmark (or add a focused companion) to exercise the new
  fixed-distance/first-last paths with realistic list sizes. Preserve the established p95 key-to-paint target below 16
  ms and confirm hint entry/exit performs only bounded in-memory repainting.
- Before handoff, run `just install`, targeted Artifacts navigation/jump/help tests, `just test-visual` for intentional
  snapshot additions, the slow Artifacts navigation benchmark, and the repository-mandated `just check`.

## Risks and safeguards

- Raw `OptionList` indices are unsafe because Commits and Plans interleave disabled rows and all panes can reorder on
  refresh. Central selectable-target resolution and stable identities prevent distance errors and stale jump-back
  destinations.
- Rebuilding options can emit highlight messages and cause cursor/detail churn. Use each pane's synchronous
  programmatic-update guard and identity-preserving rebuild path; ordinary bulk navigation should only change the
  highlight.
- A global jump-mode flag can outlive the pane that painted its hints. Make sub-tab/tab lifecycle cleanup explicit and
  idempotent, and test it through actual key events rather than only helper calls.
- Hint rendering must not start data work or bypass the existing activity gate. Keep all candidate discovery in
  already-loaded in-memory row models and retain the current cached/background refresh architecture.
