---
tier: tale
title: Fold BEAD notes and +1 evidence at the first lane fold level
goal:
  The Agents metadata panel's SASE CONTEXT / BEAD lane folds its Notes and +1 Evidence
  rows to a one-line digest at the selected agent or lane's first fold level, and
  expands them in place at every higher level, without changing the lane's row set,
  labels, or lossless-wrap contract.
size: medium
proposed_by: bbugyi200.athena.x8
create_time: 2026-08-10 10:39:02
status: wip
---

# Fold BEAD Notes And +1 Evidence At The First Lane Fold Level

## Goal

In the Agents-tab metadata panel, the `SASE CONTEXT / BEAD` lane renders every bead
field at full length regardless of the panel's fold level. Two of those fields — `Notes`
and `+1 Evidence` — are append-only logs that grow without bound, so a well-corroborated
task bead or a long-running phase bead pushes the rest of the metadata panel (and the
prompt/reply body beneath it) far off screen.

Make those two rows render **folded at the first fold level** of the selected agent or
agent lane, and expand them at every higher level. Nothing is removed: a folded row
keeps its position, keeps its label, and shows a single dim digest line that states how
much text it is hiding and which chord reveals it.

## Design

### Vocabulary And Existing Machinery This Reuses

- `FoldLevel` / `FoldScale` (`src/sase/ace/tui/models/fold_state.py`,
  `src/sase/ace/tui/models/fold_scale.py`). A selected row's scale comes from
  `lane_fold_scale(agent)`: `AGENT_FOLD_SCALE` = `(COLLAPSED, EXPANDED, FULLY_EXPANDED)`
  for a single agent, `FAMILY_FOLD_SCALE` = `(EXPANDED, FULLY_EXPANDED)` for a family
  container row.
- **"First fold level" means scale position 1, not a named level.** That is `COLLAPSED`
  for a single agent and `EXPANDED` for a family container. Resolve it with
  `fold_scale_position(level, scale)`, exactly like the existing
  `slow_tool_detail_level()` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_slow_tools_detail.py`. Follow that
  precedent; do not compare against `FoldLevel.COLLAPSED` directly.
- `append_fold_glyph()` and `FOLD_STYLES`
  (`src/sase/ace/tui/widgets/prompt_panel/_fold_language.py`) supply the shared `▸`
  collapsed marker and its grey style, so a folded row speaks the same visual language
  as every other fold in the panel.
- `count_phrase()` and `COLOR_TRUNCATION`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_context_common.py`) supply the
  pluralized count and the dim-italic style already used for elided content.
