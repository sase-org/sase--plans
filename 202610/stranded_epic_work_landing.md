---
tier: tale
size: medium
title: Finish and land epic sase-1no (stranded epic work in ACE)
goal: "Make the ⚐ stranded flags bead-anchored on epic clans and wave pips, so a nested
  chain is followable from its top-level clan. Fix the blocked-detail wording bugs,
  finish the waiter tokens, help, docs, and goldens the epic plan required, and close
  epic sase-1no.

  "
proposed_by: bbugyi200.athena.sase-1no.land
proposed_with: claude/opus
bead: sase-1no
create_time: 2026-10-24 09:28:09
status: wip
---

- **PARENT:**
  [202610/stranded_epic_work.md](https://github.com/sase-org/sase--plans/blob/main/202610/stranded_epic_work.md)
- **BEAD:**
  [sase-1no](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1no/README.md)

# Plan: finish and land epic sase-1no

## Why

The sase-1no land audit checked every phase commit (5d982d2bb9, bb81495765, a7996c3b77,
c793bff54b, 0a1a8aba2f) against `plan:202610/stranded_epic_work.md` and its DECISIONS
(`flag_tasks = yes`, `doctor_report = yes`). Most of the epic is complete and correct:

- the sase-core reducer and binding (sase-core `58aca5fe`, published in 0.42.3 and
  pinned by dd0c6d8d19);
- the collector, facade, presentation vocabulary, and doctor WARNING;
- the Beads pane;
- the resolve modal.

The audit found these defects and gaps, all caused by the epic. This tale fixes them and
closes the epic.

1. **Clan rows are not bead-anchored.**
   - `agent_stranded_view` (`src/sase/ace/tui/models/agent_stranded.py`) counts `⚐N`
     only from clan member rows that happen to be loaded and are themselves `stranded`.
     It always uses the breadcrumb style.
   - The epic plan requires the following, and the "Bead-anchored: never from which rows
     happen to be loaded" invariant forbids the current approach:
     - N is the distinct stranded leaves in the epic's subtree, taken from the report.
     - The leaf style applies when a leaf is a direct phase or the epic itself;
       otherwise use the breadcrumb style.
   - Today an ancestor epic clan (the `sase-1n6` → `sase-1n6.5.1.5` → … shape) shows no
     ⚐ at all, so the amber trail from the top-level clan down to the leaf, which is the
     point of the epic, does not exist.
2. **Wave pips fold only stranded members.** The epic plan says stranded **or blocked**
   anchor rows fold to `▲` (`NEEDS_YOU`). `_wave_display_state` in
   `src/sase/ace/tui/widgets/_clan_wave_presentation.py` only receives stranded member
   keys.
3. **The `[beads]` wait lane hides the blocked sentence.**
   - In `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py` (around lines
     376–391), `else: value.append(blocked_sentence(verdict))` pairs with
     `if report is not None`. That condition is always true when a verdict was found.
   - A blocked entry therefore renders `⚐ ` plus a bold-amber age instead of
     `⚐ blocked by stranded <leaf>`.
   - No test covers the blocked lane.
4. **The blocked header callout stutters.** `_stranded_header_callout` in
   `src/sase/ace/tui/widgets/prompt_panel/_identity_header_compact.py` renders
   `⚐ blocked 1h blocked by stranded sase-visual.1 · ⏎ resolve`; see the committed
   golden `agents_stranded_blocked_detail_120x40.png`. The plan's format is
   `⚐ blocked 31h by stranded <leaf> (+N more)`.
5. **WAITING row tokens were never converted.**
   - The plan says stranded bead tokens use `⚐` (leaf style) instead of `◐`, blocked
     ones use a breadcrumb `⚐`, and multi-target counts gain `⚐N`.
   - When an agent-session target (not only a bead target) resolves to a stranded or
     blocked verdict and no token showed it, a breadcrumb `⚐` is appended.
   - Today the row appends only one trailing `⚐` and only looks at `waiting_for_beads`.
     `src/sase/ace/tui/wait_status_presentation.py` was never touched.
6. **Help is missing entries.** `⚐` rows were added only to "Agent Row Glyphs". The plan
   also requires entries in "Waiting Badges" and "Clan Waves"
   (`src/sase/ace/tui/modals/help_modal/agents_reference_sections.py`).
7. **Docs are missing the overlay.** `docs/reference/tui/agent-statuses.md` never
   describes the `STRANDED` overlay, which the plan placed next to `FINALIZING`.
8. **The nested-chain golden is missing.** The plan required two Agents goldens,
   `agents_stranded_epic_clan_120x40` and `agents_stranded_nested_chain_120x40`. Only
   one combined golden, `agents_stranded_rows_and_clan_120x40`, exists, and no golden
   shows a nested chain.
9. **The import-budget comment is inaccurate.**
   - 0a1a8aba2f raised `_MAX_MODULE_COUNT` in `tests/ace/tui/test_app_import_budget.py`
     from 3527 to 3537, with a comment claiming "no sase.\* module growth".
   - `tools/tui_import_closure --diff 62c5934eb6` (the pre-epic base, 3529) shows +7
     modules on master (3536):
     - from this epic: `sase.ace.tui.models.agent_stranded`,
       `sase.ace.tui.widgets.artifacts.beads_stranded`,
       `sase.ace.tui.widgets.artifacts.beads_stranded_refresh`,
       `sase.bead.stranded_presentation`, and `sase.core.stranded_work_facade`;
     - from the a2a38005df declaration_manifest split:
       `sase.finalizers.declaration_manifest_commit` and
       `sase.finalizers.declaration_manifest_io`.
   - The residual pre-epic overage belongs to task sase-1ic, which already carries this
     evidence. Do not work on sase-1ic here.

Follow-up triage for this epic is already finished and recorded on sase-1no; see its
`LAND TRIAGE` note. The outcomes were:

- flake task sase-1ny created;
- +1 recorded on sase-1ic;
- feature tasks sase-1nz and sase-1o0 created;
- every other proposal declined, with reasons.

Do not re-triage them.

## Before you start

- Read `tui.md`, `tui_perf.md`, `tui_screenshot.md`, `docs_writing.md`, and
  `lint_and_test.md` with `/sase_memory_read`, plus `symvision.md` for the closeout.
- Read the epic: `sase bead read sase-1no -r "<why>"`. Read its plan with
  `sase artifact read plan:202610/stranded_epic_work.md "<why>"`. Read the shared
  vocabulary in `src/sase/bead/stranded_presentation.py`. All wording and styles come
  from there; do not hard-code new strings elsewhere.
- Render paths must stay memory-only. Every input below comes from the warm report cache
  (`cached_stranded_report*`, `_verdict_index`) and the verdict fields the reducer
  already returns (`state`, `parent_id`, `blocked_by`, `blocks`). Never touch disk.

## Work

### 1. Bead-anchored clan rows (`agent_stranded.py`, `_agent_list_render_agent.py`)

Clans for epics are named after the epic bead ID: `bead work` builds
`%clan(<epic id>, …)` and members named `<epic id>.<n>` / `<epic id>.land`. For a clan
container row:

- **Epic and project.** Take the epic ID from `agent.agent_clan`, falling back to
  `presented_agent_name`. Take the project key from the row, or from the first member
  that has one; the existing code already does this for the Enter target.
- **Leaves.** If the report has a verdict for the epic ID, the clan's leaves are
  `{epic}` when its state is `stranded`, plus `verdict.blocked_by`. Union in any
  member-derived stranded leaves from the current loop, so a clan that is not a
  candidate epic keeps working.
- **Leaf style.** Use `STRANDED_LEAF_STYLE` when the epic itself is stranded, or when
  any leaf verdict's `parent_id` equals the epic ID. Otherwise use
  `STRANDED_BREADCRUMB_STYLE`.
- **View fields.** Add the style (or a `clan_leaf_direct: bool`) to `_AgentStrandedView`
  and to its `signature`, so render caching invalidates.
- **Rendering.** `format_agent_option` must render `⚐N` with that style. N is the
  distinct leaf count.
- **Wave fold.** `clan_stranded_member_keys` must include members whose verdict state is
  `stranded` **or** `blocked`. The `▲` fold then covers a delegated phase whose child
  epic is stranded.
- **Enter target.** In `src/sase/ace/tui/actions/agents/_agent_enter_stranded.py`, a
  clan row targets the epic ID when the epic verdict is stranded or blocked. The
  existing resolve view then lists the leaves from `blocked_by`. Otherwise it targets
  the first leaf.
- **Patch lane.** Check that the warmup patch lane in
  `src/sase/ace/tui/actions/agents/_loading_bead_warmup.py` still re-renders clan rows
  when a project's report changes. It keys on the project, so it should; add a test if
  it does not.

Tests, in `tests/ace/tui/models/test_agent_stranded.py` and
`tests/ace/tui/widgets/test_agent_list_stranded.py`:

- an ancestor clan whose only leaf is two levels down shows a breadcrumb `⚐1`, with no
  member rows of the leaf loaded in that clan;
- a clan with a direct stranded phase shows a leaf-style `⚐1`;
- a stranded epic bead itself is counted;
- a blocked member folds its wave pip to `▲`;
- the Enter target for an ancestor clan is the epic ID.

### 2. WAITING row tokens (`wait_status_presentation.py`, `_agent_list_render_agent_status.py`, `agent_stranded.py`)

- **View.** `_AgentStrandedView.waiting_on` should keep its bead targets. Also resolve
  each `waiting_for` agent name through `derive_agent_bead_id_from_name`
  (`src/sase/agent/bead_display.py`) against the same verdict index, and record those
  separately, for example as `waiting_on_agents`. Keep both in `signature`.
- **Formatter.** Give `format_wait_dependency_summary` an optional keyword such as
  `stranded_bead_states: Mapping[str, str] = {}`, mapping bead ID to
  `stranded`/`blocked`:
  - In the singleton-bead form, when `single_bead_id` is in the mapping, render `⚐` in
    leaf or breadcrumb style in place of the status glyph, followed by the bead ID.
  - In the count form, append a `⚐N` token after the counts. N is the number of stranded
    or blocked bead targets. Use leaf style if any target is stranded, breadcrumb
    otherwise.
  - Default behavior must be byte-identical when the mapping is empty.
- **Row.** In `append_agent_row_status`'s WAITING branch, pass the mapping. Keep the
  trailing ` ⚐` only for the case no token showed: agent-session targets that resolve to
  stranded or blocked verdicts while no bead token carried a flag. The plan calls that
  one a breadcrumb.
- **Owner rows.** Rows whose own verdict is blocked, such as a land agent, keep their
  breadcrumb as today. Emit at most one trailing `⚐`.

Tests: the singleton token swap, `⚐N` in count form, the agent-session fallback
breadcrumb, and unchanged output with no verdicts. Put them in
`tests/ace/tui/widgets/test_agent_list_stranded.py`, or next to the existing
`wait_status_presentation` tests.

### 3. Fix the `[beads]` wait lane (`_agent_wait_section.py`)

- Stranded entries render ` ⚐ stranded <age>` in leaf style, as today.
- Blocked entries render `⚐` followed by `blocked_sentence(verdict)` (for example
  `blocked by stranded sase-x.1 (+1 more)`) in breadcrumb style, with no age.
- Add a blocked-lane case to `tests/ace/tui/test_agent_stranded_detail.py`.

### 4. Fix the blocked header callout (`_identity_header_compact.py`)

- Render `⚐ blocked <age> by stranded <leaf> (+N more) · ⏎ resolve`, all in breadcrumb
  style. Derive the tail from the shared `blocked_sentence` (strip its leading
  `blocked ` word) so wording cannot drift; do not hand-write it.
- Add a blocked case beside `test_stranded_agent_header_includes_callout` in
  `tests/ace/tui/widgets/test_identity_header_compact.py`.

### 5. Help rows (`agents_reference_sections.py`)

- **Waiting Badges:** add
  `("⚐", "Waits on stranded work (bold) or a blocked bead (dim)")` and
  `("⚐N", "Stranded/blocked bead targets in a count")`, or equivalently concise wording.
- **Clan Waves:** add a row such as `("▲", "Stranded or blocked member needs you")`.
- **Agent Row Glyphs:** reword the existing `⚐N` row so it no longer says "Dim amber".
  It should say the count is bold when a leaf is a direct phase or the epic itself, and
  dim otherwise.

### 6. Docs

- **`docs/reference/tui/agent-statuses.md`:** in "Completed Statuses", add a short
  paragraph after the table, or a table row, describing the `STRANDED` overlay:
  - A Done-bucket owner row whose bead is stranded reads `STRANDED ⚐` in bold amber.
  - Other buckets keep their louder word and append `⚐`.
  - It is presentation only; buckets, counts, filters, and dismissal are unchanged, like
    `FINALIZING`.
  - Link to [Stranded beads](../work-units/bead-lifecycles.md#stranded-beads).
  - Also add one sentence pointing to it from the `FINALIZING` paragraph in "Finalizers
    on the Agents Tab".
- **`docs/reference/tui/agent-row-glyphs.md`:** update the "Stranded work" table (`⚐N`
  styling and the subtree meaning, the waiter token swap and `⚐N`, and the wave fold for
  blocked members).
- **`docs/reference/tui/agents-main-deck.md`:** fix the `[beads]` lane wording if it
  disagrees with step 3.
- Follow `docs_writing.md`, and run the docs checks that `just check` includes.

### 7. PNG goldens

In `tests/ace/tui/visual/test_ace_png_snapshots_agents_stranded.py`:

- **New golden.** Add `test_agents_stranded_nested_chain_png_snapshot`, producing
  `agents_stranded_nested_chain_120x40`. It should show a nested chain modelled on the
  plan's `sase-1n6` mockup, with time pinned through `pin_agents_visual_now` exactly as
  the existing tests do:
  - a root epic clan;
  - a delegated parent phase, whose verdict is `blocked`;
  - a child epic clan containing a stranded leaf phase and a WAITING land agent.
- **Fixture.** Build agents like the existing `_agents()`, or adapt `epic_clan_agents` /
  `wave_clan_agents` from
  `tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py`. Seed
  `agent_stranded._STRANDED_REPORT_CACHE` with a report whose verdicts carry
  `parent_id`, `blocked_by`, and `blocks` as the reducer would produce them.
- **View.** Collapse the ancestor clan or clans and expand the leaf clan.
- **Assertions.**
  - The ancestor clan shows a breadcrumb `⚐1`.
  - The leaf clan shows `⚐1` and a `▲` pip.
  - The leaf row reads `STRANDED`.
  - The land row carries a `⚐`.
- **Existing goldens.** Expect updates to:
  - `agents_stranded_rows_and_clan_120x40`: clan `⚐1` is now leaf style; waiter token;
  - `agents_stranded_blocked_detail_120x40`: callout text;
  - the Agents help goldens that render the Waiting Badges, Agent Row Glyphs, and Clan
    Waves sections, including `agents_archive_help_120x40`, which renders Agents Help;
  - any golden showing a WAITING row with a stranded target.
- **Regeneration.** Run `just fix-tui-screenshots -- <selectors>` through
  `/sase_monitor`. If snapshot-name selectors match no tests, use the pytest path
  selector, as sase-1no.6 had to. Then:
  - Inspect the retained report and **view every created or updated PNG**; generation is
    not approval.
  - Prove host independence with two targeted `just fix-tui-screenshots --check` runs,
    one with `TZ=UTC` exported, neither reporting an `unstable` node.
  - Confirm that the Archive help golden still shows the `Archive (read-only)` heading
    and the restore footer.

### 8. Correct the import-budget record (`tests/ace/tui/test_app_import_budget.py`)

- Replace the 2026-10-24 comment with an accurate one:
  - The pre-epic base 62c5934eb6 measured 3529, already over the 3527 cap (tracked by
    sase-1ic).
  - Epic sase-1no added five first-paint `sase.*` modules: name them.
  - The a2a38005df declaration_manifest split added two.
  - Master measured 3536, so the cap is 3537.
- Do not add new first-paint modules in this tale: put new code in existing modules, or
  import it lazily the way `_agent_enter_stranded` is.
- Re-measure with `.venv/bin/python tools/tui_import_closure`. If the count changed, set
  the cap to measured + 1 and say why in the comment.

### 9. Verify

- Run `sase tool run check` (`just check`). Do not run `just check-full`.
- Triage any failure with the tool-run verdict. For KNOWN items, record the evidence in
  your close note.
- Run `sase bead epic-symbols sase-1no`. It is empty today; any symbol you add must have
  a real non-test consumer.

### 10. Close out epic sase-1no (final step; do it in this same turn)

1. Run `sase bead epic-symbols sase-1no`. For every listed entry, resolve the symbol per
   the Symvision epic-whitelist policy in `symvision.md`: wire it up, privatize it, add
   a non-test pragma, or delete it.
   - All of sase-1no's phases are closed, so never re-key an entry to one of them.
   - `sase bead close` refuses while entries remain.
2. Close the epic: `sase bead close sase-1no --note "<verification>"`. The note must
   cover:
   - steps 1–2 of the land audit, summarized from the `Why` section above: all six
     phases verified against the plan and DECISIONS, and integration with a2a38005df,
     1a5e01adba, 43314db7ad, dd0c6d8d19, and dc2cc11324 checked;
   - each of the nine fixes in this tale, and the new golden;
   - the `just check` result;
   - a pointer to the existing `LAND TRIAGE` note for the follow-up outcomes.

   All six phase beads are already closed. Never use `--force` merely to make the close
   succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. This is the PLAN path
   printed by `sase bead read sase-1no` (`plan:202610/stranded_epic_work.md` in the
   plans repo); its frontmatter currently reads `status: wip`. Open the plans repo with
   `/sase_repo` (`sase repo open plans -r "<why>"`) if the path is not already in your
   checkout, and edit only that field.
5. sase-1no has no `parent_bead`, so the landing ends here.
