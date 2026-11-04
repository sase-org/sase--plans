---
tier: epic
title: Recover reboot-stranded epic work once per boot
goal: Recover prior-boot dead epic waiters and land agents automatically, without
  duplicate launches, stale-owner claim failures, or interference with live agents
  and hijack custody.
parent_bead: sase-1ra.11
phases:
- id: core_policy
  title: Rust boot-recovery policy and durable attempt contract
  size: medium
  depends_on: []
  description: 'core_policy: first wait for all four external implementation gates;
    add the pure Rust eligibility and attempt-state contract, Python bindings, adapter,
    and policy tests.'
- id: owner_claims
  title: Safely replace claims held by proven prior-boot dead owners
  size: medium
  depends_on:
  - core_policy
  description: 'owner_claims: collect authoritative owner evidence and implement Rust-guarded
    claim replacement for reboot-dead owners in the runner and waiting-claim paths,
    retaining live-owner protection.'
- id: recovery_worker
  title: Durable per-epic recovery execution and deduplicated notices
  size: medium
  depends_on:
  - owner_claims
  description: 'recovery_worker: implement the locked per-boot ledger and durable
    worker, fresh safety checks, safe epic-launch preview, admission-aware execution,
    and one notice per recovery attempt.'
- id: scheduler
  title: AXE startup reconciliation with bounded recovery admission
  size: medium
  depends_on:
  - recovery_worker
  description: 'scheduler: register the boot-recovery job and durable proc submission,
    retain pending deferrals across ticks, and prove repeated starts and concurrent
    schedulers do not duplicate recovery.'
- id: acceptance_docs
  title: Verify the reboot incident and document watchdog replacement
  size: medium
  depends_on:
  - scheduler
  description: 'acceptance_docs: exercise the full simulated-reboot fixture, document
    the reconciler and watchdog retirement, and leave complete verification evidence
    for the child epic lander.'
proposed_by: bbugyi200.athena.sase-1ra.11
proposed_with: codex/gpt-6-astra
create_time: 2026-11-04 18:37:26
status: wip
bead_id: sase-1ra.11.1
---

- **PROMPT:** [prompts/202611/boot_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/boot_recovery.md)
- **PARENT:** [202611/next_ten_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202611/next_ten_task_beads.md)
- **BEAD:** [sase-1ra.11.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ra/sase-1ra.11.1.md)

# Recover reboot-stranded epic work once per boot

## Scope and evidence

This child epic implements only `boot_recovery` from
`plan:202611/next_ten_task_beads.md`, delegated by phase `sase-1ra.11`, addressing task
`sase-1qj`. The approved `auto_relaunch = yes` decision is final. Do not ask it again.
The parent's `next_epic = yes` decision belongs to its lander, not this child. No memory
edits are authorized. Zombie-epic reporting and the all-children-closed invariant remain
with sibling `sase-1ra.12`.

The 2026-11-01 reboot left 27 dependency waiters without live processes or `done.json`:
16 land agents and 11 phase agents. Later Continue forks failed with
`already in_progress and assigned` because their names differed from dead owners. The
read of `sase-1ra.11` showed no earlier implementation notes. At planning baseline
`0cc9fad474`, neither a `boot_recovery` module nor the scheduler job exists; recent
history contains no matching completed recovery implementation.

Read the source task, parent design, and current phase notes again before coding.
Reproduce the failure with isolated fixtures; do not reboot a host or relaunch real
epics as a test. Record baseline commits and before/after results in phase notes.

This is an epic because eligibility, ownership mutation, durable launch execution, and
scheduler orchestration each have separate failure and concurrency contracts. Each phase
is bounded direct implementation work. The serial dependency chain also prevents
competing core-pin updates.

## Mandatory implementation gates and ownership

Before **any implementation file change**, `core_policy` must confirm all four beads are
closed using `sase bead read`:

- `sase-1qz.6.1`: keep-work recovery and crash repair;
- `sase-1qz.7`: start-over recovery;
- `sase-1r2`: publication/release core-pin freeze;
- `sase-1ra.9`: this parent epic's `skew_core` phase.

All were still in progress during planning. Finish read-only exploration first. If any
remains open, use `/sase_monitor` for a pure wait (`-w 0`) that checks those beads every
15 minutes and resumes implementation after they close. Use the `WAITING <bead-id>` /
`WAITED <bead-id>` status pair. Never poll inline, mutate a gate bead, or create
speculative core edits before the gate. Re-read instructions and landed APIs after the
wait; the inspected interfaces may have changed.

