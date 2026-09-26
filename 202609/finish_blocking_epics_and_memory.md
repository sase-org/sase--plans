---
tier: epic
title: Finish the hold and remote-dispatch blockers of the memory backlog
goal:
  Land the existing hold, remote-dispatch, and Agents-parity epics with verified
  acceptance, publish the two deferred memory updates, and close their descendants and
  the sase-1ae backlog.
phases:
  - id: reconcile
    title: Reconcile live ownership and the closeout ledger
    depends_on: []
    size: small
    description:
      "reconcile: inventory non-closed descendants, owners, evidence, and exact next
      actions."
  - id: hold_landing
    title: Verify and land the existing hold epics
    depends_on:
      - reconcile
    size: medium
    description:
      "hold_landing: verify current hold contracts and normally close both existing hold
      epics."
  - id: dispatch_runtime
    title: Finish released runtime adoption for remote dispatch
    depends_on:
      - reconcile
    size: medium
    description:
      "dispatch_runtime: publish and install matching repaired SASE and core builds on
      both machines."
  - id: dispatch_snapshot
    title: Complete the live Apollo snapshot and dismissal proof
    depends_on:
      - dispatch_runtime
    size: medium
    description:
      "dispatch_snapshot: prove owner-viewer fleet state, dismissal, history, and
      restart on live Apollo."
  - id: dispatch_unified
    title: Complete unified Agents and exact remote-operation acceptance
    depends_on:
      - dispatch_snapshot
    size: medium
    description:
      "dispatch_unified: finish the cross-machine live matrix and original fault proofs."
  - id: dispatch_landing
    title: Land the remote-dispatch ancestors through sase-xe
    depends_on:
      - dispatch_unified
    size: medium
    description:
      "dispatch_landing: close reopened original phases and all remote-dispatch ancestor
      epics."
  - id: parity_landing
    title: Prove and land owner-to-viewer Agents parity
    depends_on:
      - dispatch_landing
    size: medium
    description:
      "parity_landing: establish production and same-build live parity, then close both
      parity epics."
  - id: deferred_memory
    title: Publish landed hold and dispatch guidance
    depends_on:
      - hold_landing
      - dispatch_landing
      - parity_landing
    size: medium
    description:
      "deferred_memory: publish the two authorized reference updates and close their
      tasks and post-landing phase."
  - id: final_audit
    title: Close the memory backlog and verify every requested bead
    depends_on:
      - deferred_memory
    size: small
    description:
      "final_audit: close the remaining memory phase and parent, then verify all target
      trees are done."
proposed_by: bbugyi200.athena.0sw
create_time: 2026-09-26 11:53:22
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_blocking_epics_and_memory.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_blocking_epics_and_memory.md)

# Finish the epics blocking `sase-1ae.4`

## Outcome, boundary, and sources

`sase-1ae.4` must wait for `sase-11l`, `sase-11l.11`, `sase-xe.16`, and `sase-133` to
close. The hold repairs' implementation children are closed, but their landing remains
open. The dispatch branch still has live acceptance and ancestor landing work, and the
parity branch still needs production and live evidence. The only non-closed `memory`
tasks in the live census on 2026-09-26 were `sase-134` and `sase-ya`; they cannot be
closed on pre-landing assumptions. This plan finishes the already approved work and its
ordinary landing chain, then completes `sase-1ae.4`, `sase-1ae.5`, and `sase-1ae`.

Read the live beads and their notes before acting, especially `sase-11l.11`,
`sase-xe.16.11.7.14.6.7`, `sase-xe.16.11.7.13`, `sase-xe.16.11.3`, `sase-xe.16.11.5`,
`sase-xe.16.10`, `sase-133.5.4`, `sase-134`, `sase-ya`, and `sase-1ae.4`. The approved
source plans are `plan:202609/hold_landing_repairs.md`,
`plan:202609/launch_recovery_and_xe_closeout.md`,
`plan:202609/remote_parity_landing_repairs.md`, and
`plan:202609/close_memory_bead_backlog.md`; read them with `sase artifact read`. The
latter derives from
`research:202609/memory_bead_backlog_audit/memory_bead_backlog_audit.md`. Older notes
and numerical live counts are evidence, not proof of today's state.

This is a coordination and acceptance plan over _existing_ epic beads, not a new
implementation architecture. Its nine phases are bounded direct work. **No phase may
create another epic, child plan, or broad follow-up to move one of these acceptance
gates elsewhere.** Fix a defect that is necessary to satisfy the assigned gate within
that gate, reverify affected evidence, and keep the phase open until the result is
proven. If a new requirement is genuinely outside this outcome, record it through the
applicable bead policy without treating it as a substitute for any gate here. A landed
commit, an agent launch receipt, a healthy hello, or an automatically closed bead is not
completion evidence on its own.

