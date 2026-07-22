---
tier: epic
title: AXE lumberjack and chop editor
goal: 'Users can add lumberjacks and chops and safely edit selected AXE entries from
  the sase ace AXE tab through a responsive, validated, source-preserving, and polished
  TUI workflow.

  '
phases:
- id: config-foundation
  title: Exact AXE config composition and mutation planning
  depends_on: []
  size: medium
  description: '''Exact AXE config composition and mutation planning'' section: centralize
    layered AXE composition and expose conflict-safe, exact-key entry mutation plans.'
- id: shared-editor
  title: Shared config transaction and schema-form components
  depends_on: []
  size: medium
  description: '''Shared config transaction and schema-form components'' section:
    extract the proven Config Center transaction UX and build reusable typed object-form
    pieces for AXE entries.'
- id: axe-integration
  title: AXE add and edit workflows
  depends_on:
  - config-foundation
  - shared-editor
  size: medium
  description: '''AXE add and edit workflows'' section: wire contextual add/edit actions,
    script discovery, selection restoration, and runtime reconciliation into the AXE
    tab.'
- id: polish-verification
  title: Visual polish and end-to-end verification
  depends_on:
  - axe-integration
  size: small
  description: '''Visual polish and end-to-end verification'' section: finish responsive
    styling, snapshots, documentation, performance checks, and layered-config lifecycle
    exercises.'
create_time: 2026-07-22 11:37:43
status: done
bead_id: sase-8m
---

# Plan: AXE lumberjack and chop editor

## Outcome and tier decision

This is an epic because it crosses the shared Rust configuration backend, the Python config facade and source-preserving
writer, reusable Textual modal infrastructure, and the AXE tab's command, cache, daemon, and visual systems. The backend
contract and reusable editor components can be built independently; the AXE integration depends on both, and a final
focused phase exercises the joined behavior.

The finished workflow should feel native to ACE:

- `a` opens a compact **Add to AXE** chooser. From a lumberjack or one of its chops, **New chop under <lumberjack>** is
  the primary choice; **New lumberjack** is always available. From an empty AXE tab or a background command, new
  lumberjack is primary and choosing new chop first opens a lumberjack picker.
- `e` edits the selected lumberjack or chop configuration. It is unavailable for background-command rows. A selected
  generated `for_each` instance opens its base chop and clearly warns that the edit affects every generated instance.
- Preserve the existing ability to open recorded chop output, but move that secondary operation to `E`, whose
  footer/help/palette label is **Edit chop output**. It remains conditional on a chop with recorded output.
- `a` is an AXE-wide command documented in help and the command palette. Since it is always meaningful on the AXE tab,
  it does not occupy the conditional footer. The footer shows `e` only on editable AXE rows and `E` only when chop
  output exists, following the footer convention in `src/sase/ace/AGENTS.md`.
- Successful writes return to the AXE tree with the new or edited stable identity selected and its parent expanded. When
  AXE is running, preview makes **Save & restart AXE** the explicit primary action and offers **Save only**; when AXE is
  stopped, saving never starts it. A failed restart must report that the config write succeeded while the runtime did
  not reconcile.

The editor uses AXE's gold/copper visual language, Config Center's scope and diff-preview trust model, Vim-capable text
widgets, inline validation, and a responsive two-column layout that collapses cleanly on narrow terminals. It offers
friendly fields first and retains complete schema coverage:

- Lumberjack: immutable identity while editing, `interval`, optional `chop_timeout`, and advanced `env` YAML.
- Chop: immutable identity while editing, `script`, `description`, `enabled`, optional `run_every` and `timeout`, plus
  advanced `env`, `inhibit_if`, `trigger`, `once_per`, `for_each`, and `vars` editors derived from the bundled schema.
- Adding a chop starts with discovered installed scripts from the existing chop inventory, including source/resolution
  status, and retains a **Custom script** path. A discovered script pre-fills the exact executable and a sensible
  identity; all fields remain editable before preview.

Track which fields the user actually changes. Build a sparse patch over the chosen target layer's raw contribution, not
a copy of the effective entry, so editing one inherited field does not freeze unrelated lower-layer defaults in an
overlay. Each field supports **inherit/reset** to remove that target-layer override. Name changes, deletion, and
reordering are intentionally out of scope: they require separate state/history migration and confirmation designs.

## Exact AXE config composition and mutation planning

Implement the frontend-neutral configuration behavior in the linked `sase-core` repository and expose it through the
existing Rust binding and thin Python facades.