Before editing, inspect `sase agent list` for newly active file owners. The parent
design's protected files remain protected: `core/wait_dependency_resolution/`,
`ace/scheduler/stale_running_cleanup.py`, `agent/_restart_planning.py`,
`agent/_restart_execute.py`, `_dismiss_persistence.py`, `agent/hijack/`, the TUI
prompt-bar stash, every `_has_staged_handoff` check, and `_staged_origin_retains_claim`.
Consume their public results read-only. If their modification appears necessary, record
a `PROPOSED FOLLOW-UP:` on the assigned phase and a coordination note on the owning
hijack phase rather than editing them.

Open `sase-core` with `/sase_repo` and follow its `AGENTS.md`. Shared eligibility, state
transitions, and ownership rules live in Rust; Python gathers facts and performs I/O.
Advance `sase-core-revision.txt` through host completion or `just ratchet-core-revision`
only after the release gate closes. Never invent a SHA. Newly consumed bindings must be
included by the pin. Do not globally reinstall SASE.

## Safety and execution contract

The recovery identity is `(machine identity, current boot id, project key, epic id)`.
Display project names through the existing registry; durable keys are not user-facing
labels. A missing or unreadable boot identity means no automatic recovery. Prove
prior-boot death from the owner's recorded `process_identity = <boot_id>:<start_ticks>`
and tri-state liveness, not age or a bare PID. Never infer that a remote host's boot id
belongs to this machine.

The eligible set is open work in in-progress epics with a current assignment to provably
dead local prior-boot waiters/land agents. Include land-only recovery when its own
prior-boot death is proven, but do not infer eligibility merely from all children being
closed. Terminal done markers and deliberate stop/dismissal outcomes are not reboot
evidence. Retained work, active handoffs, recovery operations, or custody require the
existing ownership checks to succeed; unreadable evidence means skip. A newer
owner/successor overrides an older dead row.

Select innermost unfinished epics; an ancestor with an open descendant epic is not
independently relaunched. Check all relevant phase/land/session members, monitor
members, launch procs, and replacement owners. Any live or unknown owner that the epic
launch would replace blocks automatic recovery of that epic. Never release, launch, or
publish a recovery notice for hijack-custody targets. Read custody through existing
public helpers such as `agent/hijack/custody.py:matching_active_record`; unavailable
custody information also blocks action.

Use the watchdog's 900-second post-boot service grace period. An early tick is a
deferral, not a spent attempt. A same-boot scheduler restart performs no second launch.
Capacity/hold deferrals remain pending and may be reconsidered without consuming the
attempt. A cheap boot/completion fast path must not permanently hide pending deferrals
or a recovery worker that still needs settlement.

At most one launch attempt per recovery identity may cross the effect boundary. Persist
the claim and `launching` transition before invoking the launcher. If the worker dies
after that point, reconcile a recorded launch outcome when possible; otherwise record an
indeterminate failure and surface it once for manual recovery. Do not retry a
potentially completed launch just because the result write is missing. A failure before
launch with proof of no side effects may defer. This provides once-only successful
recovery in the normal reboot case and avoids claiming guaranteed exactly-once effects
after an ambiguous process death.

## Phase core_policy

Add a focused `crates/sase_core/src/boot_recovery/` domain and bindings in
`crates/sase_core_py/src/boot_recovery/`. Keep `mod.rs` files as facades and register
every binding. Add thin Python wire/adapter modules in `src/sase/core/` and update
binding/schema validation as required (read `tools` memory before changing tools).

Define versioned input facts for machine/current and previously observed boot, uptime,
project/epic hierarchy and status, current assignments and member generations, row boot
identities, tri-state liveness, done/stop outcomes, custody, retention/handoff state,
admission state, and durable attempt markers. Return typed per-epic `recover`,
`skip(reason)`, or `already_handled` decisions, with explicit retryable deferrals and
selected owner generations. Avoid using the existing stranded report's boolean
`process_live` as proof of death: it loses the required unknown state.

Define the attempt-state reducer and a shared owner-replacement decision used by the
next phase. Model admitted, deferred, launching, launched, failed, and indeterminate
outcomes explicitly. Reuse existing auto-restart ledger conventions where appropriate,
without broadening the update-skew classifier or its ledger.

On first activation, no previous sweep marker is available. Authorize only from explicit
prior-boot row evidence; missing baseline alone is neither recovery evidence nor a
reason to suppress all recoverable rows. Never make an old row eligible when a newer
local or remote owner exists.

