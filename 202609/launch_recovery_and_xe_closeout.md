---
tier: epic
title: Repair approval launches and finish the remote Agents epic
goal: Preserve requester continuation and exact launch targeting, complete the outstanding
  real and live acceptance evidence, and carry the existing landing chain through
  a verified normal close of sase-xe.
parent_bead: sase-xe.16.11.7.14.6
phases:
- id: launch-targets
  title: Preserve workspace and remote targeting through approved admission
  size: medium
  depends_on: []
  description: 'launch-targets: repair the Rust typed launch round trip and Python
    approved dispatch path, including remote identity and durable receipt handling;
    prove the reviewed project and machine are the actual execution targets.'
- id: requester-continuation
  title: Resume requesters and isolate inherited operation context
  size: medium
  depends_on: []
  description: 'requester-continuation: add an explicit durable continuation contract
    for helper launches, prevent competing family successors, scrub parent operation
    sidecars at agent boundaries, and update launch skill guidance and regressions.'
- id: acceptance-regressions
  title: Finish actual instance replacement and unified cutover checks
  size: medium
  depends_on:
  - launch-targets
  - requester-continuation
  description: 'acceptance-regressions: complete the captured-old-instance fencing
    proof, verify the existing successful HTTPS host fixture, resolve unified flag
    retirement, and reproduce and repair the reported target-picker focus hazard.'
- id: released-runtime
  title: Publish and install the repaired execution cohort
  size: medium
  depends_on:
  - launch-targets
  - requester-continuation
  - acceptance-regressions
  description: 'released-runtime: adopt the published repaired Rust contract, validate
    clean wheel and installed commands, deploy landed skill sources, refresh both
    machines through supported workflows, and record running binary identities.'
- id: stale-row-proof
  title: Complete the interrupted snapshot and dismissal proof
  size: medium
  depends_on:
  - released-runtime
  description: 'stale-row-proof: prove actual Apollo presentation from Athena, controlled
    dismissal and death reconciliation, older history, snapshot changes and restart
    resilience; produce the evidence owed by sase-xe.16.11.7.14.6.6.'
- id: unified-live-proof
  title: Complete the unified workflow and combined acceptance matrix
  size: medium
  depends_on:
  - stale-row-proof
  description: 'unified-live-proof: execute the remaining unified Agents research
    scenarios, prove exact remote output and stop, attention and uncertainty recovery,
    run combined checks and supply evidence for all reopened original phases.'
proposed_by: bbugyi200.athena.0jc
create_time: 2026-09-11 09:46:22
status: wip
bead_id: sase-xe.16.11.7.14.6.7
---

- **PROMPT:** [prompts/202609/launch_recovery_and_xe_closeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/launch_recovery_and_xe_closeout.md)
- **BEAD:** [sase-xe.16.11.7.14.6.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.14.6.7.md)

# Repair approval launches and finish the remote Agents epic

## Outcome and scope

Finish the existing epic, including its older unfinished obligations. A successful
helper launch must leave a durable path back to its requester's assignment. A reviewed
Apollo/SASE launch must actually run on Apollo in the requested project. The final
acceptance must prove the released, unified Agents experience and culminate in normal
bead and plan completion through `sase-xe`.

This is an epic because independent backend/host repairs precede release adoption and
two serialized live sessions. Every implementation phase is bounded direct work. The
parent is the existing snapshot-acceptance epic; this child supplies its missing work
and the evidence required further up its ancestry. Lifecycle recovery belongs to the
landing section below, not an extra implementation phase. Do not invent another broad
redesign, reimplement the already delivered fleet machinery, or restore Focus/Fleet UI.

## Evidence and causal chain

Investigation baseline: SASE `8f6e65361`, 2026-09-11. Source inspection used this
checkout and a core checkout opened with `sase repo open sase-core`. No implementation
or operational repair was performed while authoring this plan.

1. The actual worker transcript is
   `gh_sase_org__sase-tmp_260911_082250-main-260911_084441`, obtainable with
   `sase chat show --basename`. The named lookup currently resolves to the family's
   final launch gate, not the whole worker conversation. The worker's stable run
   artifacts are under the SASE project artifact record for `20260911072717`.
