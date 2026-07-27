---
tier: epic
title: Add the ci_watch lumberjack chop for SASE CI health and release merges
goal: A bugyi_chop_ci_watch script chop sweeps GitHub Actions health for all SASE
  repos every five minutes, proposes at most one CI-fix agent per tick and only when
  zero other agents are running, deterministically merges green release-please/release-plz
  PRs once explicitly enabled, and ships configured (merges off) as an athena lumberjack
  lane.
phases:
- id: chop-context-flags
  title: Thread source and dry_run into the chop script contract
  depends_on: []
  size: small
  description: 'chop-context-flags: add source and dry_run to ChopScriptContext with
    SASE_CHOP_SOURCE/SASE_CHOP_DRY_RUN env mirrors and a tolerant reader, so mutation-capable
    chop scripts can honor sase axe chop run --dry-run.'
- id: ci-watch-chop
  title: Implement bugyi_chop_ci_watch in bugyi-chops
  depends_on: []
  size: medium
  description: 'ci-watch-chop: implement the actstat sweep, per-repo classification,
    zero-agents-gated fix proposals, guarded release-PR merging, notifications, and
    a fake-adapter test suite in the bugyi-chops repo.'
- id: athena-rollout
  title: Configure the athena ci_watch lane and deploy
  depends_on:
  - ci-watch-chop
  size: small
  description: 'athena-rollout: add the chop-owned actstat config and ci_watch lumberjack
    lane in chezmoi, apply with chezmoi, refresh the installed bugyi-chops, and dry-run
    smoke test the chop.'
create_time: 2026-07-27 12:51:08
status: wip
bead_id: sase-a2
---

# Plan: ci_watch — an actstat-driven CI watch / release-merge lumberjack chop

## Context

Consolidated research for this chop lives in the research sidecar at `202607/actstat_ci_release_chop/` (open with
`sase repo open research` if background is needed; this plan is self-contained). Facts below were verified against the
live system and code.

**What the chop does.** One script chop, `bugyi_chop_ci_watch`, runs on a five-minute lumberjack lane. Each tick it
performs one `actstat -f jsonl` sweep over a chop-owned actstat config, then decides per repo: an actionably-red default
branch gets one deduplicated fix-agent launch proposal; a green default branch with a fully-green
release-please/release-plz PR gets a deterministic `gh pr merge --squash --match-head-commit`. Everything unknown or
unsettled fails closed. It ships with `merge_enabled: false`.

