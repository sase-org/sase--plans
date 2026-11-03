---
tier: epic
title: Finish stranded workspace retention after the landing audit
goal: Terminal handoffs retain the exact dirty workspace through settlement failures,
  use real custody evidence, and converge recovery state after transfer or release.
parent_bead: sase-1pj.7.1
phases:
- id: settlement-evidence
  title: Make retention evidence and failure settlement safe
  size: medium
  depends_on: []
  description: 'settlement-evidence: unify monitor repository evidence, require actual
    successor custody, and preserve exact claims when retention cannot complete.'
- id: gate-origin
  title: Preserve the actual origin across failed gate creation
  size: medium
  depends_on:
  - settlement-evidence
  description: 'gate-origin: repair session-member creator identity and retain dirty
    origin work when detached gate publication fails.'
- id: recovery-convergence
  title: Converge recovery records and verify the combined lifecycle
  size: medium
  depends_on:
  - settlement-evidence
  - gate-origin
  description: 'recovery-convergence: settle records through one contract after real
    transfer or release, give truthful recovery guidance, and verify the integrated
    acceptance matrix.'
proposed_by: bbugyi200.athena.sase-1pj.7.1.land
proposed_with: codex/gpt-6-astra
create_time: 2026-11-02 16:57:19
status: done
bead_id: sase-1pj.7.1.5
---

