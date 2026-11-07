---
tier: tale
title: Stop fencing booting monitor follow-ups to needs_attention
goal:
  A spawned monitor follow-up that is still booting or queued is proven live by its
  spawn-time launch receipt, so concurrent terminal-delivery reconciliation never fences
  it and it adopts its delivery instead of crashing; dead receivers are still fenced.
size: medium
proposed_by: bbugyi200.athena.19j
proposed_with: claude/opus
create_time: 2026-11-07 09:27:55
status: wip
---

# Plan: Stop fencing booting monitor follow-ups to `needs_attention`

## Problem

Monitor follow-up agents intermittently die on their first step with:

```text
WorkflowExecutionError: Step 'main' failed: Error: conflict: cannot transition terminal
delivery from needs_attention to acknowledged
```

The most recent victim was phase bead `sase-1rq.4.1.2`. Its planner staged an unfiltered
`macos-ci-check` monitor (`vkv33r475zps`). The monitor finished at 13:39:44Z with the
expected red result. Settlement spawned the follow-up `sase-1rq.4.1.2--1` at 08:40:40
EST. That follow-up waited about 3.5 minutes for a runner slot (8/8 in use), then failed
in `prepare_provider_invocation` → `adopt_ordinary_continuation_delivery` with the error
above, after 5s of runtime. Nothing resumes a failed follow-up, so the bead was stranded
and its epic's land agent stayed blocked.

This is a recurring production failure, not a one-off. The same traceback appears in
five `~/.sase/workflows/*/…ace-run-*.txt` logs:

- `2026-10-09` (follow-up of `sase-1io.7.5`)
- `2026-10-20` (follow-up of `sase-1n8.6`)
- `2026-10-23` (bob-cli `bob-cli-at.1`)
- `2026-11-01` (follow-up of `sase-1pj.6.1.8`)
- `2026-11-07` (`sase-1rq.4.1.2`)

Every victim was an ordinary monitor follow-up, and every one had a multi-minute
duration before the adopt step.

## Root cause

The delivery record for an ordinary monitor continuation goes
`pending → reserved → dispatching` in the settling monitor supervisor
(`claim_ordinary_continuation_dispatch` in `src/sase/monitor/continuation_delivery.py`).
The receiver itself moves it to `acknowledged` only when it reaches provider invocation.
Between those two points, any other process that calls
`sase.monitor.store.list_monitors()` runs `reconcile_terminal_deliveries()` →
`reconcile_terminal_delivery()` → `resume_monitor()` on every terminal monitor that
still has an active (`pending`/`reserved`/`dispatching`) delivery. Callers that trigger
this include:

- `sase monitor list/show/resume/stop` (`src/sase/main/monitor/common.py`)
- `sase tool` stop/outputs
- named-agent kill/dismiss (`sase.agent.running._stop_live_monitors_for_named_agent` →
  `owned_live_monitors_for_name`)
- cleanup stop intents

These callers run in any of the ~8 concurrent agents, across all projects.

`resume_monitor` collects receiver proofs with `collect_receiver_proofs()`
(`src/sase/monitor/resume_adoption.py`) and asks the Rust `decide_resume_adoption` for a
verdict. In the Rust core (`sase_core` `continuation/delivery.rs`), a proof with
`spawn_recorded && !process_alive && !discoverable` counts as **stale**. For a stale
active record the verdict is `needs_attention` with that key in `fence_keys`, even for
an ordinary (non-manual) reconcile. `apply_resume_adoption` then rewrites the record to
`needs_attention`. `needs_attention` is terminal, so the receiver's later
`dispatching → acknowledged` adoption is refused and the turn crashes.

The proof is wrong for a receiver that has been spawned but has not booted yet:

- `spawn_recorded` becomes true right after spawn. In `followup.py::_spawn`,
  `update_delivery_workspace(...)` sets `workspace_identity`, and `_record_launched`
  sets `monitor_followup_agent`/`monitor_followup_outcome=launched` on the monitor meta.
- `process_alive`/`discoverable` come only from
  `resolve_exact_agent(project, identity)`. This is a name lookup over artifact records
  (`agent_meta.name == identity`). As `spawn_agent_session_successor` documents, "the
  subprocess does not report its own name back until it boots". Until the child boots
  and its artifact is indexed under the reserved name, the lookup returns nothing (or an
  older same-named record), so the receiver looks dead. Boot alone took ~16s for a
  launch observed today under load (`launch_submitted_at` 14:04:25 → `booted_at`
  14:04:41). A read-only probe of the live fleet showed every booted agent, including
  ones still queued for a runner slot, resolving as alive. The exposure is the spawn →
  boot/index window.
- The spawner already knows the receiver's real pid and artifacts dir.
  `spawn_agent_session_successor` returns an
  `AgentLaunchResult(pid=…, artifacts_dir=…)`, and `launch_turn_followup` threads both
  into `record_launched(...)`. But `record_followup_launched`
  (`src/sase/turns/followup_records.py`) drops them, and neither reaches the delivery
  record or any proof.

