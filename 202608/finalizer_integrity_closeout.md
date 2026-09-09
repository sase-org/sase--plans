---
tier: epic
status: done
title: Close finalizer protocol integrity gaps
goal:
  Make unconditional host-owned finalizers enforce their sealed plan and execution
  policy consistently before sase-rr can close.
parent_bead: sase-rr
phases:
  - id: seal-plan
    title: Seal and authenticate the execution plan
    depends_on: []
    description:
      "seal-plan: keep the authoritative finalizer plan in host-owned state, reject
      artifact or live-configuration drift before context publication and every
      dispatch, and bind worker requests to the authenticated turn and selection."
    size: medium
  - id: provider-contract
    title: Normalize provider identity and dispatch
    depends_on: []
    description:
      "provider-contract: canonicalize Python distribution identities across discovery
      and execution and replace exception-based callable-versus-factory detection with
      an explicit, single-invocation provider contract."
    size: small
  - id: execution-ledger
    title: Enforce bounded execution and immutable evidence
    depends_on:
      - seal-plan
      - provider-contract
    description:
      "execution-ledger: centralize whole-instance attempt, status, output, timeout, and
      evidence semantics across commit, command, and plugin finalizers, including
      fail-closed aggregation and bounded subprocess handling."
    size: medium
  - id: deterministic-reconcile
    title: Make declaration and commit reconciliation deterministic
    depends_on:
      - seal-plan
    description:
      "deterministic-reconcile: serialize context publication with submission
      acceptance, execute repository decisions in host context order, and prove clean
      transitions against the accepted obligations."
    size: medium
  - id: integrity-acceptance
    title: Run combined adversarial integrity acceptance
    depends_on:
      - execution-ledger
      - deterministic-reconcile
    description:
      "integrity-acceptance: exercise every confirmed integrity failure through focused,
      cross-repository, and live disposable-repository scenarios and repair any
      remaining in-scope defect before handing control back to the sase-rr land agent."
    size: medium
proposed_by: bbugyi200.athena.sase-rr.land--2
bead_id: sase-rr.5
create_time: 2026-09-09 19:50:21
---

- **PROMPT:**
  [prompts/202608/finalizer_integrity_closeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finalizer_integrity_closeout.md)
- **BEAD:**
  [sase-rr.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rr/sase-rr.5.md)

# Plan

## Context and scope

The `sase-rr` combined tree made pluggable finalizers unconditional and passed its
original live scenarios, but a post-implementation integrity audit against the current
tree found ten concrete correctness failures. These failures remain part of the parent
epic because the approved retirement plan required a host-owned sealed plan, uniform
per-instance attempt policy, bounded provider output, durable retry evidence,
deterministic multi-repository dispatch, and fail-closed outcomes before the legacy
controller could be retired.

The critical reproduction replaces the model-visible `finalizer_plan.json` with an empty
plan carrying an invalid digest. The controller accepts it and records aggregate success
with no instances. Related probes show that live configuration can diverge from the
recorded entry, plugin and commit execution do not share the advertised attempt budget,
provider-authored `skipped` can produce contradictory exit and artifact states,
packaging-equivalent distribution names disagree across discovery and the worker, pipe
limits are checked only after unbounded buffering, reactivation overwrites evidence,
submission JSON controls repository execution order, provider `TypeError` is mistaken
for a factory signal, worker requests report the wrong selection and omit turn identity,
and context publication can race a successful submission.

This child epic fixes only those confirmed correctness defects. Provider-description
handshakes, trigger/profile expansion, repository-aware command modes, and new doctor or
replay features are capability work and are deliberately excluded. Do not close
`sase-rr`, close `sase-ro`, update the parent plan status, deploy generated skills, or
perform the parent's final Symvision closeout in a child phase; the `parent_bead` link
returns that work to the waiting land agent.

Shared wire, digest, result-validation, and aggregation behavior belongs in `sase-core`.
Open that linked repository through `/sase_repo`, update its Rust tests and bindings
where the frontend-neutral contract changes, run its full `just check`, and then update
the Python side through the supported binding/floor workflow. Host process
orchestration, locks, subprocesses, runner-owned state, filesystem evidence, and
provider discovery remain in this repository. Do not edit SASE memory files under this
plan.

## Phase 1: seal and authenticate the execution plan

Make the plan resolved before the model turn authoritative throughout declaration and
execution. Do not trust a digest stored beside the artifact it supposedly authenticates.
Carry the host-owned expected plan, or sufficient independently held authentication
state, across the provider turn and resume paths. Treat `finalizer_plan.json` as
model-visible evidence rather than the source of authority.

Before initial context publication and again immediately before every instance dispatch:

- parse the full structure strictly instead of silently dropping malformed entries;
- recompute and validate the plan digest using the shared Rust contract;
- compare ordered entries, required instances, selectors, provider refs, dependency and
  resolved order, policy, configuration digests, and provenance against the
  authoritative snapshot;
- compare the live executable instance configuration with the sealed entry so a long
  turn cannot run a changed command, provider, or policy; and
- write a durable failed aggregate with a stable `plan_integrity_failed` diagnostic
  before any provider side effect when corruption or drift is found.

Bind `FinalizerExecutionContext` and every external worker request to that same
authenticated state: real run ID, agent ID, turn nonce, plan and context digests,
selected instance IDs in resolved order, accepted payloads, and host-issued obligations.
Never recompute `selected` from current defaults during execution. Preserve safe no-op
behavior outside a SASE agent and for a genuinely host-resolved empty plan.

Add adversarial tests that mutate, truncate, reorder, add, and remove entries after
resolution; change `provider_ref`, `max_attempts`, configuration, provenance, and
dependency order; forge or omit the digest; and remove a required instance. Assert that
each case fails before execution and that a legitimate `%final:none` plan still
succeeds. Cover resume/recovery so authority is not lost across a second model turn.

