---
tier: tale
title: Agents-tab l jumps to a clan's lone running member
goal:
  Pressing `l` on an Agents-tab clan row expands the clan and selects its running member
  iff that member is the clan's only running member; otherwise `l` is unchanged.
size: medium
decisions:
  footer_hint:
    ask:
      Should the Agents footer show `l running member` whenever the selected clan has
      exactly one running member?
    choices:
      show:
        Footer gets an `l running member` chip there; every golden selecting such a clan
        is regenerated
      omit: "`l` stays documented only in help and docs; fewer PNG goldens change"
    default: show
    why: Mirrors the `h parent clan` chip, so the jump is discoverable where it applies
    answer: show
proposed_by: bbugyi200.athena.16t
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 16:53:06
status: wip
---

# `l` on an Agents-tab clan jumps to its lone running member

## Goal

On the Agents tab, pressing `l` on a clan row should expand the clan and move the
selection to its running member when that member is the only running member of the clan.
With zero or two-plus running members, `l` keeps today's behavior.

Example from `~/tmp/screenshots/20261101_163839.png`: the `research.2z` clan row is
selected in the `@research` tribe panel. The clan is already expanded and shows
`[R1 W2 D5]`. Its only running member is `research.2z.final` (wave 2). Today `l` does
nothing there. After this change, `l` selects the `research.2z.final` row.

## Current behavior (what changes)

`AgentGroupFoldingMixin._expand_fold` in
`src/sase/ace/tui/actions/agents/_folding_agent_groups.py` handles the Agents tab:

- Whole-panel focus: `l` expands or enters the panel. **Unchanged.**
- Grouping-banner focus (`_current_group_key` set): `l` expands the banner.
  **Unchanged.**
- Selected row: for a clan container (`agent.is_clan_container`), `l` expands the clan
  fold only when it is `FoldLevel.COLLAPSED` and is otherwise a no-op (the clan's outer
  fold is binary). For other rows, `l` steps that row's workflow/session fold.

`AgentFoldingMixin.action_expand_or_layout` (`_folding.py`) routes the Archive
(`route_archive_key_if_active`) and a focused Tools/LLM Calls card
(`_route_llm_calls_detail_level`) before it reaches `_expand_fold`. Both routes stay as
they are.

## Target behavior

This applies only to a selected clan-container row on the Agents tab: no whole-panel
focus, no banner focus, not the Archive, no focused Tools card.

| Running members | Clan fold | `l` does                                        |
| --------------- | --------- | ----------------------------------------------- |
| exactly 1       | collapsed | expand the clan **and** select that member      |
| exactly 1       | expanded  | select that member (fold unchanged)             |
| 0 or ≥2         | collapsed | expand the clan, keep the clan selected (today) |
| 0 or ≥2         | expanded  | no-op (today)                                   |

Definitions (decided, not open):

- **Running** uses the same rule as the clan row's `R` count chip
  (`clan_member_counts(...).running` in `src/sase/ace/tui/models/_agent_clan.py`). A
  member is running when it is a unique direct clan member (`clan_members(agent)`,
  deduped by `identity`) and its `agent_status_bucket(member)` is `"Running"` or
  `"Starting"`. A `FINALIZING` or `RETRYING` overlay still counts, since the bucket is
  Running. Waiting, Queued, Done, Failed, and Stopped/asking members neither count nor
  block the jump. So the jump fires exactly when the row shows `R1`.
- **Target row**: the direct clan member row. This is the same row a `0-9` member jump
  selects. For a sequential-session lane it is the session row, not its executing turn.
  For a workflow it is the workflow row. The member's own fold is not opened.
- **Jump semantics**: reuse the existing member-jump reveal path. It records a back
  anchor in the Agents jump history like digit member jumps and `h`. It expands only the
  folds on the member's path (clan fold plus any grouping banner, such as the by-status
  `Running` group), re-focuses the member's panel, and acknowledges unread state. No new
  fold-mutation code.
