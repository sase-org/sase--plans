---
tier: tale
title: Cut Patches over to the shared inline filter bar
goal:
  Make the Patches Artifacts pane use the shared boolean query profile, persistent
  FilterBar, Rust query index, and pane-scoped query state while preserving Patch
  grammar, hide controls, stable selection, and hidden-pane isolation.
size: medium
proposed_by: bbugyi200.athena.sase-m6.6.1.6
bead: sase-m6.6.1.6
create_time: 2026-08-15 19:49:29
status: wip
---

- **PROMPT:**
  [prompts/202608/patch_inline_filter_bar.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/patch_inline_filter_bar.md)
- **PARENT:** [202608/unified_artifacts_query_1.md](unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.6.md)

# Plan: Cut Patches over to the shared inline filter bar

## Goal

Complete phase `sase-m6.6.1.6` by moving the Patches pane from its modal query editor
and compatibility-only Patch corpus wrapper onto the same contract-configured inline
filter and profile-driven Rust evaluation path already used by the other Artifacts
panes. Preserve the complete boolean dialect, make live editing reversible and
responsive, and ensure every saved-query, history, selection, and project-scope action
mutates only the active Patches pane.

## Grounding and invariants

- Use `ArtifactsPaneContract.query_profile` for Patches. Its compiled profile already
  declares `boolean=true`, the Patch fields, `+`/`^`/`~`/`&` sigils, `%` status macros,
  and closed host predicates; do not introduce a second Patch grammar or widget-owned
  dialect table.
- Build one immutable `ArtifactQueryIndex` from the full Patch snapshot on the existing
  background load path and evaluate edits through the shared bounded
  `ArtifactQuerySession`. Query compilation/evaluation, Patch row conversion, disk
  reads, project discovery, and profile compilation must not run on the Textual event
  loop or render path.
- Preserve Patch-only presentation semantics around query matches: hide-reverted and
  hide-submitted remain post-query controls, but an explicit terminal/submitted target
  in the parsed boolean expression still disables the corresponding hide control.
- Keep committed query source, parsed expression, canonical form, filtered rows, and
  selected `ArtifactEntryTarget` coherent. An invalid live edit remains visible in the
  bar without replacing the committed result; Escape restores the last committed query
  and selection.
- Keep the bar persistent when idle, show profile-derived completions and observed
  facets, and report a truthful live match count plus exact/preview coverage. Cached
  worker results must be generation/profile/query keyed and rejected when stale.
- Scope all durable state to pane id `patches`. Loading a slot, navigating history, or
  restoring a target must never switch to or mutate a hidden Artifacts pane. Loading a
  Patch slot from outside Artifacts may still intentionally open the Patches pane.
- Preserve compatibility APIs outside this TUI migration. `QueryEditModal` may remain
  for non-Patch consumers, and legacy Patch query/corpus entry points remain available
  until the conformance phase decides what can be retired.

## Implementation

1. Add the Patches query-row/index boundary to the shared Artifacts query layer. Build a
   generation-tagged `ArtifactQueryIndex` from Patch rows using the contract's compiled
   boolean profile, retain observed facets for completion, and install the index
   atomically with each full Patch snapshot. Replace the TUI's compatibility
   `QueryCorpus` cache/evaluate route with the generic profile corpus while keeping the
   old facade available to non-migrated callers and tests.

2. Introduce a Patch-specific `FilterBar` adapter configured from the contract profile
   and mount it persistently in `ArtifactsPatchesPane` where `SearchQueryPanel`
   currently renders the committed query. Apply the Patches accent and pane-specific
   DOM/message ids, expose completion values from the current query index, render live
   match counts/coverage/errors, and remove only the obsolete Patch query-panel/modal
   styling and rendering path.

3. Implement a reversible Patch filter session around the shared `ArtifactQuerySession`.
   Opening captures the committed source, filtered result, and stable selected target;
   valid edits parse/canonicalize with the Patch profile and request generation-keyed
   Rust results without reading disk; invalid edits preserve the committed view; Enter
   commits one coherent query transition; Escape restores the captured query, result,
   target, and Patch-list focus. Reuse the existing hide-status introspection and
   selection-by-identity helpers when applying live or committed matches.

4. Consolidate Patch query transitions behind one helper used by inline submit, saved
   slots, previous/next history, and external Patch-target rewrites. The helper records
   the prior `QueryRecord`, saves/restores the current `ArtifactEntryTarget`, updates
   source/AST/canonical state, refilters from the warm index, persists last-query state
   only on commit, and restores the nearest stable selection without touching hidden
   pane widgets.

5. Move the `#N`/`#` saved-query grammar into the Patch bar submit handler. Preserve
   explicit-slot save/delete, next-free-slot allocation, duplicate/move behavior,
   profile digests, cache invalidation, and notifications; advertise the save grammar
   through completion/help text. Loading a Patch slot and using the `*` picker should
   apply through the same transition helper, while non-Patch pane namespaces remain
   untouched.

6. Complete the contextual actions. Add a Patches filter action bound to `f` alongside
   the existing configurable `/` edit-query route, resolving the current `f` Patch
   action collision explicitly in default/fallback keymaps and help/command metadata.
   Allow the shared `p` picker on Patches and rewrite only the committed positive
   `project:` scope token: replace it for a selected project, remove it for All
   projects, and preserve every other boolean term, grouping, negation, and source
   intent. Route both actions to Patches only when it is the active pane.

7. Extend focused and contract coverage. Test Patch profile grammar/sigil/macro parity
   on the generic Rust route; persistent-bar completion, live counts, invalid-query
   behavior, submit and Escape rollback; `#N` save/delete; `f` and `/` dispatch;
   project-only query rewriting; hide-status behavior; slot/history target restoration;
   stale-result rejection; and pane-switch isolation. Update Patch Artifacts visual
   snapshots only after inspecting the modal-to-inline differences, and leave the
   Agents/Axe modal behavior covered unchanged.

## Verification

1. Run `just install` before tests so the editable package and current Rust binding are
   synchronized.
2. Run focused query-profile/corpus parity, Patch loading/filtering, FilterBar,
   saved-query/history/selection, keymap/help, project-scope, and Artifacts pane tests
   while iterating.
3. Run the dedicated ACE PNG visual suite for affected Patches states, inspect
   actual/expected/diff artifacts, and accept only the intended persistent-bar layout.
4. Run the Artifacts navigation benchmark/trace checks and confirm Patch navigation p95
   remains below 16 ms with no profile compilation, row conversion, resolver, disk, or
   Git work in typing/navigation paths.
5. Run `just check`. If it escalates unusually or the change touches the repository's
   broadening set, use the prescribed monitored `just check-full` path. Record any
   unrelated defect only as a `PROPOSED FOLLOW-UP:` note on `sase-m6.6.1.6`.
