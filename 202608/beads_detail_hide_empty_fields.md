---
tier: tale
title: Drop empty rows from the Artifacts → Beads detail property grid
goal:
  The Beads sub-tab's Details property grid renders a labeled row only when that field
  has a value, so an ordinary bead no longer wastes half the pane on em-dash
  placeholders.
size: medium
proposed_by: bbugyi200.athena.06e
create_time: 2026-08-18 13:19:13
status: wip
---

# Plan: Drop empty rows from the Artifacts → Beads detail property grid

## Problem

On the **Artifacts → Beads** sub-tab, the right-hand **Details** pane renders a fixed
property grid. Every field is emitted whether or not the bead actually has a value for
it, and the missing ones render as a dim em dash (`—`). On a typical task bead the pane
reads:

```
   Assignee sase-nf
      Owner bryanbugyi34@gmail.com
      Model —
    Created 2026-08-16 13:56:07 EDT · 1d ago
    Updated 2026-08-18 12:52:05
     Closed —
    Project sase
Dependencies —
External issue —
 References —
      Patch —
External bug —
Plan reference —
```

Eight of those thirteen rows carry no information. They push the description body down
the pane, and they make the rows that _do_ carry information harder to scan.

Note on the screenshot: the prompt referenced
`.sase/artifacts/home/tmp/screenshots/20260818_131204.png`, which is a capture of the
**Agents** tab, not the Beads sub-tab. The sibling capture
`.sase/artifacts/home/tmp/screenshots/20260818_125901.png` (same session, ~13 minutes
earlier) is the one that shows this defect on Artifacts → Beads, and the block quoted
above is transcribed from it. The fix below targets the Beads detail pane the prompt
describes.

## Root cause

`src/sase/ace/tui/widgets/artifacts/beads_detail.py:62` (`bead_properties_header`)
builds a `list[DetailProperty]` where a large block of rows is appended unconditionally
(`beads_detail.py:121-142`: `Assignee`, `Owner`, `Model`, `Created`, `Updated`,
`Closed`, `Project`, `Dependencies`, `External issue`, `References`, `Patch`,
`External bug`), and `src/sase/ace/tui/widgets/artifacts/beads_detail_properties.py:40`
(`properties_header`) adds a table row for every entry it is handed.

"No value" then gets converted into a visible em dash in two different ways, so there is
no single place that currently knows a field is absent:

1. `_property_text` (`beads_detail_properties.py:281-286`) turns an empty `str` into
   `Text("—", style="dim")`.
2. Five helpers pre-empt that by returning the em-dash `Text` themselves:
   `flag_properties`'s unresolved due state (`beads_detail_properties.py:74`),
   `references_text` (`:115`), `dependencies_text` (`:131`), `created_text` (`:205`),
   and `external_issue_property_text` (`beads_detail_external.py:20`).

`plan_reference_properties` (`beads_detail_properties.py:88-103`) is a third shape of
the same problem: with no `issue.design` it deliberately returns
`(("Plan reference", ""),)`, i.e. it asks for a row it knows is empty.

## Approach

Make **"has no value" mean "has no row"**, enforced in exactly one place.

`properties_header` becomes the single gate: it renders each value once, and skips the
row when the rendered text is blank. Every producer upstream then signals absence the
same way — with an empty `Text`/`str` — instead of hand-rolling a placeholder. That
keeps `bead_properties_header`'s row list readable (the unconditional
`properties.extend([...])` block stays unconditional) and means any property row added
to this grid later inherits the behavior for free, rather than each new call site having
to remember to guard itself.

Blank is tested with `.plain.strip()`, not `.plain`, because chips built by `_chip`
(`beads_detail_properties.py:289`) intentionally carry padding spaces — a chip must
never be mistaken for an empty value, and a whitespace-only value is not a value.

### Scope boundaries

- **Only the Beads grid changes.** `plans_detail.py` carries its own private copy of
  `_properties_header`/`_property_text` (`plans_detail.py:185-203`) and its own `"—"`
  fallbacks; the Plan sub-tab is not part of this request and must keep rendering
  exactly as it does today. Do not refactor the two into a shared helper as part of this
  change.
- **`bead_preview_markdown` already does the right thing** — it appends `**Assignee:**`,
  `**Owner:**`, `**Model:**`, `**Patch:**` and friends only when truthy
  (`beads_detail.py:222-245`). Leave it alone; the fix is only for the property grid,
  and after this change the two surfaces agree.