**The requirement added on top of the research:** any agent this chop launches must only start when **zero other SASE
agents are running** (an in-flight epic's lander may itself fix CI; an independent fixer would collide with it). The
gate must not block the merge half or observation, so it cannot be an `inhibit_if` guard (those suppress the whole chop
run) and `wait_on` only chains proposals within one result document. The gate therefore lives inside the script: see
"Zero-agents gate" in the ci-watch-chop phase.

**Verified platform facts:**

- `actstat` (`~/.cargo/bin/actstat`) emits JSONL records `active_commit` / `commit` / `repo_error`; per-repo failures
  are isolated rows; auth falls back `ACTSTAT_GITHUB_TOKEN` → `GH_TOKEN`/`GITHUB_TOKEN` → `gh auth token`; both
  `--config PATH` and the `ACTSTAT_CONFIG` env var are honored. It never inspects pull requests, `cancelled` runs can be
  superseded rather than red, and its newest _settled_ commit may be behind an in-flight HEAD — so `gh` must revalidate
  before any action.
- **No `sase-org` repo has branch protection** (404 from the branch-protection API on all release repos). GitHub will
  merge a release PR with failing checks, and a merge triggers publish workflows (PyPI/crates.io). The chop's merge
  predicate is the only gate, which is why merging must be deterministic code with redundant guards and why agents are
  never put in the merge path.
- Release PRs are identified by `headRefName` starting with `release-please--` (sase, sase-github, sase-telegram) or
  `release-plz-` (sase-core). All four repos are squash-merge-only with auto-merge disabled; do not key off author or
  labels. `sase-core`'s publish pipeline self-heals (PyPI-gated jobs plus a 6-hourly cron), so the chop never babysits
  publication after a merge.
- Per-repo gating (not a global red-blocks-everything rule) is required: `sase` CI's `published-core-minimum-smoke` job
  depends on the _published_ `sase-core-rs` package, so a red `sase` must not block the `sase-core` release that would
  turn it green. Only process-level failures (actstat invocation/auth/parse) fail the whole tick.
- AXE chop contract: scripts are external executables reading `--context <context.json>` via the `sase.chops` SDK; agent
  launches happen only through schema-versioned `proposed_launches`; a proposal's `dedupe_key` takes precedence over
  `once_per`; an explicit `agent_name` makes a launch-time name collision idempotent (recorded skip, key released);
  failed launched agents release their dedupe key so later ticks retry. Chop `env:` values resolve whole (no
  interpolation), so absolute binary paths travel in `vars`, never via a composed `PATH`.
- `sase agent list -j` emits a stable-schema JSON array of live agents across **all** projects (STARTING and
  `%wait`-blocked agents included; terminal agents appear only with `-a`). This is the zero-agents probe.
- Deployment topology on athena: the `sase` uv tool venv (`uv tool list`) installs `sase` **editable** from a local
  checkout and `bugyi-chops` **from its GitHub repo**; `bugyi_chop_*` entry points resolve from that venv's bin
  directory. So the daemon and the chop SDK share one `sase` install, phase ci-watch-chop deploys via
  `uv tool upgrade sase` after its changes merge, and phase chop-context-flags reaches the daemon when the editable
  checkout is updated and the daemon restarts.
- The runner already tracks `ChopRunSource = Literal["scheduled", "manual", "oneshot"]` (`src/sase/axe/_state_chops.py`)
  and threads `dry_run` through `src/sase/axe/chop_runner.py`, but `ChopScriptContext`
  (`src/sase/axe/chop_script_context.py`) exposes neither, so a script's _direct_ side effects (like `gh pr merge`)
  cannot currently honor `--dry-run`. Runner-level dry-run only intercepts proposal launching.

## Shared decision policy

Both halves of the chop follow this per-repo policy; implement it as pure functions so tests can cover every row.

| Default-branch state                       | Release PR                 | Action                                  |
| ------------------------------------------ | -------------------------- | --------------------------------------- |
| process-level actstat/auth/parse failure   | —                          | whole tick → `check_error`              |
| `repo_error` row                           | —                          | count `errors`, no action for that repo |
| active/queued run in flight, or newer HEAD | —                          | skip: `run_in_flight`                   |
| actionably red (below)                     | —                          | propose one fix agent (gated, deduped)  |
| `cancelled` with no failing job            | —                          | skip: `superseded`, not red             |
| green                                      | none                       | no-op                                   |
| green                                      | exists, not fully eligible | skip: `release_pr_not_ready`            |
| green                                      | exists, eligible           | `gh pr merge` (only if merging enabled) |

**Actionably red:** some retained run has `conclusion ∈ {failure, timed_out, startup_failure, action_required}`, or is
`cancelled` with at least one job whose conclusion is `failure`. `stale` and job-less `cancelled` runs are superseded,
not red. Before proposing, revalidate with `gh` that the red run's head SHA is still the current default-branch HEAD and
no newer run is queued or running.

**Merge eligibility — all guards must hold, else no merge:** (1) exactly one non-draft candidate PR whose `headRefName`
matches the release prefixes and whose base is the current default branch (ambiguity is an error, never a guess); (2)
`mergeable == MERGEABLE` and `mergeStateStatus == CLEAN` (skip on `UNKNOWN`); (3) nonempty `statusCheckRollup` with
every entry `COMPLETED` and conclusion in {SUCCESS, SKIPPED, NEUTRAL}; (4) the repo's default branch is green per this
tick's sweep; (5) the release generator is settled (no queued/running release-please Publish or release-plz run on the
default branch); (6) the PR is re-read immediately before merging and the head pinned with `--match-head-commit`; (7)
`merged_this_tick < max_merges_per_tick`. Iterate repos in `merge_order` (sase-core first, sase last) so the cross-repo
dependency converges; already-merged PRs vanish from the next sweep, keeping partial batches idempotent.

## Phase chop-context-flags

Work in the main SASE repository. Small, self-contained contract change; no Rust-core involvement.

1. Add `source: str = "scheduled"` and `dry_run: bool = False` fields to `ChopScriptContext`
   (`src/sase/axe/chop_script_context.py`). Defaults keep previously-written context files parseable.
