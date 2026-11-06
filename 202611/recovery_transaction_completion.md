---
tier: epic
title: Complete the production keep-work recovery transaction
parent_bead: sase-1qz.6.1.4
goal: 'Recover held work through one durable admitted launch, validate the successor
  before provider execution, and settle or roll back only from proven ownership evidence.

  '
phases:
- id: execution_policy
  title: Rust execution validation and confirmation contract
  depends_on: []
  size: medium
  description: 'execution_policy: implement reservation-aware target validation and
    confirmation/rollback decisions in sase-core, with registered bindings and compatibility
    tests.'
- id: prompt_preparation
  title: Prepare and pin the real recovery prompt
  depends_on:
  - execution_policy
  size: medium
  description: 'prompt_preparation: replace held-only post-CAS preview calls, preserve
    effective model overrides, and exercise the real preview and continuation adapters.'
- id: durable_admission
  title: Submit and resume the pinned recovery request
  depends_on:
  - prompt_preparation
  size: medium
  description: 'durable_admission: connect the winning reservation to durable typed
    admission and a reconstructible strict-workspace dispatcher, including cancellation
    and retry evidence.'
- id: bootstrap_ack
  title: Prove required inheritance before provider execution
  depends_on:
  - execution_policy
  size: medium
  description: 'bootstrap_ack: validate goal, bead, permission scope and original
    repository baselines before provider execution, then persist identity-bound bootstrap
    evidence.'
- id: settlement_rollback
  title: Confirm the actual successor and fail safely
  depends_on:
  - durable_admission
  - bootstrap_ack
  size: medium
  description: 'settlement_rollback: consume bootstrap evidence, verify identity and
    model, and return to held only after cancellation, verified termination and safe
    custody restoration.'
- id: integration
  title: Verify the production transaction and closure evidence
  depends_on:
  - settlement_rollback
  size: medium
  description: 'integration: cover immediate and queued recovery with real policy
    and isolated fake providers, run required checks, and prepare evidence for the
    land agent to close sase-1qz.6.1.4.'
proposed_by: bbugyi200.athena.sase-1qz.6.1.4
proposed_with: codex/gpt-6-astra
create_time: 2026-11-05 10:12:01
status: done
bead_id: sase-1qz.6.1.4.1
---

- **PROMPT:** [prompts/202611/recovery_transaction_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/recovery_transaction_completion.md)
- **BEAD:** [sase-1qz.6.1.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qz/sase-1qz.6.1.4.1.md)

# Complete the production keep-work recovery transaction

## Why another child epic is necessary

This is the unfinished remainder of `sase-1qz.6.1.4`, not a replacement for work already
landed. Commit `8462195314` implements the transaction modules and its tests; it is an
ancestor of both the inspected HEAD (`71fd5b6c2a`) and `origin/master`. Commit
`9ffaf50c33` subsequently split those tests by behavior. The phase's four obsolete
`--epic-symbol` rows are gone. Do not redo that work or reopen the phase by hand: it
remains in progress.

The phase notes previously described upstream exemption removal as the only close
blocker. Review on 2026-11-05 found actual acceptance gaps. The existing 108 focused
recovery/successor tests pass, but they replace the important production seams. Isolated
probes, using temporary artifacts and no live agents, found:

1. Run `Flow` from `tests/_agent_hijack_recover_helpers.py` with
   `dataclasses.replace(flow.deps(), preview=None)`, leaving the real Python/Rust
   preview active and fixing only model-candidate input to known constants. A Continue
   request records `begin_resolution`, `record_launch`, then `resolution_failed`; it
   returns `target_unavailable` and never spawns. The executor calls
   `_revalidate_pinned_target` with a `resolving` record, whose public preview
   deliberately disables recovery and clears the row target.
2. Calling the real `build_hijack_continuation_prompt` with that resolving record raises
   `HijackRecoveryRefusal(code="not_held")`. This is the next production barrier even if
   target revalidation is bypassed.
3. The default spawn chain is `recover._default_spawn_successor` →
   `recovery_successor.spawn_recovery_successor` →
   `detached_child.spawn_agent_session_successor` → `spawn_agent_subprocess`. There is
   no submission to `dispatch_typed_launch_request`. The computed `admission_request_id`
   is only ledger data. `_ClaimQueued` catches workspace contention and returns `queued`
   without creating a durable request a coordinator can resume.
4. A successful `Flow` with otherwise valid inheritance but child metadata naming a
   different provider/model and launch identity still returns `launched` and sets the
   record to `resolved`. Confirmation does not prove the effective model or persisted
   launch identity. The resolver writes the inheritance ack itself; `read_recovery_ack`
   has no production consumer.
