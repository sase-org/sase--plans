---
tier: tale
title: Shared config transaction and AXE schema-form components
goal:
  Config Center and AXE entry editors share responsive, validated, conflict-safe
  transaction and typed object-form infrastructure.
bead: plans-7.2
parent: sase/repos/plans/202607/axe_config_editor.md
create_time: 2026-09-09 19:53:16
status: wip
---

- **BEAD:** plans-7.2

# Shared config transaction and schema-form components

## Outcome and boundaries

Implement the `plans-7.2` phase as a reusable, event-loop-safe editing foundation rather
than wiring AXE actions into the running AXE tab. The result will preserve the Config
Center's existing scalar/list/map behavior and snapshots, provide a pure schema-object
form model and generic property picker, and expose an AXE lumberjack/chop editor modal
that `plans-7.3` can open with backend-specific inventory, plan, apply, and restart
callbacks.

This is a `tale`: the transaction refactor, form model, picker generalization, and AXE
surface share contracts and should be implemented and verified together by one coding
agent. It does not alter AXE keymaps, tree collection, daemon reconciliation, selection
restoration, or documentation/snapshot goldens assigned to later phases. It also does
not reimplement layered AXE composition or entry mutation planning assigned to
`plans-7.1` and the linked Rust core.

## Implementation

1. Extract the reusable config transaction lifecycle from `ConfigEditModal`.
   - Add focused transaction types/protocols for immutable display metadata, writable
     scopes, a caller-built mutation request, a normalized preview (target path,
     effective before/after, diagnostics/warnings, and diff), and an apply outcome. Keep
     adapters thin so the existing `EditPlanResult`/`AppliedResult` path and the AXE
     entry planner can use the same controller without either UI importing the other's
     backend types.
   - Move scope selection, overlay creation, plan/apply worker startup, busy/error
     state, preview scrolling/rendering, chezmoi propagation, and stale-write conflict
     recovery into a reusable controller/mixin plus preview renderer. Config Center
     supplies its current single-field operation builder and AXE supplies sparse field
     operations.
   - Run overlay rebuilding, planning, apply/chezmoi, reload/re-plan, inventory/schema
     loading, and any file/git work in thread workers. Keep mount/call-after-refresh
     callbacks synchronous, tag results so superseded workers cannot mutate newer state,
     re-read mounted/current state before applying completions, and cancel every
     outstanding worker on unmount.
   - Treat the backend's typed stale-plan conflict distinctly from ordinary errors:
     retain the caller's draft, return to an actionable edit/preview state, and allow an
     asynchronous reload/re-plan instead of overwriting the external change. Keep
     compatibility with the current generic config error surface while the backend phase
     lands.
   - Migrate `ConfigEditModal` and its rendering base to the shared transaction path
     while preserving public imports, widget IDs, key behavior, typed scalar/list/YAML
     editors, scope/overlay semantics, diff layout, dismissal result, and existing
     snapshots. Keep post-write commit discovery in the existing off-thread Config Pane
     flow.

2. Add a pure reusable schema-object form model.
   - Build ordered property descriptors from a supplied object-schema node, resolving
     local `$ref`, `allOf`, and compound `oneOf`/`type` shapes well enough to expose
     descriptions, required/default status, enums, numeric/string constraints, and a
     raw-YAML fallback for every compound value.
   - Represent each property with immutable effective value/provenance plus mutable
     draft state that distinguishes untouched, explicitly set, and reset/inherit.
     Required and optional properties remain deterministically ordered; callers can
     classify fields into Basics and Advanced without hard-coding AXE behavior into the
     generic model.
   - Reuse/generalize the Config Center editor-kind, formatting, constraint, YAML,
     boolean, enum, numeric, string, and Vim-text-area helpers. Keep bounded live
     parsing for large YAML and perform full parsing/validation only in the planning
     worker.
   - Provide a pure conversion from the form draft to exact-segment field operations.
     Emit only touched set/reset operations, preserve dynamic identities as opaque path
     segments, and surface per-field parse errors without mutating the effective/source
     data.

