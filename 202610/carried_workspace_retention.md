---
tier: epic
title: Retain stranded handoff workspaces
goal: A terminal handoff that leaves session-owned uncommitted work without a same-workspace
  successor retains its claim and recovery evidence, shows why it is stranded, and
  supports explicit recovery or release.
parent_bead: sase-1pj.7
phases:
- id: retention-core
  title: Shared retention policy and guarded claim settlement
  depends_on: []
  size: medium
  description: 'retention-core: add the Rust retention decision and Python evidence/effect
    adapter, preserving exact claim ownership and snapshotting held work.'
- id: monitor-retention
  title: Retain stopped and unlaunchable monitor work
  depends_on:
  - retention-core
  size: medium
  description: 'monitor-retention: apply guarded settlement to stopped, lost, terminal,
    degraded, and failed-follow-up monitors in both handoff protocols.'
- id: gate-adoption-retention
  title: Retain cancelled questions and failed handoffs
  depends_on:
  - retention-core
  size: medium
  description: 'gate-adoption-retention: integrate retention into terminal gate and
    question branches and failed staged adoption, without disturbing a real successor.'
- id: recovery-surfaces
  title: Surface held work and complete its recovery lifecycle
  depends_on:
  - monitor-retention
  - gate-adoption-retention
  size: medium
  description: 'recovery-surfaces: expose stranded workspace evidence, support safe
    resume and explicit dismissal, and verify stale cleanup respects terminal holds.'
proposed_by: bbugyi200.athena.sase-1pj.7
proposed_with: codex/gpt-6-astra
create_time: 2026-10-31 08:52:37
status: wip
bead_id: sase-1pj.7.1
---

- **PROMPT:** [prompts/202610/carried_workspace_retention.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/carried_workspace_retention.md)
- **PARENT:** [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)
- **BEAD:** [sase-1pj.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.7.1.md)

# Retain stranded handoff workspaces

## Scope and existing work

This is the implementation child epic for `sase-1pj.7`, the carry-hardening phase of
`plan:202610/staged_handoffs.md`. Read that artifact and the assigned phase with audited
commands before implementation. Its accepted decisions remain final. This child epic
edits no memory notes and does not reopen any reviewer decision. The parent epic's other
phases own adoption reconciliation, flag retirement, and documentation/memory migration.

Inspection found no previous remaining-work notes on `sase-1pj.7`. The carry wire,
host-synthesized carry, staged adopters, and origin custody records have landed. Reuse
`run_agent_exec_staged_custody.py` and `commit_finalizer_inheritance.py`; do not redo
their implementations. Remaining behavior crosses shared Rust policy, several settlement
paths, claim cleanup, and recovery presentation, so it is split into four medium phases
rather than another plan requiring one broad coder.

Concrete gaps in the inspected tree:

- `turns/settlement.py` releases claims for lost/stopped turns, failed follow-up
  launches, and degraded successors. It does not inspect remaining work.
- `monitor/settlement.py` releases directly when continuation capture blocks dispatch,
  even after saving a recovery diff.
- `gate_turn/handoff_launch.py` releases a failed attempt without checking dirty work.
  `release_gate_turn_claim` protects release-mode gates and can restore a live creator;
  both behaviors must remain.
- Runner failure holds exist, but `_should_hold_workspace` can suppress them for
  hidden/auto-dismissed runs or suppressed notifications. Those display choices cannot
  prove carried work is safe to release after failed adoption.
- `running_field/_hold.py` already pins atomically but has no explicit expected-PID
  input. Some terminal release callers match only workspace/workflow/Patch.
- Held-claim cleanup uses artifacts plus `done.json` and the existing held-claim TTL.
  Explicit dismissal currently rejects any claim whose PID is alive, which needs care
  when a settled gate's temporary holder is the live scheduler.
- `bead/stranded_facts.py` and the Rust `stranded_work` reducer report beads; stranded
  workspace evidence must also be visible for runs without a bead.

