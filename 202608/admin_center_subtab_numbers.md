---
tier: tale
size: medium
title: Add numbered navigation to Admin Center Config sub-tabs
goal: "The Admin Center Config catalog is alphabetized and every Config child is
  reachable through a scoped numeric prefix chord, while Config and Statistics show the
  complete default 01-style shortcuts without regressing nested input, relation,
  lazy-loading, session, busy-write, or responsive-layout behavior.

  "
proposed_by: bbugyi200.athena.sase-ri.land.w2.f3
create_time: 2026-08-21 09:04:03
status: wip
---

# Plan: Add numbered navigation to Admin Center Config sub-tabs

## Outcome and interaction contract

Sort the Config sub-tabs by their displayed titles and make that one canonical order for
the strip, bracket cycling, click targets, and numeric selection:

| Shortcut | Config sub-tab |
| -------- | -------------- |
| `01`     | Glossary       |
| `02`     | Launch         |
| `03`     | Memory         |
| `04`     | Misc           |
| `05`     | Snippets       |
| `06`     | XPrompts       |

The first key is a Config-scoped, configurable `select_subtab` prefix whose bundled
default is `0`; the second key is the stable one-based position in the alphabetized
catalog. Mirror the existing Statistics interaction: the prefix arms one numeric
selection, a valid digit selects the corresponding child, a repeated prefix remains
armed, an out-of-range digit safely cancels, and any other key cancels the pending
selection before continuing through its normal binding path. A configured replacement
prefix must drive the same behavior, even though the compact ordinal badges continue to
show the requested canonical defaults (`01` through `06`) just as Statistics' strip is a
compact reminder of its default `01` through `08` chords; effective remaps remain
documented through the scoped keymap configuration and Statistics' existing live hints.

Preserve bare-digit behavior. Without an armed Config prefix, digits continue to reach
the active child (including the numbered relationship links in Glossary, Memory, and
Snippets) or the enclosing Admin Center's top-level `1`-`6` navigation. When a filter or
other text input owns focus, typing `0` and digits remains text and must not arm or
resolve Config navigation. Nested modal editors remain isolated on their own screen.

Alphabetizing changes ordinal and bracket order, not identity semantics. An unseeded
first visit to Config continues to open XPrompts; a remembered child and explicit
Glossary, Memory, Snippets, Launch, or XPrompts entry seed still win by stable ID.
Successful numeric selection updates the existing session bookmark and uses the same
lazy, cached, failure-safe switch path as brackets and clicks. A child that refuses
deactivation during a write keeps its focus, strip selection, bookmark, and mounted-pane
set unchanged.

Statistics keeps its existing view order and `0`-then-`1`-`8` dispatch. Change only its
visible numeric affordances from `1`-`8` to `01`-`08`, including the view list in
contextual help where it identifies those tabs. Do not prefix the Admin Center's six
top-level tab numbers; this request applies only to the nested Config and Statistics
strips.

## Keymap and navigation implementation

- Add a focused `ConfigHubKeymaps` registry scope with `select_subtab: "0"`, following
  the Statistics scope's dataclass, bundled-default loader, binding metadata/builder,
  registry construction, public exports, override validation, and fail-soft fallback
  conventions. Expose it as `ace.keymaps.config.select_subtab` in `default_config.yml`
  and the JSON schema. Pass the resolved scope from the Admin Center Config factory into
  `ConfigHubPane`; constructing or dispatching it must not load a catalog child or
  perform data-scaled work.
- Give `ConfigHubPane` an instance-local binding and one pending-selection flag. Resolve
  the second digit from the pane's canonical `_subtab_order`, then call the existing
  `_schedule_switch` / `_switch_to` path rather than adding a second mounting or state
  path. Reset pending state after selection, cancellation, deactivation, and teardown so
  a prefix cannot leak across top-level tabs or modal lifetimes.
- Route the pending second digit ahead of child and Admin Center digit bindings, while
  leaving those bindings untouched when no prefix is armed. Explicitly protect embedded
  filter inputs and existing child numeric relationship actions in interaction tests;
  extend the small Config-hub key helper only if Textual event ownership requires a
  cooperative forwarding seam.
