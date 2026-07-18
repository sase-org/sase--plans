---
tier: epic
title: Agent metadata panel folding with a rich clan summary
goal: 'The agent metadata panel supports three-level section folding driven by a `z`
  fold keymap group (zoom migrates to `Z`), and the clan summary page becomes its
  first use-case: a beautiful, fold-aware aggregate of every section represented across
  the clan''s members.

  '
phases:
- id: fold-keymaps
  title: Fold keymaps and panel fold state
  depends_on: []
  description: '''Fold keymaps and panel fold state'' section: migrate zoom from `z`
    to `Z`, enable fold mode on the Agents tab with an `agents` sub-map (`zz`, `zZ`,
    `za`, `zA`), and add the panel fold state model, dispatch, footer hints, palette
    commands, and help-modal updates.'
- id: clan-aggregation
  title: Clan section aggregation layer
  depends_on: []
  description: '''Clan section aggregation layer'' section: build the data layer that
    aggregates member sections into clan section snapshots — pure in-memory aggregates
    plus cached, off-thread loading for disk-backed member content.'
- id: clan-render
  title: Fold-aware clan summary rendering
  depends_on:
  - fold-keymaps
  - clan-aggregation
  description: '''Fold-aware clan summary rendering'' section: render the clan summary''s
    aggregate sections honoring per-section three-level fold contracts, with headings,
    counts, fold indicator, section-navigation anchors, loading placeholders, and
    PNG snapshots at every level.'
- id: polish-exercise
  title: Docs, help sync, and end-to-end fold exercises
  depends_on:
  - clan-render
  description: '''Docs, help sync, and end-to-end fold exercises'' section: update
    user docs and the help popup, run end-to-end clan fold exercises against a real
    launched clan, and verify performance targets and visual goldens.'
bead_id: sase-6u
---

# Plan: Agent metadata panel folding with a rich clan summary

## Product context

