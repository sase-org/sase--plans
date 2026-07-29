---
tier: tale
title: Agent-lane counts in kill/dismiss confirmations
goal:
  Every kill/dismiss confirmation's headline count equals the number of agent lanes listed beneath it, with the concrete
  cleanup-row total shown as a secondary detail only when it differs, without changing cleanup scope or execution.
create_time: 2026-07-29 09:07:34
status: done
---

- **PROMPT:** [202607/prompts/lane_counts_in_cleanup_confirmations.md](prompts/lane_counts_in_cleanup_confirmations.md)

# Plan: Report agent-lane counts in kill/dismiss confirmations

## Problem

The Agents-tab kill/dismiss confirmation dialogs print a headline count that is computed from **concrete cleanup rows**
while the roster printed directly beneath it is the **de-duplicated agent-lane projection**. The two numbers disagree
whenever a lane is a sequential family, a workflow with steps, or a clan member with descendants.

Reported case (tribe panel `@research`, whose panel border title reads `@research · 8`):

```
Panel: @research (48 agents)
Dismiss: 48 agents
  › research.o.image
  › research.o.final
  › research.o.cld
  › research.o.cdx
  › research.p.image
  › research.p.final
  › research.p.cld
  › research.p.cdx
```

Eight lanes are listed, the panel chrome says 8, and the dialog says 48.

The same defect is already baked into a golden PNG snapshot:
`tests/ace/tui/visual/snapshots/png/agent_lane_cleanup_confirmation_120x40.png` renders `Dismiss: 9 agents` above a
three-entry lane roster.

### Root cause

Commit `339e06f65` ("fix(ace): summarize cleanup confirmations by agent lane") introduced
`src/sase/ace/tui/actions/agents/_confirmation_lanes.py` and routed every roster through `confirmation_lane_entries()`,
which collapses workflow descendants and sequential-family members into one lane each. Its plan deliberately said
"Retain useful scope/action headers and concrete cleanup counts", so the counts kept using `len(...)` over the concrete
row lists:

- `src/sase/ace/tui/actions/agents/_marking_kill.py:201` — `Kill: {len(killable)} running agent…`
- `src/sase/ace/tui/actions/agents/_marking_kill.py:214` — `Dismiss: {len(dismissable)} agent…`
- `src/sase/ace/tui/actions/agents/_kill_all_actions.py:60` — `Kill: {len(killable)} running agent…`
- `src/sase/ace/tui/actions/agents/_kill_all_actions.py:73` — `Dismiss: {len(dismissable)} completed agent…`
- `src/sase/ace/tui/actions/agents/_dismissing.py:124` — `Count: {len(dismissable)} agent…`
- `src/sase/ace/tui/actions/agents/_kill_action_flow.py:80,182,212` — `Clan:` / `Panel:` / `Group:` headers ending in
  `({count} agent{plural})`

That retention decision is exactly what now reads as wrong; this plan amends that contract.

### Secondary defect found while diagnosing

The three `_kill_action_flow.py` header counts are computed **before** `_present_bulk_kill_modal()` expands clan
containers (`src/sase/ace/tui/actions/agents/_marking_kill.py:159-178`). `_bulk_kill_group_agents()` and the focused
clan-container branch therefore count synthetic clan rows rather than the rows actually cleaned up, so those headers can
under-count even in the concrete-row unit they claim to use. Dropping the header counts (below) removes that divergence
instead of duplicating the expansion logic in the caller.

## Behavior contract

1. Every confirmation section line reports **agent lanes**, and that number is always exactly the number of `›` roster
   entries rendered directly beneath it. This is the defining invariant of the fix.
2. When the concrete cleanup-row count for a section differs from its lane count, the line appends `· <n> agents`. When
   the two are equal the suffix is omitted, so simple confirmations stay terse.
   - `Dismiss: 8 lanes · 48 agents`
   - `Dismiss: 3 lanes`
   - `Kill: 1 lane`
3. Scope headers `Panel:`, `Clan:`, and `Group:` drop their parenthesized agent count and become pure scope labels
   (`Panel: @research`), matching the already-count-less `Tribe: @x`, `Tribes: …`, `Clans: N selected`, and
   `Custom selection` headers. The per-section lane counts underneath carry the magnitude.
4. Section lane counts are independent. A lane may legitimately appear in both the `Kill:` and `Dismiss:` sections (the
   operation kills its active member and dismisses earlier completed members), so the two section counts may sum to more
   than the number of distinct lanes. That is pre-existing, intended roster behavior and is not "fixed" here.
