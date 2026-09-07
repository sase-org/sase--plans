---
tier: tale
title: Implement remote question, gate, and notification attention parity
goal:
  A followed remote agent's pending question or gate shows up in Focus with the same
  visual language as local attention, is readable before it is answered, and is answered
  or approved exactly once through a journaled consume-once mutation that tells a losing
  controller "Already answered on <host>" with the settled result, while reconnects and
  refreshes never re-toast an attention request the viewer has already been shown.
size: medium
proposed_by: bbugyi200.athena.sase-xe.14
bead: sase-xe.14
status: done
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.14](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.14.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-xe.14](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.14.md)
- **COMMITS:**
  - [287048d](https://github.com/sase-org/sase/commit/287048d601b6a2003aa01e12743c2ed053c7c982)
    — feat(dispatch): surface and answer remote question/gate attention in Focus

# Plan: Implement remote question, gate, and notification attention parity

## Objective

Complete phase `sase-xe.14` as one bounded cross-repository implementation: project the
owning host's pending questions and gates into a safe, versioned attention contract;
serve and journal them over the fleet API; carry them through the federation worker, the
Python facade, a durable CLI operation, and ACE; render remote attention in Focus using
the existing local attention vocabulary; let the reviewer read the decision and command
detail through bounded opaque handles before approving; execute the answer or approval
as a consume-once mutation on the owning host under the operation journal; and
deduplicate viewer notices by origin plus request identity so reconnects and periodic
refreshes never repeat a toast.

This phase adds no new discovery, no new timer, and no new remote read tier. Attention
is fetched only for logical keys this viewer already follows, on the same refresh that
already runs for followed rows.

## Settled constraints

- Owner resolution stays on the owning host. A viewer never reads a gate bundle, a
  question response directory, a notification file, or any local path for a remote row;
  it holds an opaque attention request key, a bounded projection, and opaque content
  handles. Answering is always a request to the owner, never a local write.
- Attention identity is `(origin installation ID, request key)`. The request key is
  derived on the owner from the notification's pending-action identity, which is the
  same identity `sase gate answer` and the mobile bridge already consume, so the same
  request is one request no matter which surface answers it.
- Every answer/approval carries a controller-scoped operation key, a payload
  fingerprint, the exact attention request key, and the observed attention revision as a
  precondition. A repeat with the same key and payload returns the original receipt; the
  same key with a different payload conflicts; an expired or tombstoned key is rejected
  rather than retried as a new mutation.
- Consume-once is enforced twice and reported once. The journal admits a single
  execution per operation key, and the host's existing gate executor refuses a second
  answer. Either refusal surfaces as the `already_settled` outcome carrying the settled
  response and the host name, rendered as "Already answered on <host>", never as a
  failure and never as a second execution.
- An answer submitted against a superseded attention revision is refused with a typed
  `stale_revision` reason naming the host. It never falls through to the current
  request, because the reviewer read a different question than the one now pending.
- A gate runs commands. Its follow-up launch happens on the execution host inside the
  journal-admitted execution, through the host's own gate executor and gate-shell
  settlement. No branch of this phase may execute a gate outside the journal because its
  button looks like notification UI.
- Toast dedupe is durable and viewer-local. The ledger is keyed by origin installation
  ID plus request key and stores the announced revision, so an ACE restart, a reconnect,
  or a resync replays no toast; a genuinely new revision (a superseded and re-asked
  request) announces once.
- Laziness is preserved. With no enrolled machine, nothing new runs: no attention read,
  no ledger write, no timer. Attention is requested only for followed logical keys, in
  the request that already fetches followed rows, and detail/preview bytes are fetched
  only when the reviewer opens the modal. All I/O stays off the Textual event loop and
  off the serial message pump, the answer runs as a tracked proc, and selection is
  re-read after every await.
- `remote_dispatch` gates every new surface. With the flag off, the new CLI children,
  the ACE attention action, the facade attention calls, and the notice ledger refuse or
  no-op with the existing beta-gate message, and the Off branch stays trivially
  deletable.
- Nothing here builds on `agents_sync`, and no viewer dismissal, read-mark, or unfollow
  may mutate the remote authority's notification record.

## Implementation

Before starting, open the Rust core repository with the `/sase_repo` skill (every
`crates/...` path below is relative to that checkout) and read the `tui_perf.md` and
`lint_and_test.md` reference memory with `/sase_memory_read`. Do not edit anything under
`sase/memory/`; this epic leaves memory untouched.

1. Define the fleet attention contract in `sase-core` and expose it to Python.
   - Add `crates/sase_core/src/fleet_attention.rs` (a new module beside
     `fleet_mutation`, re-exported from `lib.rs`) with versioned wire types: attention
     kind (`question`/`gate`), attention state (`pending`/`settled`/`expired`/
     `unknown`), a request key (origin installation ID plus the owner's opaque request
     ID and pending-action prefix) with a canonical string form, an attention revision
     (request key plus a `u64` derived on the owner), and a bounded attention entry
     carrying kind, state, the correlated logical key and logical locator when known,
     title, a bounded summary, selectable option IDs and labels, whether feedback is
     required, the question form shape when the request is a question, an optional
     preview content handle, and the settling host label and settled response when the
     request is already settled.
   - Add the pure projection `project_fleet_attention` taking the origin installation
     ID, the owner's notification rows with their resolved action states, and the
     resolved rows' logical identity (agent label plus logical key/locator), and
     returning a deterministic, deduplicated attention snapshot. Reuse the existing
     `pending_action_identity`, `MobileActionKindWire`, and
     `mobile_action_detail_from_notification` helpers rather than reimplementing gate
     and question shape parsing. Correlate a request to a row through the gate
     notification's `origin_agent` action-data key and, for questions, through the
     asking agent's question session identity; an uncorrelated request is still
     returned, keyed to no row, so it can never be silently dropped. The projection
     never serializes a bundle path, response directory, request path, preview path,
     PID, or credential.
   - Add the answer contract: a validated attention intent (kind, request key, observed
     revision, selected option IDs, optional bounded feedback, and the question form
     fields the existing question action already accepts), the request envelope (scoped
     operation key, target installation pin, fingerprint, acceptance window), a typed
     outcome (`applied`, `already_settled`, `stale_revision`, `unknown_request`,
     `capability_missing`, `precondition_failed`), a receipt carrying the settled
     response, settling host label, and a safe message, the durable record, and the
     decision request/decision/response types.
   - Add `fleet_attention_payload_fingerprint`, `validate_fleet_attention_request`,
     `decide_fleet_attention_replay` built on the same replay semantics
     `decide_fleet_mutation_replay` uses, the kind-to-capability map
     (`attention.answer_question`, `attention.approve_gate`), and
     `evaluate_attention_precondition`, which compares an intent against an observed
     attention entry and returns a typed refusal (unknown request, stale revision,
     already settled, capability missing, invalid option).
   - Add the notice-dedupe decision: a ledger entry (request key, announced revision,
     announced-at timestamp) and `decide_attention_notices`, a pure function taking the
     current entries, the existing ledger, a retention window, and the current time, and
     returning the entries to announce, the entries suppressed as already announced, and
     the pruned/updated ledger. Announcing is keyed by request key plus revision, so a
     reconnect that re-delivers the same request announces nothing while a new revision
     announces exactly once.
   - Cover in Rust unit tests: projection determinism and correlation, secret/path
     rejection, fingerprint stability, replay accept/return/conflict/expiry, every
     precondition refusal reason, settled entries reporting the settling host, and
     notice dedupe across a simulated reconnect, a superseding revision, and ledger
     pruning.
   - Expose `fleet_project_attention`, `fleet_attention_payload_fingerprint`,
     `fleet_validate_attention_request`, `fleet_evaluate_attention_precondition`, and
     `fleet_decide_attention_notices` through `crates/sase_core_py` in the existing
     dict-in/dict-out style with `py.allow_threads`, document them in the binding module
     docstring, and add them to `tools/validate_sase_core_rs`'s required binding list
     and to the binding-name list asserted by
     `tests/test_fleet_contract_sase_core_rs.py`.

2. Serve and journal attention from the target gateway.
   - Advertise the new resource capabilities from `fleet_reads.rs`, derived from the
     artifact record alone: `attention.answer_question` for a non-terminal row with a
     pending question marker, and `attention.approve_gate` for a non-terminal gate-shell
     row. Terminal rows and rows with neither advertise nothing.
   - Add `FLEET_SCOPE_ATTENTION_READ` and `FLEET_SCOPE_ATTENTION_RESOLVE` to
     `fleet_auth.rs` and to the default scope set so `fleet_capabilities` publishes
     them.
   - Add `POST /api/fleet/v1/attention` to `routes.rs`: authenticate the read scope,
     negotiate the protocol version, accept a bounded list of logical keys (reusing the
     existing batch ceiling), read notifications through the existing
     `notification_bridge` (`list_notifications` plus `action_state` per row), resolve
     the requested rows through `fleet_reads`, and return the `project_fleet_attention`
     snapshot with the host's observation timestamp. When the request carries no logical
     keys the route returns an empty snapshot without touching the notification store.
   - Add `crates/sase_gateway/src/fleet_attention.rs` with a `FleetAttentionStore`
     modeled on `FleetMutationStore`: exclusive file lock, atomic 0600 write, receipt
     and tombstone retention through the acceptance window, `reserve` returning the
     replay decision, and `settle` binding the outcome, settled response, settling host
     label, and safe message.
   - Add `POST /api/fleet/v1/attention/resolve` to `routes.rs`: authenticate the resolve
     scope, verify the installation pin, re-project the current attention entry for the
     intent's request key, evaluate the precondition (returning specific codes for stale
     revision, unknown request, already settled, and missing capability), reserve the
     operation, then execute through the existing notification bridge — gate approvals
     via `execute_gate_action`, question answers via `execute_question_action` — settle
     the receipt, audit without prompts or secrets, and publish both a
     notifications-changed and an agents-changed invalidation so a connected viewer sees
     the settlement without polling.
   - Map the bridge's `conflict_already_handled` failure to an `already_settled`
     settlement carrying this host's label rather than to an error, so a controller that
     loses the race gets the settled result instead of a failure.
   - Add both routes, their records, scopes, and limits to
     `fleet_api_v1_contract_snapshot()` in `contract.rs` and regenerate the committed
     `crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json` through the existing
     snapshot test.
   - Add gateway tests: scope denial for each route, an empty request performing no
     notification read, a pending gate and a pending question projected with correlated
     rows, an accepted approval returning a settled receipt, identical replay returning
     the original receipt, changed payload conflict, stale revision refusal, a second
     controller receiving `already_settled` with this host's label, missing capability,
     bridge failure mapping, and invalidation publication.

