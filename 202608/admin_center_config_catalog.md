---
tier: epic
status: done
title: Consolidate configuration tools in the SASE Admin Center
goal: "The Admin Center Config section becomes a polished, lazy catalog for XPrompts,
  Snippets, Glossary, Memory, and miscellaneous layered settings, while prompt shortcuts
  continue to open the right content with their contextual selection intact.

  "
phases:
  - id: glossary_pane
    title: Extract a reusable Glossary content pane
    depends_on: []
    size: medium
    description:
      "glossary_pane: separate Glossary content and lifecycle behavior from its
      standalone modal host without changing current user behavior."
  - id: memory_pane
    title: Extract a reusable Memory content pane
    depends_on: []
    size: medium
    description:
      "memory_pane: separate Memory content and lifecycle behavior from its standalone
      modal host without changing current user behavior."
  - id: snippets_pane
    title: Extract a reusable Snippets content pane
    depends_on: []
    size: medium
    description:
      "snippets_pane: separate Snippets content and lifecycle behavior from its
      standalone modal host without changing current user behavior."
  - id: config_hub
    title: Build and integrate the nested Config catalog
    depends_on:
      - glossary_pane
      - memory_pane
      - snippets_pane
    size: medium
    description:
      "config_hub: add the lazy Config sub-tab host, move every requested surface into
      it, and route contextual prompt entry through a guarded integration path."
  - id: cutover
    title: Polish, verify, and make the consolidated experience unconditional
    depends_on:
      - config_hub
    size: medium
    description:
      "cutover: complete responsive visual and interaction coverage, remove the
      temporary old route, and verify the combined epic before landing."
proposed_by: bbugyi200.athena.sase-rd.land.w1
bead_id: sase-ri
create_time: 2026-09-09 19:49:42
---

- **PROMPT:**
  [prompts/202608/admin_center_config_catalog.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/admin_center_config_catalog.md)
- **BEAD:**
  [sase-ri](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ri/README.md)

# Plan: Consolidate configuration tools in the SASE Admin Center

## Outcome and interaction design

The Admin Center will have six main sections in its existing alphabetical order:
`Config`, `Logs`, `Procs`, `Projects`, `Statistics`, and `Updates`. The current
top-level `XPrompts` section disappears. Its selection bookmark and persisted `xprompts`
history value must migrate to Config rather than being discarded.

Config becomes a nested catalog with this order:

1. **XPrompts** — browse, preview, edit, add, and load reusable prompts and workflows.
2. **Snippets** — browse and maintain prompt snippets and their relationships.
3. **Glossary** — browse and maintain project terms and their relationships.
4. **Memory** — browse, maintain, and publish scoped SASE memory notes.
5. **Misc** — the current layered schema-driven Config browser, unchanged in capability.

This order intentionally puts the most frequently reusable authoring assets first and
the catch-all settings browser last. A normal first visit to Config opens XPrompts;
later visits resume the last Config sub-tab and each sub-tab's project/scope and row. An
explicit prompt shortcut always overrides that resume target for its initial open, then
participates in the same session bookkeeping.

The nested strip should reuse `PanelTabStrip`, the Config teal accent, compact labels,
and the visual language already established by Projects and Updates. It should read as
one hierarchy: the existing Admin Center strip remains the primary navigation, a quieter
Config strip sits immediately inside the content area, and the active tool uses the
remaining height without an extra modal border or duplicated title. Preserve each tool's
strong card/list layout, empty and diagnostic states, theme-aware accents, and
responsive behavior.

Use `]` and `[` for next/previous Config sub-tabs, matching other Admin Center nested
tabs, including forwarding from focused filter inputs. Preserve the configured Glossary,
Memory, and Snippets pane bindings—including `Tab`/`Shift+Tab` relationship navigation
and numbered relationship jumps—by making the Admin Center's priority bindings
context-aware rather than silently stealing child actions. Specify and test a single
precedence matrix for main-tab digits, main-tab cycling, Config-sub-tab cycling, filter
editing, relationship navigation, and `Escape`: an inline filter consumes its own Escape
first; otherwise Escape or `q` closes the entire Admin Center. Mouse clicks, the Admin
Center opener/alternate action, and the nested strip must stay coherent with keyboard
navigation.

