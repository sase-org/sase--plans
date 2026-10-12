---
tier: epic
title: Complete repository creation in the New Project wizard
goal: Deliver the provider form, live create checks, danger Review, durable apply,
  and journal-backed recovery required by sase-1jx.12, then have this child epic's
  land agent close that phase.
parent_bead: sase-1jx.12
phases:
- id: provider-form
  title: Render provider creation fields and constraints
  depends_on: []
  size: medium
  description: 'provider-form: implement the standalone RepoCreationForm widget with
    all five field kinds, searchable typed choices, Advanced disclosure, and Rust
    lenient evaluation; verify constraints and deterministic widget snapshots without
    exposing a new production entry point.'
- id: create-configure
  title: Configure new repositories with live checks
  depends_on:
  - provider-form
  size: medium
  description: 'create-configure: implement the create draft and Configure stage,
    owner and visibility defaults, pure name normalization, debounced off-thread availability
    and checkout checks, Rust identity previews, and explicit add-instead transition;
    keep production Create affordances deferred until the complete path lands.'
- id: create-execution
  title: Review and apply the exact create intent
  depends_on:
  - create-configure
  size: medium
  description: 'create-execution: make shared Review and durable submission support
    project.create with exact provider options, visibility and danger styling; reuse
    Progress/init/Ready, persist successful per-owner visibility, register proc producers,
    and remove the consumed record_visibility_selection epic-symbol exception.'
- id: recovery-entry
  title: Recover incomplete creates and expose the complete wizard
  depends_on:
  - create-execution
  size: medium
  description: 'recovery-entry: recover create intent from durable evidence, retain
    created repositories and resume through checked journal-backed plans, then expose
    the capability-gated pinned Create row and Source n action; verify create-to-Ready,
    reopened-TUI recovery, keyboard flows, and required wide/narrow snapshots.'
proposed_by: bbugyi200.athena.sase-1jx.12
create_time: 2026-10-12 10:07:49
status: done
bead_id: sase-1jx.12.1
---

