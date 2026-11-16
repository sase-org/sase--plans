---
tier: epic
title: Complete monitor hijack custody and recovery
goal: Hijack the occupying monitor, retain its workspace, and recover exactly once
  through the recorded follow-up or a fresh command run.
parent_bead: sase-1t2.11
phases:
- id: custody
  title: Monitor stop adapter and durable hijack settlement
  depends_on: []
  size: medium
  description: 'custody: implement monitor-specific stop and settlement effects, durable
    recovery evidence, exact claim custody, and marker preservation for proc-backed,
    reconciled, and legacy monitors; keep public monitor eligibility refused until
    activation.'
- id: followup
  title: Recover the recorded monitor follow-up
  depends_on:
  - custody
  size: medium
  description: 'followup: implement monitor-scope Continue and Escalate through exactly-once
    recovery admission with bounded untrusted partial output; provide truthful Rust
    recovery rows and preserve the same workspace.'
- id: rerun
  title: Re-run the command as a new monitor
  depends_on:
  - followup
  size: medium
  description: 'rerun: implement a durable rerun recovery action using the existing
    monitor-start machinery in the held workspace, with duplicate and crash reconciliation,
    CLI support, and Rust-owned action validation.'
- id: activation
  title: Enable running and settling monitor hijacks
  depends_on:
  - rerun
  size: medium
  description: 'activation: gather durable monitor occupancy and pending-follow-up
    facts, enable Rust immediate/monitor and latched/monitor decisions with their
    mechanisms, and prove request-versus-settlement races preserve custody and launch
    at most one successor.'
- id: surfaces
  title: Present monitor recovery in the TUI
  depends_on:
  - rerun
  size: medium
  description: 'surfaces: render Continue, Escalate, Re-run command, View, Shell,
    and Dismiss from the shared preview, hide unsupported monitor actions, preserve
    worker-based decision caching, and verify targeted visual snapshots.'
- id: canary
  title: Exercise a real monitor hijack and recovery
  depends_on:
  - activation
  - surfaces
  size: medium
  description: 'canary: extend the real fakey launch/loader/TUI-operation canary with
    a running sleep monitor, same-workspace recovery and command rerun; flip the screenshot-shaped
    fixture and verify CLI/TUI parity.'
- id: land
  title: Verify and land the monitor adapter phase
  depends_on:
  - custody
  - followup
  - rerun
  - activation
  - surfaces
  - canary
  size: medium
  description: 'land: integrate all phases, finish monitor reference documentation,
    verify required checks and symbol ownership, and supply completion evidence for
    sase-1t2.11 without closing sase-1t2 or any higher ancestor.'
proposed_by: bbugyi200.athena.sase-1t2.11
proposed_with: codex/gpt-6-astra
create_time: 2026-11-16 15:27:09
status: wip
bead_id: sase-1t2.11.1
---

