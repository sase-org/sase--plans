---
tier: tale
title: Prove monitor workspace custody and finish the retention landing
goal: Preserve dirty work through unproven monitor delivery and settlement failures,
  then close the retention epics and carry-hardening phase.
size: medium
proposed_by: bbugyi200.athena.sase-1pj.7.1.5.land
proposed_with: codex/gpt-6-astra
bead: sase-1pj.7.1.5
status: done
---

- **PARENT:**
  [202611/retention_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202611/retention_landing_repairs.md)
- **BEAD:**
  [sase-1pj.7.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.7.1.5.md)

# Prove monitor workspace custody and finish the retention landing

This is the remaining work from landing `sase-1pj.7.1.5`. One coding agent can repair
the three bounded evidence/failure paths below and finish the landing. There is no
separate land agent for this tale: its final step closes the repaired epic, rechecks its
parent plan, and closes the enclosing phase when complete. Do that in the same turn as
the final code; never wait for this work's own commit, SHA, push, or CI result. The host
commits after the turn ends.

Read the latest notes on `sase-1pj.7.1.5`, its three closed phases, and `sase-1pj.7.1`
through `sase bead read`. Read these artifacts with audited commands:

- `plan:202611/retention_landing_repairs.md`
- `plan:202610/carried_workspace_retention.md`
- `plan:202610/staged_handoffs.md`

Their accepted decisions remain binding. No memory edits, new command grammar,
keybindings, flag retirement, or changes to top-level user-kill behavior belong in this
tale. Preserve stopped/lost monitors' inspection-only semantics. Reuse the existing
recovery, claim, delivery, and retention services.

## Evidence already established

The land audit inspected sase `ce87e357cb`, all child notes and the implementation
commits `4dc226bdea`, `5dfd28c74b`, and `ce87e357cb`. Gate root/member identity and the
`settled_at` producer/reader contract are implemented. The linked Rust
`workspace_retention/policy.rs` correctly chooses retention for probe uncertainty and
trusts its supplied successor facts; incorrect Python evidence can therefore still
authorize release.

After refreshing `origin/master`, no non-epic commit followed this repair epic's first
commit. Creation-time drift `ce7d60a666` only removed an unrelated stale Justfile
comment; preserve it. Refresh and inspect subsequent drift before closing. The current
`sase bead epic-symbols sase-1pj.7.1.5` scan is empty.

Seventy existing tests passed in 10.78 seconds across:

```text
tests/monitor/test_monitor_retention.py
tests/turns/test_workspace_retention.py
tests/gate_turn/test_creator_origin_retention.py
tests/test_staged_recovery_retention.py
tests/workspace_provider/test_retention_record.py
```

The following counterexamples used the existing `_init_repo`, `_make_monitor`, and
`_read_meta` helpers in `test_monitor_retention.py`, real temporary Git repositories and
RUNNING claims, and the actual Rust retention binding. Use a separate temporary
`SASE_HOME` for each case because the fixtures reuse an artifact timestamp. Convert
these probes into regression tests; do not preserve their unsafe outcomes as
expectations.

1. Create a dirty monitor with `outcome="launched"`. Set
   `monitor_followup_agent="receiver-that-does-not-exist"`. Persist a delivery using
   `new_delivery_record`, `transition_delivery`, and `persist_delivery_record`: key
   `{monitor_id: abc123def456, result_id: result, branch: failed}`, action `continue`,
   then `reserved` naming that identity, then `dispatching`. Repeat with an additional
   `acknowledged` transition naming the same nonexistent receiver.
   `_release_monitor_claim` returns `None`, deletes the real original claim, and writes
   no retention record in both cases while `tracked.txt` still contains `session owned`.
   No receiver process, artifacts, claim, or inherited obligations exist.
   `has_transfer_proof` in `monitor/settlement_fresh_owner.py` trusts delivery
   labels/names. The current `test_same_workspace_successor_does_not_hold` models this
   unsafe condition without constructing a successor.
2. Create a dirty legacy monitor without a baseline or opened-repository marker.
   Simulate a failed Git preflight by making
   `monitor.settlement._retention_git_toplevel` return `None`. `_release_monitor_claim`
   again deletes the claim with no retention record. `_retention_checkout_dir` drops the
   known workspace before dirt collection, which reports an empty repository set as
   clean. `_monitor_retention_repos` also catches discovery exceptions and silently
   continues, potentially omitting dirty linked repositories. The fresh-interpreter
   collector duplicates these weaknesses.