- **PROMPT:** [prompts/202610/create_repository_wizard.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/create_repository_wizard.md)
- **PARENT:** [202610/new_project_wizard.md](https://github.com/sase-org/sase--plans/blob/main/202610/new_project_wizard.md)
- **BEAD:** [sase-1jx.12.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/sase-1jx.12.1.md)

# Complete repository creation in the New Project wizard

## Scope and sizing

Implement the `tui-create` phase of the accepted design,
`plan:202610/new_project_wizard.md`, for assigned bead `sase-1jx.12`. That bead was
already in progress, has no earlier implementation notes, and depends on the closed
CLI-create and TUI-entry phases. Leave its status unchanged during implementation.

This is a child epic because the remaining work combines a new generic form widget,
asynchronous Configure state, durable command transport, and recovery across TUI
sessions. Each of the four sequential medium phases has a concrete boundary and can be
completed directly by one coding agent. The generated land agent integrates their
evidence and finishes the original phase. SASE records the parent association; do not
manually create beads.

The accepted parent design is authoritative, especially Stage 2b, danger Review,
provider hook contracts, recovery, and the `tui-create` acceptance list. Its final
decisions remain final:

- `human_only_record = no`: do not create or edit `decisions:project-create-human-only`.
- `glossary_project = yes`: only `glossary:sase-project` was authorized for memory
  edits. An audited read confirms it already describes CLI add/create and the wizard;
  this child epic requires no memory edits.

Do not create feature flags, change existing TypedInputForm host behavior, invent
provider-specific validation in Python, or change the human-only create guard. Agents
verify with fakes and read-only checks; live repository creation remains the parent
epic's human smoke test. Private-sidecar behavior remains owned by the existing init
flow.

## Verified implementation seams

- `src/sase/workspace_provider/_hookspec.py` defines `RepoCreationForm`,
  `RepoCreationField`, `RepoCreationChoice`, `ParsedRepoRef`, and `RepoProbe`; the
  registry exports `describe_repo_creation`, `parse_repo_ref`, and `probe_repo`. Owner
  and repository name are target identity, not form fields. The bare-git provider also
  supplies a form, with advanced `bare_dir` and `checkout_dir` options.
- `src/sase/core/project_onboarding_facade.py` exposes
  `evaluate_repo_creation_form(form, values, "lenient")` and the shared planner. The
  lenient result supplies field availability/reasons and `effective_options`; strict CLI
  validation already exists in `project_onboarding/creation.py` and `facts.py`.
- The existing New Project modal already hosts Source, Configure, Review, Progress,
  Ready, and Failed. Its pump-free tasks and generation checks provide the concurrency
  pattern. Configure helpers currently hardcode mode `add`, and the state holds only an
  add draft.
- `new_project_review_model.py` has a preliminary create REMOTE row, but
  `apply_submission` still chooses `project.add` and reconstructs only add options.
  `new_project_apply.py` likewise hardcodes the durable add operation. The CLI create
  handler checks that the durable request's `options` exactly matches argv; losing
  options is an actual transport mismatch.
- `new_project_resume.py` currently reads only `project.add` operation requests and
  rejects create journals. In `new_project_modal.py`, the broad `configure/resume`
  branch currently makes the subsequent failed-resume branch unreachable. Create
  recovery must correct the dispatch and preserve add recovery.
- `project_onboarding/planning.py` already accepts mode, creation, and options and
  builds create argv. Its display envelope currently takes target visibility from the
  probe alone; a missing new remote can leave Review without the requested visibility.
  Correct that thin envelope if needed, without duplicating Rust decisions.
- `ops/names.py` already registers `project.create`. TUI producer declarations live in
  `_proc_producer_sites_actions.py`; declarations and tests must match any new or
  generalized submission sites.
- `record_visibility_selection` is an existing public preference writer with a temporary
  `Justfile` exception keyed to `sase-1jx.12`. It needs a real production success-path
  consumer and removal of that exception.

## Contracts between phases

Use a separate frozen create draft containing owner, raw repository name, canonical
target, provider form values, and the existing project ConfigureDraft values. Add an
explicit wizard mode and create-draft state; keep add draft defaults and behavior
intact. Shared planner adapters take mode/creation/options explicitly with
add-compatible defaults. The create stage is a dedicated widget selected within
Configure; Source, Review, Progress, and Ready stay shared.

The form widget exposes raw values, the latest lenient evaluation, canonical effective
options, and a change message. Advanced disclosure hides controls without dropping their
values. Unavailable controls show reasons and cannot be selected; their values are
omitted according to the core result, not a second Python constraint engine. Pure Rust
evaluation may run directly; all provider description calls, disk observations,
plan-envelope I/O, probes, and subprocesses run outside the Textual pump.

A complete create request consists of `mode="create"`, human UI intent, canonical
target, project `requested` values, provider `options`, and creation visibility/summary.
Preserve this intent through Back, Review, durable operation storage, and retry. Do not
substitute add mode when a remote exists; only the explicit add-instead action does
that. The CLI actor detection and human-only apply guard remain authoritative.

Before the final phase, components and create integration are reachable from test
harnesses only. The existing production add wizard continues to work, and no Create row
or Source shortcut advertises an unfinished path. This uses ordinary internal
composition, not a feature flag.

## Phase provider-form

Add `src/sase/ace/tui/widgets/repo_creation_form.py` and small supporting widget/model
modules where useful. Render provider-supplied fields in their declared order and
groups:

1. `line` and `text` use the existing single-line/multiline Vim editor primitives and
   mode routing, without importing another module's private editor classes.
2. `bool` uses a space-toggle control and preserves the provider's string
   representation.
3. `choice` uses an inline radio row with canonical values, labels, and visible
   per-choice disabled reasons. Disabled Internal visibility must remain visible.
4. `search` opens a searchable picker and can accept a typed canonical value. The
   existing EnumChoicePickerModal permits declared choices only, so add a
   creation-specific picker or a compatible opt-in extension with unchanged defaults for
   its other callers.
5. Advanced defaults collapsed, shows a field count, and toggles with `a` when focus is
   outside a text editor. All basics and advanced options are available; template
   browsing remains out of scope.

Feed `dataclasses.asdict(form)` and the full raw value map to the Rust facade in lenient
mode on change. Reconcile value updates without recursive change events or resetting
focused editors. Display core errors and availability, and let the host see
`can_create`, message, and remedy argv. Treat owner capability as tri-state; unknown is
not permission denied. Render all provider text literally with control characters
stripped, using the existing cleaning helper.

Verify every field kind, typed search selection/cancellation, Advanced value retention,
template disabling README/gitignore/license/team, all-branches requiring template,
static field restrictions, disabled Internal choice, and restoring enabled controls when
exclusions clear. Use the real core evaluator with fake provider forms. Add focused
widget snapshots for basics and Advanced with greyed fields. No provider network or real
remote mutation belongs in these tests.

## Phase create-configure

Add `new_project_create.py` and focused draft/model/flow helpers. Extend modal
composition and state so an internal test entry can select create Configure while the
normal Source affordances remain deferred.

- Deduplicate namespace owners plus `account`. Default to a still-valid last-used owner,
  then the current project's owner for that provider, then account, then an available
  namespace. Bare git supports an empty namespace without inventing an account.
- Describe the form off-thread for the selected provider/owner. Paint loading first,
  show the capability chip, and retain provider messages/remedy. Failed or None form
  responses must not leave a confirmable draft or apply stale defaults. Late owner/form
  completions must not replace the current owner's edits.
- Default first-use visibility to Private, then the last successful choice for that
  owner if still supported/enabled. README follows the provider's on default. Setup
  defaults to the existing core create-auto recommendation; make-current defaults off.
  Preserve per-owner drafts on navigation and apply defaults only when seeding a fresh
  owner draft.
- Normalize the raw name on every edit with the pure `parse_repo_ref` hook and display
  `raw → canonical` when changed. Preserve the provider's canonical identity, checkout
  path, key/name bases, and web URL. Follow the normalized repository name for the
  default project name until the user explicitly edits the project name.
- Invalidate availability immediately on any target change. Debounce probes by 300 ms
  and run them off-thread. Cancel pending timers/tasks on owner/provider switch, Back,
  and unmount; discard results by generation, mode, owner, and canonical identity.
  Represent checking, available (`not_found`), taken (`found`), and unavailable
  separately. Unknown/errors never become available.
- Observe checkout paths off-thread and feed checkout facts into the existing core
  planner. Do not run full `gather_facts`, provider form fetching, or subprocesses on
  every keystroke. Use the existing local preview adapter with explicitly supplied
  observed facts for name/key/alias uniqueness and collision blockers. Matching
  checkouts follow planner policy; occupied unrelated paths are blocked, never removed
  or reset.
- Thread lenient effective options into the local create plan and show the project
  name/key/accent, enabled state, setup, and make-current controls. Reflect bare-git
  path options rather than silently displaying unrelated default paths.
- Taken remote names offer an explicit Enter/button action to add that canonical ref
  instead. Only activating that offer switches to existing-repo Configure and seeds its
  target. Typing Enter in ordinary text fields must not create or apply anything.
- Make Back, Escape, printable keys, and bracketed paste stage-aware. The existing modal
  paste handler must not send create name/description text into the hidden Source
  filter. Reuse focus/mode conventions and keep draft state through
  Configure/Source/Review navigation.

Test defaults for multiple owners and bare git, invalid/normalized names, project
override retention, name/alias collisions, occupied checkout paths, stale
owner/probe/preview results, permission and auth failure, explicit add-instead, and
unmount. Instrument provider/probe/subprocess seams to prove immediate typing does no
subprocess or network work and that only the latest settled target starts a probe. Add
Configure basics, name-taken, and scrollable 80x24 snapshots.

## Phase create-execution

Make the shared Review pipeline mode-aware and hook the create Configure Review action
to it. Keep blockers disabling confirmation and the existing generation/fingerprint
checks.

- Run the exact `sase project create REF … --check --json` in the session worker, using
  bounded noninteractive subprocess hygiene (`GH_PROMPT_DISABLED=1`,
  `GIT_TERMINAL_PROMPT=0`, `stdin=DEVNULL`). Parse exit-1 blocked documents as plans, as
  the add path already does.
- Use `ConfirmKind.DANGER` semantics/styles for create Review. Show Create, requested
  visibility and repository name in the heading, REMOTE host/URL and creation summary,
  clone/register, project enabled/accent, setup commit/push, init handoff, and the exact
  argv. Show PUBLIC in amber even in compact layouts. State that later failure keeps the
  created repository and retry resumes setup. Preserve the neutral Add view.
- Ensure requested create visibility survives a not-found probe in the document
  envelope. Add requested creation metadata only through a thin envelope/adapter; keep
  shared decisions in Rust.
- Build ApplySubmission operation `project.create` for create documents and
  `project.add` for add. Preserve exact reviewed argv and fingerprint,
  project-onboard/checkout concurrency keys, and reconstruct all `--option`,
  description, and visibility values in the operation request without lossy token
  filtering. Option values equal to transport flag spellings are data, not removable
  argv flags. Verify the request through the existing CLI handler's equality checks,
  rather than weakening them.
- Generalize the durable submission and callback display labels or use a small separate
  registered create producer. Update `_proc_producer_sites_actions.py` and inventory
  assertions to cover both modes. Apply is a durable proc and survives modal dismissal;
  never invoke `ws_create_repo` directly from the TUI.
- Reuse shared NDJSON Progress, project reload, init preview/terminal valve, and Ready.
  Persist provider/owner and per-owner visibility on successful create using
  `record_visibility_selection`, based on the submitted immutable intent. Ensure
  completion after modal dismissal still reaches the success consumer; do not require an
  open widget to persist preferences. No failed operation may overwrite a saved
  visibility choice.
- Remove `--epic-symbol 'sase-1jx.12(record_visibility_selection)'` once that production
  consumer exists. Keep other agents' Justfile exceptions intact.

Test private/public danger Review, unavailable/blocked checks, Back and stale check
completions, exact option round-trip into a real CLI request-validation seam, duplicate
apply, create Progress/init/Ready, and completion with a dismissed modal. Retest
existing add Configure/Review/apply/init cases. Add danger Review goldens for Private
and PUBLIC, including compact visibility rendering.

## Phase recovery-entry

Finish create recovery before exposing Source affordances.

- Make resume operation loading choose the journal's `project.add` or `project.create`
  mode. Validate journal/request identity, provider, ref, and mode. Recover create
  options and project choices from the typed durable request, with recorded argv as a
  validated fallback. Strip proc transport/result flags structurally; never execute an
  archived command wholesale.
- Dispatch stopped/failed RESUME rows to hydration and checked Review, including
  correction of the current unreachable resume branch. Running rows continue to jump to
  the existing proc. On Back from create resume, restore the create Configure draft.
- If complete create intent cannot be recovered, show an explanatory blocker and an
  explicit add-existing alternative. Do not guess Private/README/options and call that
  the original intent. Ownership/adoption remains the existing core plan plus journal
  matching; unrelated existing repos and uncertain outcomes without matching evidence
  stay blocked.
- Render `Repository created; setup incomplete` from plan/checklist/journal evidence,
  retain successful steps, and make `r` re-check the same create intent before
  submitting it. A found remote is a normal resume only with the existing journal
  evidence. Do not silently convert it to add or issue a second creation. Keep
  retry/terminal behavior mode-correct and preserve init-only recovery on Ready.
- Expose a pinned first `✚ Create a new <Provider> repository…` Source row when
  asynchronous provider form discovery confirms support (form not None). Keep it ahead
  of pasted-ref, loading, error, and repository groups across filters and refreshes. A
  denied form remains explanatory and blocks creation; a provider returning None has no
  Create shortcut. Do not probe the synthetic row as a repository.
- Wire Source `n` to that same action when it is available and list-focused; typing `n`
  in editors stays text. Update Source hints, wizard stage hints, and applicable help
  popup descriptions. Update `default_config.yml` only if an actual configurable keymap
  is added or changed.

Test pinned order in populated/loading/empty/filter/error states,
supported/unsupported/denied providers, bare-git configuration, Source Enter/n, and
ordinary add flow. Cover a fake create that records a remote then fails checkout/setup,
retry without a second create, closing/reopening the wizard with recovered
options/fingerprint, stale or missing intent, external remote_exists, and
create-to-Ready including cancelled init. Tests must exercise the actual mode-aware
command/operation seams with fake effects, not just compare model strings.

Add the created-but-incomplete golden and integration goldens needed to finish the
parent acceptance list: Configure basics, Advanced template greying/reasons, name taken,
danger Review Private/PUBLIC, and 80x24. Adapt existing Source goldens for the complete
pinned row, controlling capability results, time, HOME fixtures, hostnames, timezone,
and loaded state.

## Verification and landing

Each worker reads the applicable TUI, screenshot, performance, and lint/test memory
notes through `sase memory read`. Run focused meaningful tests, `just fmt` or
`just fix`, then `sase tool run check` for source changes. Use the monitor skill for
commands that outlive the turn and poll the handoff command itself until it exits. Do
not run `just check-full` without a new explicit instruction.

Rendered changes require targeted `just fix-tui-screenshots -- <selectors>`. Inspect its
retained report and every created/updated/removed golden; a partial update does not
prove skipped images current. Run targeted check-only snapshots twice, including once
with `TZ=UTC`, to establish host independence. Use `sase screenshot` and inspect real
PNG captures to confirm safe Configure/Review keyboard navigation and 80x24 usability;
do not confirm a real create. Existing full-path tests supply deterministic mutation
evidence.

Intermediate new public APIs need real consumers. Where a later phase is the intended
consumer, use a temporary epic-symbol exception keyed to a still-open bead, preferably
`sase-1jx.12`, and document that later consumer. Remove consumed exceptions promptly.
Before closing any child phase, run its `sase bead epic-symbols` and resolve or re-key
leftovers. Before closing the original phase, the child land agent must run
`sase bead epic-symbols sase-1jx.12` and leave no entries keyed to it.

Workers close only their own child phase with a verification note. They never close
`sase-1jx.12`, their child epic, or `sase-1jx`. Discovered follow-ups go in
`PROPOSED FOLLOW-UP:` notes on the worker's assigned phase; do not manually file task
beads. A check failure reproduced identically on the clean base is recorded with
evidence and any existing tracking bead and does not strand a completed phase.

The generated child epic's land agent reviews all phase evidence, runs the required
integration checks, and confirms every original `tui-create` acceptance item. It owns
settling this child epic and then closing `sase-1jx.12` with
`sase bead close sase-1jx.12 --note "<specific checks and UI evidence verified>"`. Do
not close the outer epic `sase-1jx` or any other ancestor plan bead. The original phase
remains open until this full scope is implemented and verified; a plan submission alone
is not completion.
