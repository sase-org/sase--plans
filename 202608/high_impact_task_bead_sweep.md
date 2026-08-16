---
tier: epic
title: High-impact task bead sweep
goal: 'Reconcile every ready sase task bead, retire stale recommendations with evidence,
  and complete the five live fixes with the largest durability, verification, responsiveness,
  and commit-workflow impact.

  '
phases:
- id: audit_ready_queue
  title: Audit and reconcile the ready task queue
  depends_on: []
  size: medium
  description: 'audit_ready_queue: Revalidate every non-in-progress task bead, close
    only evidence-backed stale items, and preserve the five selected tasks for implementation.'
- id: protect_bead_streams
  title: Protect append-only bead event streams
  depends_on:
  - audit_ready_queue
  size: medium
  description: 'protect_bead_streams: Fix sase-li by preventing publication or sync
    from shrinking event streams and by diagnosing the offending history precisely.'
- id: attribute_dirty_runs
  title: Exclude attributable dirty-tree failures from flake debt
  depends_on:
  - audit_ready_queue
  size: medium
  description: 'attribute_dirty_runs: Fix sase-lc by making reproducible-flake evidence
    distinguish dirty source-audit failures from shared master flakes.'
- id: cache_agent_page_links
  title: Bound agent page-link resolution latency
  depends_on:
  - audit_ready_queue
  size: medium
  description: 'cache_agent_page_links: Fix sase-lw with a correctly invalidated registry
    snapshot so TUI selections do not repeat a 400-800ms scan.'
- id: stabilize_publication_budget
  title: Stabilize the large publication backlog contract
  depends_on:
  - audit_ready_queue
  size: small
  description: 'stabilize_publication_budget: Fix sase-mb with a contention-resistant
    performance tripwire that retains the queue''s scaling contract.'
- id: bound_post_push_publication
  title: Bound post-push agent publication
  depends_on:
  - stabilize_publication_budget
  size: medium
  description: 'bound_post_push_publication: Fix sase-mh so a stalled agent-page render
    cannot indefinitely block commit finalization while durable publication retry
    remains intact.'
- id: verify_and_reconcile
  title: Verify the combined tree and reconcile task beads
  depends_on:
  - protect_bead_streams
  - attribute_dirty_runs
  - cache_agent_page_links
  - stabilize_publication_budget
  - bound_post_push_publication
  size: medium
  description: 'verify_and_reconcile: Run combined verification, leave a concise outcome
    note on each selected task, and close every task whose acceptance criteria pass.'
proposed_by: bbugyi200.athena.02y
create_time: 2026-08-15 20:00:30
status: wip
bead_id: sase-mi
---

- **BEAD:** [sase-mi](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mi/README.md)

# Plan: High-impact task bead sweep

## Scope and ranking

At planning time, the `sase` project has 19 task beads that are active but not in
progress; all 19 are `ready`:

`sase-dc`, `sase-jq`, `sase-ke`, `sase-kh`, `sase-lc`, `sase-li`, `sase-lm`, `sase-ln`,
`sase-lw`, `sase-m0`, `sase-m1`, `sase-m2`, `sase-m3`, `sase-m7`, `sase-m8`, `sase-ma`,
`sase-mb`, `sase-md`, and `sase-mh`.

The five selected tasks are:

1. `sase-li` — a concurrent bead-sidecar publication deleted an append-only event,
   corrupting canonical history and wedging every later sync.
2. `sase-lc` — dirty workspace audit failures can become shared flake debt and block
   unrelated agents' required verification for the retention window.
3. `sase-lw` — selecting a page-publishing agent repeatedly refreshes the entire name
   registry, adding a measured 400-800ms to an interactive TUI path.
4. `sase-mb` — a host-sensitive one-second assertion fails required default-branch CI
   even when all functional large-backlog publication invariants hold.
5. `sase-mh` — post-push agent-page rendering can hold the publication lock and leave
   the commit finalizer stuck after the primary commit is already safely pushed.

This ranking favors irreversible data loss first, then cross-workspace verification
availability, repeated user-visible latency, default-branch CI reliability, and a commit
workflow that can hang after success. The publication tasks are separate but serialized
because both touch the queue transaction and its tests.

## Phase 1: Audit and reconcile the ready task queue

Capture a fresh JSON listing before any bead mutation. Review all task beads that are
still `open`, `ready`, or `snoozed`, excluding anything that became `claimed` or
`in_progress` after this plan was proposed. Compare each description, notes, +1
evidence, dependencies, current code, recent commits, and a focused reproduction where
needed. Do not treat age or a passing isolated run as sufficient evidence that a flake
is irrelevant.