1. Move the keyed AXE layer composition rules currently coordinated by `sase.axe._config_layers` into a Rust core API.
   It must normalize legacy list-form chops to stable keyed identities for composition, honor each layer's
   concatenate/replace policy, preserve per-field provenance, retain duplicate and migration diagnostics, and produce
   the same effective config consumed by the runtime. Keep compatibility wrappers only where existing Python patch/test
   surfaces require them, and route runtime loading through the shared core result so the editor, CLI, and daemon cannot
   drift.
2. Add a frontend-neutral AXE entry inventory/mutation contract for a lumberjack or base chop. Identify dynamic names as
   exact path segments, never by splitting a dotted display string. The inventory must expose the effective entity, each
   writable layer's raw contribution/representation, field provenance, enabled state, and base/expanded-instance
   relationship.
3. Plan an entry edit from a target layer plus a set of per-field set/unset operations. Apply the operations to that
   layer's raw sparse contribution, promote a target-layer legacy chop list to an equivalent keyed map only when
   required, then return the established config write plan, effective before/after preview, target-file diff inputs,
   generic schema diagnostics, and strict AXE diagnostics. Adding a chop over the built-in list-form defaults must
   preserve every lower-layer chop in the effective preview.
4. Extend the generic config edit wire/API to carry exact key-path segments while retaining dotted scalar-path
   compatibility for existing callers. Cover names containing dots, dashes, and other currently valid mapping-key
   characters. Do not silently impose a new global name grammar from the TUI.
5. Harden the shared Python apply step used by Config Center and AXE: compare the target bytes/token used to create the
   preview immediately before the write, refuse a stale plan with a typed conflict result, and replace the file
   atomically while preserving applicable mode bits. Clear config caches only after a successful replacement. A conflict
   leaves the user's draft in the modal and offers reload/re-plan rather than overwriting an external edit.
6. Add Rust and Python tests for exact dynamic keys, sparse inherited edits, field reset, default-list plus overlay-map
   composition, replace layers, in-layer list-to-map promotion, duplicate/provenance diagnostics, strict validation,
   comment-preserving output, atomic failure behavior, and stale preview rejection. Prove parity between runtime-loaded
   and previewed AXE config for the same layer stack.

## Shared config transaction and schema-form components

Refactor the strongest existing Config Center behavior into focused reusable pieces rather than cloning
`ConfigEditModal`.

1. Extract a config transaction controller/mixin and preview renderer that own writable-scope selection, overlay
   creation, asynchronous plan/apply workers, validation rendering, diff scrolling, chezmoi propagation, busy/error
   states, and conflict recovery. The caller supplies an immutable mutation request builder and display metadata.
   Migrate `ConfigEditModal` onto this shared path without changing its current scalar/list/map editing behavior or
   snapshots.
2. Keep all config inventory building, script discovery, YAML parsing beyond the bounded live-edit limit, planning, file
   access, chezmoi work, and git inspection off Textual's event loop. Timer/call-after-refresh callbacks may only
   schedule pump-free work. Cancel workers/tasks on unmount and re-read mounted/current state after awaits.
3. Build a reusable schema-object form model: ordered required/optional properties, touched/reset state, effective value
   and provenance, schema description/constraints, typed parsing, and a pure conversion to field operations. Reuse the
   existing Config Center schema dereferencing, bool/enum/numeric/string/YAML helpers and Vim text areas.
4. Generalize the polished frontmatter add-property picker into a reusable property picker (with the frontmatter modal
   retained as a thin wrapper), and use it for optional/advanced AXE fields. Provide deterministic accelerators,
   keyboard and mouse support, live guidance, and a raw-YAML escape hatch for every compound schema shape.
5. Create an AXE entry editor surface on top of those components with an edit stage and a validated diff-preview stage.
   The edit stage has a compact identity/status header, scope rail, Basics/Advanced property list, focused typed editor,
   inline source badges, and a persistent key-hint strip. Preview names the target file, shows effective before/after
   and warnings, and makes the save/restart consequence unambiguous.
6. Add pure form-model and Textual widget tests for keyboard/mouse navigation, validation and reset semantics,
   scope/overlay changes, preview/back draft retention, conflict recovery, worker cancellation, narrow layout, and
   regression coverage for the migrated Config Center modal.

## AXE add and edit workflows

Join the backend and reusable editor into the running AXE tab without adding I/O to navigation or render paths.

