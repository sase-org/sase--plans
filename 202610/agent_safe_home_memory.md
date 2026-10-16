---
tier: tale
title: Stop agents from poisoning each other through the shared chezmoi home
goal:
  In-agent memory init never mutates shared home memory, and SASE tool commits in shared
  siblings never fail other agents' completion.
size: medium
decisions:
  tool_commit_exemption:
    ask:
      Should the raw-commit backstop ignore SASE tool commits that carry no agent
      trailer?
    choices:
      exempt: Human SASE tool commits in shared siblings stop failing running agents
      count:
        Every human home-memory refresh during agent runs keeps failing running agents
    default: exempt
    why:
      raw git commit never writes SASE_TYPE, and change 3 makes in-agent tool commits
      name their agent
    answer: exempt
  check_drift_scope:
    ask:
      Should --check report home-memory drift as a deferred warning for humans too, not
      only in agents?
    choices:
      always:
        Home drift warns everywhere like init skills --check; stale home deploys stop
        failing validate
      agents_only:
        Only agent and monitor runs demote home drift; humans keep the hard check
        failure
    default: always
    why: matches the existing init skills --check precedent for home-deploy drift
    answer: always
proposed_by: bbugyi200.athena.10u
decided_by: reviewer
decided_via: tui
create_time: 2026-10-16 17:53:46
status: wip
---

# Stop agents from poisoning each other through the shared chezmoi home

## Context: why `sase-1li.3` "failed"

`sase-1li.3`'s work is fine. Its stitch landed on master as `4b000ca2c6`
(`docs(beads): document closed-bead launch attach in prompts and memory`), and bead
`sase-1li.3` is closed. The follow-up session member `sase-1li.3--1` still ended FAILED.
After the push, the commit finalizer's raw-commit backstop
(`raise_if_unpublished_local_commits` in `src/sase/finalizers/commit_validation.py`)
found three local-only commits in `sibling:chezmoi` (`~/.local/share/chezmoi`) newer
than the session baseline `f870d35f`:

| Commit     | Time (UTC) | Created by            | Command                                                           |
| ---------- | ---------- | --------------------- | ----------------------------------------------------------------- |
| `c9b92e7c` | 21:11:49   | `sase-1lh.7.2--plan`  | `sase memory init`, then `.venv/bin/sase memory init --no-commit` |
| `184432b8` | 21:13:11   | `sase-1le.20.1.16--1` | `sase init memory` (older global `sase`)                          |
| `a399db6d` | 21:20:42   | `sase-1le.20.1.16--1` | `.venv/bin/sase init memory --no-commit`                          |

All three commits are `chore: initialize sase memory` with only `SASE_TYPE=memory` and
no `SASE_AGENT`. They ping-pong the same seven home files (+70/−58, +58/−70, +70/−58).
The globally installed `sase` and a workspace `.venv/bin/sase` render the home `sase.md`
template differently. Each run also called `chezmoi apply --force`, which rewrote the
live `~/AGENTS.md`, `~/CLAUDE.md`, and the other home files that every agent on the
machine loads.

Causal chain:

1. **The validate gate pushes agents toward a global mutation.** `just check` →
   `just validate` → `sase validate` runs `init memory --check`. That check fails on
   chezmoi home drift caused by workspace-vs-installed `sase` skew. No agent's change
   caused it, and no agent can fix it from a workspace. `sase-1le.20.1.16--1` ran memory
   init specifically to clear it, and `sase-1li.3--1` hit the same failure.
   `init skills --check` already reports the same class of drift as a warning: "redeploy
   is deferred until land. Rerun `sase init skills` after landing."