3. Carry attention through the federation worker and the Python facade.
   - Add an `Attention { request, cache_only }` read operation and a
     `ResolveAttention { target, request }` mutation operation to
     `crates/sase_gateway/src/federation_worker.rs`. The read fans out across configured
     hosts exactly like `FollowedBatch`, honoring per-host deadlines, backoff, and the
     read cache; the resolve resolves exactly one host by alias, enforces the
     installation pin, honors the deadline and both concurrency limiters, posts
     `/attention/resolve`, and never reads or writes the read cache.
   - Add `attention`/`attention_sync` and `resolve_attention`/`resolve_attention_sync`
     to `src/sase/dispatch/federation/_facade.py`, with the disabled-read shape for
     reads and `FederationWorkerUnavailable` for the resolve when no machine is
     configured, mirroring `followed_batch_sync` and `mutate_sync`.
   - Add worker tests for deadline expiry, unknown alias, pin mismatch, cache behavior,
     and structured remote error propagation.

4. Build the source-side attention client, journal, and notice ledger.
   - Add `src/sase/dispatch/attention.py`: require the flag and a non-quarantined
     enrolled machine; fetch attention for a bounded set of followed logical locators;
     build the intent, operation key, and fingerprint through the new bindings; submit
     through the facade; and classify the response into a typed outcome with a
     host-named message, including the "Already answered on <alias>" message and the
     settled response for `already_settled`.
   - Add `src/sase/dispatch/attention_intent.py` mirroring `mutation_intent.py`: an
     atomic, lock-protected record per operation key holding alias, installation pin,
     kind, request key, observed revision, fingerprint, status, receipt, and timestamps,
     so a lost reply reconciles under the same key instead of issuing a new one.
   - Add `src/sase/dispatch/attention_notices.py`: a durable viewer-local ledger under
     the existing `fleet` state directory, using the same lock/atomic-write discipline
     as `follow_store.py`, that loads the ledger, calls
     `fleet_decide_attention_notices`, persists the returned ledger, and returns the
     entries to announce. With no enrolled machine and no entries it performs no write.
   - Keep each new module self-contained and under the repo's file-size norms; split a
     module rather than letting it grow past the existing `toobig` warning tier.