2. The worker updated Athena and Apollo, restarted gateways onto core 0.34.2, ran index
   GC, and inspected a projected catalog. These are partial results. Its transcript
   contains no completed pane/dismissal/death/restart/paging proof and no combined
   acceptance gate. Preserve its command evidence, but do not promote its commentary or
   numeric counts into a current acceptance result.
3. Both initial test launches failed because an inherited `SASE_PROC_REQUEST_PATH` named
   an `agent.drain` request while `run.launch` was loading it. The worker then cleared
   operation variables and retried `sase run`. Agent-side interception created gate
   `launch-871d5601-2ef9-493c-b7d8-f204eb5a6084` and terminated the worker turn.
4. That gate was approved at about 09:13 EDT. Its immutable request has null follow-up
   prompts for approve, reject, failure, timeout and cancellation. Approval dispatched
   one unit and ended; it never resumed the acceptance assignment. The source is
   `src/sase/agent/launch_request.py::_launch_shell_gate_spec`; the request normalizer
   also has no supported requester-continuation field. `done.json` saying completed or
   the gate saying LAUNCHED does not mean the bead's goal completed.
5. The requested prompt explicitly included an Apollo dispatch target, an explicit
   observation-agent identity, and the SASE GitHub workspace reference. The typed plan
   retained `selected_project=gh_sase-org__sase`, but its Agent unit lost the workspace
   reference. Its reconstructed prompt retained the dispatch directive as text.
   `launch_admission_runtime.dispatch_agent_unit` calls the local
   `launch_agents_from_cwd` path; the remote routing in `query_handler/_launch.py` is
   outside that path. The result was `sase-xe-live-dismiss-6614` running locally in
   project home. The second running/death observation agent never launched.
6. A pure current-source probe against the installed core 0.34.4 reproduces the lost
   workspace reference and reconstructed remote directive without launching anything.
   The checkout's editable core extension is missing, so this probe is evidence of the
   installed binding, not a passing checkout test suite. Restore the supported
   development install before implementation verification.
7. Read-only runtime checks found Athena's installed core already at 0.34.4 while its
   running gateway remained 0.34.2. Apollo's installed and running core were 0.34.2.
   Both gateways were healthy and loopback-bound. Package metadata alone cannot prove
   that running services have adopted a fix.

The gate request, response, launch-admission receipt, worker tool log, and local child's
run metadata are the primary incident evidence. Read them through the applicable SASE
chat/status/artifact tools and retain redacted excerpts in the implementation report. Do
not edit or replay the old approved bundle or force-reuse its agent name.

## Remaining work ledger

The current store contains 67 beads in the `sase-xe` tree; 11 are not closed. Re-read
the live store before acting. These five phase obligations remain:

| Original phase           | Actual remaining obligation                                           | Evidence supplied here                           |
| ------------------------ | --------------------------------------------------------------------- | ------------------------------------------------ |
| `sase-xe.16.11.7.14.6.6` | Actual stale-row, dismissal, death, history and restart acceptance    | stale-row-proof and combined verification        |
| `sase-xe.16.11.7.13`     | Unified cross-machine research scenarios, fresh output and exact stop | unified-live-proof                               |
| `sase-xe.16.11.3`        | Real healthy-host/deadline and captured replaced-instance proof       | acceptance-regressions and combined verification |
| `sase-xe.16.11.5`        | Real dispatch receipt, fresh visibility, output/stop and recovery     | both live phases                                 |
| `sase-xe.16.10`          | Completion of the original Apollo end-to-end proof                    | released-runtime and both live phases            |

The six open plan ancestors are `sase-xe.16.11.7.14.6`, `.14`, `sase-xe.16.11.7`,
`sase-xe.16.11`, `sase-xe.16`, and `sase-xe`. Original phases `.16.10` and `.16.11.5`
were auto-closed by commits despite explicit unmet acceptance, then reopened. Their
reopenings are deliberate and must not be erased with another automatic close.

Read these canonical artifacts with `sase artifact read`:

- `plan:202609/fleet_remaining_acceptance.md`
- `plan:202609/fleet_stale_remote_rows.md`
- `file:explicit:e2c64521530457f9913aa9aa` (previous detailed fleet land audit)
- `plan:202609/unified_agents_across_machines.md`
- `research:202609/agents_across_machines/agents_across_machines.md`
- `plan:202609/remote_dispatch_landing_remaining.md`
- `plan:202609/remote_dispatch_completion.md`
- `plan:202609/remote_dispatch_fleet.md`