- **Em dashes elsewhere in the TUI stay.** Placeholders in fixed-width tables
  (`modals/statistics_pane_*.py`, `modals/alias_history_rendering.py`, etc.) hold
  columns in alignment and are not in scope.
- No CSS change is required: `#beads-detail-properties` is already `height: auto`
  (`src/sase/ace/tui/styles.tcss:587-599`), so the grid and the markdown body below it
  reflow as rows disappear.

### Rows that must keep rendering

Some fields are always populated and must not regress: `ID`, `Type`, `Status`,
`Readiness`, `Project` (and `Task type`, `Size`, `Tier` where the existing conditionals
already admit them). The grid can therefore never collapse to nothing.

Two cases deserve an explicit decision, because "empty" is arguably meaningful:

- **`Resolution` on a closed bead with no recorded resolution** keeps rendering.
  `beads_detail.py:109-115` already substitutes the literal `"(unrecorded)"`, which is a
  value — an assertion that we looked and found nothing — not an absence. Do not change
  it.
- **`Created` with an unparseable timestamp** loses its row. `created_text` currently
  maps `BEAD_TIME_UNKNOWN_LABEL` to the em dash; under the new rule that becomes an
  empty `Text` and the row disappears. This is consistent, and it is not a regression
  against `tests/test_bead_time_surface_coverage.py` — that suite drives
  `bead_preview_markdown` and `beads_rendering._bead_text` for this surface, not
  `bead_properties_header`, and it asserts creation time is present for beads that
  _have_ one.

## Implementation

### 1. `src/sase/ace/tui/widgets/artifacts/beads_detail_properties.py`

- `properties_header` (`:40`): render the value once, skip blank rows.

  ```python
  for label, value in properties:
      text = _property_text(value)
      if not text.plain.strip():
          continue
      table.add_row(label, text)
  ```

  Keep the title, the `Table.grid` construction, and the divider exactly as they are —
  the divider must still render even if every optional row is dropped.

- `_property_text` (`:281`): delete the em-dash fallback. An empty `str` now becomes an
  empty `Text`; the caller decides what to do with it.

  ```python
  def _property_text(value: str | Text) -> Text:
      if isinstance(value, Text):
          return value
      return Text(value, style="white", overflow="fold")
  ```

- `flag_properties` (`:64`): build the list without the `due_text` placeholder and
  append `("Due state", …)` only when `due is not None`. `Flag` and `Removal` stay
  unconditional — `flag_properties` already returns `[]` when the bead carries no flag
  record.

- `plan_reference_properties` (`:88`): return `()` instead of
  `(("Plan reference", ""),)` when `issue.design.strip()` is empty. The populated
  branches (`Plan reference` + `Linked plan`, including the `"cannot resolve"` warning)
  are unchanged.

- `references_text` (`:113`), `dependencies_text` (`:124`), `created_text` (`:201`):
  return an empty `Text()` in place of `Text("—", style="dim")`.

- `created_text` still needs its `BEAD_TIME_UNKNOWN_LABEL` comparison, so that import
  stays live. Re-check the module's imports after editing and drop anything the edits
  orphaned, so Symvision's unused-symbol gate stays green (read
  `sase/memory/symvision.md` with `/sase_memory_read` if it complains).

### 2. `src/sase/ace/tui/widgets/artifacts/beads_detail_external.py`

- `external_issue_property_text` (`:16-20`): return an empty `Text()` when `links` is
  empty. `external_issue_markdown` and `external_issue_inline` are untouched.

### 3. `src/sase/ace/tui/widgets/artifacts/beads_detail.py`

No structural change expected. `bead_properties_header`'s unconditional
`properties.extend([...])` block (`:121-142`) stays as-is and is filtered centrally —
including `("Close reason", issue.close_reason or "")` (`:116`), which now simply
disappears when a closed bead has no reason recorded.

Verify by reading, not by assumption, that every entry appended in this function reaches
`properties_header` as either a real value or a blank one; if any producer still
hard-codes a placeholder string, fix the producer rather than special-casing it in the
filter.

## Tests

