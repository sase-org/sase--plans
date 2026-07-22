---
tier: tale
title: Complete Agents parent navigation for every house member
goal: 'Every rendered member of an agent house, including agent, Bash, Python, parallel,
  and compatibility workflow steps, can navigate to its validated immediate owning
  row with lowercase h, while lowercase l remains truthful structural expansion, uppercase
  H keeps collapse precedence, and malformed ancestry remains safely inert.

  '
create_time: 2026-07-22 06:31:03
status: wip
---

- **PROMPT:** [202607/prompts/agents_all_house_member_navigation.md](prompts/agents_all_house_member_navigation.md)

# Plan: Complete Agents parent navigation for every house member

## Confirmed root cause and key contract

The screenshot `20260722_062150.png` is a real loader shape, not a selection, refresh, or key-dispatch failure. Its
selected `setup` row is a Python pre-prompt step of the `gh` workflow. The sibling `prepare` and `diff` rows are Bash
steps, `checkout` is another Python step, and the concrete `main` row is an agent step. The loader gives every one of
these rows the owning workflow's timestamp as both `raw_suffix` and `parent_timestamp`; pre-prompt script rows are also
marked hidden. Their common parent is therefore already represented by `agent_parent_fold_key()`,
`tree_parent_lookup()`, and the rendered indentation.

The incomplete behavior was introduced deliberately by the previous navigation fix. After resolving a canonical parent,
`_resolve_agent_left_navigation_target()` rejects every selected `is_hidden_step`, rejects every selected row that is
not `is_agent_entry`, and only accepts a non-clan parent when that parent is a sequential-family container. That admits
the concrete `main` agent step in a plan family but excludes Bash, Python, parallel, and other script/compatibility
steps with the same valid edge. It also excludes agent/script steps owned by a standalone workflow house. The focused
regression `test_h_accepts_only_real_agent_family_children` currently codifies this defect by requiring a Bash child to
remain selected.

The reported lowercase `l` no-op has a separate, correct mechanical explanation. On an Agents row, `l` calls
`_get_workflow_key_for_agent()` and expands the selected structural owner or, for a child, its owning workflow fold. A
hidden/pre-prompt step can only be visible when that owning fold is already `FULLY_EXPANDED`, so another expansion has
no state to advance. A leaf Bash or Python step does not own a fabricated child fold. The repair must make the actual
parent action discoverable and functional on that row without changing `l` into a second parent key or inventing
descendants.

Apply this interaction contract:

| Focus/state                                                                                       | Lowercase `h`                                                                          | Lowercase `l`                                                             | Uppercase `H`                                                                                              |
| ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Agent, Bash, Python, parallel, embedded, or compatibility workflow step                           | Select the validated immediate owning workflow/family row                              | Expand the owning fold by one real level; no-op if already fully expanded | Collapse the owning workflow/family according to existing structural precedence and reanchor when required |
| Family member                                                                                     | Select its family root                                                                 | Expand only a fold actually owned by the member                           | Preserve family-collapse precedence                                                                        |
| Direct clan member or family root in a clan                                                       | Select the clan container                                                              | Expand only a real owned fold                                             | Preserve clan/family/workflow collapse precedence                                                          |
| Top-level standalone house or clan container                                                      | Select its split tribe panel when eligible                                             | Expand its real structural fold                                           | Preserve structural-then-group collapse precedence                                                         |
| Grouping banner or selected split panel                                                           | Preserve the existing tribe selection, panel collapse, and panel enter/expand behavior | Preserve existing banner/panel behavior                                   | Preserve existing group and panel isolation/restoration behavior                                           |
| Merged layout, missing owner, duplicate structural owner, self-edge, cycle, or inconsistent depth | No fabricated parent target                                                            | No fabricated fold                                                        | Preserve the next valid existing collapse behavior                                                         |

Use `workflow` for the contextual parent label when the immediate owner is an ordinary workflow/agent house, while
retaining `family`, `clan`, and `tribe` for those existing container kinds. A plan-family root that also owns workflow
steps remains a `family` target because that is the rendered container the user reaches. This work does not change
configured keys or non-Agents routing.

## Generalize immediate-parent validation around rendered ancestry

Keep `agent_parent_fold_key()` and `tree_parent_lookup()` as the canonical source of the immediate rendered edge. Extend
the left-navigation target kind to cover an ordinary workflow/agent-house owner, and classify the target from the
resolved parent rather than from the selected child's process/tool capability:

- accept any rendered child linkage whose exact canonical parent is a unique non-child structural owner in the loaded
  Agents projection, regardless of `step_type`, `is_hidden_step`, `is_pre_prompt_step`, or `is_agent_entry`;
- classify a sequential-family container as `family`, a synthetic clan container as `clan`, and another valid
  workflow/agent structural owner as `workflow`;
- retain the existing family-root-to-clan, direct-member-to-clan, clan-to-tribe, top-level-to-tribe, and grouping-banner
  paths;
