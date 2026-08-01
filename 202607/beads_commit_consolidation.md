---
tier: epic
title: Consolidate beads sidecar commits
goal: 'Cut routine beads-sidecar commit volume by making one logical operation produce
  one commit and one push: `sase bead work` batch-marks every launched bead in-progress
  in a single pre-spawn commit, runtime claim transitions become quiet no-ops, close-with-note
  lands in one command, projections serialize deterministically so clones stop committing
  pure-reorder echoes, and each agent `sase commit` writes at most one beads commit.
  Every `sase bead` subcommand keeps committing and pushing by default.

  '
phases:
- id: core
  title: Idempotent claim mutations and epic-capable batch preclaim in Rust core
  depends_on: []
  size: medium
  description: 'core: make runtime claim/close/update mutations quiet no-ops when
    they would not change state, and extend the batch epic-work preclaim so it can
    include the epic bead itself with the land agent assignment.

    '
- id: quiet
  title: Skip commits and pushes on no-op bead mutations and batch the claim chop
  depends_on:
  - core
  size: medium
  description: 'quiet: teach every Python bead-mutation caller to skip the commit
    and push entirely when the mutation outcome reports no change, treat in-progress-with-matching-assignee
    as a healthy held claim, and collapse the per-bead commit loops in the bead-claim-checks
    chop into one commit and one push per cycle.

    '
- id: launch
  title: Single-commit epic launch in sase bead work
  depends_on:
  - core
  - quiet
  size: large
  description: 'launch: batch-preclaim every launched phase bead and the epic bead
    inside the existing launch transaction so `sase bead work` produces exactly one
    beads commit and one synchronous pre-spawn push, with runner-side claim, promotion,
    and release falling through as no-ops.

    '
- id: detproj
  title: Deterministic bead projection output
  depends_on:
  - core
  size: medium
  description: 'detproj: make `issues.jsonl` and every other regenerated projection
    byte-stable across all writers so clones never dirty their worktree with pure
    reorders that later get swept into commits under recycled semantic messages.

    '
- id: closenote
  title: Close-with-note in one mutation and one commit
  depends_on:
  - core
  size: medium
  description: 'closenote: add a `--note` option to `sase bead close` that appends
    an attributed note entry and closes the bead in one mutation and one commit, then
    update the runtime working-loop guidance so agents stop issuing separate note
    and close commands.

    '
- id: postcommit
  title: One beads commit per agent commit
  depends_on: []
  size: medium
  description: 'postcommit: fold the commit finalizer''s bead-state sync commit and
    the bead-pages publication commit into a single beads-repo commit per agent commit,
    and keep finalizer retry passes from re-committing when nothing changed.

    '
create_time: 2026-07-28 16:21:28
status: done
bead_id: sase-aj
---

