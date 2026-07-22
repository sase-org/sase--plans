---
tier: tale
title: Shared config transaction and AXE schema-form components
goal: Config Center and AXE entry editing share responsive, typed, scope-aware transaction
  and schema-form primitives.
bead: sase-8m.2
parent: sase/repos/plans/202607/axe_config_editor.md
create_time: 2026-07-22 12:04:07
status: wip
---

- **PROMPT:** [202607/prompts/shared_config_editor.md](prompts/shared_config_editor.md)

# Shared config transaction and AXE schema-form components

## Goal

Deliver the reusable UI foundation described by phase `sase-8m.2`: extract Config Center's proven scope/preview/apply
transaction behavior, model sparse typed edits to schema-defined objects, generalize the property picker, and build an
AXE lumberjack/chop editor surface that phase 3 can open with the phase-1 inventory and mutation APIs. Preserve Config
Center's current scalar/list/map behavior and visual snapshots, keep all planning/apply/YAML work off Textual's event
loop, and cancel or ignore stale work when a modal unmounts.

This tale intentionally does not wire AXE-tab keybindings or selection restoration, perform daemon restarts, discover
scripts, or implement the Rust AXE inventory/mutation backend; those belong to the sibling epic phases. The new AXE
surface will accept immutable editor seed data and injected plan/apply callbacks so those phases can integrate without
duplicating UI transaction logic.

## Implementation

1. Extract the config transaction lifecycle and preview rendering from `ConfigEditModal` into reusable modules under
   `src/sase/ace/tui/modals/`.
   - Define typed immutable transaction metadata/request/preview records and a controller/mixin contract whose caller
     supplies the mutation-request builder plus planning/apply callbacks.
   - Move writable-scope cycling and picking, new-overlay creation, edit/preview/back transitions, diff scrolling,
     validation gating, busy/error status, worker result handling, chezmoi propagation, and conflict recovery onto the
     shared path.
   - Run plan/apply callbacks with thread workers, cancel outstanding workers on unmount, ignore late worker events, and
     re-check mounted/current transaction identity before updating UI state.
   - Keep a reusable pure Rich preview renderer for target path, effective before/after values, diagnostics, warnings,
     and unified diff. Represent a stale-write conflict distinctly, leave the draft intact, and expose a reload/re-plan
     route through an injected callback rather than overwriting external changes.
   - Adapt `ConfigEditModal` to this controller while retaining its public import path, DOM IDs, shortcuts, typed
     scalar/list/map editors, and existing snapshot output.

2. Add a pure reusable schema-object form model.
   - Resolve local schema references and object properties in deterministic schema order, with required properties
     before optional properties and caller-selected Basics/Advanced group metadata.
   - Track each field's effective value, target-layer contribution, provenance, touched state, explicit inherit/reset
     state, draft value, schema description, enum/type information, and numeric/string constraints without copying
     untouched effective defaults into the outgoing patch.
   - Reuse Config Center's editor-kind selection, boolean/enum/scalar/numeric/string-list/raw-YAML parsing and Vim
     editors. Validate patterns and bounded live parsing, and provide raw YAML for every compound or ambiguous schema
     shape.
   - Convert the model purely into exact-segment per-field set/unset operations. Untouched fields emit no operation;
     reset fields emit unset; edited fields emit typed set values. Surface required-field and parse diagnostics without
     file or UI access.

3. Generalize the frontmatter add-property picker.
   - Introduce a reusable property-picker record/modal with configurable title/guidance, deterministic unique
     accelerators, keyboard navigation, direct accelerator selection, mouse row selection, detail text, and graceful
     empty state.
   - Retain `AddableProperty` and `AddPropertyModal` as thin frontmatter-compatible wrappers/aliases so existing panel
     imports, layout, behavior, and snapshots stay stable.
   - Use the generic picker from the AXE form for optional/advanced properties; compound properties always enter the
     raw-YAML editor.

4. Build the reusable AXE entry editor modal on those components.
   - Accept immutable lumberjack/chop identity, effective/raw values, per-field provenance, writable scopes, schema,
     generated-instance warning, running-state metadata, and injected transaction callbacks.
   - Render an immutable identity/status header, clickable scope rail, Basics/Advanced property list with source badges,
     focused bool/enum/scalar/YAML editor, inline validation/error text, and a persistent Vim-aware key-hint strip.
     Provide the documented lumberjack and chop friendly-field ordering while deriving descriptions, constraints, and
     compound editors from the bundled schema.
   - Preserve drafts when moving to preview and back. Render target file, effective before/after, warnings, validation,
     and diff through the shared renderer. When AXE is marked running, make the primary result request save-and-restart
     and retain an explicit save-only result; when stopped, return save-only. The caller performs actual restart/runtime
     reconciliation in phase 3.
   - Add responsive TCSS for a two-column normal layout and a single-column constrained-width layout using AXE's
     gold/copper palette, without changing Config Center's selectors or appearance.

5. Add focused regression coverage.
   - Pure tests cover schema ordering/dereferencing, editor selection and typed parsing, provenance/effective values,
     untouched sparse patches, set/reset transitions, required/constraint diagnostics, exact dotted key segments, and
     compound raw-YAML operations.
   - Picker widget tests cover deterministic accelerators, keyboard/direct-key/mouse selection, guidance, and the thin
     frontmatter wrapper.
   - Transaction and Config Center widget tests cover scope/overlay selection, plan/apply worker success and failure,
     validation gating, preview/back draft retention, chezmoi behavior, typed conflict recovery, late-result rejection,
     and worker cancellation on unmount while retaining the existing Config Center regression/snapshot suite.
   - AXE editor widget tests cover lumberjack and chop field groups, keyboard/mouse navigation, optional-property
     picking, validation/reset semantics, source badges, generated-instance warning, running versus stopped commit
     choices, preview/back retention, conflict recovery, and narrow-layout composition.

## Validation

1. Run targeted pure and Textual tests for the schema form, property picker, transaction controller, migrated Config
   Center modal, and AXE entry editor while iterating.
2. Run the existing Config Center visual snapshot test module without accepting new goldens; inspect failures as
   regressions because this phase promises unchanged Config Center visuals.
3. Run `just install`, then the repository-mandated `just check` after all source/test changes.
4. Re-run the focused new tests after `just check` to confirm formatting or generated setup did not mask worker,
   cancellation, or constrained-layout failures.

## Completion

Close only `sase-8m.2` after the source, focused tests, existing Config Center visual regression, and `just check` all
pass. Do not close parent epic `sase-8m` and do not create additional beads.