- **PROMPT:** [prompts/202611/monitor_hijack_adapter.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/monitor_hijack_adapter.md)
- **PARENT:** [202611/agent_hijack_dependability.md](https://github.com/sase-org/sase--plans/blob/main/202611/agent_hijack_dependability.md)
- **BEAD:** [sase-1t2.11.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t2/sase-1t2.11.1.md)

# Monitor hijack adapter

## Scope and evidence

This is the implementation child epic for `sase-1t2.11`, the `monitor_adapter` phase of
`plan:202611/agent_hijack_dependability.md`. Read that artifact with
`sase artifact read`; its architecture and final decisions govern this plan. The
supporting research is
`research:202611/agent_hijack_dependability/agent_hijack_dependability.md`. The parent
plan overrides the research's obsolete claim-carried latch proposal: the latch lives in
the hijack ledger and is enforced at explicit boundaries.

The assigned phase was in progress with no prior implementation notes when inspected.
Earlier shared work is already present: occupant resolution, target deduplication,
stop-blocked custody, durable agent-turn latches, launch guards, and exactly-once
recovery admission. Reuse it. At sase revision `b586776bf3`:

- Rust `agent_hijack/eligibility.rs` still returns `monitor_turn` refusal.
- `monitor/store.py::stop_monitor` has proc-backed and legacy stop paths but no hijack
  contract. `_stop_orchestrator_main.py` only terminates the LLM adapter.
- `monitor/_settlement_followup.py::settle_claim_and_followup` has no hijack checkpoint.
  It can dispatch continuation or prepared host completion, or settle workspace
  retention, before a hijack is observed.
- `monitor/proc_adapter.py`, `supervise.py`, and `reconcile.py` write their own terminal
  markers. A checkpoint that writes `hijacked` without adapting those writers will be
  overwritten by an ordinary `monitored` marker.
- `agent/hijack/latch.py` creates scope `llm` and rejects monitor targets and
  settled/dead owners. Those assumptions cannot serve monitor settlement.
- Rust preview lacks scope and a rerun action. Python `_with_release_rows`
  unconditionally appends Restart/Edit, and the TUI also constructs fixed rows. Existing
  recovery dispatch is for LLM successors.

This is multiple bounded coding tasks, not one tale. The ordered backend phases avoid
concurrent changes to core schemas and the revision pin. Only the TUI phase can overlap
activation; it consumes the already-established recovery contract.

## Contract and implementation boundaries

1. A running monitor holding the session's numbered workspace is immediately hijackable
   through adapter `monitor`. A command that has settled while its follow-up is still
   pending accepts a durable latch at boundary `monitor`. An already-delivered follow-up
   is never replayed; re-resolve the current occupant or return the existing precise
   terminal/ownership result.
2. Persist intent before signaling, stop through `stop_monitor`, and keep claim custody
   until command termination is proven. A surviving command remains `stop_blocked`; it
   must not acquire a false `held` completion or release path.
3. The shared settlement checkpoint wins before follow-up dispatch, frozen continuation
   policy, prepared host completion, or workspace release. It records the command's real
   state as `boundary_outcome`, suppresses automatic progression, and settles the member
   as `hijacked` with recovery identity.
4. Continue launches the recorded follow-up now; Escalate launches that follow-up on the
   selected higher rung. Both use the held workspace and bounded partial command output
   explicitly labeled untrusted. They do not rerun the command. Re-run command starts a
   new monitor attempt for the same recorded command in that workspace. View and Shell
   are read-only access actions; Dismiss snapshots before release. Hide Restart/Edit and
   LLM prompt-editing actions for monitors.
5. Missing follow-up data disables Continue/Escalate with an actionable reason; it does
   not prevent truthful Re-run/View/Shell/Dismiss availability. Never synthesize a task
   from an absent follow-up or claim interrupted output passed.
6. Rust owns decisions, action validity, transitions, and preview policy. Python gathers
   evidence and performs I/O, locking, signals, launches, and rendering. Use additive
   defaulted wire fields where compatible; keep ledger schema 1. Update bindings and
   every mirrored schema fixture if a version changes.
7. Preserve ordinary monitor stop, timeout, lost-owner, completion, retention, and
   continuation behavior when no matching hijack intent or latch exists. No feature
   flag: public monitor eligibility stays refused through the first three phases and
   becomes enabled only alongside the completed mechanism.

The parent authorizes only the `glossary:agent-hijack` memory change, owned by its land
phase. This child epic edits no memory notes and asks no new product decision. Gate,
waiter, and remote hijack work remains with the parent's other phases.

## Phase custody

Own `agent/hijack` monitor-effect glue, `monitor/store.py`, the shared settlement
funnel, its three callers, and the small Rust contracts required for these effects. Do
not broaden eligibility yet.

- Add a monitor stop adapter under the existing target reservation and intent protocol.
  Resolve the monitor by durable ID/artifacts and validate supervisor identity before
  calling `stop_monitor`. Do not send the generic LLM process-tree stop to a monitor.
  Preserve stop retry, duplicate request, and blocked-stop behavior. A dead supervisor
  goes through canonical monitor reconciliation.
- Persist an immutable monitor recovery descriptor before publishing a held result.
  Retain monitor/session/member identity, command or execution argv, cwd, timeouts,
  output policy, frozen follow-up/checkpoint/model references, command result and
  retained-output locators, and existing completion provenance. Use
  `suppressed_followup = {kind: monitor_followup, ref: <descriptor>}` in the ledger;
  keep large output outside the record. Inspect current continuation records rather than
  reconstructing the modern protocol from legacy `--next` alone.
- Add one settlement hook at the start of `settle_claim_and_followup` that handles a
  matching durable intent and, when present, a monitor latch. Use shared CAS and
  evidence/snapshot helpers. Skip normal release and host-completion paths once hijack
  wins. Protect the exact project/workspace/artifacts generation and re-own custody from
  the exiting supervisor as required by the existing claim API; never use an
  ownership-blind release or blindly steal a replacement claim.
- Return an explicit hijack settlement disposition to all callers so subsequent metadata
  and done-marker writes retain the recovery record, `hijacked` outcome, real monitor
  state, and Stopped presentation. Preserve those fields on replay, reconciliation, and
  stale metadata merges. A half-finished checkpoint must retain custody and remain
  repairable rather than resume ordinary progression.
- Observe lock order: adoption, monitor lane, hijack target/ledger, then patch lock
  innermost. Do not hold a monitor lane lock while waiting for a supervisor whose
  settlement needs that lock. Persist/fence under lock, release it for
  signaling/waiting, and reacquire for identity-checked settlement as needed.

Tests: real marker-shaped proc and legacy inputs; intent persistence failure before
signals; dead supervisor; duplicate settlement; surviving writer; changed claim;
snapshot/marker failure; no automatic follow-up, completion finalizer, or release on
hijack; ordinary monitor behavior unaffected. Exercise all three terminal writers, not
just a mocked call to the new hook. Seed legitimate ledger records in these internal
tests while public monitor decisions remain refused.

## Phase followup

Own Rust monitor preview/continuation policy and Python keep-work recovery adapters.
Consume the descriptor from custody and reuse the existing recovery reservation,
revision checks, launch identity, admission, and confirmation path.

- Make scope and recorded-follow-up availability inputs to the Rust preview. Return
  truthful monitor rows for Continue, Escalate, View, Shell, and Dismiss. Re-run may be
  explicitly unavailable until its executor lands in the next phase. Do not append
  Restart/Edit back in Python. Preserve LLM previews.
- Compose the recorded monitor follow-up using the existing frozen-intent,
  result-selection, and continuation-budget machinery. Label interrupted/partial output
  and suppressed descriptor contents untrusted; bound and truncate before assembly. Use
  the follow-up's concrete model selection for Continue and the existing escalation
  policy for Escalate, not a stale starter model by accident.
- Dispatch through hijack recovery ownership. Reuse monitor prompt assembly but do not
  invoke an ordinary launch path that can fall back to a fresh/pool/#0 workspace.
  Authorize exactly the reserved successor through the hijack spawn guard. Preserve
  goal, bead, repository baseline, session and finalizer lineage.
- Keep the old monitor's automatic-delivery receipts suppressed after explicit recovery.
  A recovery launch failure or unconfirmed child leaves custody held and a retryable or
  reconcilable record. Confirm a successor before resolving the predecessor; do not
  replay prepared host completion on partial output.

Tests: modern and legacy descriptors, explicit next-model selection, bounded output with
hostile instruction text treated as data, missing/corrupt follow-up, same-workspace
Continue/Escalate, duplicate actions, admission failure and crash reconciliation. Cover
Rust policy and PyO3 round trips as well as Python effects.

## Phase rerun

Own the new command recovery action and its CLI/operation path. Use canonical action
spelling `rerun` and user label `Re-run command`.

- Extend Rust action validation and resolution policy for monitor-only rerun; expose it
  in the preview only when the immutable command descriptor and held claim are valid.
  LLM records must refuse it. This is a keep-work operation, never the Restart
  release/fresh-workspace path.
- Reserve a unique successor monitor and launch identity through the durable recovery
  transaction. Adapt existing monitor-start preparation and supervisor acknowledgement
  to transfer the exact held claim and start the same command, argv/cwd, limits, and
  follow-up policy under a new monitor/member ID. Do not revive the stopped process or
  copy old terminal markers/delivery receipts.
- Reuse admission and reconciliation infrastructure; extend it for a monitor successor
  where its LLM assumptions prevent this. Confirm new supervisor and claim ownership
  before resolving the hijack. Repeated clicks, retries, and crashes around
  launch/acknowledgement must create at most one new command.
- Never reuse a verification/completion proof bound to the interrupted ToolRun. Recreate
  eligible completion provenance through the existing fresh-run protocol; if
  reconstruction is not valid, disable rerun with the exact reason while retaining
  custody. Keep the original monitor's automatic follow-up suppressed.
- Wire `sase agent recover -a rerun` through parser, CLI validation/output, durable
  `persist-recover` operations and backend dispatch. Reuse existing option names, sort
  action choices, provide help/examples, and reject incompatible prompt/model/discard
  options explicitly. Generate CLI references with `just fix`.

Tests: command/argv/cwd preservation, same workspace/new monitor ID, eventual ordinary
follow-up, no old automatic delivery, duplicate action, failed start, crash before/after
acknowledgement, and stale/moved claims. Include real local short commands where mocks
would hide claim-transfer or process-lifetime errors.

## Phase activation

This phase enables public monitor hijacks only after custody and both recovery executors
are present. Own eligibility, facts, monitor-aware latch validation, and race coverage;
leave TUI rendering to surfaces.

- Add explicit durable facts for a command-settled but follow-up-pending monitor. Derive
  them from monitor settlement/delivery records, not a status label or merely the
  existence of `monitor_next_action`. A done marker must not mask a still-pending owned
  handoff. Keep claim-first occupant selection and identity re-resolution; preserve
  genuinely terminal monitor decisions.
- In Rust return immediate/adapter `monitor` for supported running monitors and
  latched/boundary `monitor` for pending follow-up settlement. Keep numbered custody,
  locality, session attachment, and exact-owner checks. A monitor uses its supervisor
  identity, not an inherited LLM capability or starter PID as proof that it can be
  stopped. Cover proc-backed and legacy identity evidence.
- Route the immediate decision into custody's stop adapter. Extend latch request
  validation and scope creation for monitor owners without relaxing LLM/gate invariants.
  A command may be dead while the settlement owner still owns the boundary; use durable
  ownership facts and revalidation rather than requiring the command to be alive.
  Reconcile abandoned monitor latches through the monitor settlement adapter, not an
  LLM-only target builder.
- Serialize requests versus follow-up reservation under the monitor lane and existing
  hijack locks. If the hijack wins, apply once and preserve the real command outcome. If
  delivery wins, never reclaim its transferred workspace or create a second successor.
  Preserve existing cancel/force behavior; an explicit forced monitor stop must use the
  monitor adapter and truthful uncertainty.
- Integrate terminal manual-resume/reconciliation paths so they cannot revive an
  automatically suppressed delivery behind an active hijack. Continue/rerun
  authorization comes from the recovery transaction, not an incidental resume.

Tests: exhaustive Rust running/pending/delivered/no-claim decisions; real facts for
session name and container row; request versus completion/delivery barriers; cancel,
abandoned owner, forced stop and duplicate requests; `HijackLatchedError` never
triggering a fallback workspace; lock-order tracing and replay convergence.

## Phase surfaces

Read `ace.md`, `tui.md`, `tui_perf.md`, and `tui_screenshot.md` through memory. Own
`_hijack_recovery_panel_impl.py`, preview/action adapters, necessary help, and their TUI
tests. Consume the shared rows rather than duplicating scope policy.

- Display Continue, Escalate, Re-run command, View, Shell, and Dismiss for monitor
  records, with unavailable reasons from the backend. Keep current escalation naming
  when the existing policy chooses Switch. Hide Restart/Edit/Instruct for this scope.
  Preserve LLM panels and their bindings.
- Route rerun through the durable recovery operation and show the actual monitor
  member/workspace in the result. Enter from a collapsed session must open the occupying
  monitor's recovery. View/Shell must not resolve the hijack.
- Keep fact gathering off the Textual event loop and preserve the decision cache/refresh
  behavior. Update `?` help and default keymap configuration only if an actual
  binding/config change is needed.
- Add panel/action tests for held, resolving, expired, absent follow-up, and
  command-unavailable states. Capture targeted monitor recovery goldens with
  `just fix-tui-screenshots`; inspect the report and images before finalizing.

## Phase canary

Reuse the current split fakey canary helpers under `tests/fakey/`; do not build a second
harness. Start through the real launch pipeline, stage/adopt a sleep monitor, and wait
on explicit barriers that prove the monitor owns the claim.

- Load actual TUI rows, select the session container, and assert the cached decision and
  CLI explain agree on immediate/monitor and the occupying member.
- Submit through the TUI's durable hijack operation. Verify one ledger record, command
  termination, HIJACKED/Stopped rendering, retained snapshots, exact numbered claim, and
  no recorded automatic follow-up launch.
- Continue through the real recovery command; prove one successor runs the recorded
  follow-up in the same workspace and sees bounded untrusted output. Add a separate
  fresh scenario for rerun: one new monitor command starts in the held workspace and its
  eventual follow-up advances once.
- Flip the existing screenshot-shaped fixture in `test_agent_hijack_facts.py` from the
  precise monitor refusal to immediate. Cover a settled command held before delivery
  with a deterministic boundary test. Keep lower-level legacy coverage from custody; do
  not fake runner metadata to make the canary pass.

Run the targeted canary with its established e2e markers/budget. Record the monitor
checkpoint and successor counts for the parent chaos phase `sase-1t2.13`; do not take
over that phase's global crash/inventory work.

## Verification and landing

Every worker checks existing work and predecessor notes before editing. Open `sase-core`
with `/sase_repo` and read its instructions. Core tests use its `just` wrappers, never
bare cargo. Follow its current release-plz convention: do not hand-edit versions or
CHANGELOGs. Keep schemas, PyO3 registrations/round trips, Python wrappers, and source
pin compatible. Read `docs/contribute/rust-backend.md` for the CI source revision pin;
coordinate the pin after host finalization provides the actual core commit, and do not
invent a SHA or call unavailable bindings. Rebase concurrent parent-epic changes.

After code changes, run focused behavioral tests and the required `sase tool run check`
in each changed repository. Read `lint_and_test.md`; run format/fix before final
verification. Route long checks and visual captures through `/sase_monitor` and wait for
the staging command to exit. Do not run `check-full` without an explicit instruction.
Passing focused tests never replace the required checks. A failure reproduced
identically on clean base is recorded as `PROPOSED FOLLOW-UP:` with evidence and any
existing tracker ID; it does not leave a completed phase open. Do not create task beads
manually.

The child land agent:

1. Integrates all phases and verifies the enabled policy, both recovery paths, three
   settlement writers, CLI/TUI agreement, screenshots, and real canary. Check that the
   final sase-core source pin includes every new contract.
2. Reads `docs_writing.md` and updates `docs/reference/automation/monitor-operations.md`
   with monitor hijack, partial-output Continue/Escalate, rerun, and claim retention
   semantics. Fix only now-incorrect related Reference/help text and regenerate CLI
   references. Do not edit memory; the outer epic land phase owns the authorized
   glossary edit.
3. Runs `sase bead epic-symbols` for each phase before its close, and for `sase-1t2.11`
   before recording parent completion. Resolve entries or re-key only genuinely
   unfinished symbols to an appropriate still-open bead.
4. Records the completion evidence on `sase-1t2.11` and completes this child epic
   through its host land workflow. The land workflow is responsible for closing the
   delegated phase after its child descendants are settled. Individual child phase
   workers close only their own assigned bead; an ancestor-close instruction here is
   preparation/evidence for the owning land workflow, never permission for a phase
   worker to close an ancestor. Never close outer epic `sase-1t2`.

The outcome is complete when a running or pending-follow-up monitor can be hijacked
once, its command and workspace remain consistently accounted for, and the user can
recover the intended next action without a duplicate command, duplicate successor, lost
workspace, or overwritten hijack marker.
