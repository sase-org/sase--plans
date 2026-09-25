---
tier: epic
title: Show agent bead notes in the Main deck Context card
goal: Agent-authored bead notes appear as readable, correctly attributed previews
  under SASE CONTEXT / ARTIFACTS / Beads, with current text and reliable navigation
  to the complete note log.
phases:
- id: note_index
  title: Project current agent notes in the Rust bead touch index
  description: 'note_index: extend the core touch-index wire and reduction with bounded,
    edit-aware, removal-aware note previews; cover schema rebuilds and the Python
    binding.'
  depends_on: []
  size: medium
- id: context_card
  title: Render note previews in the Context card
  description: 'context_card: carry indexed notes through the Python touch loader
    and render compact, attributed note blocks with accessible overflow and full-detail
    navigation; update docs and visual coverage.'
  depends_on:
  - note_index
  size: medium
proposed_by: bbugyi200.athena.0rx
create_time: 2026-09-25 07:12:27
status: wip
bead_id: sase-18z
---

- **PROMPT:** [prompts/202609/agent_bead_note_previews.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_bead_note_previews.md)
- **BEAD:** [sase-18z](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18z/README.md)

# Agent bead notes in the Context card

## Purpose and current path

The Main deck's `context` card already contains `SASE CONTEXT / ARTIFACTS / Beads`. Its
rows show the five newest beads an agent touched, read, viewed, or owns. Each row has a
timestamp, verb glyph, full bead ID, verb chips, and a continuation containing the
newest audited read reason or the bead title. A numbered hint opens live bead detail in
the pager. The card composition in `widgets/decks/` consumes the existing prompt-panel
document; it needs no new deck or navigation state.

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_summary.py` resolves the
`artifacts` lane off the UI thread. It loads `src/sase/ace/tui/bead_touches.py`, which
reads the cached, Rust-produced touch index through
`src/sase/core/bead_touch_index_facade.py`, then folds rows into `BeadTouchEntry`.
`_agent_artifacts_lane.py` and `_agent_bead_touches.py` paint the result. The current
Rust index (`sase-core/crates/sase_core/src/bead/touch_index.rs`) records `noted` verb
counts but drops note text. The indexed projection is the right source: the render path
must not scan bead streams or invoke `sase bead show` for every selection.

This is separate from the task/phase worker's `SASE CONTEXT / BEAD` lane. That lane
describes the assigned bead; `ARTIFACTS / Beads` covers every bead the selected agent
interacted with. Keep both, and do not copy a whole bead's notes into a row belonging to
an agent who merely read it. Existing ordering, five-bead cap, hint mappings, and
`Reads` exclusion of `bead:` refs remain intact.

The related deck design is `plan:202609/agents_tab_decks_and_cards.md` and its
`sase-17d` epic. Its Context card is the destination, not a new card.

## Product behavior

For a visible bead row with one or more **current structured notes originally authored
by a contributing agent**, show the newest such note directly beneath the row. Use the
note's original author and append timestamp for attribution and ordering, even if
someone else edited it later. Show the current edited text and a quiet `edited` marker
when applicable. A retracted note vanishes from the preview; the `noted` verb chip still
reflects historical actions. An edit by another actor does not make that actor the
note's author or show the note under that actor's unrelated row. Older legacy free-text
note fields without trustworthy per-note attribution do not produce previews.

Suggested visual grammar (exact glyph/style can be tuned against a screenshot):

```text
  Beads:
    14:32:06  ✎ sase-42 · own · noted ×2
               │ 14:31 · Fixed the refresh race and verified the result.
               │ +1 earlier note in bead detail
               ↳ read: Checking the regression