## Phase 2: normalize provider identity and dispatch

Use one packaging-compatible distribution-name normalization rule everywhere a provider
package is discovered, configured, compared with required plugins, deduplicated, or
looked up by the isolated worker. Keep raw metadata names only for user-facing display
and provenance. The normalized provider ref must satisfy the Rust lowercase-slug wire
contract without making installed mixed-case, hyphen, underscore, or dot variants
unaddressable.

Replace `dispatch_provider_request()`'s catch-all `TypeError` factory heuristic. Define
and document one unambiguous entry-point shape, or inspect callable shape before
invocation, so provider code runs at most once and a real provider `TypeError` is
reported unchanged as provider failure. Preserve method-bearing providers and the chosen
compatibility form deliberately; reject ambiguous shapes with a precise diagnostic.

Exercise real entry-point metadata fixtures with mixed-case and punctuation-equivalent
distribution names from discovery through isolated execution. Add call-count and error
identity assertions proving a provider is never invoked twice and internal `TypeError`
is not rewritten as a missing-operation error.

## Phase 3: enforce bounded execution and immutable evidence

Define `max_attempts` as a host-owned budget of whole mutating instance executions.
Maintain a controller-owned per-instance ledger across cycles and reactivation. Consume
the budget before execute, distinguish retryable from terminal failures with a closed
host policy, and enforce the same semantics for built-in commit, built-in command, and
external plugin providers. Keep declaration recovery, conflict repair, and controller
cycle/no-progress limits separate and explicitly named. Do not let a provider choose its
retry count.

Choose one fail-closed status model. For the current protocol, reject provider-authored
`skipped`; only the host may record an untriggered instance as skipped. Make Rust result
validation and aggregation agree with the controller, and ensure an explicit controller
failure can never be overwritten as aggregate success in `finalizer_result.json`.

Replace post-`communicate()` output checks with incremental, deadlock-safe draining of
stdout and stderr for plugin, command, and stitch/repair subprocesses. Retain only
bounded diagnostic data, terminate the full process group as soon as a hard cap or
timeout is exceeded, and reap it reliably. Keep configurable limits inside hard global
maxima and avoid unbounded input or output allocation.

Allocate monotonic attempt IDs in the controller. Store stdout, stderr, evidence, and
diagnostics in immutable per-attempt paths that cannot be overwritten by retry or
reactivation. Merge prior and current evidence instead of replacing it, associate each
diagnostic with its attempt, and require unique increasing attempt numbers plus coherent
terminal status in the shared wire validator. Conflict repair success must add explicit
success evidence rather than silently changing the outcome.

Test retryable and terminal failures at the exact budget boundary for all provider
kinds, later-dirt commit reactivation, all-skipped/failed aggregation, oversized output
on either pipe, simultaneous stdout/stderr pressure, timeout with descendants, and
artifact/evidence preservation across retries and controller cycles.

## Phase 4: make declaration and commit reconciliation deterministic

Give context publication and submission acceptance a single documented lock order. Under
that order, re-read and validate the current context immediately before atomically
accepting a submission so `sase final submit` cannot report success for an already stale
context. Add deterministic interleaving tests using synchronization hooks rather than
timing sleeps, including simultaneous republish and submit attempts.

Treat the commit declaration as a map of decisions only. Iterate the ordered repository
obligations from the authenticated host context and look up each decision by opaque
repository ID; never use JSON object or payload order to choose mutation order. Preserve
the existing rule that a conflict in the first repository blocks all later dispatch.

When execution observes a repository already clean, prove which accepted dirty
obligation disappeared and why before returning success. If the accepted context is no
longer current, fail or recover through the existing bounded stale-declaration path
before mutation. Cover reversed manifest order, multi-repository conflict blocking,
post-submit cleanup and edits, and stale clean-transition evidence.

## Phase 5: run combined adversarial integrity acceptance

Reinstall the combined dependency state, then run the complete focused finalizer,
invocation, plugin, declaration, commit, reporting, and fakey suites. Exercise the
confirmed probes through the public plan/declaration/controller seams rather than only
unit-testing helpers. Use disposable Git repositories and local bare remotes for live
mutation cases; never point acceptance at a user repository.

The retained acceptance matrix must demonstrate:

1. every plan artifact corruption and live config drift case fails durably before a
   provider runs, while authentic empty selection remains a success;
2. `%final` overrides and required instances reach workers with truthful ordered
   selection and turn/context identity;
3. packaging-equivalent plugin names discover and execute, and provider exceptions are
   single-invocation and accurately diagnosed;
4. commit, command, and plugin attempts share the documented budget semantics and leave
   immutable evidence for every retry/reactivation;
5. provider skip, explicit failure, timeout, oversized output, and descendant process
   cases fail closed with bounded resource use and matching durable aggregate status;
6. reversed repository decisions still execute in host context order, first-repository
   conflict blocks later repositories, and clean/stale transitions are proven; and
7. the original nine `sase-rr.4` live scenarios remain green, including declaration
   recovery, later-finalizer dirt, conflict resume, refusal, and intentional handoff.

Run `just check` after each repository's changes as required. Because the final combined
tree crosses the Rust binding, invocation identity, provider subprocesses, and broad
finalizer safety surface, run the primary repository's `just check-full` only through
`/sase_monitor` and inspect its complete output. Run the linked core repository's full
`just check`. Fix all in-scope failures and record unrelated failures as proposed
follow-ups on this phase for the parent land agent to disposition under
`/sase_new_task`; do not weaken assertions or declare the child complete with any
confirmed finalizer defect outstanding.
