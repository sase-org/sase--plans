---
tier: epic
title: Finish New Project wizard repairs before sase-1jx.13 lands
goal: Fix the epic-caused defects that the sase-1jx.13 land audit still found after
  its five phases closed, so that wizard creates work for every provider, the Projects
  pane integrates every outcome in the real app, the Source stage tears down cleanly
  and meets its spec, and Master Gate has no New Project failures. After this child
  epic lands, its land agent resumes the sase-1jx.13 landing, which then resumes the
  sase-1jx landing.
parent_bead: sase-1jx.13
phases:
- id: provider-gaps
  title: Close the strict-mode and provider test gaps in sase-core and sase-github
  depends_on: []
  size: small
  description: 'provider-gaps: in sase-core, make an explicit request for an excluded
    field''s behavior a strict-mode conflict even when it equals the field default;
    in sase-github, add the missing add_readme argv tests.'
- id: create-path
  title: Make wizard creates runnable for every provider
  depends_on: []
  size: medium
  description: 'create-path: give the TUI create submission a parse_create_ref-valid
    ref, hide visibility for forms without it, remove dead checkout_dir paths, keep
    custom bare_dir creates resumable, fix the agent refusal wording, and make the
    project help tests pass on Python 3.12.'
- id: projects-pane
  title: Integrate wizard outcomes with the real Projects pane
  depends_on: []
  size: medium
  description: 'projects-pane: find the Projects pane through the screen stack, stop
    the double filter pass, fix show-project status copy, make ? help and Show new
    project keyboard-usable, fix the stale create docstring, restore lost tests, make
    the Failed t test hermetic, and capture a narrow hints golden that shows the hints.'
- id: source-stage
  title: Finish Source stage teardown, layout, and spec states
  depends_on:
  - projects-pane
  size: medium
  description: 'source-stage: stop cancelled owner refreshes from touching an unmounted
    modal, keep automatic provider fallback out of preferences, restore account sanitizing,
    show real cache age, collapse every detail card below 30 rows, make the no-owners
    card reachable, fix ref-less disabled rows, keep remedy refreshes, serialize preference
    writes, and tighten the Source tests.'
proposed_by: bbugyi200.athena.sase-1jx.13.land
create_time: 2026-10-12 19:35:31
status: done
bead_id: sase-1jx.13.6
---

- **PROMPT:** [prompts/202610/new_project_wizard_final_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/new_project_wizard_final_repairs.md)
- **PARENT:** [202610/new_project_wizard_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_wizard_landing_repairs.md)
- **BEAD:** [sase-1jx.13.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/sase-1jx.13.6.md)

# Finish New Project wizard repairs before sase-1jx.13 lands

## Scope and sizing

Child epic `sase-1jx.13` (plan `plan:202610/new_project_wizard_landing_repairs.md`)
closed all five phases. Its land agent audited every phase against the code on master
`9fa479245c` and against the accepted design `plan:202610/new_project_wizard.md`. Most
items are done. The defects below remain, and every one is caused by the New Project
wizard epics (`sase-1jx` or `sase-1jx.13`). Either a planned item is incomplete, or a
repair introduced a regression, or the item breaks Master Gate.

Out-of-epic follow-ups are already triaged and recorded on `sase-1jx.13`. Do not file
task beads for them again.

The work spans three repositories and splits into four phases:

- `provider-gaps` runs in **sase-core** and **sase-github**. It is small.
- `create-path` runs in **sase** project onboarding, CLI, and the TUI create and review
  models.
- `projects-pane` runs in **sase** TUI: the Projects pane and wizard apply integration.
- `source-stage` runs in **sase** TUI: the wizard Source stage. It depends on
  `projects-pane` so that it can reuse that phase's hint-truncation helper, and so the
  two TUI phases do not regenerate goldens at the same time.

Open linked repositories only with `sase repo open <name> -r "<why>"`, and work in the
path it prints. Run checks there with `sase tool run check`.

### File ownership

Parallel phases must stay inside their own files so the host can merge them cleanly.