Agent clans (epic sase-6n) gave the Agents tab a synthetic clan container row whose detail panel
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`, `build_clan_detail_text`) is currently a thin
aggregate: a header block (Name / Tribes / Status / Runtime / Members) plus a single MEMBERS section. Members' actual
content — errors, replies, output variables, SASE context, slow tool calls, prompts — is invisible from the clan row;
the user must drill into each member.

This plan makes the clan summary a genuinely useful triage surface by representing **every section that appears in any
member** as an aggregate section of the clan summary — and, because that could be an enormous document, introduces
**three-level section folding for the agent metadata panel** as new general functionality, with the clan summary as its
first consumer. The `z` key is freed for a fold keymap group on the Agents tab by migrating zoom to `Z`.

## Existing machinery to reuse (do not reinvent)

- **`FoldLevel` + `cycle_forward`** (`src/sase/ace/tui/models/fold_state.py`): the established 3-state fold vocabulary
  (COLLAPSED → EXPANDED → FULLY_EXPANDED). The ChangeSpec detail view on the Artifacts/PRs sub-tab already uses it with
  per-section builder contracts (reference implementation: `src/sase/ace/tui/widgets/commits_builder.py`, whose
  docstring spells out what each level shows). Panel fold levels 1/2/3 map exactly onto this enum.
- **Fold mode** (`z` prefix): `src/sase/ace/tui/actions/navigation/_fold.py`
  (`FoldNavigationMixin.action_start_fold_mode` / `_handle_fold_key`), keyboard routing in
  `src/sase/ace/tui/actions/_event_keyboard.py`, config under `ace.keymaps.modes.fold_mode` in
  `src/sase/default_config.yml`, typed schema `FoldModeKeymaps` in `src/sase/ace/tui/keymaps/types.py`, loader in
  `src/sase/ace/tui/keymaps/loader.py`. `ModeKeymaps.keys` already supports nested `dict[str, str | dict[str, str]]`
  values.
- **Tab multiplexing of `z`**: `check_action` in `src/sase/ace/tui/app.py` (~lines 374-377) currently suppresses
  `start_fold_mode` on the Agents tab and `zoom_panel` elsewhere.
- **Section anchors and navigation**: `PromptPanelSectionAnchor` / `SectionTrackingVisual` in
  `src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py`, plus `_cycle_agent_metadata_section` in
  `src/sase/ace/tui/actions/navigation/_basic.py` (resolves the section anchor relative to the viewport — the same
  machinery gives us "the current section" for per-section fold keys).
- **Fold-mode footer hints**: `update_fold_bindings` in `src/sase/ace/tui/widgets/_keybinding_modes.py`; fold badge
  glyphs in `src/sase/ace/tui/widgets/changespec_info_panel.py` (`_FOLD_CHARS` ▸/▾/▼ vocabulary).
- **Clan aggregation model**: `src/sase/ace/tui/models/_agent_clan.py` (`clan_members`, `clan_member_counts`,
  `aggregate_clan_status`) and the synthetic container built in `src/sase/ace/tui/models/_agent_tree.py`
  (`runtime_children` holds the member rows).
- **In-memory member data** (`src/sase/ace/tui/models/_agent_state.py`): `error_message`, `error_traceback`,
  `output_variables`, `epic_bead_id`/`phase_bead_id`, `plan_path`, `workspace_num`/`workspace_dir`, `model`, `activity`,
  `attempt_history`, `response_path`, `artifacts_dir` are already loaded on every row — no disk access needed to count
  or digest them. Reply bodies, prompts, SASE context lanes, and slow tool calls are disk-backed and loaded
  asynchronously in the regular agent display path (`src/sase/ace/tui/widgets/prompt_panel/_agent_display.py` /
  `_agent_display_render.py`; note the clan branch currently cancels these workers and renders purely).

This is a presentation-layer feature: it aggregates data the wire already delivers, so no `sase-core` (Rust) changes are
expected. If any aggregation rule turns out to be shared domain behavior needed by non-TUI frontends, move that rule to
`sase-core` per the backend-boundary convention rather than duplicating it here.

## Design

### Fold levels

The metadata panel has a single session-only **panel fold level** with three values, mapped onto `FoldLevel`:

| Level | FoldLevel      | Meaning for clan sections                                                          |
| ----- | -------------- | ---------------------------------------------------------------------------------- |
| 1     | COLLAPSED      | Default. Heading + count only — except MEMBERS, which shows its full member table. |
| 2     | EXPANDED       | **Triage digest**: every section unfolds to bounded, one-line-per-entry summaries. |
| 3     | FULLY_EXPANDED | Forensics: full section bodies, grouped per member.                                |

- `zz` cycles forward 1 → 2 → 3 → 1. `zZ` cycles backward (2 → 1, 1 → 3).
- Per-section **overrides**: `za` cycles the section currently at the top of the viewport forward one level; `zA`
  toggles it between COLLAPSED and FULLY*EXPANDED (mirroring the ChangeSpec
  `cycle*_`/`toggle__`split). Overrides are stored per section id in a`FoldStateManager`-style registry; a section's *effective* level = its override if set, else the panel level. Cycling the panel level (`zz`/`zZ`)
  clears all overrides — the level is authoritative.
- State lives on the app like the ChangeSpec fold reactives: one reactive `FoldLevel` for the panel plus the override
  registry. Session-only, not persisted; defaults to level 1 at startup.

**Why level 2 looks the way it does.** Level 1 answers "who is in this clan and how are they doing"; level 3 answers
"show me everything". Level 2 must earn its place between them: it is the level where the user triages the whole clan in
one screenful without opening any member. Every section renders one truncated line per entry — who failed and on what
(first error line per failed member), what each member concluded (inbox-style first-line reply previews), what they
produced (`member.var = value` one-liners), and what they touched (deduped context-lane digests). No multi-line bodies
appear at level 2; everything is width-truncated single lines with bounded entry counts ("+K more" tails where needed).

### Clan summary sections

The clan summary keeps its current header block and gains one aggregate section for every section kind represented in at
least one member (member body-section kinds are grouped into canonical clan sections; the member-level distinction is
kept as a per-entry label). A section that no member exhibits is omitted entirely. Order (triage-first):

1. **MEMBERS** (`section_id="members"`) — the existing member table (label · kind · status glyph · model · duration,
   families with `├─`/`└─` nesting). Level 1: as today. Level 2: adds a per-member annotation with `activity` / wait /
   retry info when present. Level 3: adds timestamps and attempt-history counts. This is the one section that is _never_
   reduced to a bare heading — it is the "unfolded by default" section at the top.
2. **ERRORS** (`section_id="errors"`) — present when any member has `error_message` or a failed status. L1: `ERRORS · N`
   heading. L2: one line per failed member — member suffix + first line of its error. L3: full `error_message` +
   `error_traceback` per member (reuse the existing traceback `Syntax` rendering), each under a member sub-label.
3. **OUTPUT VARIABLES** (`section_id="output-variables"`) — union of member `output_variables`. L1: heading with total
   variable count. L2: `member.var = value` one-liners. L3: full values, grouped per member. A WORKFLOW VARIABLES
   aggregate renders analogously when any member has them.
4. **REPLIES** (`section_id="replies"`) — aggregate of member AGENT REPLY / AGENT CHAT / STEP OUTPUT content (entry
   label notes the kind for steps). L1: heading with reply count. L2: per-member one-line preview (first meaningful
   line). L3: full bodies with member sub-headings, reusing the regular panel's reply rendering/truncation conventions.
5. **SASE CONTEXT** (`section_id="context"`) — member context lanes aggregated per lane in the existing lane order
   (BEAD, PLAN, ARTIFACTS, MEMORY, SKILLS, WORKSPACES; see `_agent_context.py`). L1: heading. L2: one digest line per
   non-empty lane — deduped bead ids with titles, deduped plan paths, artifact totals with a bounded name list,
   memory-read paths, skill names with use counts, workspace numbers. L3: full per-lane detail reusing existing lane
   renderers where practical.
6. **SLOW TOOL CALLS** (`section_id="slow-tool-calls"`) — L1: heading + count. L2: top ~5 slowest across the clan, one
   line each (member · tool · duration). L3: all, grouped per member.
7. **PROMPTS** (`section_id="prompts"`) — aggregate of member AGENT PROMPT / AGENT XPROMPT. L1: heading with count. L2:
   per-member first-line preview (this is how a user tells swarm segments apart), with xprompt chips on the line when
   present. L3: full prompts per member.

Presentation polish requirements:

- Collapsed headings carry the ▸/▾/▼ fold glyph vocabulary already used by the ChangeSpec fold badges, plus dim `· N`
  counts, so a level-1 clan page reads as a clean table of contents.
- The clan header block gains a small dim fold indicator (e.g. `Fold: 1/3`) so the current level is always visible; keep
  styling consistent with the existing `#D75FFF` clan identity palette and the field-label style used by
  Name/Tribes/Status.