Preserve later superseding decisions: one list with machine attributes/filtering and
global attention replaces Focus/Fleet membership. Explicit history is now required by
the newer snapshot contract, despite older bounded-history wording. Mac remains best
effort. Watch, automatic placement/fallback, cross-machine scheduling, and always-on
attention while ACE is closed remain intentionally deferred. No user study results may
be inferred from scripted acceptance.

## Common implementation rules

- Read current AGENTS instructions and relevant memories through `/sase_memory_read`:
  beads, artifacts, xprompts, generated skills, lint/test, TUI performance, Symvision
  and flags. Read CLI rules if adding an option. Open every other repo with
  `/sase_repo`; artifact bodies still require audited artifact reads.
- Shared targeting, continuation policy, identity and environment-boundary rules live in
  Rust core with PyO3/wire coverage. Python supplies host execution and Textual
  presentation through thin adapters. Reuse existing gate/shell settlement and remote
  operation journals. The active `sase-zl` monitor-continuation epic owns its broader
  context/result framework: integrate its current contracts, do not redesign it here.
- No direct commits, branches or PRs. Use host finalizers. Land source changes before
  deployment or a handoff that would skip their finalizers. Release-plz owns versions.
- Ordinary repair does not need a new beta flag. Preserve compatibility at persistent
  wire boundaries deliberately; consult flag policy if a migration branch is needed. Do
  not edit memory notes or generated instruction shims under this plan.
- Long checks, release waits and continuation waits use `/sase_monitor` with concrete
  checkpoints and next actions. Every gate that interrupts remaining work must retain a
  durable continuation. Do not poll in a provider turn or promise to resume.
- Phase workers close only their assigned implementation phase after its acceptance
  passes; original phase and ancestor closures are land duties. The specifically
  assigned `sase-z6` flag retirement follows the flag lifecycle below. Discoveries are
  PROPOSED FOLLOW-UP notes; land agents triage with `/sase_new_task`. Preserve unrelated
  live, Unknown, waiting and pending-question work and all artifact data.

## launch-targets

Repair the approved execution path, not just its preview text.

In core `agent_launch/mod.rs` and `agent_launch/admission.rs`, preserve each Agent
unit's exact workspace provider/reference and remote dispatch intent through parsing,
serialization, content digest, approval preview and dispatch reconstruction. A plan-wide
selected project cannot replace a per-unit branch/Patch/project reference. Include
binding/wire adapters in SASE and PyO3. Keep literal fenced syntax inert, family
workspace inheritance intact, and multi-prompt per-unit contexts distinct.

Make approved admission execute remote units through the existing validated remote
launch service (`dispatch/launch.py` and its operation ledger), local units through the
local launcher, and reject unsupported combinations before spawning. Avoid a recursive
CLI call that creates a second approval gate. Preserve source revision, attachments,
published/dirty guards, target installation identity and configured authorization. There
must be no local or home fallback for a failed remote/project launch.

Carry actual remote receipt/locator, operation key and uncertainty into admission
results and durable recovery records. Repeated approval, crash recovery and a lost reply
must reconcile the same operation, never create a fresh remote execution. Preview and
result must make actual project and machine inspectable.

Resolve the additional explicit-name conflict recorded on `.16.11.7.13`: gateway launch
currently supplies an operation-derived name while the mobile bridge rejects a prompt
already containing an identity directive. Give identity one authoritative owner and
preserve supported explicit identities without duplicating the directive. Use existing
collision/refusal semantics; never force name reuse. Test automatic and explicit names,
remote receipts and stop locators together.

Regression coverage must drive the submitted incident-shaped prompt through request
planning, serialized approval/admission and dispatch. Assert the target invocation and
absence of a local spawn, not merely a string in the preview. Include mixed local/remote
units, distinct workspace refs, family attachment, source refusal, offline target,
duplicate settlement and uncertain receipt recovery. Add a real gateway/bridge boundary
test; mock only external process execution or transport where isolation requires it.

## requester-continuation

Define and validate a durable requester-continuation field for agent-origin launch
requests using the existing shell next/branch vocabulary. Ordinary helper launches
should resume the requester by default with its assignment, bead, workspace/family
identity, checkpoint and typed launch results. Allow an explicit terminal handoff when
that is the request's intent. A requested family successor and an automatic requester
successor must not race for the same sequential family lane; validate/select one owner
before handing off. Show the continuation disposition in the approval preview.

