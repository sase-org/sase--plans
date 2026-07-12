---
create_time: 2026-07-10 19:32:49
status: done
prompt: 202607/prompts/init_all_active_projects.md
tier: tale
---
# Plan: Add `sase init --all` for Every Active Main Project

## Goal

Add `-a, --all` to the bare `sase init` coordinator so one invocation can check or initialize every known active main
SASE project. The command should use the same Rust-backed project inventory boundary as `sase vcs log --all`, while
applying the narrower lifecycle scope requested here: active main projects only, never inactive projects, sibling
bookkeeping records, or the system-managed `home` project.

The batch must be useful both interactively and in automation:

- `sase init --all --check` checks every eligible project without writing and exits nonzero if any project has drift or
  cannot be checked.
- `sase init --all` visits every eligible project and preserves the existing per-initializer prompts in an interactive
  terminal; in a non-interactive shell it remains read-only unless `--yes` is supplied.
- `sase init --all --yes` attempts every needed initializer for every eligible project without prompting.
- A bad project record, missing primary workspace, planning exception, or initializer failure is isolated to that
  project. Later projects are still attempted, and the aggregate command exits nonzero after reporting all failures.

## Current Behavior and Reusable Design

Bare `sase init` currently plans and runs memory, SDD, and skills for the current directory. Its coordinator already
encapsulates the important single-project semantics: read-only planning, TTY-aware prompting, `--check`, `--yes`,
initializer ordering, failure reporting, and deferred chezmoi deployment.

`sase vcs log --all` provides the inventory precedent. It calls
`list_project_records(sase_projects_dir(), ..., include_home=False)`, validates each record's ProjectSpec and primary
workspace, sorts projects deterministically, and turns unusable records into warnings rather than aborting discovery.
For initialization, the existing project lifecycle facade already exposes exactly the required core behavior, so no new
`sase-core` API or linked-repository change is needed.

The important scope difference is intentional:

- VCS log asks for active and inactive records because historical repositories belong in a global timeline.
- Init will ask only for `active`, exclude system-managed records defensively, and therefore omit both inactive and
  sibling projects. Each eligible record's recorded primary `WORKSPACE_DIR` becomes the working directory for its normal
  single-project onboarding run.

## CLI Contract

1. Add `-a, --all` to the bare `sase init` parser with help text that says it attempts every known active main SASE
   project. Keep the option list alphabetized and update parser/help tests in accordance with the CLI rules.
2. Allow `--all` to combine with the existing coordinator modes:
   - `--check` selects a global read-only drift check.
   - `--yes` selects an unattended global apply.
3. Reject combinations whose meaning would otherwise be surprising or silently ignored:
   - `--all` with an explicit compatibility subcommand (`memory`, `sdd`, or `skills`), because this feature batches the
     bare coordinator rather than changing the scoped command surfaces.
   - `--all` with `--enable-project-memory`, because `-M` is documented as an explicit opt-in for the current project;
     silently opting every project into managed memory would be a materially broader and riskier operation.
4. Keep ordinary `sase init`, `sase init --check`, `sase init --yes`, and all explicit aliases behaviorally unchanged.
   `sase validate` also remains current-project scoped unless it explicitly adopts `--all` in a future change.

## Implementation Design

### 1. Add an all-project inventory layer for initialization

Extend the initialization project-scope module (or a small adjacent module if that keeps responsibilities clearer) with
a typed resolver for active main project targets:

- Load `list_project_records(sase_projects_dir(), "active", include_home=False)` through the existing lifecycle facade.
- Exclude `home` and any `system_managed` record defensively.
- Sort deterministically by effective/display project name and stable directory key, matching the user-facing ordering
  style of global VCS resolution.
- Validate the ProjectSpec path and recorded primary workspace before execution. Preserve record/parse warnings and
  represent unavailable paths as per-project failures rather than dropping them silently.
- Return enough identity for clear output: stable project key, effective display name, ProjectSpec path, and primary
  workspace path.

This remains Python orchestration over the existing Rust-owned lifecycle inventory; it must not reparse ProjectSpecs or
reimplement lifecycle rules locally.

### 2. Add a batch coordinator around the existing single-project coordinator

Add a dedicated all-project entry point in the init onboarding layer and dispatch to it only when bare `sase init`
receives `args.all`.

For each resolved project, the batch coordinator will:

- Print a compact project heading containing the effective name (and directory key when they differ), so the existing
  initialization rows and any warnings have unambiguous ownership.
- Enter the recorded primary workspace with an exception-safe working-directory context, invoke the normal
  single-project onboarding coordinator, and always restore the caller's original directory.
- Preserve the existing per-project memory → SDD → skills plan/run order and TTY prompt behavior rather than duplicating
  initializer logic.
- Catch ordinary per-project planning/execution failures at the batch boundary, report them, and continue. User
  cancellation/interrupt should still abort cleanly rather than being disguised as a project failure.