- Every aggregate section registers a `section_id` anchor so `[`/`]` section navigation cycles through the clan summary.
- Counts on level-1 headings must come only from already-in-memory row state; if a count would require disk (e.g. slow
  tool calls before enrichment has run), render the heading without a count rather than blocking.

### Keymaps

- **Zoom migrates `z` → `Z`**: change `zoom_panel` in `src/sase/default_config.yml`, the Textual `Binding` in
  `src/sase/ace/tui/bindings.py`, the help-modal text (`src/sase/ace/tui/modals/help_modal/agents_bindings.py`), and
  command availability metadata. There is no existing top-level `Z` binding, so this is collision-free
  (`fold_mode.keys.toggle_all: "Z"` is inside the chord and unaffected).
- **Fold mode opens on the Agents tab**: drop the `start_fold_mode`-on-agents suppression in `check_action` (keep
  `zoom_panel` gated to the Agents tab). `_handle_fold_key` branches on `current_tab == "agents"` and consults a new
  nested `agents` sub-map under `ace.keymaps.modes.fold_mode.keys`:

  ```yaml
  modes:
    fold_mode:
      prefix: "z"
      keys:
        # existing ChangeSpec keys unchanged
        agents:
          cycle_level: "z" # zz — panel level 1→2→3→1
          cycle_level_back: "Z" # zZ — panel level back (2→1, 1→3)
          cycle_section: "a" # za — cycle current section forward
          toggle_section: "A" # zA — toggle current section collapsed↔fully expanded
  ```

  This keeps one fold mode, one prefix, and one config group starting with `z`, with ChangeSpec fold behavior untouched
  on other tabs.

