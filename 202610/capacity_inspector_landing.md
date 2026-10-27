---
tier: tale
title: Finish capacity inspector correctness and land sase-1ox
goal:
  Correct the remaining capacity presentation defects, verify integration, and close
  sase-1ox with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.sase-1ox.land
proposed_with: codex/gpt-6-astra
bead: sase-1ox
status: done
---

- **PARENT:**
  [202610/capacity_inspector_redesign.md](https://github.com/sase-org/sase--plans/blob/main/202610/capacity_inspector_redesign.md)
- **BEAD:**
  [sase-1ox](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ox/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1ox.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1ox.land.md)
- **COMMITS:**
  - [83b1822](https://github.com/sase-org/sase/commit/83b1822411e5495f0c26b42defd46e2b07ad5629)
    — fix(ace): land remaining capacity inspector presentation defects

# Finish the capacity inspector and close sase-1ox

The capacity redesign is implemented, but its landing review found a handful of
presentation defects that violate the approved design. Fix these in one coding turn,
verify the result, and finish the epic's landing in that same turn. This tale has no
land agent or automatic continuation back to the original lander.

## Scope and inherited decisions

The epic is `sase-1ox`, whose plan is `plan:202610/capacity_inspector_redesign.md`. Its
file is `202610/capacity_inspector_redesign.md` within the plans repository. Resolve the
actual path through `sase bead read` and `/sase_repo`; do not assume a checkout
location.

Read the epic and its latest notes, all three children (`sase-1ox.1`, `.2`, `.3`), and
the original plan using `sase artifact read` before implementing. The final decisions
remain **cli_parity = adopt** and **inactive_records = collapsed**. Keep the adopted CLI
order, folded inactive groups, exact JSON copy/output, stable selection, and
authoritative Rust accounting. This is presentation work: do not change admission, claim
ownership, queue semantics, the Rust wire contract, or the core revision pin.

Read the required ACE, TUI/performance, screenshot, Symvision, and lint/test reference
memories through `/sase_memory_read`. If documentation needs correction, read
`docs_writing.md` first. Keep new or modified Python modules under 700 lines; the modal
is already close to that limit, so extract cohesive rendering/option-building code if
necessary. Preserve lazy imports and the existing chip startup behavior.

## Completed landing review and follow-up disposition

The lander reviewed every child note and the three epic commits:

- `d16b79f3c1`: view model, tiers, verdict, gauge, durations, lazy chip constants.
- `c8bc7a58e2`: inspector, interactions, screenshot settling, five goldens, docs.
- `daa96a5bd0`: adopted CLI parity and legacy-field cleanup.

At `daa96a5bd0`, 82 focused capacity/view/CLI/modal/chip/queued-refresh/lineage tests
passed. Stable selection also survived a report refresh after the Textual event pump
drained. All five committed inspector PNGs were inspected. They demonstrate the missing
third detail line, and the narrow busy PNG demonstrates clipped facts.

The sole child `PROPOSED FOLLOW-UP` is `sase-1ox.1` note 2: the startup import budget.
The lander completed `/sase_new_task` triage and independently reproduced
`test_tui_app_import_stays_under_startup_budget` at 3541 modules against strict 3539.
`tools/tui_import_closure --diff d16b79f3c1^` measured 3541 both on the reviewed tip and
on pre-epic `a42eedc1c467`, with no added or removed `sase.*` modules. Existing owner
`sase-1ic` received verified-after-close corroboration, taking it to 32 independent
reports. No duplicate task was created and no proposal was declined. The relevant recent
task search and active startup epic `sase-132.4`/`.7` review found no causal ownership
for this recurrence beyond that existing task. This outcome is recorded on `sase-1ox`;
include it in the eventual close note. Do not duplicate the task or raise the import cap
as part of this tale. Preserve the actual verification exit status and follow the
repository's KNOWN-failure policy if this independent failure appears.

The integration review covered `c328233430` (live-PID persisted-lineage fixture), the
only non-epic commit after the first epic commit, and all changes since epic creation:
`3b99dac0a0` (wait labels), `353d833bf0` (GC quiet window), `2243ebc97b` (retried
handoff waits), `691ef46153` (Files scroll), `8d41b72218` (queued-session/clan refresh),
and `a42eedc1c4` (monitor capacity test isolation). No new production consumer or
conflict requiring integration was found. The queued-refresh and lineage regressions
passed. After fetching, `origin/master` and HEAD were both `daa96a5bd0`. Recheck later
drift before close, integrating anything newly relevant.
`sase bead epic-symbols sase-1ox` was empty. The epic has **no parent_bead** as of this
review.

## 1. Make every empty state truthful

Both `CapacityInspectorModal._build_options` and `cli_load._charged_table` currently
print `Nothing holds capacity — all 8 units are free` for a report containing only an
invalid-weight record, even though `accounting_valid` is false. This contradicts the
blocked verdict and the epic's requirement that unknown capacity never read as free.

Use the all-free message only when valid, available accounting proves it. For the
invalid empty charged set, say `No known charged claims — free capacity is unknown` (or
equivalent equally explicit wording). Keep attention entries and diagnostics visible.
Retain the separate totals-only unavailable state. Cover both the real CLI output and
modal options with an invalid-only report built through the report helper; also prove a
genuinely valid empty report still shows all-free capacity.

## 2. Keep snapshot ages deterministic and project labels consistent

`CapacityInspectorModal._project_view` passes `now=self._clock()` to `capacity_view`. A
holder whose report says it has run 180 seconds therefore reads 300 seconds when the
modal opens 120 seconds after computation. Folding a group reprojects with a new time as
well. The approved design requires row durations relative to `report.computed_at`; only
freshness follows the live clock.

Project the view at the report timestamp. Preserve the injectable live clock for the
headline freshness ticker. Use the view's projected override age/provenance for facts,
and reuse `CAPACITY_REPORT_STALE_AFTER_SECONDS` and
`CAPACITY_OVERRIDE_WARNING_AFTER_SECONDS` instead of the renderer's private 25-second
and 14-day copies. Avoid changing row ages or override validity merely because the user
folds a group or selects a row. Add a delayed-report/fake-clock regression that checks
holder and waiter ages through folding/refresh while the freshness label ages.

In `_capacity_inspector_render.render_detail`, the rider branch prints `row.project_key`
directly while holders and queued rows call the sanctioned project label helper. Resolve
riders through that same label path and omit empty label separators. Test with a
canonical key deliberately different from its human label. Keep rendering and
highlighting free of filesystem work; use the existing display snapshot/cache contract
rather than deriving labels from paths or keys.

## 3. Finish the promised row grid and visible detail strip

`render_capacity_row` computes name width from each row's note length, then appends an
unpadded name. At width 110, `worker-kind` starts at column 12 for `short` and column 30
for `much-longer-worker-name`. Use column widths derived from the available content
width, consistently across rows: four-cell mark, bounded middle-truncated name, 16-cell
kind at widths >= 96, note, and a reserved right-aligned age. Pad the name to its
allocated cell width. Keep narrow kind omission and the 40/28-cell name caps. Budget
separators and Unicode cell widths accurately so rows fit. Preserve queue tones (the
existing TUI blocker text inherits its row tone) and inactive dimming. Restore the
missing `inherited` holder note alongside explicit weight, budget, and rider tags. Test
alignment across different name/note lengths and terminal-cell widths, not just string
containment.

The CSS detail widget has `height: 3` **including** a top border, leaving only two
content rows. All five goldens omit the final line with sharing/budget/jump information;
queued rows likewise lose their blockers. Allocate three content lines plus the border,
handle wrapping at the supported narrow width, and verify the actual visible widget
region. Make narrow headline/facts layout show complete freshness and summary
information instead of silently clipping it; the approved design allows header widgets
to size to their content. Keep the list as the sole scrolling list and keep the queued
section visible for the busy 80x32 fixture.

Make the facts count agree with the zero-charge section: the busy fixture has seven
charged holders, one zero-weight monitor, and one serial rider, so the approved summary
is `9 running · 7 charged · 2 at 0`, not the current `1 at 0`. Count both zero-weight
claims and serial riders in that display summary without altering capacity totals. Add a
direct assertion for this case.

`render_verdict` currently appends the legacy `over_limit_note` after the new verdict
detail, repeating the over-budget explanation and consuming another header line. It
similarly repeats the invalid-accounting explanation. Render the new title/detail once.
Keep any legacy view fields still consumed by the chip, notably `over_limit_note`; this
is not permission to delete live fields.

## 4. Preserve Rich styling in CLI parity

`cli_load._attention_text`, `_zero_weight_text`, `_waiting_text`, and
`_diagnostics_panel` build styled `Text` then return `Text(str(text).rstrip(...))`,
which drops all style spans. Trim the original Rich text without flattening it. Retain
attention/diagnostic warning colors, zero-weight dimming, and section/name emphasis. For
invalid accounting, known used gauge cells should use the rejected style as in the
inspector. Test the resulting styles using Rich spans or rendered segments with a
color-enabled console. Keep the CLI's section order, ignored inactive count, and
unchanged JSON contract.

## 5. Verify the repair and the integration

Add focused regressions for the defects above to the existing capacity view, CLI, modal,
and renderer tests. Exercise selection/folding after queued messages drain so the layout
fixes do not regress stable identity or scroll-to-highlight. Use actual production
styles for the detail-height assertion; the small `_InspectorApp` tests without the
production stylesheet cannot detect the clipping bug.

Improve the busy visual fixture's currently identical holder start timestamps to
exercise staggered relative durations and sort order. Keep host-independent UTC/clock
pinning. Reuse the existing five snapshots and add only targeted cases needed to show
the invalid-only empty state or the previously hidden detail content. Update the
capacity inspector goldens, inspect every changed image, and confirm the wide and narrow
fixtures visibly retain the queue, all three detail lines, and readable facts. Run the
targeted screenshot checks in the default timezone and `TZ=UTC` according to the
screenshot memory. Use `/sase_monitor` if these commands need a handoff, with a
successor explicitly instructed to continue this tale's closeout.

Run the focused regressions plus the existing chip, inspector-opener,
`test_capacity_roster_refresh_queued_status.py`, and
`test_run_agent_runner_slot_lineage.py` tests. Run `just fix`, inspect its changes, then
the mandatory `sase tool run check` (the wrapped `just check`). Do not run
`just check-full`. Do not attach prepared completion that would end the work before
golden review and epic closure; if verification is monitored, its follow-up must finish
these steps. No step depends on this tale's future commit, SHA, push, or CI.

## 6. Close sase-1ox in this coding turn

This is a required implementation step, not a suggestion for another land agent.

1. Re-read `sase-1ox` and all descendants/notes and the original linked plan. Recheck
   commits since `daa96a5bd0` on the current branch/base for relevant drift. Verify the
   above fixes, preserved decisions, and recorded follow-up triage. Finish any epic
   defects before proceeding. If the tale itself has an assigned descendant bead,
   complete it normally before closing its parent so descendant validation can pass.
2. Run `sase bead epic-symbols sase-1ox`. Resolve every listed exemption by wiring,
   privatizing, adding a justified non-test pragma, or deleting the symbol under the
   Symvision policy. Re-key only when a named still-open later bead truly needs the
   exemption. Re-run required file-change verification after any cleanup edits.
3. Run `sase bead close sase-1ox --note "<verification>"`, replacing the placeholder
   with the reviewed phases/commits, approved decisions, repaired defects, actual
   verification results, post-epic integration outcome, and the `sase-1ic` follow-up
   disposition above. If symbols or unfinished phases block the close, finish them and
   retry. Never force a successful nested landing or force merely to pass validation;
   canceled/superseded descendants require an intentional justified outcome.
4. After the successful close, run `just symvision` and confirm no stale whitelist
   remains. Open the plans repository through `/sase_repo`, read the original plan
   through `sase artifact read plan:202610/capacity_inspector_redesign.md`, and set
   `status: done` in that file's frontmatter. Include the changed plans repository in
   the final declaration.
5. Run `sase bead read sase-1ox -r "Need the parent link after close"`. This epic has no
   parent link in the reviewed state, so no ancestor close is due. If one has been added
   meanwhile, follow the landing rules: normally close only a completed parent phase;
   for a parent plan, re-audit its prior landing note, all descendants, linked plan
   readiness and drift, clean its epic symbols, close normally, run Symvision, and mark
   its plan done. Stop at the first incomplete or ambiguous ancestor and record its
   blocker. Never force an ancestor close.
6. Finish with `/sase_final` so the host commits all changed repositories. Do not wait
   for that host commit before closing the epic or marking its plan done.