5. On inheritance failure, `_terminate_successor` suppresses termination errors and only
   sends SIGTERM in the default path. Its caller immediately returns the record to
   `held`. This does not prove a partially booted successor is dead or restore the
   predecessor's claim before another attempt is allowed.

The clean checkout also reproduces the marker-path audit failure tracked by `sase-1by`:
five unreviewed contexts in `recover_subject.py` and `recover_settle.py`. The parent
epic has independent evidence of that recurrence. This preexisting audit failure alone
is not a reason to hold completion; the production gaps above are.

The remainder crosses core policy, admission persistence, provider bootstrap, and
process/custody effects. Keep each phase bounded as below instead of sending the entire
remainder to another single coding agent.

## Authority and boundaries

Read `plan:202611/hijack_keep_work_recovery.md` and the authoritative parent
`plan:202611/agent_hijack.md` with `sase artifact read`. Read the assigned bead and its
current notes before implementation. The parent's research reference was still missing
during this review; the accepted plans contain the contract.

Inherited decisions are final: Escalate/Switch is primary, Continue is the fallback, and
successor context defaults to fork with budget fallback to brief. This child epic
changes no memory files. The parent reserves its authorized `glossary:agent-hijack` edit
for feature landing; no decision record or other memory edit is authorized here.

Preserve the existing public executor/result surface used by later phases where
possible. `sase-1qz.6.1.5` owns stale reconciliation, pending terminal repairs, and
successor-following waits. `sase-1qz.6.1.6` owns CLI/ops integration. This child
supplies the durable identities and observation seams they need; it does not take over
either phase. Restart, Dismiss, TUI changes, flag removal, and feature documentation
remain with the outer epic.

Shared target, lifecycle, confirmation, and rollback policy belongs in `sase_core`;
Python gathers facts and performs effects. Open that repository with `/sase_repo`, read
its `AGENTS.md`, and use only the printed checkout path. Use `/sase_memory_read` for
applicable macros, tools, docs, ACE, or other reference instructions before touching
those areas. No global installation is part of this work.

## Transaction contract

- Preview remains read-only, and resolving records remain disabled in the public menu.
  Do not make menu actions available on resolving records or falsify the record's state
  to get past the guard. Execution validates its winning reservation separately, through
  core-owned policy.
- Prepare side-effect-free prompt and model facts while held. CAS establishes the exact
  action, target including effort, reserved session member, resolver process identity,
  launch identity, and admission identity before externally committing reservations,
  requests, claims, or a child process.
- Explicit model selection overrides an edited prompt's model, which overrides
  Continue's predecessor target. Feed the same effective model to the prompt, launch
  request, bootstrap, and settlement. Revalidate availability at dispatch without
  rotating an alias or replacing the pinned target. Instruct's model need not equal the
  unedited Continue row.
- Every queued result names a real durable admission request. A retry addresses the same
  bundle; a fresh coordinator reconstructs its recovery dispatcher from persisted data.
  Admission success means accepted/queued, not confirmed custody. A workspace conflict
  is not evidence of an admitted queue entry.
- The positive workspace number and directory remain fixed. Use the held owner recorded
  by the custody contract, including a missing old claim and a foreign claimant; do not
  assume the stopped provider PID is necessarily the claim PID. Never fall back to a
  pool workspace or workspace zero.
- Before provider execution, the successor proves the original goal, bead claim when
  present, permission scope, original baseline fingerprints, and all opened repository
  obligations. Missing evidence fails closed. Caller-supplied goal or bead values are
  expectations, not proof of completed inheritance.
- Bind bootstrap evidence to the reserved launch identity, name, process token,
  project/workspace pair, lineage and effective provider/model/effort. A resolver must
  read that evidence; it must not manufacture a successful child ack from incomplete
  parent-side observations.
- Definite prelaunch rejection can return to held only after the request cannot
  dispatch. Post-spawn uncertainty retains resolving. Returning to held after spawn
  requires verified process-tree death and safe claim restoration. If any step is
  uncertain, preserve identities and custody for reconciliation.
- Settlement preserves `outcome: hijacked` and unrelated fields, rejects real non-hijack
  outcomes, and uses the existing Rust status mapping. It is idempotent across ledger,
  predecessor marker, and successor metadata writes.

## Phase: execution_policy

Inspect the existing Rust
`agent_hijack/{preview,reconcile,ledger,wire,escalation, continuation}.rs` modules and
Python adapters before adding types. Reuse the current resolver/successor observation
conventions and state transitions.

Add a narrow execution validation contract for the winning reservation that
distinguishes held request preparation from resolving dispatch. It must check the stored
attempt and effective pinned target against current availability, without reopening the
public menu. Consolidate the shared target precedence and normalization currently
embedded in `recover_target.py` into core policy, reusing existing directive/model
parsing rather than creating another parser.