Handle approve, partial/uncertain dispatch, rejection with feedback, failure, timeout
and cancellation deliberately. A refusal may resume to report the blocker or revise
work, but must not silently re-request the refused launch. Stopping the family must not
unexpectedly resurrect it. Preserve exactly-once successor reservation, replay and claim
handling from the existing shell implementation. A missing/failed required continuation
must be visible as recoverable failure, not successful task completion. Do not
retroactively rewrite already hashed historical gate bundles.

Fix the inherited operation environment at the process boundary. Inspect the path from
the `agent.drain` proc through spawned runners, gate commands and successors.
`SASE_PROC_REQUEST_PATH`, `SASE_PROC_RESULT_PATH`, `SASE_PROC_OPERATION` and
`SASE_PROC_ID` belong to the owning operation, not arbitrary descendant agents. Scrub
ambient ownership at ordinary agent spawn; inject fresh operation context explicitly
only for an actual operation command. Preserve required agent identity, approval guards,
legitimate proc result writing and other supported launch metadata. Never solve this by
ignoring a mismatched explicit sidecar in `ops.cli.load_request`.

Tests must prove a drain-launched agent can issue a launch without reading or writing
its parent's sidecars, and the real proc still produces its own result. Cover helpers
and family handoffs, no double successor, repeated settlement, requester termination,
and continuation-launch failure. An end-to-end fake-provider run must pass through a
real gate settlement and perform a post-approval task step on the original bead.

Update `src/sase/xprompts/skills/sase_run.md` and launch documentation to teach the
explicit continuation/checkpoint, target preflight and batched test-launch recipe.
Preview generated skills while editing; deployment comes only after landing in the
released-runtime phase. This is a skill-source change, not a memory-note change.

## acceptance-regressions

Inventory existing source and closed-phase evidence before changing tests. The real TLS
success fix already exists in core:
`worker_trusts_pinned_ca_and_preserves_healthy_host_beside_faults` requires an
authenticated successful host beside a genuinely connected hung peer. Reuse it, verify
its released ancestry and rerun it; do not rebuild the old TLS work.

The current `fleet_mutate_refuses_stale_revision_and_superseded_instance` still
fabricates `other-run`. Add the missing sequence: capture a real old locator, replace
the fixture's actual execution under the same logical identity, refresh owner state,
submit the captured locator, and prove refusal with zero effects on the replacement.
Then prove the replacement's real locator works. Retain stale-revision, bootstrap,
transport trust and successful-healthy-host/deadline checks. Fix production behavior if
the genuine sequence exposes a defect.

Reproduce the `.16.11.7.13` target-picker focus report using the current UI: opening a
custom launch from a filtered remote list and pressing Enter must submit the prompt, not
a bulk stop action on the underlying list. Cover focus restoration, explicit owner and
cancellation. Fix a reproduced defect with a narrow Textual regression; record concrete
passing evidence if intervening source already repaired it.

Complete the original unified cutover obligation. `sase-z6` is still an open live flag
bead for `ace_unified_agents` although its registry definition was removed. Inspect the
Off-branch removal, generated schema and current consumers, then retire the flag bead
normally under the flag workflow when the cutover contract is satisfied. This is caused
by the epic, not unrelated lint debt. Preserve other active flags, including the
separately owned completion recipe work; route unrelated failures to their actual owner
rather than suppressing the audit. Keep default keymaps, help and docs consistent.

## released-runtime

After the preceding fixes land through finalizers, adopt a published core release that
contains them. Use supported pin/floor/lockfile tools and preserve newer concurrent core
requirements. If a release is still running, monitor it and continue after the real
result; do not close with a promise to publish later. Do not edit release versions
manually or claim a dev build is a released wheel.

Restore the implementation checkout with the supported install workflow. Verify the new
wire/binding contracts and packaged gateway/worker in a clean published-wheel
environment without editable core overrides. Run full core checks including PyO3 and
record exact host/core commits, wheel version and release evidence.