## Architecture and invariants

- Add a `ConfigHubPane` (name may follow the local naming convention) as the sole pane
  constructed by the top-level `config` factory. It owns an immutable typed sub-tab
  catalog, a `ContentSwitcher`, active-sub-tab session state, and child factories.
- Mount only the requested Config child. Cache mounted children for the Admin Center
  modal's lifetime, preserve their selections, and forward top-level visibility and
  focus activation to the active child. A failed child mount must leave the previous
  sub-tab visible and focused and allow a later retry.
- Do not compose or load all five children eagerly. In particular, `XPromptBrowserPane`
  currently loads during construction, and the other catalogs start worker-backed I/O on
  mount. Opening Config must trigger only the active child's work; no disk or subprocess
  work may enter a key handler, render path, or Textual pump callback.
- Extend `AdminCenterSessionState` with a typed Config session object: active sub-tab,
  the existing Misc and XPrompt bookmarks, and bounded project/scope/entry selection
  state for Glossary, Memory, and Snippets. Explicit direct-entry seeds win once without
  destroying useful bookmarks for other scopes.
- Introduce a typed direct-entry request shared by `BaseActionsMixin`,
  `ConfigCenterModal`, and `ConfigHubPane`. It carries the Config sub-tab plus only the
  relevant launch workspace and term/note/trigger seed; avoid a growing collection of
  loosely related optional constructor arguments.
- Keep data/catalog behavior in its existing facades. This is Textual presentation,
  navigation, lifecycle, and Python glue, so it does not cross the Rust core backend
  boundary.
- Keep current standalone routes intact while the three independent pane extractions
  land. The production consolidation is then staged under one temporary beta feature
  flag created only with `sase flag new admin_center_config_hub`. Use these authored
  semantics: enabled means the six-section Admin Center and nested Config catalog own
  all five tools and prompt shortcuts; disabled means the current seven-section Admin
  Center and standalone prompt-opened panels remain available; remove the flag when all
  nested navigation, contextual entry, visual, and full-suite acceptance checks pass.
  Maintain explicit tests for both states until the cutover phase deletes the disabled
  branch and closes the separately created flag bead.
- Preserve prompt focus restoration as an end-to-end contract. Closing the Admin Center
  after entry from a prompt returns to the originating pane (or its mounted replacement)
  with Vim mode, cursor, and Snippets selection restored exactly as today, even if the
  user visited another Config or Admin Center tab before closing.

## Phase: `glossary_pane` — Extract a reusable Glossary content pane

Refactor the current `GlossaryPanel` so its browse/edit implementation can be mounted as
a child widget. Keep a thin standalone modal adapter during this phase so all current
call sites and behavior remain unchanged while later phases are still absent.

- Move composition, scoped bindings, worker-backed initial/project loads, debouncing,
  selection guard, relationship travel, mutation actions, and source/copy/help actions
  onto a reusable `GlossaryPane`-style widget. Keep mixin ownership clear rather than
  duplicating state between the pane and adapter.
- Define a small host contract for close, focus-default, and visibility activation. The
  standalone adapter dismisses itself; an embedded host will close the Admin Center.
  Worker and debouncer teardown must remain tied to unmount, while hidden cached panes
  neither steal focus nor repaint another sub-tab.
- Add session-state injection for active project and selected term, while retaining
  `launch_workspace`, explicit project, and explicit term precedence. Record selection
  changes through the injected object instead of relying only on ephemeral fields.
- Convert unit helpers to exercise the reusable pane directly where possible and keep a
  narrow adapter test for modal close/focus behavior. Preserve project cycling, stale
  worker-result rejection, filter behavior, relationship trail, add/delete flows, and
  source/help overlays.
