---
tier: tale
title: Drop the redundant Readiness row from the Artifacts → Beads detail and preview
goal:
  The Artifacts → Beads detail property grid and its preview no longer render a
  "Readiness" row, and the three now-dead helpers behind it (`readiness_chip`,
  `readiness_label`, `snooze_readiness_label`) are deleted rather than left for
  Symvision to flag.
size: small
proposed_by: bbugyi200.athena.06l
create_time: 2026-08-18 14:04:19
status: wip
---

# Plan: Drop the Beads "Readiness" row

## Goal

`Readiness` restates the `Status` row on every bead the user can actually select, so
remove it from both Beads sub-tab surfaces:

- the detail property grid built by `bead_properties_header()`
  (`src/sase/ace/tui/widgets/artifacts/beads_detail.py:85`), and
- the preview markdown built by `bead_preview_markdown()` (same file, line 198), which
  is what `ArtifactsBeadsPane.preview_for_row()` feeds the preview panel.

Both are the same sub-tab, so they change together; leaving `**Readiness:**` in the
preview would just move the redundancy one keypress away.

## Why The Row Is Redundant (verified)

`readiness_label()` in
`src/sase/ace/tui/widgets/artifacts/beads_detail_properties.py:258-281` returns:

| Bead status   | Readiness renders      | Already shown by                            |
| ------------- | ---------------------- | ------------------------------------------- |
| `closed`      | `closed`               | Status row                                  |
| `claimed`     | `claimed`              | Status row                                  |
| `ready`       | `ready`                | Status row                                  |
| `in_progress` | `in progress`          | Status row                                  |
| `snoozed`     | `snoozed · wakes in …` | Status row + title wake chip + `Snooze` row |
| `open`        | `blocked` / `waiting`  | see below                                   |
| (no snapshot) | `unknown`              | nothing — pure noise                        |

The first five rows are literal restatements of `Status`. The `open` branch is the only
one that derives anything, and it derives almost nothing:

- `snapshot.ready_ids` comes from `bead_read_facade.ready()` →
  `ready_issues_in_issues()` in `sase-core` (`crates/sase_core/src/bead/read.rs:813`),
  which keeps only issues whose status **is** `Ready` and whose type is `Task` and which
  have no active blocker. An `open` bead therefore can never land in `ready_ids`, so the
  `"ready"` branch at line 279 is unreachable for the only status that reaches it.
- That leaves `blocked` (line 277) vs. `waiting` (line 281), i.e. "this open bead has at
  least one dependency that is not closed" vs. "it does not".

So the single fact the row carries that `Status` does not is _blocked-ness_, and that
fact is already on screen three other ways:

1. The list row's state glyph — `_state_glyph()` in `beads_rendering.py:485-497` paints
   the red `BLOCKED_STATE_GLYPH` for exactly the same `blocked_ids` membership.
2. The `Dependencies` property row (`dependencies_text()`,
   `beads_detail_properties.py:127-147`) already prints every dependency with its live
   status glyph and label, which is _why_ the bead is blocked, not just _that_ it is.
3. The list filter query supports `status:blocked` (`beads_filtering.py:97`).

Nothing else in the repo consumes the readiness wording:
`grep -rn "readiness_chip\|readiness_label\|snooze_readiness_label" src tests` returns
only the definitions, the two call sites above, and the two test assertions listed under
**Tests**.

### Considered and rejected: a conditional `Blocked` row

The honest cost of this change is that an `open`, blocked bead loses its one-word
summary in the detail pane. A `Blocked` row rendered only when
`(project, issue.id) in snapshot.blocked_ids` would preserve it. Rejected because it
re-introduces derived state one row below `Dependencies`, which already spells out the
same thing per dependency, and the list glyph plus `status:blocked` cover discovery.
Flagging it here so the owner can ask for it at approval if the summary is wanted.

## Files And Changes

### 1. `src/sase/ace/tui/widgets/artifacts/beads_detail.py`

- In `bead_properties_header()`, delete line 85 from the `properties` literal so it
  becomes:

  ```python
  properties: list[DetailProperty] = [
      ("ID", issue.id),
      ("Type", bead_type_chip(issue.issue_type)),
      ("Status", _status_chip(issue.status)),
  ]
  ```