2. Make `read_chop_context` tolerant of unknown JSON keys (filter to known dataclass fields) so a newer writer never
   crashes an older SDK parsing the same document. Today it calls `ChopScriptContext(**json.load(f))`, which raises on
   any extra key.
3. Thread the real values into every script run: the per-run context copy made by `prepare_chop_run_context` (called
   from the chop runner / lumberjack launch path, which already holds `ChopRunSource` and `dry_run`) must carry both
   fields, and the script subprocess environment must gain mirrors `SASE_CHOP_SOURCE` (literal source string) and
   `SASE_CHOP_DRY_RUN` (`"1"`/`"0"`), alongside the existing `SASE_CHOP_RESULT_FILE` handling. Manual runs pass
   `source="manual"`, scheduled ticks `"scheduled"`, oneshot `"oneshot"`.
4. Dry-run must still execute the script (that is the point: the script can now _know_ and skip its own direct side
   effects); runner-level proposal preview behavior is unchanged.
5. Update the chop-contract documentation in `docs/axe.md`: the new context fields, the env mirrors, and the rule that
   scripts with direct side effects must honor them.
6. Tests: context roundtrip with unknown keys and with legacy documents (fields absent → defaults); env mirrors reach
   the subprocess for scheduled and manual runs; `sase axe chop run <chop> --dry-run` yields `dry_run=True` in both
   context and env; existing dry-run proposal previews unchanged.

Run `just install` before the final `just check`, as repository policy requires.

## Phase ci-watch-chop

Work in the `bugyi-chops` repository: open it with `sase repo open gh:bbugyi200/bugyi-chops` and use the printed path.
Follow the existing house style: reuse `_common.py` (`run_chop`, `result_with_summary`, `safe_fragment`), strict mypy,
ruff, and the 90% coverage gate (`just` targets). Add the `bugyi_chop_ci_watch` entry point to `[project.scripts]`
(`bugyi_chops.ci_watch:main`) and bump the package version per repo convention.

**Structure** (keep pure decisions separate from subprocess adapters; argv lists only, never `shell=True`; validate
every repo/branch/workflow string; no tokens in vars, evidence, or logs):

- `ActstatClient` — runs `<vars.actstat_bin> -f jsonl`, parses each line into typed observations (`active_commit` /
  `commit` / `repo_error`), never trusts exit codes alone, and treats malformed output as a process-level failure (whole
  tick `check_error`).
- `GitHubReader` — narrow `gh` calls: current default-branch HEAD and queued/in-progress runs (red revalidation),
  release-PR discovery by head-branch prefix, PR detail (`isDraft`, `baseRefName`, `headRefOid`, `mergeable`,
  `mergeStateStatus`, `statusCheckRollup`), and generator-workflow settledness.
- `AgentsGate` — runs `<vars.sase_bin> agent list -j` and reports the number of live agents (see gate below).
- Pure functions `classify_repo(...) -> ERROR | PENDING | RED | GREEN` and `plan_release_merge(...)` implementing the
  Shared decision policy, plus the gate decision.
- `main` — SDK invocation, gate checks, counters, bounded JSON decision-ledger evidence, atomic result write; any
  unexpected exception fails closed via `run_chop`.

**Zero-agents gate (the launch condition for every proposal this chop emits):**

- Once per tick, before emitting any fix proposal, run `<vars.sase_bin> agent list -j` (no `-p`, no `-a`): a JSON array
  of live agents across all projects, including STARTING and `%wait`-blocked agents.
- If the array is non-empty, suppress **all** fix proposals this tick: set counter `agents_running=N`, record skip
  reason `agents_busy` per red repo, and include a bounded list of busy agent names in the evidence ledger. Nothing is
  proposed, so no dedupe key is consumed and the next tick retries cleanly once the system is idle.
- If the command fails or its output cannot be parsed, treat the system as busy (`agents_check_failed`): the fix half
  fails closed rather than launching into an unknown agent state. Do not fail the whole tick — the merge half launches
  no agents and keeps its own guards.
- `max_fix_proposals_per_tick` (default 1): even on an idle tick, propose at most one fix agent, choosing the first
  actionably-red repo in `vars.repos` order. Combined with the gate this strictly serializes the chop's own fixers — a
  new fix agent can only ever start when zero agents, including earlier ci_watch fixers, are running.