- **Failure**: if the reveal fails (for example, a search filter hides the running
  member), show no toast and fall back to today's behavior. Expand if collapsed and keep
  the clan selected. The real `_refilter_agents` (`_loading_filter.py`) keeps the
  selection by identity, so the clan stays selected even if the reveal already opened
  the clan fold.
- After the jump the member row is selected, so the next `l` applies the member's own
  fold rules (for example, it expands a session's turns). `h` from the member returns to
  the clan through the existing structural-parent navigation.

## Implementation

### 1. Model helper: `clan_lone_running_member`

In `src/sase/ace/tui/models/_agent_clan.py`, add
`clan_lone_running_member(agent: Agent) -> Agent | None` next to and modeled on
`clan_lone_stopped_member`:

- Return `None` for non-clan rows.
- Walk `clan_members(agent)`, dedupe by `identity`, and count members whose
  `agent_status_bucket(member)` is in a shared running-bucket set
  (`{"Running", "Starting"}`).
- Return the single match. Return `None` for zero or two-plus matches, short-circuiting
  on the second.
- Hoist the running-bucket set into one private module constant and use it in both the
  new helper and the `elif bucket in {"Running", "Starting"}` branch of
  `clan_member_counts`, so the jump rule and the `R` chip cannot drift apart.

Keep this in Python. It is TUI navigation over already-loaded TUI rows, the same layer
as `clan_member_counts` and `clan_lone_stopped_member`, and no other frontend needs it.
It is not core-backend logic.

### 2. Resolver plus `_expand_fold` hook

In `src/sase/ace/tui/actions/agents/_folding_clans.py` (`AgentPanelClanFoldingMixin`,
which `AgentGroupFoldingMixin` inherits), add:

- `_resolve_clan_running_member_jump_target(self) -> Agent | None`. Return the lone
  running member only when all of these hold: `self.current_tab == "agents"`; no
  whole-panel focus (`_resolve_focused_panel()` is `None`);
  `_current_group_key is None`; the selected agent (`_get_selected_agent()`) is a clan
  container; and `clan_lone_running_member` returns a row. The footer (step 3) reuses
  this one resolver, so the chip and the key cannot disagree. Do not check the Archive
  or Tools-card routes here. The dispatcher routes both before `_expand_fold`. The
  Archive has its own footer path (`_apply_archive_footer_update`), and the live footer
  already gates on `llm_calls_visible`.
- `_jump_to_clan_running_member(self, member: Agent) -> bool`. Look up
  `_try_reveal_agent_row` via `getattr`; it lives on `MemberJumpNavigationMixin` in
  `src/sase/ace/tui/actions/navigation/_member_jump.py` and is composed into the real
  app. Return `False` when that method is absent. Otherwise return
  `try_reveal(member.identity) is None`. Use `_try_reveal_agent_row` rather than
  `_reveal_agent_row`, which would show a "Member is no longer visible" toast.

In `_expand_fold`'s Agents selected-row branch
(`src/sase/ace/tui/actions/agents/_folding_agent_groups.py`), inside
`if agent.is_clan_container:` and before the existing `!= FoldLevel.COLLAPSED` early
return: resolve the target; if a member is found and
`_jump_to_clan_running_member(member)` succeeds, `return`. Otherwise fall through
unchanged. Don't add another refilter or refresh: `_try_reveal_agent_row` already
refilters after a fold change and refreshes the display when structure or panel focus
changed. Its `current_idx` assignment covers the already-expanded case, as it does for
digit jumps.

Update the `_expand_fold` docstring to state the clan rule.

### 3. Footer chip (per `footer_hint`)

> [!decision] footer_hint = show
>
> - `src/sase/ace/tui/actions/agents/_display_detail_footer.py`: next to
>   `left_navigation_kind`, compute
>   `running_member_jump_available = not panel_focused and not llm_calls_visible and resolver() is not None`.
>   Use a `getattr` for `_resolve_clan_running_member_jump_target`, matching the
>   neighboring resolver lookups. Pass it to `footer_widget.update_agent_bindings(...)`.
> - `src/sase/ace/tui/widgets/_keybinding_modes.py`: thread a new
>   `running_member_jump_available: bool = False` keyword through both
>   `update_agent_bindings` signatures and the forwarding call, as is done for
>   `left_navigation_kind`.
> - `src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`: add the same keyword to
>   `_compute_agent_bindings`. Next to the `left_navigation_kind` block, append
>   `(self._kd("expand_or_layout"), "running member")` when it is true and
>   `not panel_focused`. The shared footer layout handles sorting.
> - Tests: extend `tests/ace/tui/widgets/test_keybinding_footer_llm_calls_detail.py` (or
>   a sibling footer test) to cover: the chip appears when the flag is true, is absent
>   by default, and is suppressed under `panel_focused=True`. Extend the producer-side
>   test pattern in `tests/ace/tui/test_agent_display_defer_detail.py` (which asserts
>   `call["left_navigation_kind"]`) to assert the new kwarg for a selected lone-running
>   clan.

> [!decision] footer_hint = omit
>
> Skip this step entirely; the footer is unchanged.

### 4. Help modal and config comment

- `src/sase/ace/tui/modals/help_modal/agents_mode_sections.py`: in the "Panel / Group /
  Clan / Workflow Folding" section, add a row after the existing
  `(d(a.expand_or_layout), "Expand fold / enter panel (❯)")` row:
  `(d(a.expand_or_layout), "Clan: jump to its lone running member")`. Add that pair to
  the assertions in `tests/test_keymaps_display_help_agents.py`.
- `src/sase/default_config.yml`: the comment above `hooks_or_collapse` /
  `expand_or_layout` (around "l expands or enters a selected panel") gains one clause.
  On a clan row whose only running member is unique, `l` also jumps to that member. The
  key binding itself does not change. Leave the generated keymap label
  `Expand / Enter Panel` in `src/sase/ace/tui/keymaps/metadata.py` alone, so the
  generated `docs/reference/tui/keymaps.md` row needs no regeneration.

### 5. Docs (Reference pages; behavior of an existing feature)

- `docs/reference/agents/agent-clans.md` (the paragraph with "Press `l` once on a
  collapsed clan to reveal its direct members"): add one or two sentences. When exactly
  one direct member is running (the row shows `R1`), `l` on the clan row also selects
  that member. It expands the clan first if needed, like any member jump. With zero or
  several running members, `l` only reveals the members.
- `docs/reference/tui/agents-tab-grouping.md` (the sentence ending "pressing `l` again
  on the clan row itself has no effect"): that sentence becomes false. Reword it to say
  that a second `l` on the clan row has no effect unless the clan has exactly one
  running member, in which case `l` jumps to it. Link to the clan reference instead of
  restating the details.

Keep both edits within the page budgets in `docs/contribute/writing-docs.md`. Run
`just fmt` for Markdown wrapping.

### 6. Tests

Unit tests:

- `tests/test_agent_clan.py`: `clan_lone_running_member` returns `None` for a non-clan
  row. It returns the lone `RUNNING` member and treats a lone `STARTING` member as
  running. It returns `None` for two running members and for zero running members
  (Done/Waiting/Failed only). It ignores non-running members alongside the lone runner,
  dedupes duplicate identities, and respects a row-level bucket override (a member whose
  bucket is overridden to Done does not count). Assert agreement with
  `clan_member_counts(...).running == 1` on the same fixtures.
- New module, for example `tests/ace/tui/test_agent_clan_running_member_l_jump.py`. Use
  a harness that composes `AgentFoldingMixin` with the production-shaped member-jump
  harness (`JumpHarness` in `tests/ace/tui/_member_jump_navigation_helpers.py` or
  `ClanJumpHarness` in `tests/ace/tui/_agent_jump_clan_helpers.py`), so the real
  `_try_reveal_agent_row` runs. Cover:
  - Collapsed clan with one running member: `action_expand_or_layout()` leaves the clan
    fold not collapsed, selects the member by identity, and pushes one back anchor.
  - The screenshot case: an already-expanded clan with one running member plus
    Done/Waiting members. `l` selects the running member, and every fold state is
    unchanged.
  - Two running members: a collapsed clan only expands, and the clan stays selected. An
    expanded clan is a no-op.
  - Zero running members: today's behavior.
  - Reveal failure (target filtered out, for example by a search query in the harness):
    no notification, the clan stays selected, and it expanded if it was collapsed.
  - Whole-panel focus and banner focus still take their existing branches.
- Existing `tests/ace/tui/test_agent_fold_transitions_tree.py` tests use `StubFoldApp`,
  which lacks `_try_reveal_agent_row`, so they keep today's semantics. Confirm they
  still pass rather than rewriting them.
- One real-app test in the style of
  `test_real_app_named_agent_jump_expands_collapsed_clan`
  (`tests/ace/tui/test_agent_jump_clan_members.py`): launch `AcePage`, select a clan
  with one running member, press `l`, and assert the selected identity.

Visual PNG tests, `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`. These
fixtures already contain lone-running clans, so behavior changes there:

- `test_clan_wave_gutter_png_snapshot`: the `wave-epic` clan's only running member is
  `wave-epic.family`. The test currently presses `l` then `j` to reach it. Drop the `j`
  and assert that `l` alone lands on `wave-epic.family`. The following `l` and the
  captured golden should then be unchanged.
- `test_clan_tree_fold_levels_png_snapshots` (`clan_tree_agents`, `[R1 W1 D1]`): after
  the first `l`, assert that the selection is `research.family`, the lone running
  session row. The `agents_clan_tree_expanded_120x40` golden changes because the
  selection and detail panel now show the member. Keep the later `j`-to-`research.audit`
  loop working; it starts from the member row now.
- `test_clan_unread_count_png_snapshots` (same fixture): the
  `agents_clan_unread_expanded_120x40` golden changes for the same reason. Assert the
  new selection.
- `running_clan_runtime_agents` and the other suites that press `l` on rows inside a
  clan (`*_agents.py`, `*_agents_stranded.py`, `*_agents_neighbors.py`,
  `*_agents_archive_list.py`, `test_agent_panel_collapse_isolation.py`,
  `test_agent_fold_transitions_navigation.py`): check each `l` press against the target
  table. Update assertions only where the selected row is a clan container with exactly
  one running member.

Golden regeneration (see the TUI screenshot memory's Golden Maintenance rules): run
`just fix-tui-screenshots` through `/sase_monitor`. Use the targeted form
(`-- tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`) for
`footer_hint = omit`. Use the full run for `footer_hint = show`, because every snapshot
with a lone-running clan selected gains the chip. In the follow-up, inspect the report
and every changed golden. Expected changes: only the selection/detail move described
above and, under `show`, the added footer chip. Investigate any other diff. Prove host
independence with the targeted `--check` run twice, once with `TZ=UTC`, before
committing images.

## Verification

1. `sase tool run check` (the `just check` lint gates plus the scoped test lane) passes.
2. The targeted (or full, per `footer_hint`) `just fix-tui-screenshots` run is inspected
   and its `--check` reruns are clean.
3. Optional live confirmation: `sase screenshot` (see the TUI screenshot memory) of the
   Agents tab with a lone-running clan selected, before and after pressing `l`. Inspect
   the PNG to confirm that the running member row is highlighted, and under `show` that
   the footer shows `l running member` on the clan row.

## Out of scope

- Changing `L`, `h`, `H`, digit member jumps, Enter's lone-stopped-member delegation, or
  the Archive's `l`.
- Descending into a running member's own session turns or workflow steps.
- Any Rust core (`sase-core`) change.
