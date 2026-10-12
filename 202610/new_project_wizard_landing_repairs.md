---
tier: epic
title: Repair New Project wizard defects found while landing sase-1jx
goal: Fix every epic-caused defect the sase-1jx land audit confirmed across sase-core,
  sase-github, and sase, so that `sase project suggest | add | create` and the Admin
  Center New Project wizard behave as plan:202610/new_project_wizard.md specifies.
  After this child epic lands, its land agent resumes and finishes the sase-1jx landing.
parent_bead: sase-1jx
phases:
- id: core-repairs
  title: Keep explicit create settings and fix candidate classification
  depends_on: []
  size: medium
  description: 'core-repairs: in sase-core project_onboarding, keep explicitly-off
    bool fields in effective_options, stop strict mode from flagging harmless off
    values of excluded fields, stop a numbered sibling from hiding its primary checkout,
    stop inventory seeding from creating ref-less rows, and move sase''s core pin.'
- id: gh-repairs
  title: Make GitHub create and discovery correct
  depends_on: []
  size: medium
  description: 'gh-repairs: in sase-github, stop template creates from passing flags
    gh rejects, verify every requested setting after create, re-probe after a create
    timeout, paginate team and org lists, strip ports when matching hosts, fix numbered-copy
    and two-level host detection, keep Enterprise hosts in canonical refs, and document
    SASE_GITHUB_CLONE_TIMEOUT.'
- id: cli-repairs
  title: Fix CLI create and suggest gaps
  depends_on:
  - core-repairs
  size: medium
  description: 'cli-repairs: in sase, make create apply_argv round-trip for every
    provider (prefixed bare-git refs, visibility only when the form has it), honor
    bare-git custom directories, report exists-without-evidence as a remote_exists
    blocker, stop suggest -o from failing providers without owners, make suggest and
    create tests hermetic, and fix the skill and workspace docs.'
- id: tui-entry-repairs
  title: Finish Projects-tab entry and post-completion integration
  depends_on: []
  size: medium
  description: 'tui-entry-repairs: make the palette wizard opener one-shot, implement
    width-aware priority hints, pass current-project context into the wizard, share
    one post-add sequence (reload, select, MRU, lifecycle, tag catalog, Show new project
    when filtered), handle added outcomes, fix Ready + and c, record preferences after
    a mid-run close, and add the missing tests, goldens, and docs.'
- id: tui-source-repairs
  title: Meet the Source stage spec and add its missing guards
  depends_on:
  - tui-entry-repairs
  size: medium
  description: 'tui-source-repairs: make loading rows unselectable, guard the t remedy,
    collapse the detail card at 80x24, implement the missing Source copy and health-chip
    states, cancel per-owner tasks at teardown, stop provider-preference clobbering,
    and add the planned perf, debounce, paste, and health-chip tests plus an 80x24
    golden.'
proposed_by: bbugyi200.athena.sase-1jx.land
create_time: 2026-10-12 14:55:54
status: wip
bead_id: sase-1jx.13
---

- **PROMPT:** [prompts/202610/new_project_wizard_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/new_project_wizard_landing_repairs.md)
- **PARENT:** [202610/new_project_wizard.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_wizard.md)
- **BEAD:** [sase-1jx.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/sase-1jx.13.md)

# Repair New Project wizard defects found while landing sase-1jx

## Scope and sizing

Epic `sase-1jx` (accepted design `plan:202610/new_project_wizard.md`) has all twelve
phases closed. Its three nested child epics (`sase-1jx.7.1`, `sase-1jx.10.1`,
`sase-1jx.12.1`) are also closed. The land agent read every bead note and audited the
code on master `101c853939` against the design. That audit found the defects below. All
are caused by this epic: each one is either a design requirement that was not met or a
behavior the epic introduced that is wrong. No `--epic-symbol` entries remain for
`sase-1jx`.

The work spans three repositories, so it has five phases that different agents can run
in parallel.

- `core-repairs` runs in **sase-core**, then bumps sase's pin.
- `gh-repairs` runs in **sase-github**.
- `cli-repairs` runs in **sase** CLI and engine code.
- `tui-entry-repairs` and `tui-source-repairs` run in **sase** TUI code. They run in
  sequence because both touch the wizard modal files and the PNG goldens.

