---
tier: epic
title: Group scheduled routines by declaring source in Services navigation
goal:
  Make user, plugin, and builtin routines immediately findable in distinct, reliable
  Services nav sections while preserving fast navigation and visible health.
phases:
  - id: core_origin
    title: Establish routine and job declaring-source contract
    depends_on: []
    description:
      "core_origin: add stable first-declaration metadata to AXE inventory, with
      composition and generated-job tests."
    size: medium
  - id: python_origin
    title: Consume source metadata and expose it consistently
    depends_on:
      - core_origin
    description:
      "python_origin: ratchet the core revision and expose typed origin through cached
      config, CLI, and Services details."
    size: medium
  - id: source_panels
    title: Build source-based Services nav sections
    depends_on:
      - python_origin
    description:
      "source_panels: replace the single routine panel with fixed source sections and
      coherent row ordering, focus, sizing, and navigation."
    size: medium
  - id: density_polish
    title: Make dense builtin navigation calm and observable
    depends_on:
      - source_panels
    description:
      "density_polish: add cached health badges before default folding, then verify
      visual design, documentation, and layout."
    size: medium
proposed_by: bbugyi200.apollo.1v
create_time: 2026-09-26 07:23:35
status: wip
---

- **PROMPT:**
  [prompts/202609/routine_source_nav_sections.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/routine_source_nav_sections.md)

# Design

Today every scheduled routine and its jobs share one alphabetic panel. On a 120×40
Services screen, the seven builtin routines and their roughly 31 jobs can push a
user-declared routine below the viewport. The research report
`research:202609/routine_source_nav_sections/routine_source_nav_sections.md` confirms
this in live captures and shows that the current sidebar has one flat row list, a
two-panel index, and a height allocator that already accepts more panels. The flat list
is valuable: actions, jump history, and identity restoration depend on it.

Use **real nav sections** (separate `BgCmdList` panels), not in-list dividers. Compose
only four fixed panels, in this order: **Service Procs → User Routines → Plugin Routines
→ Builtin Routines**. Show each routine panel only when it has a routine. If no routines
exist, show the User Routines panel with the existing add hint. This caps chrome and
gives a clear empty state. User routines lead because they are the routines the operator
declared and can remove. Plugin routines identify external declarations without making a
panel per plugin. Builtin routines are last and receive the remaining vertical space.
Labels remain `User Routines`, `Plugin Routines`, and `Builtin Routines`, using the same
provenance vocabulary as service procs.

Source means the **first config layer that declared the entity**, not the layer that
last changed one of its fields. A builtin routine with a user interval override remains
Builtin. A plugin routine with user overrides remains Plugin. User, overlay, and
project-local declarations are User. Jobs always appear directly below their parent
routine in the parent's panel; a job's own declaring source can still appear in its
detail. Generated jobs inherit their base job's origin. Keep alphabetical order within
each panel; do not move rows when runtime status or interval changes.

The routine panels have independent scroll positions and per-panel counts. The
scheduler's stopped/unavailable badge appears on the first visible routine panel only.
Keep the existing status, job, missing-script, and overrun chips, scoped to the routines
in that panel. Use a restrained source cue in a plugin routine row only when it helps
identify `declared_by`; avoid repeating the section label on every row. A selected row,
its parent, and its panel title should have one clear visual hierarchy. Titles must
truncate cleanly at narrow terminal widths and use singular `job` when appropriate.

For the common Builtin+User case, the left column should read approximately:

```text
╭ ◷ User Routines · 1 [R1] · 1 job ───────╮
│ ▌ telegram                               │
│   └ tg_outbound                          │
╰──────────────────────────────────────────╯
╭ ◷ Builtin Routines · 7 [R7] · 31 jobs ─╮
│ ▌ checks                                 │
│ ▌ comments                               │
│ ▌ external_mirror                        │
│ ▌ hooks                                  │
│ ▌ housekeeping  !1                       │
│ ▌ usage                                  │
│ ▌ waits                                  │
╰──────────────────────────────────────────╯
```

**No temporary feature flag is needed:** the first two phases add complete provenance
data without changing panel behavior; the panel phase lands all navigation and layout
semantics together. The final phase improves density without changing source membership.
The fold preference is a permanent user choice, so it is a config value rather than a
feature flag.

# Phase details

## 1. `core_origin` — sase-core

- Open the linked `sase-core` checkout with `/sase_repo`. In
  `crates/sase_core/src/config/axe.rs`, add `source` (`builtin | plugin | user`) and
  `declared_by` to routine, base job, and generated-job `AxeInventoryEntryWire` entries.
  Derive both from the earliest ordered layer whose raw entity value exists.
  `raw_routine_value` and `raw_contribution` already understand public `routines`/`jobs`
  and legacy `lumberjacks`/`chops`; reuse them rather than field provenance or the
  writable-only `contributions` projection.
- Share the layer-kind classification and declaration-label formatting used by service
  procs in `service/config.rs`; do not create a second set of origin rules. Define
  behavior for invalid/missing origin at the core boundary rather than silently
  assigning User. Preserve compatibility with the current wire schema and fixtures; if
  schema/version handling changes, update both repositories' matching constants and
  fixtures together.
- Cover first declaration after a later override; plugin declaration overridden by user;
  overlay/local declaration; job newly added under a builtin routine; legacy and public
  spellings; generated job inheritance; and every effective inventory entry having an
  origin. Verify with focused Rust tests, then the core repo's guarded
  `sase tool run check` gate.

## 2. `python_origin` — sase

