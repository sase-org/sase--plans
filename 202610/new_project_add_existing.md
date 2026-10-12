---
tier: epic
title: Add an existing repository from the New Project wizard
goal: "Configure, Review, durable apply, live progress, the shared init hand-off, and
  Ready or Failed work for an existing repository, and the Projects pane init flow keeps
  its current behavior. The land agent closes phase bead sase-1jx.10 and does not close
  epic sase-1jx.

  "
phases:
  - id: models
    title: Textual-free add-existing models
    depends_on: []
    size: small
    description:
      "models: add Textual-free configure, review, and progress models and the stage
      transitions they drive."
  - id: init-flow
    title: Shared Projects init controller
    depends_on: []
    size: medium
    description:
      "init-flow: extract the Projects init check, preview, apply, and terminal flow
      into a controller the pane and the wizard share."
  - id: configure-review
    title: Configure and Review for existing repos
    depends_on:
      - models
    size: medium
    description:
      "configure-review: replace the Configure and Review placeholders with the
      existing-repo cards and the add-plan review."
  - id: apply-ready
    title: Apply, progress, Ready, and Failed
    depends_on:
      - configure-review
      - init-flow
    size: medium
    description:
      "apply-ready: submit durable project add, tail its checklist, and finish on Ready
      or Failed including the init hand-off."
proposed_by: bbugyi200.athena.sase-1jx.10
parent_bead: sase-1jx.10
create_time: 2026-10-12 04:39:44
status: wip
---

- **PROMPT:**
  [prompts/202610/new_project_add_existing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/new_project_add_existing.md)
- **PARENT:**
  [202610/new_project_wizard.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_wizard.md)

# Plan: Add an existing repository from the New Project wizard

This is the child epic for phase bead `sase-1jx.10` (`tui-add`) of
`plan:202610/new_project_wizard.md`. One coding agent cannot finish the configure cards,
the CLI-plan review, the init-flow extraction, the durable apply path, and the goldens
together. `models` and `init-flow` can run in parallel. `configure-review` waits for
`models`. `apply-ready` waits for both `configure-review` and `init-flow`.

Read `plan:202610/new_project_wizard.md` with `sase artifact read` before coding. The
binding sections are Stage 2a, Stage 3 (add mode only), Stage 4, and the `tui-add`
phase. Parent decisions are final: do not add `decisions:project-create-human-only`, and
do not edit any memory note, including `glossary:sase-project`. That glossary update
belonged to `cli-add`.

The private-primary sidecar prerequisite has landed. `explicit_sidecar_config_update`
accepts `private_primary`, and init already threads it through. Rely on that init flow.
Do not reimplement sidecar visibility.

## What is already on the tree

- `NewProjectModal` is a test-pushed shell. Source is real. Configure, Review, Progress,
  Ready, and Failed are placeholders in `src/sase/ace/tui/modals/new_project_modal.py`.
  No key opens the wizard. `sase-1jx.11` still owns that entry, so do not import
  `NewProjectModal` or `NewProjectSelection` from the Projects pane or the command
  palette.
- `NewProjectStateMachine` (`new_project_state.py`) moves a chosen source row to
  `configure`, a failed resume row to `review`, and an enabled project or a running
  resume row out of the wizard. Back to Source keeps the selected row.
- `build_plan_document(request, facts=...)` in `src/sase/project_onboarding/planning.py`
  accepts local facts and skips `gather_facts`. Its `apply_argv` ends in `--yes --json`.
  `next_argv` is `["sase", "init", "-p", KEY]` when an init step is pending.
- `parse_event_line` in `src/sase/project_onboarding/events.py` is public and has no
  non-test consumer. The Justfile allow-lists it as
  `--epic-symbol 'sase-1jx.10(parse_event_line)'`.
- `ProjectsPaneInitActionsMixin` in `projects_pane_init_actions.py` owns check,
  `InitPlanModal`, apply, and the `t` terminal valve. Producer sites
  `projects.init_check` and `projects.init_apply` name those functions.
- `_submit_durable_proc` lives on the app. `project.add` is already a registered
  operation. The CLI checks a durable request payload against argv: `mode == "add"`, the
  same `identity_key` and `ref`, and the same `requested` name, aliases, setup (`auto` /
  `on` / `off`), and `make_current`.
- `init_cwd()` is `Path.home()`. Session workers go through
  `app._submit_session_worker`, as init check does.
- `record_provider_selection` is already called when the provider tab changes.
  `record_owner_selection` and `record_visibility_selection` are still allow-listed for
  `sase-1jx.12`. This epic may call `record_owner_selection`. It must not call
  `record_visibility_selection`.

## Shared rules

