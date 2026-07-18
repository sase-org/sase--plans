---
tier: tale
title: Gate primary action footer
goal: 'Gate review footers name and visually emphasize the action triggered by the
  configured primary-submit key, so reviewers can recognize the default choice immediately
  without exposing internal branch terminology.

  '
create_time: 2026-07-18 10:01:24
status: done
prompt: 202607/prompts/gate_primary_footer.md
---

# Plan: Gate primary action footer

## Context and outcome

The branch-driven plan and custom gate modals currently describe their primary-submit binding as “Submit primary.” That
wording exposes the gate model instead of telling the reviewer what will happen, and the hint has no stronger visual
hierarchy than secondary actions. The plan-review screenshot makes the consequence clear: the primary Tale button is
visually prominent in the panel, but its matching `Enter` shortcut is easy to miss and requires the user to translate
“primary” back to “Tale.”

Replace that implementation-oriented description with the declared primary branch's display name and give the whole
key/action pair a restrained call-to-action treatment. A tale plan should read `Enter=Tale`; an epic plan should read
`Enter=Epic`; a custom singleton or group should use its configured option or group label. This is a footer-only
presentation change: the primary branch, current selection, feedback behavior, configurable keymaps, and response
protocol must remain unchanged.

## Product design

### One label rule for every branch-driven gate

Derive the footer name from the same immutable `GateBranchData` that renders and submits the controls, rather than from
plan-tier conditionals or hard-coded modal copy:

| Primary branch shape | Footer label source            | Example                       |
| -------------------- | ------------------------------ | ----------------------------- |
| One option           | That `GateOption.label`        | `Epic`, `Approve deployment`  |
| AND group            | The matching `GateGroup.label` | `Tale`, `Deploy signed build` |

Use label text only, without the option/group icon: the result should scan as a keymap (`Enter=Tale`), while the nearby
button remains the richer icon-plus-label representation. Keep the action ID `submit_primary`, keymap configuration, and
hidden binding metadata stable so existing configuration and dispatch behavior do not change. Specialized gates that
already advertise concrete actions such as `Enter/a=Approve` are outside this change.

The shared label resolver must be deterministic for both verified envelopes and the legacy/directly constructed plan
data used by callers and tests. If an AND group lacks an explicit label in a defensive construction path, fall back to
the first option's label rather than displaying `None` or generic “primary” copy. Gate validation still owns malformed
branch/group rejection.

### A clear but quiet primary-action badge

Render the configured primary-submit key and resolved label together as a padded, bold, high-contrast Rich `Text` badge
using an established ACE keycap accent (dark text on the mint/green interaction color). This creates a visible link to
the green primary control without adding animation or relying on color alone. Preserve the existing muted hierarchy and
ordering of navigation, toggle, alternate submit, debug/edit/copy, cancel, and scroll hints around it. Both
`PlanApprovalModal` and `CustomGateModal` should consume the same pure badge/label formatter so wording, escaping, and
emphasis cannot drift.

Build the badge from literal `Text` spans, not interpolated Rich markup, because custom gate labels are external display
data and may legitimately contain brackets or other markup-like characters. Use the effective configured key via
`key_display_name`, so remapping primary submit changes the hint and the binding together. Bound only the footer copy of
unusually long labels with terminal-cell-aware ellipsis so the badge remains a compact, unbroken affordance at normal
and narrow widths; the complete label remains visible on its gate button. Keep footer containers auto-height so the
surrounding secondary hints may wrap naturally without clipping.

## Implementation shape

1. Add a small shared presentation helper alongside the branch-driven gate modal code. It should resolve the primary
   branch's display label and construct the styled, cell-safe primary key/action badge without I/O, widget queries, or
   render-time mutation. This keeps the keystroke and compose paths synchronous and constant-time.
2. Refactor the plan-review footer in `src/sase/ace/tui/modals/plan_approval_modal.py` to compose Rich `Text`, insert
   the shared badge where `Submit primary` appears today, and preserve all other effective-key hints and their current
   visual semantics.
3. Refactor the generic custom-gate footer in `src/sase/ace/tui/modals/custom_gate_modal.py` to use the same badge and
   label contract while retaining its secondary hints. Do not alter `GateBranchControls.submit_primary_branch`, input
   submission behavior, bindings, gate schemas, or default keymap configuration.
4. Accept the intentional visual changes in the existing tale, epic, singleton custom, grouped custom, and
   required-feedback gate PNG snapshots only after inspecting the rendered actuals and diffs for alignment, contrast,
   wrapping, and consistency with the primary button.

## Reliability and verification

Add focused unit coverage for label resolution and footer rendering:

- tale and epic plans produce `Tale` and `Epic`, with no remaining user-facing “Submit primary” copy;
- custom singleton and AND-group gates use the option and group labels respectively, including a defensive unlabeled
  group fallback;
- custom configured primary-submit keys are displayed instead of hard-coded `Enter`;
- bracketed/markup-like and wide Unicode labels render literally, and an overlong label is truncated by display cells
  with an ellipsis rather than breaking the footer layout;
- the emphasized span includes both the key and action label, so its hierarchy is not color-only and secondary hints
  remain visually secondary.

Keep or extend the existing interaction tests proving that primary submit still resolves the declared primary branch
even when focus is elsewhere, that group toggles preserve their current selected subset, that alternate submit still
uses the active branch, and that required feedback still blocks/submits as before. Include at least one remapped-key
interaction assertion to couple displayed affordance and actual dispatch.

Run `just install` before repository checks, then run the focused plan/custom gate modal and keymap tests. Regenerate
and inspect only the affected gate PNG goldens with the visual snapshot update flag, rerun `just test-visual` without
update mode to prove stability, and finish with the repository-required `just check`.

## Acceptance criteria

- The screenshot scenario displays an emphasized `Enter=Tale` footer badge in place of `Enter=Submit primary`.
- Every branch-driven plan/custom gate derives the badge label from its declared primary control and the badge key from
  its effective keymap; internal action names never leak into the visible footer.
- The treatment remains readable with long, Unicode, and markup-like labels and at narrower terminal widths.
- Mouse/button behavior, keyboard submission semantics, feedback handling, gate responses, and configuration remain
  backward compatible.
- Updated PNG snapshots demonstrate a balanced footer hierarchy, and all targeted tests plus `just check` pass.