- In `bead_preview_markdown()`, delete line 198 (`f"**Readiness:** …"`) from the opening
  `lines` literal. `**Type:**` and `**Status:**` stay.
- Drop `readiness_chip as _readiness_chip` and `readiness_label as _readiness_label`
  from the `.beads_detail_properties` import block (lines 53-54). Every other name in
  that block is still used.

Both functions keep their `snapshot` and `project` parameters: they still feed
`_flag_properties`, `_epic_phase_sizes`, `_dependencies_text`,
`_plan_reference_properties`, `snapshot.plan_links`, and `snapshot.triage_gates`. No
signature or caller changes.

### 2. `src/sase/ace/tui/widgets/artifacts/beads_detail_properties.py`

- Delete `readiness_chip()` (lines 233-255) and `readiness_label()` (lines 258-281).
- Delete `"readiness_chip"` and `"readiness_label"` from `__all__` (lines 308-309).
  `__all__` is keep-sorted; removing entries keeps it sorted, but
  `just lint-keep-sorted` is in `just check` either way.
- Drop `snooze_readiness_label` from the `sase.bead.snooze_presentation` import block
  (lines 17-22). `SNOOZE_ACCENT`, `snooze_plus_one_label`, and `snooze_until_label` are
  still used by `snooze_text()`.
- Keep `bead_status_presentation` (still used by `status_chip()` and
  `dependencies_text()`), keep the `Status` import (used by `dependencies_text()`), and
  keep the private `_chip()` helper (still used by `status_chip()`).

### 3. `src/sase/bead/snooze_presentation.py`

- Delete `snooze_readiness_label()` (lines 106-119) and its `__all__` entry (line 165).
  `readiness_label()` was its only consumer anywhere in the repo, and Symvision counts
  test references as no consumer at all, so leaving it would fail `just _lint-symvision`
  as an unused public symbol. Deletion is step 1 of the symvision decision hierarchy —
  do not add a pragma or an epic-symbol entry for it.
- Keep `_WAKE_NOW_LABEL` and `_snooze_remaining_label()`: `snooze_until_label()`,
  `snooze_wake_chip()`, and `snooze_summary()` all still use them.

Nothing else imports the deleted name — `cli_detail.py`, `cli_detail_json.py`,
`cli_crud_snooze.py`, `bead_pages/rendering_identity.py`, `bead_snooze_modal.py`, and
`beads_rendering.py` each import other snooze helpers only.

## Tests

Two existing assertions depend on the row and must change; no test imports the deleted
helpers directly, so nothing else moves.

### `tests/ace/tui/test_artifacts_beads_rendering.py`

`test_detail_drops_empty_property_rows_for_a_sparse_task` (line ~283) lists `Readiness`
among the labels that must be present (line 289). Move it: drop it from the
present-labels tuple and add `"Readiness"` to the absent-labels tuple in the same test,
so the removal is regression-locked on both surfaces' shared property builder.

### `tests/test_bead/test_snooze_surfaces.py`

This module's docstring — "a surface that renders a snoozed bead without [the wake
conditions] is a bug rather than a style choice" — stays true and must stay true:

- `test_the_beads_pane_detail_carries_a_wake_chip_and_a_snooze_property` (line ~146):
  `assert "snoozed · wakes in 3d" in rendered` (line 161) passes only because of the
  readiness chip. Replace it with an assertion on the `Snooze` row's own text, which
  `snooze_text()` renders as
  `2026-08-09 12:00:00 EDT · in 3d · +1 target: 2 more (3 total) · waiting on the upstream fix`.
  Print the captured render first and pick a substring that survives the 120-column
  grid's folding (`"· in 3d"` is the safe anchor) rather than pinning the whole string
  blind. Keep the existing `"◈ in 3d"` (title chip) and `"Snooze"` assertions, and add
  `assert "Readiness" not in rendered`.
- `test_the_beads_pane_preview_repeats_the_snooze_line` (line ~165): keep the
  `**Snooze:** 2026-08-09 12:00:00 EDT · in 3d` assertion, and replace line 168 with
  `assert "**Readiness:**" not in preview`.

## PNG Visual Snapshots

The detail property grid loses a row, so every golden that frames it shifts up one line:

