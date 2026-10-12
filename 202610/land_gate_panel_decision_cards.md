---
tier: tale
title: Finish and land the Plan Review decision cards epic (sase-1ka)
goal: "Decision card headers style the right characters and render without letter-spaced
  chips, memory cards never clip their edits line, the two-column layout matches the
  accepted spec, the decision goldens are regenerated and inspected, and epic sase-1ka
  is closed with its plan file marked done.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1ka.land
bead: sase-1ka
create_time: 2026-10-12 11:52:04
status: wip
---

- **PARENT:**
  [202610/gate_panel_decision_cards.md](https://github.com/sase-org/sase--plans/blob/main/202610/gate_panel_decision_cards.md)
- **BEAD:**
  [sase-1ka](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ka/README.md)

# Plan: Finish and land the Plan Review decision cards epic (sase-1ka)

## Context

Epic `sase-1ka` ("Pinned decision cards for the Plan Review gate panel", plan
`plan:202610/gate_panel_decision_cards.md`) has all five phases closed (`sase-1ka.1` to
`.5`, commits `ace43b398a`, `648ef1a390`, `e21669907c`, `a93229e65d`, `c9c1df82cc`). Its
land agent verified the work and found a few defects the epic caused, all in the pure
Rich card builders in `src/sase/ace/tui/modals/plan_decision_cards.py`. They break the
epic's accepted spec, and one breaks an explicit `inbox` phase acceptance criterion.
This tale fixes them, regenerates the decision goldens, and closes the epic. Read the
epic's "Shared design spec" section in its plan file for the full design. The plan file
is at the PLAN path printed by `sase bead read sase-1ka -r "<why>"`.

The land agent has already done follow-up triage and recorded it on `sase-1ka` (see its
`LAND TRIAGE` note). Do not create any task beads for those items again. The epic
reviewer's decisions are final: `rail_cap = cap84`, `narrow_sheet = no`,
`focus_color = cyan`.

### Defects found (all reproduced on master `c9c1df82cc`)

1. **Header spans use cell offsets as character offsets.** `_header_line` builds the
   header text, then calls `Text.stylize(...)` with `cell_len(...)`-based offsets.
   Rich's `stylize` takes **character** offsets. Memory rows start with `🧠 `, where the
   brain is 1 character but 2 cells, so every span after it is off by one. On a current,
   focused, changed memory toggle, `‹` stays undimmed, `☑` turns dim, and the gold
   colour skips the value's first character. On every row, the gold "changed" span also
   runs into the trailing ` ›`, which should stay muted.
2. **The default `★` is never muted, and chips render letter-spaced in PNGs.** The
   spec's chip at default is `value ★` with **★ muted**. Today the padding and the chip
   are one unstyled run, e.g. `"<spaces>repoint ★"`. Rich's SVG export emits each
   same-style run as one `<text textLength=...>`. When a run holds leading padding plus
   `★`/`☐`/`☑`, resvg letter-spaces it. Every collapsed chip in the goldens shows this
   (`r e p o i n t ★`, `☑ y e s ★`, `p a n e ★`). The `inbox` phase was required to
   confirm "no such spacing artifact", and the inbox pending golden still has it. The
   land agent checked with the project renderer
   (`sase.ace.tui.visual_render.render_svg_to_png`): once the ` ★` is its own dim
   segment, or the chip starts its own segment, the chip renders with normal spacing.
3. **The memory card `edits` line never wraps.** In `expanded_card`'s memory branch, the
   `edits` selector line and the note-kind chip line are built with `no_wrap=True`. A
   realistic selector such as `decisions:project-create-human-only` (the spec's own
   mock), or two selectors, runs past the card width and is silently cropped by the
   `Static`. The spec says "silent clipping anywhere is a bug". The existing fixtures
   only use `tui.md`, so no test caught it.
4. **Two-column option heads dim the wrong cell.** In `_choice_option_rows` (two-column
   branch), `prefix` is right-padded to the shared `head_width`. The `★` is then dimmed
   at `cell_len(prefix) - 1`, so for any head shorter than the widest, a padding space
   gets dimmed and the `★` stays bold.
5. **The two-column `why` ignores the spec layout.** The spec mock for inner width ≥ 60
   puts `why ★` in the head column and the text in the label column with a hanging
   indent:

   ```text
   ▌ ◉ db ★    ~/db: matches the existing $DB/$DBB/$DBH variables in ~/.profile
   ▌ ○ legacy  ~/legacy: names it as the pre-chezmoi legacy tree; $DB is
   ▌           repointed at it
   ▌
   ▌   why ★   ~/.profile already calls this root $DB, and a ~/ sibling makes
   ▌           link retargets one-component swaps
   ```

   Today it renders as one full-width `why ★ …` paragraph with no hanging indent. The
   stacked layout (inner width < 60) is correct and stays as is.

6. **Dead CSS and a swallowed query error.** The `.plan-decision-row { … }` rule in
   `src/sase/ace/tui/styles.tcss` styles a class nothing uses since `sase-1ka.1`.
   `PlanDecisionRows._scroll_current_into_view` in
   `src/sase/ace/tui/modals/plan_decision_rows.py` still has a bare
   `except Exception: pass`. The cards phase said to stop swallowing query errors.

Out of scope:

- Deploying the `sase_plan` skill change from `sase-1ka.5`. A `sase skill init` deploy
  today would also publish `sase-1jx`'s unlanded `sase project create` skill text, which
  the installed CLI lacks. It is deferred to `sase-1jx`'s landing, which is recorded on
  both epics.
- Any `sase-core` change. This is presentation-only Textual code.

## Steps

### 1. Build the header from styled segments (defects 1 and 2)

In `src/sase/ace/tui/modals/plan_decision_cards.py`:

- Add a named constant `CHANGED_COLOR = "#FFD700"` next to `FOCUS_COLOR` and export it.
  Replace the literal `"#FFD700"` in `_header_line` with it.
- Split the chip into parts. Add a small helper that returns
  `(value_text, mark, changed)`, where `value_text` is `☑ yes` / `☐ no` for toggles or
  the value for choices, and `mark` is `★` or `●`. Keep `answer_chip_text` as the joined
  `f"{value_text} {mark}"` built from that helper, so its callers and tests are
  unchanged.
- Rewrite `_header_line` to append each piece with its own style instead of stylizing by
  offsets afterwards:
  - id: `bold`; `bold FOCUS_COLOR` on the current editable card when focused,
    `bold FOCUS_BAR_MUTED` when it is current but focus is elsewhere; `bold dim` when
    read-only. Ellipsize the id exactly as today.
  - padding: unstyled spaces, computed from cell lengths as today.
  - `‹ `: `dim`, only on the current editable card.
  - value: `CHANGED_COLOR` when changed, otherwise unstyled (normal foreground).
  - ` ★`: `dim` at the default. ` ●`: `CHANGED_COLOR` when changed.
  - ` ›`: `dim`, only on the current editable card. It must never be gold.
- Width math must stay cell-based (`cell_len`), and the chip must never ellipsize.
  `collapsed_card` and the inbox block (`_plan_decisions_block` in
  `src/sase/ace/tui/modals/notification_modal_gate.py`) get the fix for free because
  they share `_header_line`.

### 2. Wrap the memory `edits` and chip lines (defect 3)

In `expanded_card`'s memory branch, wrap the joined selector text with `_wrap_lines` at
`content_width - 11` (folding unbreakable runs). Put `  edits    ` before the first line
and an 11-space hanging indent before each continuation, like the `default` line below
it. Wrap the note-kind chip line the same way under the 11-space indent. Keep the
existing dim styles.

### 3. Fix the two-column heads and `why` (defects 4 and 5)

In `_choice_option_rows` (two-column branch), dim the `★` at its real position: the end
of the unpadded `head`, not the end of the padded `prefix`. Building the head from
separate segments is equally fine. Leave the selected/unselected bold and dim rules as
they are.

In `expanded_card`, when the choice card uses the two-column layout, render the `why`
footnote per the mock above: a `  why ★` head padded to the same head-column width as
the option heads, the same 4-space gap, then the `why` text wrapped to the label column
with a hanging indent under it. The whole footnote is `dim italic`. Compute the shared
head width once, covering the option heads and the `  why ★` head, so every label and
the `why` text start in the same column. Return it from `_choice_option_rows` or compute
it in a helper both call. Keep the blank line before `why`. The stacked layout (< 60)
and plain toggle `why` lines are unchanged.

### 4. Remove the dead CSS and the swallowed error (defect 6)

- Delete the `.plan-decision-row { height: auto; text-align: left; }` rule from
  `src/sase/ace/tui/styles.tcss`. Confirm with
  `grep -rn "plan-decision-row[^s]" src tests` that nothing else uses it.
- In `PlanDecisionRows._scroll_current_into_view`, catch only
  `textual.css.query.NoMatches` (the only expected failure when a slot is not mounted)
  instead of a bare `except Exception: pass`.

### 5. Tests

Add focused builder tests. Prefer a new module, e.g.
`tests/ace/tui/test_plan_decision_cards_builders.py`, so the existing files stay small.
Map plain substrings to styles from the rendered segments
(`list(text.render(Console()))`) or `Text.spans`:

- **Memory header styling.** A current, focused, changed memory toggle
  (`🧠 human_only_record`, value `True`, `changed: True`):
  - the id segment is `🧠 human_only_record` with bold plus `FOCUS_COLOR`;
  - `‹ ` is dim and not gold;
  - `☑ yes` and `●` are `CHANGED_COLOR`, and `☑` is not dim;
  - ` ›` is dim and not gold.

  Repeat at default (`changed: False`): ` ★` is dim, the value is not dim. Add a
  non-memory row too.

- **Mark segmentation.** In a collapsed card, the chip value and the ` ★` are separate
  segments, and the padding run does not contain `★`. This is the letter-spacing guard.
- **Memory `edits` wrapping.** Use a memory toggle with selectors
  `["decisions:project-create-human-only", "glossary:another-long-strand-name-for-wrapping"]`
  at widths 46, 76, and 100. Check that no rendered line exceeds the width
  (`render_lines` plus `cell_len`) and that every selector character appears once
  whitespace is removed from the joined output.
- **Two-column `★`.** At an inner width ≥ 60, where the default key is shorter than the
  widest key, the `★` on the default option's head line is dim.
- **Two-column `why`.** For the syncthing `root` row at width 100: the `why ★` head and
  every option label start in the same column, no line exceeds the width, and every word
  of `why` appears.
- Existing builder, render, sheet, and inbox tests stay green, especially:
  - `tests/ace/tui/test_plan_decision_ace_draft.py`
    (`test_card_builders_wrap_at_widths`, `test_toggle_chip_glyphs_measure_one_cell`);
  - `tests/ace/tui/test_plan_decision_ace_render.py`;
  - `tests/ace/tui/test_plan_decision_ace_sheet.py`;
  - `tests/ace/tui/test_notification_plan_gate.py`;
  - `tests/test_plan_decisions_corrects.py`.

  Update any assertion that pinned the old unstyled `★` or the old two-column `why`
  text.

Run the decision tests directly while iterating, e.g.
`.venv/bin/python -m pytest -q tests/ace/tui/test_plan_decision_cards_builders.py tests/ace/tui/test_plan_decision_ace_draft.py tests/ace/tui/test_plan_decision_ace_render.py tests/ace/tui/test_plan_decision_ace_sheet.py tests/ace/tui/test_notification_plan_gate.py tests/test_plan_decisions_corrects.py`.

### 6. Regenerate and inspect the decision goldens

Read the `tui` and `tui_screenshot` memories with `/sase_memory_read` first. Then run a
targeted update through `/sase_monitor` if it is long:

```bash
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_plan_gate.py tests/ace/tui/visual/test_ace_png_snapshots_notification_gates.py tests/ace/tui/visual/test_ace_png_snapshots_plan_toast.py -k decisions
```

Expected changes are every `plan_gate_*decisions*` golden, both
`notification_gate_plan_decisions_{pending,answered}_120x40`, and possibly
`plan_toast_tale_decisions_120x40`. Read the run report. If the status is `partial`,
read the WARNING / `skipped` list, fix the cause, and rerun. Open **every** changed PNG
and confirm:

- collapsed chips read `repoint ★`, `☑ yes ★`, `pane ★` with normal letter spacing;
- `★` is muted and a changed `●` is gold;
- the memory card header shows a dim `‹`, an undimmed `☑`, and a dim `›`;
- the wide (200×56, 262×72) and sheet renders show the two-column `why ★` hanging indent
  aligned with the option labels;
- nothing else moved unexpectedly.

Prove host independence with the targeted `--check` run twice, once with `TZ=UTC`:

```bash
just fix-tui-screenshots --check -- <same selectors>
TZ=UTC just fix-tui-screenshots --check -- <same selectors>
```

### 7. Verify

Read the `lint_and_test` memory, run `just fix` (or `just fmt`), then run
`sase tool run check`. Do **not** run `just check-full`. These known pre-existing
failures reproduce on clean master and are already tracked, so they are not this tale's
work:

- the Symvision `_load_archive_bundle` private import (DISCOVERED ISSUE on `sase-1jm`);
- `test_app_import_budget` 3517 vs 3513 (`sase-1ic`);
- the agent header `Project: tmp` tests (`sase-1k4`);
- `test_custom_search_preview_describes_recent_unscoped_filtering` (DISCOVERED ISSUE on
  `sase-1jm`).

Treat any other failure as yours.

### 8. Close out epic sase-1ka (final step)

1. Run `sase bead epic-symbols sase-1ka`. It listed none at planning time. If any entry
   appears, resolve it per the Symvision epic-whitelist policy: wire it up, privatize
   it, add a non-test pragma, or delete it. There is no still-open later bead to re-key
   to.
2. Close the epic. Never use `--force`. If the close is rejected, fix the stated cause
   and close again.

   ```bash
   sase bead close sase-1ka --note "Land verified all five phases against the plan and DECISIONS (cap84, narrow_sheet=no, cyan): cards/rail/sheet/inbox/nudge source and tests reviewed, goldens inspected. Landing tale fixed epic-caused builder defects: header spans used cell offsets as char offsets (memory rows mis-styled; gold leaked onto ›), ★ now muted and segmented (removes resvg letter-spaced chips, the inbox phase acceptance criterion), memory edits/chip lines wrap instead of clipping, two-column ★ dim + spec why hanging indent, dead .plan-decision-row CSS removed, NoMatches-only catch; decision goldens regenerated and inspected, --check clean twice (TZ=UTC). Integration: no conflicts with e4b3a85f4e/f26ac18475/0defe76db8/ee673ce10d. Follow-ups triaged in the LAND TRIAGE note (sase-1ic/sase-1k4 +1, sase-1jm corroborated, stale sase-1jx.11 symbol already removed); sase_plan skill deploy deferred to sase-1jx landing. epic-symbols: none."
   ```

   Adjust the note to match what you actually did and verified.

3. Run `just symvision` (use `sase tool run symvision` if the raw recipe is refused).
   Confirm the epic whitelist is clean. The only expected finding is the pre-existing
   `_load_archive_bundle` private import owned by `sase-1jm`.
4. Set `status: done` (it is `status: wip` today) in the frontmatter of the epic's plan
   file, `plan:202610/gate_panel_decision_cards.md`, at the PLAN path printed by
   `sase bead read sase-1ka -r "<why>"`. Change nothing else in that file.
5. `sase-1ka` has no `parent_bead`, so nothing further needs closing.
