---
tier: tale
title: Separate Agents-tab left navigation from collapse actions
goal: 'Lowercase h navigates outward through the Agents-tab hierarchy while uppercase
  H owns contextual collapsing, without regressing tribe-panel controls, grouping
  folds, Tools detail, navigation bookkeeping, or other tabs.

  '
create_time: 2026-07-21 14:03:49
status: done
---

- **PROMPT:** [202607/prompts/agents_h_left_navigation.md](prompts/agents_h_left_navigation.md)

# Plan: Make Agents-tab h navigate left and H collapse

## Context and interaction contract

The Agents tab currently splits related behavior across the two configurable actions whose defaults are `h`
(`hooks_or_collapse`) and `H` (`hooks_or_collapse_all`). Lowercase `h` collapses workflow/family/clan folds before
selecting a tribe panel, while uppercase `H` performs the recently added family/clan parent jump before falling back to
panel isolation or grouping-banner folding. That makes the direction key change structure and makes the collapse key
move focus.

Keep the existing action identifiers and configurable keymap fields so user overrides and the meanings on ChangeSpecs
and Axe remain compatible, but change their Agents-tab dispatch to this contract:

| Current Agents-tab focus                                | `h` (`hooks_or_collapse`)                                                              | `H` (`hooks_or_collapse_all`)                                                                                    |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Family member                                           | Jump to its immediate family without changing folds                                    | Collapse the selected/containing family fold; on later presses continue outward                                  |
| Family inside a clan                                    | Jump to its clan without changing folds                                                | Collapse the family first when it owns an open fold, then its containing clan                                    |
| Agent directly inside a clan                            | Jump to its clan without changing folds                                                | Collapse the selected agent/workflow fold first when applicable, then the containing clan                        |
| Top-level agent, family, or clan in a split tribe panel | Select that whole tribe panel without changing tree or grouping folds                  | Collapse the selected structural fold when one is open; otherwise collapse its deepest enclosing grouping banner |
| Grouping-strategy banner                                | Select the containing tribe panel                                                      | Collapse that banner, or its parent banner when the selected deep banner is already collapsed                    |
| Selected expanded tribe panel                           | Collapse that panel, preserving the existing explicit exception to navigation-only `h` | Keep the existing only-panel / restore-panels toggle                                                             |
| Selected collapsed tribe panel                          | Retain the existing already-collapsed handling                                         | Keep the existing only-panel / restore-panels toggle                                                             |
| Tools detail visible                                    | Navigate using the underlying row/banner context; do not reduce Tools detail           | Keep the existing compact-to-minimum Tools behavior and do not fall through to tree/group collapsing             |
| No valid or selectable parent                           | No-op rather than skipping malformed ancestry                                          | Use the next valid collapse target, ending with the enclosing grouping banner                                    |

“Immediate parent” is based on the rendered structural tree, not the active grouping strategy. It includes member →
sequential family, family → clan, and direct clan member → clan. A row is top-level only when it has no structural
family/clan parent; stale, duplicate, self-referential, hidden-script, or otherwise malformed edges must not be treated
as top-level shortcuts to the panel. Whole-panel selection remains available only where the split-panel model already
supports it (for example, it stays a harmless no-op in a single-panel or merged-panel layout).

The uppercase collapse priority on Agents is therefore:

1. Compact active Tools detail, as today.
2. Isolate or restore an explicitly selected tribe panel, as today.
3. Collapse the selected or nearest containing workflow/family/clan structural fold using the existing fold-level and
   re-anchoring rules.
4. Collapse the deepest enclosing grouping-strategy fold, with the existing deep-banner → parent-banner progression.

This preserves `H`'s non-Agents all-fold behavior and gives grouping banners the requested lowest priority. Lowercase
`h` continues its existing ChangeSpecs/Axe routing; the navigation-only rule is scoped to Agents.

## Navigation and collapse design

Refactor the Agents helpers so capability resolution and state mutation are separate and each action has one clear job:

- Generalize the recently added validated parent resolver to describe every supported structural left-navigation edge,
  including a direct agent → clan edge. Add an explicit top-level-to-panel result only after proving the selected row
  has no structural parent, and resolve a focused grouping banner to its containing panel without fabricating an agent
  target.
- Route lowercase `h` around the Agents Tools-detail collapse handler. For a row or banner, perform the resolved parent
  move; for an already selected panel, retain the existing panel-collapse path. Parent row moves reuse the established
  jump anchor, manual-unread departure, attempt reset, panel-selection memory, and unread acknowledgement behavior.
  Moving to a whole panel should likewise retain the row/banner as that panel's remembered selection, save a reversible
  jump origin, clear the pinned attempt, and refresh only panel focus/detail state.
- Extract the current Agents structural-collapse portion of lowercase `h` into a helper that reports whether it changed
  a workflow/family/clan fold. Preserve its one-level workflow/family transitions, binary clan collapse, focus
  re-anchoring to visible fold owners, and selective refilter. Invoke it from uppercase `H` after Tools and
  selected-panel handling; call grouping collapse only when no structural fold can be collapsed.
- Remove the parent-jump mutation from uppercase `H`. Keep the existing grouping-fold persistence and panel-isolation
  persistence untouched, and do not rebuild the list for navigation-only `h` moves.

All ancestry and fold decisions must use the already-loaded Agents projection and canonical tree keys/lookups. The
keypress path remains prompt-free and performs no disk, subprocess, network, or Rust-core work. This is presentation and
selection behavior, so no `sase-core` API change is needed.

## Discoverability and compatibility

Drive footer affordances from the same non-mutating capability resolvers used by dispatch so labels cannot disagree with
behavior:

- Show lowercase `h` as `parent family`, `parent clan`, or `parent tribe` as appropriate, including while Tools detail
  is visible. On a selected expanded panel retain `h: collapse panel`.
- Show uppercase `H` for its effective collapse target (Tools compaction, selected/containing family or clan/workflow,
  or grouping fold), while preserving `only panel` / `restore panels` precedence for selected panels.
- Remove the lowercase `h: less detail` Agents affordance because Tools reduction now belongs to uppercase `H`.

Update the fallback binding descriptions, command catalog metadata, Agents help section, keymap schema/display fixtures,
and the explanatory default-config comment. Do not rename the actions or default keys, and do not alter their behavior
or documentation on other tabs beyond wording needed to describe the contextual Agents behavior accurately.

## Verification

Expand the focused folding/navigation tests into explicit ladders and precedence checks:

- `h` walks member → family → clan → tribe panel, direct clan member → clan → panel, and standalone/top-level
  agent/family/clan → panel without mutating tree folds, grouping folds, or panel fold state before panel selection.
- Parent navigation remains grouping-mode independent, accepts supported workflow-agent family rows, rejects hidden
  script rows and ambiguous/malformed ancestry, and does not skip invalid parents to select a panel.
- `h` on a grouping banner selects its panel; `h` on a selected expanded panel still collapses it; single/merged panel
  layouts remain safe no-ops; navigation updates history, remembered selection, unread state, attempt state, and focused
  detail consistently.
- With Tools visible, `h` navigates and leaves detail level unchanged, while `H` compacts Tools and stops. Other tabs
  retain their existing lowercase/uppercase fold routing.
- `H` collapses an open selected workflow/family before a containing clan, a clan before a grouping banner, and a
  grouping banner only after structural targets are exhausted. It preserves one-level/binary fold semantics,
  re-anchoring, persistence, and per-panel grouping-registry isolation.
- A selected tribe panel still gives `h` panel collapse and `H` only/restore behavior, including the armed restore flow.
- Footer tests cover family, clan, tribe, grouping, selected-panel, and Tools precedence; help, command-catalog, runtime
  binding, fallback binding, and default-keymap expectations match the new contract.

Run focused Agents navigation/folding, panel-collapse/isolation, footer, help, binding, and command-catalog tests first.
Run the affected Agents visual snapshot suite and update goldens only for intentional conditional-footer changes. After
`just install`, finish with the repository-required `just check` gate.
