---
tier: tale
title: Finish the location-first picker landing
goal: Close the snippet picker origin-loss race and complete epic sase-1cu.
size: small
proposed_by: bbugyi200.athena.sase-1cu.land
bead: sase-1cu
create_time: 2026-09-29 22:08:42
status: wip
---

- **PARENT:**
  [202609/save_location_picker.md](https://github.com/sase-org/sase--plans/blob/main/202609/save_location_picker.md)
- **BEAD:**
  [sase-1cu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cu/README.md)

# Finish the location-first picker landing

## Context

Epic `sase-1cu` has three closed phases and no parent bead. Its approved plan is
`plan:202609/save_location_picker.md`. The three epic commits are `1a4bbd3e76`,
`c44ec32abe`, and `3e03e0add9`. The feature and docs are present, and 47 focused choice,
modal, and flow tests passed at `3e03e0add9`. Intervening commits concern attachments,
ToolRuns, agents tabs, and test splits; their only shared edited file is `docs/ace.md`,
where the final picker documentation coexists with the agents-tab edit.

The remaining defect is in
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_snippet_pane.py`:
`_SnippetLocationFlow.load_and_deliver` checks the origin after the initial
`asyncio.gather` but then awaits `_build_snippet_picker_tables` in a thread. If the
origin disappears during that build, the flow can call `picker.set_choices` and leave
the picker open. Its error branches can also leave a loading or error picker after the
origin disappears. `redeliver_for_name` on the Shift+Tab path lacks an origin check
after its awaits and silently returns on a build error, leaving an endless loading
state. The approved epic plan requires the picker to close with an origin-lost warning
whenever the origin disappears during loading and to show a load error on failure.

The five `PROPOSED FOLLOW-UP:` entries from phases `.1`, `.2`, `.3` were triaged by the
land agent and recorded in epic note `LAND FOLLOW-UP TRIAGE`. The terminology defect was
corroborated on existing task `sase-1cv` and active causal epic `sase-1ck`; the
Symvision private-import pair was corroborated on `sase-1ck`. They are separate from
this epic's remaining work. `sase tool run check` run `02ae8f4b88958c1ac4073ea4cda26f7c`
stopped at the same 14 terminology fixture defects. Direct `just symvision` reported the
two existing attachment private imports. Do not run `just check-full`.

## Steps

1. Read `tui.md` and `tui_perf.md` through `sase memory read` before changing the
   event-loop flow. Keep choice construction off-thread and retain the synchronous
   picker-first push. In `_SnippetLocationFlow`, revalidate the origin after every
   awaited load/build and before applying results or errors to the picker. When the
   origin has disappeared, dismiss only the active picker, issue one origin-lost
   warning, and ignore late results. On an ordinary Shift+Tab reload failure while the
   origin still exists, show a load error and notify instead of leaving the loading
   state forever. Avoid double notifications and stale-picker callbacks.
2. Add regression tests in `tests/ace/tui/actions/test_prompt_snippet_location_flow.py`
   for origin loss while the choice builder is gated, plus the Shift+Tab reload/error
   path. Assert that no name modal opens, no orphaned picker remains, and the
   warning/error is visible as appropriate. Run the focused picker and mini/snippet flow
   tests. This is a behavior fix, so no PNG golden update is expected.
3. Finish this epic's landing in the same coder turn. Recheck the plan's acceptance
   criteria, child notes, and intervening commits. Run `sase tool run check` (the
   existing terminology failure is tracked as `sase-1cv`/`sase-1ck`; do not treat it as
   picker work). Run `sase bead epic-symbols sase-1cu`; resolve or re-key every entry if
   any appeared. Close normally with
   `sase bead close sase-1cu --note "<verified picker flows, integration, tests, follow-up outcomes, and verification limits>"`.
   Never use `--force` merely to make closure succeed. Then run `just symvision` and set
   `status: done` in the frontmatter of the linked plan file
   `plan:202609/save_location_picker.md` at the path printed by
   `sase bead read sase-1cu`. Use `sase repo open plans` before editing the sidecar. The
   epic has no `parent_bead`. Finish with the normal SASE final declaration, including
   the main and plans repositories if changed.

## Acceptance

- The snippet picker always closes when its origin disappears during either initial
  choice construction or Shift+Tab reload, with one warning and no late name step.
- A Shift+Tab reload failure gives the user a recoverable error instead of an
  indefinitely loading picker.
- Focused flow tests pass; all `sase-1cu` symbol exemptions are gone; the epic is closed
  normally and its linked plan is marked done.