Add to `tests/ace/tui/test_artifacts_beads_rendering.py`, using the existing
`snapshot(tmp_path)` fixture from `tests/ace/tui/_artifacts_beads_helpers.py` and the
same `Console(width=100, color_system=None)` capture pattern the file already uses (see
`test_detail_uses_shared_metadata_and_triage_callout`, `:207`):

1. **Sparse bead drops its empty rows.** Render `bead_properties_header` for the
   fixture's `alpha-open` task (no assignee, owner, model, patch, dependencies, refs,
   design, and not closed) and assert the capture contains `"ID"`, `"Type"`, `"Status"`,
   `"Readiness"`, `"Project"`, and `"Created"`, and does **not** contain `"Assignee"`,
   `"Model"`, `"Closed"`, `"Dependencies"`, `"External issue"`, `"References"`,
   `"Patch"`, `"External bug"`, or `"Plan reference"`. Assert `"—"` is absent from the
   whole capture — that is the regression guard that keeps a future contributor from
   reintroducing a placeholder anywhere in this grid.

   Watch for substring collisions when writing these assertions: `"Patch"` is a
   substring of nothing else here, but `"Closed"` occurs inside `"Previously closed"`
   only with different casing, and `"Type"` occurs inside `"Task type"` — pick the
   fixture bead and the assertion strings so each check is unambiguous, or assert on
   whole grid lines.

2. **Populated bead keeps its rows.** Render the fixture's `alpha-1` epic (it has
   `owner`, `assignee`, and `design="plan:202608/beads.md"`, and the snapshot supplies
   dependencies) and assert `"Assignee"`, `"Owner"`, `"Plan reference"`, and
   `"Dependencies"` are all still present with their values — this is what proves the
   filter drops absence, not content.

3. **Unresolved flag due state drops its row.** The existing flag test (`:140-162`)
   asserts `"Due state"` and `"DUE ⧗ +6d"` are present when `flag_due` is populated; add
   the mirror case with `flag_due` empty and assert `"Flag"` and `"Removal"` still
   render while `"Due state"` does not.

4. **Closed-but-unrecorded resolution still renders.** Assert a closed bead with
   `resolution=None` still shows `"Resolution"` and `"(unrecorded)"`, pinning the
   deliberate exception above.

Check the existing assertions in `tests/test_timezone_display_artifacts.py:95` and the
rest of `test_artifacts_beads_rendering.py` still hold — they assert on populated
values, so they should, but confirm rather than assume.

## Visual snapshots

The Beads detail pane is in-frame for several PNG goldens, which will need regenerating
with `--sase-update-visual-snapshots` (see `just test-visual` and the PNG snapshot notes
in `CLAUDE.md`):

- `tests/ace/tui/visual/snapshots/png/artifacts_beads_populated_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_beads_collapsed_relations_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_beads_reopened_detail_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_split_even_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_split_wide_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_split_narrow_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_split_narrow_80x24.png`

`artifacts_beads_empty_120x40.png` renders with no selection, so its property grid is
already hidden (`beads_navigation.py:394-398`) and it should not move. Do not
blanket-accept the whole visual suite: run it, confirm the diff for each changed golden
is only the property grid shrinking, and leave any unexpected golden untouched pending
investigation.

The token assertions in those visual tests (`"Tasks"`, `"Epics"`, `"awaiting triage"`,
`"-status:closed"`, `"Previously closed"`, `"↺1"`, `"canceled"`) are all on populated
content and should keep passing; if one starts failing, that is a real signal, not a
golden to accept.

## Verification

- `just install` first (ephemeral workspace).
- `just test-visual` to regenerate and then re-verify the goldens above.
- `just check-full` before landing — this touches shared TUI presentation and
  regenerates visual goldens, so the scoped lane is not sufficient. Run it through
  `/sase_monitor` with a `--next` action, never inline.
- Manual confirmation in the real TUI is worthwhile: open ACE, press `2` for Artifacts
  and `3` for Beads, select a bare task bead, and confirm the Details pane now leads
  with populated rows and no em dashes.

## Out of scope

- The Plan sub-tab's property grid and any shared-helper refactor between
  `beads_detail_properties.py` and `plans_detail.py`.
- The Agents tab's `SASE CONTEXT → BEAD` section
  (`widgets/prompt_panel/_agent_bead_section.py`) and the neighbor rows visible in the
  referenced `20260818_131204.png` screenshot.
- Any change to which fields a bead _has_; this is presentation only.
