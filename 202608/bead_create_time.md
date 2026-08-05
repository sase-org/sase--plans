---
tier: epic
title: Show bead creation time on every bead surface
goal: "Every SASE surface that displays a bead also displays when that bead was created, rendered through one shared
  presentation module so the glyph, color, wording, and timezone are identical everywhere, and so persisted or
  content-validated surfaces stay byte-stable.

  "
phases:
  - id: presentation
    title: Shared bead time presentation module
    depends_on: []
    size: medium
    description: "presentation: add src/sase/bead_time_presentation.py as the single source of bead instant and age
      rendering (glyphs, teal accent, absolute labels, compact age labels, Rich chips, ANSI CLI cells), with honest
      placeholders for unparseable values and a patchable clock indirection, plus unit tests.

      "
  - id: cli
    title: Bead CLI detail, list, search, and dependency surfaces
    depends_on:
      - presentation
    size: medium
    description: "cli: add a CREATED section to sase bead show, a created-age cell to compact list and search rows, and
      created context to dependency list/tree rows, then pin the test clock and regenerate the affected CLI golden
      files.

      "
  - id: gate
    title: Task triage gate payload, preview, and validation
    depends_on:
      - presentation
    size: medium
    description: "gate: thread the bead created_at through the TaskTriage gate payload, notification note, and Markdown
      preview using absolute-only rendering, extend the strict payload and preview-reconstruction validation to match,
      and add created_at to the chop presentation fingerprint.

      "
  - id: context_lane
    title: BEAD lane in the SASE CONTEXT agent metadata panel
    depends_on:
      - presentation
    size: medium
    description: "context_lane: add created_at to BeadSummary and both summary builders, render a trailing Created row
      in the BEAD lane for task and phase beads, register the new module with the visual-snapshot clock pin, and
      regenerate the affected PNG snapshots.

      "
  - id: ace_panel
    title: ACE Beads pane rows, detail pane, and reference completion
    depends_on:
      - presentation
    size: medium
    description: "ace_panel: replace the ambiguous single age on Beads pane rows with explicit created and updated
      cells, move the detail pane and preview Markdown onto the shared helpers, add created age to bead
      reference-completion rows, and regenerate the affected PNG snapshots.

      "
  - id: wire_pages
    title: Mobile wire, bead pages, and clan epic summary
    depends_on:
      - presentation
    size: medium
    description: "wire_pages: add created_at to the mobile helper bead summary wire, add a Created column to the bead
      pages phase and dependency tables while keeping page bytes stable, and show creation time in the clan epic summary
      header and phase lines.

      "
  - id: audit
    title: Cross-surface audit and documentation
    depends_on:
      - presentation
      - cli
      - gate
      - context_lane
      - ace_panel
      - wire_pages
    size: small
    description:
      "audit: sweep the repo for any remaining bead-rendering site that omits creation time, add a regression test that
      enumerates the covered surfaces, document the created-time contract in docs/beads.md, and run just check."
proposed_by: bbugyi200.athena.tc
create_time: 2026-08-05 16:28:24
status: wip
---

- **PROMPT:**
  [prompts/202608/bead_create_time.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_create_time.md)

# Plan: Show bead creation time on every bead surface

## Goal

A bead's creation time is the one fact that is guaranteed to exist for every bead — `created_at` is `TEXT NOT NULL` in
the store schema (`src/sase/bead/db.py:38`) and is populated on all 2,726 beads in the live `sase` store, always in
aware-UTC ISO form (`2026-04-28T01:34:17Z`). Despite that, most surfaces that render a bead never show it, and one
surface renders it ambiguously.

After this epic, every surface that displays a bead also displays when it was created, and every one of those renderings
comes from a single module so they agree on glyph, color, wording, and timezone.

## Design

### One vocabulary, three density tiers

All bead time rendering moves behind a new `src/sase/bead_time_presentation.py`, following the established sibling
pattern of `src/sase/bead_status_presentation.py`, `src/sase/bead_type_presentation.py`, and
`src/sase/phase_size_presentation.py`: a small frozen presentation record plus pure formatting helpers that both CLI
(ANSI) and TUI (Rich) callers share.

