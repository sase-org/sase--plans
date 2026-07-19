---
tier: tale
title: Adaptive two-character TUI jump hints
goal: 'TUI entry-jump surfaces retain compact zero-based one-character hints for small
  target sets and switch to deterministic two-character hints when more than 62 visible
  targets need unique shortcuts.

  '
create_time: 2026-07-19 10:19:40
status: done
prompt: 202607/prompts/two_character_tui_hints.md
---

# Plan: Adaptive two-character TUI jump hints

## Context and scope

ACE currently assigns entry-jump hints from a 62-character alphabet ordered `1`–`9`, `0`, `a`–`z`, `A`–`Z`. Allocation
stops after 62 targets, and every consumer immediately resolves one printable keypress. This leaves later rows
unreachable by hint when a pane or modal contains more than 62 eligible targets.

This tale will evolve the shared entry-jump system used by the apostrophe mode, the cross-tab Jump All modal, non-PR
Artifacts panes, notification and saved-group modals, the model picker, and the logs pane. It will not change unrelated
numeric fold-selection hints, clan/family member numbers, file/hook hints, or modal accelerator key sets that do not use
the shared 62-character jump alphabet. The change stays in the Python/Textual presentation layer; it does not introduce
backend behavior or synchronous work on the UI thread.

## Hint allocation contract

- Define one canonical base-62 alphabet in the order `0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ`.
- Choose one fixed width for an entire hint session from the number of eligible targets. Counts from 1 through 62 use
  one-character hints beginning with `0`; counts from 63 through 3,844 use two-character hints beginning `00`, `01`, …,
  `09`, `0a`, …, `0Z`, `10`, and ending at `ZZ`.
- Preserve visual target order and bidirectional maps for agents, collapsed banners, split-panel headers, PRs, Axe rows,
  Artifact entries, and modal rows. On the Agents tab, choose the width from the complete visible jump-target list, not
  only the backing agent list, so banners and panel headers cannot silently consume the last one-character slots.
- Treat `ZZ` as the explicit fixed-width capacity. If a surface ever exceeds 3,844 eligible targets, keep the existing
  graceful truncation behavior for the remainder rather than introducing an unrequested third width.

Implement the allocation rules once in `src/sase/ace/tui/actions/navigation/jump_hints.py`, and make Jump All use the
same map builder instead of maintaining a second zip/render sequence. Keep uppercase keypress normalization
case-sensitive for each character.

## Multi-key dispatch lifecycle

Add a small shared prefix-matching abstraction for generated hint maps, with explicit outcomes for an incomplete prefix,
a completed target, and an invalid sequence. Each active jump surface should own/reset its tiny in-memory prefix state
while delegating matching semantics to that abstraction.

Wire it through the main ACE entry-jump path and every shared-alphabet modal:

- The first character of a two-character hint is consumed without navigating, closing jump mode, or repainting the full
  list; the second character resolves the exact target. One-character sessions continue to dispatch immediately.
- Escape cancels from either an empty or partial prefix. Invalid characters or invalid completed sequences retain the
  current surface-specific cancellation behavior, and every successful jump, tab/model refresh, or mode teardown clears
  the pending prefix so it cannot leak into the next session.
- Apostrophe retains its special first/back behavior and `Ctrl+O` retains its fast first/back behavior without
  synthesizing a literal hint such as `1` or `0`; both should select the first allocated target directly, so they work
  at either width. Jump All's backtick path and modal-specific back stacks likewise remain available as control actions
  rather than becoming hint characters.
- Preserve event consumption and shifted-uppercase handling in the app and modal key handlers, including when only the
  first character has been entered.

Update state declarations, test harnesses, and stale-mode cleanup paths alongside the handlers. Continue using the
Agents selective hint-refresh path; prefix entry must not add disk access, subprocess work, timers, or a full agent-list
rebuild.

## Rendering and documentation

The existing `[hint]` chips mostly accept arbitrary strings, but audit each shared consumer for one-cell assumptions.
Ensure two-character chips remain aligned in agent rows, collapsed banners, panel border titles, PR/Axe lists, Artifacts
rows, and modal content. In particular, derive Jump All's displayed hints from its allocated maps and allow panel
requested-width/cache calculations to reflect the extra cell without retaining stale one-character renders.

Revise `docs/ace.md` and nearby code/docstrings to describe adaptive one- or two-character entry hints, the zero-based
alphabet, the 62-target threshold, and the `00` through `ZZ` sequence. Remove statements that promise all jump hints are
single-keypress or that multi-character hints are unnecessary.

## Tests and validation

Expand the focused jump-hint tests before updating broad expectations:

- Unit-test exact allocator boundaries and ordering: one target, exactly 62, the 63-target transition (`00` and `10` at
  the expected positions), uppercase symbols, `ZZ`, and capacity truncation.
- Unit-test prefix matching for pending first characters, exact two-character matches, invalid prefixes/sequences,
  Escape/reset, case-sensitive uppercase, and unchanged immediate one-character matching.
- Add Agents-tab coverage with more than 62 visible targets to prove every row, collapsed banner, and split-panel title
  participates in the same two-character namespace and that entering both characters navigates only after the second.
  Retain exact-62 and small-list coverage proving compact hints still apply and now start at `0`.
- Exercise the shared multi-key lifecycle in representative app, non-PR Artifacts, Jump All, and modal/picker
  integrations. Cover apostrophe/fast-first, back-stack behavior, partial-prefix cancellation, uppercase second
  characters, successful cleanup, and model/tab refresh cleanup. Update existing literal `1`/`2` expectations to `0`/`1`
  where they describe the shared alphabet.
- Update intentional PNG goldens for small-list hints beginning at `0`, and add or extend a focused visual/layout
  assertion for a two-character chip so the extra width is protected without requiring a huge on-screen fixture.

Before final handoff, run `just install`, the focused jump/navigation and visual tests while iterating, and
`just check`. Accept visual snapshots only for the intentional zero-based or two-character rendering changes, then rerun
the visual suite normally to verify the checked-in goldens.