3. Create the dirty monitor and set `parent_timestamp="20260906115900"`, different from
   its claim timestamp `20260906120000`. Make `_build_retention_owner` raise
   `AttributeError("stale owner adapter")`; make only the subprocess invocation with
   `argv[1] == "-c"` raise `OSError("fresh interpreter unavailable")`, allowing Git
   probes through. `_release_monitor_claim` reports the error but leaves the real claim
   unpinned, with neither pending nor terminal retention evidence. The failure pin
   incorrectly uses the parent's timestamp. Fresh retry only writes pending evidence
   inside the child after imports, so an unavailable child cannot protect the original.
   Early import/payload failures return without protection too.

## 1. Require actual successor custody

Repair the evidence contract used by normal and fresh-interpreter monitor settlement. A
`dispatching`, `acknowledged`, or `settled` delivery and matching name is historical
evidence, not proof of a live receiver owning this workspace and its obligations.
Collect exact receiver/process identity, workspace claim and inherited repository
custody using the existing delivery, `resume_adoption.collect_receiver_proofs`,
successor delivery, and inheritance machinery. Check the relevant monitor/result/
branch, and reject degraded or unrelated deliveries. Keep an unchanged dirty origin held
when proof is missing, stale, mismatched, or unreadable.

Open `sase-core` through `/sase_repo` and follow its `AGENTS.md`. Shared matching and
custody classification remain Rust-owned. Reuse existing Rust policies; add only the
narrow typed evidence/decision support needed if the existing booleans cannot express
the proof safely. Python gathers observations and applies effects, with no policy
fallback. Keep the existing exact-owner mutation guards under the ProjectSpec lock. An
already-transferred claim or successor occupant must remain untouched. Do not implement
transfer by releasing a claim and then reacquiring it.

Replace the misleading same-workspace-successor fixture with an actual transferred claim
and inherited evidence. Assert the successor remains owner and its occupant survives,
rather than asserting an empty claim list. Add negative tests for reserved, dispatching,
acknowledged and settled history without a live matching successor, wrong result/branch,
dead receiver, and different-workspace custody. Confirm normal clean release and
unchanged foreign baseline dirt still release correctly. Where resume/readopt settles an
old hold, use the same proof contract so an unknown or degraded receiver cannot clear
`unresolved` merely through a success label.

## 2. Preserve failed repository observations

Unify normal/fresh repository observation around shared discovery. Keep authoritative
workspace, baseline, opened-repo, and carry identities even when a Git preflight fails.
Distinguish a positively known plain non-repository command from an unavailable owned
repository; an operational failure is uncertainty, never absence. Preserve real IDs and
the starter evidence directory. Do not swallow a failed linked-repo discovery and
classify only the clean primary.

Feed uncertainty through the existing Rust `probe_failed` contract (or its narrow
extension from step 1) so the exact owner is retained. Keep probes bounded and off TUI
render/refresh paths. Test failed preflight without baselines, clean primary plus dirty
opened linked repository when discovery fails, unavailable owned paths, repositories
beyond position eight, starter foreign dirt, and genuinely repo-less commands. Assert
the actual files, claim identity and typed retention reader output.

## 3. Protect the exact owner before retrying settlement

Make owner bootstrap, successor evidence collection, discovery, classification and
effects share the failure boundary. The monitor owner artifact timestamp comes from the
monitor artifacts passed to settlement, never `parent_timestamp`; validate every
identity field before an effect and never pass an empty wildcard as proof.

Persist a minimal exact-owner pending fact before importing potentially stale adapters
or starting the bounded fresh interpreter. Establish guarded custody before an error
return when the adapter, binding, discovery, subprocess, persistence, or effect is
unavailable. Keep this bootstrap independent of the module whose import is being
recovered. If pending persistence fails, an exact guarded pin must still protect the
claim; if identity changed, record the stale attempt without pinning or clearing another
owner. Never weaken the ownership arguments to accommodate an old effect signature.
Retry the same Rust-backed settlement, never spawn or rerun the monitored command.

Use the existing reconciliation/recovery path to retry pending settlement without
repeating follow-up delivery. Expose a failed attempt through existing diagnostics and
held-work evidence. Clear pending only after guarded settlement/recovery has completed;
preserve configured TTL once a terminal hold is established. Audit gate and
staged-adoption callers so reuse of shared protection cannot weaken their exact origin
and settled-record guards. Do not create a second retention service.