- Presentation stays in the TUI. Do not reimplement planning, apply, or identity
  allocation.
- Provider and plan strings render as literal `Text` through `clean_source_text`. Never
  interpolate them into Rich markup.
- Read `tui.md` and then `tui_perf.md` before editing widgets or workers. No disk, JSON,
  subprocess, or `asyncio.to_thread` inside a `set_interval`, `set_timer`, `call_later`,
  or `call_after_refresh` callback. Slow work uses `spawn_pump_free_task` and is
  cancelled with `cancel_pump_free_tasks`. Re-read the selected row after every await.
  Debounce plan previews; do not debounce the highlight. Keystroke paths do not spawn
  subprocesses.
- Keep each new module under 700 lines. Stage widgets are their own modules. Do not grow
  `new_project_modal.py` into the stage implementation.
- CSS stays in the existing `NewProjectModal` block in `src/sase/ace/tui/styles.tcss`.
  Do not change Source markup or Source goldens. If a shared rule shifts those pixels,
  narrow the selector.
- The wizard is still not on a keymap. Tests push `NewProjectModal` directly, the way
  `tests/ace/tui/visual/test_ace_png_snapshots_new_project_source.py` does. Default
  goldens are `120x40`. The narrow Review golden is `80x24`.
- Drop a Justfile `--epic-symbol` row in the same change that gives the symbol a
  non-test consumer. The symvision command is one continued line. Put comments above it,
  never between its arguments.
- Do not create beads. Record follow-up on your own phase bead with
  `sase bead note <phase-id> 'PROPOSED FOLLOW-UP: <summary — detail>'`.
- Phase workers close only their own phase bead. They do not close `sase-1jx.10` or
  `sase-1jx`.
- Verify with `just fmt` and then `sase tool run check`. Do not run `just check-full`.
  PNG phases also run targeted `just fix-tui-screenshots` and inspect every created or
  updated golden. Use `/sase_monitor` when that capture is long.

## Textual-free add-existing models

Add three modules beside the source model. No Textual imports.

`src/sase/ace/tui/modals/new_project_configure_model.py` builds the Project card and the
SASE setup card from a plan document plus the editable draft (name, aliases, setup
override, make-current). Drive the copy from `setup.enabled`, `setup.reason`,
`target.can_push`, `target.sase_managed`, `project.state`, and whether the workflow is
bare git.

- Already managed: hide the setup checkbox. Say that sidecars connect on first launch.
- No push rights: leave setup unchecked and mark the reason amber. Name `sase init -M`
  as the human follow-up. Do not flip visibility to "fix" it.
- Bare git: omit the sidecar sentence.
- A disabled project: say that adding re-enables it.
- Otherwise the checkbox mirrors `setup.enabled` and shows the planner's reason,
  including the commit-and-push line when setup is on.

Map identity results to `unique`, `taken` (name the project that holds it), or
`invalid`. Use the planner's reason text. Do not invent a second uniqueness check. Local
validation is `build_plan_document` with supplied facts that contain records only. Tests
pass those facts in. They do not call `gather_facts` or a provider hook.

`src/sase/ace/tui/modals/new_project_review_model.py` turns one plan document into
review rows, warnings, blockers, the argv line, and `can_confirm`. Export
`operation_glyph(operation) -> tuple[str, str]` from `init_plan_modal_rendering.py` over
the existing `_OPERATION_GLYPHS` table, with the same `("~", "yellow")` default. Import
that function. Do not copy the table. Add mode omits a REMOTE row. A `blocked` status or
any blocker clears `can_confirm`. PUBLIC visibility is a flag the widget later paints in
the warning color. Also export `apply_submission(document)`:

- `argv` is the document's `apply_argv`, unchanged.
- `operation` is `project.add`.
- `request_fingerprint` is `intent_fingerprint`.
- `concurrency_keys` are `project-onboard:{identity_key}` and `checkout:{checkout_dir}`.
- `request` is the object the CLI compares: `mode`, `actor`, `target` (`ref`,
  `workflow_type`, `identity_key`), and `requested` (`name`, `aliases`, `setup`,
  `make_current`). `actor` is `human`. Do not set a bypass env var.

`src/sase/ace/tui/modals/new_project_progress_model.py` reduces NDJSON into checklist
rows and a raw-log tail. Every line goes through `parse_event_line`. `None` appends to
the raw log. Step events move a row from pending `◌`, to running, to done, to failed
(message and hint), or to skipped. There is no percentage. A `result` event records
`ok`, `partial`, `failed`, or `blocked`, plus `project_key`, `project_name`,
`next_argv`, and `message`. This import is the non-test consumer of `parse_event_line`.
Remove only the `sase-1jx.10(parse_event_line)` Justfile row.

