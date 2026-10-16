---
tier: tale
title: Finish clan-wave fitting and close both epics
goal:
  Clan-wave JUMP fitting and shared imports pass verification, and sase-1lc.8 plus its
  parent close normally.
size: medium
proposed_by: bbugyi200.athena.sase-1lc.8.land
bead: sase-1lc.8
create_time: 2026-10-16 02:40:41
status: wip
---

- **PARENT:**
  [202610/clan_waves_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/clan_waves_landing_repairs.md)
- **BEAD:**
  [sase-1lc.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lc/sase-1lc.8.md)

# Finish clan-wave fitting and close both epics

## Context and scope

This is the last implementation turn for epic `sase-1lc.8`, whose parent is the plan
bead `sase-1lc`. Implement the two remaining defects below and finish both landings in
this same turn. A tale has no later land agent. Do not wait for this turn's commit,
push, SHA, or CI result before closing: host finalization commits after the turn ends.

Read these audited references before editing:

- `sase bead read sase-1lc.8 -r "Need the remaining landing scope and triage notes"`
- `sase artifact read plan:202610/clan_waves_landing_repairs.md "Need approved repair scope"`
- `sase artifact read plan:202610/clan_waves_in_agents_tab.md "Need the normative Design and parent scope"`
- `sase memory read tui.md tui_perf.md symvision.md lint_and_test.md -r "Need fitting, import-budget and verification rules"`

The lander reviewed every note on phases `sase-1lc.8.1`, `.8.2`, `.8.3`, their three
sase commits (`04fee05559`, `375bceb6ea`, `d5efed6535`), and linked-core commit
`4c83f2de965791ae78a2d0a6a12c823ebc76d2eb` (already pinned). At review HEAD
`d5efed6535`, the key-based Rust wire, adapter, state folding, workflow lanes, ladder
folds, shared after-phrases, wording, fleet repaint, title budget, deferred imports and
approved glossary repair are implemented. 186 focused tests pass. Import closure is
3519, +0 versus `91350ad5b6^`, below the unchanged cap. No unrelated commits landed
after this repair epic started or after its first implementation commit, and there are
no epic-symbol entries for either epic. Recheck drift since `d5efed6535` and integrate
any newly landed code that intersects these surfaces before closeout.

The epic's only memory decision was `glossary_clan_wave_strand = yes`; it is already
implemented. No memory edit is needed for this tale. Preserve the Rust-owned layout
boundary and all parent Design invariants: no render/projection I/O, no hidden members,
identity-based selection, no wave nodes, correct cache keys, unchanged one-wave clans,
and no new modules in the TUI startup import closure.

## 1. Make the shared suffix index a valid cross-file API

`src/sase/ace/tui/widgets/_clan_wave_gutter.py` imports `_rows_by_suffix` from
`src/sase/ace/tui/models/_agent_tree_clan.py`. Phase `.8.2` introduced that import;
`.8.3` note #3 proposed it as a follow-up, but it is epic work. `just symvision`
independently fails exactly on this private import.

Rename the existing helper to `rows_by_suffix`, update its in-file callers and the
gutter import/call, and update any test reference that actually names it. Keep the
prefer-non-child collision behavior, function location, and lazy gutter loading. It has
a real non-test cross-file consumer, so it needs neither a pragma nor an epic-symbol
exemption. Do not duplicate the helper or add a module.

Use the existing workflow-step same-suffix lane test in
`tests/ace/tui/widgets/test_agent_list_clan_wave_gutter.py` to check behavior. A new
test that only checks the rename is unnecessary.

## 2. Fit grouped JUMP targets using the space actually rendered

In `src/sase/ace/tui/widgets/_agent_jump_legend.py`, dividers now render as a three-cell
separator between same-row target neighbors (normal separator: two cells), but
`_collapsed_selection_for_columns` still adds `_selection_divider_count` as whole target
slots, including boundaries across a row break. This hides targets even when they fit.

Independent reproduction using helpers from
`tests/ace/tui/widgets/test_agent_jump_legend.py`:

```python
jump_map = _single_section_map(
    ["a0", "a1", "b0", "b1"],
    legend_groups={"a0": "wave 1", "a1": "wave 1",
                   "b0": "wave 2", "b1": "wave 2"},
)
```

