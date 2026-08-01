---
tier: tale
title: Render the TRIBE panel description as a labeled block below the header fields
goal:
  The ACE Agents-tab TRIBE header renders the tribe description as an explicitly labeled `Description:` row placed
  beneath the header's field stack, wrapped with a hanging indent at a fixed 80-cell measure, and present on every tribe
  panel — showing the actionable `ace.tribes.<name>.description` key when no description is set.
create_time: 2026-07-31 08:47:34
status: done
---

- **PROMPT:** [prompts/202607/tribe_description_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/tribe_description_panel.md)

# Plan: Render the TRIBE panel description as a labeled block below the header fields

## Context

The ACE Agents tab renders a whole-panel `TRIBE` metadata document when a tribe panel is selected. The document is built
by `build_tribe_detail_text()` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`; the header portion is
`_append_header()` (currently lines 158-206 of that file).

The tribe description was added recently (commit `ba611aa48`, "feat(ace-tribes): require a description for configured
agent tribes"). It is sourced from `ace.tribes.<name>.description`, sanitized and length-bounded by
`_sanitize_description()` in `src/sase/ace/tui/models/tribe_display.py` (`MAX_TRIBE_DESCRIPTION_CHARS = 160`, newlines
collapsed to single spaces), and carried onto the panel snapshot by `_panel_description()` in
`src/sase/ace/tui/models/agent_tribe_summary.py` as the `description` / `description_missing` pair on
`AgentTribeSummarySnapshot`.

### Current rendering

```
TRIBE
Name: ⌂ @default
  Agents with no assigned tribe. Presentation-only: never written to the tribe store.
Status: RUNNING [R3 D8]
Composition: 7 families · 11 lanes · 14 nested
Runtime: 2h07m
Fold: 1/4
```

### Objective problems with the current rendering

1. **It splits the field stack.** The description is injected between `Name:` and `Status:`, so the four scannable facts
   (`Status`, `Composition`, `Runtime`, `Fold`) are pushed down by a variable-height prose block. Vertical positions of
   the header fields depend on description length, which defeats scanning.
2. **It is unlabeled.** A bare two-space-indented italic line requires the reader to infer what it is. Every other value
   in the panel carries a `Label: ` prefix.
3. **It has no hanging indent.** The line is emitted as one `Text` run; Rich wraps it at the panel width back to column
   0, where continuation text lines up with the bold field labels and reads as a new field.
4. **The row silently disappears.** Today there are three shapes: prose (configured with a description), an amber hint
   (configured, blank description), and _nothing at all_ (no `ace.tribes` entry). An unconfigured ad-hoc tribe panel
   gives the reader no signal that a description is even a thing, and no pointer to the key that would add one.
5. **The missing-description hint uses ASCII `-` as a dash.** The panel's established separator glyph is `·`
   (`7 families · 11 lanes · 14 nested`, `• label · status · preview`).
6. **`italic #B0B0B0` is dimmer than the panel body style** (`_BODY_STYLE = "#D7D7FF"`), so authored prose currently
   reads as chrome rather than as content.

## Design

### Target rendering — description present

```
TRIBE
Name: ⌂ @default
Status: RUNNING [R3 D8]
Composition: 7 families · 11 lanes · 14 nested
Runtime: 2h07m
Fold: 1/4

Description: Agents with no assigned tribe. Presentation-only: never written
             to the tribe store.
```

### Target rendering — no description set

```
Fold: 1/4

Description: not set · add ace.tribes.default.description
```

### Design decisions and their rationale

- **Placement: after `Fold:`, separated by one blank line.** The header keeps a stable, fixed-height stack of structured
  facts; prose lives below it. `Fold: N/M` stays the last _field_ line, which preserves the shared header grammar used
  by the clan and family panels (`append_fold_header_line` is the terminal field line in each). The blank line is the
  signal that what follows is prose rather than another field.
- **Label: `Description: ` in `_FIELD_LABEL_STYLE`.** Same bold sky-blue used by `Name:`, `Status:`, `Composition:`,
  `Runtime:`, and `Fold:`. The description becomes unmistakably labeled while joining the header's existing visual
  vocabulary instead of inventing a second one. A full underlined major section (`append_major_section_divider` +
  heading) was rejected: it is far too heavy for a ≤160-character sentence and would push `TRIBE MEMBERS` down by four
  rows.
- **Hanging indent at a fixed 80-cell measure.** Continuation lines are indented by the label width (13 cells) so the
  prose forms a single left-aligned column that can never be mistaken for a field. The wrap budget is a fixed measure
  rather than the live panel width because `build_tribe_detail_text()` is a pure builder that returns a `Text` (the
  widget calls `self.update(...)` with it) and has no console width available; a fixed measure is also deterministic for
  snapshot tests and gives a comfortable typographic line length on wide terminals. 80 cells is the measure already used
  elsewhere in this panel (`REASON_LINE_CELL_LIMIT = 80`, `BEAD_SECTION_MAX_WIDTH = 80`). On a panel narrower than 80
  cells, Rich re-wraps the pre-wrapped lines at the panel width; that degrades gracefully to the status quo instead of
  clipping.