- `create-path` owns:
  - `src/sase/project_onboarding/`;
  - `src/sase/main/project_handler_*.py`;
  - the bare-git workspace plugin;
  - `new_project_review_model.py`;
  - `new_project_create_model.py`, `new_project_create_widget.py`,
    `new_project_create_rendering.py`, and `new_project_create_flow.py`;
  - their tests, plus `tests/main/test_parser_command_help.py`.

  It must not touch `preferences.py`, which `source-stage` owns.

- `projects-pane` owns:
  - `projects_pane.py`, `project_list_controller.py`, `project_management_rendering.py`,
    `new_project_apply.py`, and the `new_project_create.py` facade docstring;
  - Admin Center files;
  - `default_config.yml` and keymap docs;
  - their tests and the `config_center_projects_*` goldens.
- `source-stage` owns:
  - `new_project_source.py`, `new_project_source_model.py`, `new_project_modal*.py`,
    `new_project_rendering.py`, `panel_tab_strip.py`, and
    `src/sase/project_onboarding/preferences.py`;
  - their tests and the `new_project_source_*` goldens.

### Inherited decisions

Every phase inherits the decisions of `sase-1jx` and `sase-1jx.13`, which are final:

- `human_only_record = no`;
- `glossary_project = yes`.

No memory note may be edited in this child epic.

Each item cites where the audit observed it on master `9fa479245c`. Workers must
re-confirm each item against current master before changing code. When an item is
already fixed, record that in the phase's closing note instead of re-implementing it.

## Phase provider-gaps (sase-core, sase-github)

1. **Strict mode silently drops an explicit request that equals the default.**
   - In `crates/sase_core/src/project_onboarding/form.rs` (around lines 98-103, plus
     `requests_excluded_behavior` around 229-243), an excluded field's supplied value is
     a conflict only when `normalized != field.default`.
   - So `sase project create gh:o/r -O template=o/t -O add_readme=true` validates in
     strict mode, and `add_readme` silently disappears. Yet `true` on an excluded bool
     requests exactly the behavior the template excludes.
   - **Fix:** in strict mode, a value the user supplied for an unavailable field is a
     `conflict` whenever it requests the excluded behavior, whether or not it equals the
     default. A bool `"true"` and any non-empty non-bool value count as requests. An off
     bool (`"false"`) or an empty value is still not a request.
   - Lenient mode is unchanged.
   - **Tests:** add table tests for:
     - template + `add_readme=true` → `conflict`;
     - template + `add_readme=false` → valid, and the field is omitted;
     - an all-numbered local set (every entry numbered) staying hidden as
       `numbered_copy`;
     - inventory-only `linked` and `sidecar` identities seeding no selectable row (the
       existing test covers only `external`).
   - Keep the PyO3 wire shapes unchanged. Run `sase tool run check` in sase-core.
   - Do **not** move sase's `sase-core-revision.txt`. This phase's sase-core commit
     exists only after the turn ends, so the land agent moves the pin.
2. **The sase-github add_readme argv paths are untested.**
   - In `tests/test_github_onboarding.py`, add argv tests (fake `subprocess.run` only)
     for:
     - a template plus `add_readme="false"`: not a conflict, no `--add-readme`;
     - no template plus `add_readme="false"`: no `--add-readme`;
     - no template with `add_readme` missing: `--add-readme` is emitted;
     - each conflicting flag (`add_readme=true`, `gitignore`, `license`, `team`) alone
       with a template: rejected before any subprocess.
   - Run `sase tool run check` in sase-github.
   - If its venv cannot resolve `sase` from PyPI (tracked by `sase-16p`), set the venv
     up from the sase source the way sase-github's Justfile documents, with
     `SASE_PYTHON_PATH` and `SASE_RUST_CORE_PATH`.

## Phase create-path (sase onboarding, CLI, TUI create)

