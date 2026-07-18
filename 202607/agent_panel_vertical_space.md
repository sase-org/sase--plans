---
tier: tale
title: Use all vertical space for overflowing agent panels
goal: 'The Agents tab keeps compact tag panels readable while allowing an oversized
  untagged panel to consume all remaining vertical space without leaving a blank region
  at the bottom of the panel stack.

  '
create_time: 2026-07-18 14:00:50
status: wip
prompt: 202607/prompts/agent_panel_vertical_space.md
---

# Plan: Use All Vertical Space for Overflowing Agent Panels

## Context and root cause

The Agents tab stacks one `AgentList` per untagged or tagged group inside `#agent-list-container`. TCSS supplies only a
`1fr` fallback; the effective heights are assigned by `PanelLayoutMixin._apply_panel_heights()` in
`src/sase/ace/tui/actions/agents/_display_panel_layout.py` after each relevant refresh and again when terminal geometry
changes.

The overflow allocator normally protects collapsed panels and compact expanded panels with fixed cell heights, then
gives the remaining large panels content-weighted fractional heights. A special case currently removes the first
split-mode untagged panel from that process: if its natural height exceeds half the content budget, the allocator fixes
it at exactly half the budget. Once the later tag panels are also found to fit at their natural heights, every panel is
fixed and Textual has no fractional child that can absorb the leftover rows. The result is an internally scrolling
untagged panel plus unused vertical space below the tag panels, as in the reported screenshot.

This behavior is presentation-only and remains in the Python TUI layer; it does not require a Rust core or TCSS change.

## Sizing policy

Update `_apply_panel_heights()` so an oversized expanded untagged panel follows the same overflow allocation rules as
every other expanded panel:

- Preserve the fits regime: when all natural panel heights and separators fit, one expanded panel remains fractional to
  absorb slack and every other panel uses its natural cell height.
- Preserve border-only fixed heights for collapsed panels and the existing very-short-terminal fallback.
- Preserve the ascending-natural-height protection pass so compact panels, including a genuinely small untagged panel,
  stay fully visible when the remaining panels can take the cropping pressure.
- Remove the half-budget fixed-height reservation for a large first untagged panel. It should remain an overflow
  candidate and therefore receive a fractional height after compact panels are protected. In the reported shape, compact
  tagged panels keep their natural sizes while the untagged panel expands through all remaining rows and scrolls only
  when its contents still exceed that larger viewport.
- Retain `option_count + 1` fractional weights when multiple large panels must share the remaining budget, so
  eliminating the special cap does not make one large panel unconditionally dominate other large panels.

Keep the calculation synchronous and geometry-only. Do not add filesystem work, option-list rebuilds, timers, or a
second refresh path; resize handling must continue to call `_reapply_panel_heights()` over the existing widgets.

## Regression coverage

Update `tests/ace/tui/test_agent_panels_display.py` to describe the revised overflow contract:

- Keep coverage proving a small untagged panel is fixed at its natural height.
- Replace the half-height-cap expectation with a screenshot-shaped case where an oversized untagged panel coexists with
  compact tagged panels and the full natural stack barely overflows. Assert that compact panels remain fixed while the
  untagged panel is fractional, ensuring some expanded child can absorb all slack.
- Retain coverage for proportional sharing between multiple large panels, separator accounting, collapsed panels, very
  short containers, the fits regime, and a single-panel layout.
- Keep the resize regression proving height recomputation does not call `update_list()` or rebuild agent rows, and
  exercise the revised overflow result through that path where useful.

Add or extend an Agents-tab PNG visual scenario under `tests/ace/tui/visual/` with enough untagged rows to overflow
alongside small tag panels. After layout settles, assert through widget/container geometry that the stack reaches the
bottom of the available list container, assert the expected fixed-versus-fractional height roles, and capture the 120x40
golden. The snapshot should visibly preserve the compact panels while showing the untagged viewport using the recovered
rows. Accept only the intentional golden change for this new or directly affected scenario.

## Verification

Run focused unit tests first, then the visual regression and the repository-wide required checks:

1. `just install`
2. `just test tests/ace/tui/test_agent_panels_display.py`
3. `just test-visual` (inspect actual/expected/diff artifacts before accepting any intentional snapshot update)
4. `just check`

## Risks and safeguards

- Removing the cap must not regress the original reason for the overflow allocator: a tiny tag panel should still show
  its banner and agent rows when a larger panel can scroll instead. The compact-panel unit and visual assertions guard
  this explicitly.
- Mixed fixed-cell and fractional Textual sizing can differ from scalar-level expectations once borders and margins are
  applied. Geometry assertions plus the PNG snapshot guard the actual rendered bottom edge and separator rows.
- All-expanded-large and very-short-terminal layouts still need fair degradation. Retaining the existing weighted
  fractional branch and its tests prevents the fix from turning into an unbounded untagged-panel preference.
- The allocation runs in refresh and resize hot paths. Keeping the change to the existing in-memory arithmetic and
  preserving the no-rebuild resize test avoids a TUI responsiveness regression.
