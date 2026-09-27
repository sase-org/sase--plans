---
tier: epic
title: Finish the remaining sase-1aq live acceptance
goal: The original remote-dispatch, owner-to-viewer parity, and dispatch-memory beads
  meet their approved live gates and close normally.
parent_bead: sase-1aq.10
phases:
- id: exact_ops
  title: Repair exact remote operations on fleet-dispatched agents
  depends_on: []
  size: medium
  description: 'exact_ops: make settled dispatch rows addressable and prove exact
    stop and retry on a released cohort.'
- id: viewer_matrix
  title: Finish the viewer acceptance matrix and dispatch landing
  depends_on:
  - exact_ops
  size: medium
  description: 'viewer_matrix: finish the original live viewer and fault cases, then
    land the remote-dispatch bead chain.'
- id: parity
  title: Capture and land same-build owner-to-viewer parity
  depends_on:
  - viewer_matrix
  size: medium
  description: 'parity: prove production owner and remote row equality on matched
    deployed builds and land the parity beads.'
- id: dispatch_memory
  title: Publish dispatch guidance and finish the memory backlog
  depends_on:
  - viewer_matrix
  - parity
  size: medium
  description: 'dispatch_memory: publish the authorized dispatch note, close its task
    and original memory ancestors, and audit the backlog.'
proposed_by: bbugyi200.apollo.sase-1aq.10.land
create_time: 2026-09-26 20:01:45
status: wip
bead_id: sase-1aq.10.7
---

