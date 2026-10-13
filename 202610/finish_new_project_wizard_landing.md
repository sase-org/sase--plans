---
tier: tale
title: Finish the last New Project wizard landing gaps
goal:
  Pin the committed core repair, scope Projects completion dedupe to each operation,
  prove narrow hints visibility, and close the verified epic and ready plan ancestors.
size: medium
proposed_by: bbugyi200.athena.sase-1jx.13.6.land
bead: sase-1jx.13.6
create_time: 2026-10-12 23:16:46
status: wip
---

- **PARENT:**
  [202610/new_project_wizard_final_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_wizard_final_repairs.md)
- **BEAD:**
  [sase-1jx.13.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/sase-1jx.13.6.md)

# Finish the last New Project wizard landing gaps

## Scope

Finish three bounded items found while landing `sase-1jx.13.6`, then perform its
closeout and resume the ready plan ancestors in this same coding turn. There is no
separate tale land agent. This plan deliberately has no `parent_bead` frontmatter: the
coder must close the already-existing epic directly, without adding an unfinished
descendant that would prevent its normal close.

The land audit read all four phase scopes and every note, the approved plan
`plan:202610/new_project_wizard_final_repairs.md`, the parent plan
`plan:202610/new_project_wizard_landing_repairs.md`, and the committed implementation in
sase, sase-core, and sase-github. The audit baseline is sase master `c69c31236f`. All
phase beads are closed. Source teardown, automatic fallback, sanitizing, cache age,
short-terminal cards, no-owners guidance, disabled-row copy, queued remedy refreshes and
serialized preference writes are implemented. Bare-git and gh request round-trips,
custom bare-dir recovery, explicit false options, the refusal text, version-independent
help assertions, screen-stack pane lookup, Projects help and Show-new keymaps are
implemented.

The original decisions remain final: `human_only_record=no`, `glossary_project=yes`. No
memory edits are required or authorized by this tale. Use the existing Rust backend and
Python host adapters; do not add a Python backend fallback. Do not run a live remote
create, a global reinstall, or `just check-full`.

## 1. Put the committed core fix on the CI path

Open `sase-core` through
`sase repo open sase-core -r "Verify and pin the committed New Project strict-default fix"`;
use only the printed path. Read its `AGENTS.md`.

Phase `sase-1jx.13.6.1` committed its strict-mode repair at
`2fc3d7754a3585180d0d2d7bd7ca1e034de31cdc`. It removes the default-value exemption in
`crates/sase_core/src/project_onboarding/form.rs`: explicitly requested unavailable
behavior conflicts even when it equals the default, such as template plus
`add_readme=true`. Off/empty values remain valid and omitted. Rust table tests cover
that pair, all-numbered local hiding, and inventory-only linked/sidecar identities. The
phase passed its linked-repo `sase tool run check` and left the pin for the lander.

Sase's `sase-core-revision.txt` still names `546afe324e268249240b9011ade13394869bce75`,
the immediate predecessor. Update it to the committed fix or a verified descendant,
retaining all intervening Archive core work. Follow `docs/rust_backend.md` and the
existing ratchet tooling; there is no need to change Rust code or wire schemas. Do not
invent an SHA or wait for this tale's own future commit. Verify the loaded extension
actually contains the fix. Extend the existing facade form contract test in
`tests/core/test_project_onboarding_facade.py` to assert strict template plus an
explicit default-true `add_readme` conflicts and explicit false remains valid/omitted.
This must exercise the real binding, not a stub.

## 2. Scope Projects completion dedupe to an operation

`src/sase/ace/tui/modals/projects_pane.py::_integrate_added_project` remembers
`_last_integrated_project_key` for the pane's entire lifetime and immediately returns
for the same key. Its sole initialization is in the constructor. This was added by
`bde3e7b213` (`sase-1jx.13.6.3`) to suppress the durable completion plus Ready-dismiss
double pass, but also suppresses a later independent wizard outcome.

A real mounted `AcePage` probe using `_patch_panes` from
`tests/ace/tui/test_projects_pane.py` demonstrated:

1. `NewProjectSelection("show-project", project_key="alpha")` selects alpha and says
   `Showing alpha`.
2. `pane.action_next_option()` selects beta.
3. A second independent Show-alpha outcome leaves beta selected while still saying
   `Showing alpha`.

