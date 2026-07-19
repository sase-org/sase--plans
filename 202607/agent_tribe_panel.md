---
tier: epic
title: Agent Tribe panel summaries and whole-panel selection
goal: 'Selecting an Agent Tribe panel — collapsed or expanded — shows a rich, fold-aware
  tribe summary in the metadata panel with full clan/family-panel parity (numbered
  member jumps, per-section folding), tribe panels become first-class selectable and
  foldable elements with intuitive h/H/j/k/l semantics and a visually distinct selected
  state, and fold hints cover every visible collapsible element.

  '
phases:
- id: fold-scales
  title: Kind-scaled fold levels
  depends_on: []
  description: '''Kind-scaled fold levels'' section: add a fourth fold level and per-kind
    fold scales (family 2, clan 3, tribe 4) with scale-aware cycling, clamping, and
    n/N indicators.'
- id: tribe-summary
  title: Tribe summary document
  depends_on:
  - fold-scales
  description: '''Tribe summary document'' section: build the fold-aware tribe detail
    document (snapshot model + renderer) with a numbered roster, section anchors,
    and member-jump parity, replacing the static collapsed-panel summary.'
- id: panel-selection
  title: Whole-panel selection and keymaps
  depends_on:
  - tribe-summary
  description: '''Whole-panel selection and keymaps'' section: make expanded tribe
    panels selectable; rework h/H/l/j/k semantics, add per-panel selection memory,
    selected-panel visuals, and footer/help updates.'
- id: tribe-enrichment
  title: Deep tribe sections and statistics
  depends_on:
  - tribe-summary
  description: '''Deep tribe sections and statistics'' section: off-thread disk-backed
    tribe sections (replies, slow tool calls) and the level-4 runtime statistics lane
    with caching and loading states.'
- id: fold-hints
  title: Unified fold hints
  depends_on:
  - panel-selection
  description: '''Unified fold hints'' section: extend the leader fold-hint mode to
    all visible foldable elements with owner-level dedupe, and add apostrophe jump
    targets for expanded panels.'
- id: verify-polish
  title: End-to-end verification and polish
  depends_on:
  - panel-selection
  - tribe-enrichment
  - fold-hints
  description: '''End-to-end verification and polish'' section: end-to-end keymap
    flows, PNG visual snapshots for all tribe fold levels, selected-panel styling,
    and hint overlays, plus perf checks and help/footer audits.'
create_time: 2026-07-18 21:55:52
status: wip
---

# Plan: Agent Tribe panel summaries and whole-panel selection

## Context

The ACE Agents tab renders one left panel per agent tribe (the user-managed `tag`; panel titles render `@<tag>`, with
`(untagged)` for the `None` key). Panels are modeled by `AgentPanelGroup` / `PanelKey` in
`src/sase/ace/tui/models/agent_panels.py` and are the _only_ representation of a tribe — there is no tribe container
row. A tribe can span clans, families, and standalone agents; structural descendants inherit their rendered root's tag,
so a clan or family subtree never splits across panels.

Today's behavior, which this epic changes:

- **Metadata panel.** Clan containers get a rich, fold-aware detail document (`build_clan_detail_text` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`) and family containers get a similar treatment
  (`_agent_display_family.py`), both sharing the numbered member roster (`_member_roster.py`), section anchors
  (`_section_navigation.py`), per-section fold overrides (`SectionFoldStateManager`), and digit member-jump keymaps
  (`actions/navigation/_member_jump.py`). A focused _collapsed_ tribe panel, by contrast, only gets a basic static
  widget (`widgets/agent_panel_summary.py`, fed by `models/agent_panel_summary.py` via `_apply_collapsed_panel_summary`
  in `actions/agents/_display_detail.py`) with no folding, no section navigation, and no member jumps.
- **Selection.** Whole-panel focus exists only when the focused panel's key is in `_collapsed_panel_keys`
  (`_resolve_focused_collapsed_panel` in `actions/agents/_selection.py`). An expanded panel can never be selected as a
  panel; `j`/`k` are dead when a collapsed panel is focused (`_panel_navigation_stops()` returns `[]`).
- **Keymaps.** On the Agents tab: `h` (`action_hooks_or_collapse` → `_collapse_fold`, `actions/agents/_folding.py`)
  steps agent/family/clan folds down one level and, when nothing is left, escalates to collapsing the enclosing
  grouping-strategy group (e.g. `Running` under by-status grouping). `H` (`action_hooks_or_collapse_all`) collapses the
  whole focused panel; `L` expands it. `l` expands folds/groups. `J`/`K` cycle panel focus, landing on a row inside the
  next panel. `,H` (`toggle_selected_agent_panels`, `actions/agents/_panel_hint_folding.py`) shows numeric hints on
  panel titles only and toggles whole-panel collapse. `'` (`jump_to_entry`, `actions/navigation/_entry_jump_mode.py`)
  shows jump hints for visible agents, collapsed banners, and _collapsed_ panels only.
- **Fold levels.** `FoldLevel` (`models/fold_state.py`) has three values. The shared metadata-panel level
  (`app.panel_fold_level`) and section overrides (`_panel_fold_overrides`) drive both clan (3 useful levels) and family
  summaries (level 1 is a headings-only view of little value).

Everything here is presentation logic over already-loaded rows; per the Rust core boundary, no `sase-core` changes are
needed. The only Rust touchpoints are reused as-is: the artifact scan that populates `tag`/`clan_tribe`, and the
existing per-tribe runtime statistics query (`query_run_stats(runtime_group_by="tribe")` in `src/sase/stats/query.py`)
which the deepest fold level surfaces.

## Terminology

- **Tribe panel**: one Agents-tab tag panel (`@<tag>` or `(untagged)`).
- **Unit / house**: one top-level rendered root inside a tribe panel — a clan container, a family container (an "agent
  house"), a standalone agent, or a workflow parent.
- **Whole-panel focus**: the selection state in which the panel itself, not a row inside it, is selected. Today it
  exists only for collapsed panels; this epic extends it to expanded panels.

## Design overview

### Kind-scaled fold levels

Add a fourth `FoldLevel` member (`EXHAUSTIVE`) and introduce per-kind _fold scales_ — ordered tuples of the levels a
summary kind actually uses:

- Family: 2 levels `(EXPANDED, FULLY_EXPANDED)` — the old headings-only `COLLAPSED` rendering is dropped; level 1
  becomes the preview view, level 2 the full view.
- Clan: 3 levels `(COLLAPSED, EXPANDED, FULLY_EXPANDED)` — unchanged semantics.
- Tribe: 4 levels `(COLLAPSED, EXPANDED, FULLY_EXPANDED, EXHAUSTIVE)`.

The single shared `panel_fold_level` reactive is retained. Renderers resolve their _effective_ level by clamping/mapping
the shared level onto their scale; fold-mode cycling (`_handle_panel_fold_key` in `actions/navigation/_fold.py`) becomes
scale-aware for the currently selected summary kind, wrapping within that kind's scale (a family cycles 1→2→1; a tribe
1→2→3→4→1). Section overrides cycle within the owning kind's scale as well. The `Fold:` header indicator renders the
scale-relative position (`1/2`, `2/3`, `3/4`, …). The ChangeSpec-tab cyclers (`cycle_forward`/`cycle_backward`) keep
their three-level behavior and never observe `EXHAUSTIVE`. All `FoldLevel`-keyed glyph/style tables gain the fourth
entry (glyph ladder `▸ ▾ ▼ ◆`).

### Tribe summary document

A new fold-aware document rendered in the metadata panel whenever whole-panel focus is active (collapsed _or_ expanded
panel), replacing the static `AgentPanelSummary` widget. It renders through `AgentPromptPanel` — exactly like clan
documents — so section anchors, viewport-based section fold cycling (`_current_agent_metadata_section_id`), and
member-jump publication all work unchanged.

**Header (always visible):** `TRIBE` heading with the panel label (`@<tag>`, styled in the established tag gold
`#FFD75F`; the untagged panel reads `(untagged)`), aggregated status + count chip (reusing
`agent_summary_status_counts`), composition line (`N clans · N families · N agents`, plus nested counts), runtime span,
panel state (`expanded` / `collapsed`), and the `Fold: n/4` indicator.

**Four fold levels, each with a distinct job:**

1. **Pulse** (`COLLAPSED`) — orientation at a glance: header plus a `NEEDS ATTENTION` digest (failed / stopped / waiting
   units as capped triage lines with first meaningful error/activity line), and count-only headings for every other
   section. The member roster renders as a heading with count only.
2. **Roster** (`EXPANDED`) — the command view: the numbered `TRIBE MEMBERS` roster appears, one line per unit (kind,
   status glyph, model, duration, and a status count chip for clan/family units), published as a member jump map so
   digit keymaps jump straight to a unit's row. Aggregate sections (`ERRORS`, `OUTPUT VARIABLES`, `WORKFLOW VARIABLES`)
   show capped triage previews.
3. **Members** (`FULLY_EXPANDED`) — the drill-down view: roster units expand their nested members (clan members inline,
   family chains as child rows, like the clan roster), with activity/waiting/retry annotations; disk-backed sections
   (`REPLIES`, `SLOW TOOL CALLS` — see the enrichment phase) show per-member triage previews.
4. **Forensics** (`EXHAUSTIVE`) — the everything view: full error bodies with highlighted tracebacks, full variable
   values grouped by member, full member annotations (workspace, start/run/stop timestamps, attempts), full disk-backed
   section bodies, and the `RUNTIME STATISTICS` lane (runs, total/mean/p50/p95/max runtime for this tribe from the Rust
   stats query).

Every section is independently foldable through the existing override machinery using tribe-scoped section ids; each
roster entry keeps its own `member:<name>` anchor. The snapshot model is pure (built from in-memory rows, no I/O),
mirroring `build_agent_panel_summary_snapshot`'s approach but structured around units; in-memory aggregation generalizes
the clan aggregation helpers to a mixed list of roots.

### Whole-panel selection

Generalize `CollapsedAgentPanelFocus` into a panel-focus concept that carries `panel_key` and collapsed state.
Whole-panel focus is now active when (a) the focused panel is collapsed (unchanged), or (b) a new explicit
expanded-panel-focus flag is set. `_get_selected_agent()` returns `None` in both cases, so the existing panel-scoped
machinery (footer conditions, panel kill/mark bulk actions, info-panel `summary` view mode) generalizes with mostly
mechanical renames (`collapsed_panel_focused` → panel-focus context with a collapsed flag in `commands/context.py`,
`commands/types.py`, `commands/availability.py`, `widgets/_keybinding_bindings.py`).

Add per-panel selection memory: a `PanelKey → last selected stop` map updated on every in-panel selection change, used
when descending back into a panel. Whole-panel focus is split-mode only (as panel collapse already is); merged mode
(`,g`) keeps current behavior.

### Keymap semantics (Agents tab)

| Key      | Row selected (agent/banner)                                                                                                                                                                                                                                    | Whole-panel focus, expanded                                                                                                                                                                      | Whole-panel focus, collapsed                               |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| `h`      | Step agent/family/clan folds down one level as today, but **without** the group-collapse escalation; when nothing is left to collapse, select the containing tribe panel (whole-panel focus). On a collapsed banner: select the containing panel.              | Collapse the panel (stays selected; panel re-partitions to the collapsed tail as today, focus follows by key).                                                                                   | No-op (brief notify).                                      |
| `H`      | **New:** collapse the enclosing grouping-strategy group — exactly the escalation `h` performs today (L1 then L0; on a banner, collapse or escalate to the parent). Replaces `H`'s old collapse-focused-panel behavior, which `h`-on-selected-panel now covers. | No-op.                                                                                                                                                                                           | No-op.                                                     |
| `l`      | Unchanged: expand folds; on a collapsed banner, expand the group.                                                                                                                                                                                              | Descend into the panel: restore the remembered selection (or the first stop) and clear panel focus.                                                                                              | Expand the panel, keep it selected. A second `l` descends. |
| `j`/`k`  | Unchanged row navigation.                                                                                                                                                                                                                                      | Move whole-panel focus to the next/previous panel (wrapping), keeping whole-panel focus regardless of that panel's collapsed state. This also replaces the current dead j/k on collapsed panels. | Same as expanded.                                          |
| `J`/`K`  | Unchanged: jump into the next/previous panel's first/last row.                                                                                                                                                                                                 | Unchanged.                                                                                                                                                                                       | Unchanged.                                                 |
| `L`      | Unchanged (expand focused panel).                                                                                                                                                                                                                              | Unchanged.                                                                                                                                                                                       | Unchanged.                                                 |
| digits   | Member jump within clan/family summaries (unchanged).                                                                                                                                                                                                          | Member jump to a tribe roster unit: selects that unit's row and clears panel focus.                                                                                                              | Same (jump implies expanding into the panel).              |
| `Escape` | —                                                                                                                                                                                                                                                              | Return to the remembered row selection (like `l`).                                                                                                                                               | Unchanged.                                                 |

`l`/`h` on the Tools panel keep their existing routing precedence (`_route_tools_detail_level`).

### Hints

- **`,H` fold hints** (`toggle_selected_agent_panels`; the config key name is kept for compatibility): extended from
  panels-only to **all visible foldable elements** — every tribe panel title (collapsed and expanded), every group
  banner, every clan container row, every family root, and every agent owning a workflow fold. One hint per fold
  _owner_: child steps never get hints (their collapse action belongs to the parent agent's fold), and no two hints may
  perform the same action. Numeric multi-selection semantics are preserved (`parse_numeric_hint_selection`); each
  selected element toggles between collapsed and its expanded state atomically in one repaint. Row-level hint chips
  reuse the existing jump-hint rendering channel in `widgets/_agent_list_build.py`.
- **`'` jump hints**: `("panel", key)` targets are emitted for _expanded_ panels too (rendered on the panel title).
  Selecting an expanded-panel target activates whole-panel focus on it; the dispatch validation that currently rejects
  non-collapsed panels is updated, and `("panel", key)` back-jump anchors work for both states.

### Visual design

- **Selected expanded panel** must be unmistakable: the focused `AgentList` gets a distinct border + title treatment in
  the tribe gold `#FFD75F` (e.g. heavy border and a `❖` title marker), and the in-panel row cursor highlight is
  suppressed while whole-panel focus is active, so it cannot be confused with a selected row inside the panel.
- **Identity triad**: clans keep `_CLAN_IDENTITY_COLOR`, families keep `#00AFFF`, and tribes adopt the tag gold
  `#FFD75F` (roster number chips `bold black on #FFD75F`, roster rule and headings in the same accent) so the three
  container summaries are visually distinguishable at a glance.
- Fold glyph ladder `▸ ▾ ▼ ◆` with a brighter style for `◆` (forensics).

## Phase details

### Kind-scaled fold levels

Scope:

- Add `FoldLevel.EXHAUSTIVE` to `models/fold_state.py` plus a small fold-scale module (scales per summary kind; helpers
  for effective-level mapping, scale-relative position, and forward/backward cycling within a scale).
- Make `_handle_panel_fold_key` (`actions/navigation/_fold.py`) scale-aware: determine the selected summary kind (family
  container / clan container / whole-panel focus / plain agent), cycle `panel_fold_level` within that kind's scale, and
  cycle/toggle section overrides within the same scale. Plain-agent selections keep today's behavior and notification.
- Family renderers (`_agent_display_family.py` and the family paths in `_agent_display_header.py` /
  `_agent_display_render.py`): map the shared level onto the 2-level scale — level 1 renders what `EXPANDED` renders
  today (previews, digests, roster annotations), level 2 renders the full view. Drop the headings-only branches;
  `fold_number` and the fold indicator become scale-relative (`n/2`).
- Clan renderer: clamp `EXHAUSTIVE` → `FULLY_EXPANDED`; indicator stays `n/3`.
- Audit every `FoldLevel`-keyed dict (`_FOLD_CHARS`, `_FOLD_STYLES`, `_FOLD_NUMBERS`, member-roster annotation switches)
  for the fourth member.
- ChangeSpec/axe fold paths remain untouched; add a regression test proving the shared reactive never leaks `EXHAUSTIVE`
  into clan/family/ChangeSpec rendering un-clamped.

Testing: unit tests for scale cycling/clamping/position; updated family panel tests and PNG goldens for the 2-level
family summary; existing clan goldens must remain byte-identical.

### Tribe summary document

Scope:

- New pure snapshot model (e.g. `models/agent_tribe_summary.py`): units in panel render order (clan containers, family
  roots, standalone agents, workflow parents) with per-unit status/model/duration/count-chips and nested member digests;
  tribe-level aggregate counts; `NEEDS ATTENTION` digest; in-memory `ERRORS` / `OUTPUT VARIABLES` / `WORKFLOW VARIABLES`
  entries built by generalizing the clan in-memory aggregation (`models/_agent_clan_sections.py`) to a list of mixed
  roots. No filesystem access.
- New renderer (e.g. `widgets/prompt_panel/_agent_display_tribe.py`) implementing the four levels from the design
  overview: header, attention digest, numbered `TRIBE MEMBERS` roster via `append_member_roster` (heading count-only at
  Pulse; entries without children at Roster; children + annotations at Members; full annotations at Forensics — a small
  roster extension for heading-only rendering is expected), and the fold-aware sections with tribe-scoped section ids
  and dividers.
- Route rendering through `AgentPromptPanel`: when whole-panel focus is resolved, `_apply_agent_detail_immediate` /
  `_apply_agent_detail_update` build the tribe document into the prompt panel (clan-style: immediate paint stays cheap;
  the full document goes through the existing debounced path). Retire the static `AgentPanelSummary` widget, its scroll
  container, the `-summary-active` layout class, and `is_panel_summary_visible` branches (including
  `_get_agent_detail_scroll_id`), so detail-panel scrolling, section folding, and `ctrl+d/u` operate on the tribe
  document like any other.
- Member-jump parity: publish the roster jump map under a panel-scoped container identity (widen the jump-map registry
  key beyond agent identity) and generalize `_selected_member_jump_container` so digits work under whole-panel focus; a
  jump selects the unit's row (clearing panel focus).
- Footer/info updates: info-panel view mode reads `tribe` while the document is shown; footer bindings for panel focus
  come in the panel-selection phase.
- The document renders identically for collapsed and expanded panels (the header's panel-state line differs), and for
  the `(untagged)` pool.

Testing: snapshot-model unit tests (unit ordering, mixed clans/families/ standalone agents, attention digest,
marked/unread projection); renderer tests per fold level; member-jump tests from tribe rosters; PNG goldens
`agents_tribe_panel_*_120x40.png` for all four levels following the `agents_clan_panel_*` naming pattern.

### Whole-panel selection and keymaps

Scope:

- Panel-focus model: generalized focus resolution (collapsed-or-explicit), expanded-panel focus flag on the app,
  per-panel last-selection memory, and clearing rules (tab switches, panel disappearance, merged-mode toggle, refresh
  reconciliation by key).
- Keymap rework in `actions/agents/_folding.py`, `_panel_navigation.py`, and `actions/navigation/_basic.py`, exactly per
  the semantics table: `h`'s escalation now terminates in panel selection instead of group collapse; new `H` behavior
  carries the group-collapse logic verbatim (including banner escalation and focus snapping); `l` descends/expands;
  `j`/`k` navigate panel-to-panel under panel focus (reusing `AgentPanelGroup.focus_next/prev` and preserving jump
  anchors, unread-departure arming, and nav-stops cache discipline); `Escape` exits panel focus into the remembered row.
- Visual distinctness: selected-expanded-panel border/title styling and row cursor suppression, applied through the
  existing panel-title/border refresh path (`_display_panel_titles.py` and the AgentList styling hooks) without full
  list rebuilds.
- Bulk/panel-scoped actions and availability: generalize `collapsed_panel_focused` context/conditions so kill/mark/panel
  actions work identically under expanded-panel focus.
- Config/help/footer: `src/sase/default_config.yml` keymap comments/docs reflect the new `h`/`H` semantics; help modal
  (`?`) agents section updated within the 57-char box convention; keybinding footer gains the conditional panel-focus
  bindings (and drops ones that no longer apply), honoring the footer's conditional-only rule.

Testing: pilot-style keymap tests (h-chain into panel selection; h collapse; l/l descend; j/k panel cycling incl. wrap
and mixed collapsed/expanded panels; H group collapse parity with today's h behavior; Escape), navigation order/stops
cache tests, footer condition tests, persistence tests proving `h`-driven panel collapse records intents like `H` used
to.

### Deep tribe sections and statistics

Scope:

- Disk-backed tribe sections for Members/Forensics levels: `REPLIES` and `SLOW TOOL CALLS`, aggregated per unit by
  reusing the clan section snapshot machinery (`widgets/prompt_panel/_agent_clan_aggregation.py`) — per-clan cached
  snapshots are reused as-is; the per-root loader is generalized for family/standalone units with the same mtime-keyed
  caching, coalescing (loading/pending flags, last-request-wins), and capped entry counts. Collapsed levels render
  count-only headings; unknown-yet sections render the established `loading…` placeholder. All loading happens
  off-thread via the existing pump-free enrichment pattern; render paths never stat or glob.
- `RUNTIME STATISTICS` lane (Forensics only): off-thread, TTL-cached call to the existing Rust per-tribe runtime stats
  query filtered to this tribe (runs, total/mean/p50/p95/max, share), with loading placeholder and a dash state when the
  archive has no runs for the tribe. The cache is keyed per tribe and never recomputed on j/k or render.
- Repaint plumbing mirrors `ClanSectionSnapshotLoaded`: enrichment completion posts a message that schedules the
  debounced detail update only when the same panel is still focused.

Testing: aggregation unit tests over mixed units with seeded artifacts; cache reuse tests (clan snapshot reused, no
duplicate loads); loading-state renderer tests; stats-lane tests with a stubbed query; a guard test that Pulse/Roster
levels trigger no disk section loads.

### Unified fold hints

Scope:

- Extend `,H` (`actions/agents/_panel_hint_folding.py`) to build a hint list over all visible foldable elements in
  visual order: panel titles (both states), group banners, clan containers, family roots, and step-owning agents.
  Owner-level dedupe per the design; hint chips on rows reuse the jump hint rendering channel; panel-title hints keep
  their current rendering. Selection toggling stays atomic (one repaint) and panel toggles keep their persistence
  recorder; group toggles go through the group fold registry with its version bump; agent/clan/family toggles go through
  `_fold_manager`.
- Apostrophe mode: emit `("panel", key)` targets for expanded panels, update dispatch to activate whole-panel focus for
  them, and extend back-jump anchor validation accordingly.

Testing: hint enumeration/dedupe unit tests (child steps produce no hints; no duplicate-action hints), multi-select
toggle tests across mixed element kinds, apostrophe tests for expanded-panel targets and back-jump, and a PNG golden for
the hint overlay.

### End-to-end verification and polish

Scope:

- End-to-end pilot flows stitching the phases together: navigate rows → `h` to panel focus → cycle fold levels 1–4 →
  digit-jump to a unit → back-jump; `j`/`k` across expanded and collapsed panels; `,H` bulk-toggling a mix of panels,
  groups, clans, and agents; behavior under search filtering, grouping mode cycling (`o`), merged-mode toggle (`,g`),
  and refresh churn (panels appearing/disappearing while focused).
- Full PNG visual snapshot coverage: tribe document at all four fold levels, selected-expanded-panel styling, hint
  overlays, and the updated 2-level family summary.
- Performance verification per the TUI perf rules: `bench_tui_jk` p95 within budget on the Agents tab with tribe panels
  focused; a spot `SASE_TUI_PERF=1` run; confirm no new stall-watchdog entries and no disk I/O on the render path
  (Pulse/Roster levels are pure).
- Final audits: help modal content and box widths, footer conditional bindings, `default_config.yml` docs, and removal
  of dead code paths from the retired static summary widget.

## Testing strategy (cross-cutting)

Each phase lands its own unit + pilot tests alongside the code, following the existing patterns
(`tests/ace/tui/test_agent_collapsed_panel_selection.py`, `test_agents_panel_fold_mode.py`,
`test_member_jump_navigation.py`, `test_jump_hints_for_folded_banners.py`). PNG goldens live in
`tests/ace/tui/visual/snapshots/png/` and are updated only via `--sase-update-visual-snapshots`; `just test-visual` runs
the suite. `just check` gates every phase.

## Performance constraints (must hold throughout)

- Tribe documents are built from in-memory rows; Pulse/Roster levels perform zero I/O. Disk/statistics enrichment is
  off-thread, coalesced, mtime/TTL-cache keyed, and loading-aware — never blocking the pump.
- The tribe document follows the clan pattern for j/k: highlight paints immediately; the full document build rides the
  existing debounced detail path.
- Panel-focus changes and selected-panel styling go through selective update paths (title/border refresh, `patch_row`),
  not full list rebuilds; the nav-stops memo keys gain the new focus state.
- Keystroke paths stay read-only and prompt-free; no new refresh code paths — reuse `_refilter_agents()` /
  `_schedule_agents_async_refresh()` plumbing.

## Risks and mitigations

- **Fold-scale regressions in existing summaries.** Mitigated by clamping at render boundaries, leaving ChangeSpec
  cyclers untouched, and pinning clan goldens byte-identical in the fold-scales phase.
- **Keymap muscle-memory changes (`h`/`H`).** The plan deliberately preserves `J`/`K`/`L` and the `,H` config key; help
  modal and footer updates land in the same phase as the behavior change so the UI documents itself.
- **Large tribes.** Roster capped by `MEMBER_ROSTER_LIMIT` with an explicit `+N more` tail; triage sections keep the
  clan entry caps; enrichment is per-unit cached so cost scales with cache misses, not panel size.
- **Refresh churn under panel focus.** Focus is key-based (like `AgentPanelGroup`), reconciled on every rebuild; a
  vanished panel falls back to the first available panel without leaving a dangling expanded-focus flag.