Open linked repositories only with `sase repo open <name> -r "<why>"` and work in the
path it prints. Run checks there with `sase tool run check`.

Every phase inherits the parent epic's decisions, which are final:

- `human_only_record = no`: do not add a `decisions:project-create-human-only` record.
- `glossary_project = yes`: already implemented.

No memory note may be edited in this child epic.

Each item below cites where it was observed. Workers must re-confirm each item against
current master before changing code. If an item is already fixed, record that in the
phase's closing note instead of re-implementing it.

## Phase core-repairs (sase-core, then the sase pin)

Module: `crates/sase_core/src/project_onboarding/` in the linked `sase-core` checkout.

1. **Explicit "off" bools are dropped.**
   - Today, `form.rs` (around lines 88-92) puts an available field into
     `effective_options` only when `field_is_set` is true or the value equals a
     non-empty default. A default-true bool that the user set to `false` matches neither
     rule, so it vanishes.
   - `tests.rs::defaults_fill_effective_options_and_bools_normalize` asserts that
     dropping (`wiki=off` is absent).
   - sase sends `effective_options` to `ws_create_repo` on two paths: the CLI
     (`src/sase/project_onboarding/_apply_run.py`) and the TUI (lenient effective
     options).
   - Result: `issues=false` and `wiki=false` silently disappear from a GitHub create.
     The plan says "never drop a setting silently".
   - **Fix:** always emit an available bool field's normalized `"true"`/`"false"` value.
     Update the table test and add strict and lenient cases for an explicit `false` on a
     default-true bool.
2. **Strict mode flags harmless off values.**
   - Strict mode reports a `conflict` for `template=o/t` plus `add_readme=false`, even
     though the excluded field is merely off.
   - **Fix:** treat a supplied value on an unavailable field as a conflict only when it
     requests the excluded behavior. An off bool or an empty value is not a request.
     Keep real conflicts (for example `template` plus `gitignore=Python`) as errors. Add
     table tests for both.
3. **A numbered sibling hides its primary.**
   - In `classify.rs` (around lines 382-385), local entries that share an identity are
     merged with `numbered_copy` combined by OR, and `local_path` is overwritten by
     whichever entry comes last.
   - Result: a primary checkout plus a `_2` sibling yields no group row and
     `hidden=[numbered_copy:1]`.
   - **Fix:** mark a merged row `numbered_copy` only when every local entry is numbered,
     and prefer the non-numbered path as `local_path`. Add a table test with the primary
     and the sibling given in both orders.
4. **Inventory seeds dead rows.**
   - `seed_from_inventory` (around line 94) creates rows with an empty `ref` whose
     display is the identity key. For example, an external-only
     `github.com/steveyegge/beads` inventory entry lands in Suggested, but neither the
     TUI nor the CLI can act on a ref-less row.
   - **Fix:** inventory either annotates rows that other sources produced, or seeds only
     kinds that carry a usable ref. Never emit a ref-less selectable row. Add a table
     test.
5. **Bindings and pin.**
   - Keep the PyO3 wire shapes unchanged, or update the binding tests and the facade in
     `src/sase/core/project_onboarding_facade.py` if they change.
   - Run `sase tool run check` in sase-core.
   - Then move `sase-core-revision.txt` in sase past the new sase-core commit (see
     `docs/rust_backend.md`), and run the sase facade and onboarding tests
     (`tests/core/test_project_onboarding_facade.py` and `tests/project_onboarding/`).
   - If explicit-false options change any committed sase expectations (create
     `apply_argv` goldens, Review rows), update those in this phase.

## Phase gh-repairs (sase-github)

Module: `src/sase_github/workspace/onboarding.py` and its helpers in the linked
`sase-github` checkout. Read its `CLAUDE.md` or `AGENTS.md` first.