Test policy matrices: same/different/unknown boot, PID reuse, missing/corrupt metadata,
remote owners, old terminal rows, mixed live/dead/unknown members, nested epics, active
custody, handoffs/retention, closed phases, land-only prior boot death, startup grace,
deferred capacity, repeated attempts, and invalid ledger transitions. Include PyO3 round
trips and Python wire validation. Leave production launch paths untouched in this phase.

## Phase owner_claims

Build reusable Python evidence collection under `agent/boot_recovery/`, using
`axe/wait_marker_scan.py:waiting_runner_liveness`, process identity helpers,
project-scoped artifact/session lookup, and read-only custody APIs. Start from bead
assignments, not only the artifact index's active tier: the incident rows are dead, may
lack done markers, and can have disappeared from an active index. Use targeted history
lookup with a safe fallback; unavailable history is unknown.

The current failure path is `axe/run_agent_runner_bead.py:claim_bead_for_agent_launch`
calling `bead/force_reuse.py:issue_is_in_progress_for_another_agent`. The latter only
compares owner names. The Rust `bead/mutation/claims.rs` launch mutation currently
assigns any non-closed bead, relying on that Python precheck; its `release_agent_claim`
only releases `claimed`, not `in_progress`. Do not remove the Python refusal and thereby
allow arbitrary takeovers, or loosen generic release.

Add a dedicated Rust-guarded recovery claim operation (or an equivalently explicit
guarded extension to launch context) with expected status, owner, and bead revision.
Feed the shared Rust owner policy with freshly collected evidence. Revalidate the
artifact generation and custody immediately before applying the mutation under the
existing store-lock span. A changed owner/revision refuses without writes. A prior-boot
process cannot become live again, but its assignment can change, so both proofs are
necessary.

For a replacement, atomically transfer the claim to the accepted new owner rather than
creating an unowned intermediate `open` state. Cover both waiting reservation and launch
promotion, including a differently named Continue fork. Preserve the ordinary
same-owner/force-reuse and closed-bead-launch contracts. Write through the managed
machine bead store and its existing commit/publish path; do not edit projection files or
write to a primary sidecar clone. Persist recovery provenance without erasing earlier
work or custody records.

Regressions must reproduce the `.f0` different-name rejection, then prove safe recovery
for both `claimed` and `in_progress`. Assert refusal for current-boot death,
live/unknown/remote ownership, owner changes between scan and claim, hijack custody,
newer successors, and publication/lock failures. Include Rust mutation replay/event
parity and Python runner integration tests.

## Phase recovery_worker

Implement the host recovery worker and durable per-identity ledger under
`agent/boot_recovery/`. Python owns file locks, atomic writes, reads, and launch
effects; Rust owns transition decisions. Use a bounded machine-local state directory
under SASE home and a versioned ledger record containing candidate owners, reason,
outcome, and durable launch identifiers.

Hold an identity-scoped lock across decision persistence and effect admission. Before
the launch boundary, refresh the epic, owners, custody, admission, and retention facts.
Run the structured dry-run equivalent of `sase bead work -n` via
`bead/cli_work_handler_launch.py:launch_epic_bead_work`. A failed preview or one
requiring killing, wiping artifacts, discarding work, or other destructive cleanup must
refuse automatic launch and preserve evidence. Narrow stale claim/name reconciliation is
allowed only for the exact prior-boot-dead owners proven by the policy. Never blindly
accept every cleanup target with `yes_to_all`.

Execute only that revalidated safe selection through the same epic launch path,
non-interactively with `yes`/`yes_to_all` as required. Use the existing operational
workspace lease and machine bead-operation context. Preserve phase dependency waits,
models, queue capacity, closed phases, and the epic's land agent. Never launch via
`sase agent restart`, use a primary checkout, or create a parallel raw subprocess path
that bypasses admission. If the existing launch API cannot pin the safe selection
through execution, add a narrow validation callback or public wrapper within the allowed
bead-launch code.

Publish one inbox notice per recovery identity through
`notifications/store.py:upsert_notification`, using a stable dedup key. Update that
notice with actual launched members or the refusal/failure and manual resume command. A
deferred admission does not falsely report success. Custody targets get no recovery
notice. Make notification retry independent of launch retry.

Test two workers racing, failure before/after the launching marker, launch success
followed by a failed outcome write, repeated notification delivery, lock/write failure,
preview refusal, changed selection, and an active manual epic launch. The
launch-then-crash case must never spawn twice.

## Phase scheduler

Add `scripts/sase_chop_boot_recovery.py` and register its builtin job and script entry
points using the existing `agent_auto_restart` pattern in `pyproject.toml` and
`src/sase/default_config.yml`. Put it with the existing wait/claim recovery jobs. Use a
60-second backstop: first tick runs on AXE start, and later ticks revisit deferrals
after the 900-second grace. Keep policy constants and reason codes in Rust, not
duplicated scheduler conditionals.