Extend `NewProjectStateMachine` so a draft survives back and forth:

- Configure goes back to Source.
- Review goes back to Configure.
- Failed goes back to Review.
- `replan` from Failed returns to Review.
- Progress finishes on Ready or Failed.
- Source outcomes already implemented stay as they are.

Tests in `tests/ace/tui/modals/` cover the card states, plan-document to review rows
(fresh add, blocker, warning, public visibility, omitted REMOTE), `apply_submission`
keys, the NDJSON reducer including a non-JSON line, and the stage machine through Ready
and Failed with a preserved draft. No widgets, no PNG, no subprocesses.

## Shared Projects init controller

Move the sequencing in `ProjectsPaneInitActionsMixin` into
`src/sase/ace/tui/modals/projects_pane_init_flow.py`. The controller takes a small host:
session-worker submit, `push_screen`, notify, set-status, `app.suspend`, `is_mounted`,
and reload. It runs check, opens `InitPlanModal`, applies, and runs the terminal valve
with the argv, cwd, timeouts, dedup keys, exclusive scope `sase-init`, toasts, and
reload the mixin uses today.

The mixin becomes the host adapter. `i` and `I` keep their current behavior. Export one
entry the wizard can call later, `run_project_init(host, scope)`, where `scope` is
`InitScope.for_projects([key])`. Do not call it from `NewProjectModal` in this phase.

If the real submit functions move, update `projects.init_check` and
`projects.init_apply` in `_proc_producer_sites_actions.py` to the new path and function.
`tests/ace/tui/test_proc_producer_inventory.py` must stay green.
`tests/ace/tui/test_projects_pane_init_flow_*.py` and the init-plan modal tests stay
green without changed assertions, except import paths.

Do not change init argv, the `t` valve (`terminal_argv()` has no `--yes`), or sidecar
prompts. A cancelled or failed init still leaves the project registered. Only init is
retried.

## Configure and Review for existing repos

Replace the Configure and Review placeholders. Leave Progress, Ready, and Failed as
placeholders for `apply-ready`.

`new_project_configure.py` renders the Project card (name, aliases, checkout and whether
it will clone, key, accent swatch from the plan, enabled chip) and the SASE setup card.
Name and aliases are inline editors. `tab` and `shift+tab` move fields. `space` toggles
the focused checkbox. Match `ConfigEditModal`: `enter` and `ctrl+s` confirm. Confirm
advances only from the focused `Review ›` button when the draft is valid. While a text
input is focused, printable shortcuts do not fire, `enter` does not advance, and the
first `escape` blurs the input. `b`, or `escape` with no focused input, goes back to
Source.

Changing name, aliases, or a checkbox schedules a 300 ms preview. The timer callback
only records that a preview is due. A pump-free task calls `build_plan_document` with
local facts on a thread and marshals the card back. Re-read the selected source row
before applying the result. Typing in the name field spawns no subprocess and no
provider hook.

Choosing a normal Source row opens this card with the draft seeded from the row and the
local plan. Going back and forward keeps the draft.

`new_project_review.py` starts a session worker that runs
`sase project add … --check --json` via `sase_argv()`, cwd `init_cwd()`, and
`check_timeout(1)`. Accept return codes 0 and 1. Parse stdout with the review model.
Blockers render in red with their hint and the confirm control is disabled. Warnings
render amber. PUBLIC renders in the warning color. The argv line uses the plan's
`apply_argv`. Keys are `y` when confirm is allowed, and `b` or `escape` back to
Configure. `y` calls `apply_submission` and asks the modal to start apply. In this phase
the modal only stores that submission and shows the existing Progress placeholder.
`apply-ready` replaces the placeholder and performs the submit. Register the
session-worker producer site (`ui_only`, not durable) in
`_proc_producer_sites_actions.py`.

For `progress`, `ready`, and `failed`, `render_stepper` shows Source, Configure, and
Review all done. Source goldens stay on the Source stage, so they do not move.

Tests: the debounced preview applies a newer draft and ignores a stale one; blocked
review cannot confirm; `y` stores the submission the model built; back from Review
restores the draft. PNG goldens, pushed directly and patched so no real `sase` or `gh`
runs:

- Configure for an existing repo whose setup is recommended.
- Configure for an unmanaged repo without push rights, with the amber line.
- Review for a normal add.
- That same Review at `80x24`.

Run `just fix-tui-screenshots` with those selectors. Inspect every golden.

## Apply, progress, Ready, and Failed

Replace the three remaining placeholders. Progress, Ready, and Failed are widgets fed by
`new_project_progress_model.py`.

