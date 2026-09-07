---
tier: tale
title: Implement reliable remote dispatch launch
goal:
  "`%dispatch:<machine>` launches exactly once on the pinned remote host and remains
  recoverable across lost replies without local fallback."
size: medium
proposed_by: bbugyi200.athena.sase-xe.12
bead: sase-xe.12
status: done
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.12.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-xe.12](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.12.md)
- **COMMITS:**
  - [06fb5c3](https://github.com/sase-org/sase-core/commit/06fb5c38ec612253ce1d6ced75e4a4ae89278ed4)
    — feat(fleet): add remote launch dispatch contract

# Implement `%dispatch` and reliable remote launch

## Objective

Complete phase `sase-xe.12` as one bounded cross-repository implementation: add the
gated `%dispatch:<machine>` directive, route it before local launch allocation, submit a
portable and idempotent launch through the federation worker to the owning gateway,
persist and reconcile source intent without local fallback, and activate the existing
dispatch follow only after binding the authoritative receipt.

## Settled constraints

- Preserve the accepted `%dispatch:<alias>` and `%dispatch(alias)` spelling with no
  short alias; `local` is reserved for explicit local routing.
- The Python runtime vocabulary and Rust editor/completion contract remain exact-set
  peers, with `remote_dispatch` gating completion and runtime acceptance.
- Source-side routing is side-effect-free and occurs in `launch_query()` before any call
  to `launch_agents_from_cwd`; xprompt expansion, substitutions, workspace setup, and
  finalizers happen only on the target.
- Never fall back to local execution or another machine. Distinguish definitely unsent,
  acceptance-uncertain, and accepted/settled outcomes, and reuse one operation key for
  reconciliation.
- Portable project context uses configured project/provider identity and revision or
  Patch evidence, never a source checkout path. Reject unsupported local-only files,
  dirty worktree state, and unpublished references with preparation guidance.
- V1 permits only one outer dispatch target and rejects mixed-host fan-out plus
  cross-host `%wait`/`%clan` composition.
- Existing enrollment credentials, fleet authentication, mutation contracts,
  federation-worker deadlines, and follow-store tombstones are authoritative; explicit
  unfollow continues to win during receipt reconciliation.
- Empty machine configuration remains worker-free, and completion consults cached
  configuration only.

## Implementation

1. Extend the shared directive vocabulary and parser.
   - Add `dispatch` to the Rust directive metadata with machine-valued colon and
     parenthesized forms, `remote_dispatch` visibility, hover/examples/recipes, and
     contract tests.
   - Add `dispatch: str | None` to `PromptDirectives` and update Python collect,
     extraction, protected/literal-zone scanning, prompt metadata, inspect, and parity
     tests.
   - Add a dedicated side-effect-free routing scan that validates one non-empty target,
     conflicting/branch-local targets, and unsupported `%wait`/`%clan` combinations
     without allocating names or resolving xprompts.
   - Offer enrolled aliases from the pure cached dispatch config in ACE and shell/LSP
     completion while the flag is enabled; do not load providers or contact hosts.

2. Define the portable launch and receipt contract in `sase-core` and expose it through
   the gateway contract snapshot.
   - Add versioned launch-intent, portable-project-context, attachment/reference,
     admission-state, reservation/locator, receipt, and reconciliation wire types with
     strict validation and canonical payload fingerprinting.
   - Model operation identity as controller plus operation key, pin the intended target
     installation, and make replay/conflict/expiry behavior explicit.
   - Cover invalid local paths, unsupported attachments/references, target mismatches,
     duplicate payloads, and receipt state transitions in Rust unit tests.

3. Implement target admission and federation transport.
   - Add a scoped fleet launch permission and authenticated `/api/fleet/v1/launch`
     endpoint.
   - Persist a lock-protected target operation/reservation record before invoking the
     normal target-side agent bridge; return the original receipt for identical replay,
     conflict for a changed payload, and a durable pending/uncertain receipt across
     interrupted reply boundaries.
   - Bind successful bridge results to authoritative logical/run locators, publish fleet
     invalidations, and include the route and new types in the generated fleet contract
     snapshot.
   - Add a single-target mutation operation to the federation worker and its Python
     facade, preserving deadlines, installation-pin quarantine, exact-alias selection,
     and structured remote errors without read-cache fallback.

4. Integrate enrolled machines and durable source intent.
   - Reconcile the duplicate `dispatch:` defaults/schema produced by earlier phases so
     the federation worker derives hosts from `dispatch.machines` and resolves bearer
     tokens through the existing 0600 credential store; retain compatible worker
     settings under the same section.
   - Add an atomic source dispatch-intent store containing reconstructible prompt,
     portable context, target pin, operation key, payload digest, follow request, and
     unsent/uncertain/accepted/settled status.
   - Intercept `launch_query()` before local force-reuse/typed/local launch paths,
     validate the flag and target, prewrite the dispatch follow, submit through the
     federation facade, bind/activate the receipt, and emit the existing typed
     `run.launch` result shape with pending/reconciliation metadata.
   - On replay or restart, reconcile the same operation key. Preserve explicit unfollow
     tombstones and never generate a second launch key after a timeout or lost reply.
   - Enrich ACE's durable launch-proc payload and placeholder record with the target and
     dispatch state so pending and “checking outcome” launches remain visible without
     blocking Textual's event loop.

5. Verify both repositories and close only the assigned phase.
   - Add focused Python tests for parser/contract/completion parity, outer-level
     restrictions, flag-off rejection, config-to-worker integration, portable-context
     rejection, unreachable/unsent behavior, duplicate submission, lost reply/restart
     reconciliation, capacity/pending receipts, follow activation, and
     unfollow-during-pending.
   - Add Rust gateway/worker fault tests for authentication/scope, operation replay and
     payload conflict, target pin mismatch, durable pending recovery, receipt binding,
     and exact single-host routing; regenerate/check the fleet API contract.
   - Install the local Rust binding as required, run targeted suites, then the mandated
     fast `just check` lane and repository-appropriate Rust checks. Record any genuinely
     out-of-scope discovery only as a `PROPOSED FOLLOW-UP:` note.
   - Run `sase bead epic-symbols sase-xe.12`, resolve or re-key every remaining symbol,
     and close only `sase-xe.12` with a note naming the verification performed.

## Acceptance criteria

- With `remote_dispatch` off, `%dispatch` is hidden from completion and rejected before
  any local or remote launch side effect.
- With the flag on and alias `apollo` enrolled, `%dispatch:apollo` is completed from
  cached config, stripped from the target prompt, and admitted exactly once on the
  pinned installation through the target's normal launch machinery.
- A missing alias, quarantine, unreachable host, unsupported local context, conflicting
  payload replay, or uncertain reply produces a specific durable result and never a
  local/alternate-host launch.
- Replaying or restarting an uncertain submission queries/resubmits the same operation
  key and binds the original target receipt; explicit unfollow prevents automatic follow
  resurrection.
- Default config has one coherent `dispatch:` section, an empty machine registry starts
  no worker, and enrolled machine records feed worker connection plans without copying
  secrets into YAML or IPC diagnostics.
- Python/Rust directive and completion parity, fleet wire snapshots, targeted fault
  suites, the binding validator, and `just check` pass before the phase closes.