All source paths below are relative to the appropriate checkout. Open `sase-core` with
`/sase_repo` and use the returned path. New shared policy belongs in `sase_core`,
exposed through `sase_core_rs`; Python gathers facts and applies effects. Never add a
Python policy fallback.

## Required behavior

The decision concerns the exact claim and checkout being relinquished, including
session-owned changes in private linked/opened repositories belonging to it.

| Current evidence                                                                                         | Settlement                                                                       |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Confirmed successor owns this same workspace and inherited its obligations                               | Preserve transfer; no stranded hold or old-owner cleanup                         |
| No successor takes this workspace; session-owned uncommitted work remains                                | Pin the existing claim and publish stranded recovery evidence                    |
| Follow-up launches in a different workspace after degraded inheritance                                   | Hold the original dirty workspace; the new agent does not discharge its custody  |
| No successor; all relevant repositories are proven clean or contain only unchanged foreign baseline dirt | Keep the existing normal release path                                            |
| Dirty/custody evidence exists but a probe or snapshot fails                                              | Retain the matching claim, record uncertainty, and do not treat failure as clean |
| Claim has moved or belongs to another owner                                                              | Do not pin, release, or clear its occupant; record the stale attempt             |
| Gate deliberately released its workspace and owns no claim                                               | Do not reacquire or mutate the workspace now owned by someone else               |

Use both custody records and current worktree/baseline facts. A historical `carried_to`
string alone does not prove an active successor or remaining dirt; equally, legacy
handoffs have no staged carry record and still need protection. Use existing finalizer
repository discovery, fingerprints, opened-repo records, and inheritance rather than
scanning sibling checkouts or treating every dirty file as session-owned. A clean
primary with a dirty session-owned linked repo is enough to retain the hosting
workspace.

Retention is separate from the original command/gate outcome: a stopped monitor stays
stopped, a failed adoption stays failed, and the hold records why work needs attention.
Explicitly stopping a monitor stops its process; it is not permission to discard its
carried edits. Preserve the parent design's separate top-level user-kill behavior
(discard the staged handoff); this is not a redesign of general agent kill or discard
semantics.

Use the existing best-effort worktree diff format, including staged, unstaged, and
untracked changes. The claim preserves the actual checkout, including binary and other
data the diff cannot fully reconstruct. Lossless snapshots and a new workspace archive
service are out of scope. Never release because diff capture failed or produced no patch
despite other dirty evidence.

## retention-core

1. Add typed request/decision wires and a pure retention policy in the existing Rust
   workspace/claim domain (use a focused module if needed). Inputs describe claim
   identity, same-workspace successor/creator custody, session-owned dirt, and evidence
   availability. Return an explicit retain/release/ownership-moved disposition with a
   stable reason; do not infer it from display visibility. Keep classification and
   identity matching in Rust, with a registered PyO3 binding, round-trip tests, and thin
   Python facade.
2. Add a shared Python settlement adapter under `turns/` or `workspace_provider/`.
   Collect only the relevant workspace and repository evidence, resolve the starter's
   existing baseline/custody through durable references, and perform the Rust-selected
   effect. Preserve origin repository IDs and staged IDs when known, without requiring
   them for legacy sessions.
3. Extend the claim mutation API as necessary so both hold and release recheck the exact
   owner under the ProjectSpec lock: workspace, workflow, Patch, artifacts identity, and
   expected owner PID/token. PID alone is insufficient because a long-lived scheduler
   can settle many gates. Reuse atomic pinning; never release and reacquire on disk.
   Clear occupants only on a successful ownership-checked release. A mismatched/replaced
   claim is a no-op.
4. Persist one replay-safe terminal retention record associated with the owner
   artifacts: workspace identity, originating session/turn, terminal reason, affected
   repo IDs, evidence/probe errors, and snapshot paths. Use atomic persistence. Preserve
   a terminal marker and index visibility so cleanup and dismissal can find the owner.
   Define retry ordering so a crash between evidence persistence and pinning cannot
   silently free known dirty work.