Provide pure confirmation and safe-rollback decisions over observed facts:
identity/model/workspace/lineage/ack mismatch, queued or dispatching request, verified
absent/dead child, and restored versus foreign/uncertain custody. Reuse existing
reconcile structures where appropriate, but do not impose its 60-second stale-resolver
delay on normal transaction completion. Specify the minimal bootstrap ack schema and the
admission descriptor fields needed by the dependent phases. Keep old records readable
and treat missing evidence as uncertain, not successful.

Expose registered PyO3 bindings with wire round-trip tests and thin Python adapters.
Avoid consuming an unlanded core commit from Python: let host finalization land the core
change, and have dependent Python phases ratchet `sase-core-revision.txt` to that actual
commit before using it.

Acceptance: held menu availability is unchanged; resolving menu actions remain disabled;
only the owner of the exact reservation can dispatch; explicit and prompt model
overrides retain effort; unavailable or changed targets refuse; mismatched/missing
confirmations and unproven rollback never permit settlement or a fresh attempt. Run the
linked repo's required full `check` gate, including binding registration tests, not bare
cargo.

## Phase: prompt_preparation

Update `recover.py`, `recover_target.py`, and narrowly scoped preview/context helpers to
use the new core policy. Build and validate continuation inputs at a valid point before
CAS, then preserve the prepared prompt and pinned target for durable dispatch. Remove
the held-only post-CAS calls responsible for probes 1 and 2. Execution-time availability
checks must still run at actual dispatch.

Support Continue with an explicit model or edited prompt as required by the parent CLI
contract; the internal Instruct spelling may remain an adapter. Ensure an explicit
override updates the actual launched prompt/options instead of only the ledger's
declared target. Keep fenced launch examples inert and refuse active conflicting
launch/fan-out directives through existing core semantics.

Add regressions using the real Python preview, continuation builder and Rust bindings,
mocking only external observations or process effects. Cover Continue, Escalate/Switch
and Instruct; edited/explicit effort; oversized fork fallback; provider disabled at
dispatch; and stale preview revisions. No public preview operation may write artifacts,
reserve names, or mutate the ledger.

This phase owns preparation, not a new queue. Leave a concrete prepared payload for
`durable_admission` and keep the executor/result signatures usable by later CLI and
reconciliation phases. Record the payload seam in the phase note.

## Phase: durable_admission

Connect the successful CAS reservation to the existing typed admission system:
`launch_admission.dispatch_typed_launch_request`, its store/engine, and
`_agent_dispatcher_for_request`. Study the persisted AXE chop dispatcher pattern in
`axe/chop_typed_admission.py` as an example of reconstructing a specialized dispatcher;
do not route recovery through a generic allocator.

Persist one validated request descriptor with the prepared prompt, exact target,
reserved member/attach plan, project and fixed workspace pair, expected held owner,
lineage, and the core-defined attempt identities. Bind the actual request ID and
dispatch fingerprint to the ledger; the current locally computed hash must not
masquerade as an unrelated admission engine fingerprint. Reject reused identity with
conflicting contents. Make descriptor storage and ledger ordering repairable across
interruption, without inventing a parallel queue.

Register recovery dispatcher reconstruction for both initial dispatch and a new
coordinator process. Revalidate the attempt, pinned model and custody immediately before
invoking the strict successor primitive. Preserve normal hold/capacity behavior.
Cancellation and definite prelaunch failure must durably prevent a later coordinator
from spawning that attempt before returning it to held.

Expose read-only request status/identity observations for sibling reconciliation phase
`sase-1qz.6.1.5`. A repeated recovery call must return the existing attempt, not submit
a second bundle. If engine receipts cannot represent an uncertain dispatch, extend the
shared Rust contract instead of treating it as a retryable ordinary error.

Acceptance: a real held admission request produces no child until eligible; reloading
its descriptor into a fresh coordinator dispatches once with the same
workspace/name/model; two resolvers create one request and one child; queue cancellation
cannot later spawn; foreign ownership fails closed; queued metadata does not claim that
inheritance or custody has been confirmed. Use actual admission journals and locks with
isolated fake process effects.

## Phase: bootstrap_ack

Finish the before-provider enforcement around the existing recovery successor and runner
bootstrap seams. Current strict baseline inheritance is useful and must remain: no fresh
baseline snapshot of the predecessor's dirty tree is allowed.

Wire the reservation descriptor into child bootstrap. Bind the same goal through the
real goal machinery and transfer any actual bead claim through its existing ownership
path. Require the predecessor's live permission record and apply the normal explicit
autonomy rules without ambient widening. Validate all opened repository identities and
baseline fingerprints, not just file existence or equal counts. Missing predecessor
inventory must not become an empty inventory. Old verification receipts and final
declarations remain with their original turn.