2. **`sase memory init` always deploys the home root.** It writes the home root, commits
   it in the chezmoi primary, and runs `chezmoi apply --force`. The commit is never
   pushed (`pull_push=False`), so it stays local-only. `--no-commit` only governs the
   project repo. The documented `/sase_memory_write` workflow ("Run `sase memory init`
   to regenerate `AGENTS.md`…") therefore mutates the user's global home from any
   workspace, using whatever `sase` version the agent happened to invoke.
3. **The tool commit carries no agent provenance.**
   `ChezmoiDeployBehavior.include_runtime_commit_tags` defaults to `False`, and only
   skills deploys set it. Every concurrently running agent baselines `sibling:chezmoi`,
   so each one counts the commit as its own unattributed raw commit and fails closed.
4. **The detector also misreads real agent trailers.** `_is_same_agent` in
   `src/sase/llm_provider/commit_finalizer_raw_commits.py` compares the trailer label
   (`bbugyi200.athena.<name>`, possibly a linked `[label][n]` value) against the bare
   local name. Verified: `_is_foreign_agent_commit(own_msg, "sase-1li.3--1")` returns
   `True` for the agent's own commit. The discarded-work guard's
   `_agent_provenance_matches` (`commit_finalizer_git_progress.py`) matches the same
   message correctly. The module docstring's promise that "commits trailered with this
   agent's own name count too" is false today.

Since the raw-commit backstop landed (epic `sase-1kh`, phase `sase-1kh.2`), the old
"low-reach" papercut `sase-l0` has become a cross-agent failure. One agent following the
memory-write workflow can fail every other running agent's completion.

Still live when this plan was written: the three commits remain unpushed, so the running
`sase-1lh.7.2--1` and `sase-1le.20.1.16--1` will probably fail the same way. The human
unblock is `git -C ~/.local/share/chezmoi push` (net content = master's home template)
or
`git -C ~/.local/share/chezmoi reset --soft f870d35f8afd79aa4a6af0580d9cf00e572b218a`.
That cleanup is not part of this plan; the implementing agent must not touch the chezmoi
primary.

## Goals

- An agent or monitor running `sase memory init` (or bare `sase init`) never writes,
  commits, or applies the shared home memory root, unless a human approved it through a
  gate or an explicit bypass. The project-root refresh keeps working, so the
  memory-write workflow is unchanged for project memory.
- `init memory --check` stops failing agent gates on home drift that only a post-land
  human refresh can fix.
- Any SASE-authored chezmoi commit made inside an agent names that agent, so concurrent
  agents' finalizers treat it as foreign.
- The raw-commit backstop attributes commits correctly. An agent's own real-format
  trailers count. Other agents' commits do not. Under decision
  `tool_commit_exemption = exempt`, SASE tool commits with no agent trailer do not count
  either.

## Changes

### 1. Home memory is human-only inside agents (`src/sase/main/init_memory_handler.py`)

- Add one shared, public agent-context helper. Factor `_agent_present` out of
  `src/sase/project_onboarding/human_only.py`, for example as
  `agent_marker_present(env)` next to `discover_agent_identity` in
  `src/sase/agent/identity.py`. It treats `SASE_AGENT`, `SASE_AGENT_NAME`,
  `SASE_MONITOR_ID`, or a discovered identity as agent context. `detect_actor` keeps its
  exact behavior and reuses the helper.
- Home deploys are permitted in agent context only when `SASE_GATE_COMMAND=1` (an
  approved `/sase_gate` command) or when `SASE_HOME_MEMORY_DEPLOY_BYPASS` holds a
  non-empty reason. Print the bypass reason to stderr once, mirroring
  `SASE_PROJECT_CREATE_BYPASS` and `SASE_GLOBAL_INSTALL_BYPASS`.
- In `run_init_memory`, when agent context applies and no permission holds:
  - Still load and validate `home_entries` read-only. Project rendering, the
    `unreferenced` validation, and config errors must behave exactly as today. The
    project root may reference home roots through `chezmoi_home_roots`, so keep passing
    them.
  - Do not write the home root. Skip the home `_initialize_memory_root` write; use the
    read-only `_plan_memory_root` result wherever validation needs it.
  - Do not call `defer_chezmoi_paths` or `_deploy_to_chezmoi`.
  - Print one notice, for example:
    `init memory: home memory refresh deferred until land (agent run); rerun \`sase
    memory init\` outside an agent after landing — home target: <path>`.
  - Exit with the project result's code. The skip itself is never an error.
  - The skip applies to both the chezmoi and the non-chezmoi home root, because
    non-chezmoi mode writes `~` directly.
- The bare `sase init` memory step goes through `run_init_memory`, so it inherits the
  skip. Verify that a deferred bare-init deploy in agent context gets no memory paths.
- Leave `--no-commit` semantics alone (`sase-l0`). The TUI Memory panel's no-commit
  publish branch (`src/sase/ace/tui/modals/memory_panel_publish.py`) relies on today's
  behavior for home scopes.

### 2. `--check` demotes home drift (`plan_init_memory`)

- In check mode, home-root write actions no longer count as "needs attention". Replace
  them with one warning, worded like `deferred_skill_deploy_warnings` in
  `src/sase/main/_init_skills_interaction.py`:
  `N home memory files out of sync with rendered sources; refresh is deferred until land. Rerun \`sase
  memory init\` after landing.`
- Decision `check_drift_scope = always` applies this in every context, matching skills.
  `agents_only` gates it on the change 1 helper (agent context without permission).
- Project-root drift, config errors, required-plugin blockers, validation errors, and
  the `tracking_blockers_for_root` blockers still fail exactly as today.
- Non-check mode, including the `--diff` preview, is unchanged.
- Confirm that `sase validate` (`src/sase/main/validate_handler.py`, check
  `init memory --check`) now passes with home drift and surfaces the warning the way it
  already surfaces the skills warning.

### 3. Chezmoi deploy commits carry agent provenance (`src/sase/main/_init_chezmoi_deploy.py`)

- Flip the defaults of `ChezmoiDeployBehavior.include_runtime_commit_tags` and
  `_DeferredChezmoiDeploy.include_runtime_commit_tags` to `True`. Every SASE-authored
  chezmoi commit made inside an agent then gets `SASE_AGENT=` provenance. That covers
  memory init, owner-identity config init, completion deploy, and the deferred bare
  `sase init` deploy. The skills path already passes `True`.
- Outside agents, `_resolve_runtime_commit_tags()` returns nothing, so human commits are
  unchanged apart from stale `AGENT`/`MACHINE` keys being stripped.
- Remove now-redundant explicit `include_runtime_commit_tags=True` arguments only if
  that keeps the diff cleaner; either is fine.

### 4. Raw-commit backstop attribution (`src/sase/llm_provider/commit_finalizer_raw_commits.py`)

- Replace `_is_same_agent` with the discarded-work guard's matching semantics. Move
  `_agent_provenance_matches` (plus its `_names_match` / `_sase_agent_of` helpers) out
  of `commit_finalizer_git_progress.py` into a small shared module, and have both
  callers use it. Do not duplicate it. An agent's own trailer in global form
  (`owner.machine.<name>`) or linked form (`[owner.machine.<name>][n]` with a link
  footnote) must count as own. Another agent's trailer stays foreign.
- Decision `tool_commit_exemption = exempt`: a commit whose footer has a SASE `TYPE` tag
  but no `AGENT` tag is SASE tooling run outside any agent, so it is not this turn's raw
  commit and is skipped. Commits with neither tag still count, since that is what raw
  `git commit` produces. Commits whose `AGENT` names this agent still count. Under
  `count`, keep today's rule that unattributed means counted, regardless of `TYPE`.
- Update the module docstring and `_is_foreign_agent_commit`'s docstring to state the
  final rule.
- Absorption needs no change. `_prove_owned_suffix` already refuses any range that
  contains a filtered commit; add a test that pins this for the exempted case.

## Tests

- Memory init in agent context (`SASE_AGENT_NAME` set) against a disposable git-backed
  fake chezmoi home: project files regenerate; the home root is untouched with no new
  commit and no `chezmoi apply`; the notice prints; exit code 0. With
  `SASE_GATE_COMMAND=1`, or with the bypass var (reason printed), the home deploy
  happens and the commit carries `SASE_AGENT=`. Outside agent context, behavior is
  unchanged.
- Check mode: home-only drift → exit 0 plus the deferred warning (per
  `check_drift_scope`); project drift still fails; a `sase validate` level test covers
  the same.
- Chezmoi deploy: with an agent identity in the environment, a deploy commit message
  carries `SASE_AGENT`; without one it carries only `SASE_TYPE`.
- Raw-commit detection, as real-git cases in
  `tests/llm_provider/_commit_finalizer_raw_commits_detection_tests.py` (and the absorb
  and protected suites where relevant). Use real trailer shapes, not just
  `SASE_AGENT=current-agent`:
  - The agent's own linked-form trailer counts.
  - A plain global-form trailer for the agent counts.
  - A foreign linked-form trailer is skipped.
  - `SASE_TYPE=memory` with no agent is skipped (exempt) or counted (count).
  - A plain raw commit counts.
  - Regression for this incident: three foreign `SASE_TYPE=memory` commits in a sibling
    do not fail `raise_if_unpublished_local_commits`.
- Test hermeticity: the new agent-context gate makes any existing memory-init test that
  expects a home deploy depend on the ambient environment. Confirm that the autouse
  agent-env scrubbing in `tests/_conftest_environment.py` clears every marker the helper
  reads, including `SASE_MONITOR_ID` and `SASE_ARTIFACTS_DIR`. Extend it if it doesn't,
  so the suite behaves the same in CI and inside agent `just check` runs.

## Docs

- Where docs describe `sase memory init`'s home deployment or `--check` (search `docs/`
  with `rg -n "memory init"`), document the in-agent deferral, the
  `SASE_HOME_MEMORY_DEPLOY_BYPASS` escape hatch, and the check-mode warning. Read the
  `docs_writing` reference memory before editing docs. No CLI options change, so the
  generated CLI reference should not move.

## Non-goals

- Making `--no-commit` skip the chezmoi home commit (`sase-l0`). The TUI no-commit
  publish branch depends on current behavior, and agents no longer reach that path.
- Making memory init push chezmoi.
- Cleaning up the three existing chezmoi commits (human action, see Context).
- The unrelated `just check` failures in the same monitor run: the TUI import budget is
  at its 3520 cap on clean HEAD, and
  `test_supervisor_sigterm_records_stop_intent_worker_settles_stop` hits a SQLITE_BUSY
  flake. Both are already recorded as `PROPOSED FOLLOW-UP:` notes on `sase-1li.3`.
- Memory note changes. `/sase_memory_write` step 4 stays accurate, because
  `sase memory init` still regenerates project `AGENTS.md`, shims, and README.

## Verification

- Targeted pytest for the touched modules, then the repo gate per the `lint_and_test`
  reference memory (`just check` through `sase tool run`).
- Manual smoke in a scratch project with an isolated chezmoi home, never the real
  `~/.local/share/chezmoi`: with `SASE_AGENT_NAME=x`, `sase memory init` prints the
  deferral and leaves the fake chezmoi repo's `git log` unchanged.
