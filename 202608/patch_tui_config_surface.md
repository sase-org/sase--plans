---
tier: tale
title: Rename the ACE TUI and configuration surface to Patch and stitch
goal:
  ACE and its configuration present canonical Patch and stitch terminology while legacy
  config, saved state, tabs, and completion payloads continue to work unchanged.
size: large
proposed_by: bbugyi200.athena.sase-hn.4
bead: sase-hn.4
create_time: 2026-08-08 18:41:55
status: wip
---

- **PARENT:**
  [202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_and_stitch_terminology.md)
- **BEAD:**
  [sase-hn.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.4.md)

# Rename the ACE TUI and configuration surface to Patch and stitch

## Goal

Make Patch and stitch the canonical terminology throughout ACE, its TUI-facing
configuration, persisted UI state, and completion presentation, while preserving the
behavior of existing installations and leaving genuine VCS commit surfaces unchanged.

## Context and invariants

- The Python domain/storage and workflow phases already provide canonical `Patch` and
  `Stitch` APIs plus legacy adapters. New ACE code must import and name the canonical
  APIs; compatibility spelling may remain only at explicit normalization boundaries.
- The ACE Artifacts `Commits` pane, commit SHAs, incoming update commits, agent commit
  metadata, commit statistics, and commit-view modals represent real repository commits
  and must retain commit terminology.
- The Patch detail section currently presented as `COMMITS:` represents ordered stitch
  records. Rename its widget/builder/fold/hint presentation to stitches and render
  `STITCHES:` while continuing to display legacy ProjectSpec data parsed through the
  canonical domain model.
- TUI loading, refresh coalescing, selection restoration, fold/navigation order, focus,
  and cached render paths are behaviorally frozen. This phase renames those paths but
  does not add synchronous I/O or change their scheduling.
- Existing user keymaps, saved grouping state, CLI `--tab changespecs`, repro captures,
  completion catalogs, and axe guard configuration must continue to load. Canonical
  identifiers win when canonical and legacy values are both supplied.

## Implementation

1. **Canonicalize ACE source structure and domain names.**
   - Rename the ChangeSpec-specific action package, grouping/tree/index/load models,
     widgets, modal, clipboard implementation, layout/status helpers, fixtures, and
     corresponding tests to Patch names.
   - Replace TUI-local `ChangeSpec*`, `changespec*`, and `change_spec*` symbols, fields,
     trace names, row kinds, comments, and type aliases with `Patch*` / `patch*`, using
     `sase.ace.patch.Patch` and `Stitch` as the source of truth.
   - Rename the Patch-detail commit-entry builder, fold state, hint mappings, and copy
     targets to stitch vocabulary. Do not rename code belonging to the Artifacts Commits
     pane or other genuine VCS commit concepts.

2. **Update every visible ACE surface.**
   - Present “Patch”/“Patches” and “stitch”/“stitches” in list/detail panels, onboarding
     and empty states, help/guide sections, command palette metadata, copy palettes,
     notifications, errors, banners, tooltips, accessibility text, statistics
     drilldowns, project/agent grouping labels, and completion rows.
   - Keep the public Artifacts tab and its PRs subtab layout/order intact, but make
     `artifacts` the canonical internal top-level tab identifier. Normalize the legacy
     `changespecs` identifier at CLI, constructor/repro, and saved-input boundaries.
   - Rename CSS classes/IDs and visual fixture/test names where they identify Patches,
     updating selectors and tests atomically so focus and layout remain unchanged.

3. **Introduce canonical config and saved-state identifiers with legacy adapters.**
   - Rename app keymap actions such as `next_changespec`, `start_agent_from_changespec`,
     and `jump_to_agent_changespec` to Patch forms in `AppKeymaps`, bindings, metadata,
     command IDs, and `src/sase/default_config.yml`. Normalize legacy overrides before
     unknown-key validation, with canonical values taking precedence.
   - Move the copy-mode Patch group to a canonical Patch key and normalize the legacy
     `changespecs` group. Update the JSON schema and config tests together.
   - Persist the Patch grouping mode to a canonical filename; read it first, fall back
     to the legacy `changespec_grouping_mode.txt`, and write only the canonical file.
     Preserve saved query selection contents and grouping/fold semantics.
   - Make `patch` the canonical axe inhibit-provider key in defaults/schema/runtime.
     Accept keyed and tagged legacy `changespec` guards and normalize them to one
     canonical provider before evaluation. Update the Rust core validation/wire engine,
     because it is the authority for this shared config behavior.

4. **Migrate completion kinds without breaking mixed versions.**
   - Make the in-process project/ref completion entry kind canonical `patch` and update
     ACE badges, widths, filtering, and tests.
   - Bump the materialized completion catalog schema and dual-emit a canonical entry
     discriminator plus the stable legacy `kind: changespec` field required by older
     Rust LSP consumers. Update current Rust core/LSP readers to prefer the canonical
     discriminator and fall back to the legacy field, then emit canonical `patch` in
     current completion results.
   - Add old-catalog/new-reader and new-catalog/legacy-field compatibility coverage;
     insertion text, ordering, caching signatures, and keystroke-path behavior remain
     byte-for-byte unchanged.

5. **Audit and verify terminology boundaries.**
   - Search ACE source/config/tests for separator and case variants. Classify retained
     occurrences as explicit legacy adapters/fixtures or genuine VCS commit concepts; do
     not weaken the later epic-wide compatibility audit with a broad allowlist.
   - Update Textual assertions and PNG goldens only for intentional label/identifier
     changes. Inspect generated actual/expected/diff/source artifacts before accepting
     any visual snapshot.

## Verification

1. Run `just install` before repository checks.
2. Run targeted canonical/legacy tests for ACE patch models, grouping/navigation,
   detail/list/onboarding, copy/help/command palette, tab normalization, keymap/default
   config/schema loading, saved grouping migration, axe guard providers, and VCS
   project/ref completions.
3. In `sase-core`, run formatting plus the focused axe-chop, editor completion, and
   xprompt-LSP tests; run the repository's required Rust check lane if the focused tests
   expose broader changes.
4. In the main SASE repository, run `just check`. If scoped selection escalates or the
   broad config/completion changes require it, run `just check-full`.
5. Run the dedicated TUI performance/grouping tests to confirm navigation behavior and
   run `just test-visual`. Inspect visual diff artifacts, accept only terminology-driven
   snapshots, and rerun the visual suite to green.
6. Re-run terminology searches and `git diff --check` in every touched repository.
   Confirm no unexplained ChangeSpec/CommitEntry/`COMMITS:` presentation remains in the
   canonical ACE surface, legacy config/state/catalog inputs still pass, and all real
   VCS commit surfaces still say commit.

## Completion

Record any genuinely separate discovered work only as a `PROPOSED FOLLOW-UP:` note on
`sase-hn.4`. Close `sase-hn.4` with a verification note after all required checks pass;
do not close the parent epic.