At widths 16, 18, 20 and 24 the current collapsed renderer shows only `a0`, `a1`, and
`+2 ▶2`. All four targets actually fit as two 15-cell rows, with the group boundary at
the row break and no divider. Calling `_collapsed_cells` with all four,
`_collapsed_divider_gaps(..., columns=2)`, and
`_collapsed_grid_fits(..., columns=2, width=16)` returns true.

Add a regression for this actual rendered behavior before fixing it. Make grouped
selection and column fitting cooperate using actual column widths, same-row separator
widths and a real overflow cell. A divider must never consume a target slot or a grid
column; a boundary at a row break takes no space. Preserve the two-row limit, width
limit, unique labels, in-flight-first and always-shown selection, and accurate `+N ▶M`
overflow arithmetic. Keep the no-group path's pre-epic output and performance exactly as
already pinned. Keep grouped work bounded by visible capacity, rather than starting a
search at the entire roster count or doing expensive duplicate work per candidate.
Delete `_selection_divider_count` if no valid consumer remains after the change.

Extend fitting coverage to both row-break boundaries and separators within a row,
including a case that genuinely needs overflow. Update existing tests only where they
pin the old extra slot reservation; do not weaken the row-edge, width, selection,
overflow, or no-group equality assertions. Recheck 30/100-target no-group layout cost at
widths 80/120/200 against the `.8.3` note (about 1.3/2.0/3.7 ms for 30 and 2.7/3.5/6.1
ms for 100 on this host); report timings without brittle timing assertions.

## 3. Verify and inspect the visual change

Read the relevant TUI screenshot memory before using the visual harness. Run the focused
tests for the changed surfaces, then `just fix` and `sase tool run check` (the recorded
form of `just check`). Never run `just check-full`. The prior focused command was:

```bash
.venv/bin/python -m pytest -q \
  tests/ace/tui/models/test_agent_clan_waves.py \
  tests/ace/tui/widgets/test_agent_display_clan_waves.py \
  tests/ace/tui/widgets/test_agent_display_clan_roster.py \
  tests/ace/tui/widgets/test_clan_wave_presentation.py \
  tests/ace/tui/widgets/test_agent_list_clan_wave_gutter.py \
  tests/ace/tui/widgets/test_summary_fold_contracts.py \
  tests/ace/tui/widgets/test_agent_jump_legend.py \
  tests/ace/tui/widgets/test_agent_jump_panel_visibility.py \
  tests/ace/tui/test_agent_display_diff_grouping.py \
  tests/ace/tui/test_app_import_budget.py
```

Run `.venv/bin/python tools/tui_import_closure --diff 91350ad5b6^`; keep the 3519-module
baseline and the current cap. Run `just symvision` and require it to pass for this work.

Regenerate the affected layered JUMP goldens, including
`agents_jump_panel_layered_wave_dividers_{60,80,120}x40`, and any clan-panel golden
whose JUMP packing changes. Use targeted `just fix-tui-screenshots -- <selectors>` and
inspect the retained manifest/report plus every changed golden: no row-edge divider, no
padded divider column, accurate overflow, and targets shown whenever the actual two-row
layout fits. Use `/sase_monitor` for a long command; its continuation still owes the
closeout below. Run two targeted `--check` captures, one with `TZ=UTC`, across the
affected clan/clan-panel/tribe/JUMP coverage. Do not infer successful capture from a
`partial` update. Include all changed goldens in host finalization.

The lander completed `/sase_new_task` triage for every unrelated proposal and recorded
outcomes on `sase-1lc.8` plus a detailed `DISCOVERED ISSUE` on active docs epic
`sase-1le`. These are independently reproduced on clean `d5efed6535`, not epic work:

- `.8.1 #1`, `.8.2 #1`, `.8.3 #1`: one generated Markdown/Prettier drift defect in
  `render_glossary` and `render_design_decisions`, owned by `sase-1le.8`. Check ToolRun
  `129a12ae52c8dfb624dc0c3fa1f3fe33` exits 1 there in 17.4 s.
- `.8.2 #2`, `.8.3 #2`: the same eight mypy errors in `tools/render_cli`, introduced by
  docs phase `sase-1le.16` / commit `8f22863f32`; reproduced with
  `.venv/bin/mypy tools/render_cli`.
