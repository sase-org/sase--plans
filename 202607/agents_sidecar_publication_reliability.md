---
tier: tale
title: Make automatic agents-sidecar publication isolated, truthful, and bounded
goal:
  Every successful primary commit is either proven in a pushed complete hood snapshot or
  remains individually retryable, without malformed legacy records or repeated scans
  blocking unrelated sidecar publication.
create_time: 2026-09-09 19:52:59
status: wip
---

# Plan: Make automatic agents-sidecar publication isolated, truthful, and bounded

## Goal

Restore the `sase-8v` contract that every successful primary-repository `sase commit`
creates a durable obligation to publish the committing agent's complete project-scoped
top-level hood to `<project>--agents`, and that the obligation is retired only after the
pushed sidecar snapshot demonstrably contains that global agent and primary revision.

The repair must drain the existing backlog without rewriting historical artifacts,
prevent one malformed legacy record or one failed hood from blocking unrelated
publications, and remove the repeated project scans and registry rebuilds that make the
shared sync lock last for minutes. It must preserve strict v2 transport validation,
full-hood publication, foreign import receipts, non-fast-forward recovery, bounded lock
contention, and the cached-versus-full-sync behavior already delivered by `sase-8v`.

This is a tale because the inventory boundary, outbox drain, import batching, and tests
form one coupled correctness transaction in the primary SASE repository. Splitting them
would either leave the queue vulnerable to false acknowledgement or leave the live
poison/lock convoy in place.

## Authoritative expectations and diagnosis

Use these durable sources as the acceptance baseline:

- `sase/repos/plans/202607/prompts/global_agent_hoods.md` is the original epic prompt.
  It explicitly requires all necessary agent data to be pushed automatically when
  `sase commit` runs, complete recursive top-level hood/family data, remote family
  revival, linked global `SASE_AGENT` provenance, and an explicit full-sync path.
- `sase/repos/plans/202607/global_agent_hoods.md` and epic bead `sase-8v` settle the
  stronger implementation contract: auxiliary failure leaves the primary commit
  successful and durably queued; the next commit or full sync retries it; exact hood
  closure is project-bounded; malformed input is validated/quarantined; and lock
  contention is bounded.
- `~/.sase/chats/202607/gh_sase_org__sase-ace_run-jw_cdx-260724_191541.md` records the
  repeated full-project inventory scans and long-held sync lock.
- `~/.sase/chats/202607/gh_sase_org__sase-ace_run-jw_cld-260724_191540.md` records the
  head-of-line poison request, invalid legacy name, per-import registry rebuilds, and
  two-second lock starvation.
- Phase bead `sase-8v.3` already documents that legacy fanout spelling `4x--epic.f-0`
  cannot be globalized by the current strict family grammar and must remain locally
  readable with `canonical_global_name: null`. Publication must honor that compatibility
  boundary rather than attempting to put the legacy spelling into strict v2 data.

The observed failure chain is:

1. Automatic publication is triggered: primary commits are being appended to
   `~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json`.
2. The oldest request is for hood `toobig-r`, but the project-wide inventory also
   contains historical local artifact `4x--epic.f-0`. The inventory admits the legacy
   name and later applies strict hood/family parsing while selecting or rendering a
   hood, so an unrelated legacy record can abort `toobig-r`.
3. `_prepare_publications()` processes requests in creation order and returns on the
   first exception. `_record_failure()` then copies that one error and an attempt
   increment onto every queued request. No prepared successes are committed or pushed,
   and no request can pass the head item.
4. At investigation time the sidecar was still at `174c61f` from 14:21 while 30 primary
   revisions, beginning at 15:44, were queued; all 30 carried the same `4x--epic.f-0`
   error even though most belong to unrelated hoods.
5. Each queued request independently rebuilds the same project inventory. The inbound
   legacy-v1 pass also repeatedly scans artifacts and calls a one-name registry mutation
   whose staleness check can rebuild the approximately 4,800-entry registry after each
   artifact mutation. This work occurs while holding the exclusive sidecar sync lock, so
   concurrent commits time out, enqueue again, and enlarge the next drain.