- **PROMPT:** [prompts/202607/beads_commit_consolidation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/beads_commit_consolidation.md)
- **BEAD:** [sase-aj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-aj/README.md)
- **AGENTS:**
  - [bbugyi200.athena.nd](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.nd/README.md)
  - [bbugyi200.athena.sase-aj.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aj.1/README.md)
  - [bbugyi200.athena.sase-aj.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aj.2/README.md)
  - [bbugyi200.athena.sase-aj.3--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-aj.3.md#member-code)
  - [bbugyi200.athena.sase-aj.3--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-aj.3.md#member-plan)
  - [bbugyi200.athena.sase-aj.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aj.4/README.md)
  - [bbugyi200.athena.sase-aj.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aj.5/README.md)
  - [bbugyi200.athena.sase-aj.6](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aj.6/README.md)
  - [bbugyi200.athena.sase-aj.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aj.land/README.md)

# Plan: Consolidate beads sidecar commits

## Goal

The beads sidecar repo accumulates commits far faster than the semantic history justifies. Of the last ~230 commits in
the sase project's beads sidecar, 107 (47%) are per-bead `chore(beads): claim <id> for <agent>` commits, 64 (28%) are
closes (several of them duplicates), and 42 (18%) are notes that immediately precede a close by the same agent. A single
10-phase epic launch currently produces roughly 20–25 claim-related commits, each with its own blocking managed-sync
push. This epic makes one logical operation produce one commit and one push, without changing the default that every
`sase bead` subcommand commits and pushes its own mutation.

Expected end state for one 10-phase epic lifecycle:

- Launch: 1 beads commit + 1 synchronous push (today: ~1 checkpoint + ~11 wait-claims + ~10 promotions + land claims,
  each with its own push).
- Per finished phase: 1 `close --note` commit (today: 1 note commit + 1–2 close commits).
- Per agent `sase commit`: at most 1 beads commit (today: `sync bead state` + `Publish bead pages` = 2).
- Claim-checks chop cycle: at most 1 commit + 1 push (today: 1 per reconciled bead).
- Pure projection-reorder echo commits: 0 (today: recurring, committed under recycled `claim`/`close` messages).

## Current state

How a bead mutation becomes a commit:

- Canonical state is JSONL event streams under `events/streams/<root>.jsonl` plus the generated `issues.jsonl`
  compatibility projection. The Rust core (`sase_core` crate, `crates/sase_core/src/bead/mutation.rs` in the sase-core
  linked repo) performs every mutation and writes files into the worktree; it never runs git. Python commits whatever is
  dirty afterward.
- The commit primitives are `commit_sdd_files` (`src/sase/sdd/_commit_store.py:33`), `commit_sdd_store_files`
  (`src/sase/sdd/_commit.py:49`), and the bead-specific `_commit_bead_state` (`src/sase/bead/_sync_git.py:200`) with its
  named wrappers (`commit_bead_claim`, `commit_bead_claim_release`, `commit_epic_graph_checkpoint`,
  `commit_bead_work_launch`, `commit_epic_creation_rollback`). Crucially, the commit layer stages _all_ changed files
  under the store and commits them under whatever message the current operation supplies.
- CLI mutations take one of two paths that must stay in behavioral lockstep: the Rust fast path
  (`src/sase/main/bead_fast_path.py`, side effects at `_apply_mutation_side_effects`, messages from
  `_mutation_commit_message`) and the Python slow path (`src/sase/bead/cli_crud.py` via `bead_store_mutation` /
  `auto_commit_bead_store` in `src/sase/bead/cli_common.py`). Each CLI invocation is one commit + one push today.

Where the commit spam comes from:

1. **Per-agent runtime claims.** Every launched agent independently commits and synchronously pushes: a wait-time claim
   (`claim_bead_for_waiting_agent`, `src/sase/bead/claims.py:132`, invoked from
   `src/sase/axe/run_agent_runner_bootstrap.py` when the segment has wait directives), a promotion commit right before
   model execution (`claim_bead_for_agent_launch`, `src/sase/axe/run_agent_runner_bead.py`, invoked from
   `src/sase/axe/run_agent_runner_launch.py`), and a release commit if it dies before starting
   (`release_bead_claim_for_agent`, `src/sase/bead/claims.py:235`, invoked from
   `src/sase/axe/run_agent_runner_lifecycle.py`). The claim-checks chop
   (`src/sase/scripts/sase_chop_bead_claim_checks.py`) additionally loops over release and acquire candidates with one
   commit + one synchronous push per bead.
2. **Non-idempotent core mutations.** In the sase-core linked repo, `claim_for_agent_wait` already returns
   `changed=false` without writing when the bead is not open, but `claim_for_agent_launch` unconditionally appends an
   event and saves even when the bead is already in-progress with the same assignee, and `update_issue` always emits an
   event even when no field value changes. The promotion caller (`run_agent_runner_bead.py`) then _requires_ a commit
   and hard-fails the agent if none is produced.
3. **Dirt sweeping under recycled messages.** The projection writer is not byte-stable across write paths, so a clone
   that rebases or refreshes can be left with a dirty, purely-reordered `issues.jsonl`. The next bead operation in that
   clone commits the dirt under its own semantic message. Observed in the sase beads sidecar as duplicate
   `chore(beads): close sase-ai.8` commits two minutes apart (the second a 348-line pure reorder of `issues.jsonl` with
   no event change) and as repeated identical `claim <epic> for <epic>.land` commits — `claims.py` explicitly commits
   when `held_by_us and not bead_state_is_clean(beads_dir)` even though nothing changed.
4. **Two beads commits per agent commit.** After every agent `sase commit`, the commit finalizer commits
   `chore(beads): sync bead state` (`src/sase/llm_provider/commit_finalizer.py`, pre-pass and again per finalizer retry
   pass) and the bead-pages publication step commits `Publish bead pages for <root_id>`
   (`src/sase/bead_pages/publication.py:127`, invoked from `src/sase/workflows/commit/workflow.py`).
5. **Two-command finish sequence.** The documented working loop tells agents to run `sase bead note` and then
   `sase bead close`, producing two commits seconds apart for every finished bead.

Relevant history — do not rediscover this the hard way:

- A batch preclaim already exists. `preclaim_epic_work_plan` (sase-core `crates/sase_core/src/bead/mutation.rs`)
  validates all phase targets, flips them to in-progress with per-bead assignees in one lock + one save, and returns a
  rollback payload. It is exposed through the PyO3 binding as `bead_preclaim_epic_work` but has **no Python caller**
  today. It only accepts phase children; it rejects the epic bead itself.
- `sase bead work` used batch preclaim from 2026-05-09 (`e1b0a1bf6`) until 2026-07-20, when the JIT migration
  (`9d8b7e280`, plan `202607/jit_epic_bead_work.md` in the plans sidecar) deliberately moved the in-progress transition
  to each runner to keep waiting work unclaimed and make launch-failure cleanup honor runner ownership. This epic
  reverses the _commit_ consequence of JIT while keeping its recovery properties: launch failures still roll back
  cleanly, rerun still reassigns surviving phases, and dry runs still mutate nothing. The user has explicitly accepted
  the semantic trade-off that beads belonging to a launched epic read `in_progress` (with their planned agent as
  assignee) while their agents are still waiting on dependencies.
- Epic-graph creation is already consolidated: `epic_from_plan.py` creates the epic, all phase beads, and all dependency
  edges without committing, and one `chore(beads): checkpoint approved epic graph <id>` commit plus one synchronous push
  barrier (`publish_epic_graph_before_launch`, `src/sase/bead/cli_work_from_plan_store.py`) lands it all before agents
  spawn. That barrier pattern is the model for this epic.
- Existing consolidation primitives to reuse rather than reinvent: `bead_store_mutation` accumulates one commit for many
  mutations (`src/sase/bead/cli_common.py`), `already_locked`/lock-handoff lets one commit cover a locked mutation span,
  `epic_plan_launch_lock` is documented as the outermost launch transaction boundary, `defer_push` and the
  `bead.push_after_commit` / `sdd.push_after_commit` config already batch pushes, and `bead_state_is_clean`
  (`src/sase/bead/_sync_git.py`) is a cheap is-there-anything-to-commit probe.

## Design constraints

- Every `sase bead` subcommand keeps committing and pushing by default. Consolidation means one logical operation writes
  one commit; it never means deferring a subcommand's own commit to some later flush. A session-level deferred commit
  mode was considered and rejected for violating this constraint.
- Rust core boundary: mutation semantics, idempotence rules, batch preclaim, close-with-note, and projection
  serialization are shared domain behavior and belong in the sase-core linked repo (`crates/sase_core/src/bead/`),
  reached with `sase repo open sase-core`. Python phases only adapt callers. Dev installs build `sase_core_rs` from the
  local sase-core checkout (see the Justfile core-overrides machinery), so Python phases can consume unreleased core
  changes; the published `sase-core-rs` floor in `pyproject.toml` is raised at land time once wheels ship, following the
  existing `build(deps): require sase-core-rs <version>` pattern.
- Runner-side claims stay the authoritative path for beads that were _not_ pre-marked by an epic launch (ad-hoc
  `%id(bead=...)` agents, hand-claimed backlog work). Nothing in this epic may regress that path; it merely becomes a
  no-op when the launch already recorded the same state.
- The launch commit must be pushed synchronously before any agent spawns. Runners refresh their own beads clones; if the
  pre-marked state were not visible remotely, a runner's wait-claim would see `open` and re-commit a claim,
  reintroducing the spam this epic removes.
- Scope is the beads sidecar. Plans-sidecar commit reduction (e.g. merging the archive and plan-link commits) is a known
  follow-up, not part of this epic.

## core: Idempotent claim mutations and epic-capable batch preclaim in Rust core

All work in the sase-core linked repo (`crates/sase_core/src/bead/`), plus the PyO3 binding surface.

- `claim_for_agent_launch`: when the bead is already `in_progress` with the same assignee, return the issue with
  `changed=false` without appending an event or saving. Keep the closed-bead error and the unconditional write for every
  other state.
- `claim_for_agent_wait`: add a quiet-success case — `in_progress` with the same assignee returns `changed=false` with
  **no** contention message, mirroring the existing repeated-claim no-op. Genuine contention (held by a different
  assignee) keeps its message.
- `update_issue`: when every requested field equals the current value, emit no event, do not bump `updated_at`, and
  return `changed=false`. Verify `close_issues` reports `changed=false` (and writes nothing) when every listed id is
  already closed, and fix it if it does not.
- `preclaim_epic_work_plan`: accept an optional epic-bead assignment (the land agent's name) alongside the phase
  assignments. The epic bead flips to `in_progress` with that assignee in the same lock + save, appears in the rollback
  payload, and keeps today's validation for phase targets (children of the epic, not closed; already in-progress phases
  stay reassignable for reruns).
- Update the PyO3 binding for the extended preclaim signature and make sure `changed` is faithfully exposed for every
  mutation outcome the Python side consumes.
- Tests: unit coverage for each new no-op case (no event appended, no file written, `updated_at` untouched), epic-
  inclusive preclaim (single save, rollback payload includes the epic), and reassignment of an in-progress phase. Run
  the sase-core test suite.

## quiet: Skip commits and pushes on no-op bead mutations and batch the claim chop

Python-side callers in this repo. Depends on core for honest `changed` reporting.

- `auto_commit_bead_store` consumers: when a mutation outcome reports `changed=false`, skip the commit and the push
  entirely, in both the slow path (`bead_store_mutation` in `src/sase/bead/cli_common.py`, handlers in
  `src/sase/bead/cli_crud.py`) and the Rust fast path (`_apply_mutation_side_effects` in
  `src/sase/main/bead_fast_path.py`). This is what stops unrelated worktree dirt from being swept into commits under
  recycled semantic messages; leftover dirt remains for the honest sync paths (commit finalizer, store refresh).
- While in there, deduplicate the two parallel commit-message tables (`cli_crud.py` literals and
  `bead_fast_path._mutation_commit_message`) into one shared helper so future changes cannot diverge.
- `claim_bead_for_waiting_agent` (`src/sase/bead/claims.py`): drop the `held_by_us and not bead_state_is_clean` commit
  branch (no-change claims never commit), extend the held-by-us notion to include `in_progress`-with-matching-assignee
  so a pre-marked bead prints a quiet retention line instead of "claim declined", and skip the publish when nothing was
  committed.
- `claim_bead_for_agent_launch` (`src/sase/axe/run_agent_runner_bead.py`): when the core mutation returns
  `changed=false`, succeed without committing or publishing. Keep the hard fail for the case where a real state change
  produced no commit.
- `release_bead_claim_for_agent` (`src/sase/bead/claims.py`): already a core-level no-op for non-claimed beads; ensure
  no commit or push happens on `changed=false`.
- `sase_chop_bead_claim_checks` (`src/sase/scripts/sase_chop_bead_claim_checks.py`): collapse the per-bead release and
  acquire loops into batched mutations under one store write lock producing at most one
  `chore(beads): reconcile bead claims` commit and one push per chop cycle, and zero commits when nothing changed.
- Tests: commit-count assertions — a redundant `sase bead update`/`close` produces no commit; a wait-claim or promotion
  against a pre-marked bead produces no commit and no warning; the chop batches N reconciliations into one commit.

## launch: Single-commit epic launch in sase bead work

Depends on core (epic-capable preclaim) and quiet (runners fall through silently). Reverses the eager-claim removal of
`202607/jit_epic_bead_work.md` at the commit layer while preserving its recovery boundaries; update any lifecycle tests
that still assert the JIT ownership split (`tests/test_bead/test_epic_jit_claim_integration.py`,
`tests/test_bead/test_cli_work_epic_lifecycle.py`, and friends).

- In `launch_epic_bead_work` (`src/sase/bead/cli_work_handler.py` and the `cli_work_from_plan*` flow): after worker
  identities are rendered and force-reuse rewriting is final, batch-preclaim every rendered phase bead with its worker's
  agent name plus the epic bead with the land agent's name, inside the same launch transaction as `mark_ready_to_work`.
  The assignee strings must be exactly the agent names those runners later pass to their claim calls, so the core
  idempotence checks match. Closed phases and phases not rendered as workers are untouched.
- Plan-file path: fold the preclaim and `mark_ready_to_work` into the existing
  `chore(beads): checkpoint approved epic graph <id>` commit and its synchronous pre-spawn push barrier
  (`publish_epic_graph_before_launch`), and retire the separate post-launch `commit_bead_work_launch` commit
  (`src/sase/bead/cli_work_commit.py`) for this path. Net: exactly one beads commit and one push per launch. A message
  rename to `chore(beads): launch epic <id>` is welcome but optional.
- Bead-id path (`sase bead work <epic-id>`): this path currently has no pre-spawn commit at all — the launch commit
  happens after spawning with a deferred push. Move the single consolidated commit (readiness + preclaim) and a
  synchronous push _before_ agent spawn, mirroring the plan-file barrier.
- Failure handling keeps the JIT boundaries: a zero-spawn failure restores the preclaim rollback payload and the
  readiness rollback in the existing single recovery commit (`rollback_work_launch`,
  `src/sase/bead/cli_work_cleanup.py`); a partial-spawn failure terminates spawned runners and leaves bead state alone
  (live runners now simply find their beads pre-marked); post-launch commit failures stay distinguishable from spawn
  failures. `--dry-run` mutates nothing.
- Rerun/resume: a rerun batch-reassigns every non-closed rendered phase (including still-in-progress ones) to the fresh
  agent names in the same single commit, preserving the existing retry semantics.
- Behavior shift to document: an agent that dies while waiting no longer flips its bead back to `open` (there is no
  wait-claim to release); recovery is a `sase bead work` rerun, which reassigns it. Update the statuses section of the
  beads reference skill and any prompt wording that promises wait-time `claimed` states for epic-launched work. The
  skill is generated from sources under `src/sase/xprompts/skills/` — read the `generated_skills.md` long-term memory
  (via `sase memory read`) before touching them.
- Tests: launching a mocked epic produces exactly one beads commit and one push before any spawn; simulated runner
  wait/promote/release against the pre-marked store produce zero additional commits; zero-spawn and partial-spawn
  failure paths; rerun reassignment; dry-run.

## detproj: Deterministic bead projection output

All in the sase-core linked repo. Evidence for this phase: beads-sidecar commit `d1e76a0` is a 348-line pure reorder of
`issues.jsonl` with no event change, committed under a recycled `chore(beads): close <id>` message two minutes after the
real close.

- Make `issues.jsonl` (and any other regenerated projection artifact, e.g. the events manifest if it shares the problem)
  byte-stable across every writer: the mutable-store save path, projection rebuild/maintenance APIs, JSONL export, and
  the DB-refresh path. Pick one canonical ordering (stable sort by issue id is the obvious choice) and apply it
  everywhere, so two clones that hold the same logical state always serialize identical bytes.
- Investigate which writer produces the divergent order today (mutation-order save vs. rebuild-from-events) before
  choosing, and cover the answer with a test that round-trips one store state through each write path and asserts byte
  equality.
- A one-time reordering diff per existing store on first write after upgrade is acceptable; it should land through
  whatever honest sync commit next touches the store, not require migration tooling.
- With quiet's no-commit-on-no-change rule, this phase eliminates the echo-commit class entirely: no writer creates
  reorder dirt, and no-op mutations no longer sweep dirt into semantic commits.

## closenote: Close-with-note in one mutation and one commit

Core work in the sase-core linked repo, CLI and prompt work here. 42 of the last 230 beads commits are `note` commits
immediately preceding a `close` by the same agent.

- Rust: extend the close mutation so an optional note entry is appended — attributed and formatted exactly like
  `append_issue_note`, same timestamp/actor conventions — to every listed bead in the same locked mutation before the
  close events, one save total. Update the PyO3 binding and the Rust-side CLI argument parsing
  (`crates/sase_core/src/bead/cli.rs`) so the fast path accepts the new option.
- Python: add `--note` (with a short alias; `-r`, `-R`, and `-f` are taken on `close`) to `sase bead close` in the
  argparse layer (`src/sase/main/parser_bead.py`) and the slow-path handler (`src/sase/bead/cli_crud.py`), keeping
  fast/slow paths in lockstep and the commit message unchanged (`chore(beads): close <ids>`). Follow the CLI rules
  long-term memory (`sase memory read cli_rules.md`): alphabetized options, short alias, quality help text.
- Update the runtime working-loop guidance so the standard finish sequence becomes one command:
  `sase bead close <id> --note "<what you verified>"`. Touch the phase-worker/land prompt contracts, commit
  instructions, and the beads reference skill sources under `src/sase/xprompts/skills/` — read the `generated_skills.md`
  long-term memory first. Keep `sase bead note` itself unchanged for mid-work progress notes.
- Tests: one commit for close-with-note (both CLI paths), note formatting parity with `sase bead note`, multi-id close
  applying the note to each listed bead, and `--note` with `--force`/`--resolution` combinations.

## postcommit: One beads commit per agent commit

Python only; independent of the other phases. Today every agent `sase commit` produces two beads-repo commits.

- Fold the bead-pages publication (`publish_committed_bead_pages`, `src/sase/bead_pages/publication.py`, invoked from
  `_run_agent_publication_step` in `src/sase/workflows/commit/workflow.py`) and the finalizer's
  `chore(beads): sync bead state` commit (`_auto_commit_separate_sdd_store_if_possible`,
  `src/sase/llm_provider/commit_finalizer.py`) into one beads commit per agent commit: render the pages first and let a
  single commit cover both, reusing the existing `store_git_write_lock` handoff (`already_locked=True`) so the span
  stays atomic. Pick one honest message covering both (e.g. `chore(beads): sync bead state and pages for <root_id>`).
- Keep the publication best-effort: a pages-rendering failure must not block the bead-state sync, matching today's
  behavior.
- The finalizer can re-run its sync per retry pass; guard the repeat passes with `bead_state_is_clean` (or the outcome
  of the first pass) so retries produce zero additional beads commits when nothing changed.
- Preserve the async push default; at most one push per consolidated commit.
- Tests: an agent commit flow produces exactly one beads commit including both state and pages; finalizer retries do not
  add commits; pages failure still syncs state.

## Validation

Every phase that changes files in this repo runs `just install` and then the mandatory `just check`. Core phases run the
sase-core test suite in the linked repo. Beyond unit tests, each phase adds commit-count assertions as described above —
the regression this epic guards against is measured in commits, so tests should count them. The land agent verifies the
end-to-end effect by launching a real epic and confirming the beads sidecar gains exactly one launch commit plus one
close commit per phase, then raises the published `sase-core-rs` floor in `pyproject.toml` once wheels carrying the core
phases are released.
