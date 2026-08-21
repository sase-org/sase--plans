---
tier: tale
title: Use a dot prefix for Glossary and Memory numbered links
goal:
  Glossary and Memory links use .1 through .9 without breaking filtering or Admin Center
  navigation.
size: small
proposed_by: bbugyi200.athena.sase-ri.land.w2.f2.w2.f1
create_time: 2026-08-21 10:38:19
status: wip
---

# Plan: Use a dot prefix for Glossary and Memory numbered links

## Goal

Make the fixed one-shot numbered-link shortcuts in the Glossary and Memory panes use
`.1` through `.9` instead of `>1` through `>9`, while preserving Admin Center tab
selection, filter-body toggling, editable filter text, and the intentionally different
Snippets behavior.

## Context and decisions

The recently added shared numbered-link dispatcher gives `GlossaryPane` and `MemoryPane`
a fixed `>` prefix so their bare digits no longer compete with the SASE Admin Center's
top-level numeric tab bindings. The same panes already use `.` as the default
configurable action for extending a term/note filter into definition or body text.
Simply changing the dispatcher constant would therefore make that filter action
unreachable.

Resolve the collision by swapping those two defaults only in Glossary and Memory: `.`
becomes the fixed numbered-link prefix, and `>` becomes the default configurable
`toggle_definition_filter` / `toggle_body_filter` key. This keeps the more frequent
direct-link gesture on the easier unshifted key without removing the filter feature.
Keep the numbered-link prefix fixed rather than adding a new configuration field, as in
the approved predecessor design. Reserve `full_stop` from configurable Glossary and
Memory actions so an explicit old/default-style override cannot silently shadow or be
shadowed by the fixed prefix; warn and fall back to that action's effective default. The
freed `greater_than_sign` remains a valid configurable key.

Leave Snippets unchanged: its relation chips continue to show and accept bare `1`-`9`,
and its body-filter toggle remains `.`. Also leave the compact glossary preview's bare
numeric shortcuts unchanged. Bare digits in the embedded Glossary and Memory panes must
continue selecting top-level Admin Center tabs, and Config's existing configurable
`0`-then-number sub-tab selector must continue to take precedence when armed.

## Implementation

1. Update the shared numbered-link key module to bind and recognize Textual's
   `full_stop` / `.` spelling and to render `.` before numbered Glossary and Memory
   chips. Preserve the existing one-shot state machine: a repeated `.` stays armed,
   `1`-`9` follows through the panes' existing numbered actions, `0` is consumed and
   cancels, and another key cancels then continues through normal dispatch. Keep Config
   sub-tab forwarding ahead of this dispatcher, clear pending state on hide/unmount, and
   keep the path synchronous and state-only.

2. Preserve text editing and the displaced filter action. When a Glossary or Memory
   filter `Input` has focus, `.` and digits remain ordinary query text and never arm or
   follow a link; `>` remains ordinary text there as well. Outside the input, change the
   bundled and dataclass defaults for `toggle_definition_filter` and
   `toggle_body_filter` to `greater_than_sign`. Extend the focused-scope keymap loader
   with an explicit reserved-key check for Glossary and Memory so a user override that
   assigns `full_stop` to any configurable action is diagnosed and reverted to its new
   default. Do not change Snippets defaults or introduce a schema field for the fixed
   prefix.

3. Update every user-facing affordance to match the new sequence: render `.1 name`,
   `.2 name`, and so on in Glossary `SEE ALSO` / `REFERENCED BY` and Memory `PARENT` /
   `CHILDREN` rows; advertise `.1-9` in global and panel-scoped help; update Memory's
   Enter caveat; and make help show `>` for the two remapped filter-body toggles. Update
   the ACE guide, remapping examples, and configuration reference to describe the key
   swap, the fixed/reserved `.` prefix, bare-digit Admin Center behavior, and unchanged
   Snippets/compact-preview behavior.

## Regression coverage

- Update the shared dispatcher tests to exercise `.` by both canonical key name and
  printable character where useful: repeat-prefix, valid and invalid digits, non-digit
  cancellation/passthrough, filter-input literal text, and pending-state cleanup. Add an
  assertion that `>` no longer arms the dispatcher.
- Update Glossary and Memory interaction tests to follow `.N`, render `.N` labels, and
  prove bare digits still do not follow. Update the filter-rail tests to prove the new
  default `>` key still toggles definition/body matching, while `.N` remains literal
  inside the active input.
- Add keymap-loading coverage for the new defaults and the reserved `full_stop`
  fallback/warning, including a non-conflicting custom toggle binding. Update pure
  help/rendering assertions for `.1-9` and the displayed `>` filter key.
- Update both real Admin Center child regressions so a bare digit still switches a
  top-level tab, returning to Config and pressing `.N` follows without leaving Config,
  and `0N` sub-tab selection still wins when armed. Retain the Snippets regression for
  its bare-number behavior.
- Update the populated Glossary and Memory visual scenarios to navigate with `.1`,
  regenerate only their affected dark/light PNG goldens, and inspect actual/diff
  artifacts to confirm the new prefix is legible and layout remains stable.

## Verification

Run `just install` before repository checks. Run focused non-visual tests for the
numbered-link dispatcher, Glossary and Memory chip/filter interactions, Config-hub
integration, keymap defaults/loading/help, and render helpers. Run the affected visual
snapshot tests with intentional golden updates, inspect the four changed populated
Glossary/Memory images, and rerun those tests without the update flag. Finish with
`just check`; if scoped selection broadens or reports an unusual selection, run
`just check-full` only through `/sase_monitor` as required by repository policy.