- `.8.3 #4`: three terminology nodes fail entirely in generated Reference pages:
  `test_macro_docs_and_memory_avoid_xprompt_terms` (glossary xprompt alias/history),
  `test_current_docs_skills_and_memory_avoid_stale_shell_phrases` (CLI gate,
  design-decisions and glossary), and
  `test_current_docs_skills_and_memory_avoid_stale_family_phrases` (glossary). This
  belongs to `sase-1le.8/.16`. Existing task `sase-1hr` has a different source
  string-allowlist root cause and its node passes; no duplicate task or +1 was made.
- `.8.3 #4`: `test_pdf_check_recipe_keeps_strict_mkdocs_export` and
  `test_rss_plugin_images_use_the_default_social_image` remain witnessed docs-epic
  failures, already recorded on `sase-1le` before this landing and reproduced again.
- `.8.3 #3` is fixed by step 1 and is explicitly retained as epic work.

No new task was needed; duplicate task creation was declined because these are causally
owned by the active docs epic, or belong to this epic. Preserve every outcome in the
close note. If known docs failures persist, record the exact check result and
independent attribution instead of changing docs or claiming the whole gate passed. Any
newly uncovered failure caused by this epic must be fixed before closing. A `just check`
pass with a check-full-only failure is test infrastructure work, not a landing blocker;
do not run check-full to look for one.

## 4. Finish this landing and its parent in this same turn

Do this after verification and visual inspection, before final declaration. No step
depends on the new code's commit SHA, push, or CI result.

1. Reread `sase-1lc.8` and all three children (`sase-1lc.8.1`, `sase-1lc.8.2`,
   `sase-1lc.8.3`) with `sase bead read ... -r "Verify repaired epic readiness"`. Review
   all notes, source and newly landed drift; confirm every descendant is closed and both
   defects above are fixed. Verify the linked repair plan is still the plan being
   completed. Run `sase bead epic-symbols sase-1lc.8`; resolve each entry by wiring,
   privatizing, a justified non-test pragma, deletion, or re-keying only to an actually
   open later consumer bead. Do not leave any entry tied to a closing bead.
2. Close normally:
   `sase bead close sase-1lc.8 --note "<source/commit/decision review, two repaired defects, integration, verification results, and every follow-up outcome above>"`.
   Do not force. If rejected for whitelist entries or unfinished phases, resolve the
   actual problem and close normally; never use `--force` for a successful nested
   landing. Run `just symvision` after close and confirm it passes.
3. Use `/sase_repo` to open `plans` with an audit reason, then set `status: done` in
   frontmatter of the repair plan `202610/clan_waves_landing_repairs.md` inside the
   printed checkout. Do not hand-edit artifact link tables. The root plan is also in
   this checkout and will need a commit declaration for its changes.
4. Read `sase bead read sase-1lc.8 -r "Need the parent link"`; its parent is the plan
   bead `sase-1lc`. Read that parent and its previous landing/triage notes, all seven
   original phases (`sase-1lc.1` through `sase-1lc.7`), every descendant and every note,
   and the linked plan `plan:202610/clan_waves_in_agents_tab.md` through audited reads.
   Recheck source/commit evidence, parent decisions, post-child drift, descendant
   readiness and linked-plan readiness. Its prior landing note enumerates twelve gaps;
   the repair epic plus this tale must have resolved them all. Its follow-ups were
   already triaged: feature tasks `sase-1lf` and `sase-1lg`, docs failures on
   `sase-1le`, resolved glossary task `sase-1lb`, and deliberately declined speculative
   features; reread the actual parent note rather than inventing new tasks.
5. If `sase-1lc` is fully complete, retire/re-key any entries from
   `sase bead epic-symbols sase-1lc`, then close normally with
   `sase bead close sase-1lc --note "<what was rechecked, resolved gaps, integration, verification and prior/current follow-up outcomes>"`.
   Run `just symvision` again and mark frontmatter `status: done` in its linked plan
   `202610/clan_waves_in_agents_tab.md`. Read its parent link and repeat for directly
   parented plan ancestors only while complete (it currently has no parent). If a parent
   is a phase, close only that phase normally and leave its epic to its lander. If any
   parent is incomplete or ambiguous, leave it open, append a specific blocker note
   there and report it. Do not force or silently skip readiness checks.
6. Submit `/sase_final` as the last action, with a commit declaration for every changed
   repo (sase and the opened plans checkout, if changed). Choose bead closure from
   completed work, never from whether the host has created a commit yet.