- **The row is always present.** Every tribe panel gets a `Description:` row: authored prose, or
  `not set · add ace.tribes.<name>.description`. This removes the third shape (silent omission) and makes the panel
  predictable — the reader learns one place to look. It also means the hint reaches ad-hoc tribes, which are exactly the
  tribes whose config key the user does not yet know. Consequence: the `configured` distinction is no longer needed for
  rendering, so the `description_missing` snapshot field collapses into `not snapshot.description`.
- **One hint, phrased as an invitation, with the key as the payload.** The configured-but-blank case is already an
  error-severity diagnostic enforced by `sase doctor -C config.tribes` and by the Config Center's write refusal; the
  panel does not need to duplicate that alarm. Its job is to surface the exact key at the point of use. So the copy is
  neutral (`not set`, recessive gray) and the config key carries the accent color, making it the scannable, copyable
  payload in both cases.
- **Contrast bump for authored prose.** `italic #B0B0B0` → `italic #C6C6C6`: comfortably readable as content while
  staying visibly recessive relative to `_BODY_STYLE` (`#D7D7FF`) and to the bold field values.

### Styles

| Element                      | Constant                        | Style            | Notes                                         |
| ---------------------------- | ------------------------------- | ---------------- | --------------------------------------------- |
| `Description: ` label        | `_FIELD_LABEL_STYLE` (existing) | `bold #87D7FF`   | Identical to the other header field labels    |
| Authored prose               | `_DESCRIPTION_STYLE`            | `italic #C6C6C6` | Was `italic #B0B0B0`                          |
| `not set` / `add ` copy      | `_DESCRIPTION_MISSING_STYLE`    | `italic #8A8A8A` | Was `italic #D7AF87`; now recessive           |
| `·` separator                | (inline)                        | `dim`            | Matches the panel's separator vocabulary      |
| `ace.tribes.<n>.description` | `_DESCRIPTION_CONFIG_KEY_STYLE` | `bold #D7AF87`   | The actionable payload keeps the amber accent |

### Non-goals

- Do **not** reword the shipped descriptions in `src/sase/default_config.yml`. (The `default` entry's second sentence,
  "Presentation-only: never written to the tribe store.", is arguably implementation-facing, but rewriting shipped copy
  is a separate judgement call for the project owner.)
- Do **not** change `MAX_TRIBE_DESCRIPTION_CHARS`, the sanitizer, the JSON schema, or the `sase doctor` check.
- Do **not** add descriptions to the clan, family, or agent panels, or to agent-list panel titles.
- Do **not** make the description fold-aware. It is capped at 160 characters (at most three wrapped lines), so gating it
  behind fold levels would add a state dimension for no readability gain and would make the row's shape less
  predictable.

## Implementation

### Step 1 — Promote the cell-accurate wrapper to a shared prompt-panel helper

`src/sase/ace/tui/widgets/prompt_panel/_agent_context_common.py` already contains the exact wrapping algorithm needed
(breaks on whitespace, never on hyphens, hard-splits a token wider than the budget so progress is always made), but as
module-private functions. Importing a private name across modules is a symvision private-misuse failure, and copying 20
lines into the tribe module is worse. Promote it instead.

In `src/sase/ace/tui/widgets/prompt_panel/_helpers.py`:

- Add `PROMPT_PANEL_LINE_CELL_LIMIT = 80` with a docstring/comment noting it is the shared prose measure for
  prompt-panel lanes.
- Move `_split_token_by_cells` (keep it module-private here) and `_wrap_reason_by_cells` from
  `_agent_context_common.py`, renaming the latter to the public, domain-neutral
  `wrap_text_by_cells(text: str, width: int) -> list[str]`. Keep the existing docstrings, updating the wording from
  "reason text" to generic prose.
- `_helpers.py` imports `from rich.cells import cell_len` for these.

In `src/sase/ace/tui/widgets/prompt_panel/_agent_context_common.py`:

- Import `PROMPT_PANEL_LINE_CELL_LIMIT` and `wrap_text_by_cells` from `._helpers`.
- Keep the existing public name by defining `REASON_LINE_CELL_LIMIT = PROMPT_PANEL_LINE_CELL_LIMIT` — do not delete or
  rename it; `tests/ace/tui/widgets/test_agent_memory_reads.py` imports it (lines 14, 217, 246, 268).
