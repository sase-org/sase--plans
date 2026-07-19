---
tier: tale
title: Restore contextual ACE query shortcuts
goal: 'Bare slash opens query or filter editing everywhere it did before the Agents
  metadata-search work, while the Agents tab alone keeps leader-slash for its structured
  agent query and reserves bare slash for inline metadata search.

  '
create_time: 2026-07-19 13:02:35
status: wip
prompt: 202607/prompts/restore_contextual_query_shortcuts.md
---

# Plan: Restore contextual ACE query shortcuts

## Context and intended behavior

The `sase-76` keymap work moved `edit_query` from the app keymap to leader mode globally so that the Agents tab could
use bare `/` and `?` for Vim-style metadata search. Only Agents needed that query remap. Restore the narrower contract:

| Context | Bare `/`                                    | `,/`                                    |
| ------- | ------------------------------------------- | --------------------------------------- |
| PRs     | Open the structured ChangeSpec query editor | No query action                         |
| Commits | Open the inline commit filter               | No query action                         |
| Plans   | Open the inline plan filter                 | No query action                         |
| Bugs    | Remain inert                                | No query action                         |
| Agents  | Start forward inline metadata search        | Open the structured Agents query editor |
| Axe     | Open the current Axe query editor           | No query action                         |

Keep the existing local `f` aliases for the Commits and Plans filter bars. Keep reverse Agents metadata search on bare
`?` and the current leader-mode `,?` Help binding on all tabs; changing Help is outside this correction. This is
presentation and TUI key-routing work and does not require a Rust-core API change.

## Contextual keymap model and dispatch

Restore `edit_query` as a configurable app-level action with default `slash` in the typed registry, bundled
configuration, fallback bindings, app command metadata, and generated Textual bindings. Stop treating only
`ace.keymaps.app.edit_query` as retired so existing or newly restored app-level overrides are honored again; continue
treating the retired app-level `show_help` override as retired. Preserve `leader_mode.keys.edit_query` as an independent
configurable binding for the Agents-only chord.

Allow the two query/search actions to share the default physical slash because their applicability is disjoint. Gate
`search_forward` to Agents and gate the app-level `edit_query` action away from Agents while retaining its PRs, Commits,
Plans, and Axe routes and leaving Bugs inert. Scope leader `edit_query` dispatch, repeat-last behavior, leader footers,
and command-palette metadata to Agents so the chord is neither advertised nor executable elsewhere. Preserve the
existing `action_edit_query` destinations rather than duplicating query/filter-opening logic.

Ensure the command catalog exposes the app command in PRs, Commits, Plans, and Axe, exposes the leader command only in
Agents, and exposes neither query command in Bugs. Configuration validation should continue to support remapped keys and
should represent the intentional contextual default overlap without weakening unrelated duplicate-key checks.

## User-visible key hints and documentation

Render the configured app-level query key on every non-Agents surface: PR and Axe Help sections, ChangeSpec onboarding
and empty/no-match states, normal PR footers, Commits and Plans hint bars, and their Help rows. Continue rendering the
configured leader chord on Agents onboarding, Help, quick-start, and leader-mode footer surfaces. Where a shared
renderer serves both Agents and PRs, choose the display from its explicit tab context instead of assuming one global
query shortcut.

Update the ACE and configuration documentation to describe the split ownership and override locations. Examples and
tables should say `/` for PRs, Commits, Plans, Axe, and the general query editor; only the Agents structured-query entry
should say `,/`. Retain documentation of bare `/` and `?` as Agents metadata search and `,?` as Help.

## Tests and verification

Update registry, loader, binding, command-catalog, leader-dispatch, footer, onboarding, Help, and quick-start tests to
assert the two independently configurable query bindings and their contextual displays. Add or revise an end-to-end
context matrix proving that bare `/` opens the expected editor/filter on PRs, Commits, Plans, and Axe; remains inert on
Bugs; and starts metadata search on Agents, while `,/` opens the structured query only on Agents. Cover custom app and
leader remaps separately, including Agents leader repeat behavior and non-Agents rejection of the leader query command.

Convert Commits and Plans interaction tests and visual-snapshot drivers from the leader chord to bare slash. Regenerate
only snapshots whose visible query hints intentionally change, inspect the generated diffs, and rerun the visual suite
to confirm exact stability. Finish with the repository-required validation:

1. Run `just install` before repository checks.
2. Run focused keymap, command availability, query/filter, rendering, and contextual end-to-end tests while iterating.
3. Run `just test-visual`; if expected PNGs fail, inspect the artifacts, accept only the intended key-hint changes, and
   rerun `just test-visual` cleanly.
4. Run `just check` and resolve every failure.