- Keep the existing visual appearance unchanged in the standalone adapter during this
  phase; Config-host styling belongs to the integration/cutover phases.

## Phase: `memory_pane` — Extract a reusable Memory content pane

Refactor `MemoryPanel` into reusable content plus a thin current modal adapter, with no
user-visible route change yet.

- Preserve all Memory-specific machinery: tree-ordered notes, scope ring and picker,
  note/link trail, unpublished-scope tracking, digest restat after editor return,
  add/edit/delete, publish, generated-note protections, help, viewer, and copy actions.
- Give the reusable pane the same host close/focus/visibility contract as Glossary and a
  typed session object for active scope and selected note. An explicit `#memory/<stem>`
  seed must override the remembered row on direct entry but fall back safely when the
  note or scope no longer exists.
- Ensure every load, picker load, restat, and publish-related callback is cancelled or
  ignored safely after unmount or a newer request. Re-read live selection after awaits
  and keep programmatic OptionList selection guarded.
- Update component and adapter tests without losing coverage for the scope picker,
  unpublished state, generated/shadowed notes, mutations, publish flows, relationship
  navigation, errors, and teardown.

## Phase: `snippets_pane` — Extract a reusable Snippets content pane

Refactor `SnippetsPanel` into reusable content plus a thin current modal adapter, again
without changing its production entry route in this phase.

- Preserve catalog composition, project cycling, raw/composed previews, inbound and
  outbound relationship travel, filter modes, add/edit/delete flows, destination and
  conflict modals, source/viewer/copy actions, and stale-load rejection.
- Implement the common close/focus/visibility host contract and typed session state for
  active project and selected trigger. Keep direct trigger selection and prompt text
  selection restoration distinct: the former seeds the catalog, while the latter is
  restored only when the enclosing Admin Center is dismissed.
- Update direct pane and thin-adapter tests, retaining all relation, trail, mutation,
  diagnostic, filter, and worker teardown cases.

The three extraction phases may run in parallel. They must agree on the host/session
contract shape before editing shared exports or styles; each phase owns its panel family
and tests, and none changes the Admin Center catalog or prompt handlers.

## Phase: `config_hub` — Build and integrate the nested Config catalog

After all reusable panes exist, create the beta flag with `sase flag new` using the
semantics above, register the generated entry exactly as instructed, and integrate the
complete enabled path in one phase.

- Add the typed Config sub-tab catalog and lazy/cached host. Reuse the existing
  `ConfigPane` as the Misc child and `XPromptBrowserPane` as the XPrompts child; do not
  fork either implementation. Mount the three extracted content panes through the same
  factory and lifecycle path.
- Under the enabled branch, remove XPrompts from `CenterTab`, `_TAB_SPECS`, landing-page
  copy, main strip, descriptions, digit range, and factories. Keep main numbers 1–6
  stable. Map legacy persisted top-level `xprompts` history to `config`; because a new
  Config session defaults to XPrompts, an old resume still reaches equivalent content.
  Keep the disabled branch behavior byte-for-byte compatible where practical.
- Replace the old `config` bookmark plus top-level `xprompts` bookmark with the typed
  Config session object without losing current selection-resume behavior. Test repeated
  open/close, main-tab alternate history, sub-tab resume, child caching, lazy loader
  counts, failed mounts, stale selections, and activation callbacks.
- Route `GlossaryPanelRequested`, `MemoryPanelRequested`, and `SnippetPanelRequested`
  through the existing central `_open_config_center` path when enabled. Pass a typed
  Config target containing the prompt workspace and captured seed. Continue using the
  existing note-reference normalization and prompt-focus capture/restore helpers;
  consolidate their duplicated mechanics only if doing so keeps the three distinct
  restoration contracts obvious.
- Make nested navigation work from every child and focused input. Update XPrompt filter
  assumptions that currently reserve Tab and main-section digits, and add explicit
  arbitration in `ConfigCenterModal`/`ConfigHubPane` so scoped child bindings win only
  in the contexts that own them. No key should leak text into a filter or unexpectedly
  switch a main tab.