1. Add a configurable `add_axe_item` app keymap with default `a`; update `src/sase/default_config.yml`,
   keymap/schema/types, runtime bindings, command metadata, and availability. Reinterpret the existing contextual
   `edit_spec` action as AXE config edit on lumberjack/chop rows, and extend the `E` action to open chop output on
   eligible AXE rows. Update help, command palette labels/aliases, conditional footer behavior, and their guard tests.
2. Add an AXE config-action mixin that snapshots only stable selected identities on the keystroke path, opens the add
   chooser/parent picker or editor, and loads inventory plus discovered scripts in a thread worker. Duplicate names are
   rejected live against base chop identities; missing custom executables are a visible warning in preview, not a reason
   to lose a valid draft.
3. Enrich cached chop metadata enough to render and edit all configured base chops, including disabled chops, without
   disk reads per row. Disabled rows remain visible with a quiet disabled chip so `e` can re-enable them, and manual run
   availability excludes them. Preserve recorded-run rendering and clearly map expanded runtime instances back to their
   base edit identity.
4. On apply, reuse the shared config commit-offer helpers and the existing tracked commit/pull/push task. Then consume a
   pending stable AXE item key in the coalesced full refresh, expanding the parent and restoring the intended selection
   only if the current tab/state still permits it. Never let a late refresh move the cursor the user has since moved
   elsewhere.
5. If restart was requested and AXE is still running when the write completes, call the established verified daemon
   restart flow with an AXE-config-edit source label and existing transition/status UI. If it is no longer running,
   honor the stopped state. Refresh configuration immediately even for **Save only**, and clearly notify that the
   running daemon retains its prior configuration until restarted.
6. Improve the zero-lumberjack AXE surface with an `a` keycap call to action, while keeping background commands usable.
   Update tree/dashboard/header copy so add/edit/restart state reads clearly without crowding the operational run view.
7. Add action, availability, collector/cache, selection, daemon-result, commit follow-up, and AcePage integration tests.
   Assert that add/edit opening and j/k rendering perform no synchronous config, inventory, subprocess, or git reads.

## Visual polish and end-to-end verification

Finish the feature as a coherent product surface rather than a collection of working controls.

1. Add deterministic PNG snapshots for the Add to AXE chooser, discovered script selection, new lumberjack form, chop
   Basics and Advanced states, generated-instance warning, validation failure, diff preview, disabled chop row, empty
   AXE call to action, and a constrained-width layout. Reuse the pinned AXE and Config Center visual fixtures and accept
   new goldens only after inspecting actual/expected/diff artifacts.
2. Update `docs/ace.md` and the `?` help/guide content with the `a`, `e`, and `E` behavior, source scopes,
   preview/restart choices, generated-instance semantics, and the raw-YAML escape hatch. Keep help box widths and
   default keymap documentation synchronized.
3. Exercise these end-to-end scenarios: default list plus overlay-map add; inherited one-field edit without copied
   defaults; edit of a user list-form chop; exact names containing dots/dashes; add from empty AXE; add from a
   background-command row; disable then re-enable; edit a `for_each` instance; missing script warning; concurrent
   external config change; chezmoi write and apply; optional commit/push; AXE stopped; AXE running with restart;
   save-only; and restart failure after a successful write.
4. In `sase-core`, run formatting, clippy/lints, and the full Rust workspace tests. In `sase`, run `just install` before
   `just check`, run the dedicated visual suite, and run the targeted AXE/config widget and integration tests. Use the
   existing TUI performance harness to confirm cached AXE j/k p95 remains below 16 ms and inspect the stall log for
   modal open/plan/apply and post-write refresh paths.

## Acceptance criteria

- A first-time user can add a lumberjack and an installed or custom chop from the AXE tab without manually editing YAML.
- `e` always edits the selected AXE configuration identity; advanced schema features remain reachable, and `E` retains
  direct access to recorded output.
- Every write is scope-aware, sparse, previewed, strictly validated, source-preserving, conflict-detecting, and atomic;
  existing Config Center behavior continues to pass.
- Layered preview equals runtime composition across legacy and keyed chop forms, including provenance and
  replace/concatenate semantics.
- Disabled chops and generated instances have honest, recoverable edit behavior.
- Config persistence, inventory/script discovery, daemon reconciliation, git follow-up, and refreshes never block
  Textual's event loop or steal a newer selection.
- The complete workflow is documented, visually snapshot-tested at normal and narrow sizes, and passes both
  repositories' required checks.
