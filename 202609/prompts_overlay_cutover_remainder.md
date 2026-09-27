---
tier: epic
title: Finish the Prompts overlay cutover
goal:
  The Prompts overlay fails truthfully when lifecycle reads fail, passes the History
  responsiveness soak, and has clean cutover symbols and reviewed PNG coverage.
parent_bead: sase-1au
phases:
  - id: fail_closed_read
    title: Fail closed when the Prompts lifecycle snapshot cannot be read
    size: medium
    depends_on: []
    description:
      "fail_closed_read: remove the active-only fallback and cover stale bindings and
      failed lifecycle reads without hiding Trash."
  - id: retire_old_modal
    title: Retire dead prompt modal surface and repair extraction test drift
    size: medium
    depends_on: []
    description:
      "retire_old_modal: resolve cutover unused symbols under Symvision policy and make
      the residual freeze soak exercise the live History pane."
  - id: overlay_visuals
    title: Capture and inspect Prompts overlay visuals
    size: medium
    depends_on:
      - fail_closed_read
      - retire_old_modal
    description:
      "overlay_visuals: add narrow and wide Prompts overlay PNG snapshots with populated
      Stash and Trash and inspect the targeted goldens."
proposed_by: bbugyi200.athena.sase-1au.land
create_time: 2026-09-26 20:01:57
status: wip
---

- **PROMPT:**
  [prompts/202609/prompts_overlay_cutover_remainder.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/prompts_overlay_cutover_remainder.md)
- **PARENT:**
  [202609/prompt_recall_tabs_and_stash_trash.md](https://github.com/sase-org/sase--plans/blob/main/202609/prompt_recall_tabs_and_stash_trash.md)

# Finish the Prompts overlay cutover

Parent epic: `sase-1au`. Its five original phases landed the Rust stash lifecycle,
Python wire and configuration, three-pane modal, staged Trash actions, and entry-point
routing. This plan covers only defects found in the landing audit. Do not put the parent
epic close, Symvision post-close pass, or parent plan status update into a child phase;
the child epic's `parent_bead` link is the landing handoff.

## Evidence and boundaries

The approved parent plan is `plan:202609/prompt_recall_tabs_and_stash_trash.md`. At
master `899bdba64`,
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore.py` catches every
exception from `read_prompt_stash_lifecycle`, reads an active-only v1 snapshot, and
opens the new overlay with empty Trash. A missing binding or read failure can therefore
look like successful recovery with zero rows. The parent plan requires a clear stale
wheel failure and truthful visible rows on failed reads.

The original History modal extraction moved `load_prompt_record_page` to
`src/sase/ace/tui/modals/history_pane.py`, but
`tests/ace/tui/test_residual_freeze_soak.py::test_lowered_threshold_soak_keeps_fixed_paths_responsive`
still monkeypatches the old module. That node now fails with `AttributeError` and must
exercise the production Prompts overlay's History tab. `just symvision` also reports the
cutover's unused `PromptHistoryModal`, `TrashCommitPreview`, `sort_trash_records`,
`stash_empty_text`, and `trash_empty_text`. Follow the `symvision.md` hierarchy: remove
dead compatibility surfaces and their stale tests, privatize file-local helpers, or
retain a public symbol only for a real non-test consumer. Do not add an epic whitelist
to suppress a dead public symbol.

The current visual prompt-stash suite captures only standalone `StashedPromptsModal`; it
has no `PromptsModal` or Trash capture. Phase 4 proposed overlay-with-Trash goldens, and
the parent plan requires narrow and wide targeted visual acceptance.

The unrelated `named-proc` versus `proc-shell` contract test failure belongs to the
active turn-rename epic `sase-1ab` and has been recorded there. Do not fix it here or
run `just check-full`.

## Phase `fail_closed_read`

Make the lifecycle snapshot read authoritative. A missing lifecycle binding, parse
failure, or store read/lock error must surface the actual error and prevent opening a
misleading overlay. Preserve the existing off-thread read and prompt-origin checks. Do
not retry through the old active-only binding. Cover the failure path at a real entry
point, including a stale wheel simulation and a store-read exception; assert that no
empty Trash pane is presented and the error identifies the missing binding or read
problem. Confirm normal Stash/History entry points and the bare-`@` fast path still work
with the current core. If any benign no-file behavior needs special handling, do it
within the lifecycle binding's contract, not as a broad exception fallback.

## Phase `retire_old_modal`

Audit every non-test consumer of the five Symvision names above. Delete the standalone
`PromptHistoryModal` if production no longer instantiates it; update lazy exports, type
stubs, old-modal tests, and references to exercise `PromptsModal`/`HistoryPane` instead.
Keep the behavioral assertions, including project-tag filter/preview and history
callbacks. Make the four file-local Trash/Stash symbols private if they have no external
consumer, updating in-file references and tests; delete truly dead code. Update the
residual freeze soak to monkeypatch the live History pane loader, enter History through
the production overlay, and preserve the original responsiveness assertions. Run the
exact soak node and relevant modal tests. Run `just symvision` and confirm these five
cutover findings are gone; unrelated master findings may remain under their own triage.

## Phase `overlay_visuals`

Read `tui_screenshot.md` and the PNG snapshot guidance in `lint_and_test.md` before
capture. Add deterministic targeted PNG cases for populated Stash and Trash in the
actual `PromptsModal`, at wide and narrow widths, including tab counts, list/preview
layout, footer, and the Trash empty or zero-limit state if it can be covered compactly.
Use `just fix-tui-screenshots` with a targeted selector, inspect the resulting PNGs and
every changed golden, then use the targeted check-only visual run. Keep the existing
dedicated stash behavior coverage where still useful. Run `just fix` and
`sase tool run check` (`just check` equivalent); classify unrelated pre-existing
failures through ToolRun evidence, without broadening this plan to the turn-rename epic.
Report the exact verification result to the child land agent.