1. **Every bare-git create from the wizard fails at apply.**
   - `new_project_review_model.py::apply_submission` (around line 322) copies the plan
     document's `target.ref` into the durable request. For bare git that ref is the bare
     `NAME`, and only the argv uses `git:NAME`.
   - The CLI parses the request's ref with `parse_create_ref` in
     `project_handler_add.py` (around lines 250-262). `parse_create_ref("tools")`
     raises, so the command exits `invalid`.
   - **Fix:** give the create request a ref that `parse_create_ref` accepts, through
     `creation.create_apply_ref(workflow_type, ref)`, or one shared helper both sides
     call.
   - **Test:** build a wizard bare-git create submission, then run it through the real
     CLI request-file path (`SASE_PROJECT_REQUEST`-style intent plus the emitted argv)
     against a temp `SASE_HOME`. Assert that the CLI accepts the request and the
     identities match. Add the same round-trip for a gh create.
2. **Bare-git creates still show visibility.**
   - `new_project_create_model.py` (around line 81) offers private/public even when the
     provider's creation form has no `visibility` field.
   - The draft defaults to `FIRST_USE_VISIBILITY`, and `apply_submission` (around
     line 339) falls back to `"private"`. So bare git shows a visibility toggle and a
     `CREATE PRIVATE/PUBLIC` heading for a value the argv silently drops.
   - **Fix:** when the form has no `visibility` field:
     - hide the toggle;
     - give the Review heading and confirm label no visibility word;
     - leave visibility out of the request.
   - Update the affected create and Review goldens, and inspect each one.
3. **Dead `checkout_dir` paths.**
   - `sase-1jx.13.3` removed `checkout_dir` from the bare-git creation form. Strict
     validation now rejects it, but two places still handle it:
     - `new_project_create_model.py` (around lines 370-372, tested at
       `test_new_project_create.py` around line 437) still applies an override;
     - the bare-git `ws_create_repo` still reads `options["checkout_dir"]`.
   - **Fix:** delete both paths and the test.
4. **A custom `bare_dir` create cannot be resumed.**
   - After a create with `-O bare_dir=/custom/x.git`, `_adopt_create_result` and
     `journal.py` rekey the journal to the custom path's identity.
   - Re-running the same command plans with the default-path identity, finds no journal,
     and stops on `remote_exists`. The TUI RESUME row reports "no longer matches"
     (`new_project_resume.py` around lines 257-269).
   - **Fix:** a re-run of the same create command, and the TUI RESUME row, must both
     find that journal and continue from the adopted ref. For example, keep the
     plan-time identity as the journal lookup key and store the adopted ref and identity
     alongside it. Do not regress `test_bare_git_create_honors_custom_bare_dir`.
   - **Test:** under a temp `SASE_HOME`:
     1. create with a custom `bare_dir`;
     2. force a later step (for example checkout) to fail once;
     3. re-run the same argv;
     4. assert that it completes without a second bare repository and records the custom
        path.
5. **The agent refusal message still offers the bypass.**
   - `human_only.py::HUMAN_ONLY_CREATE_MESSAGE` tells agents to "set
     `SASE_PROJECT_CREATE_BYPASS`". That contradicts the repaired skill text: agents
     propose the exact command through `/sase_gate`, and the bypass is for humans or an
     explicit user instruction.
   - **Fix:** reword the message to match the skill, and update the two tests that pin
     the text (`test_cli.py` around line 523, `_apply_create_remote_cases.py` around
     line 139).
6. **Project help tests fail on Python 3.12 (Master Gate red).**
   - CI runs Python 3.12, and local venvs run 3.14. Two tests assert the 3.13+ argparse
     spelling `-o, --owner`, while 3.12 prints `-o OWNER, --owner OWNER`:
     - `tests/main/test_parser_command_help.py::test_project_create_help_documents_human_only_and_options`;
     - `::test_project_suggest_help_documents_owner_refresh_and_json`.
   - **Fix:** make both tests independent of the Python version, and keep asserting that
     each short and long option is documented. Check other assertions in the file for
     the same pattern.
7. **Missing coverage.**
   - `test_planning.py::test_create_apply_argv_round_trips_github` compares only
     `argv[0:4]`. Assert the full argv, including `--visibility` when the gh form has
     it. Use a fake gh creation form so the test does not depend on the installed
     sase-github.
   - Add a test that an explicit `wiki=false` / `issues=false` reaches
     `ws_create_repo`'s options (`creation_effective`) on the CLI path, and reaches the
     wizard's `lenient_effective_options` and `apply_argv` on the TUI path.

