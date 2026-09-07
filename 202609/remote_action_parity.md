---
tier: tale
title: Implement remote lifecycle management parity
goal:
  A followed remote agent can be stopped, retried, forked on its own machine, and read
  through bounded content handles with the same action vocabulary as a local agent,
  under journal protection and exact-instance fencing, with bulk actions partitioned per
  origin and honest per-target results.
size: medium
proposed_by: bbugyi200.athena.sase-xe.13
bead: sase-xe.13
status: done
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.13.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-xe.13](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.13.md)
- **COMMITS:**
  - [1a3a12a](https://github.com/sase-org/sase/commit/1a3a12a7eff807fa93da4d788242a3adbfba5b6d)
    — feat(dispatch): add remote fleet stop, retry, fork, and bounded content

# Plan: Implement remote lifecycle management parity

## Objective

Complete phase `sase-xe.13` as one bounded cross-repository implementation: add a
journaled fleet mutation contract and endpoint for stop, retry, and fork-on-target;
execute those mutations only on the owning host against an exactly identified run
instance; expose them through the federation worker, Python facade, a durable CLI
operation, and the existing ACE kill/fork mixins with optimistic UI; read remote
chat/output/diff/artifact content through the already-served opaque handles with bounded
range reads and digest validation; and partition bulk actions per origin with per-target
results.

## Settled constraints

- Viewers never act locally on a remote row. No PID signal, no path, no artifact-index
  or running-marker mutation may run for a row that carries a fleet origin; owner
  resolution and execution belong to the owning host (phase `viewer-purity` invariant).
- Every mutation carries a controller-scoped operation key, a payload fingerprint, the
  exact `AgentInstanceLocatorWire`, and the observed `ResourceRevisionWire` as a
  precondition. A repeat with the same key and payload returns the original receipt; the
  same key with a different payload conflicts; an expired or tombstoned key is rejected,
  never retried as a new mutation.
- Target-side revalidation is mandatory even when the row was just fetched. A reused
  name, a superseded run instance, or a stale row revision produces a precondition
  mismatch and never targets the replacement.
- Capability checks combine host scope, resource capability, authorization, and
  freshness. A missing capability surfaces a specific reason instead of an attempt.
- Fork defaults to fork-on-target and follows the result; "fork here" stays out of
  scope. Retry and fork execute through the target's own launch machinery, so the source
  needs no published revision, no clean checkout, and no portable project context.
- Bulk actions partition by origin, submit one operation per origin, and report
  per-target results. They are never presented as atomic fleet transactions, and
  destructive work against an unreachable host fails immediately rather than being
  queued for later execution.
- Content reads stay bounded: handles only, range reads with a byte ceiling, digest
  validation, and no full-history download. Origin and observation age are visible
  wherever remote content or feedback is shown.
- Laziness is preserved. With no enrolled machine nothing new runs; content is fetched
  only on an explicit open; no mutation path adds a timer, poll, or background remote
  read. All I/O stays off the Textual event loop and off the serial message pump, user
  mutations run as tracked procs, and selection is re-read after every await.
- `remote_dispatch` gates every new surface. With the flag off, the new CLI children,
  ACE remote actions, and facade mutation calls refuse with the existing beta-gate
  message and the Off branch stays trivially deletable.
- Feedback names the host ("Stop requested on apollo"). Remote location alone adds no
  extra confirmation step; the existing confirmation policy applies with the target
  stated explicitly.

## Implementation

Before starting, open the Rust core repository with the `/sase_repo` skill (every
`crates/...` path below is relative to that checkout) and read the `tui_perf.md` and
`lint_and_test.md` reference memory with `/sase_memory_read`. Do not edit anything under
`sase/memory/`; this epic leaves memory untouched.

1. Define the fleet mutation contract in `sase-core` and expose it to Python.
   - Add `crates/sase_core/src/fleet_mutation.rs` (new module beside `fleet_contract`,
     re-exported from `lib.rs`) with versioned wire types: mutation kind
     (`stop`/`retry`/`fork`), mutation intent (exact target, row revision, optional
     bounded reason, fork prompt, retry `kill_source_first`, follow request), request
     (scoped operation key, target installation pin, fingerprint, acceptance window),
     receipt (operation receipt fields plus outcome, resulting logical locator, safe
     message), durable record, decision request/decision, and response.
   - Add canonical payload fingerprinting, strict intent/request validation (bounded
     strings, no paths or secrets, fork requires a prompt, retry/stop reject one), a
     replay decision built on the existing `decide_operation_replay` semantics, the
     kind-to-capability map (`lifecycle.stop`, `lifecycle.retry`, `lifecycle.fork`), and
     a precondition evaluator that compares an observed `ResolvedAgentSummaryWire`
     against the intent and returns a typed refusal reason (unknown row, instance
     mismatch, stale revision, capability missing, already terminal).
   - Add a pure per-origin bulk partition helper that groups requested targets by origin
     installation with deterministic ordering and reports targets it cannot attribute.
   - Cover in Rust unit tests: fingerprint stability, validation rejections, replay
     accept/return/conflict/expiry, every precondition refusal reason, and partition
     determinism.
   - Expose `fleet_mutation_payload_fingerprint`, `fleet_validate_mutation_request`,
     `fleet_evaluate_mutation_precondition`, and `fleet_partition_bulk_targets` through
     `crates/sase_core_py` in the existing dict-in/dict-out style, document them in the
     binding module docstring, and add them to `tools/validate_sase_core_rs`'s required
     binding list.

2. Serve journaled mutations from the target gateway.
   - Add `FLEET_SCOPE_MUTATE` to `fleet_auth.rs` and to the default scope set, and
     publish lifecycle capabilities from `fleet_reads.rs`: `lifecycle.stop` only for a
     live current agent-shell instance, `lifecycle.retry` and `lifecycle.fork` for
     agent-shell rows the host can relaunch. Non-agent rows and terminal instances
     advertise no lifecycle capability.
   - Add `crates/sase_gateway/src/fleet_mutations.rs` with a `FleetMutationStore`
     modeled on `FleetLaunchStore`: exclusive file lock, atomic 0600 write, receipt and
     tombstone retention through the acceptance window, `reserve` returning the replay
     decision, and `settle` binding the outcome, result locator, and safe message.
   - Add `POST /api/fleet/v1/mutate` to `routes.rs`: authenticate the scope, negotiate
     the protocol version, verify the installation pin, resolve the current resolved
     record for the intent's logical key, evaluate the precondition (returning a
     specific error code for fencing and stale-revision refusals), reserve the
     operation, then execute through the agent bridge — stop via `kill_agent`, retry via
     `retry_agent`, fork via a new bridge operation — settle the receipt, audit the
     operation without prompts or secrets, and publish an `agents_changed` invalidation.
   - Add the `fork_agent` method to `AgentHostBridge`/`DynAgentHostBridge` and to
     `CommandAgentHostBridge` (`fork-agent` subcommand), keeping the unavailable-bridge
     default.
   - Add the route, records, scope, and limits to `fleet_api_v1_contract_snapshot()` in
     `contract.rs` and regenerate the committed
     `crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json`.
   - Add gateway tests: scope denial, accepted stop returning a settled receipt,
     identical replay returning the original receipt, changed payload conflict, stale
     revision and superseded instance refusals, already-exited target, missing
     capability, bridge failure mapping, and invalidation publication.

