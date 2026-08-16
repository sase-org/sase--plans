---
tier: tale
title: Flag beads on every bead-rendering surface
goal:
  Flag beads render with one shared identity and due-state vocabulary across CLI, pages,
  ACE, notifications, Telegram, and mirror boundaries.
size: medium
proposed_by: bbugyi200.athena.sase-nb.8
bead: sase-nb.8
create_time: 2026-08-16 17:52:33
status: wip
---

- **PARENT:** [202608/feature_flags.md](feature_flags.md)
- **BEAD:**
  [sase-nb.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-nb/sase-nb.8.md)

# Flag beads on every bead-rendering surface

## Goal

Render `flag` beads unmistakably and consistently on every surface named by phase
`sase-nb.8`, using only the shared presentation vocabulary established by its closed
`look` dependency. Keep due-state derivation centralized, keep the ACE render path pure
and cache-backed, and make flag beads explicitly internal-only at the external issue
mirror boundary.

## Grounding and constraints

- The Python bead model, `sase bead` detail JSON/prose fields, flag due-state predicate,
  and shared `bead_flag_presentation` key/countdown chips already exist. Extend them; do
  not introduce another glyph, color, countdown calculation, or due-state model.
- Due state is derived from the flag record, local date, and current SASE release. It is
  never persisted and never changes the bead status.
- ACE must load and precompute all flag rows and due labels off the event loop, then
  filter and render the immutable snapshot without filesystem access, subprocesses, or
  data-scaled work on keypress/render paths. Counts belong in the existing state/count
  lane, not in another identity header.
- The Telegram formatter lives in the linked `sase-telegram` repository. Its checkout
  must be opened through `sase repo open`, changed only at that audited path, and
  verified with that repository's own checks.
- Phase workers do not create follow-up beads or close ancestors. Any discovered work is
  recorded as a `PROPOSED FOLLOW-UP:` note on `sase-nb.8`.

## Implementation

1. Complete the core SASE CLI and page projections.
   - Add the shared flag key/countdown cells to compact bead list and search rows while
     preserving derived glyph widths and existing metadata ordering.
   - Reuse the existing `FLAG` detail and JSON projection, filling any missing styling
     or regression coverage so key, both removal thresholds, and derived due state are
     stable across prose, styled, and JSON output.
   - Extend bead-page identity, roster, and table rendering so a published flag page
     states the key and both thresholds and remains self-describing.
   - Extend shared summaries with a due-flag count that can feed `sase bead stats` and
     ACE without re-deriving urgency in each renderer.

2. Model flags as their own ACE Beads group and query facet.
   - Extend the immutable Beads snapshot, worker-side load/sort/group logic, row-kind
     model, option construction, navigation, entry targets, and empty/count copy to
     include flag beads as a peer group beside Tasks and Epics.
   - Precompute each flag's `live`/`soon`/`due` state while building the off-thread
     filtering/query index. Add the repeatable, negatable `due:` enum facet with those
     three values to the shared parser, query profile, filter-bar completions, query
     corpus rows, and matched-count lane.
   - Add a `flag_text()` row renderer composed from the shared type glyph, key chip,
     title, status, and countdown; add Flag properties/markdown detail using the same
     shared chips and threshold fields.
   - Keep flag rows non-launchable and non-snoozable unless an existing flag workflow
     explicitly enables an action; preserve selection, fold, filtering, and cached
     refresh behavior.

3. Cover ACE bead modals, prompt context, and gate/notification presentation.
   - Allow the create modal to select `flag` and collect its key, removal date, and
     release threshold, mapping them to the existing `flag(...)` type argument without
     weakening task/plan/phase validation.
   - Verify the editor's derived Flag label, make close/wait modal copy correct for the
     new type, and include the flag identity/thresholds in the agent prompt bead
     section.
   - Register the `flag` notification tag through the shared type presentation accent
     and glyph, and add focused coverage showing the gate debug panel can render a
     `FlagTriage` bundle without a type-specific fallback failure.

4. Make integrations intentional.
   - In the external mirror apply path, explicitly exclude `IssueType.FLAG` with a
     comment documenting that temporary-flag hygiene is internal and must not create
     GitHub issue noise; cover both create and reconciliation paths.
   - In `sase-telegram`, add a real flag-bead fixture proving the all-caps `FLAG`
     section and JSON-derived fields survive `bead_show_to_markdown` with correct
     escaping and no formatter special-case loss.

5. Add deterministic visual and behavioral regression coverage.
   - Extend focused unit/TUI tests for row grouping, counts, details, query parsing and
     completions, modal serialization, page markdown, notification tags, gate debug, and
     mirror exclusion.
   - Add a dedicated ACE PNG scenario containing live, soon, and due flag beads. Run the
     visual test first to inspect actual/expected/diff artifacts, accept only the
     deliberate golden, and rerun exact local pixel comparison.
   - Assert renderers reference the shared flag presentation helpers rather than
     hard-coded flag glyphs or colors outside the look modules.

## Verification

1. Run the focused Python tests for CLI/detail/pages/summaries, Beads filtering and
   rendering, modal/prompt/notification/gate behavior, and external mirror handling.
2. Run the `sase-telegram` formatter test suite and that repository's `just check` for
   its linked-repository change.
3. Run `just install`, then `just test-visual`, deliberately update the new golden if
   the inspected diff matches the design, and rerun `just test-visual` without update.
4. Run the primary repository's required `just check`; if scoped selection escalates or
   reports unusual coverage, use the prescribed monitored `just check-full` path.
5. Exercise a real flag fixture through compact list, full detail, JSON, page markdown,
   the ACE Beads pane/filter, and Telegram conversion; confirm all three due states, the
   shared coral flag identity, and the absence of external issue creation.
6. Recheck `git diff` in both repositories for accidental unrelated changes, then close
   only `sase-nb.8` with a note naming the checks and visual golden that passed.
