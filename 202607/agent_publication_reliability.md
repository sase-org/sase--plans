---
tier: epic
title: Make post-commit agent publication survive sidecar commits and unpublishable
  hoods
goal: 'A `sase commit` inside an SDD sidecar checkout publishes to the sidecar''s
  host project instead of failing after the commit already landed, and a publication
  request that can never be satisfied is retired with a recorded reason instead of
  cycling through quarantine forever.

  '
phases:
- id: target-resolution
  title: Resolve the publication project from the committed repository path
  depends_on: []
  size: medium
  description: 'target-resolution: resolve the post-commit publication target by matching
    the commit''s repository path against the repo inventory (including workspace
    clones) so sidecar and linked-repo commits publish to the host project, and stop
    the auxiliary publication step from failing a commit that already landed.

    '
- id: terminal-disposition
  title: Retire publication requests that can never be published
  depends_on: []
  size: medium
  description: 'terminal-disposition: add a durable terminal disposition to the publication
    outbox, classify "hood has no publishable runs" as terminal once a retry has confirmed
    it, and stop `--retry-quarantined` from resurrecting terminal requests into an
    endless quarantine cycle.

    '
- id: unpublishable-surface
  title: Operator surface for retired requests and residue cleanup
  depends_on:
  - terminal-disposition
  size: small
  description: 'unpublishable-surface: expose retired requests through `sase agent
    sync`, `sase doctor`, and the post-commit warning with accurate remediation, and
    purge the three residue requests currently stuck in this machine''s sase outbox.

    '
create_time: 2026-07-28 14:19:09
status: done
bead_id: sase-ah
---

- **PROMPT:** [202607/prompts/agent_publication_reliability.md](prompts/agent_publication_reliability.md)

# Plan: Make post-commit agent publication survive sidecar commits and unpublishable hoods

## Goal

Two independent defects in the post-commit agent-hood publication step were reproduced on this machine. Neither is
transient. Fix both, and clear the residue the second one already left behind.

## Evidence

### Defect 1 — a sidecar commit reports FAILED after it already committed and pushed

`publish_committed_agent_hood` resolves its target from the _workspace repo name_
(`src/sase/agents_sync/commit_publication.py:73-84` → `_current_project()` → `get_project_from_workspace()` in
`src/sase/workflows/utils.py:39-63`). Inside an SDD sidecar checkout that name is the sidecar **repo** name, and
`resolve_sync_targets` only considers **projects** (`src/sase/agents_sync/targets.py:31-36` passes
`include_home=False, projects_only=True`). Reproduced from `sase/repos/plans`:

```
provider:                    VCSPluginManager
get_workspace_name:          (True, 'sase--plans')
get_project_from_workspace:  sase--plans
resolve_sync_targets:        targets=()  outcome.error="project 'sase--plans' was not found"
```

`_run_agent_publication_step` sees `outcome.error` with no `queued` and no `skip_reason`, so it takes the hard-failure
branch at `src/sase/workflows/commit/workflow.py:431-438`: it prints "agent publication could not be queued" and returns
`RunResult.FAILED` — _after_ the primary commit was created and pushed. `sase commit --resume` from the same sidecar
resolves the same name and fails identically; the loop only breaks by resuming from the primary workspace, which is what
the `sase-ag.land--code` agent eventually discovered by trial.

The inventory already knows the answer, including for numbered-workspace clones:

```
MATCH name=plans kind=sidecar project=sase key=gh_sase-org__sase
  path=/home/bryan/projects/github/sase-org/sase/sase/repos/plans
  clone=RepoCloneRecord(workspace_num=12,
        path='<workspace>/sase/repos/plans', exists=True)
```

### Defect 2 — an unpublishable request cycles through quarantine forever

`gh_sase-org__sase`'s outbox holds three requests that cannot ever be satisfied:

```
bbugyi200.athena.k4 | attempts 1 | quarantined False | created 2026-07-25
    published agent page for 'bbugyi200.athena.k4' did not materialize during full sync
bbugyi200.athena.lt | attempts 1 | quarantined False | created 2026-07-27
    published agent page for 'bbugyi200.athena.lt' did not materialize during full sync
bbugyi200.athena.lz | attempts 1 | quarantined False | created 2026-07-27
    published agent page for 'bbugyi200.athena.lz' did not materialize during full sync
