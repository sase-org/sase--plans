---
tier: epic
title: Enforce user-owned primary workspace boundaries
goal: "Every SASE-initiated repository mutation runs in a claimed disposable workspace,
  conflict recovery is destructive only inside that ownership boundary, and configured
  primary sidecar checkouts converge safely without disturbing user work.

  "
phases:
  - id: ownership-contract
    title: Workspace ownership and mutation contract
    depends_on: []
    size: medium
    description:
      "ownership-contract: codify writable operational contexts and make canonical
      primary-project access read-only by default."
  - id: operation-leases
    title: Durable operational workspace leases
    depends_on:
      - ownership-contract
    size: medium
    description:
      "operation-leases: add reusable claimed-workspace lifecycle support for
      synchronous jobs, monitors, and detached procs."
  - id: disposable-retry
    title: Reset-and-replay conflict recovery
    depends_on:
      - operation-leases
    size: medium
    description:
      "disposable-retry: recover conflicts by resetting only leased machine-owned
      checkouts and replaying bounded idempotent operations."
  - id: approval-launches
    title: Approval and task launches off the primary checkout
    depends_on:
      - disposable-retry
    size: medium
    description:
      "approval-launches: move approval-time plan archiving plus epic and task launch
      orchestration into durable operational workspace leases."
  - id: background-mutators
    title: Background bead mutations off canonical primary clones
    depends_on:
      - disposable-retry
    size: medium
    description:
      "background-mutators: route runner and scheduled bead writers through writable
      workspace-local stores while retaining canonical stores for reads."
  - id: sidecar-autosync
    title: Generic primary-sidecar auto-sync
    depends_on:
      - ownership-contract
    size: medium
    description:
      "sidecar-autosync: add opt-in clean fast-forward synchronization for plans, beads,
      research, and arbitrary configured sidecars."
  - id: invariant-audit
    title: End-to-end ownership audit and regression gates
    depends_on:
      - approval-launches
      - background-mutators
      - sidecar-autosync
    size: small
    description:
      "invariant-audit: prove primary-checkout immutability, disposable retry safety,
      and sidecar convergence across automated workflows."
proposed_by: bbugyi200.athena.035
create_time: 2026-08-15 21:47:50
status: done
---

- **PROMPT:**
  [prompts/202608/primary_workspace_ownership.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/primary_workspace_ownership.md)

# Plan: Enforce user-owned primary workspace boundaries

## Findings and decisions

The reported ownership defect is confirmed, and it is broader than the approval example:

- Approved epic launches resolve `get_workspace_directory(project, 1)` in
  `sase.bead.epic_launch.resolve_epic_launch_cwd`, then start `sase bead work` with that
  primary checkout as its cwd. Setting `inherit_lane_workspace_claim=False` prevents
  inheritance of the planner's claim; it does not allocate a replacement workspace. The
  monitor resolves the supplied cwd back to workspace `#0` and runs there.
- Task-triage launches reuse the same primary-cwd resolver and submit an unclaimed
  detached proc there.
- Approval-time plan archiving explicitly materializes workspace `#1` (the legacy alias
  for primary `#0`) and commits to the plans store reached from it.
- Runner-owned bead claims, periodic claim reconciliation, and external-issue mirroring
  resolve `canonical_beads_dir_for_project`. For sidecar storage that is the clone
  rooted under the primary checkout, and those paths write, commit, and publish bead
  state.

The reset claim is correct only after adding a strict ownership precondition. A blanket
`git reset --hard origin/main` is not safe in the user's primary checkout, in a
user-edited primary sidecar, or after an operation has produced an external side effect
that cannot be replayed. The implementation will use the checkout's configured upstream
(normally `origin/main`, but not hard-coded) and authorize destructive reset only for a
live, claimed, numbered machine workspace and its workspace-local sidecars. The reset is
followed by replay of the high-level idempotent operation, not merely a retry of
`git rebase`; otherwise a local mutation discarded by the reset would never be
recreated.

Primary-sidecar sync exists today only in fragments. Sidecar clones are integrated when
materialized, bead commands have a TTL-gated refresh mode, and the `bead_store_refresh`
chop refreshes canonical bead stores for projects with live bead waiters. There is no
generic per-role auto-sync policy, no scheduled coverage for plans or research, and no
durable "this role just changed" hint. `auto_clone` means materialize before agent
launch and is not a primary-sidecar sync setting.

## Ownership model

Enforce these rules in shared helpers rather than relying on call-site convention:

1. A project's primary checkout (`#0`, including legacy API arguments that spell it
   `#1`) is user-owned. SASE automation may inspect it to resolve config, remotes,
   branches, project identity, and sidecar metadata, but may not change its worktree,
   index, HEAD, refs, Git operation state, or nested writable stores.
2. A numbered checkout from the unified claim pool (`#10` and above) is machine-owned
   only while a live operation holds its workspace claim. Its workspace-local linked and
   sidecar clones share that disposable ownership.