- Accumulate checked, initialized/current, drifted, unavailable, and failed outcomes, render a final concise batch
  summary, and return `0` only when every eligible project completed successfully (and, in check mode, all were
  current). Inventory failure or an empty eligible inventory should produce a clear message and a nonzero result.

If direct reuse exposes control-flow that currently assumes immediate return on the first failed initializer, factor a
small single-project result-returning helper out of `run_init_onboarding`; keep `run_init_onboarding` as the compatible
single-project wrapper. Do not duplicate plan rendering, prompting, or apply argument construction in the batch path.

### 3. Preserve safe deployment and project context semantics

Audit the coordinator boundary while factoring it so batch execution does not leak one project's mutable argument
markers, current directory, or deferred chezmoi paths into another project:

- Give each project a fresh/shallow-copied argument namespace before invoking onboarding, especially for internal memory
  opt-in/config state fields.
- Keep project-local commits and SDD materialization associated with that project's primary checkout.
- Ensure chezmoi writes retain the existing deferred-deploy guarantees. Prefer one well-defined aggregate deploy after
  all selected initializers if the existing context can safely span projects; otherwise preserve isolated per-project
  deploys. In either case, deduplicate shared home paths and test partial-failure behavior so successful writes are not
  lost or accidentally attributed to a later project.
- Do not materialize missing workspaces merely to satisfy `--all`; an unavailable recorded workspace is reported and the
  batch continues.

## Output and Failure Semantics

The human-readable output should make a large batch scannable without replacing the existing rich per-project plan:

```text
Project: Alpha
SASE initialization check
...

Project: Beta
SASE initialization check
...

Initialization summary: 2 checked, 1 current, 1 needs attention, 1 unavailable
```

Exact wording can follow the established Rich styles, but tests should lock down these properties:

- deterministic project order;
- explicit project ownership for warnings and failures;
- continued processing after one unavailable or failed project;
- one aggregate status whose exit code reflects every project;
- no traceback for expected inventory/path/initializer failures;
- restoration of the original working directory on success, failure, and cancellation.

## Tests

Add focused tests near the existing init onboarding parser, flow, apply, and entrypoint suites:

1. **Parser/help contract**
   - `init -a` and `init --all` set the new flag; default is false.
   - `--all --check` and `--all --yes` parse successfully.
   - Help lists `-a, --all` with active-main-project wording and keeps options ordered.
   - `--all` with `-M` or an explicit init alias is rejected with a useful usage error.
2. **Inventory scope**
   - Active records are selected from the Rust facade; inactive, sibling, `home`, and system-managed records are not.
   - Display-name sorting is deterministic.
   - Missing ProjectSpecs/workspaces and record warnings are surfaced without hiding healthy projects.
   - Inventory lookup works from outside any project workspace, like `vcs log --all`.
3. **Batch check/apply behavior**
   - Every healthy project runs from its recorded primary workspace, then the original cwd is restored.
   - `--check` writes nothing, checks all projects, and aggregates drift into exit status `1`.
   - Interactive mode preserves per-initializer prompts; non-TTY mode without `--yes` stays read-only.
   - `--yes` applies every changed project in deterministic order.
   - A failed/unavailable project does not prevent later projects from being attempted, but makes the final status
     nonzero.
   - Keyboard interruption aborts cleanly and restores cwd.
4. **Regression coverage**
   - Existing single-project init flow, initializer order, `-M`, explicit aliases, and chezmoi deploy tests remain
     unchanged and passing.
   - Add a batch deploy test covering shared home/chezmoi outputs and one project-local failure.

## Documentation

Update the user-facing command references together with the implementation:

- `docs/init.md`: add examples for `--all`, `--all --check`, and `--all --yes`; explain active-main scope, invocation
  from outside a workspace, continuation on per-project failures, exit semantics, and the rejection of `--all -M`.
- `docs/configuration.md`: add the alphabetized `-a, --all` flag row and summarize batch behavior.
- `docs/cli.md` and the README common-command block: expose the new global initialization/check workflow succinctly.

## Verification

1. Run the focused init parser/onboarding/inventory tests while developing.
2. Run `just install` before repository checks, as required for an ephemeral workspace.
3. Run `just check` before handoff.
4. Manually inspect `sase init --help` and exercise read-only `sase init --all --check` from both inside a project and a
   non-project directory, confirming identical inventory scope and a clear aggregate summary.

## Non-Goals

- No initialization of inactive or sibling projects.
- No automatic creation/materialization of missing primary workspaces.
- No new bulk mode for `sase memory init`, `sase sdd init`, `sase skill init`, or their compatibility aliases.
- No bulk project-memory opt-in; projects continue to require their own `memory.enabled: true` configuration.
- No lifecycle parsing or project discovery reimplementation outside `sase-core`.