5. The `Kill:` / `Dismiss:` adjectives (`running`, `completed`) are dropped, because the section label plus the modal
   message ("Kill running agents and dismiss completed agents?") already carry that distinction and
   `Kill: 2 lanes · 5 running agents` reads badly. This also removes the existing inconsistency where one builder says
   `Dismiss: N agents` and another says `Dismiss: N completed agents`.
6. Cleanup scope and execution are unchanged: the `killable` / `dismissable` lists handed to the confirmed action, the
   cascade, optimistic removal, persistence, notification dismissal, task display names, and the post-action toasts all
   keep operating on and reporting concrete rows. The toast `Dismissed 48 agents` stays as-is and now matches the
   `· 48 agents` detail in the confirmation it followed.
7. Confirmations that show a single lane and no count — the focused-agent `Agent lane:` kill subject
   (`_kill_action_flow.py:100`), the wait-relaunch subject (`_wait_actions.py:445`), and the kill-and-edit relaunch
   subject (`agent_workflow/_entry_relaunch.py:252`) — are unchanged.
8. `ConfirmDialog` styling and parsing are unchanged. `_SUBJECT_COUNT_RE` in
   `src/sase/ace/tui/modals/confirm_dialog.py:34` already splits `Dismiss: 8 lanes · 48 agents` into the styled count
   `8` plus dim units ` lanes · 48 agents`, and a count-free `Panel: @research` header falls through to the same
   `plain_subject` style it uses today. Non-agent confirmations, including the AXE background-command
   `ConfirmKillModal`, are untouched.

## Implementation

### 1. Extend the shared lane projection with a counted summary

`src/sase/ace/tui/actions/agents/_confirmation_lanes.py`

- Add a frozen dataclass `AgentConfirmationSummary` holding `entries: tuple[_AgentConfirmationEntry, ...]` and
  `agent_count: int`, with a `lane_count` property (`len(self.entries)`).
- Add `confirmation_lane_summary(targets, loaded_agents, *, include_running_family_members=False)` that calls the
  existing `confirmation_lane_entries()` once and counts **unique** concrete targets. De-duplicate on `Agent.identity`,
  falling back to `id(row)` when the identity attribute is missing or unhashable, matching the defensive posture the
  rest of this module already uses for legacy rows.
- Add `AgentConfirmationSummary.subject_lines(label: str) -> list[str]` returning
  `[f"{label}: {phrase}", *format_confirmation_entries(self.entries)]`, where `phrase` is `"<n> lane"` / `"<n> lanes"`
  plus `" · <m> agents"` (or `" · 1 agent"`) only when `m != n`. Return `[]` when there are no entries so callers never
  emit an orphan header.
- Keep `confirmation_lane_entries()` and `format_confirmation_entries()` exported and behaviorally unchanged — the
  single-lane call sites and the projection unit tests still use them directly. Add the two new names to `__all__`.

Building the count inside the same helper that builds the roster is what makes the contract-1 invariant structural
rather than a convention each call site has to remember.

### 2. Route the counted call sites through it

- `src/sase/ace/tui/actions/agents/_marking_kill.py` (`_present_bulk_kill_modal`): replace both hand-built
  count-plus-`extend` blocks with `desc_parts.extend(summary.subject_lines("Kill"))` and `…("Dismiss")`. Keep
  `include_running_family_members=True` on the kill summary and the clan-container expansion above it exactly as-is, and
  keep passing the unmodified `killable` / `dismissable` lists to the confirm callback.
- `src/sase/ace/tui/actions/agents/_kill_all_actions.py` (`_kill_and_dismiss_agents_from`): same substitution for its
  `Kill:` / `Dismiss:` sections.
- `src/sase/ace/tui/actions/agents/_dismissing.py` (`_dismiss_done_agents_from`): replace the `Count: N agent…` line and
  its separate `format_confirmation_entries(...)` call with `subject_lines("Dismiss")`. This intentionally renames the
  label from `Count:` to `Dismiss:` so both dismiss-all entry points (`x` on a panel and the `X` cleanup panel's
  "Dismiss Panel Done" / "Dismiss All Done" actions) render an identical subject; call it out in the commit message.