3. Execute forks on the target through the existing Python launch machinery.
   - Add a `fork-agent` mobile-bridge operation (`src/sase/main/parser_mobile.py`,
     `src/sase/integrations/mobile_agents.py`, `_mobile_agent_lifecycle.py`) that
     resolves the named source agent, composes the existing `#fork:<name>` prompt
     vocabulary with the requested instruction, launches through the same helper
     `retry-agent` uses, and returns the launch result including the new agent name.
     Reuse the existing not-found/not-running/permission error classes so the gateway
     keeps mapping them to stable error codes.

4. Carry mutations through the federation worker and the Python facade.
   - Add a `Mutate { target, request }` IPC operation to
     `crates/sase_gateway/src/federation_worker.rs` that resolves exactly one host by
     alias, enforces the installation pin, honors the request deadline and both
     concurrency limiters, posts `/mutate`, and returns the host-attributed payload with
     structured remote errors. Mutations never read from or write to the read cache.
   - Add `mutate`/`mutate_sync` to `src/sase/dispatch/federation/_facade.py`, refusing
     with `FederationWorkerUnavailable` when no machine is configured, mirroring
     `launch_sync`.
   - Add worker tests for deadline expiry, unknown alias, pin mismatch, and error
     propagation.

5. Build the source-side mutation client and durable intent journal.
   - Add `src/sase/dispatch/mutations.py`: build the intent, operation key, and
     fingerprint from a row snapshot (origin installation, exact locator, logical key,
     revision, capability set) through the new bindings; require the flag and a
     non-quarantined enrolled machine; submit through the facade; and classify the
     response into a typed outcome (`applied`, `already_settled`, `precondition_failed`,
     `capability_missing`, `unsent`, `uncertain`) with a host-named message. Bulk
     submission partitions through `fleet_partition_bulk_targets` and returns one result
     per target.
   - Add `src/sase/dispatch/mutation_intent.py` mirroring `launch_intent.py`: an atomic,
     lock-protected record per operation key holding target alias, installation pin,
     kind, exact locator, fingerprint, follow request, status, receipt, and timestamps,
     so a lost reply reconciles under the same key instead of issuing a new one.
   - On a settled fork or retry that returns a result locator, activate the follow
     through the existing follow-store helpers, honoring explicit unfollow tombstones.
   - Add `src/sase/dispatch/content.py`: an explicit-open content client that fetches
     detail for handles, performs bounded range reads with the contract's byte ceiling,
     validates the returned digest, caches decoded chunks keyed by handle, revision, and
     digest, and exposes growth-aware tail continuation without downloading history.