- Fold-mode **footer hints** on the Agents tab show the agents sub-map (existing FOLD mode-label mechanism). Fold
  actions are global, so per the ace UI conventions they belong in the help modal, not the conditional-keymap footer;
  the transient FOLD mode hints are the established exception.
- **Command palette**: extend the generated fold commands (`src/sase/ace/tui/commands/`) so the agents fold actions are
  reachable by palette on the Agents tab, mirroring how ChangeSpec fold keys are exposed today.
- Update the default keymap config in `src/sase/default_config.yml` for every new or changed key (project convention),
  and keep the `?` help popup in sync (57-char box format rules in `src/sase/ace/tui/modals/help_modal/`).

### Behavior on non-clan selections

Fold state is panel-wide, but in this epic only the clan summary implements fold contracts. Cycling folds while a
regular agent is selected updates the state and shows a subtle toast noting that fold levels currently shape clan
summaries. Regular-agent section contracts are explicitly future work — the state model, keymaps, and dispatch built
here must not assume clan-only forever (no clan-specific naming in the generic layers).

### Performance rules (binding, from the TUI perf conventions)

- The **level-1 render stays pure and synchronous** — exactly as cheap as today's clan panel plus in-memory counts. No
  stat/glob/read in any render path.
- Disk-backed section content (replies, prompts, context lanes, slow tools) loads through a **clan aggregation worker**
  integrated into the existing async display path in `_agent_display.py` / `_agent_display_render.py` (replace the
  current "cancel workers and return" clan branch). Load off-thread, only for sections whose effective fold level
  requires content, marshaled back to the UI thread; re-read the current tab and selected identity after every await
  before applying results; coalesce with the existing loading/pending guards; route repaints through the existing
  `DetailPanelDebouncer` path. Sections awaiting content render a dim `loading…` placeholder line.
- Cache per-member loaded content keyed by member identity + source mtime so fold-level cycling re-renders from cache
  instead of re-reading N member files on every `zz`.
- j/k key-to-paint p95 < 16 ms must hold on the Agents tab with a clan selected at every fold level (verify with
  `SASE_TUI_PERF=1` and the stall watchdog).

## Phase details

### Fold keymaps and panel fold state

Deliver the full input surface and state model with no clan-rendering changes yet:

- Migrate `zoom_panel` to `Z` (config, binding, help modal, command availability) and verify zoom still works only on
  the Agents tab.
- Allow `start_fold_mode` on the Agents tab; add the nested `agents` sub-map to `FoldModeKeymaps` defaults,
  `default_config.yml`, and the loader's canonicalization/merge path; extend `_handle_fold_key` with the agents branch
  dispatching `cycle_level` / `cycle_level_back` / `cycle_section` / `toggle_section`.
- Add the panel fold state model: a reactive panel `FoldLevel` plus a per-section override registry with the
  effective-level rule and clear-on-level-cycle semantics (reuse `FoldLevel`/`cycle_forward`; add the backward cycle
  helper).
- Resolve "current section" for `za`/`zA` from the section-anchor cache the `[`/`]` navigation already uses
  (top-of-viewport anchor).