5. Add the durable CLI operation ACE submits.
   - Add the `sase machine attention {answer, approve}` group in
     `src/sase/main/parser_machine.py`, placed alphabetically after `agent`, with
     positional `ALIAS` and `REQUEST`, `approve` taking one or more option IDs
     positionally, `answer` taking the answer positionally, and short-aliased optional
     `-j/--json` and `-t/--timeout`. Follow the CLI rules memory: sorted subcommands and
     options, every public long option short-aliased, no required options, excellent
     `-h` text.
   - Route it from `machine_handler.py` to a new
     `src/sase/ops/commands/machine_attention.py` runner under a new
     `MACHINE_ATTENTION_ACTION` operation name in `src/sase/ops/names.py`. The runner
     prefers the durable request sidecar (ACE supplies the request key, observed
     revision, selection, and operation key) and otherwise resolves the current pending
     request from a fresh bounded attention read for direct human use. It emits the
     typed durable result and never falls back to another machine.
   - Register the new proc producer site in
     `src/sase/ace/tui/_proc_producer_sites_actions.py` and add
     `submit_machine_attention_action` to `src/sase/ace/tui/actions/agent_durable.py`,
     mirroring `submit_machine_agent_action` with a per-alias-and-request concurrency
     key.
   - Refresh `tests/completion/snapshots/cli_spec.json` with
     `just sync-completion-spec`.

