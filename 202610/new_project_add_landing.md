---
tier: tale
title: Finish the add-existing wizard and land sase-1jx.10.1
goal:
  The add-existing wizard handles observed Configure facts, resume and asynchronous
  failures correctly, and sase-1jx.10.1 and parent phase sase-1jx.10 close normally.
size: medium
proposed_by: bbugyi200.athena.sase-1jx.10.1.land
bead: sase-1jx.10.1
status: done
---

- **PARENT:**
  [202610/new_project_add_existing.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_add_existing.md)
- **BEAD:**
  [sase-1jx.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/sase-1jx.10.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1jx.10.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1jx.10.1.land.md)
- **COMMITS:**
  - [81377d5](https://github.com/sase-org/sase--plans/commit/81377d5c33d16e1bfa7bc40a23719d81ecdc5e98)
    — docs(plans): mark add-existing New Project epic plan done

# Finish the add-existing wizard and land sase-1jx.10.1

Repair the bounded frontend defects found by the land review, then finish the landing in
this same coding turn. This tale has no land agent. Do not wait for your own commit,
SHA, push, or CI result before the closeout below; the host commits after your turn
ends.

## Already verified and binding constraints

Read `sase bead read sase-1jx.10.1 -r "Need landing audit and follow-up triage"`, its
four closed phase children, and
`sase bead read sase-1jx.10 -r "Need parent phase scope"`. Use audited reads for
`plan:202610/new_project_add_existing.md` and the binding add-existing sections of
`plan:202610/new_project_wizard.md`.

The original phase commits are `86b8fe2cce`, `393d0a4a5a`, `5714395778`, and
`558a45eca4`. The lander reviewed their source, every phase note, and concurrent commits
through `558a45eca4`, particularly onboarding executor split `313492a4f0` and CLI-create
`597931b222`. The add request contract still matches the CLI. The shared init controller
preserves the pane's argv, scopes, timeouts, and terminal valve. Sixty-six focused
models/configure/review/apply/init/modal/producer-inventory tests pass, but their fake
previews and callbacks miss the defects below. All eight
Configure/Review/Progress/Failed/Ready PNG goldens were inspected, including narrow
Review at 80x24.

Keep the existing Rust planner, CLI apply, journal, proc, and init implementations as
the shared backend. The repairs are TUI presentation and thin adapters to those APIs; do
not duplicate domain rules. Do not introduce new journal schemas, CLI options, a feature
flag, or an alternative apply engine. Do not edit memory. Parent decisions remain final:
no `decisions:project-create-human-only` record; the glossary change belongs to the
earlier CLI phase. Private sidecar creation stays in init. Entry keymaps and create mode
belong to still-open `sase-1jx.11` and `sase-1jx.12`. Do not wire those entry points or
consume `record_visibility_selection` here.

Read `tui_perf.md`, `tui_screenshot.md`, `lint_and_test.md`, and `symvision.md` through
`sase memory read`. Keep all new modules below 700 lines; extract remaining
Configure/resume glue from `new_project_modal.py` if needed. Source pixels and its
keyboard flow must stay stable. Provider strings remain sanitized literal Rich `Text`.
Slow work uses cancellable pump-free tasks or the existing session workers; timer
callbacks remain synchronous and free of I/O.

## 1. Make real Configure plans match the observed source

`NewProjectModal._prepare_configure` passes only target spelling and project records
into `validate_draft`. `validate_draft` supplies facts with only `projects`.
Consequently real Configure always loses `can_push`, `sase_managed`, checkout state, and
the matching disabled record. Even adding those first two fields to the target does not
help: `build_plan_document` reads them from `facts.remote`. A direct call with a known
managed/no-push target yields `can_push=None`, `sase_managed=None`, setup
`unknown_push`, a visible unchecked checkbox, and no amber follow-up.

Keep records-only local identity validation, and build the Configure display plan with a
snapshot of facts already available from the Source probe cache and local inventory. Add
a thin TUI adapter that passes these observed facts through the existing
`build_plan_document(..., facts=...)` contract. Reuse a matching probe; if one is
unavailable, an off-thread bounded probe may populate the initial card, but typing must
never call `gather_facts`, spawn a subprocess, or invoke a provider probe. Show
unknown/loading honestly until observation exists. Do not infer push rights or
management from provider names.

Preserve the planner's setup resolution and copy: managed hides the checkbox; no-push is
unchecked/amber with `sase init -M`; bare git omits sidecar prose; disabled registration
says it will be re-enabled. Do not make provider facts part of a second identity
allocator. Invalidate previews immediately when the draft or source changes, and recheck
selected identity, provider, draft, and generation after every await. An old worker
returning during the 300 ms debounce must not paint an obsolete draft. Keep setup
auto/on/off and name/aliases/current choices through back navigation. Restrict provider
switching to Source or deliberately return there before switching; hidden provider
shortcuts in Progress must not change the workflow associated with the in-flight add or
success preferences.

## 2. Hydrate failed resumes into a usable, checked Review

`_source_chosen` handles configure/show/jump but has no resume branch. Choosing the
existing fixture's failed `sase-org/forge` resume row leaves `stage=review`,
`draft=None`, Review document `{}`, and zero check submissions: the user cannot resume
or go back to a populated Configure.

Add an asynchronous resume adapter. Use the existing `lookup_journal(identity)`
off-thread and, when present, its `proc_id` plus existing proc/operation request APIs to
recover the prior add intent and editable choices. Validate the recovered operation,
mode, provider, and target identity before using it. Recover a draft from the persisted
request or the proc's logical command; never execute a stored command or copy proc
transport/result flags into the new command. Rebuild a plan using the current backend
and run the normal Review check. Preserve name, aliases, setup override, and
make-current. Back opens the corresponding Configure card.

Legacy journals exposed by `list_incomplete` carry identity/ref/status but not a full
intent. If the original intent cannot be recovered, build a fresh checked add plan for
that ref and clearly show that it requires review; do not pretend the fingerprint
reconstructs missing choices. Missing or mismatched context degrades to a recoverable
Review error, not an empty stage or a mutation. Running resume and already-enabled
Source outcomes retain their existing hand-offs. A late resume read cannot overwrite
another selected row or a dismissed modal.

## 3. Protect Review against stale checks and rejected submissions

`NewProjectReview.start_check` accepts every completion while mounted. Starting a check
for `newest` and then delivering an older completion for `obsolete` overwrites the argv
and confirmable plan. If `_submit_session_worker` returns `None`, Review remains
`_loading=True`, `_view=None` forever. `set_error` also leaves any prior confirmable
`_view` intact.

Associate each check with its full reviewed intent and a generation. Invalidate it on
new checks, back navigation, supplied-document replacement, and teardown. Only a
completion for the current intent while Review is active may enable confirmation. Clear
confirmability on error and throughout loading. Handle rejected or duplicate submissions
immediately with useful recoverable state; revisiting or retrying must converge to a
fresh checked plan. Include the intent fingerprint in the check's dedup identity if
needed so a newer draft is not silently aliased to an older check of the same
repository. Preserve bounded check argv/cwd/timeouts and update producer inventory if
the submit function moves.

## 4. Finish init from verified outcomes

`NewProjectInitHost.reload_projects` calls `_finish(True)` whenever a terminal valve
returns and the host is neither waiting for check nor apply. The controller ignores the
terminal subprocess return code. Thus a canceled/failed terminal init paints Ready as
initialized and loses the retry affordance.

Use an explicit controller outcome hook for the wizard instead of treating a reload or
toast wording as proof of success. Preserve pane gestures, messages, reloads,
check/apply argv, and the terminal valve without `--yes`. Returning from the wizard's
terminal valve must inspect the exit outcome and recheck the same scope through the
shared controller before marking initialized. Failed/canceled/ unverified outcomes
remain registered on Ready with the amber not-initialized line and `i` retry. A verified
current/success outcome adds the initialized checklist row once. The check, preview
cancellation, duplicate scope, missing queue, apply failure/partial result, terminal
cancellation/nonzero result, and dismissal paths must all terminate sensibly without
touching unmounted widgets. Do not run a second durable add when only init needs retry.

## 5. Observe terminal proc failures and honor the log change token

`NewProjectProgress._poll_log` skips unchanged versions only after a result exists, so
it schedules disk-tail work every 250 ms throughout an unchanged running log. It also
relies entirely on NDJSON: submission failures or CLI errors before a result event leave
Progress polling forever. The module-level add completion callback currently only
reloads Projects.

Keep the 250 ms metadata/spinner tick, but skip proven unchanged authoritative log
versions. Preserve a bounded off-thread fallback for the observed version-zero/
missing-projection case, so live logs still advance when observer metadata lags.
Coalesce reads, guard proc/attachment identity across awaits, and stop polling and tail
subscriptions once the terminal result has been handled. A late read of the old add log
must not overwrite the initialized checklist or emit a second outcome.

Use the existing proc terminal/completion/result evidence to drive Failed when an add
settles unsuccessfully without NDJSON. Keep any checklist/raw diagnostics and the
existing retry/back/terminal actions. Handle both pending-handle submission errors and a
real proc exiting before a result event. Distinguish still-running empty logs from
settled failure; do not synthesize a successful Ready without a project identity. Keep
app-owned completion/reload after wizard dismissal; any optional mounted-modal observer
must be weakly held and guarded.

## 6. Correct completed-stage chrome and verify the repairs

`render_stepper` currently maps progress/ready/failed to active index 2; all three show
`● Review`. Map them to all three steps done, as required by the child plan. Keep
source/configure/review rendering unchanged. Replace the Source-only hints on Configure
and Review with their actual field/review/back/apply controls; hints must not advertise
hidden Source actions during checks or another stage.

Add meaningful regressions using real local plan construction and captured worker
callbacks, instead of only `_patch_fast_preview`/`show_document`:

- Managed, no-push, local existing checkout, and disabled Configure cards; newest draft
  wins even when an old preview returns before debounce expiry; editing does not invoke
  a provider hook or subprocess.
- Failed journal resume into checked Review, restored draft, back navigation, missing
  legacy context, and stale resume completion; running/enabled hand-offs remain intact.
- Out-of-order Review completions, back/reenter with a changed name, rejection, error
  after a valid view, and blocked confirmation.
- Actual shared init-controller callbacks through cancellation, failure, terminal
  exit/recheck, success, retry, and dismissal; do not just call `_on_init_finished` by
  hand in all tests.
- Unchanged authoritative log versions, version-zero live fallback, late old tails,
  unsuccessful settlement without NDJSON, and dismissal-during-run reload.
- Stage completion and relevant key handling without switching providers mid-apply.

Run the focused wizard suites, `test_projects_pane_init_flow_*`, init-plan modal suites,
and producer inventory. The lander passed 66 tests on the original tree. Run `just fmt`
then `sase tool run check` (the guarded form of `just check`). Do not run
`just check-full`. Inspect every NEW/UNKNOWN failure; baseline evidence below allows
unrelated failures to be reported without turning them into epic work. Use
`/sase_monitor` for a long command and finish the required inspection/closeout in its
successor; do not use prepared completion to bypass this tale's closeout.

Run targeted `just fix-tui-screenshots` for the New Project source, configure/review,
and apply visual modules. Inspect the report and every changed/created/removed golden.
Correct completed-step pixels and stage hints intentionally, retain the 80x24 public
warning/exact argv, and keep Source pixels unchanged. Check the targeted goldens again
with `--check` and with `TZ=UTC`, following screenshot memory. Fixtures push the wizard
directly and fake proc/provider effects; never create a live repo.

## 7. Final step: close the epic and its parent phase in this turn

First re-read current epic/children and compare source drift since `558a45eca4`,
particularly later .11/.12 consumers of these repaired seams. Ensure all four original
children remain closed and no newly attached descendant is unfinished. Retain the
lander's follow-up outcomes in the close note:

- `.1` note #1 and `.2` note #1: duplicate stale whitelist proposals declined because
  `8ed37c4ef8` privatized journal helpers/re-keyed archive context and `52e10f302a`
  retired the remaining archive exemption.
- `.3` note #1: declined because `ebd17bf5bf` removed the restored-row exemption once
  its real consumer landed.
- `.4` note #1: independently reproduced 3517 modules against `<3513` and corroborated
  existing task `sase-1ic` (+10 independent reports). Before this child epic,
  `tools/tui_import_closure --diff 86b8fe2cce^` measures 3518; only an Archive module
  was removed and no wizard module entered startup. It is unrelated.
- Additional `just symvision` failure: private `_load_archive_bundle` imported from
  `_archive_reader.py` by `_archive_restore.py:169/:177`, introduced by `ebd17bf5bf`.
  Task searches and the last-week CI sweep found no duplicate. It was recorded as
  `DISCOVERED ISSUE` on causally owning active epic `sase-1jm`, after reading that epic
  and phase `.5`. Do not fix Archive source in this tale.

These outcomes are already recorded on `sase-1jx.10.1`; re-triage only genuinely new
non-epic evidence through `/sase_new_task`. No task bead is needed for these repairs.

Run `sase bead epic-symbols sase-1jx.10.1`. It was empty at the landing audit. Resolve
every current entry by wiring, privatizing, deleting, or adding a justified non-test
pragma; re-key only to a still-open later bead with a real planned consumer. Preserve
valid `.11(NewProjectModal)` and `.12(record_visibility_selection)` entries. Then close
normally:

```sh
sase bead close sase-1jx.10.1 --note "Verified all four phase scopes and notes, planner-backed Configure and recovery, current-intent Review, durable apply and terminal proc outcomes, shared init and terminal recheck, completed-stage chrome, focused tests, inspected New Project goldens, and integration with concurrent onboarding/create/entry changes; include actual check evidence and the follow-up outcomes listed in this tale."
just symvision
```

Use an accurate expanded note with actual results. If unrelated Archive lint still
fails, record that specific existing evidence; ensure there are no stale whitelist
errors or new wizard errors. Never claim a passing command that failed. A close
rejection for symbols is a requirement to finish symbol cleanup and retry. An unfinished
descendant needs normal completion/reopening; never force merely to succeed, and never
force a successful nested landing.

Open the plans sidecar using `/sase_repo`
(`sase repo open plans -r "Mark verified add-existing epic plan done"`), read its
AGENTS.md if present, and use audited artifact reads for content. In the opened repo,
set only `status: done` in the frontmatter of `202610/new_project_add_existing.md` (the
PLAN shown by the epic). Do not edit the outer wizard plan's status. Include this
sidecar change in your final declaration.

After this epic closes, run `sase bead read sase-1jx.10.1 -r "Need the parent link"`.
Its concrete parent is phase `sase-1jx.10`. Verify that its add-existing scope is
complete, run `sase bead epic-symbols sase-1jx.10`, resolve/re-key any remaining entries
under the same policy, and close only that phase normally:

```sh
sase bead close sase-1jx.10 --note "Verified child epic sase-1jx.10.1 and this repair tale complete the planner-backed existing-repo Configure, exact checked Review, durable Progress/recovery, shared init terminal hand-off, and Ready/Failed journey; include focused test, golden, check, and symbol evidence."
```

Leave containing epic `sase-1jx` open for its already-waiting land agent. If the parent
phase has acquired new incomplete or ambiguous work, note the precise blocker on
`sase-1jx.10` and report it instead of forcing a close. Finish through `/sase_final`
with all repositories changed by this turn declared for host commit.