## Phase projects-pane (sase TUI)

Read `tui.md` (and the TUI notes it links) through `sase memory read` first.

1. **The Projects pane is never found in production.**
   - `new_project_apply.py::_find_projects_pane` (around lines 70-83) calls
     `app.query(ProjectsPane)`. In Textual, `App.query` never reaches widgets inside
     pushed screens. A probe returned 0 matches with the Admin Center on top, and 0
     again with the wizard on top. A scan of `app.screen_stack` finds the pane.
   - So the durable on-complete integration (`_integrate_added_project_if_mounted`,
     `_reload_projects_if_mounted`) and Ready `c`'s
     `_invalidate_current_project_indicator` never run in the real app. The tests pass
     only because they stub the lookup.
   - **Fix:** search `app.screen_stack` top-down for a mounted `ProjectsPane`.
   - **Dedupe:** once the lookup works, a completion followed by Ready `⏎`/`esc` must
     not run the post-add sequence twice. Make the integration idempotent per key, or
     skip the dismissal pass when completion already integrated.
   - **Tests:** with the real lookup, push the Admin Center, open the wizard, close it
     mid-run, then fire the durable completion. Assert that the new project is reloaded
     and selected. Also add a Ready `c` test asserting that the pane's current-project
     indicator is invalidated.
2. **The Projects filter runs twice per keystroke.**
   - `ProjectsPane.on_input_changed` (around lines 795-806) calls
     `super().on_input_changed`. Textual also dispatches the mixin's handler
     (`project_list_controller.py` around line 331) on its own, so `_apply_filters` and
     `_refresh_options` run twice.
   - **Fix:** run the pane's pending "Show new project" check from a hook that the mixin
     calls after it refilters (or an equivalent single-dispatch design), not from a
     second handler.
   - **Test:** assert one filter pass per input change.
3. **Wrong status for an already-registered project.**
   - A Source `show-project` outcome runs `_integrate_added_project`, which sets the
     status to "Project added".
   - **Fix:** use accurate copy for show outcomes (for example `Showing <name>`), and
     keep "Project added" for `added` and Ready outcomes.