The malformed artifact is the initiating correctness bug; repeated inventory/import work
and queue-wide error handling turn it into a multi-hour outage. Raising the lock timeout
alone is not a fix and would violate the bounded-wait intent.

## Settled repair contracts

### Durable publication is per request, while work may be coalesced per hood

Keep one outbox obligation per `(global_agent, primary_revision)`. A drain may group
multiple requests for the same local hood and render that hood once, but it may
acknowledge each request only after the final local sidecar checkout that was
successfully pushed (or was already identical to upstream) contains:

- the expected owner and project manifest entry;
- a validated snapshot for the request's exact local hood;
- a run with the request's exact `global_agent`; and
- that run's commit association for the exact `primary_revision`.

Merely completing a full sync, finding a hood manifest entry, computing a digest, or
successfully publishing another member of the hood is insufficient. Requests enqueued
concurrently after a drain captures its input are left for the next pass unless they
independently satisfy the same proof.

### Legacy incompatibility is quarantined at the inventory boundary

Locally owned historical records remain untouched and resolvable by the registry. Before
an `InventoryRun` becomes eligible for v2 publication, validate its run identity and
declared family/clan/relationship names through the existing owner-aware Rust facade. An
unrepresentable legacy record becomes a bounded diagnostic/quarantine entry rather than
an exception encountered while iterating every hood.

Strict v2 syntax is not relaxed, and the historical `4x--epic.f-0` artifact is not
renamed, deleted, or hand-edited. Valid unrelated hoods continue. A request whose own
agent cannot be represented remains individually queued with a specific diagnostic and
is never falsely acknowledged; the rest of its valid hood may still be refreshed without
claiming that request succeeded.

### Failures are isolated and successes remain transactional

Prepare each distinct requested hood independently from one immutable project inventory.
A format error for one hood records an error only on requests for that hood and does not
stop later hoods. Commit and push all successfully prepared hoods as one deterministic
sidecar transaction. If the Git commit or push fails, acknowledge none of those newly
prepared requests. On a non-fast-forward retry, re-pull and deterministically reapply
the same captured inventory and hood set without rescanning the project; re-evaluate
satisfaction after the retry push.

The targeted `sase commit` path and explicit `sase agent sync` path must share the same
satisfaction/acknowledgement helper. A generally successful full sync must not clear an
outbox item that its published snapshots do not prove.

### Expensive preparation is bounded and shared

Build the local project inventory at most once per drain and pass it into every targeted
hood publish and Git retry. Group requests by hood before rendering. Preserve the
configured bounded lock-acquisition timeout; reduce the critical section instead of
extending the wait.

For inbound integration:

- read receipts once and skip v2 packages or legacy `(machine, hood)` groups whose
  validated digest already matches a durable receipt;
- build the local legacy ownership/import lookup once per project pass instead of
  rescanning all artifacts per manifest entry;
- batch accepted legacy-v1 registry claims and persist the registry once after the
  corresponding artifact batch, so the final source signature reflects the final tree
  and a changed group cannot trigger one full rebuild per bundle;
- quarantine a colliding or invalid legacy group without aborting other groups, and
  write receipts only for groups actually integrated or proven unchanged.

Read-only discovery, local inventory construction, and reusable lookup construction
should occur outside the exclusive sidecar mutation lock where their inputs allow it.
Pull/rebase, receipt advancement, imported artifact/registry mutation, sidecar
rendering, commit, and push remain serialized. The Admin Center action continues to
invoke this synchronous engine only through its existing tracked background task; no
disk, Git, registry, or JSON work moves onto Textual's event loop or serial message
pump.

## Implementation

### 1. Make publication inventory explicitly strict and diagnostic

In `src/sase/agents_sync/inventory.py`, `inventory_io.py`, and `inventory_models.py`:

- Add a single per-record publishability check that normalizes owner-relative legacy
  spellings, validates the v2 run name and any family/clan/relationship name through the
  core identity facade, and converts validation failures into structured diagnostics.