1. **Template creates always fail.**
   - `create_gh_repo` appends `--add-readme` whenever `add_readme` is missing (default
     true). It also passes `-g`, `-l` and `-t` alongside `-p`. gh 2.100 rejects
     `--add-readme` (and the other three) together with `--template`.
   - `tests/test_github_onboarding.py::test_argv_exact` asserts exactly such an invalid
     argv.
   - **Fix:**
     - Never emit `--add-readme`, `-g`, `-l` or `-t` together with `-p`.
     - Default a missing `add_readme` to the form's default only when no template is
       set.
     - Return `failed` with a clear message before any subprocess when the options
       conflict.
   - **Tests:** replace the invalid argv test with valid argv tests (with and without a
     template), and add the plan's "conflicts rejected before any subprocess" test.
2. **Post-create verification is incomplete.**
   - It reads only `id,visibility,url`.
   - **Fix:** also read `hasIssuesEnabled`, `hasWikiEnabled`, `description` and
     `homepageUrl`, and report every requested setting the repository does not reflect
     as a mismatch.
3. **A create timeout is reported as plain failure.**
   - `CREATE_TIMEOUT_SECONDS=30`, and a timeout returns `failed` without checking
     whether the repository was created.
   - **Fix:** re-probe after a timeout and report `exists`/created the same way the
     "already exists" path does, so the journal can adopt the repository. Use a
     generous, documented timeout.
4. **Team and org lists stop at 30.**
   - `gh api orgs/{o}/teams` and `user/memberships/orgs` return only the first page.
   - **Fix:** paginate with `--paginate`, keeping the 24h cache.
5. **Ports break host matching.**
   - `config.parse_github_remote_url` keeps the port, so
     `ssh://git@github.com:22/o/r.git` gives host `github.com:22`.
   - Since gh-hygiene, `ws_detect_workflow_type` therefore returns `None` for such
     origins, and local discovery skips those checkouts. `parse_gh_repo_ref` already
     strips ports, so the two disagree.
   - **Fix:** strip the port in `parse_github_remote_url`, and add detection and
     discovery tests.
6. **Numbered copies and Enterprise layouts are misdetected.**
   - Local discovery skips any checkout whose origin identity differs from its directory
     name. So real copies like `bbugyi200/CV_10` (origin `bbugyi200/CV`) are dropped,
     and `numbered_copy` fires only for repositories literally named `*_N`.
   - `_iter_candidate_dirs` also resolves two-level `<owner>/<repo>` directories against
     the configured default host. Confirm how `github_workspace_dir` lays out github.com
     and Enterprise checkouts, then map each depth to its real host.
   - **Fix:** strip a trailing `_N` before comparing the identity, and set
     `numbered_copy` only for those copies. Add tests for a real numbered copy, a
     literal `name_2` repository, and an Enterprise default host.
7. **Enterprise refs lose their host.**
   - For a non-default Enterprise host URL, `parse_gh_repo_ref` returns a bare
     `owner/repo` ref. Clone or create through that ref then targets the default host.
   - **Fix:** give non-default hosts a host-qualified canonical ref that the provider's
     resolve, clone and create paths accept. `test_enterprise_host_url` should assert
     the ref.
8. **Documentation is missing.**
   - Document `SASE_GITHUB_CLONE_TIMEOUT` and the onboarding hooks in sase-github's
     `docs/configuration.md` (and the README if it lists environment variables).
9. **Verification.**
   - Run `sase tool run check` in sase-github.
   - If its venv cannot resolve `sase>=0.18.0` from PyPI (tracked by `sase-16p`), set up
     the venv from the sase source the way sase-github's Justfile documents, with
     `SASE_PYTHON_PATH` and `SASE_RUST_CORE_PATH`.
   - Never run a live create; use fake `subprocess.run` or a fake `gh` on `PATH`.

## Phase cli-repairs (sase engine and CLI)

1. **The bare-git create apply argv cannot be run.**
   - `planning.py::_apply_argv` writes `target.ref`, which for bare git is the bare
     `NAME`. `creation.parse_create_ref` rejects bare names.
   - `project_handler_add.py` (around lines 212-215) and `planning.py` (around lines
     97-99) default `creation.visibility` to `"private"` for every provider. So
     `--visibility private` is always emitted, and strict validation then blocks it
     (`invalid_option: unknown option visibility`), because the bare-git form has no
     visibility field.
   - The wizard submits `apply_argv` unchanged
     (`ace/tui/modals/new_project_review_model.py`), so bare-git creates from the wizard
     always fail.
   - **Fix:**
     - In create mode, emit a ref that `parse_create_ref` accepts (for example the
       provider-prefixed `git:NAME`).
     - Default visibility, and emit `--visibility`, only when the provider's creation
       form has a visibility field.
   - **Tests:** feed `apply_argv` back through the CLI request builder and planner for
     both a gh create and a bare-git create. Also extend
     `test_bare_git_create_end_to_end` to run the emitted argv.
