---
tier: epic
title: Stop losing agent work in linked-repo checkouts
goal: 'Agent work in linked and external repo checkouts survives workspace reuse,
  raw local commits can never end a turn as a silent finalizer success, and monitors
  started inside an opened repo run there.

  '
decisions:
  raw_commit_policy:
    ask: Should the host unwind an agent's raw local commits into the turn's normal
      host commit?
    choices:
      absorb: Host soft-resets eligible raw commits so the normal declaration commits
        and pushes them
      fail: Host never rewrites history; any raw local commit fails the run with restore
        steps
    default: absorb
    why: Unattended %auto runs finish their work instead of stranding it behind a
      failed run.
    answer: absorb
phases:
- id: linked-rescue
  title: Rescue linked and external clones before any eviction
  depends_on: []
  size: medium
  description: 'linked-rescue: rescue_git_repo every sase/repos/linked and external
    clone before launch eviction, re-creation, and corrupt-checkout replacement; tests
    and docs/workspace.md.'
- id: raw-commit-adopt
  title: Make the commit finalizer see raw local commits
  depends_on: []
  size: medium
  description: 'raw-commit-adopt: record baseline HEAD, detect unrecorded local-only
    commits, fail the commit finalizer instead of succeeding, and apply the raw_commit_policy
    branch.'
- id: monitor-linked-cwd
  title: Monitors started inside an opened repo run there
  depends_on: []
  size: small
  description: 'monitor-linked-cwd: default sase monitor start cwd to the caller''s
    directory inside sase/repos/, plus help, docs, and skill source updates.'
proposed_by: bbugyi200.apollo.7c.f0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 08:22:32
status: wip
bead_id: sase-1kh
---

- **PROMPT:** [prompts/202610/protect_linked_repo_work.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/protect_linked_repo_work.md)
- **BEAD:** [sase-1kh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kh/README.md)

# Stop losing agent work in linked-repo checkouts

## Context: why epic bob-cli-78's nav-gestures phase was lost twice

The bob-cli-78.6 release (nav 2.31.0) was finished and verified twice, and lost twice.
It survived only because one agent happened to save patches to `/tmp`. The forensic
trail is in the bob-cli artifacts for the runs `bob-cli-78.6`, `bob-cli-78.land--1`,
`bob-cli-78.land--mon-0`, `bob-cli-78.land--2` and `7c--plan`, all in host workspace
`bob-cli_11`. Three SASE defects combined.