Use a small audit matrix with one row per bead and these outcomes: selected here, still
relevant backlog, fixed by a named commit, superseded by a named active owner, or no
longer reproducible on current `master`. The following are strong stale candidates, but
each must be verified rather than closed mechanically:

- `sase-jq` and `sase-ke`: commit `5601920c9` added their explicitly permitted
  reproducible-flake baseline entries, and the current flake gate reports no new
  reproducible flakes.
- `sase-lm`: its own later note reports that the renamed procs suites pass and that the
  linked-core version window is expected release-branch behavior.
- `sase-m7`: commit `2c9f2b7fa` added ambient color isolation; verify on current HEAD
  with forced-color environment variables because the evidence that reopened the bead
  came from a worker tree that may have predated that commit.
- `sase-ma`: commit `28da68d4e` redesigned the Models panel and regenerated its focused
  PNG goldens; run the named snapshot before deciding whether that supersedes the old
  111-pixel drift.
- `sase-md`: the same config-center node is already baseline-owned and actively scoped
  by epic `sase-j7`; close only if that ownership genuinely supersedes this duplicate.
- `sase-m2`: its description says the failure was deterministic, while later full suites
  on master passed; rerun the exact node before deciding that the report is stale.

Retain live backlog items, including post-close reproductions such as `sase-dc`, even
when they are not among the five selected. For every task that is demonstrably no longer
relevant, append an attributed note that names the evidence and close it with the
semantically correct resolution: `done` for work already delivered, `superseded` for an
exact active owner, or `canceled` for an obsolete/non-reproducible report. Use
`sase bead close`, never a hand-edited bead store. Do not close the five selected tasks
in this phase.

If checking `sase-m8` requires reading the linked `chezmoi` repository, open it through
`/sase_repo` and use only the returned checkout path. The audit should not modify any
non-selected backlog implementation.

## Phase 2: Protect append-only bead event streams (`sase-li`)

Reconstruct the shrink scenario around `publish_committed_bead_pages`,
`commit_sdd_store_files`, the managed sync worker, and semantic conflict resolution. Add
a regression in the existing temporary Git-repository fixtures that starts with an
event-stream prefix, simulates stale page publication or a concurrent integration, and
proves that no generated "sync bead state and pages" commit can delete or rewrite a base
event.

Put the invariant at the mutation boundary that can prevent bad history, not only in the
later conflict resolver. Before committing or pushing a bead root, compare every changed
`events/streams/*.jsonl` file with the relevant committed/remote ancestor and reject a
shrink or prefix rewrite while preserving the valid local superset. Keep the existing
relocation behavior for legitimate duplicate-ID append conflicts. Ensure a failed guard
restores or leaves a recoverable worktree and records an actionable sync error instead
of publishing corruption.

Extend bead sync diagnostics so the historical-corruption case identifies the stream,
the missing event range or first rewritten event, and the first offending commit when
Git history can resolve it. Diagnostics must remain read-only and degrade cleanly when
history is shallow or unavailable. Cover ordinary append, concurrent independent
append/relocation, rejected shrink, exact state restoration, and diagnostic wording in
`tests/test_bead/`.

Run the focused conflict, page-publication, sync-worker, recovery, and doctor/diagnostic
tests, then `just check`. Append a result note to `sase-li`; close it as `done` only
when the corruption regression and diagnostics pass. Otherwise leave it ready with a
note naming the remaining blocker and evidence.

## Phase 3: Exclude attributable dirty-tree failures (`sase-lc`)

Change the full-run evidence model and flake correlation in
`tests/_test_selection_health_records.py`,
`tests/_test_selection_health_correlation.py`, and `tools/selection_health` so a test
whose result is directly attributable to the dirty tree is not promoted into shared
reproducible-flake debt. Prefer explicit recorded tree cleanliness plus deterministic
source-audit ownership metadata over filename heuristics. Preserve conservative behavior
for legacy records with missing identity: unknown data must not be silently declared
clean.

At minimum, encode the two records from the bead as a regression: changes to source
files scanned by the marker path/mutation audit may fail that audit in the editing
workspace, but those failures do not count toward the cross-workspace flake threshold.
Also prove that disjoint dirty changes, clean-tree intermittent failures, stale node
IDs, commit ordering, maximum-failure filtering, and independent-pass requirements
retain their current behavior. Make the human report explain how many failures were
excluded and why so the gate remains auditable.

Run `tests/test_selection_health_tool.py` and the focused selection-health unit tests,
then run `tools/selection_health --fail-on-new-flake` against the durable store and
`just check`. Note the exact rule and verification on `sase-lc`, closing only after the
synthetic poison records are excluded without hiding genuine flakes.