- Keep invalid records out of `ProjectHoodInventory.runs` so `hood_runs()`,
  `eligible_hoods()`, ancestor synthesis, and container rendering never apply strict
  parsing to an unvalidated project-wide candidate.
- Retain enough diagnostic identity/source information to report which artifact was
  quarantined and why. Do not include host paths in transported snapshots; diagnostics
  remain local.
- Ensure valid records in the same project and unrelated hoods remain selectable, and
  continue excluding imported provenance from local export.

In `src/sase/agents_sync/publication.py`:

- Add or expose a hood-oriented entry point that publishes one already-selected hood
  from a supplied `ProjectHoodInventory`, without requiring every old outbox request's
  original artifact to be the representative caller.
- Preserve prior published runs on temporary local absence, deterministic rendering,
  owner sharding, and strict `validate_snapshot()` behavior.
- Return inventory/quarantine diagnostics in publication counts so a successful partial
  recovery is truthful.

### 2. Replace fail-fast draining with per-hood outcomes and proof-based acknowledgement

Refactor `src/sase/agents_sync/commit_publication.py` and
`src/sase/agents_sync/publication_outbox.py`:

- Capture a stable tuple of pending requests, group it by `local_hood`, and construct
  one identity/inventory snapshot for the entire attempt.
- Replace the single `str | None` preparation result with typed per-hood/per-request
  outcomes: prepared digest, satisfied, retryable Git failure, or
  quarantined/publication failure.
- Continue after a hood-local exception. Update `attempts` and `last_error` only for the
  affected logical keys; do not overwrite unrelated request diagnostics.
- After a no-op, normal push, or successful non-fast-forward retry, read and validate
  the owner manifest/snapshots and derive the exact set of satisfied logical keys.
  Acknowledge only that set and leave every unresolved request durable.
- Report `drained` as the number actually acknowledged. Set the current commit outcome's
  `published`/`queued` state from whether its own logical key was satisfied, even when
  the same transaction drains other requests.
- Keep enqueue idempotence, primary-commit success, checkpoint/resume behavior, and no
  duplicate sidecar commit when the payload is unchanged.

Refactor `src/sase/agents_sync/git_sync.py` to use the same proof helper after full sync
instead of clearing every request captured before any broadly successful outcome.
Preserve project-level failure isolation and status rewriting.

### 3. Eliminate repeated inventory, legacy scans, and registry rebuilds

In `src/sase/agents_sync/commit_publication.py` and `publication.py`:

- Build `ProjectHoodInventory` once, pass it to every distinct hood publication, and
  reuse it on the non-fast-forward retry.
- Render a hood once even when several queued primary revisions belong to it; rely on
  the per-revision satisfaction proof to retire all obligations represented in the
  resulting snapshot.

In `src/sase/agents_sync/incoming_integration.py`, `bundles.py`, and the agent-name
registry mutation modules:

- Load receipts once and partition discovery into already-applied versus changed
  v2/legacy groups before artifact mutation.
- Introduce a reusable legacy import/provenance index for the project pass so
  current-owner proof and existing-import lookup do not traverse the complete artifact
  history for each entry.
- Add an atomic/batched legacy-v1 registry-claim operation parallel to the existing v2
  batch API. Preflight collisions per hood group, apply accepted artifact changes under
  the existing locks, and persist the final registry state once with a source signature
  matching the resulting artifact tree.
- Preserve exact-current-owner no-duplicate behavior, unknown-username v1 provenance,
  per-group quarantine, receipt semantics, artifact-index updates, and crash cleanup of
  staged files.

Do not increase `DEFAULT_SYNC_LOCK_TIMEOUT_SECONDS` as a substitute for this work. Add
cheap typed diagnostics or operation counters where needed for deterministic tests, but
do not add data-scaled status/render work.

### 4. Add regression, concurrency, and scale coverage

Extend the focused tests under `tests/agents_sync/` and the agent-name registry tests:

- Inventory containing valid `toobig-r` runs plus legacy `4x--epic.f-0` emits a
  diagnostic and successfully publishes the valid hood without changing the legacy
  artifact.