Follow `agent/auto_restart/sweep.py` for submitting work as a durable
`ProcSubmitRequest`; the scheduler never performs recovery inline. Provide an internal
worker module entry point, avoiding an unnecessary new public CLI. Use a boot-recovery
concurrency key and durable proc/operation provenance, including boot identity. Use the
existing capacity and hold-admission machinery; drain eligible epics serially and do not
bypass launch limits after reboot.

Track pending, admitted, failed, and settled work separately from the sweep boot marker.
A scheduler restart while a proc is active does not enqueue another worker; a dead
worker is reconciled from its ledger. Missing/corrupt state is visible and produces no
unsafe launch. Keep completed same-boot ticks cheap and emit useful counts and reason
codes in the job summary.

Integration tests must cover scheduler stop/start twice in the same boot, two scheduler
instances, boot A to B with missing done markers, capacity unavailable then available,
active launch conflict, first-install historical rows, early uptime then maturity, and
no usable boot identity. Assert durable submissions and actual launch calls, not just
policy return values. Run registration/config tests so both job names and entry points
resolve without a global install.

## Phase acceptance_docs

Build one end-to-end isolated fixture with the incident's 16 land and 11 phase waiters,
grouped by their actual epic ownership. Simulate boot A to B with dead owner identities
and missing `done.json`. Drive the scheduler, durable worker, real policy bindings,
claim mutation, and launch boundary; stub provider launch effects so no real agents run.
Assert one launch per eligible innermost epic, correct phase/land assignments, and no
stale-claim rejection. Repeat scheduler startup and worker delivery: zero extra launches
or notices.

Add live, unknown, remote, current-boot-dead, hijack-held, retained-work,
preview-destructive, nested, and completed historical controls. Assert their
claims/artifacts stay unchanged. Test the only-land-left prior-boot case without
activating the sibling zombie invariant. Verify the job's pending-capacity path
eventually recovers within the same boot, while ambiguous launch outcomes remain spent
and visible.

Read `docs_writing` and `docs/contribute/writing-docs.md`. Update
`docs/reference/automation/scheduler-default-routines.md` and
`docs/reference/agent-auto-restart.md` within their tier budgets. Describe evidence
requirements, automatic relaunch, the startup grace, once-per-boot semantics, safe
refusal/unknown cases, notices, and manual recovery after an ambiguous attempt.
Regenerate configuration reference material with `just fix`.

Document this as the replacement for per-epic cron watchdogs. The previous `sase-1r2`
watchdog is described in `plan:202611/v0_19_0_green_full_ci_release.md`; its bead notes
record the installed marker `sase-epic-watchdog:sase-1r2` and removal command. Inspect
`crontab -l` read-only after the implementation is verified. If still installed, use
`/sase_gate` to propose only that watchdog's removal, with backup and preservation of
every unrelated cron entry. Do not edit cron directly or read its unmanaged script as a
substitute for the recorded design. A pending human removal gate does not prevent code
completion; record it for the lander.

## Verification and completion

Each phase reads `lint_and_test` before finishing and runs focused regressions plus
`sase tool run check` in every changed code repository. In sase-core use its wrapped
`just` recipes, never bare cargo; include the PyO3 binding tests. Use `/sase_monitor`
for long verification commands, with prepared completion when appropriate. Do not run
`just check-full`. Do not weaken assertions, increase timeouts, add suppressions, or add
flake baselines.

Record independent clean-base reproduction for any unrelated failure, along with the
exact node and existing tracking bead if known, in a `PROPOSED FOLLOW-UP:` note. Such a
failure does not keep a finished phase open. Phase workers create no task beads and
close only their own assigned phase after checking `sase bead epic-symbols <phase-id>`.
Resolve every remaining symbol or re-key its Justfile entry to a still-open owning
phase/epic before closing.

The child epic lander verifies all five phases and the current core pin, confirms the
acceptance evidence, and records
`TASK RESOLVED: sase-1qj — <verified reboot and safety evidence>` on `sase-1ra.11`. Run
`sase bead epic-symbols sase-1ra.11` and resolve/re-key any leftovers. Complete this
child epic through its normal land flow; finish the explicitly delegated phase
`sase-1ra.11` with `sase bead close sase-1ra.11 --note "<what was verified>"` if it
remains open after child completion. Child phase workers must not perform that ancestor
close. Never close `sase-1ra` or task `sase-1qj`; those belong to the parent epic
lander.
