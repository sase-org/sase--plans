---
tier: epic
title: Claimed status survives the epic-graph publication lag
goal: 'Every bead owned by a live pre-launch SASE agent reaches `claimed` and stays
  there for the whole wait, even when the canonical bead store has not yet integrated
  the freshly published epic graph.

  '
phases:
- id: claim-recovery
  title: Claim retry and project-scoped store recovery
  depends_on: []
  size: medium
  description: '''Claim retry and project-scoped store recovery'' section: add a project-scoped
    read-recovery bead-store refresh and rebuild the waiting-agent claim around a
    bounded retry budget that survives a not-yet-integrated store and bead-mutation
    lock contention.'
- id: claim-marker
  title: Durable held-claim artifact marker
  depends_on: []
  size: small
  description: '''Durable held-claim artifact marker'' section: record a held claim
    in a dedicated `bead_claim.json` artifact file, write and clear it from the runner,
    and make the shutdown release funnel read it instead of depending only on runner
    process memory.'
- id: chop-acquire
  title: Bidirectional claim reconciler
  depends_on:
  - claim-recovery
  - claim-marker
  size: medium
  description: '''Bidirectional claim reconciler'' section: give the bead_claim_checks
    chop an acquire pass that claims beads for live unpromoted unmarked agents, keeping
    the zero-candidate tick free of bead-store reads and the release pass''s ordering
    invariant intact.'
- id: verify-and-document
  title: End-to-end verification and guidance
  depends_on:
  - chop-acquire
  size: small
  description: '''End-to-end verification and guidance'' section: add the integration
    test that reproduces the original failure, add the inverse doctor advisory for
    unclaimed live pre-launch agents, and document that claims are acquired best-effort
    and reconciled.'
create_time: 2026-07-25 07:36:44
status: done
bead_id: sase-94
---