So a live, correctly reserved receiver gets fenced whenever an unrelated
`list_monitors()` call lands in its boot window. The fence reason is also misleading:
`apply_resume_adoption` always writes `superseded by manual resume branch <b>`, even for
this ordinary stale fence. And the receiver's crash message drops the disposition
reason, which made the incident hard to diagnose.

## Design

Make the receiver proof use the spawn-time launch receipt (the receiver's own pid and
process identity) instead of depending only on name resolution. Keep every Rust verdict
rule unchanged, so a receiver whose recorded process is really gone is still fenced.

This change is Python-only. Process probing and receipt collection already live in
Python (`collect_receiver_proofs` "collects launch-receipt and process-identity proofs
outside the lock"). The Rust-owned `ContinuationReceiverProofWire` (`branch`,
`identity`, `discoverable`, `process_alive`, `spawn_recorded`), the delivery record
wire, and `decide_resume_adoption` do not change, so no `sase-core` change or
`sase-core-revision.txt` bump is needed.

## Changes

### 1. Persist a receiver launch receipt before the spawn is marked

Add a small module, `src/sase/monitor/delivery_receipts.py`. Keep
`src/sase/monitor/followup.py` from growing; it is already over 500 lines. The module
owns receiver launch receipts stored beside the delivery records:

- Store path:
  `<monitor artifacts>/continuation/delivery/receivers/<same safe file name as the delivery record>.json`.
  Reuse `_record_path`'s naming via a shared helper in `delivery.py`, or expose a public
  `delivery_record_filename(key)`. `load_delivery_records` globs only
  `continuation/delivery/*.json` (non-recursive) and nothing else scans that directory,
  so a subdirectory is safe.
- `record_receiver_launch(artifacts_dir, key, *, identity, pid, receiver_artifacts_dir)`
  writes
  `{schema_version: 1, key, identity, pid, process_identity, receiver_artifacts_dir, recorded_at}`
  atomically with `write_json_atomic`, under the delivery store lock
  (`delivery_store_lock`). `process_identity` comes from
  `sase.monitor.identity.process_identity(pid)`. When `pid` is not a positive int,
  record nothing and return `None`.
- `load_receiver_launch(artifacts_dir, key)` returns the parsed receipt, or `None` for a
  missing or unreadable file. Never raise into proof collection.
- `receiver_launch_alive(receipt, identity)` returns `True` only when the receipt's
  `identity` equals the delivery identity and
  `supervisor_is_alive(receipt["pid"], receipt["process_identity"] or None)` holds. A
  recycled pid fails the identity comparison and counts as dead.

In `followup.py::_spawn`, right after `spawn_agent_session_successor(...)` returns and
**before** `update_delivery_workspace(...)`, call
`record_receiver_launch(artifacts_dir, claim_key, identity=frozen_name, pid=result.pid, receiver_artifacts_dir=result.artifacts_dir or None)`
when `claim_key is not None`. This order matters: from the instant any proof could show
`spawn_recorded`, the receipt exists. Treat a receipt write failure as best-effort
(log/ignore), just as a failed `update_delivery_workspace` is today. It must not turn a
successful spawn into a failed launch.

Before relying on this, confirm that `AgentLaunchResult.pid` is the long-lived
`run_agent_runner.py` process, i.e. the same pid the receiver later records as
`agent_meta.pid` and uses for its workspace RUNNING claim, and not a short-lived
intermediate. Today's evidence says it is: the RUNNING claim is taken with this pid, and
a live relaunch showed `agent_meta.pid` equal to the runner process.

### 2. Use the receipt in receiver proofs

In `collect_receiver_proofs` (`src/sase/monitor/resume_adoption.py`), for each delivery
with an identity, also load the receipt for that delivery key:

- If a receipt exists for this identity, `spawn_recorded = True`.
- `process_alive = (name-resolved record is alive) or receiver_launch_alive(receipt, identity)`.
  Set `discoverable` the same way.
- With no receipt, keep today's behavior exactly. Older in-flight records and other
  writers stay unchanged.

Also make `receiver_is_live(project_name, identity)` consult receipts when its callers
have a key. If no caller has one, leave it alone and say so in the commit message.

### 3. Record an honest fence reason

In `apply_resume_adoption`, keep `f"{MANUAL_SUPERSEDE_REASON_PREFIX} branch {branch}"`
for `kind == "manual_revision"` fences. For ordinary fences (the stale-receiver
verdict), use the decision's `reason` (`receiver ownership is ambiguous`) plus the
identity, e.g.
`receiver ownership is ambiguous: dispatching receiver acme--1 has no live process`. The
`needs_attention` record should then say why it was fenced.

### 4. Make the receiver's refusal self-explanatory

In `adopt_ordinary_continuation_delivery` (`src/sase/monitor/continuation_delivery.py`),
when the transition is refused because the stored record is already terminal
(`needs_attention`, `cancelled`, `nonlaunchable`, or `settled`/`acknowledged` by a
different identity), raise a `ValueError` subclass (e.g.
`ContinuationDeliveryFencedError`). Its message names:

- the monitor id and branch
- the stored disposition and `disposition_reason`
- the receiver identity
- that the provider was not invoked
- `sase monitor show <monitor_id>` for inspection

Keep the behavior otherwise identical: the provider must still never run for a fenced
delivery. The idempotent re-adopt by the same identity must keep succeeding. Callers
already convert the `ValueError` into `LLMInvocationError` → `WorkflowExecutionError`,
so the improved text reaches the failure notification and error report.

### 5. Docs

Update the `sase monitor resume` paragraph in
`docs/reference/automation/monitor-operations.md`, which currently says "A dispatching
branch whose launch receipts or process identity cannot prove the receiver uninvoked is
recorded as `needs_attention`". Say that settlement records the spawned receiver's pid
and process identity as its launch receipt. A receiver that is still booting or queued
for a runner slot therefore counts as live, and only a branch whose recorded receiver
process is gone is fenced. Also say that `sase monitor list`/`show` and agent
stop/dismiss run this same terminal-delivery reconciliation. Read the `docs_writing`
reference memory (`sase memory read docs_writing.md -r "<why>"`) before editing `docs/`.

## Tests

Extend the monitor resume tests in `tests/monitor/`. Reuse `_terminal_monitor`,
`_fake_spawn`, and `_sandbox_home` from `tests/monitor/_resume_fixtures.py`.

1. **Regression for the incident (new).** Model it on
   `test_resume_refuses_stale_receiver_without_live_process` in
   `tests/monitor/test_monitor_resume_manual.py`:
   - Claim an ordinary `failed` branch for `acme--1`.
   - Write a receipt whose pid is a live process you own (`os.getpid()` or a sleeping
     child) and whose identity is `acme--1`.
   - Call `update_delivery_workspace`.
   - Leave no resolvable `acme--1` artifact.
   - Run `reconcile_terminal_delivery(record)` (and/or `resume_monitor(record)`).

   Assert:
   - the delivery stays `dispatching`
   - the result is `existing_receiver`
   - no spawn was captured
   - a following `adopt_ordinary_continuation_delivery()` with the `acme--1` delivery
     env returns `disposition == "acknowledged"` instead of raising

2. **Dead receipt is still fenced (new).** Same setup, but the receipt's pid is a reaped
   child (spawn and `wait()` a trivial subprocess) or the identity token does not match.
   Assert `needs_attention` with the new ordinary fence reason (change 3).
3. **Existing stale test stays green.**
   `test_resume_refuses_stale_receiver_without_live_process` has no receipt and must
   keep fencing. Update only its reason assertion if it checks one.
4. **Spawn path writes the receipt before the workspace mark (new).** Drive
   `launch_followup_agent`/`resume_monitor` with a fake spawn. Assert that:
   - a receipt with the fake pid and artifacts dir exists after launch
   - it was written before `update_delivery_workspace` (e.g. patch
     `update_delivery_workspace` to assert the receipt file already exists)

   `_fake_spawn` returns `pid=1111`. Make sure no test accidentally treats a real live
   pid 1111 as the receiver: patch liveness or use a known-dead pid where a test expects
   a stale verdict.

5. **Receiver refusal message (new).** Fence a record to `needs_attention` with a
   reason. Assert that `adopt_ordinary_continuation_delivery()` raises the new error
   type and that its message contains the monitor id, `needs_attention`, the reason, and
   the identity. Also assert the existing idempotent re-adopt path is unchanged
   (`tests/monitor/test_continuation_delivery_adoption.py`).

Also run the existing suites:

- `tests/monitor/test_monitor_resume_delivery.py`
- `tests/monitor/test_monitor_resume_manual.py`
- `tests/monitor/test_continuation_delivery_adoption.py`
- the fakey monitor e2e tests (`tests/fakey/test_monitor_capacity_e2e.py`,
  `tests/fakey/test_monitor_capacity_zero_weight_wait_e2e.py`)

## Verification

- Read the `lint_and_test` reference memory before finishing and run the gate it
  prescribes (`just check` through `sase tool run`; do not run `just check-full`).
- Run the targeted pytest files above.

## Non-goals

- Changing the Rust verdict table (`decide_resume_adoption`) or the delivery/proof wire
  schemas.
- Stopping read paths (`list_monitors`) from reconciling terminal deliveries. That
  reconciliation is the intended recovery for interrupted follow-up launches; the fix is
  to feed it truthful proofs.
- Letting a fenced receiver self-heal from `needs_attention`. The state machine stays
  strict.
- Gate-turn handoff reconciliation (`src/sase/gate_turn/`), which uses a separate
  mechanism.
- Changing `_receiver_process_alive`'s identity handling. Some writers stamp
  `agent_meta.process_identity` with the creator's pid identity, so comparing it against
  the receiver pid could produce false "dead" verdicts.