1. **Launch eviction destroys linked checkouts without protection. This is the proximate
   cause of both losses.**
   - Before every launch into a numbered workspace, `prepare_launch_workspace_repos`
     (`src/sase/axe/runner_workspace_protection.py`) runs
     `protect_unpushed_sidecar_commits`.
   - It then calls `clear_workspace_repos` (`src/sase/_linked_repo_workspaces.py`). That
     renames the whole `sase/repos/` tree into `.sase/trash/` and deletes it in the
     background.
   - Sidecar protection skips `linked/` and `external/` on purpose (see
     `workspace_sidecar_repo_roots` in `src/sase/axe/runner_workspace_sidecar_state.py`,
     asserted by `tests/test_bead/test_workspace_sidecar_eviction_beads.py`).
   - So unpushed commits and dirty files in `sase/repos/linked/<name>` vanish with no
     rescue entry and no notification. `docs/workspace.md` ("Preparing a numbered
     workspace … rescues anything still unpublished") reads as if linked clones are
     protected; they are not.
   - **Loss 1.** `bob-cli-78.land--plan` launched into `bob-cli_11` at 10:11Z and
     evicted 78.6's unpushed commit `db8699a`.
   - **Loss 2.** `bob-cli-78.land--2` was SIGTERM-killed at 11:34:07Z. A SIGTERM-killed
     runner releases its workspace with no hold and no notification
     (`src/sase/axe/run_agent_runner_signals.py`). `7c--plan` launched into the same
     workspace at 11:35:24Z and evicted the uncommitted restore.
   - The durable rescue store is `src/sase/workspace_provider/rescue.py`, with
     `rescue_git_repo`. It provides a local-commit bundle, a worktree patch, a manifest
     and an inbox notification, and it is never applied to linked or external clones.
2. **The commit finalizer cannot see work committed outside it.**
   - The `builtin@commit` trigger (`build_context_requirements` in
     `src/sase/finalizers/declaration_store.py`) is driven only by
     `git status --porcelain` dirt (`collect_dirty_state` in
     `src/sase/llm_provider/commit_finalizer_state/__init__.py`).
   - `finalizer_baseline.json` stores dirty-path fingerprints but no HEAD.
   - The 78.6 agent (muse provider) ran raw `git stash`, `git stash pop` and
     `git commit` in its opened bob-plugins checkout. The tree was clean at turn end, so
     the context was `not_triggered` with `repository_evidence: []`, and
     `finalizer_result.json` said `success`.
   - Nothing ever pushed `db8699a`, and the agent's closing claim "committed as
     `db8699a`" went unchallenged.
   - No provider guards against raw git commits: Claude's helper guard covers only
     root-only `sase` commands, and codex and muse run unsandboxed. So the host has to
     be the backstop.
   - The clean-tree success branch in `src/sase/finalizers/commit_execution.py`
     (`raise_if_unpublished_machine_state`, then `success_result`) is where that
     backstop belongs.
3. **`sase monitor start` ignores the caller's directory.**
   - `resolve_cwd` (`src/sase/main/monitor/common.py`) defaults to the agent's recorded
     `workspace_dir` whenever it resolves.
   - From inside `sase/repos/linked/<name>` it always resolves, because project
     inference walks up to the host checkout's `.sase/checkout.json`.
   - `bob-cli-78.land--1` ran `sase monitor start --command 'npm test'` from
     bob-plugins. The monitor ran at the `bob-cli_11` root and failed with
     `npm error Missing script: "test"`. That forced the `land--2` continuation, which
     never ran.

**Out of scope (contributing only).** `land--2` failed three times on a deterministic
Codex HTTP 400: "The 'gpt-6-luna' model is not supported when using Codex with a ChatGPT
account". It then sat in retry backoff until it was killed. Retry classification and
SIGTERM-release notifications are not part of this epic. Once phase `linked-rescue`
lands, a released workspace can no longer lose linked-repo work.

The three phases are independent and touch disjoint files, so they can run in parallel.

## Phase linked-rescue: Rescue linked and external clones before any eviction

**Goal.** No launch, workspace re-creation or corrupt-checkout replacement may destroy
an unpushed commit, stash, or dirty or untracked file in a linked or external clone
without first writing a rescue entry and notifying the user.

**Work:**

1. Add one never-raising helper,
   `rescue_repo_clones_before_eviction(workspace_dir, workspace_num)`, next to
   `rescue_sidecars_before_eviction` in `src/sase/axe/runner_workspace_protection.py`.
   - It enumerates every git checkout under `<workspace>/sase/repos/linked/*` and under
     `<workspace>/sase/repos/external/**`. External clones nest like
     `external/gh/<owner>/<repo>` or `external/projects/…`. Stop descending at a
     directory that contains `.git`.
   - Reuse the enumeration in `src/sase/_repo_inventory_workspaces.py` if it fits.
   - For each checkout it calls `rescue_git_repo(..., include_worktree=True)` with the
     label `linked-<name>` or `external-<relative path>`. The reason names the eviction
     cause, for example "launch-time eviction of linked repo clone".
   - `rescue_git_repo` already returns `None` for a clean clone whose refs are all
     remote-reachable, so ordinary clean clones create no entry and no notification.
2. Call the helper on every path that deletes these clones:
   1. In `prepare_launch_workspace_repos`, before `clear_workspace_repos`, for
      `workspace_num > 1`, right after the existing sidecar protection.
   2. In `recreate_managed_workspace`
      (`src/sase/workspace_provider/_utils_checkout.py`), next to
      `rescue_sidecars_before_eviction`.
   3. In `_remove_corrupt_checkout` and `_rescue_doomed_sidecars` (same file). Those
      only look at `sase/repos/<role>/.git` today, so a corrupt main checkout takes its
      linked clones with it.
3. Rescue only; never publish. Do not remove the linked/external exclusion from
   `workspace_sidecar_repo_roots`: pushing a linked clone at eviction would publish
   unreviewed agent work. Eviction must always proceed, the same contract as sidecar
   eviction protection. A rescue failure warns and never fails the launch.
4. Update the eviction paragraph of `docs/workspace.md`, around "Preparing a numbered
   workspace (`#2` and above) for a launch evicts …". It must say that linked and
   external clones are rescued (bundle of local-only commits plus worktree patch) but
   never published, and where to find and restore them.

**Tests.** Use real git repos under `tmp_path`, following
`tests/workspace_provider/test_rescue.py`, `tests/test_linked_repo_workspaces.py` and
`tests/test_axe_runner_workspace_reclone.py`. Cover:

- a linked clone with an unpushed commit: the bundle contains it and eviction still
  happens;
- a linked clone with only modified and untracked files: `worktree.patch` contains them;
- a clean, fully pushed linked clone: no rescue entry;
- a nested external clone: rescued;
- a rescue that raises or returns `None`: eviction still proceeds;
- the re-creation and corrupt-checkout paths: they rescue linked clones too.

**Verify.** Run `just fmt`, then `sase tool run check`.

## Phase raw-commit-adopt: Make the commit finalizer see raw local commits

**Goal.** A turn whose agent created commits outside the host commit path must never end
with the commit finalizer reporting `success` while those commits exist only locally.

**Work:**

1. **Baseline HEAD.**
   - In `src/sase/llm_provider/commit_finalizer_baseline.py`, `_baseline_record` adds
     `"head"`: the `git rev-parse --verify HEAD` sha, or `null` when HEAD is unborn or
     the repo is unreadable. Do this for both `run_start` and `opened_repo` records.
   - Add an optional `head` field to `FinalizerBaselineRecord` and parse it in
     `_parse_finalizer_baseline_record`.
   - Keep `schema_version: 1`. Older artifacts without `head` parse as before, and
     detection is skipped for them.
2. **Detector.** Add a module next to `commit_finalizer_state`, for example
   `src/sase/llm_provider/commit_finalizer_raw_commits.py`, that returns the unrecorded
   local commits for one repo:
   - the repo kinds are main, sibling (configured linked or opened linked) and opened
     external. Exclude `sdd` stores and the agents prompt archive, which have their own
     commit flows;
   - the candidates are `git rev-list --reverse HEAD --not --remotes <baseline_head>`;
   - minus every sha that the host's own stitch recorded in `commit_results.json` for
     that repo path (read with `load_commit_results`). That keeps an unpushed host
     commit waiting on `commit_unpushed_resume.py` from being treated as raw.
3. **Backstop (both decision branches).**
   - In `src/sase/finalizers/commit_execution.py`, the clean-tree / no-declaration
     success branch (next to `raise_if_unpublished_machine_state`) and the
     accepted-declaration path must both check the detector.
   - If unrecorded local commits remain in any in-scope repo, the instance fails instead
     of succeeding. It uses a non-retryable diagnostic code `unpublished_local_commits`
     that names the repo, path, each sha and subject, and the restore command
     (`git -C <path> push` or `git reset --soft <base>`).
   - The run then fails visibly, its workspace is held, and the user is notified.
   - Keep everything in Python. Do not add a new `trigger` enum value; that wire enum
     lives in sase-core.

> [!decision] raw_commit_policy = absorb
>
> 4. **Absorb before the agent declares.** Add
>    `absorb_raw_local_commits(project_dir, artifact_root)`. It must be idempotent:
>    after it runs, HEAD equals the reset target, so a second call finds nothing.
>    - **Where it runs.** Call it at the start of `_build_live_context`
>      (`src/sase/finalizers/declaration.py`, used by `sase final context` and
>      `sase final submit`), before `collect_dirty_state`. Also check whether prepared
>      host-completion (`sase final prepare`, `_prepare_observe.py`) snapshots HEAD
>      before the context is built. If it does, absorb there first, and test that a
>      prepared intent still binds after an absorption.
>    - **Eligibility.** A repo qualifies only when all of these hold:
>      - the unrecorded commits form a linear chain with no merges;
>      - HEAD is attached to a branch;
>      - no rebase, merge, cherry-pick, am, revert or bisect is in progress;
>      - none of the commits' paths overlaps a baseline-protected dirty path.
>    - **Action.** Pin `refs/sase/recovery/raw-commits/<run_id>` at the current HEAD
>      (reuse `retain_current_head_recovery_ref` in
>      `src/sase/axe/runner_workspace_sidecar_rescue.py` if it fits). Then run
>      `git reset --soft <parent of the oldest unrecorded commit>`.
>    - **Effect.** The changes become staged dirt, so the ordinary obligation,
>      declaration and `sase stitch create` path commits and pushes them with SASE
>      trailers.
>    - **Reporting.** Add a details line to the context output naming each absorbed sha
>      and subject, so the agent and the run's finalizer evidence both show it.
>    - **Ineligible repos and execution time.** Ineligible repos are left untouched and
>      fall through to the backstop. Never absorb at execution time; there, only the
>      backstop applies.

> [!decision] raw_commit_policy = fail
>
> 4. **No history rewriting.** The backstop is the whole fix. In addition, when
>    `sase final context` or `sase final submit` detects unrecorded local commits, it
>    prints an agent-facing warning. The warning names them and says to run
>    `git reset --soft <base>` and resubmit in the same turn, so a live agent can repair
>    before the backstop fails the run.

5. Update `docs/commit_workflows.md`, the finalizer flow list (around step 5, "Preserve
   protected pre-existing dirt …"). Document the baseline `head`, the unrecorded-commit
   check, and the selected `raw_commit_policy` behavior.

**Tests.** Extend `tests/llm_provider/test_commit_finalizer_baseline.py` and the
finalizer suites (`tests/test_finalizer_declaration_channel_context.py`,
`tests/test_finalizers_commit_reconciliation.py`, and
`tests/test_finalizers_live_e2e.py` where it is practical). Cover:

- the baseline records `head`, and a legacy baseline without `head` still parses;
- a raw commit in an opened linked repo no longer yields `success`: under `absorb`
  (eligible case) it is staged and committed through the declaration; under `fail` it
  fails with `unpublished_local_commits`;
- a host stitch commit with `pushed: false` is not treated as raw, and
  `commit_unpushed_resume` still works;
- remote-reachable fast-forwards since the baseline are ignored;
- SDD sidecar commits are ignored;
- for `absorb`: an ineligible chain (a merge, or a path overlapping protected dirt) is
  not rewritten and reaches the backstop.

**Verify.** Run `just fmt`, then `sase tool run check`.

## Phase monitor-linked-cwd: Monitors started inside an opened repo run there

**Work:**

1. Change `resolve_cwd` in `src/sase/main/monitor/common.py`. When `--cwd` is absent and
   the agent workspace resolves, but the caller's `Path.cwd()` is inside
   `<workspace>/sase/repos/` (a linked, external or sidecar clone), return the caller's
   resolved cwd. Every other case keeps today's behavior. Nested-path claims already
   work (`src/sase/monitor/start_lane.py`).
2. Update the `-C/--cwd` help in `src/sase/main/parser_monitor.py` to describe the new
   default: the current directory inside a repo opened with `sase repo open`, else the
   agent's workspace, else `$PWD`.
3. Update the `--cwd` text in `docs/monitors.md` and in the skill source
   `src/sase/macros/skills/sase_monitor.md` ("Useful Flags").
   - Preview with `sase skill init --diff` only.
   - Do not deploy skills (`sase skill init --force` or `chezmoi apply`): they deploy
     from a landed revision after the epic lands.
4. Make sure `sase monitor start` prints the directory the command will run in, if it
   does not already.

**Tests.** Add unit tests next to the existing monitor start/cwd tests (find them with
`rg resolve_cwd tests`). Cover:

- the caller is inside `sase/repos/linked/<name>`: the monitor uses the caller's cwd;
- the caller is at the workspace root or in a main-checkout subdirectory: the monitor
  uses the workspace root;
- an explicit `--cwd` always wins.

**Verify.** Run `just fmt`, then `sase tool run check`.

## Landing

1. Confirm all three phases are on `origin/master`, and that `sase tool run check`
   passes on the combined tree.
2. Record in the epic close note:
   - the selected `raw_commit_policy`;
   - that the `sase_monitor` skill must be redeployed from the landed revision
     (`sase skill init --force`).
3. No memory note changes. `decisions:host-owned-completion` ("an agent never creates
   commits") stays true; this epic adds the host enforcement it assumed.