3. Explicit foreground commands the user runs in their chosen cwd remain allowed to
   mutate that cwd. Project/repository initialization and user-invoked `sase bead`
   commands are not silently redirected; the new restriction applies to SASE-initiated
   host, runner, and scheduled work.
4. Primary sidecar repositories are separate checkouts and may be updated only when
   their role opts into auto-sync. Auto-sync is conservative: materialized matching
   clones may fetch and fast-forward only when clean, attached, and non-diverged. It
   never rebases, commits, hard-resets, or removes user state, and it never mutates the
   primary repository that contains or describes the sidecar path.
5. SASE state outside repository checkouts (ProjectSpecs, proc records, artifacts,
   gates, sync hints, and workspace registries) keeps its existing ownership rules.

## Phase 1: Workspace ownership and mutation contract

Introduce an explicit operation/mutation context that distinguishes user-directed
access, read-only canonical access, leased operational access, and opt-in primary
sidecar sync. Build it on the existing workspace marker, registry, and Rust-backed
atomic workspace-claim APIs instead of deriving ownership from path suffixes.

Split project-wide store discovery into clearly read-only and writable APIs.
`canonical_beads_dir_for_project` and equivalent canonical plan/sidecar discovery remain
available to mobile views, wait evaluation, catalogs, task triage, and other readers,
but automated mutation helpers must receive a writable operation context whose checkout
and sidecar roots resolve inside the claimed workspace. Preserve foreground CLI behavior
by deriving a user-directed context from its cwd rather than mistaking every primary
path for an error.

Thread a machine/user mutation origin through SDD commit and bead publication seams that
currently default every caller to `cause="user"`. Fail before staging when a machine
mutation resolves to primary `#0`, a checkout without a matching live claim, or a
read-only canonical location. Keep the enforcement close to store mutation so a future
background caller cannot bypass it merely by importing a low-level commit helper. Add
focused tests for legacy `#1` normalization, managed-root and adjacent layouts, nested
sidecar ownership, foreground user access, and fail-closed behavior when marker/claim
evidence is missing.

## Phase 2: Durable operational workspace leases

Add a reusable operational workspace lease for non-agent work. Acquisition atomically
claims the next unified-pool workspace for the project, materializes its checkout,
prepares it from the configured primary remote, records the workflow/holder identity,
and exposes the workspace number, checkout path, project file, and workspace-local store
resolution. Preparation failure releases the claim; normal completion and every
exceptional exit do the same.

Support both execution shapes already present in SASE:

- A synchronous context manager for chops and host actions that execute in the current
  process.
- A durable proc/monitor submission path that preclaims under the submitting process,
  prepares the cwd before spawn, transfers the exact claim to the acknowledged
  supervisor behind its launch barrier, persists settlement policy, and releases the
  claim exactly once after success, failure, timeout, cancellation, reboot recovery, or
  a pre-transfer spawn error.

Do not fall back to the primary checkout when allocation, materialization, claim
transfer, or recovery fails. Surface a resumable error naming the failed operation and
leave the user's checkout untouched. Reuse the existing Rust atomic allocator and
claim-transfer plan; put any new cross-frontend ownership decision in `sase-core` rather
than duplicating claim semantics in Python.

## Phase 3: Reset-and-replay conflict recovery

Define a bounded operational transaction around mutations that can be replayed. Each
attempt starts from an attached clean checkout at its configured upstream. On a stale
Git operation, rebase/merge conflict, non-fast-forward publication race that cannot be
semantically integrated, or other repository-health conflict:

1. Verify that the affected repo belongs to the current live operational lease and is
   not primary `#0` or an auto-synced user sidecar.
2. Abort supported in-progress Git operations, fetch the configured remote, and hard
   reset the affected machine-owned checkout to its verified upstream tip. Clear only
   launch-scoped generated sidecar clones/state that the lease owns.
3. Replay the operation callback from its durable inputs and retry publication, with a
   small bounded attempt count and structured diagnostics.

Require replay callbacks to be idempotent and checkpoint-aware. Epic creation may be
replayed only before its graph publication/first agent spawn barrier; after an agent has
spawned, recovery must use the existing linked-epic resume path rather than reset and
duplicate externally visible work. Preserve a diagnostic recovery ref or log when
useful, but never require the user to resolve a conflict inside a disposable clone.
Remote unavailability and lock contention remain retries/deferments, not reasons to
reset. Extend integration tests with normal plan conflicts, semantic bead conflicts,
push races, stale rebases, retry exhaustion, and explicit proof that the same helper
refuses a primary path.

## Phase 4: Approval and task launches off the primary checkout

Change approval resolution to return canonical project identity and durable plan inputs,
not a primary execution cwd. Acquire an operational lease before any mutating approval
side effect, then:

- Archive tale and epic plans through the lease's plans store and publish them using
  reset-and-replay semantics.
