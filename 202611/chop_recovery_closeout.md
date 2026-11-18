---
tier: tale
title: Finish chop recovery safety and nested landing
goal:
  Repair four verified recovery gaps and close the verified chop recovery plan chain.
size: medium
proposed_by: bbugyi200.athena.sase-1tc.3.1.6.land
proposed_with: codex/gpt-6-astra
bead: sase-1tc.3.1.6
status: done
---

- **PARENT:**
  [202611/chop_recovery_defects.md](https://github.com/sase-org/sase--plans/blob/main/202611/chop_recovery_defects.md)
- **BEAD:**
  [sase-1tc.3.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.3.1.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1tc.3.1.6.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1tc.3.1.6.land.md)
- **COMMITS:**
  - [9394c75](https://github.com/sase-org/sase--plans/commit/9394c750024cc6a0ce17635e85962ab3d3522fc4)
    — docs(plans): mark chop recovery defects plan done

# Finish chop recovery safety and land sase-1tc.3.1.6

This is the remaining work from the landing audit of **sase-1tc.3.1.6**. One coder
implements the bounded corrections below and completes the landing in the same turn.
There is no subsequent tale land agent. Do not create another phased epic for these four
corrections.

Read `sase bead read sase-1tc.3.1.6 -r "Need landing audit and follow-up outcomes"` and
its four children, then use audited artifact reads for
`plan:202611/chop_recovery_defects.md` and `plan:202611/chop_launch_recovery.md`. Their
accepted decisions and contract remain final. This plan changes no memory notes, CLI
surface, or service stop deadline.

## Audit baseline and confirmed remaining work

The review baseline is sase **5b08952c6d**, matching origin/master at initial review,
with sase-core **98a22b7906f3bdcc5a9e5c293d975a9ec2c116b5**, exactly the Python revision
pin. All four phases are closed. Their commits are `6026cc1182`, `4e5c1b9730`,
`3c6e3a06c0`, and `5b08952c6d`. The source and Rust cancellation transition were
reviewed. A serial run of recovery/process/store/replay and admission cancellation
suites passed **69 tests in 52.75s**. These passes do not cover the four reproduced gaps
below. Captured test evidence: **file:explicit:14eeac4f531a122f4b88ecf4**.

The disposable reproduction script is **file:explicit:1e5c251a1fc867b27e2230b1**; read
it with `sase artifact read` for the exact setup. It creates only isolated state and
never launches an agent. Its captured output is
**file:explicit:12427ad2e2bd1803cfb142ab**. The epic landing note links the test
evidence.

The lander reviewed all 15 non-epic commits since the first phase commit. The recovery,
proposal-launch, typed-admission and test splits preserve the feature's callers; work
must edit their current modules rather than recreate the old monoliths. The oversized
proposal launcher reported in epic note #1 was split by `d5a427a8d3`. Forced session
reuse (`05a4a68fed`) and compact-list changes (`fbc1e34fbc`, `fbfaf07123`) require
preserving conservative name-ownership checks, but add no new chop dispatch consumer.
The remaining intervening changes concern SDD, previews, update coordination, finalizer
visibility, and macro documentation. No separate integration rewrite was needed. Recheck
drift since this baseline before coding.

A later fetch during the audit found **bd8616817d** on origin/master. It finishes the
unscanned-writes finalizer guard and advances the core pin to **7ac2081b**; it does not
change chop recovery consumers. Preserve that newer pin rather than resetting it to the
audit baseline. Verification reported above was on the baseline.

1. **Unreadable evidence is treated as proof of absence.**
   `_chop_batch_recovery_observations.collect_spawn_observations` catches failures
   reading linkage, active names, blocked namespaces and registry entries, then returns
   an apparently complete observation list. `chop_agents._read_records_unlocked` also
   turns corrupt/unreadable records into an empty list. The ordinary adapter
   unconditionally passes `observations_are_complete=True`. In an isolated reproduction
   with a `dispatching` member and `get_chop_agent_records` raising `OSError`,
   observations were `[]` and Rust selected `to_launch=[0]`. This can duplicate a launch
   whose acknowledgement was lost.
2. **Generation matching is not scoped to the selected clan.**
   `_chop_batch_recovery_cancel._generation_matches` examines generations from all
   members, and may use a top-level generation for a mixed-clan batch. A batch with clan
   `a` at `20261118_120000` and clan `b` at `20261118_130000` was cancelled by
   `cancel_chop_batches_for_clan("a", generation="20261118_130000", ...)`. The selected
   generation belongs to `b`, not `a`.
3. **Dry-run recovery writes durable state.** `list_pending_chop_batches` refreshes the
   clan index and retires terminal files; recovery delivers outbox notices and emits
   disabled/maintenance holds before checking `dry_run`. The reproduction's dry run
   created two notice sidecars and notification rows, updated the clan index, and moved
   a cancelled manifest into history. The inherited contract explicitly requires
   read-only dry runs.
4. **A cancelled crossing launch can strand lifecycle forever after executor death.**
   The Rust cancellation transition correctly leaves an in-flight member `dispatching`,
   permitting a live executor's acknowledgement. But enumeration and retirement consider
   any cancelled batch terminal, and recovery skips it. With `begin_dispatch`,
   cancellation, and no surviving executor, recovery archived the manifest still
   containing `dispatching`; `pending_batch_for_run` remained non-null before and after
   recovery. Lifecycle consequently waits on an archived member that no executor can
   settle. Preserve the intentional live crossing-launch rule while completing
   dead-executor reconciliation.

## Implement and prove the corrections

Use `/sase_repo` to open `sase-core` and read its instructions before changes. Shared
generation-selection and cancelled-member settlement policy belongs in Rust
`crates/sase_core/src/axe_chop/`, with additive wire/bindings and round-trip tests.
Python owns file/process observation and effect orchestration. Keep compatibility with
existing manifests and the admission journal as the typed per-unit authority. Use the
supported pin/finalizer workflow for any new binding; do not hand-write a SHA.

- Introduce a strict recovery observation path distinguishing absent evidence from
  corrupt, unreadable or otherwise incomplete evidence. Propagate uncertain reads as a
  visible, retryable hold with zero dispatch. Ensure lower-level readers do not silently
  discard the failure first. Ordinary replay must never assert complete evidence on a
  partial scan. Apply the same distinction to typed lost-ack recovery where it consumes
  linkage. Do not infer death or absence from an I/O exception. Add regressions for read
  exceptions, corrupt linkage, unreadable name ownership, genuinely missing records,
  matched completed/dismissed launches, and recovery proceeding after the evidence
  source becomes readable.
- Match explicit cancellation against the requested `(clan, generation)` only. Members
  of other clans must neither satisfy nor veto that match. Preserve the
  no-recorded-generation protection for a demonstrably older generation and the
  deliberate rule that selecting either constituent clan cancels the whole batch. Test
  both a wrong generation that happens to belong to the other clan and a correct
  secondary-clan generation with a different first-clan generation. Include
  ordinary/typed member shapes and Rust binding coverage.
- Make read-only enumeration available without index repair, retirement, notice
  creation/delivery, counter changes or owner acquisition. Select it before any effect
  in dry-run recovery and keep query helpers read-only. Preserve normal recovery's
  bounded retention/index maintenance. Test filesystem content/path equality around dry
  runs of enabled, disabled, maintenance-held, terminal and pending-outbox cases; no
  launcher may be invoked.
- Keep cancelled batches with unsettled dispatch evidence discoverable until safe
  settlement. A live owner still wins. For a dead owner, reconcile without launching:
  durable spawn linkage/typed receipts settle a real launch and its original history
  exactly once; complete evidence of no spawn settles cancelled intent; ambiguity stays
  visibly held. Never reopen cancellation or retry a cancelled launch. Retire only after
  member evidence, history and pending notices are settled, so lifecycle can finish.
  Cover ordinary and typed lost-ack/dead-owner windows, no-spawn and proven-spawn
  outcomes, a second restart, live-owner holdoff, and unchanged once-per claim
  semantics. Use a fresh-process handshake regression for executor death during crossing
  cancellation.

Keep the existing 17 subprocess scenarios and disposable stop/resume and clan-kill
demonstration passing. Complete the approved ordinary/typed acceptance matrix rather
than treating those 17 scenarios as coverage of every row: the current typed cases are
only SIGTERM, spawn-before-ack kill, paused cancellation and first-run race; the
`k=0, 0<k<n, k=n` parameterization is ordinary-only. Reuse/parameterize the harness to
cover typed boundary k values, reservation-before-spawn, active/crossing cancellation,
multi-clan and two-interruption notices as required by the original acceptance phase.
Add the missing cases above to production-shaped fixtures: real chop-agent linkage,
project-valued `cl_name`, original names/run/index, real name registry and isolated
state. No provider calls, live scheduler restart, or live job. Use the current split
test files. Run focused Rust/binding tests and Python recovery, launch/admission,
lifecycle/once-per, inflight and clan-cleanup regressions. Run `sase tool run check`
(the wrapped `just check`) in each changed repository. Do not run `just check-full`,
weaken assertions, change flake baselines, or increase timeouts. Use `/sase_monitor` for
a long check and keep this concrete landing responsibility in its follow-up. Closing is
not contingent on this work's own SHA, push or CI result.

## Final step: finish this landing and the directly linked parent

The original lander must finish all phase follow-up triage before proposing this tale;
its outcomes are recorded on **sase-1tc.3.1.6**. Carry those exact outcomes into the
close note rather than filing the same proposals again. The four defects above are epic
work, not separate task beads.

Follow-up disposition: phase **sase-1tc.3.1.6.3 note #1** proposed 25 unrelated
broad-lane failures. All passed together serially at the audit baseline (25 tests,
16.39s; **file:explicit:daeee05e874919ec98daa999**; historical failing log
**file:explicit:4aecc722ba3df33887ba80b0**). Fourteen claim/lease failures corroborate
**sase-1oo** (including the tombstone node already merged from **sase-1op**); dismissal
and revival failures corroborate **sase-1pr** and **sase-1on**. The nine unmatched nodes
are captured in five large investigation tasks: **sase-1to** (agent-chat lookup),
**sase-1tp** (loader projection parity), **sase-1tq** (four implicit-fork dependency
cases), **sase-1tr** (two waited-producer variable-context cases), and **sase-1ts**
(named-agent lifecycle precondition). Size large reflects unproven root causes, not
scope added to this tale. The same historical run's SIGTERM notice-count failure was
fixed by the epic and now passes; no task was filed for it. No duplicate umbrella or new
cancellation-contract task was created, and active scan/global-state epics were not
assigned these flakes without causal evidence.

1. Re-read all four phase notes and the epic notes, verify these corrections and the
   complete original contract, and review all commits since `5b08952c6d` for drift. Run
   `sase bead epic-symbols sase-1tc.3.1.6`. Resolve every listed symbol by wiring,
   privatizing, a real non-test consumer pragma, or deletion; only re-key to a
   still-open later bead that actually needs the exemption.
2. Close normally:
   `sase bead close sase-1tc.3.1.6 --note "<verified contract, four repairs, tests, drift review, and all follow-up dispositions>"`.
   Run `just symvision`. Open the plans repository through `/sase_repo` and set
   `status: done` in **202611/chop_recovery_defects.md**, the epic's linked plan. A
   whitelist rejection requires cleanup and another normal close. An unfinished
   descendant requires finishing or reopening it. Never use `--force` merely to succeed
   or to advance a successful nested landing.
3. Read `sase bead read sase-1tc.3.1.6 -r "Need the parent link"`; its parent plan is
   **sase-1tc.3.1**. Resume that interrupted landing in this same closeout: read its
   previous landing notes, all five phases **sase-1tc.3.1.1** through
   **sase-1tc.3.1.5**, all other descendants and every note; use an audited read of
   **plan:202611/chop_launch_recovery.md**. Recheck descendant completion and linked
   plan readiness (`sase plan links validate --show-warnings`, then inspect findings for
   this plan chain), and post-child code drift. Preserve the existing parent follow-up
   dispositions and corrected `TASK RESOLVED: sase-1c2` evidence.
4. If that parent remains fully complete, run `sase bead epic-symbols sase-1tc.3.1` and
   resolve every entry before normal close. The audit found 15 entries in the Justfile;
   many now have production consumers, but inspect each rather than blindly removing or
   re-keying all entries. Close with
   `sase bead close sase-1tc.3.1 --note "<descendants, original defects, follow-up dispositions, readiness, drift, and verification rechecked>"`,
   run `just symvision`, and set `status: done` in **202611/chop_launch_recovery.md**.
   If the parent is incomplete or ambiguous, leave it open, append a concrete blocker
   note, and report that blocker instead of forcing closure.
5. The next parent is phase **sase-1tc.3**. Read its scope and notes, verify that the
   completed child plans fulfill it, resolve any `sase bead epic-symbols sase-1tc.3`
   entries, and close only that phase normally with the combined verification note.
   Leave containing epic **sase-1tc** and task **sase-1c2** to the already-waiting outer
   land agent. If phase scope is not complete, record the specific blocker there and
   stop the ancestor walk.
6. Finish through `/sase_final` with commit decisions for every changed repository,
   including plans and core. The final code and all eligible closes belong to this turn;
   do not wait for its own commit, push or CI. Report exactly which beads were closed
   and which, if any, remain blocked.