## Phase 4: Bound agent page-link resolution latency (`sase-lw`)

Keep the TUI render/selection path free of filesystem scans. Replace the unconditional
`snapshot_agent_name_registry()` in `src/sase/ace/tui/models/agent_page_url.py` with a
cache owned at an appropriate shared resolver or adapter boundary. Key it by the
resolved store/project/primary root and an explicit freshness token or bounded TTL;
provide deterministic invalidation for tests and for mutations that change reserved
family names. Do not use a per-keypress `stat`, glob, subprocess, or unbounded lock, and
do not serve one project's registry to another.

Expand `tests/ace/tui/widgets/test_agent_page_url.py` to prove repeated selections reuse
the snapshot, invalidation refreshes it, exceptions remain best-effort, and project/root
isolation holds. Re-run the detail-header benchmark or equivalent instrumented fixture
before and after: warm page-link resolution should be orders of magnitude below the
reported 400-800ms scan and must not move synchronous I/O onto the Textual event loop.

Run the focused URL/header-summary tests and benchmark, then `just check`. Record the
measured before/after behavior on `sase-lw` and close it only when cache correctness and
latency criteria both pass.

## Phase 5: Stabilize the large publication backlog contract (`sase-mb`)

Profile `test_large_backlog_builds_one_inventory_and_publishes_each_hood_once` enough to
separate algorithmic work from host scheduling noise. Preserve its functional scaling
assertions: one integration, one inventory build, one publish per hood, grouped updates,
and one acknowledgement of all 2,000 requests.

Replace the brittle single-sample `perf_counter() < 1.0` assertion with a
contention-resistant regression contract. Prefer deterministic operation/allocation or
complexity bounds; if wall time remains useful, isolate it as a marked performance test,
calibrate it from repository conventions, and use repeated robust statistics rather than
one hard CI sample. Do not remove the performance tripwire or weaken the functional
workload.

Run the exact node repeatedly in isolation and through `just test-contention`, then the
agents-sync publication tests and `just check`. Append the design and evidence to
`sase-mb` and close it only if the loaded lane no longer fails solely on scheduler delay
while an actual scaling regression still trips the test.

## Phase 6: Bound post-push agent publication (`sase-mh`)

Use a controlled slow/blocking publication hook to reproduce the unbounded region from
`publish_committed_agent_hood` through `_drain_agent_publications`,
`publish_queued_transaction`, and page/neighbor rendering. Confirm whether the delay is
data-scaled rendering, filesystem I/O, or the architectural fact that the finalizer
synchronously drains durable auxiliary work while holding `sase-agents-sync.lock`.

Make commit completion bounded after the durable outbox enqueue. A stalled render must
return a clear queued/retry outcome, preserve the already-pushed primary commit, retain
the publication request for `sase agent sync`, and release the shared lock without
acknowledging incomplete work. Use an existing supervised/durable execution boundary
where available; do not add an untracked fire-and-forget thread or abandon a process
that can continue mutating the sidecar after its lock is released. Keep successful fast
publication behavior and quarantine/retirement semantics intact.

Add regression coverage for a blocked render, lock release, outbox retention, later
successful retry, and normal prompt/archive publication. Re-run the large-backlog test
from the preceding phase to ensure the new boundary does not reintroduce per-request or
per-hood inventory work. Run the focused commit-finalizer and agents-sync suites, then
`just check`. Note the new bound and retry behavior on `sase-mh`; close only when the
controlled stall cannot wedge finalization.

## Phase 7: Verify and reconcile

Rebase/reconcile the combined tree, rerun `just install`, and run focused tests for all
five fixes. Because this epic touches broad test-selection, bead persistence, TUI hot
paths, and commit-publication infrastructure, run `just check-full` only through
`/sase_monitor`, with a `--next` action that triages the result and resumes this
closeout. Run `just test-visual` only if the combined diff changes rendered TUI output
or snapshots.

Re-list the five selected task beads before updating them. For each one, append a brief
note naming the implemented behavior, important files or mechanism, and verification
result. If the acceptance criteria pass, close it with `sase bead close <id> --note

<summary>` and resolution `done`. If work is incomplete, leave it open and append a
concise justification with the blocker, attempted approach, and next concrete action.
Never close a bead solely because the combined verification budget expired.

Finally, list all active non-in-progress task beads again and report: the original audit
count, every stale bead closed with resolution/reason, the five selected outcomes,
verification commands/results, and any still-live backlog that was intentionally left
untouched.