```

The warm bead accent marks the preview, while the note body uses a legible neutral color
and metadata/overflow is dim. The note block is subordinate to its bead row, aligned
with that row's continuation column. Preserve the full bead ID and the row's existing
hint target. If a note exists, show its preview first; retain the audited read reason as
a separately labeled `read` continuation if present, and omit the fallback title to
avoid repetitive text. Without a note, retain the current reason-or-title behavior. If
several agent-session members contributed notes to one bead, choose the newest current
note across them and identify its producing role on the note line; clan aggregation
retains its member label. Do not infer attribution from the bead ID, title, `own` mark,
or a mere `noted` verb.

Keep the card scannable: show **one** newest current note per visible bead, wrapping on
display-cell boundaries within the existing 80-cell row budget; show at most three
wrapped note-text lines. For longer notes, end with an explicit
`… full note in bead detail` continuation. If other current notes from the selected
agent/session exist on that bead, add `+N earlier notes in bead detail` in dim text. The
existing numbered bead hint remains the route to complete, live detail, so no new key or
ambiguous note-specific hint is needed. Short notes are shown in full. Plain note text
must be rendered as data: no Rich markup or terminal control sequence should alter the
card's styling. Handle narrow panels, wide Unicode, embedded newlines, and
empty/malformed preview fields without broken indentation or blank ornaments.

The ARTIFACTS header still counts beads, and sorting still uses the newest bead
interaction, not the newest note. The preview is a snapshot from the touch index and
follows its existing refresh cadence; it should converge after append, edit, remove,
sync, or periodic refresh without blocking the Textual event loop. A missing or stale
index keeps the existing safe empty/stale behavior.

## Phase 1: `note_index`

Work in the linked `sase-core` checkout obtained through `sase repo open sase-core`,
following its `AGENTS.md` and core conventions. Extend `BeadTouchWire` with an optional
latest-current-note preview and a count of current notes for that `(actor, bead)` row.
The preview carries stable note ID, original append timestamp and author, current text,
edit metadata, and whether the text was shortened for the cache. Bound stored text to a
modest documented Unicode-safe prefix (e.g. 1,024 scalar values), with a truncation
flag, so one pathological note cannot grow the shared index without limit. The TUI
applies its separate three-display-line limit. Keep the index a derived cache, with no
new write-time source of truth.

During the existing per-stream reduction, replay structured `note_appended`,
`note_edited`, and `note_removed` payloads by stable note ID, using the bead store's
note normalization rules where possible. Attribute an appended note to its original
author/event actor only when it is a valid agent; edits update text/metadata in place
and removal deletes the current note. If an edit or remove references an unknown ID,
skip its preview effect without discarding unrelated touch data. Select the newest
surviving note by append timestamp, with deterministic tie-breaking, and count all
surviving notes for that author and bead. Retain existing `noted` action counts and
touch timestamps exactly as they are, including edit/remove actions. Historical notes
that lack structured attribution do not enter this projection.

Bump the touch-index wire schema so old index files miss and the next established
refresh rebuilds them. Update the Rust parity fixture, reducer tests (append, multiple
authors, edit by a different actor, remove and fallback to previous note, malformed
events, Unicode/long text, old-schema miss), the `sase_core_py` binding round trip, and
any schema mirrors. Document the additive wire contract and verify that the cached query
still reads only the index file. Run targeted Rust tests and the repository's guarded
`sase tool run check`; avoid bare `cargo` and bare guarded recipes.

## Phase 2: `context_card` (after `note_index`)

Move `sase-core-revision.txt` past the phase-1 commit before Python relies on the new
wire fields, following `docs/rust_backend.md`. Extend the thin Python facade
dataclasses/conversion to accept the new optional fields while tolerating absent or
malformed values as no preview. Carry note data through `_BeadTouchDisplayEvent`, the
per-agent/session loader, and `merge_bead_touch_entries` without losing actor or role
identity. Sum current-note counts for a bead and choose the newest candidate
deterministically; deduplicate any same stable note ID produced by a merge. View-only,
read-only, and own-only rows have no note preview. Keep `sase bead touched` row/JSON
compatibility unless a deliberate additive CLI extension is needed; this request is
about the card.

Render the preview in `_agent_bead_touches.py` with a small, testable helper using the
existing context colors, cell-width wrapping, and hint state. Ensure its text
participates in the Context card's logical text for search/copy, and that a new note
does not consume a second hint number or change the existing bead hint target. Cover no
note, one short note, long/multiline note, Unicode/wide cells, edited and removed notes,
read reason alongside a note, older-note count, two members on the same bead, clan
member labeling, and five-row overflow. Check both spread and paged Main deck modes and
narrow/split layouts. Update `docs/ace.md` in the `SASE CONTEXT / ARTIFACTS` description
and add an appropriately scoped visual snapshot; inspect the generated PNG and a live
screenshot showing an actual agent-authored note. Follow the TUI screenshot and
performance runbooks.

Run focused tests during development, then `just fix` and the guarded
`sase tool run check` for the sase repo. Measure or trace that bead-note enrichment uses
the existing cached background lane and adds no per-keypress stream read or live
bead-detail resolution. If a full check exposes unrelated failures, report the evidence
distinctly while fixing failures caused by this feature.

## Done when

- A single agent's newly appended note is visible under the correct bead in the Main
  deck Context card, with its text legible and the existing bead hint opening complete
  detail.
- Two notes show the newest current note and an accurate earlier-note count; edits
  update the preview and removals remove it after the index refresh. Other agents' notes
  and old un-attributed text do not leak into the selected agent's row.
- Session/clan attribution, narrow wrapping, search/copy text, bead ordering, five-row
  limit, and existing read-reason behavior are verified.
- Rust and Python contracts are pinned together, their guarded checks pass, and the
  screenshot is visually inspected.