6. Render remote attention in Focus and answer it from ACE.
   - Extend the fleet row projection (`src/sase/ace/tui/models/fleet_agents.py`,
     `src/sase/ace/tui/models/_agent_state.py`) with a `fleet_attention` payload and map
     an attention-bearing row onto the existing local attention statuses — `QUESTION`
     for a pending question, `WAITING INPUT` for a pending gate — so the existing status
     colors, ordering, and detail rendering apply unchanged and no new visual vocabulary
     is introduced. Accept the projection's `asking` lifecycle and `needs_attention`
     flag as the fallback when no attention entry is available yet.
   - Add `src/sase/ace/tui/actions/agents/_remote_attention.py`: fetch attention for
     followed logical locators inside the existing fleet refresh in `_fleet.py` (never a
     new timer, and skipped entirely when no machine is configured or no row is
     followed), run the notice-ledger decision off the event loop, and toast only the
     announced entries with a host-named message. Add `action_answer_remote_attention`,
     which re-reads the selection, opens the modal, and submits the answer as a tracked
     proc through the durable adapter with optimistic row state and a settled-outcome
     toast.
   - Add `src/sase/ace/tui/modals/remote_attention_modal.py`: shows origin and
     observation age, the request title and bounded summary, the selectable options or
     question form, and a bounded, digest-validated preview of the decision/command
     detail fetched through the existing `RemoteContentClient` before any approval is
     possible. The modal never renders a remote path and disables submission when the
     row's capability, freshness, or the flag does not allow it.
   - Register the action end to end: `src/sase/default_config.yml` (`unbound` by
     default, in the Agents sub-tab block), `keymaps/app_keymaps.py`,
     `keymaps/metadata.py`, `commands/_app_metadata_nav.py`, the agents help modal, and
     both availability paths (`_app_action_availability.py` and
     `commands/_availability_agents.py`), gated on the flag, a remote row, a
     non-quarantined origin, the advertised attention capability, and a pending
     attention entry.