Deploy the landed launch skill through `sase skill init` from a clean canonical
revision, following the generated-skill workflow and opening chezmoi first if needed.
Use the supported managed update workflow on Athena and Apollo. Inspect and refresh
gateway, federation worker, AXE and the acceptance ACE session as necessary; compare
running health/build identities with installed packages. Preserve enrollment and active
work, and do not run concurrent updates against one managed environment.

Recheck authenticated hello, machine status/discovery and dispatch doctor. Demonstrate
that a harmless approved helper actually targets the selected machine/project and that
its requester executes a subsequent step. This must exercise the deployed approval path,
not a direct helper API that bypasses the repaired contract.

## stale-row-proof

Use current `tailnet.md`, original `.6.6` tool evidence and the latest runtime
inventory. Reuse valid deployment/enrollment; do not blindly repeat broad GC. Record
read-only GC dry-run and counts first; apply supported reconciliation only if required,
retaining before/after, protected-row and idempotence evidence. Do not erase historical
artifacts.

Launch only bounded observation workloads through the repaired approval path, with a
checkpoint describing exactly what the continuation must verify. Use the xsmall size
alias and at most three proof agents per live session, reusing them across compatible
scenarios. Confirm receipt and target-local identity before touching a workload. Use
actual ACE panes as well as owner and viewer payloads; a programmatic projection alone
is insufficient.

- Compare Apollo's actual owner-local presentable set with Athena's remote set and
  authoritative lifecycle counts. Include bounded recent completions, families, Waiting,
  Unknown, observation age and disconnected last-observed state.
- Dismiss a controlled fresh terminal agent on Apollo through the production cleanup
  path; prove disappearance on Athena after refresh. Include unloaded family members in
  the regression evidence and preserve unrelated live/protected work.
- Observe one controlled running agent, terminate only that workload, and prove honest
  last-observed status followed by owner reconciliation. Never infer death from mere
  network disconnection.
- Page explicit older history beyond the default tier and force a snapshot change during
  continuation. Prove cursor reset, no stale-page resurrection, no duplicates and counts
  independent of the number of loaded pages.
- Restart the managed Apollo gateway and refresh the viewer; prove removed rows remain
  removed, enrollment identity survives and the same ACE session recovers.

Publish redacted pane/payload/command evidence with timestamps, identities, build
versions and exact assertions. Attach the report to this phase and cite it on original
`.6.6`; leave original phase closure to the serialized landing recovery below.

## unified-live-proof

Execute the validation table from the unified research and plan on the shipped default.
Reuse the preceding session and small test cohort where possible. Complete and record:

- An unfollowed, never-cataloged remote test agent requests attention while ACE is on
  Artifacts with a Here-only query. Discover and answer it through the durable global
  inbox without changing list scope. Exercise a controlled gate resolved by another
  controller while its modal is open; stale decisions must explain the conflict and have
  no effects. Use only the proof's own harmless gates/questions.
- Project and machine queries compose; navigation restores on return. Same names on
  different origins remain distinguishable, and a temporary machine-alias rename
  preserves selection and correct-owner actions. Restore the user's alias afterward.
- Target-picker launches show machine and source context. A dirty source or unsupported
  local attachment refuses before submission and preserves the draft. The repaired
  explicit-name path works or exposes its documented preflight refusal, never a local
  fallback or delayed bridge surprise.
- A fresh Apollo agent becomes visible, its output is nonempty, and Athena performs an
  exact-instance stop independently confirmed on Apollo. Preserve other agents.
- Lose a launch/stop response, restart ACE, and reconcile with the same operation key.
  Prove no duplicate execution and no false success. The isolated transport harness may
  inject loss; the session must still use production reconciliation.
- Healthy work remains usable beside a hung host during navigation/paging. Gateway
  restart yields honest/coalesced diagnostics and recovery. Machines/Connect offers
  persistent first-enrollment and partial-repair guidance; reuse production enrollment
  and use isolated fixtures for destructive setup cases.

Capture narrow 82x28 and wider panes, monochrome meaning, family handoffs and later
pages. Run existing fleet visual coverage and fault benchmarks with faults overlapping
navigation; retain p95 below 16 ms, no pump stalls, quiet idle ticks and zero-host
laziness. State fixture evidence versus live evidence explicitly.

Run SASE `just check-full` through `/sase_monitor` with TESTING/TESTED on the combined
tree, full core checks including PyO3, and relevant visual/fault checks. Record exact
commits and outcomes, including any blocked checks. Do not baseline deterministic
contract skew as a flake or substitute old passing checks from another tree.

