---
tier: tale
title: Two-pane gate review workbench
goal: 'Branch-driven gate modals become a near-fullscreen review workbench: decision
  controls live in a left actions pane, the attached document fills a large right
  pane, and the layout degrades gracefully on narrow terminals and document-less gates
  — without changing any gate behavior.

  '
create_time: 2026-07-18 10:40:52
status: wip
prompt: 202607/prompts/gate_review_workbench.md
---

# Plan: Two-pane gate review workbench

## Context and outcome

`PlanApprovalModal` and `CustomGateModal` currently stack title, document, branch controls, and footer vertically inside
a centered box (90% wide, capped at 150 cells). The consequence is visible in the existing PNG goldens: the document
under review — the reviewer's primary object of attention — gets a minority of the vertical space, while the controls
region reserves a tall middle band full of empty gaps, and singleton buttons float in a sparse horizontal row. The
reviewer scrolls a cramped viewport to read a plan while most of the screen shows chrome.

Rebuild both branch-driven gate modals as a near-fullscreen, two-pane review workbench: decision controls in a
fixed-width left pane, the attached document in a large right pane, a slim full-width header, and the existing footer
hints. This is a presentation-only change. The gate model, response protocol, keymaps, bindings, focus/dispatch
behavior, feedback modes, and the primary-action footer badge shipped in the previous change must remain exactly as they
are. Specialized gates (launch approval, user question, workflow HITL, approve-options) are out of scope, though the
shared shell styling should be written so they can adopt it later.

## Product design

### Shell: almost the whole screen, one clear hierarchy

Both modals grow to approximately 98% of the screen in each dimension with no fixed max-width cap, keeping their
existing border-color identities (`$primary` double border for plan review, `$accent` for custom gates). The shell is:

| Region        | Placement          | Contents                                                                                                         |
| ------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Header        | full width, slim   | Modal identity: `Plan Review`/`Epic Review` plus provider badge, or the custom gate icon, sender, and request id |
| Actions pane  | left, ~42 cells    | Context, section titles, stacked gate controls, feedback input, Cancel                                           |
| Document pane | right, remaining   | Attached file content with the file name as its border title                                                     |
| Footer        | full width, bottom | Existing key hints, including the primary-action badge, unchanged                                                |

The plan file basename moves out of the header and becomes the document pane's border title (for custom gates, the
`preview_name`), so the file identity sits directly on the content it names instead of being duplicated.

### Actions pane: a vertical decision column

Gate controls render as a vertical stack of full-width buttons instead of a centered horizontal row: primary control
first with its existing success styling, AND-group toggles and group submit in their current order, remaining singletons
below, then the feedback label/input, then a muted full-width Cancel at the bottom. `CustomGateModal`'s existing Cancel
button moves into this slot; `PlanApprovalModal` gains one with identical dismiss-without-response semantics (mouse
parity with the existing `Esc`/`q` bindings). The custom gate's notes and attachment list render above the controls in
this pane under quiet section titles, replacing today's standalone “Choose one resolution branch” label. Every control
must remain reachable and scrollable at small terminal heights without nested double scrollbars; the implementation may
choose whether the pane or the controls widget owns the scroll region.

The stacking should be achieved as a styling variant of `GateBranchControls` (for example a modifier class set by the
composing modal plus CSS that switches the singleton row to vertical layout and full-width buttons), not a second
compose path: widget order, control ids, focus traversal, toggle behavior, and submit dispatch stay byte-identical.

### Document pane: the review surface

The right pane is the dominant region and holds the existing syntax-highlighted document rendering (markdown theme, word
wrap) inside the current scrollable containers. Keep the existing scroll-region ids (`#plan-approval-scroll`,
`#custom-gate-review-scroll`) so the half-page/top/bottom scroll actions and their tests keep working unmodified.
Content loading behavior is unchanged: callers continue to preload plan/preview text off-thread, and the existing
direct-construction fallback read stays as-is. Compose remains synchronous and free of new I/O, timers, or render-path
work.

### Graceful degradation

Two situations must not produce a giant empty pane:

- **No document.** A custom gate with `preview_text is None` keeps a compact, centered single-column layout — the same
  actions-column composition at roughly today's dimensions — rather than a fullscreen shell with a hollow right pane.
  Plan gates always have a document and always use the two-pane shell.
- **Narrow terminals.** Below roughly 100 columns, the panes stack vertically (document above actions) using Textual's
  screen-level `HORIZONTAL_BREAKPOINTS` so the switch is automatic on resize with no manual resize handlers. The
  120-column visual test size must render the two-pane layout.

## Implementation shape

1. Restructure `compose()` in `src/sase/ace/tui/modals/plan_approval_modal.py` and
   `src/sase/ace/tui/modals/custom_gate_modal.py` into the shared shell: header, horizontal body with actions and
   document panes, footer. Express the shell with shared CSS classes in `src/sase/ace/tui/styles.tcss` used by both
   modals, so spacing, borders, and pane widths cannot drift between them.
2. Add the vertical styling variant to `GateBranchControls` (a `classes` passthrough on the constructor plus CSS) and
   restyle its singleton/group/feedback children for the pane context. No behavioral edits to `gate_branch_controls.py`
   logic.
3. Add the narrow-terminal breakpoints to both modal screens and the stacked-layout CSS they trigger, plus the compact
   no-preview variant for `CustomGateModal`.
4. Update the five existing gate PNG goldens and add two new scenarios: a compact no-preview custom gate and a
   narrow-terminal stacked plan gate. Accept goldens only after inspecting actuals and diffs for alignment, contrast,
   and balance.

## Reliability and verification

The behavioral contract is frozen: all existing interaction tests for primary submit, group toggles, alternate submit,
required feedback, remapped keys, and scroll actions must pass without semantic modification. Add focused coverage for
what is new:

- both modals compose the two-pane shell when a document is present, with the document scroll region under its existing
  id and the file name as the pane's border title;
- a preview-less custom gate composes the compact variant instead of the two-pane shell;
- the narrow breakpoint applies the stacked layout class at a sub-threshold size and the wide layout at 120 columns;
- the plan modal's new Cancel button dismisses with no result, matching `Esc`/`q`;
- keyboard navigation still cycles every control in the vertical stack, and the feedback input still appears and gates
  submission in the actions pane.

Visual verification: regenerate the affected gate goldens with the update flag, inspect every actual and diff, then
rerun `just test-visual` without update mode to prove stability. Run `just install` before repository checks and finish
with the repository-required `just check`.

## Acceptance criteria

- Both gate modals occupy almost the entire screen with decision controls in a left pane and the attached document
  filling a large right pane; the document viewport is dramatically larger than today at 120x40.
- The document pane carries the file name as its border title; the header keeps modal identity and badges without
  duplicating the file name.
- Gate semantics are unchanged: same bindings, same dispatch, same response payloads, same feedback gating, same
  primary-action footer badge.
- A preview-less custom gate renders the compact variant, and sub-100-column terminals get the stacked layout with every
  control reachable.
- Updated and new PNG goldens demonstrate a balanced, aligned, high-contrast workbench, and all targeted tests plus
  `just check` pass.
