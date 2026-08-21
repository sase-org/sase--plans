---
tier: tale
title: Prefix Glossary and Memory numeric link shortcuts
goal: "Glossary and Memory links are followed with >1 through >9, leaving bare digits
  available for SASE Admin Center tab navigation without making the link shortcuts
  ambiguous or undiscoverable.

  "
size: small
proposed_by: bbugyi200.athena.sase-ri.land.w2.f2.w2
create_time: 2026-08-21 09:53:07
status: wip
---

# Plan: Prefix Glossary and Memory numeric link shortcuts

## Context and decisions

`ConfigCenterModal` owns bare `0`-`9` bindings for its numbered top-level tabs, while
`GlossaryPane` and `MemoryPane` currently install their own bare `1`-`9` bindings for
following numbered relation chips. The panes are reused both as standalone modals and as
Config children, so the overlapping bindings create an order-dependent conflict in the
embedded Admin Center surface.

Make `>` a fixed, one-shot prefix for only the Glossary and Memory numbered-link
shortcuts: `>1` follows chip 1 through `>9` for chip 9. This is a sequential prefix, not
a simultaneous key chord. Bare digits must no longer follow links in these two panes;
when embedded, they remain available to the Admin Center's top-level tab bindings.
Preserve Config's existing configurable `0`-then-number sub-tab selector, and leave
Snippets' separate bare-number relation behavior unchanged.

The new prefix remains a fixed pane affordance, like the existing fixed close and list
navigation keys. Do not add a new user-configurable keymap field, default-config entry,
or schema property. The existing configurable `follow_relation` and `follow_link`
actions continue to follow the currently focused chip and are otherwise unchanged.

## Implementation

1. Replace the direct Glossary and Memory `1`-`9` bindings with a shared, lightweight
   one-shot numbered-link dispatcher used by both panes. Bind `greater_than_sign` to arm
   it, then resolve the next decimal digit synchronously on the UI thread and call the
   panes' existing `action_follow_relation_number` or `action_follow_link_number` paths.
   Keep the current Config-hub forwarding first so an already-armed Config sub-tab
   prefix still wins. A repeated `>` stays armed; a digit outside `1`-`9` is consumed
   and cancels without following or switching tabs; any other key cancels and continues
   through normal dispatch so `q`, `Esc`, `/`, and other pane actions retain their
   meanings. Clear pending state when a pane is hidden or unmounted, and do not arm or
   consume the prefix while a filter `Input` has focus, where `>` and digits must remain
   ordinary filter text. Keep this keystroke path state-only: no I/O, worker launches,
   or full-pane rebuilds.

2. Make the visible affordances describe the actual sequence. Extend the shared
   numbered-chip renderer with an explicit optional shortcut prefix, pass `>` from the
   Glossary `SEE ALSO` / `REFERENCED BY` and Memory `PARENT` / `CHILDREN` card builders,
   and retain the empty default for callers such as Snippets and the compact glossary
   preview. Render chips as `>1 name`, `>2 name`, and so on while preserving continuous
   numbering and focused-chip styling. Advertise `>1-9` in both global and panel-scoped
   help, and revise Memory's Enter caveat to name the new sequence. Update conditional
   footers only if needed to keep the shortcut discoverable without duplicating or
   obscuring the prefixed chip labels.

3. Update the user guide and configuration reference to say that Glossary and Memory
   numbered links use `>` followed by `1`-`9`, that bare digits remain Admin Center tab
   selectors on the embedded sub-tabs, and that this prefix is one of the panels' fixed
   keys. Do not change the documented behavior of the standalone glossary preview or the
   Snippets panel, which continue to use their existing bare numeric shortcuts.

## Regression coverage

- Update the focused Glossary and Memory chip tests to press `>` then a digit, assert
  the `>N` labels, and prove a bare digit no longer follows a link. Cover repeated
  prefix, invalid digit, non-digit cancellation/passthrough, filter-input text, and
  pending-state cleanup through shared tests or equivalent pane-specific cases.
- Add an Admin Center integration regression for each real Config child: a bare link
  digit must select the corresponding top-level Admin Center tab without changing the
  selected term/note, while returning to Config and pressing `>N` must follow that link
  without leaving Config. Also retain coverage that `0N` selects Config sub-tabs and
  that Snippets keeps its current local numeric behavior.
- Update pure rendering and help assertions for `>1`-style chips and `>1-9` help text.
  Adjust the populated Glossary and Memory visual tests to exercise `>1`, regenerate
  only their affected dark/light PNG goldens, and inspect the actual/diff images to
  confirm that the prefix is legible and layout remains stable.

## Verification

Run `just install` before repository checks. Run the focused non-visual tests for the
Config hub, Glossary and Memory chip interactions, render helpers, and keybinding help;
then run the affected visual snapshot tests with intentional golden updates and rerun
them without the update flag. Finish with `just check`. Escalate to `just check-full`
through `/sase_monitor` only if scoped selection broadens or reports an unusual
selection, as required by the repository verification policy.