Publish one requirement-to-evidence matrix covering all five open original phases,
including existing verified work reused from `.16.11.7.3` and `.11`. Cite this on the
original phase beads. Only fixes actually revealed by these acceptance contracts are in
scope; complete them and reverify the affected evidence before calling this done.

## Serialized landing and recovery through sase-xe

These are land-agent duties, outside the implementation phases. The current snapshot
land agent and unified land agent were WAITING at investigation time; names and PIDs are
observations, not durable recovery targets. Refresh agent, gate, monitor and bead state
first. Choose exactly one owner for each landing. Preserve original raw prompts,
bead/clan/workspace/finalizer metadata and evidence before replacing any abandoned run.
Do not launch duplicate landers or force-reuse names. Never kill an active worker based
only on elapsed time.

Use the normal `bd/land_epic` workflow. After this child lands, its `parent_bead`
permits resuming the interrupted `.16.11.7.14.6` landing. The land owner must consume
the new evidence and finish original `.6.6`, not wait forever for the already-ended
launch gate to perform work. If an existing waiting lander owns the next step, hand it
the durable evidence through the recorded bead/plan context and let that owner continue.
If a replacement original phase worker is needed, launch its preserved assignment with a
fresh retry identity and the same bead through the repaired approval flow; give the
requester an explicit continuation. Reconcile old-name agent waits as well as bead
waits.

Before any close that wakes another lander, establish who owns the containing epic:
either yield to its existing healthy owner, or deliberately retire a confirmed dormant
duplicate and let the recovery lander take over. Use supported lifecycle commands and
record this decision. No lifecycle step may depend on the current provider remaining
alive after a launch, gate or monitor handoff.

Do not blanket-rerun `sase bead work sase-xe`: it would schedule original workers before
the repair is deployed and can race existing waiters. Preview any epic recovery first;
reuse live assignments and recover only abandoned ones. In particular, the new child
lander's parent traversal must not assume a closed child automatically closes an
original sibling phase. Explicitly verify/complete that phase with its proper owner,
then allow the containing lander's bead wait to settle.

The required order, allowing already-completed steps to be verified and skipped, is:

1. Finish `.16.11.7.14.6.6`, then land `.16.11.7.14.6` and `.16.11.7.14`.
2. Finish `.16.11.7.13` using the unified live evidence; the unified land owner verifies
   all descendants and every note. Its original plan explicitly assigns normal close of
   reopened `.16.11.3` and `.16.11.5` once their missing evidence is supplied.
3. Land `.16.11.7`, then `.16.11`; verify and close reopened `.16.10` with the full
   original proof before landing `.16`.
4. Audit and land root `sase-xe`, including every original child note, all prior landing
   notes, canceled/superseded branches, linked plans and intervening changes in both
   repositories. Preserve the historical follow-up dispositions; do not reactivate
   intentionally deferred features or demand fresh work for already repaired issues.

For each ancestor, compare actual code and landed commits with its requirements, review
post-child drift and the evidence matrix, and require all descendants complete. Resolve
or correctly re-key epic-symbol exemptions before normal close; run post-close
`just symvision` and set that plan's `status: done` through the normal landing flow.
Read and modify plan sidecars through the sanctioned repo/artifact workflow. No force
close, canceled resolution or automatic commit close may substitute for acceptance.

Land agents must account for every PROPOSED FOLLOW-UP, particularly explicit remote
identity, target-picker focus, cutover flag drift, TLS/fencing, old count/history
claims, loader/build skew and known unrelated flakes. Reuse existing tasks/epic owners
through `/sase_new_task` when necessary. An epic-caused or acceptance-blocking defect
remains work here, regardless of a previous note calling it a follow-up.

If separate land owners must run later, persist a monitor continuation that checks their
actual results and repairs an abandoned handoff; a launch receipt alone is not the
completion criterion. Stop closing at the first unmet requirement, preserve a specific
blocker and continue its repair within this plan's scope. Finish only after a fresh
store read proves root `sase-xe` and all descendants closed with appropriate
resolutions, all linked epic plans done, required checks passed, and no acceptance or
landing assignment has been abandoned. The final report cites repairs, release
identities, live evidence, verification and the root close.