6. Add the durable CLI operation ACE submits.
   - Add `sase machine agent {fork,retry,stop}` in `src/sase/main/parser_machine.py`
     (alphabetically placed, positional `ALIAS` and `AGENT`, `stop`/`retry` accepting
     one or more agents, `fork` taking the instruction positionally, optional
     `-j/--json` and `-t/--timeout`), routed from `machine_handler.py` to a new
     `src/sase/ops/commands/machine.py` runner with a new `MACHINE_AGENT_ACTION`
     operation name.
   - The runner prefers the durable request sidecar (ACE supplies the exact locator,
     revision, operation key, and follow request) and otherwise resolves the current
     instance from a fresh bounded lookup for direct human use; it emits the typed
     durable result with per-target outcomes and never falls back to another machine.
   - Refresh `tests/completion/snapshots/cli_spec.json` with `just sync-completion-spec`
     and keep the help text sorted and complete.

7. Route ACE actions by origin with optimistic UI.
   - Add `src/sase/ace/tui/actions/agents/_remote_lifecycle.py`: a mixin that submits
     one tracked proc per origin through the durable adapter, applies the optimistic row
     state, re-reads selection after each await, and reports the settled per-target
     outcome with host-named toasts; and
     `src/sase/ace/tui/actions/agents/_remote_content.py` plus a bounded viewer modal
     that shows origin, observation age, and a range-extending "load more" for growing
     content.
   - Branch the existing entry points instead of duplicating them: `action_kill_agent`
     (`_kill_action_flow.py`) routes a selected remote row to the remote stop flow with
     a confirmation that names the machine and the exact instance; `action_fork_agent`
     (`_fork_actions.py`) opens the prompt bar for a remote row, labels it "Fork on
     <alias>", and submits the fork mutation instead of a local launch; retry is offered
     on remote rows through the same remote mixin.
   - Partition the bulk paths (`_marking_kill.py`, `_kill_all_actions.py`, panel and
     group kills) into local rows plus per-origin remote groups, confirm once with the
     partition displayed, and never mix a remote row into a local cleanup plan.
   - Update `_app_action_availability.py` and `commands/_availability_agents.py` so
     kill, fork, and the new content action are available on a remote row only when the
     row's capability set, the flag, and a non-quarantined origin allow it, while every
     other local-only action stays blocked.
   - Register the new content action end to end: `src/sase/default_config.yml`
     (`unbound` by default, in the Agents sub-tab block), `keymaps/app_keymaps.py`,
     `keymaps/metadata.py`, `commands/_app_metadata.py`, and the agents help modal.

8. Verify both repositories and close only the assigned phase.
   - Add Python tests for: flag-off refusal on every new surface, capability-missing
     refusal, precondition mismatch after a superseded instance, stale row revision,
     unreachable host (unsent, nothing queued), duplicate submission returning the
     original receipt, lost reply reconciled under the same key, fork result follow
     activation with an unfollow tombstone honored, bulk partition producing per-origin
     procs and per-target results, remote rows never touching local kill/cleanup paths,
     and bounded digest-validated content reads with cache reuse and growth
     continuation.
   - Add ACE tests that a remote selection routes to the remote mixin, that a local
     selection is unchanged, and that no remote work happens with an empty machine
     registry.
   - Run `just install` (the Rust binding must be rebuilt from the linked checkout),
     `./scripts/check.sh all` in `sase-core`, then `just check` in the sase repo, using
     `/sase_monitor` when a lane runs long.
   - Record genuinely out-of-scope discoveries only as `PROPOSED FOLLOW-UP:` notes on
     `sase-xe.13`, including the pinned-core ratchet needed before CI's pinned-binding
     gate sees the new bindings.
   - Run `sase bead epic-symbols sase-xe.13`, resolve or re-key every remaining symbol,
     and close only `sase-xe.13` with a note naming the verification performed.

## Acceptance criteria

- With `remote_dispatch` off, `sase machine agent`, the ACE remote actions, and the
  facade mutation path all refuse with the beta-gate message and perform no remote call.
- Stopping a followed remote agent kills exactly that run instance on its owning host,
  leaves local process, path, and artifact state untouched, and reports "Stop requested
  on <alias>".
- Retry and fork execute on the target through its own launch machinery with no
  source-side revision, checkout, or portable-context requirement; a settled fork or
  retry binds the resulting locator and activates the follow unless an explicit unfollow
  tombstone exists.
- Replaying an identical mutation returns the original receipt, a changed payload under
  the same key conflicts, an expired key is rejected, and a lost reply reconciles under
  the same key rather than issuing a second mutation.
- A superseded instance, a stale row revision, or a missing lifecycle capability
  produces a specific refusal naming the host and reason, and never targets a
  replacement agent.
- A bulk action across two origins submits one operation per origin and reports one
  result per target, with a failing or unreachable origin not affecting the other and
  nothing queued for later execution.
- Opening remote chat, output, diff, or artifact content fetches only bounded ranges
  through handles, validates the digest, reuses the cache on reopen, extends growing
  output without refetching history, and always shows origin and age.
- With an empty machine registry, ACE behavior, timers, and remote traffic are
  unchanged.
- `sase-core`'s `./scripts/check.sh all` (including the regenerated fleet contract
  snapshot test) and the sase repo's `just check` pass before the phase closes.