- **PROMPT:** [prompts/202609/1aq_remaining_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/1aq_remaining_acceptance.md)
- **PARENT:** [202609/finish_1aq_live_closeout.md](https://github.com/sase-org/sase--plans/blob/main/202609/finish_1aq_live_closeout.md)
- **BEAD:** [sase-1aq.10.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1aq/sase-1aq.10.7.md)

# Finish the remaining sase-1aq live acceptance

## Boundary and accepted evidence

This is the remainder of the approved sase-1aq.10 outcome. Its parent_bead returns
control to that epic's land agent. Do not repeat the accepted uncertain-operation
reconciliation, schema-v6 store reset, snapshot/dismissal proof, or hold-memory
publication. Phase sase-1aq.10.1 reconciled operation 19044f5b44a9 to **no agent
created** and backed up then removed the stale schema-1 fleet launch store. Phase .10.2
repaired the service gateway's double-appended bridge command in commit 29f8240df5 and
proved one live Athena-to-Apollo dispatch: receipt, owner RUNNING/DONE, nonempty
reply/artifacts, source preflight, same-key duplicate refusal, and gateway restart.
Phase .10.3 normally closed the original snapshot bead sase-xe.16.11.7.14.6.7.5. Phase
.10.4 established a green production facts oracle but lacked matching installed
owner/viewer captures. Phase .10.5 published hold/proc-queue guidance and the
pull/fail-open decision. Memory task sase-134 is now closed after checking those notes
against docs/xprompt.md and docs/cli.md and a passing sase memory init --check.

At planning time sase-1aq.5-.9, sase-xe.16.10, sase-xe.16.11, sase-133.5, sase-133,
sase-1ae.4/.5, sase-1ae and sase-ya remain non-closed. Re-read them before acting.
Athena's served catalog omitted the fleet-dispatched rows, so stop by full ID and retry
by name could not find them. Viewer fault cases and same-build live parity remain
unproved. No original bead may close on a partial receipt, fixture, or test-only proof.

Read plan:202609/finish_1aq_live_closeout.md,
plan:202609/finish_blocking_epics_and_memory.md,
plan:202609/launch_recovery_and_xe_closeout.md,
plan:202609/remote_parity_landing_repairs.md, and
plan:202609/close_memory_bead_backlog.md through sase artifact read; read original beads
and notes through sase bead read. The requested
research:202609/stalled_epics_sase-1ah8_1aq_1ae.md currently resolves as missing; use
surviving bead/plan evidence and recheck if it becomes available. Read every sidecar
artifact through sase artifact read and open linked repositories through /sase_repo.
Shared fleet/dispatch behavior belongs in sase-core, with binding/adapters and a
post-binding core pin in sase. Use just check for changed trees. Do not run just
check-full without a new explicit instruction.

Before resuming original workers, inspect bead, agent, gate, monitor and operation state
on Athena and Apollo. Preserve a healthy owner and checkpoint; recover only a confirmed
abandoned assignment. Use supported published updates on both hosts, record exact
installed SASE/core versions and gateway/AXE/ACE identities, preserve enrollment and
unrelated work, and never replay an uncertain operation under a new key. Retain redacted
live panes, payloads and PNGs as audited artifacts with UTC timestamps.

## Phase exact_ops

Trace Athena's catalog_sync lookup and the settled-receipt versus landed-agent project
identity for dispatch-a464978f and dispatch-5330c1fc. Fix the owner/catalog identity or
projection contract so a fleet-dispatched row stays addressable by exact instance across
receipt settlement, catalog refresh and restart. Preserve cross-project isolation and
refusal of stale/reused locators. On one controlled live Athena-to-Apollo agent, prove
exact stop on Apollo and same-key retry without second execution. Add focused core,
binding and Python regressions for the cause; move the core pin if a binding changes.
Verify the installed gateway uses the bare sase bridge executable. Record
requirement-to-evidence rows on sase-xe.16.11.7.14.6.7.6 and sase-1aq.5; let the proper
owner close the exact-stop phase only after its full gate passes. This resolves .10.2
note #2 and .10.3 note #1.

## Phase viewer_matrix

Finish the original .7.6 matrix on matching published builds: unfollowed-remote
attention with another tab/filter active; stale-gate refusal; composable project/machine
queries; target-picker focus and source preflight; a useful healthy host beside a
genuinely hung host; and gateway restart. Use controlled agents and harmless gates. Use
sleep-free prompts for exact-output assertions because earlier codex prompts completed
through monitor continuation without printing requested sleep markers. Recheck the
post-start Prompts overlay entry-point change ade28c173a against dispatch target focus
and preflight; its history route changed while the target picker remains in the prompt
bar. Integrate a regression if it loses focus or changes target selection.

For sase-xe.16.11.3 capture a real old locator and replace it before exact-instance
rejection; do not use a fabricated locator. Recheck its genuine healthy-beside-hung-host
case. For .16.11.5, .16.10 and reopened .7.13, record one requirement-to-evidence matrix
covering receipt, catalog, follow, output, stop, restart, explicit %id, picker,
freshness and bridge persistence. Reuse accepted .14.6.6 and .7.5 evidence without
reopening only for attribution. Fix failures, rerun focused tests and just check,
inspect visual goldens, and attach audited live evidence.

Let original land owners close the remote chain in dependency order: .7.6, .7.13,
.16.11.3, .16.11.5, .16.11.7, .16.11, .16.10, .16, root sase-xe, and sase-1aq.5/.6 as
their criteria allow. Include intermediate .14.6.7, .14.6 and .14 ancestors in the
audit. At each ancestor check notes, drift, epic symbols, verification and linked plan
status. Never force or proxy-close. This resolves .10.2 note #3, .10.3 note #2, and
their repeat in .10.6 note #4.

## Phase parity

On matched published builds, rerun the production oracle through the real owner loader,
gateway serialization, federation and viewer renderer. Compare visible identities,
family membership, active/waiting/completed status, chips, runtime, nested-shell counts,
and current/compact-index paths. Resolve the workflow-step ×N/shell-chip discrepancy
against the promised node parity contract. Keep false fleet-version skew absent and
genuine version diagnostics intact. The clean-base AgentType.PROC_SHELL test failure
belongs to active rename epic sase-1ab, where this land audit recorded it; do not weaken
the oracle.

Capture the settled Apollo owner Agents pane and Athena machine:apollo pane close in
time at identical geometry through the ordered sase screenshot flow. Inspect both PNGs,
compare identities and representative families, and register audited artifact refs.
Exercise local and remote screenshot driving without resolving unrelated gates. Fix
mismatches and repeat. Let original owners normally close sase-133.5.4, sase-133.5,
sase-133 and sase-1aq.7 with current checks and linked plans. This resolves .10.4 note
#1.

## Phase dispatch_memory

After sase-xe.16 and sase-133 land, use existing sase-ya authorization and
/sase_memory_write to add a short type: reference sase/memory/dispatch.md pointer to
docs/remote_dispatch.md and one %dispatch row in sase/memory/xprompts.md. Verify
selector, clean published source, same-key recovery, enrollment/repair/quarantine,
picker/Machines no-network-probe behavior and session terminology. Include Focus/Fleet
count claims only if landed parity supports them. Preserve hold guidance and queue
multiplier wording. Run sase memory init --no-commit, sase memory init --check, audited
memory reads and just check; inspect generated changes. Close sase-ya with actual scope,
then sase-1ae.4 and sase-1aq.8 normally.

Complete sase-1ae.5 against original memory-task outcomes and an unbounded non-closed
memory-task census, confirm sase-134 remains closed, and land sase-1ae through its
owner. Re-read all promised descendants and notes, linked plan statuses and current
verification. Close sase-1aq.9 only after its own audit passes. This resolves .10.5 note
#2 and .10.6 notes #2-4. The clean-base rename/proc check failures in .10.2 note #5,
.10.3 note #3, .10.4 note #2, .10.5 note #3 and .10.6 note #5 are recorded on active
causal epic sase-1ab; do not create a duplicate task or treat a red just check-full as
unfinished dispatch work. The child epic land agent rechecks every proposal and
post-child drift, then returns to sase-1aq.10 through parent_bead for its landing
decision.