5. Generalize or wrap `monitor/worktree_recovery.py` for terminal owner kinds and
   relevant linked repositories, retaining its existing monitor field/path
   compatibility. Keep evidence capture best-effort and bounded. Do not launch
   follow-ups or change any monitor/gate branch in this phase.
6. Tests: policy truth table, same/different workspace successors, stale owner,
   primary-clean/linked-dirty, inherited versus unchanged foreign dirt, staged and
   untracked edits, missing baseline with positive custody evidence, probe/snapshot
   errors, repeated hold, and a claim transfer racing settlement. Use temporary
   repositories and real claim content for the adapter tests.
7. Verify sase-core and bindings, then advance `sase-core-revision.txt` to a host-landed
   core revision containing the API before Python consumers land. Do not invent a SHA or
   commit manually; use the project's host finalization and revision-pin workflow. If an
   exported adapter is temporarily unused, follow `symvision.md` and key any allowance
   to the child epic or its consuming open phase, never to this phase at closure.

## monitor-retention

1. Route all monitor terminal claim disposal through the shared adapter in
   `monitor/settlement.py`. Cover the legacy `settle_turn_claim_and_followup` path,
   frozen outcome policies, blocked continuation capture, stopped/lost states,
   no-follow-up outcomes, launch failure, and a degraded follow-up that uses another
   checkout. Pass concrete owner artifacts and transfer facts; do not use a naked
   recorded agent name as ownership proof.
2. Audit callers in `monitor/reconcile.py`, `monitor/store.py`, the proc settlement
   path, and host-completion settlement. Route converging paths to the same idempotent
   implementation. Preserve exit status, diagnostics, and follow-up result metadata.
   Prepared completion that committed successfully releases normally; failed completion
   with unresolved dirty work and no recovery successor retains it. A real
   same-workspace recovery successor keeps custody.
3. Connect the existing monitor resume flow to transfer a retained claim and its
   obligations safely. Clear/settle retention metadata only after transfer succeeds; a
   failed resume must leave the old hold and recovery evidence.
4. Tests exercise stopped and reboot-lost monitors, dead-supervisor and proc
   reconciliation, failed and absent follow-ups, capture blockage, degraded successor,
   prepared success/failure, and repeated settlement/resume. Assert the claim is pinned
   and the files remain in the checkout, not merely that a helper was called. Assert no
   hold after confirmed same-workspace transfer and normal release for clean work. Cover
   both supported handoff protocols.

## gate-adoption-retention

1. Integrate the shared adapter into `gate_turn/start_claim.py` and
   `gate_turn/handoff_launch.py`, including exception cleanup and terminal branches with
   no follow-up. Questions already use UserQuestion gates; cover their cancellation
   through `gate_turn/cancel.py` and settlement, rather than introducing a separate
   question policy.
2. Preserve restoration of a still-live creator's claim for synchronous gate use; that
   creator remains responsible for its work. Preserve release-mode gate no-ops. If
   publication/launch partially succeeded, resolve actual transferred ownership before
   deciding whether the original owner retains.
3. At failed staged adoption (`run_agent_exec_staged_dispatch.py` and runner shutdown),
   retain uncommitted carried/prepared work even when ordinary failure notification
   settings would skip a hold. Persist evidence before destructive cleanup and keep
   staged receipts available. Reuse existing adoption code; do not implement the parent
   epic's re-adopt command or reconciler. If those interfaces have landed meanwhile,
   connect them to clear retention only after successful custody transfer.
4. Ensure shell settlement and starter shutdown cannot undo each other's holds or delete
   a successor's occupant. Do not turn all hidden failures into holds; this override
   requires unresolved session-owned work.
5. Tests: cancelled dirty question, terminal answered/cancelled gate without follow-up,
   follow-up exception, clean terminal gate, release-mode gate with a newly occupied
   workspace, synchronous creator restoration, failed adoption with primary/linked dirt,
   hidden/auto-dismissed/suppressed origin, and success after a partial adoption.
   Exercise both protocols where they remain supported and prove a real same-workspace
   successor prevents a hold.

