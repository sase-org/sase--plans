---
tier: tale
title: Show the selected phase bead's title in the Agents BEAD lane
goal:
  "The Agents metadata panel's SASE CONTEXT / BEAD lane names the phase it describes: a new leading `Phase Title` row
  renders the selected phase's validated, whitespace-normalized title, wraps losslessly, degrades to `unavailable` like
  every other optional field, and never leaks a peer phase."
---

- **AGENTS:**
  - [bbugyi200.athena.n1--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.n1.md#member-code)
  - [bbugyi200.athena.n1--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.n1.md#member-plan)
- **COMMITS:**
  - [4dcb779](https://github.com/sase-org/sase/commit/4dcb77960eb8484913c531cd53e64914f3231f42) — — feat(ace): show
    phase titles in BEAD context

# Plan: Phase title in the SASE CONTEXT / BEAD lane

## Problem

For an epic phase worker, the Agents metadata panel renders a phase-local `BEAD` lane derived from the parent epic's
validated, frontmatter-ordered phase entry:

```
▸ BEAD · phase sase-ag.5
  Description: reconcile: add sase plan links refresh for bulk, idempotent
               header reconciliation, migrate existing parent frontmatter into
               PARENT bullets, and deprecate the frontmatter property in the
               plan schema.
         Size: medium
    Epic Plan: plans:202607/plan_header_provenance.md
   Epic Title: Plan-file provenance header block
```

The lane already shows the phase's description, the phase's size, the parent epic's plan path, and the parent epic's
title — but not the phase's own `title`. That title is the one short, human-authored name for the unit of work the
selected agent is executing, and it is the field a reader scans for first. Today the only way to name the phase is to
read a wrapped multi-line description or open the epic plan.

The data is already available and costs nothing to surface. `sase-core` validates every epic phase's required `title`,
`sase.sdd.plan_validate.ValidatedPlanPhase.title` carries it into Python, and
`sase.sdd.plan_display.PlanDisplayPhase.title` already exposes it to the `PLAN` roadmap. Only the `BEAD` lane's
`PhaseBeadSummary` drops it on the floor.

## Design

Add one new field row, `Phase Title`, as the **first** body row of the `BEAD` lane:

```
▸ BEAD · phase sase-ag.5
  Phase Title: Plan-file provenance reconciliation
  Description: reconcile: add sase plan links refresh for bulk, idempotent
               header reconciliation, migrate existing parent frontmatter into
               PARENT bullets, and deprecate the frontmatter property in the
               plan schema.
         Size: medium
    Epic Plan: plans:202607/plan_header_provenance.md
   Epic Title: Plan-file provenance header block
```

Four decisions and why:

**Position — first.** The lane then reads strictly top-down from the phase outward: what this agent is doing
(`Phase Title`), the detail of it (`Description`), how big it is (`Size`), then where it came from (`Epic Plan`,
`Epic Title`). Phase-local fields group ahead of parent-epic provenance fields, which is the grouping the current order
already implies. It also mirrors the `PLAN` lane, whose first body row is `Title`.

**Label — `Phase Title`.** It pairs symmetrically with the existing `Epic Title`, which a bare `Title` would not: two
adjacent title rows must be self-disambiguating from their labels alone. `Phase Title` is 11 cells, exactly as wide as
the current longest label `Description`, so `BEAD_FIELD_LABEL_WIDTH` stays 15, the colon column stays at index 13, and
**no existing row moves horizontally**. The change costs zero horizontal budget.

**Style — `COLOR_BEAD_PRIMARY`** (`bold #FFD75F`), the same treatment the lane already gives the bead id in its header
and the `Epic Title` value. Titles are the lane's proper nouns; they share one style. No new palette entry.

**Fallback — `unavailable` in `COLOR_EMPTY`**, identical to `Description`, `Epic Plan`, `Epic Title`, and `Size`. A
missing, unreadable, damaged, or out-of-range phase entry keeps the honest identity/path fallbacks the lane already
guarantees rather than inventing a title.

### Alternatives considered and rejected

- **Put the title in the lane header** (`▸ BEAD · phase sase-ag.5 · Plan-file provenance reconciliation`). Rejected on
  two grounds. House style keeps every SASE CONTEXT lane header to a _summary_ — `PLAN` shows `epic · 3 phases`,
  `ARTIFACTS` summarizes only its present fields — never a repeat of a body value. And the header is yielded outside the
  width-constrained field grid, so an unbounded title wraps back to column 0 with no hanging indent; truncating it there
  instead would break the lane's lossless-value contract, which the existing wrap tests enforce.
- **Both header and body.** Duplicates one value in a five-row lane for no gain.
- **A `◆` diamond prefix** borrowed from the `PLAN` roadmap's phase marker. Rejected: every value in this lane starts at
  the same column, and a glyph on exactly one row would offset that row's text by two cells and break the flush left
  edge that makes the grid readable.
- **Dimming `Epic Title` to create a weight hierarchy** now that a more important title exists. Rejected as unneeded
  churn: the labels already carry the distinction, and the lane is five rows tall.
- **A Rust core change.** Not needed and not appropriate. `sase-core` already validates and returns each phase's
  `title`, and it already reaches `PlanDisplayPhase`. Everything in this plan is Python presentation and the Python
  display-value boundary, on the presentation side of the core backend boundary.

## Implementation

### 1. Normalize phase titles at the display-value boundary

`src/sase/sdd/plan_display.py` — `_plan_display_phase()`

`_ValidatedPlan.title` and `.goal` are already collapsed to one display line by `_normalized_optional_text()`, but
`ValidatedPlanPhase.title` is passed through raw. A YAML folded or literal block title therefore reaches presentation
with embedded newlines, which would break a single-line grid row (and can already smear the `PLAN` roadmap's phase
line).

Collapse it at construction: `title=" ".join(phase.title.split())`. `PlanDisplayPhase.title` stays `str`. This fixes
both consumers at the correct layer and matches how the plan-level title is handled two functions away.

### 2. Carry the phase title in the summary value type

`src/sase/ace/tui/models/_agent_associated_plan_types.py` — `PhaseBeadSummary`

Add `phase_title: str | None` immediately after `id`. Deliberately **no default**: the type is frozen with `slots=True`
and has only two production construction sites, so requiring every caller to be explicit prevents a future path from
silently dropping the title. `phase_title` (not `title`) keeps the symmetry with the existing `epic_title` field.

### 3. Populate it

`src/sase/ace/tui/models/_agent_associated_plan_phase.py`

- `phase_bead_summary()` — set `phase_title=normalize_bead_text(phase.title)`. `normalize_bead_text` is already imported
  and already used for the description; it collapses whitespace and maps an empty result to `None`, so a blank title
  degrades to the `unavailable` fallback rather than rendering an empty row. The two normalization layers are
  complementary, not redundant: step 1 protects every `PlanDisplayPhase` consumer, this one enforces the lane's
  `str | None` optional-field contract.
- `unavailable_phase_bead()` — set `phase_title=None`. This is the path taken for a missing/unreadable/damaged parent
  plan, an unresolvable parent reference, and an out-of-range phase ordinal, and it is also the base value
  `phase_bead_summary()` returns before the ordinal is validated, so every honest-fallback route is covered by one edit.

### 4. Render it

`src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py`

- Add `"Phase Title"` to `_BEAD_FIELD_LABELS` (the tuple only feeds the label-width computation; the derived
  `BEAD_FIELD_LABEL_WIDTH` stays 15 because `Phase Title` ties `Description` at 11 cells).
- `_rows()` — prepend `(self._label("Phase Title"), self._phase_title_value())` ahead of `Description`.
- Add `_phase_title_value()` alongside `_title_value()`, returning
  `Text(self.summary.phase_title, style=COLOR_BEAD_PRIMARY)` when present and `Text("unavailable", style=COLOR_EMPTY)`
  otherwise.

Both `logical_text` (metadata search, copy, zoom, clan roll-up) and `__rich_console__` (responsive folding grid) build
from `_rows()`, so the new row is picked up by every consumer with no further wiring, keeps `overflow="fold"` /
`no_wrap=False` lossless wrapping with a hanging indent, and honors the 80-cell `BEAD_SECTION_MAX_WIDTH` budget.

### 5. Keep the clan BEAD lane consistent

`src/sase/ace/tui/widgets/prompt_panel/_agent_clan_disk_aggregation.py`

A clan panel rolls each member's phase bead into a `BEAD` context entry currently labelled
`<id> · <first line of description>`. Now that the per-agent lane leads with the title, prefer it for the same reason —
titles are short, stable, and align across many member rows where truncated descriptions do not:

- use `phase.phase_title` when present,
- otherwise fall back to the existing `first_meaningful_line(phase.description)` preview,
- otherwise the bare `phase.id`.

`first_meaningful_line` is already imported in that module; keep applying it to the chosen label text so the entry stays
one line.

## Tests

### `tests/ace/tui/widgets/test_agent_display_bead_section.py`

- `_bead_summary()` helper: rename the `title=` keyword to `epic_title=` (it always fed `epic_title` and the name is now
  ambiguous) and add a `phase_title=` keyword defaulting to a distinct value such as `"Responsive BEAD lane"`.
- Extend `test_bead_lane_has_exact_field_order_alignment_palette_and_no_old_row`:
  - `plain.index("▸ BEAD · phase sase-42.3") < plain.index("Phase Title:") < plain.index("Description:")`;
  - add `"Phase Title:"` to the field-line set and assert the colon-column set is still exactly `{13}` — this is the
    regression guard for "no existing row moved";
  - `assert_span_covers(header, "Responsive BEAD lane", COLOR_BEAD_PRIMARY)`;
  - keep the existing no-truncation assertions at widths 120 and 28.
- Add a case asserting the phase title and epic title are **distinct strings that both render**, so a future refactor
  cannot wire `epic_title` into both rows.
- Extend the lossless-wrap parametrization with three `Phase Title` cases mirroring the existing `Epic Title` cases: a
  long multi-word value, a single long unbroken token, and a wide-CJK value. The helper matches labels on the full
  `BEAD_FIELD_LABEL_WIDTH` slice, so `Phase Title:` and `Epic Title:` do not collide.
- Add: `phase_title=None` renders `Phase Title: unavailable` styled `COLOR_EMPTY`.
- Update `test_known_unavailable_plan_keeps_path_and_quiet_state` — `header.plain.count("unavailable")` becomes `4`.
- `test_cheap_phase_header_does_no_enrichment_or_bead_lookup` must keep passing unchanged: the modern phase path stays
  memory-and-plan-file only and performs no bead-store read.

### `tests/ace/tui/models/test_agent_phase_bead_summary.py`

- A folded multi-line phase title normalizes to one line, mirroring the existing
  `test_modern_phase_normalizes_epic_title_once`.
- A blank/whitespace-only phase title yields `phase_title is None`.
- A damaged plan, an unreadable plan, and an out-of-range phase ordinal each yield `phase_title is None` while the
  existing identity and path fallbacks are unchanged.
- The **selected** phase's title is used and a peer phase's title is never present on the summary — the epic fixture has
  more than one phase, so assert the non-selected phase's title is absent.

### Construction-site updates

Add the new field to every remaining `PhaseBeadSummary(...)` construction (six test modules and the one production
module in step 3; all sites already use keyword arguments): `tests/ace/tui/models/test_agent_associated_plan_phase.py`,
`tests/ace/tui/widgets/test_agent_context.py`, `tests/ace/tui/widgets/test_agent_display_plan_summary.py`,
`tests/ace/tui/widgets/test_agent_display_header_enrichment_async.py`. In `test_agent_context.py`, extend the existing
`"  Epic Title: Phase-local context lane\n"` assertion with the matching `"  Phase Title: ..."` row so lane ordering
stays covered there too.

### Clan aggregation

Cover the title-preferring clan `BEAD` entry label in the clan aggregation tests
(`tests/ace/tui/widgets/test_agent_clan_aggregation.py` / `test_agent_display_clan_sections.py`, whichever already
asserts the `<id> · <preview>` label): title present wins, description preview is the fallback, bare id when neither
exists.

### Visual snapshots — `tests/ace/tui/visual/test_ace_png_snapshots_agents_auto_approve.py`

Two goldens are affected and both are already phase-bead fixtures, so the expected phase titles are known:

- `test_agents_phase_bead_context_png_snapshot` — agent `sase-visual.2` under epic `sase-visual` selects phase index 1,
  the `render` phase titled **`Responsive BEAD lane`**. Add `assert_page_svg_contains(page, "Phase Title:")` and
  `assert_page_svg_contains(page, "Responsive BEAD lane")`, and switch the `wait_for_svg_contains` anchor to
  `"Phase Title:"` so the wait targets the first rendered row. **Keep** `assert "Typed phase metadata" not in svg` —
  that is peer phase index 0's title and it is now the sharpest available proof that the lane leaks no peer phase.
- `test_agents_phase_family_bead_and_plan_context_png_snapshot` — agent `sase-83.1` under epic `sase-83` selects phase
  index 0, the `snapshot` phase titled **`Provider update snapshot`**. Assert that string is present and add
  `assert "Render update awareness" not in svg_plain` for the same peer-privacy reason. Keep the existing
  `"medium" not in svg_plain` and `"small" in svg_plain` assertions.

Regenerate the two goldens `agents_phase_bead_context_120x40.png` and `agents_phase_bead_and_plan_context_120x40.png`
with `just test-visual --sase-update-visual-snapshots`, then re-run `just test-visual` **without** the flag to confirm
exact pixel equality. Before accepting, inspect the diffs under `.pytest_cache/sase-visual/` and confirm the only change
is the inserted row plus the one-line downward shift of everything below it.

The second fixture renders both lanes in a 120x40 zoom modal, so the extra line shifts later content down by one row.
Verify its existing SVG assertions (`"Phase plan"`, `"tale"`, `▸ BEAD` before `▸ PLAN`) still hold rather than assuming
they do; if the `PLAN` lane is pushed out of the visible viewport, adjust that fixture's scroll/wait rather than
weakening the assertion.

Also run the full visual suite and update **only** goldens that genuinely change. `agents_context_zoom_modal_120x40`
uses a tale-plan agent, the clan fixtures build no phase beads, and `models_panel_phase_worker_drilled_in_120x40` is a
model-alias bucket, so none of them are expected to move — if any of them do, investigate before accepting.

## Documentation

`docs/ace.md`, both places that enumerate the lane's fields:

- the SASE CONTEXT narrative (the paragraph containing "The lane shows `Description`, `Size`, `Epic Plan`, and
  `Epic Title`"), and
- the **SASE CONTEXT / BEAD** bullet in the Agents metadata reference ("Its fields are `Description`, `Size`,
  `Epic Plan`, and `Epic Title`, in that order").

Update both lists to `Phase Title`, `Description`, `Size`, `Epic Plan`, and `Epic Title`, in that order, and state that
the phase title comes from the same validated, frontmatter-ordered phase entry, is normalized to a single line, wraps
losslessly like the other values, and renders `unavailable` for a missing, unreadable, damaged, or out-of-range entry.
Leave the existing peer-privacy and no-bead-store-read guarantees intact — this change strengthens rather than relaxes
them.

Do not hand-edit `CHANGELOG.md`; it is generated from commit messages. Land this as `feat(ace): ...`.

## Out of scope

- The legacy compact `Bead:` metadata row and the `.land`/exact-epic compatibility inference paths.
- The `PLAN` lane's phase roadmap rendering, beyond the shared title normalization in step 1.
- Any change to the `BEAD` lane header, to `Epic Title` styling, or to the fold levels and their collapsed-heading
  behavior.
- Any `sase-core` / `sase_core_rs` change.

## Validation

```bash
just install          # ephemeral workspace may have stale deps
just check            # fmt, ruff, mypy, keep-sorted, symvision, toobig, validation, tests
just test-visual      # focused PNG snapshot suite, after goldens are regenerated
```

`just check` runs `just test`, which includes the PNG visual snapshots, so it is the final gate. Do not accept a visual
golden that was not re-verified with a clean (no `--sase-update-visual-snapshots`) run.