Add failure-injection tests for the counterexample above, failure during successor
collection, missing binding/import, old effect signature, subprocess timeout/failure,
pending-write failure, and retry/replay. Check that a real dirty claim stays pinned or
protected by an exact pending fact before stale cleanup, that a replacement claim and
occupant survive, that repaired settlement becomes visible, and that repeated
reconciliation neither launches a second successor nor recreates a settled hold. Read
`ace.md` and `tui.md` before any ACE edits.

## 4. Verify the repaired lifecycle

Run focused regressions and `sase tool run check` (the wrapped `just check`) in every
repository changed. Read `lint_and_test.md`. Do not run `just check-full`. If core
changes are needed, include binding round trips, verify the local development binding,
and use the established host-owned cross-repository pin workflow; never invent a SHA or
wait for this tale's own commit. The final declaration must commit every changed
repository. Use `/sase_monitor` for a long check and give its successor the remaining
closeout work below; do not end with a live command or a promise to resume. Do not use a
prepared completion that skips this tale's explicit closeout.

Recheck stopped/lost retention, degraded follow-up, gate publication failure, successful
resume/readopt, explicit dismissal and TTL through producer-to-reader tests. Preserve
the landed gate-origin fix and recovery guidance. Inspect all post-audit commits
affecting these paths and integrate them before closure.

## 5. Complete this landing and its parent closeout

This is mandatory final work for the coder, after verification and before its turn ends.
There is no later lander for this tale and no dependency on its own commit.

Follow-up triage is already recorded on `sase-1pj.7.1.5`: all three phases have zero
`PROPOSED FOLLOW-UP:` entries. The three failures above are epic work. Parent
`sase-1pj.7.1` note #2 and the repeated child close-block notes, plus note #6's
historical unlanded slot-refusal/epic-symbol repairs, are already tracked and
corroborated on ready task `sase-1qv`. Do not file a duplicate automatic-close task or
repeat a +1 without new evidence. Historical plan-status loss and flag retirement remain
with containing epic `sase-1pj`. Parent note #4 was resolved by `d2061888ec`. No
distinct unrelated proposal was found in this audit. Include these dispositions,
including the declined duplicate, in the close notes. Route any genuinely new unrelated
discovery through `/sase_new_task` before closing.

1. Re-read `sase-1pj.7.1.5` and every descendant with `sase bead read`; review every
   note and linked-plan readiness. Confirm the three regressions above are fixed and all
   original acceptance requirements still hold. Run
   `sase bead epic-symbols sase-1pj.7.1.5`. Resolve each listed entry by wiring,
   privatizing, a justified non-test pragma, or deletion, or re-key only to a still-open
   later bead that actually needs it. Then run
   `sase bead close sase-1pj.7.1.5 --note "<specific code, regression, integration, decision, and follow-up verification>"`,
   run `just symvision`, and set `status: done` in its plan
   `202611/retention_landing_repairs.md`. Use `/sase_repo` to open `plans` before
   editing and use only its returned path. Resolve/audit the plan with
   `sase artifact read plan:202611/retention_landing_repairs.md` first.
2. Re-read the parent link; it is plan bead `sase-1pj.7.1`. Recheck its previous landing
   notes #7-#9 and every newer note, all descendants and their notes (including all
   notes of `.1` through `.4` and `.5`), linked plan
   `plan:202610/carried_workspace_retention.md`, and post-child drift. Rerun descendant
   and linked-plan readiness checks; do not infer completion solely from child status.
   When complete, retire every entry from `sase bead epic-symbols sase-1pj.7.1`, close
   it normally with
   `sase bead close sase-1pj.7.1 --note "<rechecked acceptance, descendants, drift, tests and follow-up dispositions>"`,
   run `just symvision`, and set `status: done` in
   `202610/carried_workspace_retention.md` in the opened plans repository.
3. Its parent is phase `sase-1pj.7`. Read it and verify the finished retention plan
   covers its carry-hardening scope: stopped/lost monitor, cancelled question, terminal
   gate and failed adoption preserve work without a real successor. Resolve any
   `sase bead epic-symbols sase-1pj.7` entries and close only this phase normally with
   `sase bead close sase-1pj.7 --note "<verified carry-hardening scope and checks>"`.
   Leave containing epic `sase-1pj` and its plan to its waiting land agent.

If a close refuses leftover symbols, finish their cleanup and retry. If phases are
incomplete, finish or deliberately reopen them; never use `--force` to make a successful
nested landing advance. Stop at the first incomplete or ambiguous parent, record the
concrete blocker on that parent, and report it. No force, deferred-close loop, or wait
for this tale's own host commit is part of successful closeout.