- An outbox with a failing first hood and valid later hoods publishes/acknowledges the
  valid requests, retains only the failed logical keys, and keeps each request's own
  error/attempt count.
- Two or more requests for one hood build inventory and render that hood once, while all
  exact agent/revision obligations present in the pushed snapshot are acknowledged.
- A snapshot missing the requested global agent or primary revision is never
  acknowledged, including after a generally successful full sync.
- Push failure acknowledges nothing; non-fast-forward retry reuses preparation, pushes
  once reconciled, and acknowledges only proven requests; a request enqueued
  concurrently after capture is not accidentally removed.
- Receipt-matching v2 and v1 hoods perform no import/registry mutation. A changed legacy
  manifest with many entries performs one project artifact scan and at most one registry
  load/rebuild/save batch, using call-count assertions rather than flaky wall-clock
  thresholds.
- Lock contention still returns the typed busy/retry outcome within the configured
  bound, and no heavy preparation is added to TUI action/render/timer callbacks.

Add a local-bare-remote integration fixture with a realistic backlog and malformed
unrelated legacy artifact. Prove one targeted publish or `sase agent sync` advances the
sidecar with every valid global agent/revision association, empties the valid backlog,
leaves only genuinely unrepresentable obligations if any, and is idempotent on the next
run. Use a family-member committer in a deeper hood and assert that the pushed snapshot
still contains the full recursive top-level closure, including active and commit-less
relatives, and remains importable as one revivable family. Also keep the existing linked
`SASE_AGENT`, no-new-`SASE_MACHINE`, foreign-only indicator, cached no-network
integration, and explicit full-sync tests in the required regression set.

### 5. Validate and recover the live project deliberately

Run `just install` before repository checks, then run the focused
inventory/publication/outbox/import/registry tests and the mandatory `just check`. No
visual snapshots are expected because this plan changes no presentation.

After code validation, inspect the live outbox and sidecar head, then run the explicit
full sync for only this project:

```bash
sase agent sync -p sase -j
```

This is the authorized recovery push, not a manual edit. Verify that:

- the `sase--agents` remote advances from its pre-recovery head;
- every previously queued valid `(global_agent, primary_revision)` is present in a
  validated pushed snapshot;
- representative primary `SASE_AGENT` links resolve to the expected family/agent page
  and stable member anchor, with no new `SASE_MACHINE` footer;
- the durable outbox is empty, or contains only individually diagnosed obligations that
  genuinely cannot be represented;
- a second project-scoped sync is clean/idempotent; and
- the historical `4x--epic.f-0` artifact and all SASE memory/generated-instruction files
  are unchanged.

If live recovery exposes a different failing hood, preserve its outbox item and report
its exact diagnostic rather than editing user history or broadly acknowledging the
queue.

## Acceptance criteria

- Every successful primary `sase commit` is either proven in the pushed agents-sidecar
  snapshot or remains durably and individually queued; no successful primary commit is
  duplicated because auxiliary publication failed.
- One malformed legacy artifact, one failed hood, or one foreign import collision cannot
  prevent unrelated hoods from being committed and pushed.
- Existing invalid legacy names remain locally readable and untouched, while strict v2
  snapshots never contain them or weaken family-name validation.
- Automatic retry still publishes the complete recursive project-scoped hood and
  produces importable/revivable family state; linked global `SASE_AGENT` provenance,
  no-new-`SASE_MACHINE`, foreign-only cached updates, and explicit full sync retain the
  behavior required by the original epic prompt.
- A drain builds local inventory once, publishes each distinct hood once, skips
  receipt-matching imports, and avoids per-bundle registry rebuilds; lock acquisition
  remains bounded and TUI work remains in tracked background paths.
- Targeted publication, full sync, no-op, push retry, and non-fast-forward retry all
  acknowledge only exact agent/revision obligations proven by validated sidecar state.
- The existing live backlog is recovered through the normal sync/publication workflow,
  the sidecar remote advances, and the next sync is idempotent.
- `just install`, focused tests, and `just check` pass; no linked repository, SASE
  memory file, generated provider instruction, or historical artifact is modified.