- The seconds-wide race between the script's check and the runner launching the proposal is accepted; note it in the
  chop description body rather than engineering around it.
- The merge half is deliberately _not_ gated on idle agents: it launches no agents, and release merges only require a
  green default branch and a fully green release PR.

**Fix proposals** (one per selected red repo): workspace `gh:sase-org/<repo>`; `dedupe_key: ci_fix:<repo>:<head_sha>` so
one red commit never relaunches across ticks; stable `agent_name: ci_fix.<repo-slug>` so a launch-time name collision
(an earlier fixer still alive) resolves as idempotent skip, guaranteeing at most one in-flight fixer per repo; no
model/effort overrides. Prompt shape: `#pr(ci_fix_<repo-slug>_<sha7>, status=ready)` rollover, the global
`#actstat(repo=sase-org/<repo>)` xprompt so the agent re-runs actstat and diagnoses _current_ state, the pinned red run
URL plus failed job/step names from the sweep as bounded evidence, and an explicit instruction to first re-verify the
failure is still current on the default branch and to leave the worktree unchanged (report only) if it was superseded or
already fixed.

**Merge execution:** only when `vars.merge_enabled` is true AND the `SASE_CHOP_DRY_RUN` env mirror is present with a
false value. Missing mirror means the daemon predates phase chop-context-flags, so merging is refused with reason
`merge_context_unavailable`; a true value means dry run, so the full merge plan is rendered into counters/evidence with
zero mutations. Eligible merges run `gh pr merge --squash --match-head-commit <headRefOid>` in `merge_order`, capped by
`max_merges_per_tick`; a `--match-head-commit` conflict is recorded and never retried in the same tick. Emit one
notification per executed merge and one per proposed fix via `<vars.sase_bin> notify create -s ci_watch` (stdin JSON
with a short note; tag `release` for merges, `ci` for fixes) so releases are never silent.

**Counters:** `repos`, `green`, `red`, `pending`, `errors`, `agents_running`, `fix_proposed`, `fix_suppressed`,
`release_candidates`, `merged`, `merge_skipped`.

**Tests** (fake `actstat`/`gh`/`sase` adapters, following the existing test style): all green / no PRs → no-op; current
red + idle → exactly one pinned proposal and zero merge calls; several red repos → only the first in `repos` order
proposes (cap), others recorded; red + agents running → zero proposals, `agents_busy` recorded; agent probe failure →
zero proposals, `agents_check_failed`; old red superseded by newer queued run → pending; settled SHA ≠ current HEAD →
pending; cancelled-without-failing-job → no proposal; `repo_error` / malformed JSONL / actstat process failure → fail
closed correctly (row vs whole tick); release-PR identity rules including near-miss branch names; each merge guard
individually dis-qualifying (draft, ambiguous, dirty, UNKNOWN, pending rollup, empty rollup, red branch, generator busy,
changed head); eligible PRs merged in `merge_order` under the per-tick cap; `merge_enabled=false`, missing mirror, and
dry-run each render the plan with zero mutations; no token-like strings in evidence.

## Phase athena-rollout

Work in the `chezmoi` repository: open it with `sase repo open chezmoi` and use the printed path. After committing, run
`chezmoi update -a --force` as that repo's instructions require.

1. Add `home/dot_config/sase/actstat-ci-watch.yml` — the chop-owned actstat config (schema as in the existing
   `home/dot_config/actstat/config.yml`): a single `projects` entry for `org: sase-org` with
   `exclude: [sase-org/sase-android]`. The chop's `vars.repos` allowlist is the second, independent scoping mechanism,
   so new org repos never silently widen what the chop can touch.