- Move `sase-core-revision.txt` past the completed core commit before calling the new
  binding contract. Parse required `source` and `declared_by` fields in
  `AxeInventoryEntry.from_wire` with a closed source type and a useful error for
  malformed data. Do not infer source from names, field provenance, or writable
  contributions. Propagate origin onto `LumberjackConfig`/`ChopConfig` or an equally
  typed projection built from the same cached `AxeConfigComposition`; generated runtime
  jobs must retain their base job's origin.
- Make the off-thread AXE collector return a complete routine-name → source/declared-by
  map alongside `lumberjack_names`, using the existing config-token cache. Apply names
  and origin atomically before rebuilding rows. Header-only refreshes and targeted
  refreshes must preserve the map, and a config-token change must invalidate it. A
  malformed or incomplete composition should use the existing degraded-config state
  rather than arbitrarily assigning a panel.
- Show source in `sase axe routine list` without changing command options, and show
  `Source: <source> (<declared_by>)` in routine/job detail using service-proc wording.
  Keep execution source (`scheduled`, `manual`, `oneshot`) separate from configuration
  origin. Test config projection, plugin/user override stability, cache invalidation,
  CLI output, and details. Confirm that project-local routine visibility matches the
  scheduler's effective config before relying on the User label.

## 3. `source_panels` — sase TUI

- Generalize `actions/axe_display/_panels.py` to keys `service_procs`, `user_routines`,
  `plugin_routines`, `builtin_routines`. Its index owns the ordered visible panels and
  global ↔ local mappings. Membership uses the cached parent routine origin for both
  `LumberjackItem` and `ChopItem`; unknown item types and unknown source values fail
  clearly. Preserve the flat `_axe_items` list and stable identity keys.
- Build `_axe_items` in panel order, then alphabetically by routine with its jobs
  immediately after it. Update `_app_layout.py`, `_render_panels.py`,
  `_panel_navigation.py`, `_panel_titles.py`, `styles.tcss`, and widget event mapping to
  consume the same index/order. Remove the old `#scheduled-routines-panel` reference.
  Hide empty routine panels, except the single User empty state when no routines exist.
  Give only visible panels to `allocate_panel_heights`, with the last visible panel as
  filler; apply gaps by a shared class rather than a hard-coded panel ID. Width
  calculation ignores hidden panels.
- Title stats are computed once per visible source group from caches. A panel's title
  and rows agree about source and job counts, and the global scheduler badge is rendered
  once. `j`/`k` cross section boundaries on the global list without selecting chrome.
  `J`/`K` wrap through nonempty sections only, and `Ctrl+O`, click selection, jump
  hints, folds, add-flow cursor following, and background refresh preserve row identity
  and focus ownership. Same-panel highlight stays O(1) widget work with no disk access.
- Test index mappings and contiguous order for all source mixtures; empty, only-builtin,
  only-user, and all-three panels; grouped `j`/`k` and `J`/`K`; clicks and jump-back;
  selection after a fold, source-neutral override, added routine, and config reload.
  Test that refresh does not steal prompt/modal focus.

## 4. `density_polish` — sase TUI and docs

- Derive a per-routine `!N` health count from already cached latest job snapshots using
  the same failure, timeout, and missing-script statuses as the panel title. Include the
  count on the routine row even when its children are folded; count every
  configured/generated visible job once. Distinguish no snapshot yet from zero failures
  so loading does not claim health it has not observed.
- Add `ace.services.fold_builtin_routines: true` in `src/sase/default_config.yml` for
  **fold builtin routines on first sight**. Apply it only once the full snapshot needed
  for the health badge is ready; a header-only pre-load must not initialize a misleading
  fold. Explicit user folds persist for the TUI session. Newly added User or Plugin
  routines start expanded. Tests cover failure under a folded routine, missing script,
  no snapshot, user override of the preference, and selection restoration.
- Refine spacing, accents, truncation, empty-state copy, and onboarding text. Update
  `docs/ace.md` and `docs/axe.md` to describe source semantics, panel order, `J`/`K`,
  and folding. Update affected unit tests and inspect Services PNG goldens at 120×40,
  100×30, and a narrow layout; include Builtin+User, all three source groups, and only
  Builtin. Use the project's approved screenshot tooling and review each changed golden.
  Compare pre/post key-to-paint and idle refresh traces against the TUI performance
  guidance: no added render-path I/O and no material `j`/`k` latency regression. Run
  `just fix` and the required `just check` gate under the repository's guarded tool
  workflow.

# Acceptance and boundaries

- A routine's section is stable across field overrides, status changes, and refreshes.
  The CLI and TUI report the same declaring source; generated jobs have a truthful
  source and stay with their parent routine.
- The user can reach their routine without scrolling through builtin jobs. Navigation,
  clicks, folding, jump-back, add flow, and row selection remain predictable in every
  source combination.
- A collapsed builtin routine still reports failures and missing scripts at the parent
  row; the layout remains legible at 100×30 and narrow widths. No configuration or
  filesystem work enters paint or keystroke paths.
- This plan does not add user-authored `group:` labels, one panel per plugin,
  status/cadence regrouping, or a grouped/ungrouped toggle. The `telegram` routine
  remains User while declared in user config; moving its declaration to the plugin is a
  separate decision because it would also change where the scheduler starts it.
- The existing glossary's singular Scheduled Routines wording may need a separately
  authorized memory edit. Memory files are outside these phases because
  `/sase_memory_write` requires explicit confirmation before proposing a plan that would
  change them.