- `src/sase/ace/tui/actions/agents/_kill_action_flow.py`: make the `Clan:`, `Panel:`, and `Group:` headers label-only
  and delete the now-dead `count` / `plural` locals in `action_kill_agent`'s clan-container branch,
  `_bulk_kill_panel_agents`, and `_bulk_kill_group_agents`. Leave the surrounding focus-revalidation and
  membership-change guards untouched.

### 3. Explicitly out of scope

No changes to `_kill_cleanup_selection.py` / `_kill_cleanup_clan.py` headers (already count-less, and
`Clans: N selected` counts clans not agents), `_present_planned_cleanup()`, `_build_agent_cleanup_panel_state()` counts
in the `X` cleanup panel, panel border-title lane counts, the tribe-detail `Composition:` line, dismissal toasts, or any
Rust core code. This is presentation-only subject-string assembly consumed by exactly one frontend, so it stays in
Python per the core-backend boundary litmus test.

## Tests

- `tests/ace/tui/test_agent_confirmation_lanes.py` — new coverage for `confirmation_lane_summary` / `subject_lines`:
  family lane collapsing many rows (`Dismiss: 1 lane · 4 agents`), workflow-plus-hidden-steps, an all-standalone set
  that omits the `· n agents` suffix, singular `1 lane` and `· 1 agent` forms, repeated concrete targets counted once,
  and an empty target list producing no lines. Add an explicit invariant test that parses the emitted section line's
  leading integer and asserts it equals the number of roster lines. Update
  `test_bulk_subject_can_show_same_family_lane_in_kill_and_dismiss_sections` to the new wording and keep it as the
  both-sections-share-a-lane case.
- `tests/ace/tui/test_agent_collapsed_panel_kill.py:126-128,161,182,224,263` — headers become `Panel: @chop` etc.;
  assert the new `Kill:` / `Dismiss:` lane lines.
- `tests/ace/tui/test_agent_group_kill.py:188` — `Clan: research (2 agents)` becomes `Clan: research`; add the section
  assertion that now carries the count.
- `tests/ace/tui/test_agent_cleanup_clan_e2e.py:128-130`, `tests/ace/tui/test_agent_bulk_kill_edit.py:335-336`,
  `tests/ace/tui/test_panel_scoped_bulk.py` — update section wording; `_names_in_modal()` parses indented roster lines
  and should keep working unchanged, which is a useful signal that the roster format is untouched.
- `tests/ace/tui/modals/test_confirm_dialog.py:17-25` and
  `tests/ace/tui/visual/test_ace_png_snapshots_confirm_dialog.py:92` — these pass literal subject fixtures; update them
  to the new `Dismiss: 2 lanes` wording so the styling test covers the string shape actually produced.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py:262` — header assertion.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_cleanup.py` — add an assertion that the description contains
  `Dismiss: 3 lanes · 9 agents` (9 concrete rows: standalone, workflow, 3 steps, family root, 3 members), then
  regenerate `agent_lane_cleanup_confirmation_120x40.png`.

## Validation

1. `just install` first — this is an ephemeral SASE workspace and its virtualenv may be stale.
2. Targeted run: `just test` scoped to
   `tests/ace/tui/test_agent_confirmation_lanes.py tests/ace/tui/test_agent_collapsed_panel_kill.py tests/ace/tui/test_agent_group_kill.py tests/ace/tui/test_agent_cleanup_clan_e2e.py tests/ace/tui/test_agent_bulk_kill_edit.py tests/ace/tui/test_panel_scoped_bulk.py tests/ace/tui/test_agent_wait_resume.py tests/ace/tui/test_family_member_relaunch.py tests/ace/tui/modals/test_confirm_dialog.py`.
3. `just test-visual`. Inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/`, then accept the
   intentional change with `--sase-update-visual-snapshots` and **open the regenerated PNG** to confirm the dialog reads
   `Dismiss: 3 lanes · 9 agents` above exactly three `›` rows.
4. `just check` (mandatory for any file change in this repo).
5. Re-grep `src/sase/ace/tui/actions/agents/` for `agent{plural}`, `agent{s}`, `agent{k_s}`, `agent{d_s}`, and `Count:`
   to confirm no confirmation-subject builder still reports concrete rows as its headline number, and that the remaining
   hits are only toasts / task display names (which intentionally keep concrete counts).
6. Manual reproduction of the reported case: in a live `sase ace`, focus a tribe panel whose lanes are sequential
   families or clan members, press `x`, and confirm the section count equals both the roster length and the panel border
   title's lane count, with `· N agents` reporting the larger concrete total.