Use the core validation contract to prevent provider execution until all requirements
hold. Persist the versioned ack only after required bootstrap effects finish; include
the durable launch identity and effective concrete model alongside the custody and
inheritance evidence. Use the established `hijack_lineage` convention expected by
escalation; audit the currently mixed `recovery_lineage` naming while wiring the
descriptor. Provide a clear typed failure observation when bootstrap stops before the
provider.

Acceptance: run through the actual bootstrap entry to a fake provider boundary; inject a
failure in each required inheritance step and assert the provider is never invoked.
Preserve baseline fingerprints and linked-repository obligations; test a predecessor
without a bead; reject a different model/effort, wrong attempt identity, missing live
scope, or missing inventory. The successful fake provider must observe the original
goal, claims, scope and both repository obligations.

## Phase: settlement_rollback

Replace parent-side ack synthesis in `recover_confirm.py` with observation of the
bootstrap ack. Compare its identity, model, lineage and workspace evidence through the
core decision function before `complete_resolution`. Child artifact directory existence,
a matching name, or a caller-supplied goal/bead is insufficient. Use atomic marker
helpers and preserve unrelated done metadata.

Distinguish definite prelaunch failure from uncertain post-spawn errors by recorded
launch progress, not merely broad exception classes. Coordinate cancellation with the
durable request so its coordinator cannot race rollback. Use existing process identity
and verified process-tree stop primitives to establish death, then restore or freshly
claim the exact held workspace only when ownership permits. A failed kill, recycled PID,
surviving descendant, foreign claim, or uncertain request state retains resolving and a
diagnostic. Never transition back to held after a best-effort signal alone.

Keep settlement repair idempotent for sibling phase `sase-1qz.6.1.5`: expose the
committed attempt, ack and pending projection evidence rather than making that phase
infer success from a name. Document the public observation/helper contract in the
phase's completion note.

Acceptance: wrong identity/model/effort/lineage does not resolve; a failed stop cannot
enable another spawn; confirmed death and restored custody permit a new attempt;
canceled admission stays canceled; fast-completing valid children can settle from their
durable evidence; interrupt each settlement write and replay the helper without changing
a real outcome or duplicating the successor.

## Phase: integration

Create isolated fixtures with predecessor modifications in primary and linked
repositories. Exercise the public executor through real Rust transitions,
preview/preparation, admission persistence, strict bootstrap and settlement, replacing
only the provider/process boundary. Do not use `Flow` mocks as the sole evidence for the
transaction. Do not touch a live held workspace or launch a real provider.

Verify Continue, Escalate/Switch and edited/explicit Instruct targets, immediate
admission and a queue resumed in a fresh coordinator, concurrent callers, cancellation,
unavailable target, failed inheritance, uncertain spawn, failed termination, and
interrupted settlement. Assert one request/child, same workspace number and directory,
exact model/effort, original goal/bead/scope/baselines and repository inventory, correct
status labels, and preserved predecessor outcome.

Review the five marker-path audit sites from the existing failure in
`tests/test_agent_artifact_marker_path_passing_audit.py`. Use canonical marker access
where needed; add reviewed contexts only when their paths and intent have actually been
checked. Re-run the focused audit and relevant transaction tests. Do not broaden this
into unrelated artifact infrastructure cleanup.

Check the core revision pin against the landed bindings and run the normal repository
gates. Prepare a short acceptance/evidence note for the child land agent; do not close
`sase-1qz.6.1.4` from this coding phase.

## Verification and completion ownership

Every phase reads the applicable verification memory, runs meaningful focused tests, and
runs `sase tool run check` in each repository it changes. Use `/sase_monitor` for long
commands with a concrete continuation/completion brief. Do not run `check-full`. Host
finalization owns commits; do not invent a future core revision or leave
linked-repository edits out of the final declaration.

Before closing any child phase, run its `sase bead epic-symbols` command; resolve
consumed exemptions or re-key genuinely deferred ones to a still-open owner. Record
discovered follow-up work only as `PROPOSED FOLLOW-UP:` notes on the assigned bead. Do
not create task beads. A check failure reproduced identically on clean base is
documented with its witness and existing tracker and does not, by itself, block closure
of completed work.

The child epic's land agent reviews the integrated acceptance evidence and required
verification. The user explicitly delegated completion of `sase-1qz.6.1.4` to this child
epic's land agent. After child work and ordinary host child-epic settlement are
complete, it runs:

```bash
sase bead epic-symbols sase-1qz.6.1.4
sase bead close sase-1qz.6.1.4 --note "<implemented transaction and verified acceptance evidence>"
```

If host ordering requires the child epic to settle first, put this concrete close step
in the completion handoff. Never force a descendant close. No coding phase may close
that parent phase. Neither this child land agent nor its coding phases may close
`sase-1qz.6.1`, `sase-1qz.6`, `sase-1qz`, or any other ancestor.