2. **Bare-git custom directories are ignored.**
   - `ws_create_repo` creates the bare repo at the custom `bare_dir` and returns
     `parsed.ref` set to that path.
   - But `_adopt_create_result` (`_apply_effects.py`, around line 125) updates only the
     checkout, not `ctx.ref`. `_effect_checkout` then resolves the bare name, which
     creates a second bare repository at `~/.sase/repos/<name>.git` and orphans the
     custom one.
   - **Fix:**
     - Carry the create result's `parsed.ref` (and its checkout) into the apply context
       and the journal.
     - Either honor `checkout_dir` end to end, or remove that field from the bare-git
       creation form (`bare_git_workspace.py`, around line 347) if honoring it would
       need a broad provider change. Document the choice in the phase note.
   - **Tests:** a temp-`SASE_HOME` test with a custom `bare_dir`.
3. **`exists` without evidence fails instead of blocking.**
   - When `ws_create_repo` reports `exists` without journal evidence during apply,
     `_apply_effects.py` (around lines 96-122) re-plans but looks only at the
     `create_remote` step state. It returns `failed` and marks the journal failed, which
     later shows the row as RESUME.
   - The plan wants a `remote_exists` blocker, which `planning` already produces at plan
     time.
   - **Fix:** surface the re-plan's blockers as a `blocked` result and leave the journal
     non-resumable.
   - **Tests:** correct `test_exists_without_evidence_is_remote_exists`
     (`_apply_create_remote_cases.py`) so it asserts the blocker.
4. **`suggest -o OWNER` reports a fake failure.**
   - With no provider argument, `discovery.py` (around lines 337-343) applies the owner
     filter to every provider. The bare-git provider has no namespaces, so it reports
     `⚠ sase-org: No workspace plugin found for workflow type 'git'`.
   - **Fix:** apply `--owner` only to providers that expose namespaces or an account,
     and add a test.
