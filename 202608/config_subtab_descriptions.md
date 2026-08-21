---
tier: tale
title: Config sub-tab description rail
goal:
  Give every SASE Admin Center Config sub-tab an accurate, responsive, and visually
  polished orientation caption without reducing pane space or weakening navigation
  reliability.
size: medium
proposed_by: bbugyi200.athena.0a2
create_time: 2026-08-21 19:10:51
status: wip
---

# Plan: Config sub-tab description rail

## Outcome

Add a catalog-driven description rail directly beneath the nested Config tab strip. The
rail will always explain the currently selected child in concise, action-oriented copy,
will switch atomically with the child and selected tab, and will substitute a crafted
compact sentence on narrow terminals. It will occupy the row that is currently only the
tab strip's bottom margin, preserving every child pane's existing height.

This is a presentation-only enhancement in the Python/Textual frontend. It does not
change configuration semantics, navigation bindings, child loading, session state, or
the Rust core boundary. It also does not need a new feature flag: the result is intended
to ship complete rather than as a temporary beta or compatibility branch. The existing
`admin_center_flags` flag will continue to control whether the Flags child and its
description participate in the active catalog.

## Copy and visual language

Keep both the full and compact copy in each immutable Config sub-tab spec so labels,
factories, ordering, and descriptions have one source of truth. Use this reviewed copy:

| Sub-tab  | Full description                                                           | Compact description                                    |
| -------- | -------------------------------------------------------------------------- | ------------------------------------------------------ |
| Flags    | Review feature rollouts, effective state, provenance, and saved overrides. | Control feature rollouts and saved overrides.          |
| Glossary | Browse shared terms, aliases, definitions, and their relationships.        | Browse shared terms, definitions, and relationships.   |
| Launch   | Tune model routing, reasoning effort, runner limits, and launch defaults.  | Tune model routing, effort, and launch limits.         |
| Memory   | Browse, edit, and publish the durable context agents receive.              | Manage the durable context agents receive.             |
| Misc     | Inspect effective values, source layers, and schema-backed settings.       | Inspect effective values, sources, and other settings. |
| Snippets | Build reusable prompt fragments and preview their composed output.         | Manage reusable prompt fragments and compositions.     |
| XPrompts | Browse, preview, create, and edit reusable agent prompts and workflows.    | Manage reusable agent prompts and workflows.           |

Render the chosen sentence centered as `› <description>` in the Config accent, matching
the established top-level Admin Center caption grammar while remaining visually
subordinate through its position inside the Config surface. Keep it to one non-focusable
row with no border or extra ornament. Use the full sentence whenever its measured cell
width fits, otherwise use the hand-authored compact sentence; retain ellipsis/clipping
only as a last-resort fallback at unusually small widths.

## Implementation

1. Extend the immutable sub-tab catalog in
   `src/sase/ace/tui/modals/config_hub_catalog.py` with required full and compact
   description fields, populate every registered child with the copy above, and expose
   the metadata only through the existing active-spec helpers. Do not create a parallel
   description map or resolve feature flags at import time.

2. Add the description `Static` to `ConfigHubPane` between `#config-hub-tabs` and
   `#config-hub-switcher`. Build its Rich `Text` from the active catalog spec and the
   existing Config accent; measure terminal cell width rather than Python character
   count when deciding between full and compact copy.

3. Treat the nested strip, description, and child switcher as one piece of navigation
   chrome. Initialize the caption from direct-entry/resume state, update it only when a
   child switch succeeds, and restore the previous caption during the existing rollback
   path if a switch fails. A resize may repaint the caption only when its full/compact
   variant changes; it must perform no disk access, subprocess work, child rebuild, or
   focus mutation. Preserve lazy child creation and cached panes exactly as they work
   today.

4. Update `src/sase/ace/tui/styles.tcss`: replace the nested tab strip's one-row bottom
   margin with the one-row description rail, center it, keep it non-wrapping, and bound
   overflow. Verify that the ContentSwitcher retains `1fr` and that Flags, Glossary,
   Launch, Memory, Misc, Snippets, and XPrompts do not lose usable content height at the
   standard or narrow fixture sizes.

## Tests and acceptance criteria

Update focused tests in `tests/ace/tui/test_config_hub_catalog.py` and
`tests/ace/tui/test_config_hub_pane.py` to prove:

- all active specs have the exact reviewed full/compact copy and the description order
  remains catalog-derived;
- the initial XPrompts state, remembered/direct entries, numbered selection, bracket
  cycling, and cached return navigation show the matching caption;
- a failed child mount/deactivation leaves the prior child, strip selection, session
  state, and caption unchanged;
- the full copy is selected when it fits, the compact copy is selected at narrow width,
  and resizing does not construct or reload a child;
- Flags contributes its caption when `admin_center_flags` is on and is absent—with the
  remaining catalog still correctly numbered and described—when the flag is off; and
- the rail is plain presentation: it cannot take focus and does not interfere with
  filters, numbered links, or Config sub-tab key handling.

Regenerate and inspect the affected PNG goldens rather than hand-editing them. Existing
Config/Misc and XPrompts snapshots should demonstrate the full-width rail; existing
Flags dark, light, and 70-column snapshots should demonstrate theme compatibility and
the compact rail. Add a dedicated visual case only if those fixtures do not exercise the
longest full sentence at its boundary. Accept the result only when the caption is
clearly associated with the nested strip, the two Admin Center hierarchy levels remain
distinct, no pane content is obscured, and no sentence is awkwardly clipped at the
standard 120x40 or narrow 70x32 sizes.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the focused catalog/pane tests while iterating.
3. Regenerate intentional Config and Flags PNG snapshot changes with the repository's
   `--sase-update-visual-snapshots` flow, then rerun those visual modules normally and
   inspect the resulting images/diffs.
4. Run `just check`. If selection broadens/escalates or reports unusual coverage, run
   `just check-full` through `/sase_monitor` as required by repository policy.