| Glyph        | Meaning      | Notes                                                                                   |
| ------------ | ------------ | --------------------------------------------------------------------------------------- |
| `⧖` (U+29D6) | created      | Unused anywhere in `src/` today; `cell_len` is 1.                                       |
| `✎` (U+270E) | last updated | Already means "edited/dirty" in `src/sase/vcs_list/render.py:180`; reused consistently. |

Accent color is `#5FAFAF` (muted teal), already used for dim metadata in `src/sase/ace/tui/modals/zoom_panel_events.py`
and `src/sase/ace/tui/widgets/_agent_detail_panels.py`. It is deliberately _not_ one of the bead identity colors (gold
`#FFD700` ids, `#87D7FF` phase, `#D787FF` task) so creation time reads as provenance metadata and never competes with
the title.

Three density tiers, selected per surface:

- **Full** — labeled row on detail surfaces: `⧖ Created   2026-04-28 01:34:17 EDT · 3mo ago`
- **Compact** — glyph-prefixed cell on single-line rows: `⧖ 3mo`
- **Data** — raw stored ISO string on JSON/wire surfaces, never reformatted.

### The live-vs-persisted rule

This is the correctness constraint that shapes the whole epic:

> A relative age may appear only on surfaces that are re-rendered on every read. Any surface whose bytes are persisted,
> hashed, or reconstructed for validation renders the absolute timestamp only.

Two concrete places make this mandatory, not stylistic:

1. **The TaskTriage gate preview.** `src/sase/notification_gates/kind_validation.py:571-610` validates a stored gate by
   calling `render_task_triage_preview(...)` again and asserting `preview_content == expected_preview` byte for byte. A
   relative age computed at creation time would not match one computed at validation time, so a relative age there would
   make gates fail validation as they aged.
2. **Bead pages.** `src/sase/bead_pages/` writes Markdown to disk under digest checks. A relative age would make every
   page dirty on every refresh.

So: ACE panes, the BEAD lane, and CLI terminal output get absolute + relative; gate previews, bead pages, JSON, and the
mobile wire get absolute or raw ISO only.

### Reliability rules

- All parsing goes through `sase.core.time.parse_local`, which already accepts the aware-UTC `Z` form the store uses,
  plus naive ISO and epoch. All display goes through `sase.core.time.format_local`, which renders in the configured
  timezone (pinned to `America/New_York` for tests by `tests/conftest.py:464`).
- "Now" is resolved via `sase.core.time.local_now`, never a bare `datetime.now()` — this is an explicit repo rule stated
  in the `src/sase/core/time.py` module docstring.
- **Clock indirection.** `bead_time_presentation` must not bind `local_now` as a module-level name at import time.
  Either import it lazily inside the function (as `src/sase/ace/tui/widgets/artifacts/beads_rendering.py` does today) or
  call it through the module (`from sase.core import time as core_time; core_time.local_now()`). The visual-snapshot
  clock fixture at `tests/ace/tui/visual/conftest.py:70-89` patches a list of fully qualified `local_now` targets; a
  module-level binding would silently escape that pin and make PNG snapshots drift with wall-clock time.
- Unparseable or empty values render an honest placeholder (`unknown`, dim italic) and never a fabricated time, matching
  the `unavailable` convention already used by `bead_type_chip`.
- Elapsed time is clamped at zero so a clock skew renders `now` rather than a negative age.

### Age label scale

`now`, `<N>m`, `<N>h`, `<N>d`, `<N>mo`, `<N>y` — the scale already implemented privately as `_compact_relative_age` in
`src/sase/ace/tui/widgets/artifacts/beads_rendering.py:333-357`. That private helper is deleted and its callers move to
the shared module, so the scale exists in exactly one place.

### Fixing the existing ambiguity

`_append_metadata` in `beads_rendering.py` currently renders `issue.updated_at or issue.created_at` as a bare, unlabeled
age. A reader cannot tell which one they are looking at, and for a never-updated bead the fallback silently changes
meaning. Under the new scheme both are glyph-labeled, so the row is honest and the requirement is genuinely met.

To keep dense rows quiet, **suppress the updated cell when it would render the same label as the created cell** (the
common case for a freshly filed bead). This is a pure function of the two labels, so it stays deterministic.

---

## Phase: Shared bead time presentation module

Create `src/sase/bead_time_presentation.py`. Export:

- `BEAD_CREATED_GLYPH = "⧖"`, `BEAD_UPDATED_GLYPH = "✎"`, `BEAD_TIME_ACCENT = "#5FAFAF"`.
- `BEAD_TIME_RICH_STYLE` / `BEAD_TIME_CLI_STYLE`, the latter built with `sase.ansi_style.xterm256_foreground_style`
  exactly as `bead_type_presentation.cli_style` does.
- `bead_instant_label(value) -> str` — absolute configured-tz label, `"%Y-%m-%d %H:%M:%S %Z"`, returning `"unknown"`
  when `parse_local` returns `None`. This matches the format `src/sase/bead_pages/rendering_identity.py:251` already
  produces, so bead pages keep their current bytes when they adopt the helper.
- `bead_age_label(value, *, now=None) -> str` — the compact scale above; empty string when unparseable. The optional
  `now` is for unit tests; production callers omit it and the helper resolves the clock through `sase.core.time` per the
  indirection rule.
- `bead_created_label(value, *, relative=True) -> str` — `"2026-04-28 01:34:17 EDT · 3mo ago"`, or absolute-only when
  `relative=False`. `relative=False` is what every persisted surface uses.
- `bead_created_chip(value) -> Text` and `bead_updated_chip(value) -> Text` — compact Rich cells (`⧖ 3mo`), empty `Text`
  when the value is unparseable so callers can skip the cell without a conditional.
- `bead_created_cli(value, *, use_color, relative=True) -> str` — the ANSI form for CLI rows, honoring `use_color` the
  way `bead_type_cli_cell` does.
- `suppress_duplicate_updated(created, updated) -> bool` — the dedupe predicate described above, so every row surface
  applies it identically.

Add `tests/test_bead_time_presentation.py` covering: the `Z`-suffixed store format; naive and epoch inputs; every age
bucket boundary; negative elapsed clamped to `now`; empty and malformed inputs producing the honest placeholder rather
than raising; `relative=False` omitting the age entirely; and the dedupe predicate. Follow the structure of the existing
`tests/test_bead_type_presentation.py`.

Do not change any caller in this phase — this phase only adds the module and its tests, so the five surface phases can
proceed in parallel against a stable API.

## Phase: Bead CLI detail, list, search, and dependency surfaces

`src/sase/bead/cli_detail.py` — `render_issue_detail` currently never shows creation time. Add a `CREATED` section
rendered with the existing `palette.section(...)` idiom, placed immediately before the existing `CREATED BY` block so
the two provenance facts read together:

```
CREATED
  2025-12-31 19:00:00 EST · 7mo ago
```

Preserve the documented invariant on that function: styling must stay purely additive, so stripping SGR escapes from the
output still reproduces the `DetailStyle.PLAIN` bytes exactly. Route color through `palette`, not raw escapes.

`src/sase/bead/cli_query.py` — `_render_list_compact` and `_render_search_compact` gain a trailing created cell built
with `bead_created_cli(..., use_color=use_color)`, appended after the existing parent/badge fragments so ids and titles
keep their current column positions.

`src/sase/bead/cli_dep_list.py:277` and `src/sase/bead/cli_dep_tree.py:282` already print
`added <edge.created_at> by <created_by>` — that is the _dependency edge's_ creation time, not the bead's. Leave those
lines alone and add the bead's own created cell to the bead rows in those views, so the two timestamps stay clearly
distinct.

`--json` output already carries `created_at` (`src/sase/bead/cli_detail_json.py:55`); leave it untouched.

Golden files: `tests/test_bead/golden/cli/` is byte-compared by `tests/test_bead/test_cli_golden.py:427`. The fixture
stores under `tests/test_bead/golden/stores/` carry fixed `created_at` values (`2026-01-01T00:00:00Z` and siblings), so
the absolute half is already deterministic. The relative half is not — add an autouse fixture in
`tests/test_bead/conftest.py` that pins `sase.core.time.local_now` to a fixed instant, then regenerate every affected
golden (`show.stdout`, `list.stdout`, `list_full.stdout`, `list_implicit_closed*.stdout`, and the rest of the show/list
family). Update `tests/test_bead/test_cli_show.py`, `test_cli_list.py`, and `test_cli_search.py` assertions to match.

## Phase: Task triage gate payload, preview, and validation

`src/sase/bead/task_gate.py`:

- `create_task_triage_gate` and `_build_task_triage_gate_spec` take a new `created_at: str = ""` keyword.
- Add `"created_at": created_at` to the gate `payload`.
- `render_task_triage_preview` takes `created_at` and emits `**Created:** <absolute>` in the metadata block, next to the
  existing `**Filed by:**` line. **Absolute only** — see the live-vs-persisted rule; a relative age here would break
  preview reconstruction as the gate ages.
- `task_triage_presentation_note` appends the immutable calendar date (` · ⧖ 2026-04-28`) rather than an age, for the
  same reason: `kind_validation` compares `presentation["notes"]` against a freshly computed expected note, so the value
  must not drift.

`src/sase/notification_gates/kind_validation.py`:

- Add `"created_at"` to `expected_payload_fields` (line ~343). That set is compared with `set(payload) != expected`, so
  omitting this makes every new gate fail validation.
- Validate `created_at` is a string; accept empty defensively but reject non-string.
- Pass `created_at` into **both** `render_task_triage_preview` calls in the reconstruction path (the marker template at
  line ~571 and the `expected_preview` rebuild at line ~597). Missing either one makes preview validation fail.

`src/sase/scripts/sase_chop_bead_task_triage.py`:

- Pass `created_at=issue.created_at` at the `create_task_triage_gate(...)` call site (line ~329); the full `Issue` is
  already in scope there.
- Add `"created_at": issue.created_at` to `_presentation_fingerprint` (line ~179). This is deliberate: the fingerprint
  exists to regenerate a gate whose presentation changed, and adding the field triggers exactly one regeneration wave
  across currently-pending gates. Because `created_at` is immutable per bead, there is no ongoing churn afterward.
  Omitting it instead would leave already-pending gates permanently without a creation time.

Back-compat: `validate_gate_spec` runs only from `create_gate` (`src/sase/notification_gates/service.py:65`), and the
read path (`translate_task_triage_response`) checks only `kind`. Already-persisted bundles are therefore never
re-validated against the new field set and will not break; they simply carry the old preview until the fingerprint
change regenerates them.

Update `tests/test_axe_chop_bead_task_triage.py` and the gate validation tests for the new payload field and preview
line.

## Phase: BEAD lane in the SASE CONTEXT agent metadata panel

`src/sase/ace/tui/models/_agent_associated_plan_types.py` — add `created_at: str = ""` to `BeadSummary` (defaulted, so
existing constructions in tests keep working).

`src/sase/ace/tui/models/agent_associated_plan.py` — populate it in `_task_bead_summary` (line ~447) and in the phase
summary builder alongside it, from `issue.created_at`.

`src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py`:

- Add `"Created"` to `_BEAD_FIELD_LABELS`. It is 7 characters, shorter than the existing longest label (`Phase Title`),
  so `BEAD_FIELD_LABEL_WIDTH` is unchanged and no existing row shifts.
- Render `Created` as the **final** row of `_rows()` for both the task and phase branches, unconditionally — it is the
  one field guaranteed to exist. Keeping it last preserves the lane's narrative order (title, then content, then
  attributes, then provenance).
- Value is `bead_created_label(self.summary.created_at)` in `BEAD_TIME_RICH_STYLE`.

Register `sase.bead_time_presentation` in the visual clock pin at `tests/ace/tui/visual/conftest.py:78-88` if the
implementation ends up binding `local_now` at module scope; prefer the lazy-import form, which needs no new entry
because the existing `sase.core.time.local_now` target already covers it.

Regenerate `agents_phase_bead_context_120x40.png`, `agents_phase_bead_and_plan_context_120x40.png`, and
`agents_task_bead_notes_120x40.png` with `just test-visual --sase-update-visual-snapshots`, and confirm the fixtures
that build those beads carry a fixed `created_at` so the rendered age is stable. Update
`tests/test_agent_bead_display.py` and the SASE CONTEXT lane tests.

## Phase: ACE Beads pane rows, detail pane, and reference completion

`src/sase/ace/tui/widgets/artifacts/beads_rendering.py`:

- Delete the private `_compact_relative_age` (lines 333-357) and import the shared helpers instead.
- Rewrite `_append_metadata` to emit an explicit created cell (`⧖ 3mo`) followed by an updated cell (`✎ 2d`), applying
  `suppress_duplicate_updated` so a never-updated bead shows only the created cell. Keep the existing assignee and
  project-badge fragments in their current order.
- Note that `beads_data.py:135,142` sorts on `updated_at or created_at`; that is sort behavior, not display, and stays
  as it is.

