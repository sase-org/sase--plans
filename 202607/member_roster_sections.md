---
tier: epic
title: Numbered member rosters for clan and family detail panels
goal: 'Clan and family container rows open with a visually distinct, numbered CLAN
  MEMBERS / FAMILY MEMBERS roster whose numbers double as one-keystroke jump keys,
  whose entries fold individually through the panel fold keymaps, and every section
  of the family detail panel responds to all three fold levels.

  '
phases:
- id: roster-core
  title: Shared numbered roster section and clan panel adoption
  depends_on: []
  description: '''Phase 1: Roster core'' section: build the shared numbered member-roster
    renderer with per-member fold anchors and jump-target publication, and adopt it
    in the clan detail panel as CLAN MEMBERS.

    '
- id: family-panel
  title: FAMILY MEMBERS section and fold-aware family detail
  depends_on:
  - roster-core
  description: '''Phase 2: Family panel'' section: render the FAMILY MEMBERS roster
    at the top of the family container detail and make every family detail section
    respond to all three fold levels.

    '
- id: digit-jump
  title: Digit-key member jump navigation
  depends_on:
  - roster-core
  - family-panel
  description: '''Phase 3: Digit jump'' section: capture 0-9 on selected clan/family
    rows, buffer two-digit input, reveal the target by expanding enclosing folds,
    and select it; update footer and help modal.

    '
- id: visual-polish
  title: PNG snapshot coverage and end-to-end polish
  depends_on:
  - digit-jump
  description: '''Phase 4: Visual snapshots and polish'' section: add and refresh
    PNG snapshot goldens for the new roster and family fold levels, exercise the full
    jump/fold flows end to end, and reconcile footer/help formatting.

    '
create_time: 2026-07-18 17:47:19
status: done
bead_id: sase-6w
---

# Plan: Numbered member rosters for clan and family detail panels

## Context and product goals

The `sase ace` Agents tab has two kinds of container rows: **agent clans** (synthetic containers projected over parallel
members) and **agent families** (the renamed root agent of a sequential chain, presented under the bare family name).
Selecting a clan container already renders a dedicated fold-aware detail document
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`) whose first section is `MEMBERS`. Selecting a family
container renders the plain regular-agent path (`_agent_display_render.py`): header fields, then `AGENT XPROMPT` /
`AGENT PROMPT` / `AGENT REPLY`, with no member overview and no fold-level support.

This plan delivers one coherent feature with three intertwined parts:

1. **A numbered member roster as the very first metadata section** — `CLAN MEMBERS` on clan containers (upgrading
   today's `MEMBERS`) and a new `FAMILY MEMBERS` on family container rows. The section is styled so its mere presence
   signals "you are looking at a container", and each entry carries a visible number chip.
2. **Numbers that are keymaps.** Pressing the number shown next to a member jumps the agent-list cursor to that member
   (agent or nested family), expanding the enclosing clan/family/group folds as needed. One digit when the roster has ≤
   10 entries (`0`–`9`); two digits (`00`–`99`) beyond that; at most 100 numbered entries with an explicit truncation
   tail.
3. **Deep fold support.** Roster entries fold individually through the existing panel fold keymaps (`z z`, `z Z`, `z a`,
   `z A`), and — since this is the first foldable section on family rows — every section of the family detail document
   gains meaningful rendering at all three fold levels.

Everything here is presentation-only Textual/Rich state, keybinding dispatch, and Python glue, so per the Rust core
backend boundary it stays entirely in this repo; no `sase-core` changes.

## Design overview

Target rendering for a clan container (family container is identical in structure, with the family accent color and
phase-role kinds):

```
Name: mapreduce
Tribes: @research
Status: RUNNING ●3 ✓2
Runtime: 12m 4s
Members: 6 agents · 1 family
Fold: 1/3

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← accent rule, clan color
▸ ❖ CLAN MEMBERS · 6                                ← identity-colored banner
 0  .fan.1 · agent · ● Running · sonnet · 4m 12s
 1  .fan.2 · agent · ✓ Done · sonnet · 3m 58s
 2  .reduce · family · ● Running · mixed · 8m 01s
    ├─ --plan · agent · ✓ Done · opus · 2m 10s
    └─ --code · agent · ● Running · sonnet · 5m 51s
 3  .audit · agent · ⏸ Waiting · haiku · —
 …