Use `sase bead read` for audited bead context and `sase artifact read` for every cited
sidecar artifact. Open any linked repository with `/sase_repo` first; shared backend
policy belongs in `sase-core`, Python remains a thin adapter or presentation layer.
Before memory edits use `/sase_memory_write`; the user's request and the descriptions of
`sase-134`/`sase-ya` authorize those specified edits. Follow `sase_beads.md`,
`sase_artifacts.md`, `lint_and_test.md`, and relevant TUI/flag/xprompt/tailnet memory
through `/sase_memory_read`. Use host finalizers for commits, never manual commits.
Follow current `lint_and_test.md`: run `just check` for changed files; run
`just check-full` only where an explicit assigned requirement or a CI-only failure calls
for it, and then through `/sase_monitor`. Inspect every changed visual golden and any
`partial` update manifest. Use supported monitors/continuations for releases, checks,
and live work that outlasts a turn.

The existing phase and land agents are the preferred owners. Before launching or
recovering work, inspect current bead, agent, gate, and monitor state. Reuse a healthy
owner; recover only a confirmed abandoned assignment with the supported workflow and a
preserved prompt/checkpoint. Do not race two agents on the same bead, replay an
uncertain operation with a new key, or assume a handoff resumes automatically. Close a
bead normally with a reason and evidence only after its own requirements and all
descendants are complete. Never use `--force`, a canceled resolution, a plan-status
edit, or an automatic commit close to conceal an unmet acceptance item. An epic's land
agent owns its normal parent close and linked-plan `status: done` update; this plan's
phases verify that those events occurred and recover an abandoned lander when needed.

## Phase `reconcile`: one current ledger and one owner per bead

Re-read the full active descendant trees of `sase-11l`, `sase-xe`, `sase-133`, and
`sase-1ae`, plus `sase-134` and `sase-ya`. Record the status and current owner of every
non-closed descendant, the specific open acceptance item, its source plan, and the next
supported continuation. Reconcile the latest dispatch closeout plan with live state: its
formerly open `sase-z6` flag bead and `sase-xe.16.11.7.14.6.6` are already closed in the
2026-09-26 store; do not redo them merely because the older plan lists them. Check
whether any other item closed or changed during planning. Confirm existing dependency
edges on `sase-1ae.4` still cover all four blockers, and preserve them.

Landing criterion: a dated bead-and-owner ledger names every currently non-closed target
and its exact remaining gate; no duplicate active worker or orphaned handoff is being
mistaken for completed work. Attach concise evidence to this phase and pass the ledger
to later phases.

## Phase `hold_landing`: finish `sase-11l.11` then `sase-11l`

All implementation phases under `sase-11l.11` currently show closed. Its September 19
full-gate failure named a Services-tab test, an import-budget flake, a bootstrap flake,
and leak-detector behavior; those observations are historical and already have recorded
owners or fixes. Review the parent audit and the child landing notes, verify the current
released core pin and package floor, and exercise the real hold admission, CLI/directive
selector parity, proc queue, capture/expiry notification, and branched/hood deadlock
contracts on the current combined tree. Recheck the prior gate under current testing
policy and triage any actual current failure by evidence; an unrelated historical
failure is not a reason to force-close, and an old red run is not proof the present tree
is red.

The existing land owner closes `sase-11l.11` when its linked plan and all descendants
meet their gates, then audits and normally closes `sase-11l`. Its close note must
preserve the original nine follow-up dispositions, actual verification results, release
identity, and any remaining external task ownership. Confirm post-close Symvision and
linked plan statuses through the normal landing flow.

Landing criterion: both epic beads and every descendant are closed with evidence-based
`done` outcomes; the final hold behavior is on supported released core, relevant current
checks pass, and the original and repair plans are marked done. If a gate fails, repair
and recheck it in this branch before closing either epic.

## Phase `dispatch_runtime`: finish `sase-xe.16.11.7.14.6.7.4`

Use the current owner of `sase-xe.16.11.7.14.6.7.4`. Verify that the approved
launch-targeting, requester-continuation, operation-context isolation, real TLS/deadline
and exact-instance repairs are actually landed and available in a published core wheel.
Verify the SASE pin, package floor, lockfiles, generated skill deployment, packaged
gateway/worker commands, and clean installed behavior without an editable core override.
Update Athena and Apollo through supported managed flows; compare installed revisions
with the running gateway, worker, AXE, and ACE identities and confirm authenticated
status with no false version skew. Preserve live enrollment, credentials, and unrelated
active work. The already closed `sase-z6` and prior fleet-acceptance phase are
verification inputs, not work to repeat.