5. **Hermetic tests.**
   - `tests/main/test_project_handler_suggest.py::test_options_forward_to_discovery` and
     `::test_explicit_provider_is_remembered` pass `provider="gh"` and rely on the live
     workspace-provider registry. They fail whenever sase-github is not installed or is
     not registered (recorded as DISCOVERED ISSUE note #3 on `sase-1jx`). **Fix:**
     monkeypatch `project_handler_onboarding._known_providers`.
   - `tests/project_onboarding/test_cli.py::test_create_gate_option_allows_apply` stubs
     the request builder, so `SASE_GATE_COMMAND` is never read. **Fix:** let the real
     actor detection run and assert `actor == "human"`.
   - Add a `sase project suggest --help` test, and make the two "sorted options" tests
     in `test_cli.py` actually assert option order.
6. **Skill text.**
   - `src/sase/macros/skills/sase_project.md` (around lines 56-58) offers agents
     `SASE_PROJECT_CREATE_BYPASS` as an alternative.
   - **Fix:** agents propose the exact `sase project create … --yes` command through
     `/sase_gate`. The bypass is for humans, or when a user explicitly instructs it.
   - Do not deploy skills (`sase skill init --force` happens after landing).
7. **Docs.**
   - Add the five onboarding registry functions to the Registry Functions table in
     `docs/workspace.md` (around line 175). Fill in the table's older missing rows while
     there.
   - Note the bare-git create ref form and the visibility behavior in `docs/cli.md`.

## Phase tui-entry-repairs (sase TUI)

Read `tui.md` (and `tui_perf.md` and `tui_screenshot.md` as linked) through
`sase memory read` first.

1. **The palette opener fires on every Projects activation.**
   - `actions/_base_admin.py` (around lines 203-216) folds the one-shot
     `on_tab_activated` into the Admin Center's permanent tab-activation callback, which
     `ConfigCenterModal._switch_to` calls on every activation.
   - Result: after _Projects: New project…_, closing the wizard and returning to
     Projects reopens it.
   - **Fix:** make the callback truly one-shot, and add a test that drives
     `action_open_new_project` in the real app.
2. **The hints row is not width-aware.**
   - `project_management_rendering.py` (around lines 297-348) only prefixes `n new` to a
     fixed line. The 120x40 golden clips mid-word.
   - **Fix:** implement width-aware priority truncation as the plan specifies:
     - lead with `n new`;
     - drop the lowest-priority hints first;
     - end with `? help` when anything was dropped;
     - take the pane width at render time from cached widths, with no measuring on hot
       paths.
   - **Tests:** hints at 80, 120 and 200 columns.
3. **The wizard gets no current-project context.**
   - `projects_pane.py::action_new_project` pushes `NewProjectModal()` without the
     pane's current-project context.
   - So the current-provider tab fallback and the create default owner (the current
     project's owner) are dead code.
   - **Fix:** pass the workflow type and owner, and test both defaults.
4. **There is no shared post-add sequence.**
   - **Fix:** create one helper that runs on `show(key)`, `added(key)`, the Ready `⏎`,
     and the durable on-complete reload. It must:
     - reload and select the key;
     - switch to the PROJECTS sub-tab when needed;
     - call `_refresh_launchable_mru_snapshot` and `_notify_lifecycle_changed()`;
     - warm `ensure_project_tag_catalog()` off-thread;
     - when a filter would hide the new project, keep the filter and offer the plan's
       "Show new project" status action instead of clearing it (today `_reload_projects`
       clears it).
5. **Not every outcome is handled.**
   - Ready `esc` returns `None`, so the new project is never selected.
   - **Fix:** return `added(key)` and handle it in the pane the same way as `show`.
6. **Ready `+` only works by accident.**
   - `new_project_apply.py::_ensure_catalog_and_launch` calls
     `_dismiss_center_underneath()` and then `self.dismiss(...)`. Textual's `dismiss`
     pops the top screen, so the wizard's own callback is dropped.
   - **Fix:** dismiss the wizard first (delivering its outcome), then the Admin Center.
     Then start the agent. Add a test.
7. **Ready `c` bypasses the set-current worker.**
   - It reimplements set-current and skips `_invalidate_current_project_indicator` (the
     launch-context invalidation and MRU refresh).
   - **Fix:** route it through the pane's existing set-current worker or the same
     invalidation, and add a test.
8. **Preferences are lost after a mid-run close.**
   - If the wizard is closed mid-run, `_on_project_add_complete` gets an empty intent,
     so the last provider and owner are never recorded.
   - **Fix:** derive them from the durable proc's request, matching how create already
     persists visibility after dismissal.
9. **Tests and goldens.**
   - Add tests for:
     - `n` binding and gating (active only on the PROJECTS sub-tab; never a second
       wizard while the filter input is focused);
     - palette execution;
     - outcome handling, including the filtered case;
     - the empty-state text `No projects yet — press n to add or create one`.
   - Add PNG goldens for the Projects tab hints at 120x40 and 80x24 and for the empty
     state. Refresh the projects goldens the hint change touches.
10. **Docs and docstrings.**
    - `docs/configuration.md`: add a `new_project` row to the projects keymap table.
    - `docs/ace.md`: add the "Add or create a project" section (stages, keys, setup,
      recovery; covering both add and create). Replace the stale add-only paragraph.
    - Fix stale docstrings that still say the modal is test-pushable or that the
      production create affordances are deferred (`new_project_modal.py` module
      docstring, `enter_create_configure`).
    - Keep the help text within the 32-character rule.

## Phase tui-source-repairs (sase TUI)

1. **Loading rows are selectable.**
   - `◌ fetching owner…` rows count as selectable (`new_project_source_model.py`, around
     lines 219-227). `choose_highlighted` and `_row_selected` skip only error rows.
   - Result: `⏎` while loading can enter Configure with an empty ref.
   - **Fix:** render them as disabled options and never select a row without a ref.
2. **The `t` remedy is fragile.**
   - `new_project_source.py` (around lines 217-226) does not catch `SuspendNotSupported`
     or `OSError`, while the Failed-stage `t` does.
   - `t` acts only on a highlighted owner-error row, but the health chip advertises
     `t …` globally.
   - **Fix:** catch both errors. Make `t` run the provider-level remedy whenever the
     chip advertises one.
3. **The detail card never collapses.**
   - At 80x24 the candidate list gets 2 rows. The narrow golden was taken at 84x28
     instead of the planned 80x24.
   - **Fix:** collapse the card based on height, and add a Source golden at 80x24.
4. **Spec copy and states are missing.** Implement these from the plan's Stage 1
   section:
   - the truncation footer (`showing 200 · type to filter or paste owner/repo`, driven
     by `owner.truncated`);
   - the no-owners card that points to `github_orgs` in Config Center;
   - the health chip's cache age, `✗ gh not installed`, and the disabled provider tab;
   - the detail status wording (`✓ managed` / `○ will be set up` / `? unknown`);
   - the hidden enabled-project row text `already the project X — ⏎ show it`.
5. **Model ownership and teardown.**
   - Move health-chip text into the pure Source model, as the plan specifies, with unit
     tests.
   - Register and cancel the per-owner refresh tasks at unmount.
   - Replace `_persist_provider_selection`'s whole-object save with
     `record_provider_selection`, so it cannot clobber `last_owner`/`last_visibility`
     written by an off-thread create.
6. **Planned tests that are still missing.**
   - A perf guard: typing in the filter and moving the highlight spawn no subprocess and
     call no provider hook except the pure parse. Patch `subprocess.run` and the probe
     to raise.
   - The highlight paints immediately while the probe is debounced.
   - A bracketed paste focuses the filter.

## Verification

- Each phase runs focused tests, then `just fmt`, then `sase tool run check` in every
  repository it changed, following `lint_and_test.md`.
- Do not run `just check-full` without an explicit instruction.
- TUI phases run targeted `just fix-tui-screenshots -- <selectors>` and inspect every
  created or updated golden.
  - Run targeted `--check` twice, once with `TZ=UTC`.
  - Use `sase screenshot` to confirm the real keyboard flow and the 80x24 layout.
- Never create a live repository. Use fakes, a temp `SASE_HOME`, and local bare remotes.
- A check failure that reproduces identically on the clean base is recorded with
  evidence, and with any existing tracking bead, as a `PROPOSED FOLLOW-UP:` note on the
  worker's phase. It does not block the phase.
- Workers never file task beads themselves.
- New public symbols need real consumers. Use a temporary `--epic-symbol` keyed to a
  still-open bead only when a later phase of this child epic consumes the symbol.
- Run `sase bead epic-symbols <phase>` before closing a phase.

## Landing

Workers close only their own phase, with a verification note. They never close this
child epic or `sase-1jx`.

This child epic's land agent:

1. Verifies every item above against the code.
2. Closes this child epic.
3. Resumes the interrupted `sase-1jx` landing through `parent_bead`, as its land prompt
   describes for a parent plan bead. That landing includes:
   - re-reading `sase-1jx`'s landing notes (follow-up triage is already recorded there);
   - checking post-child drift;
   - `sase bead epic-symbols sase-1jx`;
   - closing `sase-1jx` with a verification note;
   - `just symvision`;
   - setting `status: done` in `plan:202610/new_project_wizard.md`.
4. After `sase-1jx` closes, hands the remaining post-landing steps recorded on
   `sase-1jx` to the user instead of running them itself:
   - the global reinstall that brings `sase project create` to the installed CLI;
   - the `sase skill init --force` deploy (`sase-1jx` note #4), which must follow that
     reinstall so the deployed `sase_project` skill never documents a command the
     installed CLI lacks;
   - the design's one manual disposable-repo smoke
     (`sase project create gh:<you>/sase-wizard-smoke -v private --yes`, deleted by
     hand).

   Agents never create live repositories, and global installs are human-only. Propose
   these steps, in that order, through `/sase_gate` as the landing's final action.