Make the dedupe identify the originating wizard/apply operation rather than imposing a
permanent project-key ban. Preserve one integration for duplicate delivery of the same
durable proc completion and its matching Ready dismissal, including when the wizard was
closed mid-run. Every later independent `show`, `show-project`, or `added` outcome must
reload/select the requested project and set accurate status. A simpler scoped mechanism
is fine if it has the same semantics; avoid time-based heuristics. Keep the screen-stack
lookup, current-project invalidation, MRU/lifecycle/catalog updates, retained filter,
pending Show-new action and keyboard path intact.

Update the existing outcome/idempotency and real completion tests so they distinguish
duplicate delivery from separate wizard operations. Add a regression with two real
wizard selections of alpha separated by navigation to beta, and an independent filtered
outcome for the same key after an earlier completion. Verify the new result is selected
or gets the live Show-new action as appropriate, while one completion plus its own
dismissal still integrates only once.

## 3. Capture a narrow Projects golden that actually shows the hints

Projects-pane item 9 in the approved epic plan requires an 80-column capture at the
smallest height where the Projects pane's hints row paints. The existing
`test_config_center_projects_tab_narrow_png_snapshot` in
`tests/ace/tui/visual/test_ace_png_snapshots_config_center_projects.py` still captures
`size=(80, 24)` as `config_center_projects_tab_80x24.png`. The lander inspected that
PNG: it ends at the Projects list header, with no rows, detail, or `#projects-hints`.
The test docstring incorrectly says the hints/list are readable.

Probe terminal heights at width 80 using that deterministic fixture. Choose the smallest
height at which the actual hints row and useful list content are visible. Replace the
narrow test/golden with that size and an accurate name/title/docstring. Assert the hints
widget occupies visible screen space before capture, and verify the lead `n new` and the
dropped-hints `? help` appear. Keep width-aware priority fitting and the real Admin
Center shell; do not redesign its pre-existing 80x24 chrome in this tale. Resolve this
epic's missing evidence rather than filing it as a separate task.

Read `tui.md`, `tui_screenshot.md`, `tui_perf.md`, and `lint_and_test.md` through
`sase memory read`. Run the targeted `just fix-tui-screenshots -- <selector>` update,
inspect its report and changed PNGs, then targeted `--check` twice, once with `TZ=UTC`.
Only remove the old 80x24 golden with the full-run proof required by the snapshot
policy. If a full visual maintenance run would be excessive, retain the old clipped
capture as explicitly named legacy 80x24 layout evidence, add the useful narrow-hints
capture, and explain that choice. Do not mislabel the clipped frame as hints coverage.
Use `/sase_monitor` for a genuinely long capture; never leave a live command running at
turn end. A mutating capture needs a successor to inspect its report.

## Integration and verification

The lander fetched origin/master and found no commits ahead of `c69c31236f`. Non-epic
changes since this epic's creation / first phase commit were reviewed: `f4fdb3b3d3`
split create tests; `95430b58aa` retired the Artifacts Agent strip and updated
deterministic startup/help/goldens; `e243ce730f`, `a27c6252f3`, and `be3a9a5d1d`
finished Archive shelf, arrivals, and restore. The create tests already use the split
helpers, Projects keymaps coexist with Archive help, and wizard Ready launches use the
shared launch path. No additional feature integration was found. Recheck later drift
before closeout and resolve newly caused conflicts within this scope.

Run meaningful focused tests: the real core facade/onboarding suites; create intent,
execution and resume suites; `test_projects_pane.py`, `test_new_project_apply.py` and
Projects help; Source and preferences suites. Keep provider calls fake and isolated.
Then run `just fix` and file-change verification through `sase tool run check`, which
runs `just check`. Do not run `just check-full`. Unknown/new epic-caused failures must
be resolved. Known/flaky nonzero exits remain nonzero; preserve evidence and do not
claim a pass. Follow monitor escalation with its exact run id rather than launching
another verification run.

## Follow-up triage already completed by the lander

The lander used `/sase_new_task`, ci/flake/bug semantic searches, the last-week sweeps,
active-epic scopes, and retained ToolRuns `273cb3e17344009dabc48497edf90d4c` and
`4d3a221ef9dcf0222d255aa0470b4738`. Every child proposal outcome is recorded on
`sase-1jx.13.6`; do not create duplicate tasks or repeat the same evidence as new.

- `.3 #1` narrow hints proposal: declined as a separate task; it is approved epic item 9
  and is step 3 above. The old Admin Center height limitation does not require a chrome
  redesign here.