──────────────────────────────────────────────────
▸ ERRORS · 1
…
```

Key structural decisions:

- **Placement.** The roster is the first _section_ of the document, rendered immediately after the header field rows
  (Name/Status/…) and before every other section (`ERRORS`, `BEAD`/`ARTIFACTS`, `AGENT XPROMPT`, …). For clans this
  keeps the current position of `MEMBERS`; for families it is inserted ahead of all existing sections.
- **Membership and order.** Clan rosters number **direct members only** (agents and families, in the existing
  deterministic launch order of `_ordered_members`); a family's own sequential rows still render indented beneath its
  entry (as today) but are _not_ numbered — you jump to the family, then use its `FAMILY MEMBERS` roster. Family rosters
  number the real chain rows (the renamed first member plus each subsequent family member, mirroring `_family_rows`) in
  chain order. Launch/chain order is deliberately chosen over the status-sorted agent-list order because it is stable:
  numbers must not reshuffle under the user's fingers as statuses change.
- **Numbering regimes.** `count <= 10` → single digits `0`–`9`; `11 <= count <= 100` → zero-padded `00`–`99`. Beyond 100
  entries the roster stops numbering and appends a dim italic tail: `… +N more members (not numbered)`. The regime is
  chosen per roster from its total entry count.
- **One source of truth for number → target.** The roster builder returns (and the render path publishes on the app) an
  ordered jump map: container identity plus a tuple of `(number, member identity, kind)` records. The digit-key handler
  consumes only this published map and validates the selected row's identity against it before acting, so the numbers on
  screen and the keys pressed can never drift apart.

## The roster section (shared builder)

Add a shared roster renderer (new module under `src/sase/ace/tui/widgets/prompt_panel/`, e.g. `_member_roster.py`) used
by both the clan and family paths. Inputs: the container agent, the ordered member entries, per-entry digest info
(activity/wait/retry, workspace, timestamps, attempt counts — reusing `ClanMemberDigest` and `aggregate_clan_in_memory`
for clans, and an equivalent in-memory digest for family rows), the effective fold levels, and an accent theme (clan
`#D75FFF` / family `#00AFFF`, matching `_agent_list_styling`).