Landing criterion: `.7.4` is normally closed with exact source SHA, core wheel/version,
running binary identities, skill deployment and operational health evidence. No dev-only
build or promised future release is counted as adoption. This phase must finish before
either live proof uses the repaired cohort.

## Phase `dispatch_snapshot`: finish `sase-xe.16.11.7.14.6.7.5`

After the matching runtime is installed, resume the current `.7.5` owner and complete
the interrupted Apollo stale-row proof from
`plan:202609/launch_recovery_and_xe_closeout.md`. Compare the real Apollo owner-visible
set to Athena's remote set and authoritative counts, including recent completed
families, waiting/unknown status, observation age and a disconnected host. Use
controlled short-lived workloads only, confirm target identity first, and retain
redacted pane/payload evidence. Exercise production dismissal of a controlled terminal
agent, protected live/unknown/waiting behavior, death reconciliation, older-history
paging, a snapshot change during continuation, and gateway restart without stale-row
resurrection. Record what comes from live panes versus isolated regression fixtures.
Preserve the already closed `.14.6.6` result and cite any new proof on it without
reopening merely to change attribution.

Landing criterion: `.7.5` closes only after the old and new owner/viewer identities,
counts, dismissal, history, snapshot, and restart assertions have traceable evidence on
the released cohort. A catalog projection without the required pane proof, or a cached
healthy label hiding a failed network read, is insufficient.

## Phase `dispatch_unified`: finish `sase-xe.16.11.7.14.6.7.6`

Resume the current `.7.6` owner and satisfy the unified live matrix in its approved
plan. In one bounded Athena-to-Apollo cohort, verify a dispatched agent's receipt and
fresh visibility, nonempty output, exact-instance stop confirmed on Apollo, same-key
reconciliation after an uncertain reply and ACE restart, attention from an unfollowed
remote agent while another tab/filter is active, stale-gate refusal, composable
project/machine queries, target-picker focus and source preflight, healthy-host
navigation beside a hung host, and recovery after gateway restart. Reuse accepted
fixture coverage for destructive or difficult faults, but do not label it live evidence.
Check the genuine captured-old-locator replacement and real healthy-host/deadline
regression required by reopened `sase-xe.16.11.3`; do not rely on fabricated locators or
fast-failing peers. Confirm current visual and fault-budget evidence and changed
goldens, and run the required current core/SASE checks under the testing policy above.

Create one requirement-to-evidence matrix keyed to reopened `sase-xe.16.11.7.13`,
`sase-xe.16.11.3`, `sase-xe.16.11.5`, and `sase-xe.16.10`, plus the original `.14.6.6`
proof. Address each incident-shaped `PROPOSED FOLLOW-UP` within an existing causal epic
or record the established owner; no acceptance-blocking defect becomes a new vague
follow-up. Cite exact versions, timestamps, artifacts and check results on the relevant
original beads.

Landing criterion: `.7.6` closes with the complete evidence matrix and no unmet item in
the approved live matrix. A launched agent, a healthy hello, or an auto-close event
cannot replace receipt, visibility, management and restart proof.

## Phase `dispatch_landing`: close the existing remote epic chain

Use the serialized land sequence in `plan:202609/launch_recovery_and_xe_closeout.md`,
adjusted to the live store. After `.7.6`, the proper land owners verify and normally
close `sase-xe.16.11.7.14.6.7`, then `.14.6`, `.14`, and the unified branch's reopened
`.7.13`. Verify the original `.16.11.3` real fault proof and `.16.11.5` live workflow
before their normal close. Then land `.7`, `.16.11`, finish original `.16.10` end-to-end
proof, and land `.16` and root `sase-xe`. Review all descendants, original notes and
prior follow-up dispositions, linked plans, and post-child drift at each level. Check
current epic-symbol and post-close Symvision results and set each linked plan status
done through its normal land owner. If an older waiting lander has genuinely been
abandoned, recover only that assignment with an explicit continuation and owner record;
don't blanket launch the root tree.

Landing criterion: a fresh `sase bead read`/descendant census finds `sase-xe`,
`sase-xe.16` and every descendant closed with legitimate resolution and no waiting
acceptance owner; the related plans are done, required current checks passed, and a
reviewed artifact carries released builds and live Apollo evidence. This phase may not
finish at an intermediate child close.

## Phase `parity_landing`: complete `sase-133.5.4`, `.5`, and `sase-133`