## recovery-surfaces

1. Project terminal retention records into existing agent/monitor/gate details and
   completion notifications. Show "Workspace held: stranded work", the workspace
   identity, cause, affected repositories, recovery diff locations, and the applicable
   existing resume/re-adopt action. Runs without beads must have the same visible
   recovery path; do not create synthetic beads.
2. Feed the retained-work fact into `bead/stranded_facts.py` and the Rust
   `stranded_work` reducer for associated beads. An unresolved retained owner must not
   remain indefinitely classified as an active handoff merely because an old marker or
   successful adoption receipt exists. Preserve genuine active successor precedence and
   existing ancestor propagation. Update wire mirrors, binding tests, schema checks, and
   the core pin for any new API fields using the normal cross-repo workflow.
3. Complete explicit dismissal/release using the existing action and claim lifecycle.
   Make it work for a terminal gate whose temporary PID is still the scheduler's PID,
   while refusing to release a live successor or reused claim. Resolve terminal
   ownership in Rust; TUI code remains presentation and glue. Preserve saved evidence
   and mark retention settled after release so refreshes/reconciliation cannot recreate
   the hold from stale metadata.
4. Audit `ace/scheduler/stale_running_cleanup.py`, hold artifact lookup, and existing
   held-claim TTL handling. Ensure terminal monitor/gate/adoption records have the done
   marker/artifact identity the cleanup logic requires, and cannot be reaped immediately
   as stale dead-PID claims. Preserve the existing configured TTL and explicit release
   semantics; do not add a new retention setting. Label recovery evidence as best-effort
   and surface the existing expiry policy where it is relevant to the release guidance.
5. Keep existing keybindings and command grammar. Read `ace.md`, `tui.md`, and
   applicable child notes before editing. Reuse cached/indexed facts; never run worktree
   git probes during TUI render or navigation. Add focused presentation tests and update
   only affected visual goldens through the prescribed screenshot workflow.
6. Integration tests cover hold -> refresh -> stale cleanup -> resume/explicit release,
   scheduler-owned gate holds, absent-bead visibility, bead stranded classification,
   successful later transfer, cleanup racing a new occupant, TTL behavior, and replay
   after release. Verify no dirty checkout is made allocatable before its selected
   recovery/release action.

## Verification and landing

Each phase reads `lint_and_test.md` and runs focused tests plus `sase tool run check`
from every modified repository. For sase-core use its `AGENTS.md` workflow, registered
binding round trips, and full prescribed check; never bare cargo. Test the pinned
binding in sase. Do not run `check-full` unless separately instructed. Use
`/sase_monitor` for long verification rather than leaving commands alive at turn end.
For rendered changes use the targeted visual verification required by the TUI memory
notes.

Test staged and legacy behavior while both exist. If the parent flag-removal phase has
already landed, do not reintroduce the removed flag or writer; retain historical-record
tests and unconditional dirty-work protection. Coordinate through bead notes with the
parent adoption-recovery and retirement phases.

Every worker records newly discovered work as `PROPOSED FOLLOW-UP:` on its assigned
phase; it creates no task beads and closes no ancestor. If a check failure reproduces
identically on the clean base, record the evidence and any existing tracker in that
note, then close the completed phase. Run `sase bead epic-symbols <assigned-phase>`
before closure and resolve/re-key remaining allowances to an appropriate still-open
owner.

The child epic's land agent verifies all four phases and the full acceptance matrix,
records evidence for `sase-1pj.7`, checks `sase bead epic-symbols sase-1pj.7`, and
completes the child epic through its normal host-owned landing flow. Once the child epic
and every descendant are closed, that land agent closes **only `sase-1pj.7`** as the
authorized enclosing phase, with
`sase bead close sase-1pj.7 --note "<verified retention branches, recovery/release behavior, and checks>"`.
This is preparation for that land agent, not permission for a child phase worker to
close an ancestor. Never close `sase-1pj` or any other ancestor plan bead. Keep
`sase-1pj.7` open while this proposal awaits approval and implementation.