3. Generalize the frontmatter add-property picker.
   - Introduce a configurable `PropertyPickerModal` and neutral property descriptor
     supporting custom title/guidance, deterministic accelerators, j/k and arrow
     navigation, enter/escape, direct accelerator selection, mouse row selection, and
     detailed schema guidance.
   - Retain `AddPropertyModal` and `AddableProperty` as a thin frontmatter-specific
     wrapper/compatibility surface so the Prompt Frontmatter Panel's behavior, styling,
     imports, and visual snapshots do not change.
   - Use the generic picker from the schema-object form for optional/advanced fields,
     including compound fields whose editor is the raw-YAML escape hatch.

4. Build the standalone AXE entry editor surface on those shared pieces.
   - Add public immutable context/callback types for lumberjack versus base-chop
     identity, add versus edit mode, writable scopes, effective/raw property values and
     provenance, generated-instance warning text, and whether the current save choice
     implies restart. The modal accepts already-loaded context; it performs no inventory
     or script discovery on mount or navigation.
   - Provide an edit stage with AXE gold/copper styling, compact identity/status header,
     scope rail, Basics/Advanced property rows, focused typed editor, inline source
     badges, optional-property picker, reset/inherit control, validation feedback, and
     persistent Vim/key hints. Lumberjack fields are identity, interval, optional
     `chop_timeout`, and advanced `env`; chop fields are identity, script, description,
     enabled, optional `run_every` and `timeout`, plus advanced `env`, `inhibit_if`,
     `trigger`, `once_per`, `for_each`, and `vars`. Identity is immutable in edit mode
     and editable when adding.
   - Feed the sparse exact-segment operations through the caller's immutable request
     builder into the shared transaction controller. The preview stage names the target,
     renders effective before/after, strict/generic diagnostics and warnings, shows the
     source-preserving diff, and labels save-only versus save-and-restart effects
     unambiguously. Back preserves all draft values and reset state.
   - Make the layout responsive: normal terminals use the intended scope/property/editor
     columns, while constrained widths collapse to a readable single-column flow without
     hiding validation, editor, preview, or hints.

5. Add focused regression and component coverage.
   - Add pure tests for schema dereferencing/order, required/optional grouping, editor
     selection and typed parsing, provenance, touched/reset semantics, exact dynamic key
     segments, sparse operation conversion, constraints, and compound/raw-YAML fallback.
   - Add picker tests for deterministic accelerators, keyboard boundaries, direct keys,
     mouse selection, guidance, and the unchanged frontmatter wrapper behavior.
   - Add Textual tests for the shared transaction controller and AXE editor:
     scope/overlay changes, property navigation and editing, validation/reset,
     generated-instance and provenance rendering, preview/back draft retention, save
     consequence labels, stale-conflict reload/re-plan, worker supersession/cancellation
     after unmount, and constrained-width layout.
   - Keep all current Config Center edit tests and relevant Frontmatter Panel tests
     green, adding explicit regression assertions for its unchanged widget IDs,
     dismissal results, chezmoi target handling, and no synchronous slow work in
     mount/navigation/event handlers.

## Validation

1. Run `just install` because this is an ephemeral workspace, then run the focused
   pure/model, property-picker, AXE-editor, Config Center modal, and Frontmatter Panel
   test modules while iterating.
2. Run `just fmt` as needed and `just check` as the required repository-wide
   verification.
3. Re-run the complete targeted Config Center and new AXE editor suites after
   `just check` so the final result directly demonstrates transaction, cancellation,
   draft-retention, and narrow-layout behavior.

## Completion

Close only `plans-7.2` after all implementation and validation work passes. Leave parent
epic `plans-7` open, do not create beads, and preserve `plans-7.3` as the downstream
integration phase.
