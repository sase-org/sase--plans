---
tier: tale
title: Make TUI project init create sidecars through the terminal path
goal:
  Initializing a project from the TUI either creates its sidecar repositories
  interactively or reports clearly that it did not.
size: medium
decisions:
  y_key:
    ask:
      When a plan has terminal-only steps, should `y` still run the non-interactive
      `--yes` apply?
    choices:
      keep:
        "`y` runs `--yes` and skips terminal steps; terminal is the focused primary;
        skips need attention"
      terminal:
        "`y` hands projects needing a terminal to interactive `sase init`, then plans
        the rest again"
    default: keep
    why:
      Keeps the one-`--yes`-proc contract and avoids suspending the TUI for mixed `I`
      scopes
    answer: keep
proposed_by: bbugyi200.athena.19b
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 06:49:41
status: wip
---

# Fix: initializing a new project from the TUI never creates its sidecar repositories

## Diagnosis (confirmed from the `research.5t.*` incident)

The suspicion is right, and the damage is larger than "some". Onboarding
`bbugyi200/love` from the TUI created **none** of the five sidecar remotes: `plans`,
`beads`, `research`, `agents`, and `attachments-private`.

Timeline (local time, 2026-11-07):

- 06:30:18–06:30:25: the New Project wizard ran
  `sase project add bbugyi200/love --make-current --name love --yes --json`. Its plan
  marked the `init` step `requires_tty: true, handoff: true` ("init preview; sidecar
  creation asks in a terminal").
- 06:30:43–06:30:46: the wizard's init handoff ran
  `sase init -p gh_bbugyi200__love --yes` as a TUI session worker. This produced the
  commits `chore: run sase init memory` and `chore: initialize SASE repositories`; the
  second wrote `repos.sidecar` into `sase/sase.yml` plus the `.gitignore` rule. The
  toast read "Initialized 1 — see Procs".
- 06:32–06:35: all five `research.5t.*` agents failed while preparing.
  `auto_connect_sdd_store` raised
  `SddMaterializationError: GitHub sidecar repository bbugyi200/love--plans does not exist; run sase repo init …`.
- 06:37:15–06:37:33: the GitHub sidecar repos were first created by a later manual
  interactive `sase repo init`.

Root cause: two defects combine, and either one alone would have exposed the problem.

1. **The TUI never offers its terminal path for terminal-only work that is not
   blocked.** `InitProjectPlan.requires_tty`
   (`src/sase/ace/tui/modals/projects_pane_init_payload.py`) counts a planner only when
   it has `requires_tty` **and** `blockers`. The repo planner marks missing sidecars
   with `requires_tty=True` (`plan_sidecar_actions` in
   `src/sase/main/_repo_init_sidecars.py`) but no blocker. As a result,
   `has_tty_blocked_projects()` is False, the `InitPlanModal` hides `t` / "Run in
   terminal", and the only apply path is `sase init -p KEY --yes`, run with
   `stdin=subprocess.DEVNULL` (`session_proc_reporter._stream_subprocess`). By design,
   `--yes` and non-interactive stdin can never authorize creating a remote
   (`docs/reference/agents/initialization.md`).
2. **Skipped sidecar creation is reported as success.** With `onboarding=True` and a
   non-TTY stdin, `run_configured_sidecars` prints a
   `warning: … is missing; run sase repo init interactively` line for each sidecar and
   then `return 0`. `run_repo_init` commits the wiring, and `run_changed_plans`
   (`src/sase/main/_init_onboarding_apply.py`) returns
   `InitRunResult(0, "initialized")`. So the batch summary says
   `1 checked, 1 initialized`. The TUI's `_apply_payload_from_output` shows "Initialized
   1", and the wizard calls `mark_initialized()`. Also, a `needs attention` count with
   exit 0 would currently be classified as `current`, which also counts as initialized.

Not the cause: `sase project add`, provider policy resolution (`separate_repo` resolved
fine), and sidecar spec resolution. The non-interactive deferral is intentional and
stays: `--yes` must never create remotes.

## Goal

Initializing from the TUI (New Project wizard handoff and Projects-tab `i` / `I`) must:

- offer the interactive terminal path whenever the plan contains terminal-only work,
  such as sidecar repository creation, and make that path the obvious one;
- never report a project as initialized while required terminal-only work was skipped.

Keep the safety contract: only an interactive `y`/`yes` answer at the per-sidecar prompt
authorizes creating a remote. Do not add a CLI flag that authorizes creation.

## Changes

### 1. CLI: report skipped sidecar creation as needs attention

`src/sase/main/_repo_init_sidecars.py` → `run_configured_sidecars`:

- In the `missing and onboarding and not stdin.isatty()` branch, record the skipped
  sidecars on the `args` namespace, for example
  `args._init_deferred = (*existing, "<provider> sidecar repository <repo>", …)`. Keep
  the existing per-sidecar warnings and the `return 0`: the wiring is still written and
  committed, which is the documented non-blocking behavior.
- Count only roles that the planner treats as drift. `attachments-private` is advisory
  in `plan_sidecar_actions` (a warning, no action), so leave it out. Otherwise every
  later `--yes` run would report needs attention for a store the user never wanted. Use
  one shared predicate or constant for both the planner and the runner so they cannot
  drift apart. `agents` and public `attachments` do count, because `--check` already
  reports them as drift.

`src/sase/main/_init_onboarding_apply.py` → `run_changed_plans`:

- Keep a reference to the namespace that `_apply_args(...)` builds and pass that same
  object to `spec.run`. After a zero exit, if
  `getattr(apply_args, "_init_deferred", ())` is non-empty, print one concise line, for
  example `init repo: sidecar creation needs a terminal: <names>; run `sase init -p
  <key>`or`sase repo init` interactively`. Use whatever project reference is available,
  falling back to `sase repo init`. Then mark the run as needing attention, the same way
  a declined step already does: `InitRunResult(0, "needs_attention")`.
- This automatically gives `-p` / `--all` batches the existing aggregate semantics:
  `Initialization summary: 1 checked, 1 needs attention` with exit 1. A bare
  single-project `sase init --yes` keeps exit 0. Check that post-commit and automation
  callers only rely on the bare form. A grep found no automated `--all --yes` callers in
  `src/`.

### 2. TUI: offer the terminal path for terminal-only work that is not blocked

`src/sase/ace/tui/modals/projects_pane_init_payload.py`:

- Replace the blocker-only predicate with a "needs a terminal" predicate: a project
  needs a terminal when any planner has `requires_tty` and either `blockers` or
  `has_changes`. Rename the property and helpers to match the new meaning (for example
  `needs_terminal`, `terminal_projects()`, `has_terminal_projects()`) and update every
  caller: `init_plan_modal.py`, `projects_pane_init_flow._run_init_in_terminal`, and any
  others found with grep. A planner with `requires_tty` and no changes or blockers must
  still not count.
- Note that the machine planner (`requires_tty=offer_enrollment`) now counts too. That
  is correct, because enrollment also cannot run under `--yes`. Cover it in a test.

`src/sase/ace/tui/modals/init_plan_modal.py` and `init_plan_modal_rendering.py`:

- Show the `t` row/button whenever `has_terminal_projects(payload)` is true.
- For a planner that has `requires_tty` and changes but no blockers, add a yellow note
  under it, stating that the step needs a terminal, that `y` skips it, and that `t` runs
  it interactively. For the repo planner, name sidecar repository creation explicitly.
- When terminal work is pending, make `Run in terminal (t)` the primary-variant,
  initially focused button. Make the confirm label state that it skips terminal steps
  (for example `Initialize love · skip terminal steps (y)`). Update
  `init_plan_border_subtitle` to match.

> [!decision] y_key = terminal In `ProjectsInitFlowController._on_init_plan_decision`,
> treat `apply` as `terminal` whenever `has_terminal_projects(payload)` is true. Have
> `ProjectsPaneInitHost.recheck_after_terminal` return True so the remaining projects
> get a fresh preview after the terminal run. The confirm label then reads "Continue in
> terminal (y)", and the docs say that confirm hands projects needing a terminal to
> interactive `sase init` before applying the rest.

`src/sase/ace/tui/modals/projects_pane_init_flow.py` → `_apply_payload_from_output` /
`_format_apply_toast`:

- Add an `attention` apply kind with `warning` severity. Use it when the summary has
  `needs attention` > 0 and no `failed`, `unavailable`, `cancelled`, or
  `deployment failed` counts, whatever the return code. Do this check before the
  `current` and `success` branches, so a needs-attention summary can never count as
  initialized. Mixed batches keep the existing `partial` and `failure` handling.
- The toast should point to the fix, for example
  `1 needs attention — run init again and press t to finish in a terminal · see Procs`.
- `initialized` stays limited to `{"success", "current"}`. The New Project wizard then
  lands on Ready with `needs_init=True`, where `i` offers init again. It does not call
  `mark_initialized()`.

### 3. Docs

Read the `docs_writing.md` memory first.

- `docs/reference/agents/initialization.md`, paragraph "When non-interactive bare
  `sase init --yes` discovers a missing sidecar remote…": add that the project is
  reported as needs attention (bare form exits 0; `-p` / `--all` batches exit 1 like any
  needs-attention project). Also add that `attachments-private` stays advisory.
- `docs/reference/tui/projects-tab.md`: in the "Initialize from the Projects tab"
  section, change "TTY-only blockers" to cover terminal-only steps such as sidecar
  repository creation. Document that `y` skips those steps, that `t` is the primary
  action when they are pending, and the needs-attention toast. In wizard step 4
  (Progress) and step 5 (Ready), state that sidecar creation finishes in a terminal and
  that an incomplete init leaves `i` available on Ready.
- If the `?` help modal documents the init preview keys, update it too (per the `ace.md`
  memory's help-popup rule).

## Tests

- `tests/main/` (repo init / onboarding):
  - Under `run_init_onboarding` with `stdin=StringIO()`, a managed project with missing
    `plans` / `beads` / `research` / `agents` sidecars records the skipped roles and
    returns status `needs_attention` with exit 0. A missing-only `attachments-private`
    still returns `initialized` (or `current`).
  - Under `run_init_onboarding_all` (`-p KEY --yes`), the same project prints
    `Initialization summary: 1 checked, 1 needs attention` and exits 1, and creation is
    never authorized (no `creation_authorized=True`).
  - Update the existing assertions in `tests/main/test_repo_init_handler.py:91` and
    `tests/main/_test_repo_init_handler_creation_confirmation.py:433-447` that expect
    the old `initialized` / exit behavior. Keep `== 0` where the bare form is used.
- `tests/ace/tui/modals/test_projects_pane_init_payload.py`: the new predicate is true
  for `requires_tty` with `has_changes` and no blockers (repo planner shape), true for
  blocked TTY planners (unchanged), true for the machine-offer shape, and false for
  `requires_tty` with no changes or blockers.
- `tests/ace/tui/modals/test_init_plan_modal.py`: with pending terminal work, the
  terminal button is rendered, is primary, and has initial focus; the note and the
  relabeled confirm button appear; `t` dismisses with `terminal`. Without terminal work,
  the modal is unchanged.
- `tests/ace/tui/test_projects_pane_init_flow_*.py`:
  - `_apply_payload_from_output("Initialization summary: 1 checked, 1 needs attention", 1)`
    gives kind `attention` with warning severity, and the host outcome is
    `initialized=False`. The same summary with exit 0 is also `attention`, not
    `current`.
  - A love-shaped regression: a check payload whose repo planner has
    `requires_tty=True`, changes, and no blockers offers the terminal path. Under the
    `keep` default, an apply whose output reports needs attention leaves the New Project
    wizard on Ready with `needs_init=True`.
  - If `y_key = terminal` is chosen, test that `apply` routes to the terminal and the
    pane rechecks.

## Verification

Read the `lint_and_test.md` and `tui.md` memories, then run `just check` inside
`sase tool run` as those notes require. Don't run `check-full` unless instructed. If the
modal's appearance changes visual snapshots, follow `tui_screenshot.md`.

## Out of scope / follow-ups

- The `love` project itself is already repaired: all five sidecars exist and
  `.sase/sdd-store.json` reports `discovery: found`. The failed `research.5t.*` agents
  can be relaunched as they are.
- Optional follow-up, not part of this tale: have agent launch suggest
  `sase init -p <key>` from the TUI when `SddMaterializationError` names a missing
  sidecar.