4. **The advertised `? help` is a dead key.**
   - The width-aware hints end with `? help` when hints were dropped. Inside the Admin
     Center, `?` does nothing: the modal has no `?` binding, the Projects keymaps have
     no help entry, and the app-level `?` is not a priority binding.
   - **Fix:** make `?` from the Projects pane open help that lists the Projects keys
     (the app help modal's relevant section, or an existing equivalent).
   - Update `default_config.yml` and `docs/configuration.md` if you add a keymap.
   - **Test:** press `?` from the Projects pane.
5. **"Show new project" is mouse-only.**
   - It is a Button. The Admin Center's priority `tab` binding takes focus traversal,
     and no keymap triggers `action_show_new_project`.
   - **Fix:** give it a keyboard path through the projects keymap:
     - add the `default_config.yml` entry, the `docs/configuration.md` keymap row, and a
       hint while the action is offered;
     - keep help text within 32 characters.
   - **Test:** the key works.
6. **A docstring is still stale.**
   - `src/sase/ace/tui/modals/new_project_create.py` (lines 28-30) still says the create
     stage is reachable from test harnesses only and that production affordances are
     deferred. Fix it.
7. **Lost and missing tests.**
   - Restore the shift+tab half of
     `test_projects_pane.py::test_projects_filter_yields_tab_to_admin_center`, which
     `39a081d3ea` dropped.
   - Add tests that drive wizard results through
     `ProjectsPane._handle_new_project_selection` for `show`, `show-project`, `added`
     (including the filtered case) and `jump-resume`.
8. **The Failed-stage `t` test is not hermetic (Master Gate red).**
   - `test_new_project_apply.py::test_failed_keeps_checklist_and_t_strips_json` captures
     every `subprocess.run` in `new_project_apply`. With sase-github installed (as in
     CI, and in a local venv after `tools/setup_required_plugins`), it records the
     creation-form describe's `gh repo gitignore list` instead of the terminal argv.
   - **Fix:** stub provider form discovery in that test, or assert on the terminal call
     specifically. Then confirm it passes with sase-github installed.
9. **The narrow hints golden shows no hints.**
   - `config_center_projects_tab_80x24.png` shows the Admin Center cut off below the
     list header, so it proves nothing about narrow hints.
   - **Fix:** capture the narrow golden at 80 columns and the smallest height at which
     the Projects pane renders its hints row, and replace the useless 80x24 golden.
   - If the Projects tab cannot show rows at 80x24 because of Admin Center chrome that
     predates the wizard, record that as a `PROPOSED FOLLOW-UP:` note instead of
     redesigning the Admin Center.

## Phase source-stage (sase TUI)

Read `tui.md` through `sase memory read` first. Reuse the `projects-pane` hint-priority
helper where it fits.

1. **Unmount teardown regression.**
   - In `new_project_modal_providers.py::_refresh_owner_set` (around lines 320-372),
     `except asyncio.CancelledError: break` swallows cancellation when `on_unmount`
     cancels the owner jobs. The load task then continues while `is_mounted` is still
     true. It runs `_maybe_discover_create_support` and `_sync_disabled_tabs`, and
     raises `NoMatches` for `NewProjectSource`.
   - `_update_health` after the awaits in `_load_provider` and `_refresh_all_owners` is
     also unguarded, and `_loading` is left holding the provider.
   - **Fix:** propagate the outer task's cancellation, and stop every post-await DOM
     touch once teardown starts (for example a teardown flag set in `on_unmount`).
   - **Test:** pop the wizard mid-refresh with real owner tasks (not a fake task in
     `_owner_jobs`). Assert that no task fails and nothing queries the DOM.
2. **Automatic fallback overwrites the saved provider.**
   - When a provider tab becomes disabled, `_sync_disabled_tabs` calls
     `select_provider(fallback)`, which persists `last_provider`. A temporarily missing
     `gh` therefore permanently saves `git`.
   - **Fix:** persist only user-initiated provider selections. Add a test.
3. **Account text is no longer sanitized.**
   - `health_chip_text` dropped the `clean_source_text(inputs.account)` that the old
     `_update_health` applied, so control characters from the provider reach the chip.
   - **Fix:** restore sanitizing in the pure model, with a unit test.
4. **Cache age is wrong.**
   - `_inputs_arrived_at` resets on every `_show_inputs_if_selected`. So the chip shows
     time since the modal received data, which is about 0, and it still says "cached"
     after a live revalidation.
   - **Fix:** show the cache's real age, from the timestamp the cached inputs carry. If
     the inputs do not carry one, add it through the existing Python cache layer without
     a sase-core wire change; if that is impossible, record why. Drop "cached" once
     owners revalidate live.
   - **Test:** cover the age text in the pure model.
5. **Below 30 rows the list can vanish.**
   - Only repo cards collapse. The Create card, the owner-error card and the no-owners
     card keep full height, so at 80x24 the candidate list gets 0 rows when one of them
     is highlighted. Collapse is also evaluated only on highlight or rebuild.
   - **Fix:**
     - collapse every card kind to one summary line below 30 rows;
     - re-evaluate on resize;
     - make the 80x24 Source golden use a fixture with several candidates, and capture
       one 80x24 golden with the Create row highlighted.
   - Drop the 84x28 golden if it is now redundant.
6. **The no-owners card is unreachable.**
   - It renders only when no row is highlighted. With create support, the pinned Create
     row is always highlighted, so the card never shows; the
     `new_project_source_empty_no_owners_120x40` golden shows the Create card instead.
   - **Fix:**
     - show the no-owners guidance whenever no owners are known, including when the
       pinned Create row is present, as the design's empty state specifies;
     - name the config key only for providers that have one (`github_orgs` for gh).
   - Update the golden.
7. **Disabled-project rows without a ref promise an action they cannot take.**
   - A disabled project with no matching remote or local candidate gets `ref=''`. It is
     unselectable but still reads `⏎ re-enables`.
   - **Fix:** either re-enable by project key on `⏎` (through the existing enable path),
     or show accurate copy for ref-less rows. Add a test.
8. **The post-remedy refresh is dropped.**
   - After `t` runs a remedy, `action_refresh_provider` returns early when a load is in
     progress, so the post-login refresh never happens.
   - **Fix:** queue one refresh to run after the current load. Add a test.
9. **Source hints clip the advertised remedy.**
   - At 80 columns the Source hint line is clipped, so `t …` is not visible even when
     the chip advertises it.
   - **Fix:** apply width-aware priority truncation so `⏎ choose` and an advertised `t`
     remedy survive.
10. **Preference writes.**
    - Delete the dead `_save_preferences` in `new_project_modal_shell.py`.
    - `preferences.py`'s `record_*` helpers do an unlocked read-modify-write through one
      shared `.tmp` path. A UI-thread provider write racing the apply thread's owner or
      visibility write can lose an update or fail on `replace`.
    - **Fix:** serialize those writes in-process (a module lock), use a unique temp
      file, and add a concurrency test.
11. **Weak tests.**
    - `test_remedy_falls_back_to_provider_level` never reaches the fallback, because the
      default highlight is on the owner-error row. Start it with the highlight
      elsewhere.
    - Add a `SuspendNotSupported` case beside `OSError`.
    - `test_bracketed_paste_focuses_filter` must assert focus, and must deliver the
      paste through the focused list.
    - The perf guard must also patch `subprocess.Popen`. It must assert that no provider
      hook other than the pure parse runs while typing and moving the highlight.

## Verification

- Each phase:
  - runs focused tests, then `just fmt`, then `sase tool run check` in every repository
    it changed, following `lint_and_test.md`;
  - runs `tools/setup_required_plugins` (or `just install-venv`) first, so sase-github
    is installed as in CI.
- Do not run `just check-full` without an explicit instruction.
- TUI phases:
  - run targeted `just fix-tui-screenshots -- <selectors>` and inspect every created,
    updated or removed golden;
  - run targeted `--check` twice, once with `TZ=UTC`;
  - use `sase screenshot` to confirm the real keyboard flow and the narrow layouts.
- Never create a live repository. Use fakes, a temp `SASE_HOME`, and local bare remotes.
- A check failure that reproduces identically on the clean base is recorded with
  evidence, and with any existing tracking bead, as a `PROPOSED FOLLOW-UP:` note on the
  worker's phase. It does not block the phase. Known ones are `sase-1k4`, `sase-1ic`,
  and the `sase-1jm` notes.
- Workers never file task beads themselves.
- New public symbols need real consumers. Use a temporary `--epic-symbol` keyed to a
  still-open bead only when a later phase of this child epic consumes the symbol.
- Run `sase bead epic-symbols <phase>` before closing a phase.

## Landing

Workers close only their own phase, with a verification note. They never close this
child epic, `sase-1jx.13`, or `sase-1jx`.

This child epic's land agent:

1. Verifies every item above against the code.
2. Moves sase's `sase-core-revision.txt` past `provider-gaps`' sase-core commit (see
   `docs/rust_backend.md`), and runs `tests/core/test_project_onboarding_facade.py` and
   `tests/project_onboarding/`.
3. Confirms that the latest Master Gate run has no New Project failures.
4. Closes this child epic.
5. Resumes the interrupted `sase-1jx.13` landing through `parent_bead`, as its land
   prompt describes for a parent plan bead. That landing includes:
   - re-reading `sase-1jx.13`'s landing note (follow-up triage is already recorded
     there);
   - checking post-child drift;
   - `sase bead epic-symbols sase-1jx.13`;
   - closing `sase-1jx.13`;
   - `just symvision`;
   - setting `status: done` in `plan:202610/new_project_wizard_landing_repairs.md`.
6. Continues to `sase-1jx` while each parent remains complete, as recorded in
   `sase-1jx.13`'s plan.
7. Ends that landing by proposing these human-only post-landing steps through
   `/sase_gate`, in this order:
   1. the global reinstall;
   2. `sase skill init --force`;
   3. the one manual disposable-repo create smoke.
