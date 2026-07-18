---
tier: tale
title: Artifacts detail scrolling and linked plan content
goal: 'Artifacts list navigation uses Ctrl+F/B for ten-entry jumps, Ctrl+D/U scrolls
  each active right-hand detail pane, and Plans bead details include their linked
  plan document without adding filesystem work to the UI thread.

  '
create_time: 2026-07-18 16:26:18
status: done
prompt: 202607/prompts/artifacts_detail_navigation_and_plan_content.md
---

# Plan: Artifacts detail scrolling and linked plan content

## Context

The non-PR Artifacts panes currently route both pairs of global scrolling actions into the left-hand entry navigator:
`scroll_detail_down/up` selects entries at offsets of ten and `scroll_prompt_down/up` selects entries at offsets of
five. That overrides the established meaning of `Ctrl+D/U` as half-page detail scrolling even though Commits, Bugs, and
Plans already have dedicated right-hand `VerticalScroll` containers. The Plans detail pane already shows
pending-proposal and archived-plan bodies, but epic and phase rows show only bead metadata, description, and notes; the
epic's `design` reference to the committed plan is not loaded into the pane.

Keep the existing configurable action names and default key assignments. This is a context-specific behavior correction,
not a keymap schema migration: custom bindings for the same actions must receive the corrected behavior automatically.

## Detail scrolling and fast list navigation

- Give the non-PR Artifacts surface a small presentation-level contract for locating or scrolling the active pane's
  right-hand detail viewport. Commits should target `commits-detail-scroll`, Bugs should target `bugs-body-scroll`, and
  Plans should target `plans-detail-scroll`; the app-level navigation mixin should not fall through to the hidden PR
  detail pane while any of those sub-tabs is active.
- Route `scroll_detail_down/up` (`Ctrl+D/U` by default) through that contract and move the active right pane by half of
  its visible height without animation. The action must leave the left-list selection and focus unchanged, clamp through
  Textual's normal scrolling behavior, and remain a harmless no-op if the active pane is not mounted or has no overflow.
  Preserve the existing PR, Agents, and AXE behavior outside non-PR Artifacts.
- Change the non-PR interception of `scroll_prompt_down/up` (`Ctrl+F/B` by default) from offsets of five to offsets of
  ten. Retain stable-target selection, disabled-heading skipping, boundary clamping, immediate highlighting, detail
  debouncing, and focus restoration already supplied by the shared Artifacts entry navigator.
- Keep `g/G`, single-step `j/k`, jump hints, the action allowlist, and the keymap/default-config schema unchanged.
  Update the Artifacts help rows and user documentation so they describe `Ctrl+F/B` as ten-entry list navigation and
  `Ctrl+D/U` as right-detail scrolling.

## Linked plan documents in Plans details

- Extend the immutable Plans snapshot with a lightweight linked-plan document result keyed by project and owning epic.
  During the existing background snapshot load, resolve each epic's non-empty `design` reference against its project
  workspace and canonical plans/SDD root, supporting the stable absolute, `sdd/plans/...`, `.sase/sdd/plans/...`, and
  plans-root-relative forms already persisted by plan approval. Deduplicate reads when an epic and its phases share the
  same document.
- Perform all path resolution, stat/read work, and frontmatter/body preparation inside the existing worker-backed data
  collection path. Include enough document identity in the snapshot/cache invalidation inputs for an explicit refresh or
  a changed linked plan file to publish fresh content; never read or probe the filesystem from `_update_detail`, an
  OptionList highlight handler, a render helper, or a key action.
- Treat a phase with no direct design reference as inheriting its parent epic's linked document. A missing, unreadable,
  or malformed reference must not fail the rest of the project's snapshot: retain the bead detail and surface a compact,
  deterministic unavailable message at the plan section instead of leaking an exception or stale content. Projects and
  epics without a design reference should keep their current detail presentation.
- For epic and phase rows with a linked document, preserve the current property grid and bead description/notes first,
  then append a clearly separated Plan section containing the linked plan file's complete readable content at the bottom
  of the existing Markdown detail widget. Keep proposal and archive rows on their current specialized rendering paths so
  their already-present plan bodies are not duplicated. Ensure selection, filtering, scope changes, refreshes, and epic
  expand/collapse always derive the appended document from the currently selected stable row and current snapshot.

## Tests and verification

- Update the shared Commits/Bugs/Plans navigation integration coverage to prove `Ctrl+F/B` moves exactly ten selectable
  entries with clamping while `Ctrl+D/U` changes the active detail viewport's scroll offset without changing the
  selected entry. Exercise both directions with overflowing right-pane content and retain coverage for configured action
  bindings.
- Add Plans data tests for linked-plan resolution and loading across canonical reference forms, phase inheritance,
  deduplicated worker-side reads/cache invalidation, and isolated missing/unreadable-file results. Add pane tests
  proving the plan content is appended after bead description/notes for epic and phase selections, is absent for
  unlinked beads, and is not duplicated for proposal/archive rows.
- Update help-modal assertions, `docs/ace.md`, and the populated Plans PNG fixture/golden where the appended section is
  intentionally visible. Inspect visual diffs before accepting any golden update.
- Before handoff, run `just install`, the focused Artifacts navigation/Plans/help tests, `just test-visual` for affected
  snapshots, and the repository-mandated `just check`.

## Acceptance criteria

- On Artifacts -> Commits, Bugs, or Plans, `Ctrl+F/B` selects ten selectable left-pane entries down/up and `Ctrl+D/U`
  half-page scrolls only the active right pane.
- PRs, Agents, AXE, custom keymap overrides, list focus, stable jump targets, and existing detail debouncing retain
  their current behavior.
- Selecting an epic or any of its phases shows the linked committed plan document beneath the bead detail when
  available, with refresh-safe and non-blocking fallback behavior when it is not.