```

None of the three has a run page, a hood page, or any local artifact:

```
k4: run-page=NO hood-page=NO local-artifacts=0
lt: run-page=NO hood-page=NO local-artifacts=0
lz: run-page=NO hood-page=NO local-artifacts=0
```

`lt` and `lz` are **`home`-project** agents — their transcripts are
`~/.sase/chats/202607/home-ace_run-lt-260727_061635.md` and `home-ace_run-lz-260727_070252.md`. `home` is excluded from
sync targets by `src/sase/agents_sync/targets.py:31-36`, so their hoods have no publishable destination anywhere, yet
the requests were enqueued under `gh_sase-org__sase` because `publish_committed_agent_hood` keys the outbox off the
commit's cwd.

The retry machinery cannot converge on them:

1. The commit path calls `publish_agent_hood`, which raises `AgentsSyncFormatError("hood 'lt' has no publishable runs")`
   (`src/sase/agents_sync/publication.py:88-91`). `_prepare_publications` records the error with
   `quarantine_threshold=configured_publication_max_attempts()` (`DEFAULT_PUBLICATION_MAX_ATTEMPTS = 3`,
   `src/sase/agents_sync/publication_outbox.py:20`), so after three commits the request is quarantined and every later
   commit prints the "N quarantined agent-hood publication request(s)" warning.
2. `sase agent sync --retry-quarantined` calls `clear_quarantined_agent_publications`
   (`src/sase/agents_sync/publication_outbox.py:219-243`), which resets `attempts=0`, `last_error=None`,
   `quarantined=False` for **every** quarantined item unconditionally.
3. The same full sync then re-checks `_publication_request_materialized` (`src/sase/agents_sync/git_sync.py:155-161`),
   the page still does not exist, and `src/sase/agents_sync/git_sync.py:99-122` writes the "did not materialize during
   full sync" error back with `increment_attempts=True` and the same quarantine threshold.

That is exactly the current outbox state: attempts back at 1, not yet re-quarantined. The `sase-ag.land--code` agent
reported this queue as "cleared"; it was reset, not cleared, and it will re-quarantine on the third attempt.

## Design decisions

- **Resolve by path, not by name.** The commit workflow already records the committing repository root (`cp.cwd`). The
  repo inventory maps that path — primary checkout, sidecar checkout, linked-repo checkout, or any registered
  numbered-workspace clone of those — to a `project_key`. Name-based resolution cannot distinguish a repo from a project
  and is the direct cause of defect 1.
- **The agent's hood belongs to the host project.** Publishing a sidecar commit's hood into the host project's agents
  sidecar is what the successful primary-workspace resume already did, and it matches how the agent's runs are
  inventoried.
- **An auxiliary step must never fail a landed commit.** Returning `RunResult.FAILED` is correct for an enqueue or
  persistence error the operator can act on. It is wrong when the committed repository simply has no agents target, and
  it is actively harmful when `--resume` cannot change the outcome.
- **Quarantine is "stop and ask a human", not "this can never work".** The outbox needs a second, terminal disposition
  so that an unsatisfiable request is retired once rather than resurrected on every `--retry-quarantined`.
- **Confirm before retiring.** "hood has no publishable runs" can be a transient artifact-indexing race when a commit
  happens early in an agent's run. Require at least one prior recorded attempt with the same terminal reason before
  retiring, so a race resolves itself on the next attempt and only a genuinely unsatisfiable request is retired.
- **A pre-enqueue inventory guard is rejected.** Enqueue is deliberately durable-first and runs before the agents lock
  is taken; building a hood inventory there would slow every commit and would reintroduce the same indexing race it is
  meant to prevent. The terminal disposition handles the condition durably instead.
- **Rust core boundary.** `sase_core_rs` exports no publication or outbox bindings today (verified: no matching
  symbols), so this subsystem is Python-only and the fix stays where the code lives. This plan adds no new domain that
  would belong behind the binding.

## Phases

### target-resolution — Resolve the publication project from the committed repository path

Add a resolver that maps a committed repository path to the owning `project_key`, and use it for the publication target.

- Add a path-based resolver (`src/sase/agents_sync/commit_publication.py`, or a small helper module beside it if that
  keeps the import graph clean — `tests/agents_sync/test_import_boundaries.py` guards this package's imports). It takes
  the commit cwd, resolves it to its repository root, and matches it against `collect_repo_inventory()`:
  `RepoRecord.path` and every `RepoRecord.clones[].path`. On a tie, prefer the record with the lowest `_KIND_ORDER`
  (`primary` < `sidecar` < `linked` < `external`) so a primary checkout never loses to a nested sidecar.
- Thread the commit cwd into `publish_committed_agent_hood`. It currently derives the project from the ambient process
  cwd; the workflow already has `cp.cwd` at the call site (`src/sase/workflows/commit/workflow.py:417-421`). Keep the
  existing `project=` parameter working for callers and tests that pass an explicit project.
- Keep name-based resolution as the fallback when the path matches no inventory record, so behavior in a plain primary
  checkout is unchanged.
- Make the auxiliary step non-fatal when the repository has no agents target. When resolution yields no project, or when
  `resolve_sync_targets` returns no target for the resolved project, return a `skip_reason` rather than a bare `error`,
  so `src/sase/workflows/commit/workflow.py:431-438` takes the existing skip path instead of `RunResult.FAILED`. Print a
  warning naming the repository and why publication was skipped. Preserve `RunResult.FAILED` for genuine
  enqueue/persistence failures, which `--resume` can still fix.

Tests in `tests/agents_sync/test_commit_publication.py` plus the commit-workflow tests:

- A sidecar checkout path resolves to its host `project_key`.
- A registered numbered-workspace clone of a sidecar resolves to the same host `project_key`.
- A linked-repo checkout resolves to its host project.
- A primary checkout still resolves to its own project (no regression).
- An unregistered path outside every known repo yields a skip, and the commit workflow returns OK rather than FAILED.

Verification: from a plans-sidecar checkout, the reproduction above must resolve to `gh_sase-org__sase` instead of
`sase--plans`.

### terminal-disposition — Retire publication requests that can never be published

Give the outbox a durable terminal disposition and stop the quarantine cycle.

- Extend `AgentPublicationOutboxItem` (`src/sase/agents_sync/publication_outbox.py:25-75`) with a terminal disposition
  and the reason that produced it, and serialize both in `to_json_dict`. Bump `PUBLICATION_OUTBOX_SCHEMA_VERSION` to 3
  and read schema-2 documents tolerantly (absent fields default to not-terminal), so an existing outbox loads without
  migration.
- Classify terminal failures. `publish_agent_hood` raising `AgentsSyncFormatError` for a hood with no publishable runs
  (`src/sase/agents_sync/publication.py:88-91`), and the full-sync "did not materialize" path
  (`src/sase/agents_sync/git_sync.py:99-122`), describe a project-scoped fact that a retry alone cannot change. Mark
  such a request terminal only when a prior attempt already recorded the same terminal reason, so a first-attempt
  indexing race still gets its retry. Record the classification on the item instead of incrementing toward quarantine.
- Exclude terminal requests from the active queue the way quarantined ones are excluded, so they neither block a publish
  nor count as pending work.
- Make `clear_quarantined_agent_publications` skip terminal items. It currently resets every quarantined item
  unconditionally (`src/sase/agents_sync/publication_outbox.py:219-243`); a terminal item must stay retired.
- Add a function that drops terminal requests from an outbox and returns what it dropped, for the next phase's operator
  surface.

Tests in `tests/agents_sync/test_publication_outbox.py`, `tests/agents_sync/test_commit_publication.py`, and
`tests/agents_sync/test_git_sync.py`:

- A schema-2 outbox document loads with no terminal items and round-trips as schema 3.
- A first "no publishable runs" failure retries; a second one retires the request.
- A retired request is not returned as active work and does not quarantine.
- `--retry-quarantined` resurrects a quarantined request but leaves a retired one retired — the specific loop that makes
  the current residue immortal.
- A transient failure (busy lock, git error) never retires a request.

### unpublishable-surface — Operator surface for retired requests and residue cleanup

Make retired requests visible and actionable, then clear the existing residue.

- Add a `sase agent sync` flag that drops retired requests (sibling to `--retry-quarantined` at
  `src/sase/main/parser_agent.py:144-152`, wired through `src/sase/agents/cli_sync.py:54` and
  `src/sase/agents_sync/git_sync.py:51`). Report how many were dropped and for what reason.
- Report retired requests separately in `src/sase/doctor/checks_agent_publication.py`. Today every problem routes to
  `_REMEDIATION_COMMAND = "sase agent sync --retry-quarantined"` (line 26), which for a retired request is advice that
  cannot work; point those at the drop command instead.
- Fix the post-commit warning. `_agent_publication_deferred_message` (`src/sase/workflows/commit/workflow.py:566-580`)
  tells the operator to run `--retry-quarantined`; when the outstanding requests are retired, it must name the drop
  command instead.
- Update `publication_quarantine_diagnostics` (`src/sase/agents_sync/publication_outbox.py:259-272`) and the Chats-tab
  publication rendering (`src/sase/ace/tui/widgets/artifacts/chats_detail.py:194-228`) so a retired request reads as
  retired rather than as retryable.
- Purge the residue: run the new drop command against `gh_sase-org__sase` and confirm the three `k4`/`lt`/`lz` requests
  are gone and the outbox is empty.

Tests in `tests/agents_sync/test_cli.py` and the doctor tests:

- The drop flag removes retired requests and leaves active and quarantined ones untouched.
- `sase doctor` reports a retired request with the drop remediation, not the retry remediation.
- The post-commit warning names the drop command when the outstanding requests are retired.

## Verification

- `just install` first — this is an ephemeral workspace clone.
- `just check` after each phase.
- Defect 1: from a plans-sidecar checkout, publication target resolution yields `gh_sase-org__sase`; a real
  `sase commit` in the plans sidecar completes without a FAILED result and without needing a primary-workspace resume.
- Defect 2: `gh_sase-org__sase`'s outbox is empty after the drop, and a subsequent `sase commit` prints no quarantine
  warning. Re-running `sase agent sync --retry-quarantined` does not bring the three requests back.

## Out of scope

- Migrating the publication outbox behind `sase_core_rs`. No publication bindings exist today; this is a bug fix in the
  existing Python subsystem, not a new domain crossing the boundary.
- Preventing a `home`-project agent from committing into another project's repository in the first place. The terminal
  disposition retires the resulting request durably; changing agent-to-project attribution at commit time is a separate
  feature with a much wider blast radius.
- Reworking the busy-lock path. `_record_failure` already passes no `quarantine_threshold`, so lock contention alone
  does not quarantine anything; the observed quarantine came entirely from the unpublishable-hood path above.
