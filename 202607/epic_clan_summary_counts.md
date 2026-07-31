---
tier: tale
title: Epic clan summary phase and wave counts
goal:
  The epic clan summary shows a single-line `Counts:` field above `Path:` that reports the authored plan's phase count
  and dependency-wave count, backed by one shared wave-layering helper that `sase bead work` and the renderer both use.
create_time: 2026-07-31 09:17:30
status: wip
---

- **PROMPT:** [202607/prompts/epic_clan_summary_counts.md](prompts/epic_clan_summary_counts.md)

# Plan: Epic clan summary phase and wave counts

## Context

The epic clan summary is the Rich markup an epic clan's launch stores in `clan_summary` and the ACE agents panel replays
in `build_clan_detail_text` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`).
`src/sase/scripts/sase_clan_summary_epic.py` produces it. That script has two rendering paths:

1. **Plan-backed** (`_render_plan_summary`, used whenever `SASE_EPIC_PLAN_REF` resolves to a valid authored plan). It
   calls the shared renderer `render_plan_document` in `src/sase/sdd/_plan_display_rendering.py` and then **replaces**
   the document's first intro line — the PLAN lane header `▸ PLAN · epic · 3 phases` — with `◆ EPIC <epic-id>`.
2. **Bead-backed** (`_render_epic_summary`, the fallback when no usable plan reference exists). It renders its own
   `PHASES · <closed>/<total> done at launch` and `CHILD EPICS · <n>` headings and has no `Path:` row.

Path 1 is the one this tale changes. Swapping the lane header for the epic header silently drops the only phase count
the shared renderer emitted, so today a plan-backed epic clan summary reports no counts at all:

```
◆ EPIC sase-6n
  Title: Rich clan summaries
   Goal: Share launch context with stable authored phase details.
   Path: sase/repos/plans/202607/epic_clan_summary.md
   Bead: sase-6n
  1 ◆ Persist summaries at launch                                    small
    runtime · no dependencies · model codex/gpt-5
...
```

The reader must count phase blocks by hand, and nothing at all conveys how much of the epic can run in parallel.

The shared renderer builds three canonical field rows in `plan_field_rows` — ` Title:`, `  Goal:`, `  Path:` — each
label padded to `PLAN_FIELD_LABEL_WIDTH` (9 cells). `render_plan_document` wraps those rows, appends the provenance rows
(`Plan:`, `Prompt:`, `Parent:`, `Bead:`, `Agents:`, `Commits:`) and the optional `Page:` row into `intro_lines`, and
renders one independently omittable block per phase. `plan_field_rows` is also consumed by the Agents-panel PLAN lane
(`src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py`) and by the plan clan summary script
(`src/sase/scripts/sase_clan_summary_plan.py`) — both of which keep the real lane header and therefore already display
the phase count.

"Waves" is established SASE vocabulary: `sase bead work` layers an epic's phase DAG Kahn-style, where wave _k_ holds
every phase whose dependencies all land in earlier waves. `preview_waves` in
`src/sase/bead/cli_work_from_plan_helpers.py` already computes exactly that from a validated plan's phases, and
`PlanDisplayPhase` (`src/sase/sdd/_plan_display_models.py`) carries the same `id` and `depends_on` fields the layering
needs. No new data source, bead lookup, or filesystem access is required to count waves at render time.

## What the field must show

Insert one new canonical field row, `Counts:`, between `Goal:` and `Path:`:

```
◆ EPIC sase-6n
  Title: Rich clan summaries
   Goal: Share launch context with stable authored phase details.
 Counts: 3 phases · 3 waves
   Path: sase/repos/plans/202607/epic_clan_summary.md
   Bead: sase-6n
