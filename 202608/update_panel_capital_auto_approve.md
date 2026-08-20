---
tier: tale
title: Capital-letter auto-approve shortcuts for the Update panel
goal:
  Add beautiful, reliable capital-letter companions that preserve normal update planning
  while deliberately skipping the final confirmation prompt.
size: medium
proposed_by: bbugyi200.athena.08q
create_time: 2026-08-20 13:29:45
status: wip
---

# Capital-letter auto-approve shortcuts for the Update panel

## Goal

Give each selectable Update-panel scope a capital-letter companion: `E`, `S`, `P`, and
`A` select Everything, SASE, Providers, and Agents respectively, exactly like the
existing lowercase keys, but automatically approve the resulting update instead of
opening the final `y`/`n` confirmation. Preserve the panel's keyboard-first feel and
make the faster path obvious without doubling the four-row information hierarchy.

## Product and safety contract

| Scope                | Preview key | Auto-approve key |
| -------------------- | ----------- | ---------------- |
| Everything           | `e`         | `E`              |
| SASE, core & plugins | `s`         | `S`              |
| Providers            | `p`         | `P`              |
| Agents               | `a`         | `A`              |

- Lowercase keys, `Enter` on the highlighted row, and mouse selection retain the
  existing preview-then-confirm behavior. Capital keys are the only new auto-approve
  entry points. `r`/`R` is not part of this mapping because re-check is not an update
  scope.
- Auto-approve skips only the final confirmation modal. It still runs the same
  off-thread preview planner, revalidates/captures the same inputs, handles preview
  errors and already-current scopes without mutation, and submits through the same
  tracked comprehensive-update proc with its existing deduplication and exclusive
  mutation scopes.
- Keep the Update panel presentation-only and I/O-free. Carry auto-approve as explicit,
  immutable typed intent; do not infer it later from key casing, scope, or global UI
  state.
- Keep four rows. Render each row's paired shortcut compactly (for example `e/E` with
  the capital action given a consistent `⚡`/warm-accent treatment), and replace the
  crowded one-line hint with a balanced two-line legend that clearly distinguishes
  lowercase “preview” from uppercase “apply now · no prompt.” Keep navigation/re-check/
  close hints secondary. Let the hints widget size to its content so neither line clips
  at the panel's supported width.

## Implementation

1. Extend `src/sase/ace/tui/modals/update_panel.py` so `UpdatePanelResult` includes an
   `auto_approve: bool = False` field. Add capital `E`/`S`/`P`/`A` bindings and route
   lower- and uppercase actions through one guarded selection helper that accepts the
   explicit approval mode. Preserve `_chose`'s one-dismiss guarantee, missing-row
   guards, lowercase behavior, Enter behavior, mouse behavior, re-check, and cancel.
   Update row-prefix and hint rendering with structured Rich `Text` so the normal and
   accelerated paths have a clear, theme-compatible hierarchy without backend imports.
2. Adjust the Update-panel hint styling in `src/sase/ace/tui/styles.tcss` only as needed
   for the two-line legend (automatic height/spacing and no clipping). Do not add rows,
   widen the modal unnecessarily, or introduce work on the keystroke/render path.
3. In `src/sase/ace/tui/actions/base.py`, copy the result's explicit approval mode into
   the `ComprehensiveUpdateRequest` created at the panel callback boundary. Extend the
   frozen request model in
   `src/sase/ace/tui/modals/plugins_browser_comprehensive_update_models.py` with an
   `auto_approve: bool = False` default so every existing Admin Center and test caller
   remains confirmation-first unless it deliberately opts in.
4. In `src/sase/ace/tui/actions/update_run.py`, keep preview failure and non-runnable
   handling ahead of approval routing. For a runnable preview whose request is
   auto-approved, call the existing `_submit_scoped_update_task(preview)` directly and
   return without constructing/pushing `PluginActionConfirmModal`; otherwise retain the
   current confirmation modal and callback unchanged. Do not duplicate execution logic
   or relax the mutation proc's dedup key, exclusive scopes, completion handling,
   restart behavior, or result toasts.
5. Update the Update-panel descriptions in `docs/ace.md`, `docs/configuration.md`, and
   `docs/plugins.md` so they document paired lowercase/capital keys and state precisely
   that capital keys skip `y`/`n` only after the normal preview succeeds. Rephrase broad
   “every mutation confirms” claims where needed so the documented exception is
   unambiguous.

## Tests and visual verification

- Expand `tests/ace/tui/test_update_panel.py` to cover all four lowercase keys returning
  `auto_approve=False`, all four capital keys returning the matching scope with
  `auto_approve=True`, and Enter/mouse remaining non-auto-approved. Assert the paired
  row labels and both legend meanings render, while existing single-dismiss, re-check,
  refresh/highlight, cancel, and presentation-only invariants continue to pass.
- Extend `tests/ace/tui/test_update_panel_shortcut.py` and/or
  `tests/ace/tui/test_log_panel_keymap.py` to prove the panel callback carries the
  explicit flag into the exact `ComprehensiveUpdateRequest`, with the default path still
  false and provider projection still captured at callback time.
- Extend `tests/ace/tui/test_update_run_actions.py` to prove a runnable normal request
  still opens `PluginActionConfirmModal`, a runnable auto-approved request pushes no
  confirmation and submits the existing scoped mutation proc exactly once for the
  selected scope, and auto-approved preview errors/no-op previews fail closed without a
  mutation. Retain assertions for deduplication, exclusive scopes, and scoped display/
  command-line names.
- Update both Update-panel PNG goldens through the intentional snapshot-update workflow,
  inspect the pending and never-checked images for aligned `e/E`, `s/S`, `p/P`, `a/A`
  affordances and an unclipped, centered legend, then run the dedicated visual suite.
- Before verification, run `just install` as required for an ephemeral workspace. Run
  the focused Update-panel/update-run tests, `just test-visual` for the visual goldens,
  and finally `just check`; if scoped verification escalates or reports unusual
  selection, follow project guidance and run `just check-full` through `/sase_monitor`.

## Acceptance criteria

- Pressing `E`, `S`, `P`, or `A` from the Update panel plans and, when the captured
  preview is runnable, starts exactly the same tracked update as `e`, `s`, `p`, or `a`
  followed by `y`, without displaying the confirmation modal.
- Lowercase keys, Enter, and mouse activation still require confirmation, and no-op or
  failed previews never mutate even when initiated with a capital key.
- The four paired shortcuts and their different consequences are readable at a glance;
  no hint text clips or crowds the status chips/details in either visual snapshot.
- Update execution ordering, captured-scope semantics, manual-provider reporting,
  concurrency guards, restart/toast behavior, and Update-panel re-check behavior remain
  unchanged.