`y` on a confirmable Review calls `app._submit_durable_proc` with the stored
`apply_submission`: `operation="project.add"`, the plan argv, the request object,
`request_fingerprint`, both concurrency keys, and cwd `init_cwd()`. Register that call
as a `direct_submit_durable` producer site. A duplicate key notifies and stays on
Review. Do not add `SASE_PROJECT_CREATE_BYPASS` or any other guard bypass. If this TUI
is an agent session, the CLI's own setup rule is the product behavior.

`on_complete` is a function on the app, or a module-level function that receives the
app, not a closure owned by the modal. It reloads Projects when the pane is mounted,
including when the wizard was already dismissed. The test dismisses the modal while the
proc is still running and asserts that reload still runs. Do not import
`NewProjectModal` from the pane.

While Progress is shown, a 250 ms interval checks `log.version`. Unchanged versions
return immediately. A changed version starts a pump-free task that reads
`read_proc_log_tail` off the UI thread and applies the reducer on the UI thread.
Non-JSON lines stay in a collapsible raw log. Rows use `◌`, a running spinner with
elapsed time, `✓`, `✗` with message and hint, and dim `–` for skipped. Show the dim line
`closing won't stop setup — it continues in Procs`. `escape` dismisses the wizard and
does not cancel the proc.

When the result status is `ok` or `partial` and `next_argv` names init, call
`run_project_init` for `InitScope.for_projects([key])`. On success the checklist gains
`✓ SASE initialized` and the stage becomes Ready. If the user cancels init, Ready shows
the amber line `⚠ Not initialized — agents can't launch here yet · i initialize`, and
`i` runs the same hand-off again. A result with no init hand-off goes straight to Ready.
`failed` or `blocked` goes to Failed and keeps the checklist.

Ready, in the project's accent:

- `+` ensures the project tag catalog off-thread, dismisses the wizard, dismisses
  `ConfigCenterModal` if it is the screen underneath, and calls
  `app._start_custom_agent_from_selection` with a `SelectionItem` of type `project` for
  the new key so the prompt opens on `+tag `.
- `c` makes that key current through `set_current_project` in a thread worker, using the
  same outcome handling as `ProjectManagementActionsMixin`. Do not call
  `action_set_current_project`; it uses the pane's selected row.
- `enter` dismisses with outcome `show(key)`. Extend the modal result so this outcome is
  available. Leave the pane's handling of `show(key)` to `sase-1jx.11`. Dismissing is
  enough here.
- `escape` dismisses with no outcome.
- `i` is present only for the cancelled-init case above.

Failed:

- `r` returns to Review and runs the check again with the same draft.
- `t` runs `app.suspend()` and `subprocess.run` on `apply_argv` with `--json` removed
  and `--yes` kept, then re-plans. This is the add command, not the init valve.
- `b` returns to Review with the last plan.
- `escape` dismisses.

On a result status of `ok`, a pump-free task calls `record_provider_selection` and
`record_owner_selection` for the plan's workflow type and owner. That call is the
non-test consumer of `record_owner_selection`. Remove only the
`sase-1jx.12(record_owner_selection)` Justfile row. Leave `record_visibility_selection`
allow-listed. Do not remove the `sase-1jx.11` rows for `NewProjectModal` or
`NewProjectSelection`.

Tests: reducer integration through the widget is already covered for the pure model; add
the submission signature, the duplicate concurrency key, dismiss-during-run reload,
Ready versus Failed transitions, init-cancelled copy, and `t` argv with `--json`
removed. The existing `test_projects_pane_init_flow_*` suites stay green. PNG goldens,
with the proc and init flow patched:

- Progress mid-run, one step running and one pending.
- Failed, with a kept-repository hint.
- Ready, in the project's accent, including the not-initialized amber line as a separate
  case if it does not fit on the success golden.

Inspect every golden.

## Landing

The land agent closes phase bead `sase-1jx.10` after every child phase is closed. Before
that close, `sase bead epic-symbols sase-1jx.10` must print no rows. The close note
names the models, the init-flow suites, the goldens inspected, and
`sase tool run check`.

The land agent does not close epic `sase-1jx` or any ancestor. It triages
`PROPOSED FOLLOW-UP:` notes from the child phases into task beads through
`/sase_new_task`. It does not edit memory. It does not create a live repository.

## Out of scope

- The `n` keymap, hints row, empty-state copy, command palette, and `docs/ace.md`. Those
  are `sase-1jx.11`.
- The create-repository row, creation form, danger Review, and `project.create`. Those
  are `sase-1jx.12`.
- Bulk add, publishing a local directory, and deleting a remote repository.
- A second implementation of private sidecar creation.
- Opening the wizard from the Projects pane. Tests push it.
- `just check-full`, unless a later prompt names that command.