7. Verify both repositories and close only the assigned phase.
   - Add Python tests for: flag-off refusal on every new surface with no remote call;
     attention read requested only for followed locators and skipped with an empty
     registry; two controllers racing one gate, where the loser receives
     `already_settled` with the settled response and the host name and no second
     execution; an answer against a superseded revision refused as `stale_revision`
     without touching the current request; duplicate submission under one operation key
     returning the original receipt; a lost reply reconciled under the same key; toast
     dedupe across a simulated reconnect and ACE restart with a new revision announcing
     once; capability-missing and unknown-request refusals; and the CLI runner's typed
     durable result.
   - Add ACE tests that a remote row with pending attention renders the local
     `QUESTION`/`WAITING INPUT` status, that the answer action routes to the remote
     mixin and never to a local question or gate path, that the modal requires the
     preview fetch before enabling approval, and that a local selection is unchanged.
   - Run `just install` (the Rust binding must be rebuilt from the linked checkout),
     `./scripts/check.sh all` in `sase-core`, then `just check` in the sase repo, using
     `/sase_monitor` when a lane runs long.
   - Record genuinely out-of-scope discoveries only as `PROPOSED FOLLOW-UP:` notes on
     `sase-xe.14`, including the pinned-core ratchet needed before CI's pinned-binding
     gate sees the new bindings.
   - Run `sase bead epic-symbols sase-xe.14`, resolve or re-key every remaining symbol,
     and close only `sase-xe.14` with a note naming the verification performed. Do not
     close `sase-xe` or any ancestor.

## Acceptance criteria

- With `remote_dispatch` off, `sase machine attention`, the ACE attention action, the
  facade attention calls, and the notice ledger refuse or no-op with the beta-gate
  message and perform no remote call and no state write.
- A followed remote agent with a pending question or gate shows the same `QUESTION` /
  `WAITING INPUT` status, coloring, and ordering as an equivalent local row, with origin
  and observation age visible.
- Opening the attention modal shows the request title, options, and a bounded,
  digest-validated preview of the decision and command detail before approval is
  possible, and never exposes a remote path.
- Approving a gate or answering a question executes exactly once on the owning host
  through its own gate executor and gate-shell settlement, inside the journal-admitted
  execution, and any follow-up launch the gate triggers runs on that host.
- A second controller answering the same request receives "Already answered on <host>"
  with the settled result, not a failure and not a second execution; an answer against a
  superseded revision is refused as `stale_revision` and never applied to the current
  request.
- Replaying an identical answer returns the original receipt, a changed payload under
  the same key conflicts, an expired key is rejected, and a lost reply reconciles under
  the same key rather than submitting a second answer.
- A reconnect, a resync, and an ACE restart re-deliver the same pending request without
  re-toasting it; a superseded and re-asked request toasts exactly once on its new
  revision.
- With an empty machine registry, ACE behavior, timers, remote traffic, and viewer state
  writes are unchanged; with machines enrolled but no followed row, no attention read is
  issued.
- `sase-core`'s `./scripts/check.sh all` (including the regenerated fleet contract
  snapshot test) and the sase repo's `just check` pass before the phase closes.