- Replace the `_wrap_reason_by_cells(...)` call in `append_context_reason()` with `wrap_text_by_cells(...)`.
- Delete the two moved function definitions.

Watch for an import cycle: `_helpers.py` currently imports from `...models.agent` and `._section_navigation` only, so
`_agent_context_common.py` importing `._helpers` is safe. Verify nothing in `_helpers.py` imports
`_agent_context_common`.

### Step 2 — Render the description block in the tribe header

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`:

1. Delete the description branch currently inside `_append_header()` (the
   `if snapshot.description: ... elif snapshot.description_missing: ...` block that follows the `Name:` line).
2. Update the style constants per the Styles table above and add `_DESCRIPTION_CONFIG_KEY_STYLE = "bold #D7AF87"`.
3. Add module constants:

   ```python
   _DESCRIPTION_LABEL = "Description: "
   _DESCRIPTION_INDENT = " " * cell_len(_DESCRIPTION_LABEL)
   ```

   (import `cell_len` from `rich.cells`), and derive the wrap budget as
   `max(1, PROMPT_PANEL_LINE_CELL_LIMIT - cell_len(_DESCRIPTION_LABEL))`.

4. Add `_append_description(text: Text, snapshot: AgentTribeSummarySnapshot) -> None`:
   - Append a blank line (`text.append("\n")`), then `_DESCRIPTION_LABEL` in `_FIELD_LABEL_STYLE`.
   - **Description present:** wrap `snapshot.description` with `wrap_text_by_cells(...)` at the budget; append the first
     line in `_DESCRIPTION_STYLE` followed by `"\n"`, then each remaining line prefixed with `_DESCRIPTION_INDENT` in
     the same style. `_sanitize_description()` guarantees the input is single-line and ≤160 characters, so this is at
     most three rendered lines. Use `text.append(value, style=...)` throughout — never Rich markup — so bracketed
     descriptions stay literal.
   - **Description absent:** append `"not set"` in `_DESCRIPTION_MISSING_STYLE`, `" · "` in `"dim"`, `"add "` in
     `_DESCRIPTION_MISSING_STYLE`, then `f"ace.tribes.{tribe_config_key(snapshot.panel_key)}.description"` in
     `_DESCRIPTION_CONFIG_KEY_STYLE`, then `"\n"`. The config key is a single whitespace-free token: if
     `cell_len("not set · add ") + cell_len(key)` exceeds the budget, emit the key on its own continuation line prefixed
     with `_DESCRIPTION_INDENT` (hard-splitting via `wrap_text_by_cells` only if the key alone exceeds the budget) so
     the key stays intact and copyable. In practice this never triggers — the branch exists so an absurdly long tribe
     name degrades cleanly rather than blowing the measure.
5. Call `_append_description(text, snapshot)` at the end of `_append_header()`, after `append_fold_header_line(...)`.
   This keeps the description inside the `cheap=True` early-return path (config lookups are `lru_cache`-backed and do no
   filesystem work, so the cheap path stays cheap).

### Step 3 — Simplify the snapshot's description carrier

In `src/sase/ace/tui/models/agent_tribe_summary.py`:

- Remove the `description_missing: bool = False` field from `AgentTribeSummarySnapshot` and its assignment in
  `build_agent_tribe_summary_snapshot()`. With the row always rendered, `not snapshot.description` is the only predicate
  the renderer needs, and `configured` no longer affects presentation.
- Simplify `_panel_description(panel_key)` to return just `tribe_display_for(panel_key).description` (a `str`), or
  inline it at the single call site if that reads better; update the call site accordingly.

`_TribeDisplay.configured` in `src/sase/ace/tui/models/tribe_display.py` is set in exactly one place (`configured=True`
when building a display from a config entry) and read in exactly one place — the `_panel_description()` line this step
deletes. Once that read is gone the field is dead, so remove it, remove the `configured=True` assignment, and check
`tests/ace/tui/models/test_tribe_display.py` for any assertion that constructs or reads it. Re-verify with a grep before
deleting rather than trusting this note.

### Step 4 — Update the unit tests

In `tests/ace/tui/widgets/test_agent_display_tribe.py`:

- Rename/rewrite `test_tribe_description_line_renders_under_name` as
  `test_tribe_description_renders_below_the_header_fields`: assert the `"Description: "` label appears _after_ the
  `"Fold: "` line and _before_ the first major-section divider, that it is preceded by a blank line, and that the prose
  span carries `italic #C6C6C6`.
- Add `test_tribe_long_description_wraps_with_a_hanging_indent`: configure a 160-character description, and assert every
  line of the rendered header is ≤ `PROMPT_PANEL_LINE_CELL_LIMIT` cells and that each continuation line of the block
  starts with exactly `cell_len("Description: ")` spaces.