- The hidden-content chord hint mirrors the existing neighbors tail
  (`hidden_tail_hint="zz / za to show more"` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_neighbors.py`).

### The Three Invariants This Feature Must Hold

These are what make the result intuitive and reliable, and each one is directly
testable.

1. **Stable rows.** The BEAD lane's row set, row order, and row labels are identical at
   every fold level. Folding only replaces a row's multi-line _value_ with a single-line
   digest. Expanding never reshuffles the table, so the reader's spatial memory of the
   lane survives a fold change.
2. **Folding never costs more than it saves.** A row folds only when its value occupies
   more than one authored line. A one-line note stays inline at every level, because
   `▸ 1 line (zz to show)` would be longer and less useful than the note itself.
3. **Nothing is silently dropped and nothing is elided.** A folded row always states its
   hidden line count. The BEAD lane's documented "values wrap losslessly without
   ellipses" contract still holds: no digest uses `…`, and no digest truncates.

### Which Rows Fold, And Why Only Those

Fold `Notes` and `+1 Evidence`. Leave `Task Title` / `Phase Title`, `Description`,
`Size`, `Epic Plan`, `Epic Title`, `Created`, and `+1 Reports` untouched at every level.

The distinction is principled and should be recorded in a comment: **fold the unbounded
append-only logs; never fold identity fields.** `Notes` and `+1 Evidence` grow every
time an agent appends a note or corroborates a task, while the identity fields are
fixed-size and are the reason the lane exists. `+1 Reports` is already a single derived
line and is the count signal that makes the folded `+1 Evidence` row legible, so it
stays.

The `[+N]` corroboration badge in the lane header (`plus_one_badge`) is likewise
untouched at every level — the fold hides evidence bodies, never the fact that
corroboration exists.

### Rendered Result

Selected single agent on a corroborated task bead at fold level 1/3 (the default —
`panel_fold_level` initializes to `FoldLevel.COLLAPSED` in `src/sase/ace/tui/app.py` and
is session-only):

```
▸ BEAD · ⬤ task sase-task.5  [+2]
    Task Title: Corroborated task
    Description: Carry evidence into the Agents tab.
          Notes: ▸ 3 lines (zz to show)
           Size: medium
     +1 Reports: 2 +1 reports
    +1 Evidence: ▸ 7 lines (zz to show)
        Created: 2026-08-01 10:30:00 EDT · 4d ago
```

The same lane at fold level 2/3 or 3/3 renders exactly as it does today, with the
complete notes text and every evidence block.

Styling of a folded value: `▸ ` via `append_fold_glyph(text, FoldLevel.COLLAPSED)` (grey
`FOLD_STYLES[FoldLevel.COLLAPSED]`), then the digest text in `COLOR_TRUNCATION` (dim
italic). Both rows use the same digest vocabulary — a folded row is meta-text about
hidden content, so it drops the row's normal accent (`COLOR_REASON` for notes,
`PLUS_ONE_RICH_STYLE` for evidence) rather than half keeping it.

### Why A Line Count And Not A Content Preview

The digest is `{n} lines` where `n = len(value.plain.splitlines())` — the count of
**authored** lines in the value, computed width-independently.

- A content preview would need `…` truncation, which breaks invariant 3 and the lane's
  documented lossless-wrap contract.
- Counting _note entries_ rather than lines would require parsing the
  `[<timestamp> · <author>]` entry format that `sase-core` writes in
  `appended_note_text` (entries joined with `\n\n`). That format is core-backend domain
  knowledge, and per the repo's Rust-core boundary rule it must not be reimplemented in
  this repo's presentation layer. Line counting needs no domain contract at all.
- A width-dependent count (how many lines it wraps to _right now_) would make the digest
  change as the panel resizes and would differ between `logical_text` and the responsive
  render.

Because a row folds only when it has more than one authored line, the digest always
reads `2 lines` or more; `1 line` is unreachable by construction.

Consequence to accept deliberately: a single very long note stays inline and wraps,
because it is one authored line. That is the correct trade for invariant 2, and the
lane's existing 80-cell lossless wrap already governs it. Say so in a comment so a
future reader does not "fix" it.

### Which Fold Level Each Lane Reads

Resolve the tier from the effective level of the **`bead` section**, not the raw lane
level:

```
effective_fold_level(
    lane_section_fold_overrides.get(BEAD_SECTION_ID, resolved_lane_fold_level),
    lane_scale,
)
```

This costs nothing for a single agent (no override is ever recorded for `bead` there, so
it falls through to the lane level) and it buys real control on a family container row,
whose `SASE CONTEXT` lane headings already carry section ids and are already targetable
with `za` / `zA`. On a family lane, cycling the BEAD heading therefore flips its notes
and evidence between folded and expanded, which is exactly what the heading's glyph
implies.

For a single agent the controls are the panel-level chords: `zz` cycles, `zZ` toggles to
the extreme, `z2` / `z3` select directly.

## Implementation

### 1. Bead fold tier and folded-row rendering

File: `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py`

- Add `BEAD_SECTION_ID = "bead"` next to `BEAD_SECTION_LABEL` (this is the section id
  already spelled as a `"bead"` literal in `_agent_context.py`).
- Add a positional detail tier mirroring `SlowToolDetail`:

  ```python
  class BeadDetail(IntEnum):
      """Positional detail tiers owned by the BEAD lane."""

      DIGEST = 1
      FULL = 2


  def bead_detail_level(level: FoldLevel, scale: FoldScale) -> BeadDetail:
      """Resolve a lane fold level to the positional BEAD detail tier."""
      position, _size = fold_scale_position(level, scale)
      return BeadDetail.DIGEST if position == 1 else BeadDetail.FULL
  ```

- Add `detail: BeadDetail = BeadDetail.FULL` to `ResponsiveBeadSection`. The default
  keeps every caller that has no fold context (and the existing
  `tests/test_bead_time_surface_coverage.py` constructions) rendering exactly as today,
  matching the `fold_level: FoldLevel | None = None` convention used elsewhere in this
  package.
- Add one private helper that both `_rows()` branches use for the two foldable values:

  ```python
  def _foldable_value(self, value: Text) -> Text:
      """Return ``value`` or its one-line folded digest for this detail tier."""
  ```

  It returns `value` unchanged when `self.detail is not BeadDetail.DIGEST` or when the
  value has one authored line; otherwise it builds a fresh `Text`, calls
  `append_fold_glyph(text, FoldLevel.COLLAPSED)`, and appends
  `f"{count_phrase(len(lines), 'line')} ({BEAD_FOLD_HINT})"` in `COLOR_TRUNCATION`,
  where `BEAD_FOLD_HINT = "zz to show"` is a module constant.

- Apply it in both `_rows()` branches: wrap `self._notes_value()` and
  `self._plus_one_evidence_value()`. Do not wrap anything else.
- Because `logical_text` and `__rich_console__` both iterate `_rows()`, the folded and
  rendered documents agree by construction and the responsive splice range stays
  correct. Do not fork the fold decision into either path.

Import notes: `IntEnum` from `enum`, `FoldScale` / `fold_scale_position` from
`...models.fold_scale`, `FoldLevel` from `...models.fold_state`, `append_fold_glyph`
from `._fold_language`, `count_phrase` / `COLOR_TRUNCATION` from
`._agent_context_common`. None of these introduce an import cycle (`_fold_language`
depends only on `_helpers` and the models package).

### 2. Pass the tier from the header builder

File: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`

In `build_header_text`, where
`bead_section = ResponsiveBeadSection(summary.bead_summary)` is constructed (currently
around line 276), pass the resolved tier:

```python
bead_section = ResponsiveBeadSection(
    summary.bead_summary,
    detail=bead_detail_level(
        effective_fold_level(
            lane_overrides.get(BEAD_SECTION_ID, resolved_lane_fold_level),
            lane_scale,
        ),
        lane_scale,
    ),
)
```

`resolved_lane_fold_level`, `lane_scale`, `lane_overrides`, and `effective_fold_level`
are all already in scope in that function. Do **not** gate this on
`family_fold_enabled`: the sibling fold-responsive features in this same function
(`neighbors_limit`, the slow-tool `panel_level`) already respond to
`resolved_lane_fold_level` for every lane, and gating on family would leave single
agents — the common case this feature exists for — unaffected.

Clan container rows return early at the top of `build_header_text`, so they are
untouched.

### 3. Stop telling the user this lane has nothing to fold

File: `src/sase/ace/tui/actions/navigation/_fold.py`

`_notify_lane_fold_scope()` currently notifies
`"Fold levels shape clan, family, neighbor, and slow-call summaries"` whenever a
single-agent lane has no neighbors and no slow calls. After this change that message is
wrong for any lane whose bead has a foldable row.

- Add a `_selected_lane_has_foldable_bead_rows(agent)` helper built the same way as the
  existing `_selected_lane_has_slow_tool_calls`: read the **cached** summary via
  `get_cached_detail_header_summary(panel, agent)`, return `False` when it is `None`,
  and otherwise report whether `summary.bead_summary` has notes or `plus_one_evidence`
  whose rendered value spans more than one authored line. Reuse the same predicate the
  renderer uses so the two can never disagree — export a small module-level helper from
  `_agent_bead_section.py` (for example
  `bead_summary_has_foldable_rows(summary) -> bool`) and call it from both places.
- Return early from `_notify_lane_fold_scope` when it is `True`.
- Update the notification text to name beads, since it is a general explanation of fold
  scope: `"Fold levels shape clan, family, neighbor, bead, and slow-call summaries"`.

This stays on the keystroke path, so it must remain cache-only: no disk reads, no
enrichment kick-off. `get_cached_detail_header_summary` already satisfies that.

### 4. Help popup

File: `src/sase/ace/tui/modals/help_modal/agents_bindings.py`

The ace guidance requires the `?` popup to track behavior changes. The agents fold table
already ends with a non-keybinding explanatory row
(`("Roster entry", "Inherit MEMBERS then panel")`). Add one more in the same shape after
it, e.g. `("BEAD notes / +1", "Folded below level 2")`. Keep the description at or under
32 characters and re-check the modal's fixed box width after the edit.

### 5. Documentation

File: `docs/ace.md`

- In the fold-chords section (the paragraph beginning "The `Fold: N/M` header field
  reports the position…"), the sentence **"Other sections on a regular-agent panel stay
  fold-inert."** becomes false. Replace it with a sentence stating that the
  `SASE CONTEXT / BEAD` lane's `Notes` and `+1 Evidence` rows also follow the lane's
  scale: folded to a one-line `▸ N lines` digest at position 1, complete at every higher
  position, with the row set and labels unchanged.
- In the `SASE CONTEXT / BEAD` bullet in the metadata-section reference (near the "Shown
  for an epic phase worker" bullet), add the same behavior in one or two sentences,
  including that the `[+N]` header badge and the `+1 Reports` count stay visible when
  the evidence body is folded.

Do not attempt to repair the unrelated pre-existing drift in those BEAD paragraphs (they
predate the `Notes`, `+1 Reports`, `+1 Evidence`, and `Created` rows and still describe
the lane as phase-only). If that bothers you, note it for a separate follow-up rather
than widening this change.

## Tests

### Unit — `tests/ace/tui/widgets/test_agent_display_bead_section.py`

The existing `_header()` helper calls `build_header_text` without `lane_fold_level`,
which resolves to scale position 1, so three current tests now describe folded output.
Give `_header()` an explicit `lane_fold_level: FoldLevel | None = None` parameter and
update these three to pass the expanded level, keeping their existing assertions about
full content intact:

- `test_task_and_phase_notes_follow_description`
- `test_task_bead_lane_renders_plus_one_count_and_evidence`
- `test_bead_notes_render_literal_multiline_and_wrap_losslessly`

Then add:

1. **Folds at position 1.** Multi-line notes plus two evidence entries at
   `lane_fold_level=FoldLevel.COLLAPSED`: assert both digests appear with their exact
   line counts and the `zz to show` hint, assert the notes body and every evidence
   reporter/note/ref string is absent, and assert `[+2]`, `+1 Reports:`, and
   `2 +1 reports` are still present.
2. **Row set is stable across levels** (invariant 1). Render the same summary at
   positions 1 and 2 and assert the ordered list of row labels is identical, and that
   both orders match the documented field order.
3. **Expands at positions 2 and 3.** `FoldLevel.EXPANDED` and `FoldLevel.FULLY_EXPANDED`
   both produce the complete bodies.
4. **One-line values never fold** (invariant 2). A single-line note at position 1
   renders inline with no `▸` and no `lines` digest.
5. **No ellipsis and no loss** (invariant 3). At position 1, render at widths 120 and 28
   and assert `…` never appears and every digest line fits the lane budget.
6. **Logical and rendered agree.** The digest text present in `header.plain` is also
   present in the console-rendered output at both widths.
7. **Family container row.** A family container at `FoldLevel.EXPANDED` (its position 1)
   folds; at `FoldLevel.FULLY_EXPANDED` (its position 2) it expands. This is the test
   that proves the tier is positional rather than keyed off `FoldLevel.COLLAPSED`.
8. **Section override wins.** With `lane_fold_level=FoldLevel.COLLAPSED` and
   `lane_section_fold_overrides={BEAD_SECTION_ID: FoldLevel.FULLY_EXPANDED}`, the rows
   expand.
9. **Phase beads too.** A phase bead with multi-line notes folds identically, and the
   `Epic Plan` hint number is still allocated (the plan hint is assigned in
   `_agent_context.append_bead_lane` and must not be disturbed by folding).

### Unit — fold-scope notification

Extend the existing coverage in `tests/ace/tui/test_agents_panel_fold_mode.py` (it
already asserts the `"Fold levels shape …"` string) with a case where the selected lane
has no neighbors and no slow calls but its cached summary carries a foldable bead row,
asserting no notification fires. Keep a case proving the notification still fires when
there is genuinely nothing foldable, and update its expected string to the new wording.

### Visual — `tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py`

`test_agents_task_bead_notes_png_snapshot` currently waits for `Notes:` and asserts
`alice`, `bob`, and `attribution readable` are visible. Rework it to prove both states
in one run:

- At the default fold level, assert the folded digest is present and that `alice`,
  `bob`, and `attribution readable` are **absent**, then take the PNG snapshot under the
  existing name `agents_task_bead_notes_120x40`.
- Then press the fold chord to reach level 2 (`z` then `2`), wait for the body, and
  assert `alice`, `bob`, and `attribution readable` are now present. No second PNG is
  needed; this half is an SVG-text assertion and doubles as end-to-end proof that the
  chord drives the new behavior.

Regenerate the golden with `just test-visual -- --sase-update-visual-snapshots` (or the
equivalent pytest invocation) and confirm the new image visually before committing it.
Inspect `.pytest_cache/sase-visual/` artifacts if the comparison fails.

## Performance

This is pure in-memory presentation and adds no new render-path work worth measuring:
the digest is a `splitlines()` count over strings the lane already renders in full
today, and the folded path renders strictly less text than the current code. No new
caches, no disk access, no new workers.

The one cache that matters already covers this: `AgentHintRenderCacheKey` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_cache.py` includes
`fold_level` and `fold_overrides`, so changing the panel fold level invalidates the
memoized header document for every agent kind. Do not add a cache key field for the bead
tier; it is derived from state that is already keyed.

## Out Of Scope (Record As Follow-Ups, Do Not Implement Here)

- `SASE CONTEXT` lane headings on a **single-agent** panel carry no section marker
  (`_agent_context.py` only calls `append_section_heading` on the family path), so `za`
  / `zA` cannot target the BEAD lane there. Panel-level chords cover this feature;
  making single-agent context lanes section-addressable is a separate change with its
  own section-navigation and scroll-anchoring risk.
- A single-agent lane never prints the `Fold: N/M` header line, so its fold position is
  only discoverable through glyphs. Extending that line to single-agent lanes is a
  separate UX decision.
- The lane header's leading `▸` in `append_context_lane_header` is a decorative bullet
  on single-agent panels but a real fold glyph on family panels. Unifying that is a
  separate cleanup.
- The hidden-content chord hints (`zz to show` here, `zz / za to show more` in the
  neighbors tail) are hardcoded rather than resolved from the keymap registry.
- `docs/ace.md`'s BEAD lane paragraphs predate the `Notes`, `+1 Reports`, `+1 Evidence`,
  and `Created` rows and still describe the lane as phase-only.

## Verification

```bash
just install
just check
just test-visual
```

Run `just check-full` before landing. If `just check`'s scoped test lane reports an
unusual selection or escalates, run `just check-full` instead of trusting the scoped
run.

Manual confirmation in `sase ace`: select an agent whose bead has multi-line notes or
`+1` evidence, confirm the folded digests at the default level, press `zz` (or `z2`) and
confirm both rows expand in place with no other row moving, then press `zz` back to
level 1 and confirm the lane returns to the folded shape.
