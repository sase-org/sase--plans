---
tier: tale
title: Show a tale plan's size in the ACE PLAN lane
goal: "Give a tale plan's required `size` a first-class, honest representation in the
  PLAN lane of the Agents-tab SASE CONTEXT panel, matching the size chip the BEAD lane
  and epic phase rows already render.

  "
size: medium
proposed_by: bbugyi200.athena.sase-il.land.f1
create_time: 2026-08-10 11:00:22
status: wip
---

# Plan: Show a tale plan's size in the ACE PLAN lane

## Problem

`size` is a **required** tale frontmatter field (`sase plan validate` rejects a tale
without it) and it selects the follow-up worker's default model through
`@<size>_phase_worker` routing. Despite that, the ACE Agents-tab SASE CONTEXT panel
never shows it. A tale plan's PLAN lane renders only:

```
▸ PLAN · tale
  Title: Probe tale
   Goal: Confirm whether tale size is represented.
   Path: /tmp/.../probe.md
```

Verified by rendering a real `size: medium` tale through
`sase.sdd.plan_display.render_plan_lines`: the size is absent from every line, and
`PlanDisplay` has no `size` attribute at all.

The data is already available and authoritative — the Rust validator returns it as
`_ValidatedPlan.size` (`src/sase/sdd/plan_validate.py:71`) — but the Python display
layer drops it: neither `PlanFileMetadata` nor `PlanDisplay`
(`src/sase/sdd/_plan_display_models.py`) carries a plan-level size, and
`plan_file_metadata_from_content` reads only `validation.plan.title` and
`validation.plan.goal` off the validated plan.

This makes tale plans the only sized SASE unit with no size representation:

| unit              | surface             | representation                                       |
| ----------------- | ------------------- | ---------------------------------------------------- |
| epic phase        | PLAN lane phase row | right-aligned `phase_size_chip`                      |
| task bead         | BEAD lane           | `Size:` row with `phase_size_chip`                   |
| phase bead        | BEAD lane           | `Size:` row with `phase_size_chip`                   |
| bead detail panel | Artifacts tab       | `Size` property chip                                 |
| `sase bead show`  | CLI                 | accented `Size:` line, `(default)` marker when unset |
| **tale plan**     | **PLAN lane**       | **none**                                             |

## Approach

Thread the plan-level size through the shared plan-display value and render it as a
` Size:` field row carrying a `phase_size_chip`, gated on `authored_tier == "tale"`.

Three design decisions, each with a reason:

1. **A field row, not the lane header.** `plan_lane_details` builds the dim
   `▸ PLAN · tale` header line as plain `Text`; it cannot host a background-colored chip
   without restructuring three renderers (`ResponsivePlanSection.__rich_console__`,
   `render_plan_document`, `plan_logical_text`) into tables. A field row reuses the
   exact `Size:`-row shape the BEAD lane already uses, and — unlike the header — it
   survives the clan-summary callers that replace the lane header with their own (which
   is why `plan_field_rows` already has the `include_counts` escape hatch for the phase
   count). Do not also add the size to the header: the existing `include_counts` comment
   states the deliberate rule that a value must not be shown twice.

2. **Distinguish authored from defaulted.** `validate_plan(..., mode="launch")` silently
   rewrites a legacy sizeless tale to `medium` and emits a `tale-size-missing` warning
   (`_launch_mode_compatibility_content`). Rendering that fabricated `medium` as if the
   author chose it would be dishonest. `sase bead show` already solved this: it appends
   `PHASE_SIZE_DEFAULT_MARKER` (`"(default)"`) when a bead has no stored size
   (`src/sase/bead/cli_detail.py:525`). Reuse that constant and idiom.

3. **Tale-only.** The epic frontmatter schema has no top-level `size` (confirmed against
   `plan_frontmatter_schema("epic")`); epic sizes live per phase and already render.
   Gating on `authored_tier == "tale"` keeps the epic lane byte-identical.

## Steps

### 1. Carry the size on the shared display values

In `src/sase/sdd/_plan_display_models.py`:

- Add `size: PhaseSizeValue | None = None` and `size_defaulted: bool = False` to both
  `PlanFileMetadata` and `PlanDisplay`. Both dataclasses are frozen with trailing
  defaulted fields already, so appending defaulted fields is source-compatible with
  every existing constructor.
- Document `size_defaulted` in the field's context: it means launch-mode compatibility
  supplied the value because the tale declared none.

In `src/sase/sdd/_plan_display_loading.py`:

- In `plan_file_metadata_from_content`, capture whether the raw frontmatter declared a
  size (`"size" in frontmatter`) _before_ validation, and seed
  `size = normalize_phase_size(frontmatter.get("size"))` alongside the existing
  `title`/`goal` seeds so an invalid-but-parseable tale still shows what it declared.
- When `validation_ok` and `authored_tier == "tale"`, overwrite `size` with
  `normalize_phase_size(validation.plan.size)` (authoritative), and set
  `size_defaulted = not frontmatter_declared_size`.
- Force `size = None` / `size_defaulted = False` when `authored_tier != "tale"`, so an
  epic never grows a plan-level size.
- Pass both new values through the `PlanFileMetadata` construction at the end of the
  function, through `unavailable_plan_metadata` (explicitly `None` / `False`), and
  through the `PlanDisplay` construction in `load_plan_display`.

In `src/sase/ace/tui/models/_agent_associated_plan_summary.py`:

- `build_associated_plan_summary` builds the `AssociatedPlanSummary` (= `PlanDisplay`)
  from a `PlanFileMetadata`; forward `size=metadata.size` and
  `size_defaulted=metadata.size_defaulted`. Without this the TUI lane silently keeps
  `None` even though loading resolved a size — this is the site the PLAN lane actually
  reads.

### 2. Render the Size row

In `src/sase/sdd/_plan_display_rendering.py`:

- Add a `_size_value(summary)` helper returning
  `phase_size_chip(summary.size, unavailable_style=COLOR_PLAN_EMPTY)`, appending
  `f" {PHASE_SIZE_DEFAULT_MARKER}"` in `COLOR_PLAN_EMPTY` when `summary.size_defaulted`.
  Use the unpadded chip (no `width=`), matching the BEAD lane's `_size_value`, so the
  badge hugs its label instead of trailing dead colored space.
- In `plan_field_rows`, insert `("   Size: ", _size_value(summary))` after the Goal row
  and before the optional Counts row, only when `summary.authored_tier == "tale"`.
  `"   Size: "` is 9 cells, equal to `PLAN_FIELD_LABEL_WIDTH`; assert this in tests the
  way `test_plan_display.py` already asserts it for `" Counts: "`.
- Keep Path last. `render_plan_document` computes its basename wrap hint from
  `index == len(field_rows) - 1` and `plan_show_render._properties` reads
  `field_rows[-1]` for the path; inserting before Path preserves both.
- No change is needed in `plan_logical_text`, `render_plan_document`,
  `ResponsivePlanSection._rows`, or the clan-summary scripts — they all iterate
  `plan_field_rows`, so the row appears in the TUI lane, the zoom/copy logical text, and
  the clan summary automatically.

Because the chip is a fixed-width, non-wrapping `Text`, confirm the row renders sanely
at the narrow widths `test_agent_display_plan_section.py` already exercises; if the chip
must not wrap, set `no_wrap=True` on the value the way `phase_size_chip` already does
and verify `_render_field_lines` leaves it on one line.

### 3. Give `sase plan show` the same row

`src/sase/main/plan_show_render.py` states in its module docstring that it shares the
PLAN lane's palette and row styling "so the two surfaces read as one visual family".
Adding a row to `plan_field_rows` does not reach it, because `_properties` selects rows
by index (`[0]`, `[1]`, `[-1]`).

- In `_properties`, add a `size` row for tales, sourced the same way as `title`/`goal`
  (look the row up by its `"   Size: "` label rather than by a new hard-coded index, so
  future row insertions cannot silently reorder the section).
- `_plan_display` bridges `PlanShowRecord` into `PlanDisplay`; populate `size` from
  `plan.frontmatter.get("size")` via `normalize_phase_size` and set `size_defaulted`
  from whether `"size"` is absent while the tier is `tale`, so the CLI and TUI agree.

Leave `--format json` alone: it already exposes the raw `frontmatter` mapping.

### 4. Tests

- `tests/test_plan_display.py`: the row-label assertions at lines ~418, ~431-434 and
  ~478 enumerate `["  Title: ", "   Goal: ", "   Path: "]` and the Counts variant, and
  will fail; update them and add tale-specific coverage — an authored size renders the
  chip, a legacy sizeless tale renders `medium (default)`, an epic renders no Size row
  at all, and an unreadable/invalid plan renders `unavailable` rather than a fabricated
  size.
- `tests/ace/tui/widgets/_agent_display_plan_helpers.py`: add a `size` parameter to
  `plan_summary(...)` (defaulting to a declared tale size) and keep `epic_summary`
  size-free.
- `tests/ace/tui/widgets/test_agent_display_plan_section.py`: assert the Size row
  appears in the tale lane in canonical order and that the chip carries the
  `PHASE_SIZE_STYLES[...]` style (the file already imports `PHASE_SIZE_STYLES`), plus a
  regression that the epic lane is unchanged.
- `tests/test_clan_summary_plan_script.py` and
  `tests/test_bead/test_clan_summary_epic_plan_script.py` construct `PlanDisplay`
  directly; the new defaulted fields keep them compiling, but check any golden text they
  compare against.
- PNG visual snapshots:
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py` writes a
  `tier: tale` fixture with **no** `size`, so it would newly render `medium (default)`.
  Add `size:` to that fixture so the snapshot demonstrates a real authored size, then
  regenerate the affected goldens under `tests/ace/tui/visual/snapshots/png/` with
  `just test-visual --sase-update-visual-snapshots` and eyeball the diffs in
  `.pytest_cache/sase-visual/`. Also re-check the tale fixtures in
  `test_ace_png_snapshots_agents_zoom.py`, `test_ace_png_snapshots_agents_neighbors.py`,
  `test_ace_png_snapshots_preview_panel.py`, and
  `test_ace_png_snapshots_agents_clan_panel.py`, since the extra row shifts subsequent
  lines by one.

Finish with `just check-full` (the change touches shared display code consumed by the
CLI, the TUI, and the clan-summary scripts), plus `just test-visual`.

## Out of scope

- The Rust core still accepts `size: large` and `size: xlarge` on tales even though
  `sase/memory/sase_sizes.md` declares only `xsmall | small | medium` valid. That gap is
  already owned by the open follow-up plan for epic `sase-il`
  (`finish_tale_size_semantics`); this plan renders whatever the validator normalizes
  and must not change validation semantics.
- The tale-level `model` field is likewise unrepresented in the PLAN lane. It is a
  separate, smaller gap; do not bundle it here.