- `.2 #1`, `.3 #2/#4`, `.4 #1` project-header/workflow `fake/tmp` failures: corroborated
  existing ready `sase-1k4` with relayed independent phase evidence.
- `.2 #1`, `.3 #3`, `.4 #1` import count `3517` versus `3513`: corroborated existing
  ready `sase-1ic`; the phase measured +0 versus master.
- `.2 #1` retained SDD nested git-identity failure: corroborated existing ready
  `sase-120`, explicitly without claiming a fresh lander reproduction.
- `.2 #1`, `.3 #4`, `.4 #1` node-finder dismissal/coalescing, Archive quit grouping,
  stale deleted `revive_agent_modal.py` path-label exemption, and both
  revive-versus-archive legend assertions: corroborating `DISCOVERED ISSUE` note on
  active `sase-1jm`, extending its existing notes 11/12. These have a causal link to its
  Archive changes; no new task.

The retained checks label these untouched failures KNOWN with independent witnesses;
rapid-keystrokes is FLAKY in the earlier run and KNOWN in the later one. Source phase
also reproduced them on its clean base. They are not remaining New Project work. If new
distinct unrelated failures arise, use `/sase_new_task` and record the outcome.

## Final step: finish the landing in this coding turn

Do not wait for this tale's own SHA, push, or CI result; the host commits only after the
turn ends. Closeout in the same turn as the final code is required.

1. Re-read `sase-1jx.13.6` and its four closed phase beads, review their notes against
   the three fixes and the verification evidence, and recheck drift. Run
   `sase bead epic-symbols sase-1jx.13.6` (empty at audit time). Resolve every listed
   symbol by wiring, privatizing, a justified non-test pragma, or deleting according to
   `symvision.md`; re-key only to a still-open later bead that actually needs it.
2. Close normally with
   `sase bead close sase-1jx.13.6 --note "<verified phase scope/notes, core pin, operation-scoped outcomes, useful narrow capture, drift integration, exact checks, and every follow-up disposition above>"`.
   Run `just symvision` (available), through `sase tool run` if guarded. Open `plans`
   using `/sase_repo` and set `status: done` in the frontmatter of
   `202610/new_project_wizard_final_repairs.md` in its printed checkout. Read sidecar
   context only through `sase artifact read`; use the plan path to make the authorized
   status edit.
3. `sase-1jx.13.6` is directly parented to plan bead `sase-1jx.13`, not a phase. Re-read
   it with an audited reason, its previous landing note, all descendants and every note,
   and `plan:202610/new_project_wizard_landing_repairs.md`. The five phase beads `.1`
   through `.5` were closed and reviewed at the land audit. Its landing note triaged
   older follow-ups and delegated all confirmed defects to this epic. Recheck those
   claims against code and post-child drift. Check descendant statuses and validate the
   actual linked plan before treating it as ready. When complete, retire every
   `sase bead epic-symbols sase-1jx.13` entry, close normally with a concrete
   verification note, run `just symvision`, and mark
   `202610/new_project_wizard_landing_repairs.md` done in the opened plans checkout.
4. Repeat for directly parented plan ancestor `sase-1jx`: read its previous landing
   note, all twelve phases and nested descendants/notes, and
   `plan:202610/new_project_wizard.md`; recheck code, descendant and linked-plan
   readiness, inherited decisions and post-child drift. Its prior landing note already
   triaged the original follow-ups; inspect those recorded outcomes. If fully complete,
   resolve `sase bead epic-symbols sase-1jx`, close normally, confirm `just symvision`,
   and set `202610/new_project_wizard.md` done. At the first incomplete or ambiguous
   ancestor, leave it open, append an explicit blocker note, and report it instead of
   forcing closure.
5. Any close refusal for epic-symbol leftovers requires cleanup and a normal retry. A
   refusal for an unfinished phase requires finishing/reopening it or deliberately
   documenting a canceled/superseded outcome. Never use `--force` to advance a
   successful nested landing or merely to make close succeed. If the tale itself has an
   assigned plan bead, complete its actual scope before its normal finalizer close; do
   not create a circular parent/descendant close dependency.
6. If the original root epic closes, its prior landing note requests a human global
   reinstall, then `sase skill init --force`, then a disposable gh create smoke. Propose
   those through `/sase_gate` after preparing the exact reviewable commands; never
   perform the global reinstall or live create in this coder turn. The smoke
   repository's deletion remains a human step. Finish host finalization with
   `/sase_final` when returning normally, including every repository edited.