- **PROMPT:** [prompts/202611/retention_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/retention_landing_repairs.md)
- **PARENT:** [202610/carried_workspace_retention.md](https://github.com/sase-org/sase--plans/blob/main/202610/carried_workspace_retention.md)
- **BEAD:** [sase-1pj.7.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.7.1.5.md)

# Finish stranded workspace retention

This is remaining work from the landing audit of `sase-1pj.7.1`, not a replacement for
its completed implementation. Read `plan:202610/carried_workspace_retention.md`,
`plan:202610/staged_handoffs.md`, and this epic's parent notes using audited commands.
All accepted decisions remain binding. This plan edits no memory, changes no
keybindings, and does not retire the parent's handoff flag or redesign explicit
top-level user kill.

This needs coordinated work in shared retention/claim effects, gate creation and runner
identity, and the recovery lifecycle. Each phase is a bounded direct implementation
task; the combined cross-process failure and cross-repository work is larger than one
medium tale. The dependencies keep each later phase on the landed shared contract,
including any required Rust binding pin.

## What the landing audit verified

The audit was against sase `d2061888ec` and core pin
`89a5bd8b810ab20fee5c223bb4b1e40cb2c3ab68`. It read all four child beads and every note,
including all 132 notes on `sase-1pj.7.1.4`. No child has a `PROPOSED FOLLOW-UP:` entry.
Source and commits confirm:

- `17f5aa3ca1` / core `80338d6b`: Rust retention policy, exact claim pin/release,
  repository evidence adapter, and best-effort snapshots.
- `f2e84801cc`: gate settlement and failed staged-adoption retention.
- `745d7b57b1`: monitor settlement and resume integration.
- `31a28d9618` / core `154fe9e2`: held-work surfaces, explicit dismissal, and
  retained-owner precedence in the stranded-work reducer. Both core commits are
  ancestors of the current pin, with registered bindings.
- `d2061888ec`: detached recovery writes terminal retention evidence and pinned staged
  claims obey the configured TTL. This resolves parent note #4. Preserve its exact-owner
  cleanup and successor-race protections.

The audit reviewed the non-epic commits since `17f5aa3ca1`, especially `288d2afeeb`,
`a19243cdf6`, `078e3da1d6`, and `a79207993c` (readopt, reconciliation, claim delivery,
admission), `17d8fe5937` and `71827cfecc` (gate entry and lost intents), and
`dee9354664` / `5811e00b46` (recovery docs). Use their current code; do not restore
pre-split modules or duplicate their recovery services.

## Confirmed gaps

1. `monitor/settlement.py::_retention_successor_from_meta` turns only
   `monitor_followup_outcome == "launched"` into all three positive custody facts. A
   runtime probe with that single metadata field and a still-owned dirty original claim
   makes Rust return `release / same_workspace_successor`. No successor identity,
   liveness, inherited obligations, or actual transfer was supplied.
2. Monitor discovery duplicates the shared discovery adapter. It invents
   `primary`/`linked-N` IDs, stops after eight repos, ignores opened-repo markers, and
   silently drops a repo when its preliminary Git probe fails. It reads starter baseline
   paths but does not set `evidence_artifacts_dir`, so the shared adapter compares dirt
   against the monitor member's baseline instead of the starter's. This can both retain
   unchanged foreign dirt and omit session-owned linked dirt.
3. Parent note #3 reports real stale-interpreter failures after an editable update:
   missing Rust retention binding and an old `release_workspace` signature. Monitor
   owner discovery imports the adapter before the try block. The later exception handler
   returns an error without pinning or publishing retained-work evidence. The exact
   claim must survive cleanup even when settlement is unavailable; merely leaving a
   dead, unpinned claim untouched is insufficient.
4. Parent note #5 records lost primary and linked edits from a code member whose gate
   creator timestamp/Patch differed from the live RUNNING claim.
   `gate_turn/transaction.py` settles a failed claim move with `creator_live=True`,
   suppressing claim disposal even when publication is detached and the provider has
   already ended. A malformed identity must not authorize stealing another claim, but
   the legitimate terminal origin cannot silently lose custody either.
5. `monitor/resume.py::_settle_retention_after_resume` writes `settled: true` and
   `settled_via`, while `WorkspaceRetentionRecord.unresolved` only checks `settled_at`.
   A runtime probe calling the actual producer and reader returns `unresolved=True`
   after a reported successful resume. The already-existing receiver branch does not
   call the settlement helper at all. Later staged readoption and normal clean
   settlement also need to converge the same record.
6. Held monitor details advertise `sase monitor resume` for stopped/lost monitors, but
   `_validate_resume_request` rejects both as inspection-only. Parent note #1 also
   requires stopped reboot work to stop appearing owned by a dead monitor. The current
   retained-owner reducer override helps only when a retention record was actually
   written.
7. `sase bead epic-symbols sase-1pj.7.1` lists `sase-1pj.7.1(WorkspaceRetentionResult)`
   locally and upstream. This class has only in-file consumers and should be private.
   Resolve it in the first phase so the later lander sees the cleanup already landed.

## settlement-evidence

1. Open `sase-core` through `/sase_repo` and read its `AGENTS.md`. Reuse its
   `workspace_retention` and `agent_launch` policy/wires. Shared classification,
   identity matching, and any new pending-settlement/recovery rules belong there; Python
   collects evidence and performs I/O. There is no Python policy fallback.
2. Replace monitor-specific repository enumeration with
   `discover_workspace_retention_repos` and durable starter/carry evidence. Preserve
   real repo IDs, opened-repo markers, the relevant baseline path, and all positively
   owned repos. Remove the silent eight-repository cutoff. A bounded probe that fails
   yields uncertainty for that known repo; it cannot disappear as if it were clean.
   Distinguish a genuinely repo-less command from an unavailable owned checkout. Keep
   probes out of TUI refresh/render paths.
3. Build successor facts from current exact claim/delivery/process identity and
   inherited obligations, reusing the existing delivery proofs. A recorded name,
   outcome, or historical `carried_to` is not proof. A real transferred claim must
   remain untouched; a dirty original with only an old launch label must retain. Remove
   unchecked legacy release fallbacks for unresolved owner metadata: lack of evidence
   must not authorize releasing a potentially replaced claim.
4. Put owner discovery and settlement effects inside one shared failure boundary used by
   monitor, gate, and adoption callers. Persist terminal/error evidence and pin the
   exact matching claim through the existing guarded claim API on policy, probe,
   snapshot, or effect failure. Ensure metadata-write failure cannot abort before
   custody is protected. Recheck identity under the existing ProjectSpec lock; never
   release and reacquire and never drop ownership arguments to make an old signature
   accept a call.
5. Cover the stale-interpreter case explicitly: when the loaded adapter/binding or
   effect signature is stale, run the same Rust-backed settlement in a bounded fresh
   interpreter with the recorded owner facts. Do not retry follow-up spawn or the
   monitored command. Record a durable pending-retention fact before this attempt, and
   make claim cleanup honor that fact for the exact owner until a matching guarded
   settlement/recovery decision has completed. An unavailable fresh binding leaves
   visible, retryable protected custody; it does not become a Python implementation of
   the Rust decision. Reuse the existing reconcile mechanism for the retry, with
   identity checks and terminal outcomes preserved.
6. Define replay and crash ordering across pending evidence, terminal marker, pinning,
   and final retention-record publication. Recovery must not overwrite a settled record
   with a fresh unresolved hold or clear a successor's occupant. Preserve configured
   held-claim TTL behavior after a terminal hold is established.
7. Privatize `WorkspaceRetentionResult` unless implementation establishes a real
   non-test external consumer. Remove the existing parent-keyed Justfile allowance; do
   not move it to an unrelated open bead merely to pass closure.
8. Test real claims and temporary repositories: unchanged foreign starter dirt; clean
   primary with a dirty opened linked repo beyond position eight; failed Git preflight;
   stale launch label with original claim; actual successor transfer; missing binding
   and old effect signature; persistence failure; crash/replay boundaries; and stale
   cleanup while settlement is pending. Assert preserved files and claim identity, not
   just helper calls. Verify any core changes and bindings through its prescribed
   `sase tool run check`, then let host finalization land them. Use the existing
   host-owned cross-repository revision-pin workflow for a phase that changes both core
   and its Python consumer; verify the local development binding before declaration.
   Later phases must confirm the committed sase pin contains that core change before
   using it. No manually invented SHA or manual commit, and no waiting in a provider
   turn for its own commit.

## gate-origin

1. Trace the session code-member transition into `CreatorContext`, RUNNING claims,
   `run_agent_exec_plan*`, staged creator payloads, and runner shutdown. Reproduce the
   actual same-runner/different-artifacts identity from parent note #5 with a primary
   and a privately opened linked repo. Separate durable session naming from the exact
   turn that owns the claim. Correct the producer of the mismatch; do not weaken
   `_claim_still_owned_by_creator` to a PID-only comparison.
2. Pass whether the creator is truly synchronous and still responsible through gate
   publication and replay. Keep `%auto`/synchronous restoration intact. Detached or
   failed terminal publication must settle the actual proven origin with the shared
   adapter, rather than unconditionally using `creator_live=True`. Wire claim-move
   refusal, publication exception, and partially published gate paths. A claim belonging
   to another owner remains untouched and is diagnosed.
3. Integrate runner shutdown and gate shell settlement so neither can undo a hold or
   erase a successor's occupant. Preserve explicit release-mode gates and top-level
   user-kill behavior. Protect failed legacy handoffs while that writer remains
   supported; if the parent retirement lands first, retain historical record tests
   without restoring deleted writers or flags.
4. Test the full detached code-member -> gate refusal -> terminal visibility -> stale
   cleanup path with actual primary/linked edits. Test synchronous live creator
   restoration, same PID with a genuinely different owner, release-mode reuse, failure
   after partial publication, replay, and successful same-workspace successor transfer.
   Confirm a held origin is discoverable even without a bead. Run the prescribed check
   and record precisely which identity caused the report.

## recovery-convergence

1. Use one retention-record settlement API and field contract for resume, normal clean
   release, explicit dismissal, TTL expiry, and successful staged readoption. A
   successful operation settles the old record only after proven custody
   transfer/release; failure, unknown receiver, or a degraded different-workspace launch
   leaves it unresolved. Include `existing_receiver` resume reconciliation. Handle
   existing `settled: true` records deliberately using actual transfer evidence; do not
   blanket-hide records solely because a historical boolean says success. Shared
   decisions stay Rust-owned.
2. Audit `run_agent_exec_staged_recovery*`, `run_agent_successor_delivery.py`, and
   `staged_handoff_reconcile.py` against the new adapter. Keep the terminal visibility
   and TTL fixes from `d2061888ec`. A delayed failed-adoption callback must not recreate
   an old hold after success, and the successor inherits the original repositories and
   their obligations.
3. Preserve stopped/lost monitor outcome and existing inspection-only command semantics.
   Do not suggest an ineligible resume/readopt command. Show the applicable existing
   recovery action when eligible and the explicit dismissal action for a terminal hold.
   Verify retained stopped/reboot work is classified stranded with a real live successor
   taking precedence, including old failed settlement records that the retry path
   repairs. Runs without a bead still have visible held-work details and release access.
4. Update the affected recovery docs from `dee9354664` / `5811e00b46` to match actual
   eligibility and record lifecycle. Read `docs_writing.md` before editing; read
   `ace.md`/`tui.md` before any ACE edits. Keep file-system/git work out of UI
   navigation, reuse cached facts, and run targeted visual verification if rendered
   output changes. No memory edits are part of this child epic.
5. Add producer-to-consumer regression tests through the real record reader and
   stranded/UI projections, rather than asserting only JSON keys. Exercise hold ->
   refresh -> cleanup -> recovery/release -> replay for monitors, gates, and staged
   origins, with actual claims and files. Confirm no successful transfer still says
   held, no failed transfer frees work, no successor occupant is removed, configured TTL
   remains effective, and repeated notification/retry does not create duplicate holds or
   successor launches.
6. Recheck all six original parent notes and the new audit notes against the combined
   source. Run focused tests and `sase tool run check` (the wrapped `just check`) in
   every modified repository. Never run `just check-full` without a separate explicit
   instruction. Use `/sase_monitor` for long checks; do not leave commands alive when
   ending a turn. Record evidence on this phase; it does not close an ancestor or mark
   the parent plan done.

## Follow-up dispositions already completed

The lander used `/sase_new_task`, searched all-status bug tasks, swept recent tasks, and
inspected active epic scopes. There were no child `PROPOSED FOLLOW-UP:` notes.

- Parent note #2 and the repeated close-blocked child notes corroborate existing ready
  task `sase-1qv`. At the audited HEAD, `epic_symbols.load_epic_symbol_scan` still
  unions local and upstream rows; `5585fadefd` only added freshness preflight. A +1
  records independent source evidence and this epic's repeated impact. A new
  automatic-close task was declined as duplicate desired remediation.
- Parent note #6's missing historical slot-refusal/epic-symbol repairs also belong to
  `sase-1qv`; the same +1 covers them. The enclosing `sase-1pj` received a note about
  the legacy commit-skipping plan handoff and historical plan-status loss. This work
  cannot reconstruct discarded edits and does not rewrite unrelated completed plans.
- Note #4 is resolved by `d2061888ec`, so no new task was created for it.
- Notes #1, #3, and #5 and the newly confirmed evidence/record defects remain epic work
  in this plan. They were not deferred to tasks.

## Landing handoff

The `parent_bead: sase-1pj.7.1` relationship is the handoff back to the interrupted
landing. This child epic has no phase for closing its parent. Its land agent must first
finish and close this child normally, then re-audit the directly parented plan
`sase-1pj.7.1`: all descendants and notes, the linked plan
`plan:202610/carried_workspace_retention.md`, the original acceptance matrix, and
post-child drift. Include the follow-up dispositions above in the parent's close note.

When that parent is complete, resolve every entry from
`sase bead epic-symbols sase-1pj.7.1`, close it with
`sase bead close sase-1pj.7.1 --note "<verified retention, failure, integration, and recovery behavior plus follow-up dispositions>"`,
run `just symvision`, and set `status: done` in the frontmatter of its linked plan
`202610/carried_workspace_retention.md` using the opened plans repo. Resolve the current
PLAN path through an audited bead/artifact read; no agent workspace path is portable.
The first phase's symbol removal should already be landed by then.

Next inspect parent phase `sase-1pj.7`, verify this completed child covers its
carry-hardening scope, check its remaining epic symbols, and close only that phase
normally with
`sase bead close sase-1pj.7 --note "<verified complete carry-hardening scope and checks>"`.
Leave containing epic `sase-1pj` and its plan status to its waiting land agent. Never
use `--force` to advance a successful nested landing. If any readiness check fails, keep
the affected parent open and record the specific remaining blocker.
