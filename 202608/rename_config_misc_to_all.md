---
tier: tale
title: Rename the Admin Center Config Misc sub-tab to All
goal:
  Display the Config catch-all sub-tab as All in correct alphabetical order while
  preserving its stable navigation identity and behavior.
size: small
proposed_by: bbugyi200.athena.0cu
create_time: 2026-08-24 14:38:02
status: wip
---

# Plan

## Outcome and compatibility contract

Rename the user-facing **Misc** child of **SASE Admin Center > Config** to **All** and
make the numbered Config catalog alphabetic by displayed label. The expected order is:

- With `admin_center_flags` enabled: **01 All**, **02 Flags**, **03 Glossary**, **04
  Launch**, **05 Memory**, **06 Snippets**, **07 XPrompts**.
- With `admin_center_flags` disabled: **01 All**, **02 Glossary**, **03 Launch**, **04
  Memory**, **05 Snippets**, **06 XPrompts**.

Keep `misc` as the internal `ConfigSubTab`/widget/session identity. Direct-entry
requests, lazy pane caching, and selection-resume bookmarks already use that stable
identity and do not need a migration for a display-only rename. Do not alter the
underlying `ConfigPane` behavior or add work to the TUI navigation path.

## Implementation

1. Update the Config catalog metadata in `src/sase/ace/tui/modals/config_hub_catalog.py`
   so the existing `misc` spec is first and its full, compact, and micro labels are
   `All`. Keep its descriptions, factory behavior, and `id="misc"` intact. Keep the
   immutable spec tuple and its derived maps/order aligned so rendering, click targets,
   and numeric shortcuts all consume the same catalog order.
2. Update `src/sase/ace/tui/modals/config_hub_session.py` to put `misc` first in the
   canonical order. Replace the current positional `CONFIG_SUBTAB_ORDER[1:]` Flags-off
   derivation with an explicit omission of `flags`; after `All` moves ahead of `Flags`,
   slicing would incorrectly remove `All` and leave the disabled Flags entry selectable.
3. Recalculate the full/compact/micro strip-width expectations in
   `src/sase/ace/tui/modals/config_hub_pane.py` for the shorter `All` labels, and update
   its explanatory comments plus the Config catalog summary in
   `src/sase/ace/tui/modals/config_center_modal.py`. Preserve the existing lazy
   switching, navigation lock, and event-loop behavior.
4. Update catalog and navigation coverage under `tests/ace/tui/` to assert both exact
   orders, `All` display variants, stable `misc` identity, and correct numeric targets
   with Flags on and off. In particular, adjust tests that currently encode the old
   `01 Flags`/`02 Glossary`/`03 Launch`/`04 Memory` positions, wraparound from XPrompts,
   and the Flags-off first child. Retain regression coverage that bare digits belong to
   embedded children unless the Config prefix is armed, invalid numbers cancel cleanly,
   child panes remain cached, and resume/direct entry still restore `misc` by identity.
5. Update the Admin Center catalog and keyboard-navigation documentation in
   `docs/configuration.md` and `docs/ace.md`, including the Flags pane's new **02**
   position and both enabled/disabled numbered sequences. Sweep the repository's Admin
   Center sources, tests, and docs for stale user-facing `Misc` copy or old
   number-to-child mappings; leave unrelated command-category or type-checking uses of
   “Misc” unchanged.
6. Regenerate only the affected ACE PNG goldens for Config-hub surfaces in the Config,
   config-edit, Launch, and Flags visual test modules. Inspect the produced diffs to
   ensure changes are limited to the renamed/reordered strip and its expected spacing or
   active-tab placement, including the narrow compact/micro layouts; do not accept
   unrelated visual drift.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run focused non-visual tests for the Config catalog, Config-hub pane/navigation,
   numbered embedded links, feature-flag catalog variants, and selection resume. These
   must demonstrate the exact enabled and disabled sequences, the new shortcuts, the
   stable `misc` identity, and unchanged lazy/cache behavior.
3. Run the affected visual modules once with `--sase-update-visual-snapshots`, review
   their actual/expected/diff artifacts, then rerun them without the update flag to
   prove the checked-in goldens pass exactly.
4. Run `just check` for the required whole-repository lint gates and diff-scoped test
   lane. If its selector escalates or reports unusual selection, follow project guidance
   and run `just check-full` through `/sase_monitor` rather than inline.

## Out of scope

- Renaming the internal `misc` subtab ID, widget ID, factory contract, or persisted
  bookmark representation.
- Changing the set of configuration fields shown by `ConfigPane`.
- Removing or otherwise changing the `admin_center_flags` sunset flag.
