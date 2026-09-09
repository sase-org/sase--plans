---
tier: tale
title: Fix task-move spacing and destination-picker lifecycle
goal:
  Ctrl+Shift+M moves one or more tasks into the chosen note without synthetic blank
  lines between task blocks, and the destination picker closes immediately when a
  destination is selected.
create_time: 2026-09-09 19:53:24
status: wip
---

# Plan: Fix task-move spacing and destination-picker lifecycle

## Context

The counted task-move feature was added to the linked `bob-plugins` repository in the
Bob Navigation Hotkeys plugin. Investigation identified two independent symptoms inside
that one cohesive workflow:

- Destination formatting currently injects an empty line while flattening separate moved
  task blocks and injects another empty line before appending moved blocks to a
  non-empty `## Tasks` section. Together, these make moved tasks appear
  blank-line-separated even when the source tasks and destination task list are
  contiguous.
- The shared filtered-picker selection path awaits `commitTaskMoveSession`, which scans
  and updates the affected vault notes, before it closes the modal. The task destination
  menu therefore remains visible during the asynchronous move instead of disappearing as
  soon as the user commits a destination.

The linked `bob-plugins` repository is the source of truth. Open it through
`sase repo open bob-plugins` before implementation, make all plugin changes there, and
deploy the completed plugin through `bob plugins sync`; do not edit the vault's
installed plugin files directly.

## Implementation

1. Correct destination task-list assembly in `plugins/bob-navigation-hotkeys/main.js`.
   - Stop adding synthetic separators between independently selected task blocks when
     they are flattened for insertion. Preserve the original order and any blank lines
     that are genuinely inside a captured task subtree, but make the boundary between
     one moved root task and the next root task contiguous.
   - Refine insertion into an existing `## Tasks` section so an appended moved task is
     contiguous with the preceding task rather than being prefixed by an empty line.
     Continue to preserve structural spacing after the section header, before the
     following section, the destination's existing line-ending style, terminal-newline
     behavior, placeholder replacement, and creation of a missing Tasks section for area
     notes.
   - Keep source-range removal, task subtree rebasing, identity migration,
     dependency/link rewriting, destination validation, and transactional write/rollback
     behavior unchanged except where an end-to-end spacing assertion demonstrates that a
     boundary helper must participate.

2. Make destination selection dismiss only the task-move picker immediately.
   - Add a task-picker-specific close-on-selection path (or a narrowly scoped option in
     the shared filtered picker) that marks the selection as in progress, closes the
     modal before starting/awaiting the asynchronous commit, and still awaits the
     existing commit promise so failures remain handled through the current notices and
     rollback flow.
   - Close after every valid destination selection even if the later move is rejected
     because the source or destination became stale. Retain the single-flight guard so
     Enter/click cannot start duplicate moves, and leave the close-on-success semantics
     of child-note, link, path, and property pickers unchanged.

3. Add focused regressions in `scripts/test-navigation-hotkeys.cjs`.
   - Strengthen the task-block flattening/insertion tests with exact output assertions
     covering multiple counted tasks, a destination that already has tasks, placeholder
     replacement, an area that needs a new Tasks section, CRLF input, and
     terminal-newline preservation. Assert no synthetic blank line occurs between
     adjacent root tasks while internal subtree spacing and section boundaries remain
     intact.
   - Extend the cross-file move-planning test to assert the exact destination task
     sequence, including adjacency between pre-existing and moved tasks and between
     multiple moved tasks, while retaining identity and link-rewrite coverage.
   - Exercise the picker lifecycle with a deferred asynchronous commit: selecting a
     destination must call `close` synchronously before the commit settles, invoke the
     commit once, ignore a duplicate selection, and remain closed whether the commit
     eventually succeeds or returns a guarded failure. Also retain coverage that
     unrelated filtered pickers keep their existing delayed close-on-success contract if
     shared picker code is changed.

4. Release, validate, and deploy the corrected plugin.
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.13.8` to the next patch
     version and keep the Bob Navigation Hotkeys version in `README.md` synchronized;
     the existing feature description can remain unless the implementation changes
     user-facing behavior beyond these fixes.
   - Run the focused navigation suite with
     `node --test scripts/test-navigation-hotkeys.cjs`, then run the repository-wide
     `npm test` and `npm run validate` checks.
   - Deploy only Bob Navigation Hotkeys with
     `bob plugins sync -p bob-navigation-hotkeys`, without forcing over dirty vault
     files, and verify the sync/list output reports the installed plugin at the new
     manifest version. If the deploy refuses because the vault copy is dirty, stop and
     report that state rather than overwriting it.

## Acceptance criteria

- Bare and counted `Ctrl+Shift+M` moves produce contiguous task roots in the
  destination, including when appending to an existing task list, without disturbing
  blank lines internal to a moved subtree or structural spacing around Markdown
  sections.
- Choosing a destination by keyboard or mouse removes the menu immediately, before vault
  reads/writes finish, and cannot launch the same move twice.
- Stale-state and transactional failure notices/rollback guarantees still work after the
  picker has closed.
- Focused and full tests pass, all plugin manifests validate, the manifest and README
  versions agree, and the updated source-of-truth plugin is synced to the vault.