- Rewrite `test_tribe_missing_description_hint_names_the_config_key` for the new copy
  (`Description: not set · add ace.tribes.epic.description`) and assert the config-key span carries `bold #D7AF87` while
  the `not set` span carries `italic #8A8A8A`.
- Add `test_tribe_description_row_renders_for_unconfigured_tribes`: with an `ace.tribes` config that has no entry for
  the panel's tribe, the `Description:` row is still present with the hint. This is the regression guard for the
  behavior change in Step 3.
- Update `test_tribe_missing_description_hint_maps_none_panel_to_default` for the new copy (still
  `ace.tribes.default.description`).
- Keep `test_tribe_description_with_markup_characters_renders_literally` as-is (it only checks `detail.plain`).
- Add `test_tribe_description_renders_in_cheap_mode`: `build_tribe_detail_text(snapshot, cheap=True)` still contains the
  `Description:` row.

In `tests/ace/tui/models/test_agent_tribe_summary.py`: update the tests added by `ba611aa48` that assert
`description_missing` so they assert on `description` alone.

### Step 5 — Update the visual snapshots

`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_display_config_png_snapshot`
asserts the epic description as one contiguous substring of `prompt.content.plain`. Wrapping now splits it across two
lines ("Epic phase-worker clans from sase bead work, one member per phase" / "of an approved plan."), so that assertion
must be rewritten to compare whitespace-normalized text — e.g.
`assert "<full description>" in " ".join(prompt.content.plain.split())`. Also add an assertion that `"Description: "`
appears after `"Fold: "` in that panel's plain text, so the placement is covered end to end.

Then regenerate the affected PNG goldens:

```bash
just install
just test-visual                              # observe which goldens fail
just test-visual -- --sase-update-visual-snapshots
just test-visual                              # must pass clean afterwards
```

Every snapshot that renders a selected tribe panel is expected to change (the set touched by `ba611aa48` is a good
starting list: `agents_tribe_panel_display_config_120x40`, `agents_tribe_panel_isolation_armed_120x40`,
`agents_tribe_panel_level_1..4_120x40`, `agents_tribe_panel_selected_expanded_120x40`, `agents_collapsed_panel_120x40`,
`agents_collapsed_panel_jump_hints_120x40`, `agents_selected_panel_clan_collapse_120x40`,
`agents_sole_selected_panel_120x40`) — but trust the failure list from the run, not this list. Inspect the diffs under
`.pytest_cache/sase-visual/` before accepting them, and confirm the new header reads as the mock-up above: a contiguous
field stack, a blank line, then one labeled description block.

### Step 6 — Update the docs

- `docs/ace.md` (the paragraph currently at ~line 1051, beginning "A selected tribe panel's `TRIBE` header shows the
  tribe's configured `description`..."): rewrite for the new placement, the `Description:` label, the hanging-indent
  wrap at 80 cells, and the fact that _every_ tribe panel now shows the row — with the
  `not set · add ace.tribes.<name>.description` hint standing in when none is configured, including for ad-hoc tribes
  with no `ace.tribes` entry. Delete the "unconfigured ad-hoc tribes ... show neither line" clause, which the change
  makes false.
- `docs/configuration.md` (the `description` row of the `ace.tribes` field table at ~line 632): update "Shown in the
  Agents-tab metadata panel when that tribe's panel is selected" to name the labeled `Description:` row beneath the
  header fields. The surrounding paragraphs about the doctor check and the Config Center write refusal stay accurate and
  need no edit.

### Step 7 — Verify

```bash
just install     # workspace venvs are ephemeral; required before any other just target
just check
```

Fix everything `just check` reports, including symvision. Expected symvision-relevant points: `wrap_text_by_cells` and
`PROMPT_PANEL_LINE_CELL_LIMIT` are public and consumed by two modules plus tests (fine); the new `_DESCRIPTION_*`
constants stay module-private; and any field left unused by Step 3 (`description_missing`, possibly
`_TribeDisplay.configured`) must be removed rather than left dangling.

## Acceptance criteria

1. Selecting any tribe panel in the ACE Agents tab shows the header field stack (`Name`, `Status`, `Composition`,
   `Runtime`, `Fold`) with no prose interleaved, followed by a blank line and a single `Description: ` row.
2. The header fields sit at the same vertical offsets regardless of description length or presence.
3. A description longer than the measure wraps with continuation lines aligned under the value column, and no rendered
   header line exceeds 80 cells on a panel at least 80 cells wide.
4. A tribe with no configured description shows `Description: not set · add ace.tribes.<name>.description`, with the
   config key visually accented — including for tribes with no `ace.tribes` entry at all.
5. A description containing Rich markup characters renders literally.
6. `just check` and `just test-visual` both pass, with regenerated goldens reviewed against the mock-up above.