- FOLD footer hints on the Agents tab, palette commands, and the non-clan toast.
- Tests: keymap default/registry loading (extend `tests/test_keymaps_defaults.py` /
  `tests/test_keymaps_registry_loading.py`), fold dispatch on the Agents tab vs ChangeSpec tabs, level cycle
  forward/backward incl. wrap, override set/clear semantics, zoom-on-`Z` regression.

### Clan section aggregation layer

Deliver the data layer, independent of keymaps:

- Pure aggregation functions (new module beside `_agent_clan.py`) that compute, from a clan container's member rows
  alone: error entries (member, message first-line, full message/traceback), output/workflow variable entries,
  per-member digest annotations for MEMBERS (activity/wait/retry), and level-1 heading counts. Pure, deterministic,
  unit-testable.
- An async aggregation path for disk-backed content: per-member reply bodies/previews, prompts, context lanes
  (bead/plan/artifacts/memory/skills/workspaces), and slow tool calls — reusing the existing per-agent loaders, with the
  mtime-keyed cache, identity re-checks, and coalescing guards described above. Expose a single snapshot object the
  renderer can consume without knowing where each field came from.
- Tests: unit tests for every aggregator (including dedup, ordering, truncation bounds, families contributing their
  member rows), cache-hit behavior, and stale-selection discard.

### Fold-aware clan summary rendering

Deliver the visible feature:

- Extend `build_clan_detail_text` (and the clan branch of the display path) to take the fold state and aggregation
  snapshot and render the section list above, honoring each section's three-level contract; each section builder
  documents its contract in its docstring like `commits_builder.py` does.
- Headings with fold glyphs and counts, member sub-labels, `loading…` placeholders, the header fold indicator, and
  `section_id` anchors for all sections.
- Tests: section-builder unit tests pinning level contracts; section-navigation test extension covering the new anchors;
  PNG snapshots of the clan panel at levels 1, 2, and 3 for both the epic-shaped and swarm-shaped fixtures in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` (the two existing `agents_clan_panel_*_120x40.png` goldens
  will change because level 1 now shows collapsed section headings — regenerate deliberately with
  `--sase-update-visual-snapshots` and eyeball them).

### Docs, help sync, and end-to-end fold exercises

- Update the user-facing clan documentation under `docs/` (added by the sase-6n docs commit) and the `?` help popup for
  the new fold group and the `Z` zoom key. Do **not** touch `sase/memory/*.md`, `AGENTS.md`, or generated provider
  instruction shims — memory edits require explicit user permission and are out of scope for this epic.
- End-to-end exercise: launch a real small clan (e.g. a two-member swarm), select the clan row, cycle `zz` through all
  three levels, use `za`/`zA` on individual sections, confirm `zZ` steps back, confirm `Z` zooms, and confirm ChangeSpec
  folding on the Artifacts/PRs sub-tab is unchanged.
- Verify perf: `SASE_TUI_PERF=1` j/k sampling on the Agents tab with a clan selected at each level; check
  `~/.sase/logs/tui_stalls.jsonl` stays clean during fold cycling.
- Confirm all visual goldens are updated and `just check` passes.

## Risks and mitigations

- **Panel bloat / slow renders at level 3** for large clans: bounded by per-member caching, the debouncer, and reusing
  the regular panel's truncation conventions for long bodies. The pure level-1 path guarantees the default experience
  never regresses.
- **`za` targeting the wrong section** when anchors are mid-relayout: reuse the existing anchor retry machinery
  (`queue_section_retry`) rather than resolving anchors ad hoc; if resolution is not ready, the keypress is a no-op
  rather than folding the wrong section.
- **Keymap regressions**: `z` gains behavior on the Agents tab and zoom moves — covered by dedicated dispatch tests and
  a zoom regression test; ChangeSpec fold tests guard the other tabs.
- **Fold-state/render races** (level changes mid-load): effective levels are read at render time from current state,
  loads are keyed by member identity and discarded on selection change, and repaints go through the existing debouncer.