```

Requirements:

- The label is `" Counts: "` — exactly 9 cells, so it aligns with the existing labels without touching
  `PLAN_FIELD_LABEL_WIDTH`. Verify the alignment rather than assuming it.
- The value is one line at the 76-column summary width: the phase count, the standard `·` separator already used
  throughout this renderer, then the wave count. Both counts are pluralized correctly (`1 phase · 1 wave`).
- Style the value in two tones drawn from the existing plan palette: the numerals in `COLOR_PLAN_PRIMARY` so they are
  the thing the eye lands on, and the unit words plus the separator in `COLOR_PLAN_SUMMARY` so the row stays quiet next
  to `Title:` and `Goal:`. Do not introduce a new color constant unless the two-tone treatment genuinely needs one.
- Counts are structural properties of the authored plan, not live progress. Do not add bead status, closed counts, or
  anything that would require a store read — the summary is captured once at launch and must stay render-pure.

Degraded states, which must never raise and never render a misleading number:

- **Tale plans** (`phase_availability == "not-applicable"`): omit the row entirely. Tales have no phases, and an empty
  or zeroed row would be noise.
- **Phase data unavailable** (`phase_availability == "unavailable"` — a missing, unreadable, or invalid plan): render
  `unavailable` in `COLOR_PLAN_EMPTY`, matching how `_title_value` and `_goal_value` already report missing data and how
  `plan_lane_details` already says `phases unavailable`.
- **Phases present but unlayerable** (a dependency cycle or a dependency on an id outside the plan): keep the honest
  phase count and render the wave half as `waves unavailable` in `COLOR_PLAN_EMPTY`. Strict validation rejects both
  shapes, so this path exists only to guarantee the renderer degrades instead of hanging or throwing if the validator
  contract ever regresses.

## Share one wave-layering implementation

Add `src/sase/sdd/plan_waves.py` holding the single layering implementation this repo uses for authored plans:

- A public `plan_phase_waves(phases)` that takes any iterable of authored phases and returns
  `tuple[tuple[str, ...], ...] | None` — the phase ids grouped into dependency waves, preserving authored order both
  across and within waves, or `None` when a cycle or out-of-plan dependency leaves phases unlayerable. Returning `None`
  rather than raising keeps the display caller total while letting the CLI caller decide the failure is an error.
- A **private** structural Protocol for the phase argument so both `PlanDisplayPhase` and `ValidatedPlanPhase` satisfy
  it without a shared base class. Declare its `id` and `depends_on` members as read-only properties, not bare attribute
  annotations: frozen dataclasses do not satisfy a Protocol that declares mutable attributes, and mypy will reject the
  call sites if you get this wrong. Keeping the Protocol private also keeps symvision quiet, since it is referenced only
  inside its defining module.

Then rewrite `preview_waves` in `src/sase/bead/cli_work_from_plan_helpers.py` to delegate: call `plan_phase_waves`, and
raise the existing `PlanFileWorkError("validated epic plan contains a dependency cycle")` when the helper returns
`None`. Preserve that message and the comment explaining why the defensive boundary exists — `sase bead work` behavior
must not change at all. The one intentional difference is that keying the traversal order off the phase map, rather than
off a separately built id list, stops a duplicated phase id from being emitted twice in a wave; validation already
rejects duplicate ids, so this only tightens an unreachable edge.

This replaces one implementation with one implementation. Do not add a second copy of the layering loop in the rendering
module.

## Render the Counts row

In `src/sase/sdd/_plan_display_rendering.py`:

- Give `plan_field_rows` a keyword-only `include_counts: bool = False` and return a variable-length tuple: Title, Goal,
  the Counts row when opted in and the plan is not a tale, then Path. Document in the docstring that the flag exists for
  callers that replace the PLAN lane header, so the count that header carries stays visible; that is the whole reason
  the row is not unconditional.
- Add a private `_counts_value(summary)` implementing the value spec and degraded states above, and factor the existing
  `_count_phrase` pluralization so the new two-tone rendering and the existing `plan_lane_details` phrasing share one
  rule instead of duplicating the plural suffix.
- Thread `include_counts` through `render_plan_document` (also keyword-only, defaulting to `False`) to
  `plan_field_rows`. Leave `render_plan_lines` and `plan_logical_text` on the default so the copy/search text and the
  plan clan summary are untouched.
- `render_plan_document` derives the path row's preferred basename wrap hint from the **last** field row. Inserting
  Counts above Path keeps Path last, so the existing logic stays correct — but lock that invariant down with a test
  rather than leaving it as an accident of ordering.

Export the new row behavior through the `src/sase/sdd/plan_display.py` facade only if a new public symbol actually needs
to cross it; the flag alone does not.

## Opt the epic clan summary in

In `src/sase/scripts/sase_clan_summary_epic.py`, have `_render_plan_summary` pass `include_counts=True` when it calls
`render_plan_document`. Nothing else in that script changes: the intro-line substitution, the UTF-8 byte budget, the
phase-block omission logic, and the bead-backed fallback all stay as they are. The Counts row lives in `intro_lines`,
which the byte-budget fitter always keeps, so a very large epic that sheds phase blocks still reports its totals.

Deliberately **not** in scope, and the reason for each:

- The plan clan summary (`sase_clan_summary_plan.py`) and the Agents-panel PLAN lane both keep the real
  `▸ PLAN · <tier> · N phases` header, so adding Counts there would print the phase count twice in adjacent lines.
- The bead-backed epic summary already prints `PHASES · <closed>/<total> done at launch` and has no `Path:` row to sit
  above.
- `plan_lane_details` keeps its `N phases` suffix. Moving that count into the field rows for every caller is a larger,
  separate design question and would churn goldens well beyond this feature.

## Rust core boundary

The repo's standing rule sends shared backend and domain behavior to `sase-core`, and wave layering over a _bead_ DAG
already lives there in `crates/sase_core/src/bead/work.rs`. Layering over an _authored plan's_ phases, however, lives in
Python today (`preview_waves`), and the validated-plan wire exposes no wave information. Moving it would mean adding
waves to the plan wire, bumping `PLAN_WIRE_SCHEMA_VERSION` (which the Python adapter checks for exact equality), cutting
a `sase-core` release, and moving the published `sase-core-rs` pin — a lockstep multi-repo release for a display count.

So this tale keeps the layering in Python, and shrinks the footprint by making the existing Python copy the only one. Do
not add a Rust change here. Instead, file a `sase bead create -T task` bead (marked `ready`) proposing that plan-phase
wave layering move into `sase-core` alongside the bead-DAG layering the next time the plan validation wire is revised,
naming `src/sase/sdd/plan_waves.py` as the code that would be retired.

## Tests and verification

Cover the new behavior directly, and pin the decisions that would otherwise silently rot:

- New unit tests for `plan_phase_waves`: a linear chain, a diamond DAG that collapses to fewer waves than phases, an
  all-independent plan that layers into one wave, authored order preserved inside a wave, the empty input, a cycle
  returning `None`, and a dependency on an id that is not in the plan returning `None`.
- Regression coverage that `preview_waves` still returns the same waves and still raises `PlanFileWorkError` with the
  same message on a cyclic plan, so the `sase bead work` contract is provably unchanged.
- Renderer tests in `tests/test_plan_display.py`: the row is absent by default; with `include_counts=True` it appears
  between Goal and Path with a 9-cell label; Path remains the final field row in both modes; a tale plan omits the row;
  an unavailable-phase plan renders `unavailable`; a cyclic plan renders the phase count with `waves unavailable`; the
  value's plain text and pluralization are exact for 1-phase and multi-phase plans.
- Epic script coverage in `tests/test_bead/test_clan_summary_epic_plan_script.py`: a plan-backed epic summary contains
  the `Counts:` line with the expected counts immediately above the `Path:` line, and the row survives the byte-budget
  path that omits phase blocks.
- A guard in `tests/test_clan_summary_plan_script.py` that the plan clan summary still renders no `Counts:` row, so the
  no-duplication decision is enforced rather than remembered.
- The three epic clan-panel PNG goldens (`agents_clan_panel_epic_120x40`, `_level_2_`, `_level_3_`) embed a rendered
  epic clan summary and will shift by one line. Regenerate them with the `just update-visual-snapshots` recipe, then
  **visually inspect** each regenerated PNG and confirm the diff is only the new row — do not accept goldens blind. Add
  a plain-text assertion for `Counts:` alongside the existing `Title:`/`Page:` assertions in that test so a future
  regression fails on text rather than on pixels alone.

Run `just install` first (workspace dependencies drift), then `just check` and `just test-visual`, and fix everything
they report — including symvision, which requires every new public symbol to have a real non-test consumer.
