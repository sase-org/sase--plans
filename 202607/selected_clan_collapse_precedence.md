---
tier: tale
title: Collapse the selected agent clan before the rest of its grouping scope
goal: Uppercase H first fully collapses only the agent clan that encloses the Agents-tab
  selection, and a later press fully collapses every remaining open clan in that same
  grouping scope before the grouping banner.
create_time: 2026-07-28 09:52:21
status: done
---

- **PROMPT:** [202607/prompts/selected_clan_collapse_precedence.md](prompts/selected_clan_collapse_precedence.md)
- **AGENTS:**
  - [bbugyi200.athena.my--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.my.md#member-code)
- **COMMITS:**
  - [a016706](https://github.com/sase-org/sase/commit/a016706a6118a9d5ce89e0ad817790a79d6d8fb1) — fix(ace): prioritize selected clan collapse

# Plan: Collapse the selected agent clan before the rest of its grouping scope

## Context and confirmed root cause

The supplied screenshot shows row focus on `sase-ac.6.1`, a direct member of the expanded `sase-ac.6` clan, inside the
`Running` status group of the expanded `@epic` split tribe panel. Sibling clan `sase-af` is expanded in the same group,
and clan `sase-ae` is already collapsed. One uppercase `H` in that state currently collapses `sase-ac.6` **and**
`sase-af` together, which throws away the sibling clan's expansion even though the user was working inside only one
clan. The requested behavior is a two-press transition: the first `H` collapses just `sase-ac.6` and leaves `sase-af`
open, and a second `H` — pressed while the now-collapsed `sase-ac.6` container is selected — collapses every remaining
open clan in that group, here `sase-af`.

The root cause is that the row-focused clan rung is deliberately selection-independent. `action_hooks_or_collapse_all`
in `src/sase/ace/tui/actions/agents/_folding.py` runs `_resolve_agent_house_collapse_target`, then
`_resolve_agent_clan_collapse_target`, then the selected structural fallback, then the grouping rung.
`_resolve_agent_clan_collapse_target` (in `src/sase/ace/tui/actions/agents/_folding_clans.py`) resolves the deepest
grouping scope the next `H` would reach and returns **every** open canonical clan in it, so `_collapse_agent_clan_folds`
batches all of them closed in a single press regardless of where the selection sits. The selected structural rung that
would have collapsed only the enclosing clan is never reached, because the group-wide rung always claims the press
first.

This is Textual presentation state. Keep the fix in the existing cached, synchronous fold-target machinery in this repo;
it must not introduce Rust-core behavior, I/O, awaits, background work, or a new refresh path.

## Behavioral contract and precedence

Add one new rung to the row-focused ladder, immediately after the group-wide house rung and immediately before the
existing group-wide clan rung. The new rung narrows the already-resolved group-wide clan target down to the single
canonical clan that encloses the current selection.

The Agents-tab `H` order under row focus becomes:

1. Preserve the Tools-detail compaction override.
2. Fully collapse every open canonical workflow/family house in the next grouping scope and stop if anything changed.
3. Otherwise, when the selection is enclosed by exactly one open canonical clan in that scope, fully collapse only that
   clan and stop.
4. Otherwise, fully collapse every remaining open canonical clan in that same scope and stop if anything changed.
5. Retain the selected structural transition as the defensive fallback it is today.
6. Only then perform the existing grouping-banner collapse/escalation.

"Encloses the selection" means the selected row is the clan's canonical synthetic container itself, a direct clan
member, or any deeper rendered descendant of that container. The narrowing rung must fail closed — falling through to
the unchanged group-wide sweep — in every other case, specifically when:

- a grouping banner is focused (`_current_group_key is not None`), so the selection is not a row inside a clan;
- the selection is a standalone lane or family that has no enclosing clan container;
- the enclosing clan's fold is already `COLLAPSED`, which is exactly the state produced by the first press and is what
  makes the second press sweep the rest of the group;
- the enclosing clan's fold key is not one of the keys the group-wide resolver already validated and returned, which
  covers malformed, duplicated, stale, cross-group, and cross-panel owners without re-deriving any ownership rules.

Because the second press's selection is the collapsed `sase-ac.6` container, the third and fourth clauses together
produce the requested transition without any new state: press one collapses `sase-ac.6`, press two collapses `sase-af`,
press three collapses the `Running` banner.

Every other surface keeps its current semantics: the separate expanded whole-panel ladder (panel houses, panel clans,
panel groups, then the panel), lowercase `h`, `l`, `L`, numeric fold selection, `Z` isolation, group-fold persistence,
the Tools-detail override, non-Agents tabs, and configurable key mappings. Do not change the configured key and do not
add a second action.

## Selected-clan resolution and canonical ownership

Resolve the group-wide clan target exactly once per press and derive the narrowed target from it by pure narrowing. Do
not add a second grouping-projection build: `_resolve_agent_clan_collapse_target` already builds a fully expanded
`build_agent_tree` projection with an empty `GroupFoldRegistry`, and the new rung must reuse that result rather than
repeat the work in a parallel resolver.

Implement the narrowing in `_folding_clans.py` beside the existing clan helpers:

- Add a module-level pure helper that returns the fold key of the canonical clan container enclosing the current
  selection, or `None`. Derive it structurally from the loaded projection — never from display text or dotted-name
  parsing. Use `agent_fold_key` when the selected row is itself a clan container, and otherwise walk the rendered tree
  with `tree_parent_lookup` plus `presentation_anchor`/`presentation_anchor_lookup` from
  `src/sase/ace/tui/models/_agent_tree.py` and accept the resolved anchor only when it is a clan container. Return
  `None` for a missing or out-of-range selection and for any anchor that is not a clan container.
- Add a mixin method on `AgentPanelClanFoldingMixin` that takes an `_AgentGroupClanCollapseTarget` and returns either a
  narrowed `_AgentGroupClanCollapseTarget` carrying exactly one fold key, or `None`. Reject immediately when
  `_current_group_key` is not `None`, when the helper returns `None`, or when the resolved key is not in the incoming
  target's `fold_keys`. Preserve the incoming `panel_key` and `group_key` unchanged.
- Compute the narrowed target's `reanchor_index` independently of the incoming target's. Set it to the index in the
  rendered `self._agents` projection of the unique non-child row whose `agent_fold_key` equals the selected clan key,
  but only when that row is not the currently selected row. Leave it `None` when the container itself is selected, so a
  press that does not hide the selection writes no selection memory. This deliberately covers deeper descendants that
  the group-wide resolver's direct-member-only re-anchor does not, because collapsing the clan hides them too.

Reuse `_collapse_agent_clan_folds` for the transition; the narrowed value is the same immutable dataclass, so no new
transition path, batching rule, or persistence behavior is introduced. In `_folding.py`, resolve the group target once,
attempt the narrowing, and pass whichever target results to the single existing collapse call.

## Focus, refresh, and performance safety

The press must remain one fold-only mutation batch followed by one lightweight in-memory refilter with
`refresh_content_index=False`, and must not write group-fold persistence intent. Because `_refilter_agents` captures the
selected identity from `current_idx` before rebuilding, setting the re-anchor before the collapse is what makes the
collapsed clan container the surviving selection; keep that ordering.

When the selection survives — the clan container was already the selected row — preserve its identity, attempt
selection, group/banner focus, jump history, focused panel, panel isolation state, and panel selection memory exactly as
the group-wide rung does today. When the selection is hidden, re-anchor to the clan container and update normal
split-panel selection memory once.

Resolution and the footer probe must stay bounded to already-loaded in-memory rows. No keypress or render path may add
filesystem reads, subprocess work, an extra grouping-projection build per press, or a full agent-list rebuild beyond the
established fold refilter.

## Conditional affordances and documentation

Use the same single resolve-then-narrow result for action dispatch and footer capability calculation so the footer never
advertises a different press than the one `H` will perform. Extend the footer inputs in
`src/sase/ace/tui/actions/agents/_display_detail_footer.py`, `src/sase/ace/tui/widgets/_keybinding_bindings.py`, and
`src/sase/ace/tui/widgets/_keybinding_modes.py` with one additional boolean that reports whether the next press is
narrowed to the selected clan. Render `H collapse clan` for the narrowed rung and keep `H collapse clans` for the
group-wide sweep and for the unchanged whole-panel clan probe. Preserve the higher-priority `compact tools` and
`collapse houses` labels, the existing singular structural label, `collapse group`, and custom key rendering.

Update the user-facing wording that currently describes the row-focused rung as selection-independent:

- the `hooks_or_collapse` / `hooks_or_collapse_all` comment block above the Agents keymaps in
  `src/sase/default_config.yml`;
- the Agents help-modal folding entries in `src/sase/ace/tui/modals/help_modal/agents_bindings.py`, staying within the
  established help-box width and per-entry description limits;
- the uppercase-`H` paragraph in `docs/ace.md`, which currently states that one press collapses every open canonical
  clan in the group "regardless of which clan or member row is selected";
- the `hooks_or_collapse_all` command-catalog label and aliases in `src/sase/ace/tui/commands/_app_metadata.py`, and the
  binding description in `src/sase/ace/tui/keymaps/metadata.py`, only where the existing wording has become inaccurate.

Keep every text assertion in sync in `tests/test_keymaps_display_help.py`, `tests/test_keymaps_app_bindings.py`, and
`tests/test_command_catalog.py`. Describe the behavior change in the commit message so release tooling picks it up; do
not hand-edit `CHANGELOG.md`.

## Regression coverage

Add focused transition coverage in the Agents fold-transition suites using projected clan rows and the real grouping
models, alongside the existing group-scoped clan tests rather than by rewriting them:

- Reproduce the screenshot under `BY_STATUS`: two open clans in one `Running` group of an expanded split tribe panel,
  selection on a direct member of the first clan. Assert the first `H` collapses only the selected clan, leaves the
  sibling clan and the `Running` banner open, re-anchors selection to the collapsed container, records panel selection
  memory once, performs exactly one refilter with `refresh_content_index=False`, and writes no group-fold change. Assert
  the second `H` collapses the sibling clan with the banner still open, and the third collapses `Running`.
- Cover selection on an expanded clan container itself: only that clan collapses, selection stays on the container, and
  no selection memory is written.
- Cover a deeper rendered descendant of a clan member with houses already closed: narrowing still resolves the ancestor
  clan and re-anchors to its container.
- Cover the single-open-clan case, where the narrowed press must leave exactly the same end state the group-wide rung
  produced before this change.
- Prove fall-through parity for every fail-closed case: a focused collapsed child banner, a standalone lane with no
  enclosing clan, an already-collapsed enclosing clan with an open sibling, and malformed or globally duplicated clan
  owners. In each case the unchanged group-wide sweep must still close every valid open clan in the scope, and malformed
  candidates must stay untouched.
- Assert scope isolation is unchanged: clans in sibling status/date/name groups, in other split tribe panels with
  identical display group keys, and under merged layout are never reached by narrowing.
- Extend footer integration and unit coverage to prove `collapse clan` for the narrowed rung versus `collapse clans` for
  the group-wide sweep, while retaining the existing whole-panel, Tools-detail, custom-key, help, and command-catalog
  assertions.
- Add one exact ACE PNG interaction scenario matching the screenshot shape — a selected direct member of an open clan
  with an open sibling clan in the same status group — asserting the pre-action `collapse clan` footer and the
  post-press state where only the selected clan is collapsed. The existing
  `agents_group_clan_collapse_precedence_120x40` golden selects an already-collapsed clan and must remain unchanged;
  audit any generated diff and accept only the intentional new Agents golden.

The existing selected-panel clan tests in `tests/ace/tui/test_agent_fold_transitions_panel_clans.py` remain regression
coverage for the distinct panel-wide ladder and must not be weakened or rewritten.

## Validation

Run `just install` first, since the ephemeral workspace may be stale. Then run the Agents fold-transition suites (group
clans, panel clans, tree, groups, tools), the footer integration and unit tests, and the help, keymap, and
command-catalog tests. Run the dedicated Agents visual suite with `just test-visual` without update mode after auditing
any intentional golden, using `--sase-update-visual-snapshots` only to accept the reviewed new snapshot. Finish with the
repository-required `just check` and `git diff --check`, investigating failures individually rather than accepting
unrelated snapshot drift.

## Non-goals

Do not scope the house rung to the selected clan; houses keep their group-wide saturating behavior and keep running
first. Do not change clan projection or grouping ownership in Rust core, lowercase parent navigation, clan expansion
with `l`, the `L` fold-hint selector, numeric fold mode, grouping order or persistence, whole-panel selection,
isolation, or the panel-wide clan batch. Do not change `H` behavior on ChangeSpecs, Axe, or Tools detail, and do not add
a new keymap action or rebind `H`.