- continue treating child rows that repeat their parent's `raw_suffix` as aliases, not competing owners, while still
  requiring the canonical parent object to occur exactly once and exactly one non-child owner to claim the fold key;
- validate depth and explicit `tree_parent_key` consistently for children inside and outside clans, but allow the
  loader's legacy non-clan fallback depth when only `parent_timestamp` supplies the edge;
- reject missing canonical owners, duplicate root owners, repeated parent objects, self-references, cycles, and rows
  whose declared parent/depth contradicts the loaded projection rather than skipping outward to a clan or tribe.

Prefer a small pure helper for owner validation/classification if it keeps the resolver auditable and lets footer and
action dispatch share one answer. Do not weaken global tree projection, fold filtering, suffix deduplication, panel
inheritance, or `Agent.is_agent_entry`; Bash/Python rows should remain non-agent entries for tools/process actions even
though they participate in presentation-tree navigation.

## Preserve navigation state and fold semantics

Route newly accepted workflow-step targets through the existing `_navigate_agent_left()` mutation path so they receive
the same selection semantics as other members: save a reversible jump anchor, arm unread departure only where the
existing bookkeeping considers the origin eligible, clear attempt/group focus, update `current_idx`, remember the
focused panel selection, acknowledge the destination owner, and refresh detail/footer state through the normal display
path. Navigating from a step to its owner must not mutate structural, grouping, or panel folds and must remain an
in-memory keystroke operation with no disk access, subprocess, prompt, full-list rebuild, or asynchronous work.

Keep `_get_workflow_key_for_agent()` and `l` expansion semantics intact. In particular, an ordinary visible step may use
`l` to advance its owning workflow from `EXPANDED` to `FULLY_EXPANDED` and reveal hidden siblings, but an
already-visible hidden leaf at `FULLY_EXPANDED` stays a no-op. Keep uppercase `H` as the structural mutation key: from
any workflow-step kind it collapses the owning fold, reanchors to the owner under the existing level rules, and runs
before grouping collapse. Tools-detail routing, whole-panel focus, merged layout behavior, numeric member jumps, and
panel isolation are unchanged.

## Make footer, help, and documentation truthful

Allow the conditional footer to render `h: parent workflow` in addition to the existing family/clan/tribe variants. The
screenshot-shaped Python row under a plan-family root should instead render `h: parent family`; an ordinary standalone
workflow step should render `h: parent workflow`. Continue deriving the chip from the same resolver used by the action
so malformed rows advertise nothing. Preserve Tools-detail precedence and the contextual uppercase
`H: collapse workflow/family/clan` chip.

Update the Agents help text and the structural-fold documentation to describe the complete ladder and the post-fix key
split: lowercase `h` navigates to a validated workflow/family/clan/tribe parent (and collapses only a selected panel),
lowercase `l` expands a real fold or enters a panel, and uppercase `H` performs structural/group collapse or panel
isolation. Correct stale statements that still describe lowercase `h` as the structural-collapse key. No default keymap
or schema change is needed.

## Regression coverage and verification

Strengthen the loader-shaped plan-family fixture so it contains concrete agent, Bash, Python, hidden/pre-prompt,
parallel, embedded, and compatibility workflow steps sharing the root suffix exactly as production does. Exercise it in
`BY_STATUS` and prove every step kind resolves to and navigates to the family root without changing tree, group, or
panel folds. Replace the old Bash-no-op expectation with positive, parameterized coverage.

Add an ordinary standalone workflow-house fixture and a clan-nested workflow fixture to prove:

- all workflow-step kinds reach the ordinary owning workflow with target kind `workflow`;
- a second `h` continues from that owner to its clan or eligible tribe rather than skipping a level;
- jump-back/forward anchors and focused-panel selection memory restore the exact script row;
- repeated child suffix aliases remain accepted, while duplicate non-child owners, missing parents, self-links, cycles,
  and inconsistent tree depth remain no-ops;
- `l` on an ordinary child advances the owning workflow to reveal hidden steps, while `l` on an already-visible hidden
  leaf leaves the fully-expanded fold and selection unchanged;
- uppercase `H` on agent/Bash/Python children collapses the owning workflow before any group fold and reanchors to the
  same canonical owner;
- grouping mode, split/merged layout, sole-panel behavior, and Tools-detail precedence remain unchanged.

Extend footer unit tests for `parent workflow`, aliased family-owned Bash/Python steps, invalid ancestry, saturated leaf
expansion, and uppercase collapse precedence. Add or adapt a focused Agents PNG snapshot with a selected real-shape Bash
or Python step and audit the expected footer-only delta; do not mass-refresh unrelated goldens. Run the focused
navigation, structural-fold, footer, grouping, panel, jump-history, and help/docs tests first, then the complete
Agents/Tools PNG suite. After `just install`, finish with the repository-required `just check` and `git diff --check`.

This remains a Python TUI presentation/navigation change. The Rust core wire/API, loader persistence format, global
keymap defaults, and SASE memory files are out of scope.