Use the now landed, matching release on both machines. Recheck the production fixture
oracle from `plan:202609/remote_parity_landing_repairs.md` through the actual owner
loader, gateway serialization, federation, and viewer renderer. Include rootless
completed plan-shell families, pending shells, stable identities, status, chips,
runtime, nested shell counts, and current plus compact-index paths. The September 20
pre-fix live PNGs failed parity; the development-binding replay that later found 84/84
identities is evidence of a fix, not final installed acceptance. Address the outstanding
workflow-step `×N`/shell-chip discrepancy with a precise shared contract and regression,
or demonstrate from current owner and viewer definitions that it is outside the promised
node parity without silently claiming equality. Separate the independently filed
unstable-golden task from required deterministic parity proof, and inspect any
regenerated goldens. Retain the truthful distinction between capability-envelope and
fleet-data versions.

On equal deployed builds, capture owner Apollo and Athena `machine:apollo` views close
together at the same geometry using the ordered screenshot input flow. Compare
normalized visible identities, grouping, counts, status, family chips and representative
active/waiting and completed families; inspect both PNGs, save audited artifacts and
record exact versions, freshness and any scope limit. Test local and remote screenshot
driving without resolving unrelated gates. The existing `.5.4` owner closes it only
after passing current checks and actual equality, then the `.5` and `sase-133` land
owners review notes, follow-up disposition, descendants, plan statuses and post-close
Symvision before normal close.

Landing criterion: `.5.4`, `.5`, `sase-133` and all their descendants are closed;
production-oracle and same-build live evidence support the original owner/viewer parity
claim; linked plans are done. A pre-fix screenshot or unequal viewport comparison is not
acceptance.

## Phase `deferred_memory`: close `sase-134`, `sase-ya`, and `sase-1ae.4`

Begin only when the four explicit external dependencies of `sase-1ae.4` are closed.
Re-read their landed contracts and current `docs/xprompt.md`, `docs/remote_dispatch.md`,
and `sase/memory/xprompts.md`; treat the September 26 research audit as a wording guide,
not as a substitute for the final source. Use `/sase_memory_write` and edit canonical
memory only, never generated shims. For `sase-134`, add a compact `%hold` row and
semantics for pre-admission holds on other queued agents and undispatched procs,
optional bounded TTL (then-current default/cap), selectors, scope, session/proc terminal
release, fail-open store behavior, running-work immunity, and the real CLI/rejection
matrix. Add the requested concise decisions strand for the accepted pull model,
TTL/fail-open policy, release scope, alternatives, costs and reopening condition.
Preserve the newer `%queue` multiplier wording and describe standalone proc
weight-0/capacity-at-dispatch accurately.

For `sase-ya`, create a short `type: reference` `sase/memory/dispatch.md` pointer to the
current runbook, plus one `%dispatch` table row in `xprompts.md`. State the actual
selector, clean published-source and operation-key recovery contract, enrollment/repair
and quarantine path, and the no-network-probe picker/Machines behavior. Use _session_
terminology. Include Focus/Fleet or unified-count assertions only if the landed current
parity evidence supports them; otherwise omit the assertion and explain the correction
in the close reason. Do not copy a whole runbook into memory.

Run `sase memory init --no-commit`, `sase memory init --check`, inspect generated diffs
and authored links, use audited `sase memory read` for the new note/strand, and run
`just check`. Close `sase-134` and `sase-ya` individually with reasons citing what was
published and verified; then close `sase-1ae.4` with evidence that all four dependency
epics are closed and both memory tasks are done.

Landing criterion: the two authorized memory changes render correctly against landed
behavior, their task beads and `sase-1ae.4` are closed normally, and no stale generated
instruction copy or unsupported remote-count claim remains.

## Phase `final_audit`: finish `sase-1ae` and this closeout epic

Execute the already assigned `sase-1ae.5` audit: verify memory rendering and links, read
the 15 original memory-task outcomes, confirm close reasons for the two newly finished
tasks, and run an unbounded census of non-closed memory tasks. If a new memory task
appeared, determine whether it is part of the requested backlog and resolve it or record
a specific blocker; never assert zero based on a limited list. Close `.5` only when its
audit passes. Let the `sase-1ae` land owner review all five phase outcomes and normally
close the parent, update its linked plan status and perform its required landing checks.

The new closeout epic's land agent then re-reads the complete target census: `sase-11l`,
`sase-11l.11`, `sase-xe`, `sase-xe.16`, `sase-133`, `sase-1ae.4`, `sase-1ae.5`,
`sase-1ae`, their descendants, `sase-134`, and `sase-ya`. Require all to be closed, all
relevant plan artifacts marked done, no abandoned acceptance/landing assignment, and
verified current check/live artifacts. Review every new phase's evidence and post-phase
drift, run the current landing gate and post-close Symvision as applicable, and close
this epic only through its normal land workflow. If any item fails, keep this phase and
the epic open with a precise repair owner and continuation; do not propose a nested epic
to call the work finished.

Landing criterion: the requested epic trees and memory backlog are demonstrably closed
with complete evidence, and the new coordination epic itself closes without a child epic
or an outstanding acceptance gate.