`src/sase/ace/tui/widgets/artifacts/beads_detail.py` already renders a `Created` property (line 79) and a `- Created:`
history line (line 185) via `format_local`. Move both onto `bead_created_label` so the wording and the `⧖` glyph match
every other surface, and confirm the property-grid row order still reads Created → Updated → Closed.

`src/sase/ace/tui/widgets/_artifact_ref_entity_catalogs.py` — add `created_at` to `ArtifactRefBeadCandidate` and append
`· ⧖ 3mo` to the `detail` string built in `_bead_candidate` (line ~105).

Deliberate exception to record: the completion menu's age column
(`src/sase/ace/tui/widgets/_artifact_ref_completion_menu.py:477`) is shared by bead _and_ agent rows and renders
`updated_at`. Repointing it would change agent-row semantics for no gain, so the bead's creation time rides in the
glyph-labeled `detail` string instead, where it is unambiguous.

Regenerate `artifacts_beads_populated_120x40.png`, `artifacts_beads_empty_120x40.png`, and
`notification_beads_tab_120x40.png`.

## Phase: Mobile wire, bead pages, and clan epic summary

`src/sase/integrations/_mobile_helper_beads.py` — `_bead_summary_wire` (line ~244) carries `updated_at` but not
`created_at`. Add `"created_at": issue.created_at or None` next to it, as a raw ISO string (Data tier — the client
formats it). `_bead_detail_wire` inherits it through the embedded summary, so no second change is needed there.

`src/sase/bead_pages/rendering_identity.py` — `_lifecycle_facts` (line ~192) already emits `**Created:**` via the
private `_render_instant`. Switch `_render_instant` to delegate to `bead_instant_label`, which is specified to produce
the identical `"%Y-%m-%d %H:%M:%S %Z"` output, so **page bytes must not change**. Verify with
`tests/test_bead/test_bead_page_rendering.py::test_root_and_descendant_pages_match_goldens_and_are_byte_stable` before
and after; if any golden byte moves, the helper is wrong, not the golden.

`src/sase/bead_pages/rendering_tables.py` — `render_phases` (line ~24) and `render_dependencies` list child beads
without any creation time. Add a `Created` column carrying the absolute date (`2026-04-28`, date only — the tables are
already wide and the page carries the full instant in its identity block). Update the golden pages accordingly.

`src/sase/scripts/sase_clan_summary_epic.py` — `_header_line` (line ~430) and `_phase_lines` render an epic and its
phases with no creation time. Append `⧖ <age>` to the header line, respecting the existing `_SUMMARY_WIDTH` shortening.
This summary is rendered live, so the relative form is allowed here. Update
`tests/test_bead/test_clan_summary_epic_bead_script.py`.

## Phase: Cross-surface audit and documentation

Sweep for anything the surface phases missed: grep for renderers that take an `Issue` or a bead summary and produce
user-facing text, and confirm each either shows creation time or is a deliberate, documented exception (the completion
menu age column and the dependency-edge `added <ts>` lines are the two known exceptions).

Add a regression test that enumerates the covered surfaces and asserts each one renders a creation time for a fixture
bead, so a future surface cannot silently regress the contract. Model it on the existing cross-surface presentation
tests (`tests/test_bead_status_presentation.py`, `tests/test_bead_type_presentation.py`).

Document the contract in `docs/beads.md`: the three density tiers, the `⧖`/`✎` glyphs, the live-vs-persisted rule, and
the instruction that any new bead-rendering surface must use `sase.bead_time_presentation` rather than formatting a
timestamp itself.

Run `just install` then `just check`, and `just test-visual` for the PNG suite. Confirm no golden or snapshot drifts
beyond the ones the surface phases intentionally regenerated.

## Out of scope

- No Rust core change is required. Bead reads already carry `created_at` across the `sase_core_rs` wire
  (`src/sase/core/bead_wire.py:69`), and this epic is purely presentational, which
  `sase/memory/rust_core_backend_boundary.md` keeps on the Python/TUI side of the boundary.
- The unrelated age formatters (`sase/plugins/render_common.humanize_age`, `sase/memory/review_tui` time helpers, the
  artifact/document rows in the completion menu) are left alone. Only the bead-facing formatters are consolidated.
- Bead sort order stays keyed on `updated_at or created_at`; this epic changes display, not ranking.