Roster entry content per fold level (matching today's `MEMBERS` semantics):

- **COLLAPSED** — the complete one-line table row: number chip, hood/chain suffix name, kind, status glyph + word,
  model, runtime. The full member table is always visible; folding never hides the numbers.
- **EXPANDED** — adds the activity / `wait …` / `retry …` annotations.
- **FULLY_EXPANDED** — additionally includes workspace, start/run/stop timestamps, and attempt counts.

Member kinds: clans reuse `agent` / `family` / `step`; family rosters show the phase role via `get_phase_label`
(`PLANNER`, `CODER`, `QUESTIONS`, …) falling back to `agent`, which makes plan-chain families read beautifully.

**Fold anchors.** The roster heading keeps section id `members` (stable for existing overrides and tests). Each entry
additionally registers its own section anchor (id `member:<presented member name>`) through `append_section_heading`'s
marker mechanism, so the existing `_current_agent_metadata_section_id` viewport-top resolution lets `z a` / `z A` cycle
or toggle a single member's detail. Effective level resolution is nested inheritance: member override → roster
(`members`) override → panel level. `z z` / `z Z` continue to cycle the base panel level and clear overrides, exactly as
today.

**Clan adoption.** `build_clan_detail_text` swaps its `_append_members_section` for the shared builder, retitling the
heading `CLAN MEMBERS` (section id stays `members`), and publishes the jump map. Count chip and the "COLLAPSED keeps the
complete member table" behavior are preserved. The legacy archived-parallel path in `_agent_display_header.py`
(`_append_legacy_parallel_members_section`) is left untouched.

## Phase 1: Roster core

Deliver the shared roster renderer and its clan adoption:

- New roster module: numbering regimes, truncation tail, number chips, per-entry fold levels, nested fold inheritance,
  per-entry section anchors, accent theming, and the returned jump-map records.
- Clan detail panel renders `CLAN MEMBERS` through it; the render path publishes the jump map on the app keyed by
  container identity (pure in-memory, UI-thread, no disk).
- Unit tests (extend `tests/ace/tui/widgets/test_agent_display_clan.py` plus a new roster test module): numbering at 1,
  10, 11, 100, and 150 members; truncation tail wording; order stability while statuses churn; per-member fold override
  rendering at all three levels; jump-map contents matching the rendered numbers; nested family sub-rows remaining
  unnumbered.

## Phase 2: Family panel

Make family container rows (`Agent.is_family_container_row`) first-class:

- **FAMILY MEMBERS roster** rendered as the first section after the header field rows, via the shared builder with the
  family accent, publishing its jump map the same way.
- **Fold-aware family sections.** Thread `panel_fold_level` + `_panel_fold_overrides` (via the existing
  `clan_fold_state_from_widget` helper, suitably renamed/generalized) into the family render path in
  `_agent_display_render.py`, giving every section three meaningful levels:
  - `AGENT XPROMPT` and `AGENT PROMPT`: COLLAPSED → heading plus a one-line digest (first meaningful line + total line
    count); EXPANDED → a bounded preview (roughly the first dozen lines with a `+N more lines` tail); FULLY_EXPANDED →
    the full content exactly as today.
  - `AGENT REPLY`: COLLAPSED → one line per phase (phase label, status glyph, start time); EXPANDED → per-phase tail
    previews (last meaningful lines of each phase); FULLY_EXPANDED → the full multi-phase stream as today.
  - Header summary sections that appear for the root (ARTIFACTS, OUTPUT VARIABLES, MEMORY READS, SKILL USES, SLOW TOOL
    CALLS, DELTAS, COMMITS): COLLAPSED → heading + count only; EXPANDED/FULLY_EXPANDED → their existing content. Keep
    this mapping shallow — no new per-section designs.
- **Deliberate behavior change:** with the panel default of `COLLAPSED`, a selected family now opens as a compact triage
  overview (numbered roster + digests) instead of the full prompt/reply dump — consistent with how clan containers
  already behave, with the full view two `z z` presses (or one `z A` on a section) away, and each member's full detail
  one digit away.
- At COLLAPSED, skip the prompt/reply disk reads entirely (they are only needed once a section actually renders content)
  — a small but real render win on the j/k path.
- Regular non-family agents are explicitly unchanged; update `_notify_regular_agent_fold_scope` so its "fold levels
  shape clan summaries" notice no longer fires for family containers (which now respond) and its wording covers both
  container kinds.
- Unit tests: every family section at all three levels; override + panel level interplay; the fold-scope notification
  exclusion; family-in-clan and standalone family rosters.

## Phase 3: Digit jump

**Dispatch.** In `EventKeyboardMixin.on_key`, after the existing capture-mode branches: when the Agents tab is active,
no capture mode is engaged, and the selected row is a clan container or family container row, digit keys are routed to a
new member-jump handler (an `actions/navigation/` mixin). Digits are **fixed hint labels**, not configurable keymap
entries — the same precedent as `JUMP_HINT_CHARS` — because the label rendered in the roster and the key pressed must be
the same thing by construction. No `default_config.yml` changes are needed for the digits themselves.

- Single-digit regime: the press jumps immediately; a digit with no entry (e.g. `8` in a 7-member roster) cancels with a
  brief notify.
- Two-digit regime: the first digit is buffered; the footer shows a pending indicator
  (`member 1▁ · 0-9 second digit · esc cancel`); a second digit completes the jump. Escape cancels; **any other key
  cancels the pending digit and is then processed normally** (forgiving, never blocks j/k); selection changes and list
  refreshes also clear the buffer.
- The handler validates that the published jump map still belongs to the currently selected container identity and that
  the target identity still exists; if the list changed underneath, it cancels with a notify instead of jumping
  somewhere stale (the "re-capture state after every await" rule).

**Reveal-and-select helper.** A single robust `_reveal_agent_row(identity)`-style helper performs the navigation:

1. Push the current cursor through `_save_agents_jump_anchor` so `'` back-jump returns to the container.
2. Iteratively make the target visible: expand the enclosing collapsed panel (`_expand_agent_panel`) and collapsed group
   banners (group fold registry) if needed, then step the clan/family fold keys up via `_fold_manager.expand` until the
   target row appears — clan → direct member needs the clan fold at EXPANDED; family-member rows need the deeper level
   (family fold key, or the clan fold's FULLY_EXPANDED step for families nested in clans). Implement as "expand the
   ancestor chain until the identity is present in `self._agents`, bounded by the chain length", so both nesting shapes
   and future shapes work without special cases.
3. `_refilter_agents()` (the existing fast path — no new refresh code paths), locate the row by identity, set
   `current_idx`, clear `_current_group_key`, and let the existing selection flow refresh the detail panel through its
   debouncer.

**Surfaces.** Per the ace footer convention this binding is conditional (only when a container row is selected), so add
`0-9 member` to the agents-tab footer bindings and the pending-digit indicator state; add the member-jump rows and the
roster fold behavior to the help modal's Agents box (respecting the 57-character box width rules).

Tests: single-digit immediate jump; two-digit buffering and completion; escape / other-key / selection-change
cancellation; jump from a fully collapsed clan expands and selects; jump to a family entry from a clan; jump to a family
member for both standalone and in-clan families; back-jump anchor restores the container; stale-map cancellation;
footer/help content checks in the existing keybinding tests.

## Phase 4: Visual snapshots and polish

- PNG snapshot coverage (`tests/ace/tui/visual/`, goldens under `tests/ace/tui/visual/snapshots/png/`): refresh
  `test_ace_png_snapshots_agents_clan_panel.py` goldens for the retitled numbered roster, and add family-panel snapshots
  covering the three panel fold levels and a per-member override, plus a two-digit-regime roster and the pending-digit
  footer. Use `just test-visual` and accept intentional changes with `--sase-update-visual-snapshots`.
- Exercise the flows end to end in the real TUI (launch, select clan, digit jump, `'` back-jump, `z` folds on family
  sections) and fix any rough edges found (spacing, chip alignment at both digit widths, wrap behavior at narrow panel
  widths).
- Reconcile the footer and help modal against the ace conventions (sorted ordering, box widths), and run the full
  `just check` gate.

## Visual design notes

- **Banner headings.** The roster heading is the only section heading drawn in the container's identity color with a
  full-width heavy accent rule (`━`) above it — clans magenta `#D75FFF`, families azure `#00AFFF` — plus the existing
  fold glyph (`▸ ▾ ▼`) and a dim ` · count` chip. A small leading glyph (e.g. `❖`) further distinguishes it from
  ordinary gold headings. Every other section keeps the standard dim rule, so the roster reads instantly as "container
  overview".
- **Number chips.** Numbers render as key-cap chips: bold black text on the identity color (reverse video), one cell of
  padding, constant width within a roster (one or two digits). They look pressable because they are.
- **Truncation tail** and the `+N more` idiom reuse the existing dim-italic styling so the roster degrades gracefully at
  100+ members.
- Colors are fixed hex values in the existing style-constant modules (`_agent_list_styling.py` / roster module
  constants), consistent with the pinned-color visual test fixtures.

## Reliability and performance guardrails

Per the TUI performance rules:

- Roster building, jump-map publication, and digit handling are pure in-memory computations over already-loaded rows —
  no disk I/O, no subprocesses, nothing on the keystroke path beyond dict/tuple work.
- Navigation reuses `_refilter_agents()` and the existing debounced detail refresh; no new refresh paths, no full-list
  rebuild beyond what fold changes already do.
- Fold-level gating in the family path _removes_ work at the default level (skipped prompt/reply reads) rather than
  adding any.
- All stale-state hazards (refresh between render and keypress, selection moved during pending digit) resolve by
  validating identities against the published map at press time and cancelling gracefully.

## Risks and mitigations

- **Family default-view change** is the one user-visible behavior shift (compact overview instead of full prompt/reply).
  It is intentional and symmetric with clans; the plan calls it out so reviewers see it, and the full view remains two
  keystrokes away. If it proves wrong, the fallback is a one-line default change (treat family panels' base level as
  FULLY_EXPANDED) without touching the architecture.
- **Number drift** between screen and keys is prevented structurally (the renderer publishes the map the handler
  consumes).
- **Nesting-shape surprises** (standalone family vs family-in-clan vs grouped banners vs collapsed panels) are handled
  by the iterative reveal helper and covered by dedicated tests for each shape.

## Out of scope

- `z <digit>` (fold a specific member by number) — a natural follow-up once digit capture exists, but it doubles the
  fold-mode state machine; not now.
- Numbering nested family sub-rows inside a clan roster (jump to the family first instead).
- Fold-level support for regular non-family, non-container agents.
- Any persistence of roster fold overrides or jump maps across sessions.
- Any `sase-core` (Rust) changes — this feature is presentation-only.