- Restyle the extracted panes when embedded: remove the standalone screen frame and
  redundant outer title, preserve their internal header/card hierarchy, and allocate
  height for both tab strips and the Admin Center footer. Keep the standalone disabled
  path visually unchanged.
- Update exports/type stubs, comments, help and hint strings, command/search aliases,
  Admin Center home copy, state parsing, and focused tests that name the old seven-tab
  arrangement. Do not change the user-configurable prompt opener chords or their default
  config/schema entries.
- Test both flag states. Enabled tests must prove prompt chords land on Config with the
  correct active child, scope/project, and term/note/trigger; disabled tests must prove
  the current standalone panels and top-level XPrompts tab still work.

## Phase: `cutover` — Polish, verify, and make the consolidation unconditional

Treat visual quality and interaction reliability as acceptance criteria, then remove the
beta branch rather than leaving a permanent preference.

- Re-home the existing Glossary, Memory, Snippets, XPrompts, and Config PNG scenarios
  inside the Admin Center Config frame. Cover populated, empty, loading, diagnostic,
  relation/trail, add/edit/delete/confirm, dark, and light states as applicable. Add at
  least one narrow-terminal snapshot proving the two-level hierarchy remains readable
  and content does not clip or create unusable rails.
- Add end-to-end keyboard tests starting from prompt insert and normal modes for all
  three opener chords. Verify contextual seed selection, nested `[`/`]`, relationship
  Tab/Shift+Tab, numbered relationship jumps versus main-section digits, filter Escape,
  Admin Center close, and exact cursor/Vim/selection restoration.
- Add lazy/performance regression assertions: opening Config constructs and loads only
  its active child, moving among already visited children does not reload them, closing
  cancels outstanding workers/debouncers, and rapid sub-tab/main-tab changes cannot let
  stale results move the visible selection. Run the relevant TUI responsiveness bench or
  trace with the documented p95 target if interaction code changes hot paths.
- Remove the `admin_center_config_hub` flag's disabled branch and registry definition,
  make the enabled route unconditional, regenerate/check the feature-flag schema as
  required by the flag tooling, and close the flag bead in the same change. Delete the
  now-unused standalone adapters and obsolete snapshots/tests only after equivalent
  Config-host coverage exists.
- Run `just install` first in the final combined workspace. Run focused unit and visual
  tests while iterating, then `just test-visual`. Because this is an epic landing and
  changes broad TUI navigation, run `just check-full` only through `/sase_monitor` with
  required `TESTING`/`TESTED` statuses and a follow-up action that inspects and resolves
  the result. Confirm `tools/check_feature_flags` has no orphan registry entry or live
  removal bead before landing.

## Acceptance criteria

- The main Admin Center contains no XPrompts tab and Config visibly contains exactly
  XPrompts, Snippets, Glossary, Memory, and Misc in the designed order.
- Opening Config is lazy, fast, failure-safe, and selection-preserving across sub-tab,
  main-tab, close/reopen, and legacy XPrompts-resume paths.
- Current Config functionality exists unchanged under Misc; current XPrompt, Glossary,
  Memory, and Snippets capabilities exist unchanged in their new children.
- Existing prompt opener keymaps open the Admin Center directly on the correct Config
  child, honor the prompt workspace and contextual seed, and restore prompt state on
  close.
- Nested and pane-local keymaps have deterministic, tested precedence in lists, filters,
  relationship chips, and overlays; no unexpected top-level navigation occurs.
- The two-level Admin Center is cohesive in dark/light and normal/narrow terminals, with
  no doubled frames, clipped content, ambiguous active state, or stale hints.
- No synchronous data-scaled work is added to UI handlers or render paths; hidden or
  closed panes do not steal focus or apply stale worker results.
- The temporary feature flag and disabled route are gone, all focused/visual/full checks
  pass, and no obsolete standalone entry point remains.