- **PROMPT:** [prompts/202607/claimed_bead_publication_race.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/claimed_bead_publication_race.md)
- **BEAD:** [sase-94](https://github.com/sase-org/sase--beads/blob/main/pages/sase-94/README.md)

# Plan: Claimed status survives the epic-graph publication lag

## Goal

Every bead owned by a live pre-launch SASE agent reaches `claimed` and stays there for the whole wait, even when the
canonical bead store has not yet integrated the freshly published epic graph, so `sase bead list` on a running epic
shows `◎` for every phase whose agent is alive and waiting.

## Problem

`sase-8y` shipped the `claimed` status, and its headline use case — "launch `sase bead work` on a ten-phase epic and
nine beads sit at `open` while nine agent processes are already alive" — still fails 100% of the time. Every epic
launched since the feature landed shows the same failure.

The evidence, from the `sase-92` launch on 2026-07-25:

```
07:06:25  chore(beads): checkpoint approved epic graph sase-92   (launcher's store, then pushed)
07:06:29  runner sase-92.1  → no %w, has_wait=False → no wait claim, straight to in_progress
07:06:30  runner sase-92.3  → no %w, has_wait=False → no wait claim, straight to in_progress
07:06:31  runner sase-92.2  → Warning: Failed to claim bead 'sase-92.2' ... 'Issue not found: sase-92.2'
07:06:32  runner sase-92.4  → Warning: Failed to claim bead 'sase-92.4' ... 'Issue not found: sase-92.4'
07:06:33  runner sase-92.5  → Warning: Failed to claim bead 'sase-92.5' ... 'Issue not found: sase-92.5'
07:06:34  runner sase-92.6  → Warning: Failed to claim bead 'sase-92.6' ... 'Issue not found: sase-92.6'
07:06:35  runner sase-92.land → Warning: Failed to claim bead 'sase-92' ... 'Issue not found: sase-92'
07:07:58  the canonical plans clone finally rebases onto the published graph
```

### Root cause

`_checkpoint_and_publish_graph` in `src/sase/bead/cli_work_from_plan.py` commits the epic graph into the **launch
context's** bead store and publishes it to the sidecar **remote** (`publish_epic_graph_before_launch`), then spawns
detached runners immediately. Workers that get a fresh workspace clone the sidecar from that remote and see the graph.

But `claim_bead_for_waiting_agent` in `src/sase/bead/claims.py` deliberately does not use a workspace clone — a waiting
agent has no workspace yet — so it resolves `canonical_beads_dir_for_project(project_name)`, which is the primary
workspace's long-lived plans checkout. That checkout integrates from the remote only on its own periodic cadence
(observed: every few minutes). Seconds after publication it does not contain the new epic, so the compare-and-swap
raises `Issue not found`, `claims.py` swallows it into a warning per design decision 4 of `sase-8y`, and **nothing ever
retries**.

The only thing that retries today is the post-wait runner re-exec (`refresh_runner_code_after_wait`), which re-runs the
claim milliseconds before `claim_bead_for_agent_launch` promotes the bead to `in_progress`. That is why `sase-90` and
`sase-91` logs contain a successful `Claimed bead ...` line after their failure: the claim did eventually land, with a
`claimed` window of roughly zero seconds, and only because the runner code happened to change during the wait. When the
runner code is unchanged there is no second attempt at all.

### Second failure mode

The same launches show a distinct error with the same symptom:

```
Warning: Failed to claim bead 'sase-91.5' for waiting agent 'sase-91.5': lock_timeout: timed out after 2000ms
waiting for bead mutation lock .../plans/beads/beads.db
```

When an epic spawns many phases at once they contend on the 2000 ms bead mutation lock, and the loser's claim is lost
permanently for exactly the same reason: one attempt, no retry.

### What is _not_ broken

`sase-92.1` and `sase-92.3` were the phases with no unmet dependency. They launched with no `%w` directive, so
`has_wait` was false, the wait-claim path was correctly skipped, and they went straight to `in_progress` through the
pre-existing just-in-time launch claim. That is the designed behavior and this epic does not change it.

## Design decisions

These decisions are settled; phase agents implement them rather than relitigating them.

### 1. Fix the claim, not the launch ordering

`sase bead work` already crosses a real visibility barrier: it commits the graph and pushes it before spawning anything,
and treats publication failure as fatal. The defect is entirely on the read side — the claim targets a clone that is
knowingly behind the remote. No phase may add a blocking wait, a store poll, or an extra barrier to
`_checkpoint_and_publish_graph`, `publish_epic_graph_before_launch`, or the spawn loop.

### 2. Read-recovery refresh, triggered by the miss, not by every claim

`sase/bead/sync.py` already has the exact primitive this needs: `refresh_current_bead_store()` synchronously integrates
a remote-backed store via `integrate_sdd_repository` and deliberately stops after fetch/rebase without pushing.
`sase_clan_summary_epic.py` already uses it to recover from precisely this staleness. The claim path gets a
project-scoped sibling of that helper, invoked **only after a claim attempt misses**, never speculatively. A claim that
finds its bead on the first try must not pay for any git work.

### 3. Claiming still must never fail or delay an agent launch

Decision 4 of `sase-8y` stands unchanged and is not weakened by adding retries. Every new code path lives inside the
existing catch-all in `claims.py`, degrades to a printed warning, and returns "no claim held". The retry budget is
bounded and small enough that a totally unreachable remote costs a waiting agent a few seconds, not minutes. The
backstop in decision 5 is what makes an aggressive in-process budget unnecessary.

### 4. A durable claim marker file, not a field in `agent_meta.json`

The promotion flag `bead_claim_promoted` lives in `agent_meta.json` because only the runner writes it, from its own
in-memory `agent_meta` dict. A _held-claim_ flag cannot live there: the reconciler chop needs to write it too, and the
runner rewrites `agent_meta.json` wholesale from memory at several later points, which would silently clobber an
externally written key.

So the held claim is recorded in its own artifact file, `bead_claim.json`, in the agent's artifacts directory, written
by whichever actor acquires the claim and read by both the runner's shutdown funnel and the chop. This is what keeps the
chop's steady-state cost at zero: a waiting agent whose claim already succeeded is filtered out by a file-existence
check, without opening a bead store.

### 5. The reconciler becomes bidirectional

`bead_claim_checks` today only releases claims held by dead pre-launch agents. It gains a symmetric _acquire_ pass: a
live, unpromoted, claim-bearing agent with no `bead_claim.json` is an agent whose claim failed, so the chop retries it.
That single addition makes the whole feature self-healing — it covers the publication lag, lock contention, a runner
that crashed between writing its metadata and writing its claim, and any future launch path that forgets to claim — and
it repairs beads that are already stranded, within one `waits` interval of the chop landing.

The compare-and-swap is what makes the acquire pass safe: it succeeds only from `open`, so a bead that has already been
promoted to `in_progress`, closed, or claimed by someone else is never touched.

### 6. No change to `has_wait` gating, to `in_progress`, or to `sase bead work`

Decisions 5 and 6 of `sase-8y` are unchanged. A straight-through launch with no wait still skips the claim entirely.
`sase bead work` still decides purely from agent liveness and reads no bead status other than "not closed". No phase may
add a `Status::Claimed` check to `work.rs`, `cli_work*.py`, or the force-reuse preview.

### Rejected alternatives

- **Claim from the launcher at graph-checkpoint time.** The launcher would have to write claims for agent names that do
  not exist yet: final names are resolved by the runner from `%id` plus a registry reservation, and the observed
  `Ignoring stale SASE_AGENT_PLANNED_NAME=...` log line shows that binding is not reliable enough to base a persisted
  assignee on. Worse, it would create claims with no artifact directory to prove liveness, which is exactly the
  "unresolvable owner" case the reconciler is forbidden to touch and `sase doctor` can only complain about.
- **Make `canonical_beads_dir_for_project` always integrate before returning.** It is on every cross-project bead read
  surface, including the mobile helper and the ACE plans pane. Putting a fetch there would be a large, unbounded latency
  regression for a problem that only exists at claim time.
- **Block `sase bead work` until the canonical checkout has integrated the published graph.** The launcher does not own
  that checkout, this would serialize every epic launch behind a git fetch of someone else's clone, and it would still
  not fix lock-timeout losses or a claim that dies mid-flight.
- **Widen the 2000 ms bead mutation lock timeout.** It treats a symptom, and a longer blocking lock in nine concurrent
  agent startups is worse than a bounded retry plus a backstop.

## Verification

The epic is done when, on a freshly launched multi-phase epic:

- every phase agent that is alive and waiting shows `◎ ... CLAIMED` in `sase bead list` within one `waits` interval of
  launch, and stays there for the whole wait rather than for milliseconds before promotion;
- the epic's own bead is `claimed` by its `.land` agent for the duration of the epic;
- no `Failed to claim bead` warning appears in any runner log for the launch;
- killing a waiting phase agent still reopens its bead, and a promoted claim is still never released.

## Phase claim-recovery: Claim retry and project-scoped store recovery

Make the in-process claim survive a store that has not yet integrated the published graph.

In `src/sase/bead/sync.py`, add a project-scoped read-recovery integration next to `refresh_current_bead_store`, e.g.
`refresh_bead_store(beads_dir: Path) -> None`, factored out of the existing function so both share one implementation.
It must keep the current semantics exactly: no-op for in-tree and non-remote stores, `integrate_sdd_repository` with
`op_prefix="bead.refresh"`, fetch/rebase only, never push, raise `_BeadStoreRefreshError` on failure. Refactor
`refresh_current_bead_store` to delegate to it so there is one code path, and leave its behavior unchanged for its
existing caller `src/sase/scripts/sase_clan_summary_epic.py`.

In `src/sase/bead/claims.py`, restructure `claim_bead_for_waiting_agent` around a bounded attempt loop:

- Attempt the compare-and-swap. On success (claimed, or already ours) return as today — no git work, no behavior change,
  no extra latency for the common case.
- On a _recoverable_ miss — the bead is not found in the store, or the mutation failed with a lock timeout — refresh the
  store through the new helper and retry. Distinguish the two: a not-found miss justifies a refresh, a lock timeout
  justifies only a short jittered sleep before retrying, since the store is already current and the contention is local.
- Bound the whole thing tightly: at most three attempts and at most one refresh, with the total added cost on the
  failure path staying in the low seconds. A bead that declines for a legitimate reason (`in_progress`, `closed`,
  claimed by another agent) is not a miss and must not trigger a retry or a refresh.
- Keep the outermost catch-all: any exception still degrades to the existing warning and `False`. Startup is never
  blocked and never fails.

Concurrency note for the implementer: when an epic spawns many phases at once they will all miss and all try to refresh
the same checkout. Use the existing integration freshness marker (`_integration_is_fresh` / `mark_bead_integration` in
`sase/bead/sync.py`, or whatever `integrate_sdd_repository` already honors) so a burst of agents does not turn into a
burst of concurrent fetch/rebase runs on one clone. Serialize or skip redundant refreshes rather than adding a new lock.

Tests: a claim that hits on the first attempt performs no refresh; a not-found miss refreshes once and then succeeds; a
lock-timeout miss retries without refreshing and then succeeds; a legitimate decline neither refreshes nor retries; an
exhausted budget still returns `False` with a warning and raises nothing; the refresh helper is a no-op for in-tree
stores. Run `just install` before `just check`, since workspace virtualenvs go stale.

## Phase claim-marker: Durable held-claim artifact marker

Introduce the artifact-level record of a held claim, so that both the shutdown funnel and the reconciler agree on who
holds what without depending on runner process memory.

Define `bead_claim.json` in the agent's artifacts directory with a small, explicit payload — at least the bead id, the
owning agent name, and the project name — plus tiny read/write/clear helpers. Put them where the other artifact-marker
helpers live so the chop can import them without pulling in runner internals; `src/sase/bead/claims.py` is a reasonable
home given both writers already depend on it.

Wire the runner in `src/sase/axe/run_agent_runner.py`:

- When `claim_bead_for_waiting_agent` returns `True`, write `bead_claim.json` alongside setting the existing
  `held_bead_claim` local. A failure to write the marker must warn, not raise.
- At the promotion site (around the `agent_meta["bead_claim_promoted"] = True` write that follows
  `claim_bead_for_agent_launch`), clear the marker in the same step that clears `held_bead_claim`. Order it so the
  marker is gone before execution begins, mirroring the durability requirement already stated for `bead_claim_promoted`.

Wire the shutdown funnel in `src/sase/axe/run_agent_runner_lifecycle.py`: `finalize_runner_shutdown` must release a
claim recorded in `bead_claim.json` even when the in-memory `RunnerShutdownState` fields are empty, because after the
next phase the chop can acquire a claim on a runner's behalf and that runner will have no in-memory record of it. Keep
the existing `has_pending_handoff(artifacts_dir)` suppression and the existing "never release a promoted claim" rule
exactly as they are, and keep failures non-fatal.

Tests: the marker is written on a successful wait claim and absent after promotion; a killed pre-launch runner with only
the marker (no in-memory state) still reopens its bead; a promoted agent is not released at shutdown; a pending handoff
still suppresses the release; an unwritable or corrupt marker degrades to a warning. Finish with `just check`.

## Phase chop-acquire: Bidirectional claim reconciler

Depends on the marker contract from `claim-marker` and on the recovery path from `claim-recovery`.

Extend `src/sase/scripts/sase_chop_bead_claim_checks.py` with an acquire pass that runs alongside the existing release
pass within the same chop invocation.

Acquire-pass candidates are agent records that satisfy all of: a `bead_id` in `agent_meta.json`; `bead_claim_promoted`
absent or false; no `bead_claim.json` marker; and a live process per `is_process_alive`. For each candidate, call
`claim_bead_for_waiting_agent` with the record's project, bead, and agent name, and on success write the
`bead_claim.json` marker so the candidate is filtered out on every later tick.

Cost is the constraint that shapes this pass, and the marker check is what controls it. Extend the existing cheap
pre-pass so that a tick with neither a dead unpromoted claim nor an unmarked live claim-bearing agent still returns its
zero summary without opening a single bead store — in the steady state, every waiting agent already holds a marker, so
the acquire candidate set is empty and the chop stays as cheap as it is today. Read `sase/memory/tui_perf.md` through
`/sase_memory_read` before changing the scan, and do not add a second full artifact scan; reuse `_scan_claim_artifacts`.

Two invariants the implementation must preserve:

- The release pass keeps its ordering invariant: bead stores are read before the authoritative artifact re-scan. The
  acquire pass has no such hazard, since it only ever performs a compare-and-swap that succeeds from `open`, but it must
  not be interleaved in a way that weakens the release pass's ordering.
- An agent that dies immediately after the chop acquires its claim is already covered by the release pass on the next
  tick, because the record is then dead, unpromoted, and the assignee resolves to it. Add a test for that handoff.

Update the chop's registered description in `src/sase/default_config.yml` (and its docstring) so it no longer claims to
only release: it now reconciles claims in both directions.

Tests: a live unpromoted unmarked agent gets its bead claimed and marked; a marked agent is skipped without a store
read; a promoted agent is never claimed; a bead already `in_progress` or `closed` declines without mutation; a dead
agent is not acquired; acquire-then-die is released on the following tick; and the zero-candidate tick opens no store.
Finish with `just check`.

## Phase verify-and-document: End-to-end verification and guidance

Depends on `chop-acquire`.

Prove the epic's goal end to end and record the behavior.

Add an integration-level test that reproduces the original failure and asserts the fix: a bead store that does not yet
contain the target bead, a waiting agent that claims against it, the store becoming current, and the bead reaching
`claimed` and _staying_ claimed until promotion — not a claim that lands milliseconds before `in_progress`. Extend
`tests/test_bead/test_epic_jit_claim_integration.py` rather than starting a parallel harness if it fits there.

Extend the `sase doctor` bead checks in `src/sase/doctor/checks_beads.py` with the inverse of the advisory that
`sase-8y` added: report a live, unpromoted, claim-bearing agent whose bead is **not** `claimed`, since after this epic
that state means the reconciler is not running or is failing. Like the existing advisory it must never mutate.

Update the documentation that describes the claim lifecycle — the bead docs, the `sase_beads` skill source under
`src/sase/xprompts/skills/`, and any bead xprompt wording that explains `claimed` — so they state that the claim is
acquired best-effort at wait time and reconciled by `bead_claim_checks`, and that a bead can therefore become `claimed`
a few seconds after its agent starts waiting rather than instantly. Regenerate any generated skill artifacts per
`sase/memory/generated_skills.md`, read through `/sase_memory_read`.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims in this phase; that requires
explicit user permission that this plan does not grant.

Finish with `just check`.