- Start approved epic `sase bead work` monitors in the leased checkout. Keep planner
  family/lane association, but transfer the new operation claim rather than inheriting
  the planner workspace or claiming primary `#0`.
- Make the detached fallback use the same durable leased-workspace proc path.
- Launch approved task beads through a leased checkout instead of submitting an
  unclaimed proc in primary.

Keep active-launch deduplication stable across changing workspace numbers by keying it
on project plus plan identity or task bead ID, not cwd. Ensure failures before monitor
or proc acknowledgement undo the preclaim, and ensure completion metadata still links
the archived plan, epic, task, and planner artifacts. Replace tests that currently
assert `get_workspace_directory(project, 1)` with tests asserting a claimed numbered
cwd, transfer/settlement, no primary fallback, safe pre-publication replay, and
post-spawn resume behavior.

## Phase 5: Background bead mutations off canonical primary clones

Audit every `canonical_beads_dir_for_project` consumer and classify it as read-only or
mutating. Keep read-only uses on the canonical primary snapshot. Move at least these
known writers to writable workspace-local stores:

- Waiting-agent claim acquisition, retention, promotion, and release.
- The periodic `bead_claim_checks` reconciliation batch.
- External issue mirroring and its create/close/reopen/note commit.
- Any task/epic launch checkpoint, page/header refresh, or other scheduled writer the
  audit finds resolving a canonical primary store rather than its current workspace.

Runner actions may reuse an already claimed workspace only when the writable store is a
separate workspace-local sidecar and no reset could touch the agent's code edits;
otherwise acquire a short operational lease. Chops should batch one project's changes
under one lease and one publication transaction so periodic work does not churn claims.
Thread the chosen workspace/store explicitly instead of re-resolving by project name
inside lower-level helpers.

After successful publication, schedule primary-sidecar convergence for the affected role
rather than directly pulling the primary clone from the mutation path. Add tests for
sidecar, separate-repo, local, and in-tree SDD modes; concurrent claim and mirror
writers; dead-runner cleanup; and the invariant that primary HEAD/status/refs and
primary-sidecar HEAD remain unchanged until the auto-sync worker runs.

## Phase 6: Generic primary-sidecar auto-sync

Add an `auto_sync` boolean to every configured sidecar role, independent of
`auto_clone`. Update the config schema, doctor validation, inventory/model rendering,
project initialization, and generated examples. New managed-project configuration should
enable it for plans and beads, and the generated research entry should enable it as
well. Custom roles default off and opt in with the same property, so future sidecars
require no code changes. Preserve explicit `disabled` behavior and never expose the
hidden agents sidecar through this mechanism.

Create one generic primary-sidecar synchronizer that verifies role identity and remote
before touching an existing materialized clone. It may fetch and fast-forward a clean,
attached, strictly-behind clone. Dirty, detached, diverged, mismatched, or busy clones
are skipped with actionable diagnostics; they are never reset or replaced. Missing
primary clones are reported but not auto-created, because creating paths or Git excludes
in the primary checkout would violate the ownership boundary.

Drive the synchronizer from two sources:

- Successful workspace-sidecar publication writes a durable project/role sync hint so
  known remote changes are picked up promptly without blocking the publisher.
- A scheduled chop scans enabled projects and every `auto_sync` role as a backstop,
  using bounded work/lock budgets and persistent backoff so one unhealthy clone does not
  stall the fleet.

Coalesce duplicate hints, make runs idempotent, expose per-role refreshed/skipped/
failed counters, and cover plans, split and combined beads layouts, research, and an
arbitrary custom role in tests. Retire or narrow the bead-only primary refresh path so
there is one scheduling policy rather than competing synchronizers.

## Phase 7: End-to-end ownership audit and regression gates

Run a final mutation inventory over primary resolvers, canonical store locators, Git
write helpers, workspace `#0`/legacy `#1` call sites, and background processes. Move any
remaining automated repository writer behind the operation lease or explicitly document
why it is a foreground user action or conservative auto-sync operation.

Add end-to-end regression scenarios that snapshot the primary repo's worktree, index,
HEAD, local refs, and operation markers, then exercise plan approval/archive, epic
launch, task launch, bead claim/release, claim reconciliation, external issue mirror,
conflict recovery, and retry failure. Assert byte-for-byte/oid stability of the primary
state throughout. Separately prove that an opted-in clean primary plans, beads,
research, or custom sidecar fast-forwards after a hint and after the scheduled backstop,
while a dirty or diverged sidecar is preserved.

Add a focused architectural test or lint allowlist that rejects new automated imports of
primary writable-store resolution and direct primary-cwd proc submission. Update
operator-facing configuration/help text to explain the ownership boundary, the meaning
of `auto_clone` versus `auto_sync`, why disposable conflicts are reset and replayed
automatically, and where sync/retry diagnostics are found. Run the required
whole-repository verification, using full verification if the scoped selector escalates
or any broadening-set files change.