- `tests/ace/tui/visual/snapshots/png/artifacts_beads_populated_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_beads_collapsed_relations_120x40.png`
- `tests/ace/tui/visual/snapshots/png/artifacts_beads_reopened_detail_120x40.png`

`artifacts_beads_empty_120x40.png` renders the empty-detail placeholder and should not
change — confirm that rather than assuming it. Goldens whose names mention beads but
which do not use `bead_properties_header()` (`notification_beads_*`,
`agents_phase_bead_*`, `agents_task_bead_notes_*`, `wait_modal_beads_focused_*`) must
not change either; a diff there means something unexpected happened, so stop and
investigate instead of updating it.

Procedure:

```bash
just test-visual tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads.py \
                 tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads_reopened.py \
                 tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads_empty.py
# inspect .pytest_cache/sase-visual/ actual/expected/diff artifacts and confirm the only
# delta is the removed row plus the one-line upward shift
just test-visual -- --sase-update-visual-snapshots \
                 tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads.py \
                 tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads_reopened.py
just test-visual   # whole PNG lane green, and `git status` shows exactly the goldens above
```

Prefer the scoped update over `just update-visual-snapshots`, which regenerates the
entire 480-file corpus and would hide an unrelated drift inside this change.

## Out Of Scope

- **No feature flag.** This is a ready-on-landing removal of a redundant read-only row,
  not a disabled beta, an early-landed path, or a deprecation whose old branch must stay
  reachable — so it is not flag-worthy under `sase/memory/sase_flags.md`.
- **No Rust change.** `ready()`/`blocked()` in `sase-core` keep their current meaning;
  `ready_ids`/`blocked_ids` stay in `BeadsSnapshot` and keep feeding the list glyph, the
  `status:blocked` filter, `_artifacts_beads_launch.py`, and `beads_navigation.py:481`.
  Only Rich/Textual presentation changes, which is the Python side of the core-backend
  boundary.
- **No docs change.** `grep -rn "Readiness" docs/` matches only "Provider Readiness" and
  "Hook Readiness", which are unrelated. No doc describes the bead detail property grid.
- No CHANGELOG edit — release-please owns it.

## Verification

```bash
just install    # ephemeral workspace: deps may be stale
just check
just test tests/ace/tui/test_artifacts_beads_rendering.py \
          tests/test_bead/test_snooze_surfaces.py \
          tests/test_bead_time_surface_coverage.py \
          tests/test_timezone_display_artifacts.py
```

Then the PNG procedure above. Before landing, hand `just check-full` to `/sase_monitor`
with a `--next` action — it routinely outruns a single agent turn and must never be run
inline. `just check`'s symvision stage is the gate that proves the three deletions were
complete; if it reports another newly-unused symbol, delete that too rather than
whitelisting it.

## Acceptance Criteria

- No `Readiness` row in the Beads detail property grid and no `**Readiness:**` line in
  the bead preview markdown, for every bead status including `snoozed` and for a `None`
  snapshot.
- A snoozed bead still shows its wake conditions on both surfaces (title `◈ in 3d` chip
  plus the `Snooze` row / `**Snooze:**` line).
- A blocked bead is still identifiable from the list glyph, the `Dependencies` row, and
  `status:blocked`.
- `grep -rn "readiness_chip\|readiness_label\|snooze_readiness_label\|Readiness" src tests`
  returns nothing outside the new negative assertions.
- `just check` green (symvision included), the two updated test modules green, the three
  PNG goldens regenerated and the rest of the visual lane unchanged, and
  `just check-full` green via `/sase_monitor`.

## Risks

- **Incomplete deletion.** Removing only the two call sites leaves three public symbols
  with no consumer and fails `just _lint-symvision`. The three-file deletion above is
  the whole change, not an optional cleanup.
- **Test assertions that pass for the wrong reason.** `"snoozed · wakes in 3d"` in the
  detail test is produced by the readiness chip, not by the `Snooze` row; simply
  deleting the assertion would drop real coverage of the snooze surface invariant.
  Re-anchor it on the `Snooze` row instead.
- **Golden churn.** Updating the whole PNG corpus would mask unrelated renderer drift;
  scope the update to the two goldens that legitimately change and verify the third
  (`artifacts_beads_empty`) does not.