- Reorder the typed Config catalog to `glossary`, `launch`, `memory`, `misc`,
  `snippets`, `xprompts`. Keep validation, spec lookup, rendered tabs, numeric index
  lookup, bracket cycling, and tests derived from the same order tuple so labels and
  actions cannot drift. Preserve `ConfigSubTab` identities and session fields; no
  persisted-state migration is needed.
- Populate each Config `PanelTab.shortcut` with its complete default chord and enable
  numbered rendering on the nested strip. Populate Statistics' existing tab shortcuts
  with `01` through `08`. Use the existing `PanelTabStrip` shortcut and cell-accurate
  click-range support rather than special-casing text in render handlers.

This is presentation-only Textual navigation and Python keymap plumbing. It does not
need a Rust-core change. Implement it atomically as one ready tale; no temporary feature
flag or compatibility branch is warranted because no incomplete behavior lands between
phases.

## Responsive visual design

The extra prefix cell must look intentional rather than crowding either rail. Recompute
the full/compact/micro thresholds from the rendered cell widths after the new shortcuts
and alphabetic labels are applied. At each tier, keep every ordinal attached to its
title, use the existing accent treatment for the active chord/title, center the rail,
and retain correct mouse hit ranges. It is acceptable—and preferable—for Statistics to
use its compact labels at 120 columns once `01`-style chords make the full eight-view
line too wide; no tier may clip or silently drop a tab.

Cover the production widths already represented by the Admin Center visual suite,
especially 120-column Statistics and Config views, 90-column Statistics views, and the
70-column embedded Launch/Config layout. Update the relevant Config and Statistics PNG
goldens only after inspecting that their diffs are limited to the intended ordering,
numeric chords, active-tab movement, and deliberate responsive-tier changes.

## Documentation and compatibility

Update the Admin Center and keymap documentation in `docs/configuration.md`,
`docs/ace.md`, and `docs/telemetry.md` where they enumerate Config children or explain
Statistics number selection. The Config list must include Launch in alphabetical order,
document `ace.keymaps.config.select_subtab`, and describe the `01`-`06` default chords.
Statistics prose and help should describe/display `01`-`08` while still explaining the
configurable prefix-plus-digit model. Keep existing Config `[` / `]`, direct `,m` Launch
entry, top-level `1`-`6`, `#` history, and child-local bindings unchanged.

## Tests and verification

Add or update focused coverage for:

- registry/default/schema loading of `ace.keymaps.config.select_subtab`, including a
  custom valid prefix, invalid values, and unknown actions;
- the exact alphabetic catalog and its `01`-`06` tab metadata, plus forward/reverse
  bracket wrap in the new order;
- `01`, representative middle chords, and `06`; repeated prefix, out-of-range digit,
  non-digit cancellation, a remapped prefix, and reset on top-level deactivation;
- bare top-level digits and child-local relation digits without a prefix, plus literal
  numeric typing in each embedded filter-input pattern;
- lazy construction of only the selected child, cache reuse, direct-entry and remembered
  identity precedence, mount failure rollback, and busy-child refusal under numeric
  navigation;
- Statistics `01`-`08` labels and help copy without changing `VIEW_ORDER` or existing
  selection behavior; and
- full, compact, and micro line widths and cell-accurate click ranges for both nested
  strips at their supported terminal widths.

Run `just install` before repository verification. Iterate with the Config-hub catalog,
pane, keymap registry/default/schema, Statistics number-selection/help/layout, and panel
tab-strip tests. Then run `just test-visual`, inspect and accept only intentional PNG
changes, rerun it cleanly, and finish with the repository-required `just check`.

## Acceptance criteria

- Config displays and cycles through Glossary, Launch, Memory, Misc, Snippets, and
  XPrompts in alphabetical order, labeled `01` through `06`.
- With defaults, `01`-`06` select those Config children; a configured prefix replaces
  the first key without changing the ordinal mapping.
- Statistics retains its eight existing views and navigation but displays complete
  `01`-`08` shortcuts on the strip and in its view help.
- Unprefixed digits, text entry, relation-link selection, direct entry, remembered
  selection, busy-write guards, lazy mounting, cache reuse, failure rollback, and
  top-level Admin Center navigation retain their established behavior.
- Both nested strips remain centered, clickable, readable, and unclipped at normal and
  narrow production sizes; focused tests, updated visual snapshots, `just test-visual`,
  and `just check` pass.
