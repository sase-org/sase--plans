---
tier: epic
title: Smart folding for agent clan/family/tribe summaries
goal: 'Agent clan, family, and tribe summary documents in the Agents-tab metadata
  panel hide empty sections at every fold level, always show their numbered member
  rosters with working digit jump triggers, and give each fold level a distinct, useful
  answer — rendered with one consistent, beautiful fold language across all three
  kinds.

  '
phases:
- id: tribe-doc
  title: Tribe document rework and shared fold language
  depends_on: []
  description: '''Phase 1 — Tribe document rework and shared fold language'' section:
    introduce the shared fold-language and scanning-tail helpers, render the tribe
    roster and digit jump map at every fold level, omit empty tribe sections, extend
    tribe presence enrichment to all fold levels, and rebalance the tribe fold ladder.'
- id: clan-family-docs
  title: Clan and family document rework
  depends_on:
  - tribe-doc
  description: '''Phase 2 — Clan and family document rework'' section: replace clan
    loading-stub disk sections with the shared scanning tail, polish clan and tribe-facing
    header lines, omit empty family xprompt/prompt sections, and migrate clan and
    family rendering onto the shared fold language.'
- id: consistency-polish
  title: Cross-kind consistency, goldens, and help sync
  depends_on:
  - clan-family-docs
  description: '''Phase 3 — Cross-kind consistency, goldens, and help sync'' section:
    add parameterized cross-kind fold contract tests, regenerate the remaining PNG
    goldens with updated snapshot titles, exercise member-jump flows end to end, and
    sync the help modal copy.'
create_time: 2026-07-19 08:02:43
status: done
bead_id: sase-73
---

# Plan: Smart folding for agent clan/family/tribe summaries

## Context

The Agents tab of the `sase ace` TUI renders three fold-aware summary documents in the agent metadata panel:

- **Clan** — `build_clan_detail_text` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py` (+ section
  bodies in `_agent_display_clan_sections.py`, roster adaptation in `_agent_display_clan_roster.py`), 3-level fold
  scale.
- **Family** — family branches of `build_header_text` in `_agent_display_header.py` and `_update_family_display` in
  `_agent_display_render.py` (+ helpers in `_agent_display_family.py`), 2-level fold scale.
- **Tribe** — `build_tribe_detail_text` in `_agent_display_tribe.py` (+ off-thread enrichment in
  `_agent_tribe_aggregation.py`, pure snapshot in `src/sase/ace/tui/models/agent_tribe_summary.py`), 4-level fold scale.

All three share the numbered member roster in `_member_roster.py`, the fold scales in
`src/sase/ace/tui/models/fold_scale.py`, and the fold-mode key handling in
`src/sase/ace/tui/actions/navigation/_fold.py` (`z` leader; `z`/`Z` cycle the panel level, `a`/`A` cycle/toggle the
section under the cursor). Rosters publish digit jump maps consumed by
`src/sase/ace/tui/actions/navigation/_member_jump.py`, and the footer conditionally advertises `0-9 member` for
container rows and focused panels (`src/sase/ace/tui/widgets/_keybinding_bindings.py`).

### Problems with the current documents

1. **Empty sections render everywhere.** The tribe document prints `NEEDS ATTENTION` with "No failed, stopped, or
   waiting units.", `ERRORS · 0`, `OUTPUT VARIABLES · 0`, `WORKFLOW VARIABLES · 0`, `REPLIES · —`,
   `SLOW TOOL CALLS · —`, and a `RUNTIME STATISTICS` section whose body is a lone em dash. Most of a typical tribe
   document is placeholder noise.
2. **Members are hidden at the shallowest tribe fold.** The tribe roster passes `heading_only_when_collapsed=True`, so
   at fold 1 only the `TRIBE MEMBERS · N` heading renders. The footer still advertises `0-9 member` for the focused
   panel, but pressing a digit answers "Member roster is not ready" — a real UX bug.
3. **Loading stubs flicker.** The clan document paints four disk-section stubs (`REPLIES`, `SASE CONTEXT`,
   `SLOW TOOL CALLS`, `PROMPTS`) with "loading…" bodies, then deletes the ones that turn out empty. Headings that vanish
   are jarring; sections that appear once they have content are not.
4. **Placeholder filler.** The family document prints "No xprompt file found." / "No prompt file found."; headers carry
   `Tribes: —`, `Panel: collapsed`, and zero-valued composition components ("… · 0 families", "· 0 nested").
5. **Duplicated fold language.** `_FOLD_CHARS`/`_FOLD_STYLES` tables are copied across `_member_roster.py`,
   `_agent_display_tribe.py`, `_agent_display_family.py`, and `_agent_display_clan_sections.py`; the clan hard-codes its
   `Fold: N/3` header line while family hard-codes `/2`.

This is presentation-only Textual rendering: no Rust core (`sase-core`) changes are needed, and no wire/API surface
moves.

## Design principles

- **P1 — Empty never renders.** A section with zero contents is omitted at _every_ fold level: no `· 0` headings, no `—`
  bodies, no "No failed…" filler. Fold levels change how much of _existing_ content is shown, never whether empty
  scaffolding appears.
- **P2 — Members always visible, always jumpable.** Every roster renders its numbered entries (with the highlighted
  digit chips) at every fold level, and publishes its jump map so the advertised `0-9 member` binding always works. Fold
  level controls per-member density (annotations, nested children), never member visibility.
- **P3 — Unknown is not empty.** Disk-backed sections whose presence has not been determined yet render nothing; instead
  a single dim, italic tail line — `⋯ scanning member data…` — at the end of the document says enrichment is in flight.
  When results land, non-empty sections appear with real counts; empty ones never appear. Pending-content states that
  carry meaning (e.g. a running family member's "Waiting for agent response…") are kept — they are signal, not filler.
- **P4 — Each fold level answers a distinct question.** Glance → triage → inspect → forensics (see the ladder below). No
  two adjacent levels of one section may render identical content.
- **P5 — One fold language.** Fold glyphs, glyph styles, `Fold: N/M` header lines (computed from `fold_scale_position`,
  never hard-coded), and the scanning tail all come from one shared presentation module used by all three documents and
  the shared roster.

## The fold ladder

| Level | Name      | Tribe (4)                                                                             | Clan (3)                                                         | Family (2)                                     |
| ----- | --------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------- |
| 1     | glance    | header · compact roster · attention previews · non-empty section headings with counts | header · compact roster · non-empty section headings with counts | — (family scale starts at level 2's semantics) |
| 2     | triage    | + bounded previews in every non-empty section                                         | + bounded previews                                               | header · roster · bounded previews             |
| 3     | inspect   | + roster children · full bodies grouped by unit/member (triage bounds still apply)    | + full bodies grouped by member · full roster annotations        | full bodies · full roster annotations          |
| 4     | forensics | + no bounds · error tracebacks · full roster annotations · runtime statistics         | —                                                                | —                                              |

"Compact roster" means one line per numbered entry: digit chip, relative name, kind, status glyph + status, model,
duration — no digest annotations, no nested children. Fold scales themselves (`CLAN_FOLD_SCALE`, `FAMILY_FOLD_SCALE`,
`TRIBE_FOLD_SCALE`) and the fold-mode keys do not change; this plan changes what each level shows.

## Behavior spec by document

### Shared fold language (new module)

Add a small shared presentation module under `src/sase/ace/tui/widgets/prompt_panel/` (e.g. `_fold_language.py`) owning:

- the fold glyph/style tables (`▸ ▾ ▼ ◆` and their colors), replacing the four copies;
- a `Fold: N/M` header-line helper driven by `fold_scale_position` for the kind's scale (fixes clan's hard-coded `/3`
  and family's hard-coded `/2`);
- the scanning-tail helper that appends `⋯ scanning member data…` (dim italic) when any required-but-unloaded enrichment
  exists;
- an "attention count" heading style rule: headings whose content is a problem signal (`NEEDS ATTENTION`, `ERRORS`)
  render their count in the failure accent instead of dim, so a glance-level document telegraphs trouble.

To keep Symvision clean, each helper lands in the same phase as its first consumer.

### Tribe document (`_agent_display_tribe.py`)

- **Roster.** Drop `heading_only_when_collapsed=True` and `show_empty_heading=True` from the `append_member_roster`
  call; numbered units render at all four levels. Children stay gated to level 3+
  (`children_from_level=FULLY_EXPANDED`), full annotations to level 4 (`full_annotations_from_level=EXHAUSTIVE`). The
  jump map is therefore non-empty at every level, including collapsed panels — digit jumps work wherever the footer
  advertises them.
- **NEEDS ATTENTION.** Omit entirely when there are no entries. When present, keep bounded previews at every level (it
  is the highest-value signal), unbounded at level 4.
- **ERRORS / OUTPUT VARIABLES / WORKFLOW VARIABLES.** Omit when empty. Ladder: level 1 heading + count; level 2 bounded
  previews; level 3 full bodies grouped by member (bounds apply); level 4 unbounded, with error tracebacks.
- **REPLIES / SLOW TOOL CALLS.** Render only once loaded _and_ non-empty; the scanning tail covers the unknown state.
  Ladder: level 1 heading + count; level 2 bounded previews (moved up from level 3 — content is already loaded by
  presence enrichment); level 3 full grouped bodies with bounds; level 4 unbounded.
- **RUNTIME STATISTICS.** Level 4 only, as today; omit the section when the archive query returns no data; the scanning
  tail covers loading.
- **Header.** Drop the `Panel:` line (it restates visible panel state). Composition drops zero-valued components ("1
  clan · 3 agents", never "· 0 families · 0 nested"; the agent count always renders). Keep Name, Status + count chip,
  Runtime, and the shared `Fold: N/M` line.
- **Enrichment contract.** `tribe_enrichment_sections_for_fold_state` returns `{"replies", "slow-tool-calls"}` at every
  fold level (presence detection), plus `"runtime-statistics"` at EXHAUSTIVE — mirroring the clan contract in
  `clan_disk_sections_for_fold_state`, whose docstring already establishes the pattern: even a collapsed summary needs
  one off-thread pass to distinguish represented sections from known-empty ones.

### Clan document (`_agent_display_clan.py`, `_agent_display_clan_sections.py`)

- **Disk sections** (`REPLIES`, `SASE CONTEXT`, `SLOW TOOL CALLS`, `PROMPTS`): render only when loaded and non-empty;
  drop the per-section "loading…" stub headings and bodies; append the shared scanning tail while any required section
  is pending. `SASE CONTEXT` keeps rendering its in-memory `minimal_context_lanes` immediately when non-empty (that is
  real content), with the tail indicating more may arrive. The `clan_disk_sections_for_fold_state` contract (always
  request everything) is unchanged.
- **Header.** Omit the `Tribes:` line when the clan has no tribe tags. The `Members:` rollup drops zero-valued
  components ("2 agents", not "2 agents · 0 families"). `Fold: N/3` moves to the shared helper.
- **ERRORS / OUTPUT VARIABLES / WORKFLOW VARIABLES** already hide when empty; unchanged. The roster already renders at
  every level; unchanged.

### Family document (`_agent_display_header.py`, `_agent_display_render.py`)

- **AGENT XPROMPT.** Omit the section entirely (heading included) when the family has no xprompt content, instead of "No
  xprompt file found.".
- **AGENT PROMPT.** Same treatment when there is genuinely no prompt content.
- **AGENT REPLY.** Unchanged in spirit: a phase that has produced nothing yet keeps its "Waiting for agent response…" /
  "No response content yet." state — pending is signal (P3), and the per-phase dividers stay.
- **Header/roster/variables/slow tools** already comply (sections return early when empty); migrate the `Fold: N/2` line
  and fold glyph usage to the shared module.

### Shared roster (`_member_roster.py`)

- Remove the now-unused `heading_only_when_collapsed` and `show_empty_heading` parameters (the tribe was the only caller
  of either) so the roster's contract _is_ P2: entries and jump map at every level.
- Import glyph tables from the shared fold-language module.
- Entry rendering ladder is unchanged: annotations from the kind's configured levels, children from
  `children_from_level`.

### Enrichment and performance

Render paths stay I/O-free (per `sase/memory/tui_perf.md`): presence detection runs on the existing off-thread workers
with mtime-keyed caches. The tribe worker already reuses per-clan cached disk snapshots and holds a 10-second disk TTL
and 60-second stats TTL (`_agent_tribe_aggregation.py`), so requesting presence at every fold level adds at most one
bounded background pass per panel per TTL window — the same cost profile the clan document has carried since
`clan_disk_sections_for_fold_state` adopted always-on presence. Cheap paints (`cheap=True`) still skip enrichment and
jump-map publishing. No new refresh paths are introduced.

## Phase 1 — Tribe document rework and shared fold language

Scope:

- Create the shared fold-language module (glyphs/styles, `Fold: N/M` helper, scanning tail, attention-count heading
  style) with the tribe document and shared roster as first consumers.
- Apply the full tribe behavior spec above: roster at every level, empty-section omission, ladder rebalance, header
  polish, always-on presence enrichment, runtime statistics omission when absent.
- Remove the roster's `heading_only_when_collapsed` / `show_empty_heading` parameters and update `append_member_roster`
  accordingly.

Tests (update in place, add new):

- `tests/ace/tui/widgets/test_agent_display_tribe.py` — invert the members-hidden-at-collapsed assertions; replace
  `loading…`/`· —`/`RUNTIME STATISTICS —` expectations with omission + scanning-tail assertions; extend the four-level
  ladder test to assert each level's distinct content; new tests: empty snapshot renders no ERRORS/variables/attention
  headings at any level; jump map targets non-empty at COLLAPSED.
- `tests/ace/tui/widgets/test_member_roster.py` — parameter removal; roster always renders entries.
- `tests/ace/tui/widgets/test_agent_display_clan.py` and `.../test_agent_display_family.py` — only if the roster/glyph
  consolidation shifts shared output; behavioral clan/family changes belong to Phase 2.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py` — regenerate tribe goldens
  (`--sase-update-visual-snapshots`), inspecting `.pytest_cache/sase-visual/` artifacts to confirm every diff is
  intentional.
- `tests/ace/tui/test_member_jump_navigation.py` — collapsed-panel digit jump now resolves a member instead of "not
  ready".

Done when: the tribe document contains no placeholder sections at any fold level, a collapsed tribe panel supports digit
jumps, and `just check` plus the tribe visual suite pass.

## Phase 2 — Clan and family document rework

Scope:

- Apply the clan behavior spec: disk sections hidden until loaded-and-non-empty, shared scanning tail,
  `Tribes:`/`Members:` header polish, shared `Fold: N/M` helper.
- Apply the family behavior spec: omit empty AGENT XPROMPT / AGENT PROMPT sections, keep pending reply states, migrate
  to the shared fold language.
- Delete the now-unreferenced per-file fold glyph tables.

Tests:

- `tests/ace/tui/widgets/test_agent_display_clan.py` — replace the unknown-disk-headings-at-collapsed and
  four-loading-placeholder expectations with hidden-until-known + scanning-tail assertions; update the `Members:` rollup
  expectation ("2 agents", zero families omitted); add: no `Tribes:` line without tags; disk section appears with count
  once snapshot marks it loaded and non-empty.
- `tests/ace/tui/widgets/test_agent_display_family.py` — xprompt/prompt omission when absent; unchanged pending-reply
  behavior; fold-line via shared helper.
- Visual suites `test_ace_png_snapshots_agents_clan_panel.py` and `test_ace_png_snapshots_agents_family_panel.py` —
  regenerate goldens with the same inspect-before-accept discipline.

Done when: clan and family documents render no loading stubs or filler placeholders at any fold level, all three
documents share one fold-language module, and `just check` plus the clan/family visual suites pass.

## Phase 3 — Cross-kind consistency, goldens, and help sync

Scope:

- Add a parameterized cross-kind contract test module (new file under `tests/ace/tui/widgets/`) asserting, for each kind
  at each level of its scale: (a) zero-content sections never render; (b) numbered roster entries and a non-empty jump
  map always render; (c) the scanning tail appears exactly when required enrichment is unloaded; (d) no two adjacent
  levels of a non-empty section render identical section bodies (P4).
- Regenerate the remaining PNG goldens (`test_ace_png_snapshots_agents_interactions.py` collapsed-panel/hint scenarios)
  and rename the tribe fold snapshot titles from "pulse / roster / members / forensics" to "glance / triage / inspect /
  forensics" so titles match the ladder.
- End-to-end sweeps: fold-mode chords (`tests/ace/tui/test_agents_panel_fold_mode.py`), member-jump flows including
  two-digit rosters and collapsed panels, and section-under-cursor cycling against the leaner documents
  (`test_prompt_panel_section_navigation.py` — hidden sections must simply be absent from navigation targets, not
  error).
- Help modal sync per `src/sase/ace/CLAUDE.md`: review the "Metadata Fold Mode (z)" box in
  `src/sase/ace/tui/modals/help_modal/agents_bindings.py` and the `0-9 Jump to numbered member` entry; update copy if it
  describes the old members-hidden or placeholder behavior, otherwise confirm it is already accurate.

Done when: the contract tests pass for all three kinds, all visual goldens are regenerated and reviewed, help copy
matches behavior, and `just check` passes.

## Testing strategy

Unit-level exact-text tests carry the behavior contract; the PNG suites (`just test-visual`) carry the beauty contract.
Every phase leaves the full suite green (`just check`, after `just install`). Golden updates are accepted only with
`--sase-update-visual-snapshots` after inspecting the actual/expected/diff artifacts in `.pytest_cache/sase-visual/`.

## Risks and mitigations

- **Sections appearing after first paint.** By design (P3): content arrival is additive and calm, unlike today's
  disappearing stubs. The scanning tail explains the gap. Mitigated further by the mtime-keyed caches making repeat
  visits render complete documents immediately.
- **Always-on tribe presence enrichment cost.** Bounded by visible panel rows, off-thread, TTL-gated, and cache-reusing
  — the clan has shipped this exact pattern already. No render-path I/O is added anywhere.
- **Jump-map/registry races** when rosters render at more levels: the member-jump path already revalidates targets
  against the live container (`_current_member_target_is_valid`) and cancels on roster change; Phase 1 keeps the
  publisher wiring untouched apart from removing the collapsed-suppression.
- **Golden churn.** Three visual suites change. Phases regenerate only the suites they affect, and each regeneration is
  inspected diff-by-diff before acceptance.
- **Hidden sections vs. per-section fold overrides.** Overrides for a section that is currently hidden simply have no
  anchor to act on and become effective again when the section gains content; the override store
  (`SectionFoldStateManager`) needs no change. Contract tests in Phase 3 pin this.
- **Symvision unused-symbol lint.** Shared helpers land in the same phase as their first consumer; leftover glyph tables
  are deleted in Phase 2 when the last consumer migrates.