2. Add the `ci_watch` lane to `home/dot_config/sase/sase_athena.yml` (descriptions must follow the axe description
   grammar — summary line, blank line, body):

   ```yaml
   axe:
     lumberjacks:
       ci_watch:
         description: |-
           Watch GitHub Actions health across SASE repos and drive fixes or releases

           Runs every five minutes. The single chop owns its own actstat config so unrelated orgs are never swept,
           launches at most one CI-fix agent per tick and only when zero other agents are running, and is the only
           gate on release merges because no SASE repo has branch protection. Merging stays disabled until the fix
           half has soaked.
         interval: 300
         chop_timeout: "120s"
         chops:
           ci_watch:
             script: bugyi_chop_ci_watch
             description: |-
               Sweep actstat once, propose idle-gated CI fixes, and merge ready release PRs when enabled

               Red default branches propose one deduplicated fix agent (zero-agents gate, one per tick); green
               branches with fully green release PRs merge deterministically in dependency order, capped per tick.
               Everything unknown fails closed; the seconds-wide gap between the idle check and agent launch is
               accepted.
             env:
               ACTSTAT_CONFIG: /home/bryan/.config/sase/actstat-ci-watch.yml
             vars:
               actstat_bin: /home/bryan/.cargo/bin/actstat
               gh_bin: gh
               sase_bin: sase
               repos:
                 [sase-org/sase, sase-org/sase-core, sase-org/sase-github, sase-org/sase-telegram, sase-org/sase-nvim]
               release_repositories:
                 sase-org/sase: release-please
                 sase-org/sase-core: release-plz
                 sase-org/sase-github: release-please
                 sase-org/sase-telegram: release-please
               merge_order: [sase-core, sase-github, sase-telegram, sase-nvim, sase]
               max_merges_per_tick: 1
               max_fix_proposals_per_tick: 1
               fix_enabled: true
               merge_enabled: false
   ```

3. Deploy: refresh the installed chop package with `uv tool upgrade sase` (the `sase` uv tool venv pulls `bugyi-chops`
   from its GitHub repo), then verify the `bugyi_chop_ci_watch` entry point exists in that venv's bin directory. If the
   upgrade reports anything unexpected beyond refreshed packages, stop and report rather than forcing.
4. Smoke test: `sase axe chop run ci_watch --dry-run -V` — expect populated sweep counters, zero mutations, and either
   an `agents_busy` suppression or a previewed fix proposal. Until the daemon runs a build containing phase
   chop-context-flags, the `SASE_CHOP_DRY_RUN` mirror is absent; the chop must report merges as
   `merge_context_unavailable` in that state, which the smoke test should confirm if observable.

## Post-epic rollout (manual, not phases)

1. Deliberately materialize SASE projects for `gh:sase-org/sase-core`, `sase-github`, `sase-telegram`, `sase-nvim`
   (first launch otherwise pays first-time setup, and newly enabled projects join `for_each: source: projects`
   expansions such as `refresh_docs` — accept that knowingly).
2. Soak the fix half for a few days with `merge_enabled: false`; verify the red predicate never fires on superseded
   `cancelled` runs and that the gate defers fixers whenever agents are live.
3. Update the daemon's editable sase checkout (phase chop-context-flags) and restart, making the env mirrors real.
4. Flip `merge_enabled: true` with `max_merges_per_tick: 1`; watch the first `sase-core` release merge trigger its
   publish workflow and the next tick wait while it runs; then trust the remaining repos.
5. Add minimal branch-protection rulesets requiring CI on the release repos so the chop becomes a second gate instead of
   the only one.

## Risks and boundaries

- **The merge predicate is the only thing between a tick and a PyPI/crates publish.** Never weaken or "simplify away"
  the redundant guards; redundancy is the point. Agents stay out of the merge path entirely.
- The zero-agents gate must remain in-script: `inhibit_if` would suppress observation and merging too, and `wait_on`
  cannot express "wait for system idle". Do not convert the gate into either mechanism.
- Do not model per-repo fan-out with `for_each` (wrong source set — most watched repos are not SASE projects — and N×
  the sweep cost); one chop instance iterates internally.
- `cancelled` is red only with a failing job; treating all cancelled runs as red would page on every superseded push
  (`sase` uses cancel-in-progress concurrency).
- Chop `env:` values resolve whole with no interpolation: never compose `PATH`; absolute binary paths ride in `vars`.
- `read_chop_context` tolerance (phase chop-context-flags) is required so mixed writer/reader versions on athena can
  never crash every installed chop at once.
- Keep runtimes uniform: nothing provider-specific in proposals or prompts.
- Scope: `sase-google`/`sase-gchat` and acting on red release PRs directly stay out; red-master fixing plus PR
  rebuild-on-master-move covers the release-PR case. Fix agents use runner-default model/effort.
