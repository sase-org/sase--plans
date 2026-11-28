---
tier: epic
title: Host-owned recovery for zombie epics
goal: Report stranded completed-child plans and relaunch eligible epic land agents
  once per new close event.
parent_bead: sase-1vn.4
phases:
- id: detection
  title: Shared Rust zombie-plan detection
  size: medium
  depends_on: []
  description: 'detection: implement the complete descendant snapshot, shared ownership-aware
    predicate, close-event identity, and Rust/PyO3 contracts described in Phase detection.
    Verify the Rust changes and leave automatic launches disabled.'
- id: reporting
  title: Read-only zombie report and TUI integration
  size: medium
  depends_on:
  - detection
  description: 'reporting: consume the shared predicate in a read-only bead zombies
    command and existing stranded-work TUI surfaces, advance the core pin, and capture
    the current zombie inventory without launching or closing anything.'
- id: recovery-claims
  title: Durable activation and recovery claims
  size: medium
  depends_on:
  - reporting
  description: 'recovery-claims: implement core-owned activation and recovery records
    and the machine publication adapter, including concurrent publisher and restart
    tests. Expose a verified claim to callers but do not submit any launch.'
- id: recovery-submission
  title: Idempotent land-only launch submission
  size: medium
  depends_on:
  - recovery-claims
  description: 'recovery-submission: consume published claims through the existing
    bead-work launch path with deterministic proc identity, a close-token guard, and
    crash replay tests proving that recovery never relaunches phase workers. Do not
    enable the periodic job yet.'
- id: scheduler
  title: Bounded AXE recovery job and end-to-end verification
  size: medium
  depends_on:
  - recovery-submission
  description: 'scheduler: wire the shared report and durable submission into a bounded
    AXE job, arm the rollout cutoff without launching historical zombies, add restart
    and race regressions, and document the shipped behavior and report evidence.'
proposed_by: bbugyi200.athena.sase-1vn.4
proposed_with: codex/gpt-6-astra
create_time: 2026-11-27 20:17:38
status: wip
bead_id: sase-1vn.4.1
---

- **PROMPT:** [prompts/202611/zombie_epic_liveness.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/zombie_epic_liveness.md)
- **PARENT:** [202611/backlog_epic_1.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_1.md)
- **BEAD:** [sase-1vn.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vn/sase-1vn.4.1.md)

# Host-owned recovery for zombie epics

## Scope and authority

This child epic implements phase `sase-1vn.4` of `plan:202611/backlog_epic_1.md`,
addressing the defect recorded in `sase-12u`. The parent reviewer selected
**zombie_recovery = relaunch**. That decision is final: report existing zombies, and
recover eligible future cases by relaunching the land agent. Never automatically close a
plan bead. No memory changes are authorized or needed. The parent's test-cost decision
is outside this child epic.

The assigned phase had no prior-work notes when inspected on 2026-11-27. The inspected
sase tree was `c783119ffd`; the linked sase-core tree was `34461b7f`. The preceding
failed-turn handoff phase has landed. Its inactive-handoff ownership logic must remain
in effect. This is an epic because the new shared data contract, reporting consumers,
durable claim publication, guarded launch submission, and scheduled integration each
need a bounded coding phase. All phases are sequential, including their host-owned
commits and pin updates.

Governing decisions:

- `decisions:rust-core-required`: predicates, event identity, and durable recovery
  eligibility belong in sase-core; Python supplies host I/O and thin adapters.
- `decisions:host-owned-completion` and
  `decisions:handoffs-end-through-host-completion`: host completion and the existing
  launch path retain ownership of finalization, claims, and successor launch.
- `decisions:single-turn-agents`: continuation is mechanical, never an LLM promise.
- `decisions:check-full-is-explicit`: use `sase tool run check`, not `check-full`.
- `decisions:machine-link-writes-off-primary` governs any artifact-link writes caused by
  the new machinery; use the existing machine write and operational lease paths.

Workers open sase-core with `/sase_repo`, read its AGENTS.md, and use only the printed
checkout. Read `cli_rules.md`, `ace.md`, `tui.md` and the applicable TUI child notes,
`docs_writing.md`, and `lint_and_test.md` before work in those domains. Memory reads go
through `/sase_memory_read`. Record discoveries as `PROPOSED FOLLOW-UP:` notes on the
worker's phase; do not create task beads or close ancestors.

## Current implementation and reuse points

- `crates/sase_core/src/bead/mutation/close_remove.rs` deliberately cascades delegated
  plan completion only to a phase parent. Preserve that behavior.
- `crates/sase_core/src/bead/read.rs::stranded_snapshot` provides candidates plus child
  plans of candidate phases. It does **not** provide every descendant of a plan, so an
  empty context cannot prove that an epic's descendants are closed.
- `crates/sase_core/src/stranded_work/classify.rs` already decides ownership from
  session members, finalization, monitors, gates, recovery, and staged-handoff facts.
  Reuse that policy, including the new inactive-stage checks; do not substitute a PID
  check or an agent-name suffix heuristic.
- `IssueWire` already carries `closed_at` and `close_event_id`; use event identity, not
  a plan's `updated_at`, to deduplicate recovery after a reopen/reclose.
- `src/sase/bead/stranded_facts.py`, `_stranded_facts_beads.py`, and the shared stranded
  report provide the Python collection seam. ACE caches these reports off its UI thread.
  Preserve bounded refresh and project-scoped identity.
- `src/sase/bead/cli_work_handler_launch.py::launch_epic_bead_work` is the common CLI
  and host entry point. `tests/test_bead/test_cli_work_epic_relaunch.py` already proves
  that a retry with all phases closed launches only the land agent.
- `src/sase/bead/epic_launch.py` submits approved plan-file launches with operational
  leases. Its deduplication only covers active procs and plan paths, so it cannot serve
  as the durable per-close-event claim unchanged.
- `ProcSubmitRequest` has explicit `proc_id`, `request_fingerprint`, and
  `concurrency_keys`; `procs/submission.py` returns a replayed reservation without
  spawning again. Reuse this machinery, accounting for a crash before startup.
- `src/sase/axe/staged_handoff_reconcile.py` demonstrates bounded reconciliation;
  `src/sase/scripts/sase_chop_bead_claim_checks.py` demonstrates leased, refreshed bead
  mutations. These are patterns, not invitations to change their unrelated policies.

## Shared contract

### Detection

A confirmed zombie is a plan bead whose status is `in_progress`, which has at least one
descendant, whose **entire transitive descendant tree** is closed, and whose land
session has no current owner. Return plan ID, title, tier, assignee/session identity,
latest descendant close timestamp, corresponding descendant ID and close-event ID, and
any unknown-evidence reason. Use `parent_id` edges, never ID prefixes. Check closed
direct children with open grandchildren as well as nested child epics.

Ownership includes live agents, queued/waiting work, finalizers, active monitors or
gates, adopted/in-flight handoffs, recovery, and matching launch procs. A dead failed
origin's inactive held stage does not confer ownership. Unavailable or malformed
liveness/tree evidence is unknown and cannot authorize recovery. A successful complete
scan finding no records is distinguishable from a scan failure. Remote ownership is
conservative; never infer remote death from local PID absence.

Use a stable close token derived from the selected current descendant close event; sort
timestamp ties deterministically by event ID and bead ID. Missing legacy event identity
or timestamp may produce a report row marked ineligible, but never a synthetic automatic
recovery token. Do not mistake later notes or synchronization for a new close. Report
non-epic plan tiers too, but automatically relaunch only plans that the existing
epic-work builder supports; explain unsupported rows in the report.

### Rollout and once-per-event recovery

Persist a versioned project activation record when the new job first successfully arms.
Its timestamp is the operational rollout cutoff. The arming pass launches nothing; all
close events at or before that timestamp remain report-only. Reporting must not create
activation or recovery records. Never reset the cutoff on scheduler restart, config
reload, or code upgrade. A corrupt/unreadable record disables recovery with a diagnostic
instead of silently rearming.

The durable claim key is `(project identity, epic ID, descendant close-event token)`.
Persist the activation and claims in a small core-owned recovery namespace in the
canonical beads sidecar, separate from issue lifecycle status. A claim contains its
owner machine, stable proc ID/request fingerprint, close token, and activation
generation. Use create-if-absent semantics under the bead-store lock and publish before
launch; competing sidecar writers must resolve an existing claim by reading the
published winner, never by replacing it or keeping both payloads. The losing host does
not launch. Do not assume a local flock coordinates multiple machines.

Crash replay may resume the same reserved request on its owning host. It must not mint a
new proc identity or retry a consumed event after a land attempt terminates. If a
claim's owner disappears or startup is ambiguous, retain a visible diagnostic and the
manual `sase bead work <epic-id>` recovery command rather than transfer the claim and
risk a second launch. A confirmed pre-reservation failure may resume the same claim; an
acknowledged attempt remains consumed even if the land agent later fails.

Immediately before submission, refresh and recheck the same core predicate, cutoff,
close token, and ownership. Add a recovery-only guard at the normal bead-work launch
boundary so a child reopened after scheduling aborts the recovery instead of launching
phase workers. Preserve normal manual bead-work behavior and name/claim preflight.
Automatic recovery must never kill or replace a live owner, even if one appeared after
the first scan. Use the existing safe cleanup behavior for confirmed terminal owners.

## Phase detection

1. Add focused Rust bead-domain request/response types for the complete slim plan tree
   snapshot and zombie evaluation. Reuse the stranded member facts and ownership reducer
   rather than duplicating their rules. Make tree completeness explicit.
2. Implement the same result in indexed and replay/legacy read paths. Batch tree and
   ownership facts per project; do not repeatedly replay the entire store per epic.
   Missing parents, cycles, or incomplete reads must not authorize recovery.
3. Export and register PyO3 bindings in the existing bead/stranded domains, with
   round-trip tests. Keep old callers compatible or update all copied schema versions
   and fixtures together where a wire change requires it. Return enough evidence for
   CLI, TUI, and AXE to consume exactly one predicate.
4. Test a closed descendant tree, zero children, an open child, a closed child with an
   open grandchild, nested plans, live and finalizing landers, active continuations,
   inactive failed-origin stages, remote/unknown ownership, legacy timestamps, equal
   timestamp ties, and reopen/reclose event identity.
5. Run `sase tool run check` in sase-core. The next phase consumes only the committed
   binding revision. Do not enable a job or change close/cascade behavior in this phase.

## Phase reporting

1. Add a thin Python facade and collector for the shared Rust contract. Reuse one
   indexed agent/session snapshot and project tree snapshot. Read failures remain
   distinguishable from confirmed absence.
2. Add `sase bead zombies`, defaulting to the current project, with `-a/--all-projects`
   for enabled projects and `-f/--format compact|json`. Follow sorted CLI help and
   completion conventions. Show last descendant close time, ownership confidence,
   recovery eligibility/reason, and the land command. Include the report timestamp and
   project identity. Empty output is explicit; partial failures must be reported.
3. Feed the same predicate into existing stranded-work reports so the TUI recognizes a
   completed-descendant epic with no owner. Use existing badge/detail presentation where
   possible; add the missing close-time explanation without a new navigation mode.
   Preserve other stranded cause precedence and cached off-thread reads. The new report
   command and TUI refresh never arm, claim, launch, close, or reopen beads.
4. Add CLI tests for text/JSON, project scope, unknown evidence, and no mutation or
   launch calls; add a TUI integration regression proving it receives the same verdict.
   Update affected screenshots only when rendered output changes.
5. Advance `sase-core-revision.txt` to include phase detection. Follow
   `docs/contribute/rust-backend.md`: the host updates the pin for coordinated dirty
   repos; use `just ratchet-core-revision` only when consuming an already-landed core
   commit. Do not hand-bump release versions. Verify in the workspace-local environment.
6. Run the read-only report on the current live store. Record its observation time,
   counts, rows and unknowns as a report artifact linked from a note on `sase-1vn.4`.
   Re-check the examples `sase-zu` and `sase-zn` rather than assuming they remain
   zombies. Read live beads through audited commands. Run `sase tool run check` in sase.

## Phase recovery-claims

1. Implement the activation/claim contract in the Rust bead domain, with atomic local
   writes, validation, deterministic keys, read-only inspection, and binding coverage.
   Keep claim state separate from issue status and assignee changes.
2. Add the Python host adapter for machine-owned store access, refresh, claim
   publication, and published-winner confirmation. Integrate with the existing conflict
   replay path so competing claims cannot both become launch permission. A failed or
   uncertain publish never starts a process. Test competing publishers against fixture
   remotes; do not experiment on live beads.
3. Test repeated claim reads, cutoff preservation across restarts and upgrades,
   malformed records, missing legacy event tokens, two projects with the same bead
   suffix, concurrent local callers, competing publisher checkouts, interrupted
   publication, and unconfirmed push outcomes. Assert that only the published winner
   receives a launch permit; no code in this phase submits a proc or agent.
4. Add the Python facade and pin coordination needed to consume these bindings. Run
   `sase tool run check` in both repos. Keep automatic recovery disabled.

## Phase recovery-submission

1. Consume the published claim API without changing its identity or cutoff rules. Submit
   the stable recovery proc through an operational lease with origin `axe`. Its
   execution reaches `launch_epic_bead_work` for the existing epic ID, preserving the
   reviewed plan, model, Patch/VCS context, queue capacity, naming, and preclaims. Do
   not pass a bead ID to a helper that treats every argument as a plan-file path. Use a
   recovery-only expected-close-token guard so reopened descendants or a new owner abort
   before cleanup or agent spawn.
2. Cover repeated calls after a completed/failed proc, concurrent local submitters, two
   project identities with the same bead suffix, cross-host claim competition, lost
   acknowledgement, crash before submission, crash after reservation, restart after
   acknowledgement, failed publish, corrupt claim, and a child reopened between scan and
   execution. Verify at most one land launch for the token and no phase launch.
3. The shared predicate owns eligibility; Python must not recreate its traversal or
   liveness rules. No periodic caller is enabled in this phase. Run
   `sase tool run check` in sase and in sase-core if that phase changes core,
   coordinating any binding pin through finalizers.

## Phase scheduler

1. Add `epic_liveness` as a built-in AXE job with the normal job entry point,
   registration, result summary, tests, and `src/sase/default_config.yml` configuration.
   Place it in the existing waits routine with `run_every: "60s"`. Give one pass a
   30-second work budget, at most one new launch submission globally, and a persistent
   fair project cursor so a large project cannot starve the next one. The proc performs
   the slow launch work outside the job. Reuse indexed evidence and bounded batches.
2. Arm uninitialized projects without launching. Later passes consider only complete,
   known zombie evidence whose close token is newer than activation. Require the normal
   120-second settlement grace after the latest child close or terminal owner evidence.
   Use the durable adapter from recovery-submission; never direct-spawn an agent.
3. Preserve candidate progress across restart, isolate a damaged project's errors, and
   report counters for historical, owned, unknown, grace, consumed, deferred, submitted,
   and failed candidates. Consumed failures remain diagnosable through the report and
   existing proc evidence. Do not automatically relaunch historical zombies.
4. Add fixture end-to-end tests: arm, close last child, wait through grace using a fake
   clock, submit exactly one lander, tick again, restart, and prove no second
   submission. Include pre-activation and equal-cutoff closes, a pre-activation parent
   whose child closes after activation, a later reopen/reclose, unavailable ownership,
   live lander, reopened descendant, concurrency, invalid state, fairness, and budget
   exhaustion. Assert plan status remains in_progress until a land agent closes it.
5. Regenerate CLI/config docs using `just fix`; update the nearest bead/scheduler
   Reference page for the rollout and recovery behavior under `docs_writing.md`. No
   memory edits or guide expansion. Run a fresh read-only live report, link it as
   evidence on `sase-1vn.4`, and leave historical rows untouched.
6. Run `sase tool run check` in every changed repo, plus affected targeted visual
   verification if needed. Do not globally install or restart services as an implicit
   test; runtime behavior is covered with fixture stores and fake time.

## Landing and completion

Each worker runs `sase bead epic-symbols <its-phase-id>` before closing only its own
phase. Resolve unused entries or re-key justified consumers to a still-open phase or
this child epic. Use `/sase_final` for host-owned commits. Route long checks through
`/sase_monitor`, waiting for the staging command itself to exit before finalizing.

The child epic's land agent verifies that all five phases are integrated, the source pin
includes all new bindings, both repositories passed their required checks, the current
report is attached, and no historical zombie was launched or closed by the reporting
work. It collects the tested recovery invariants and any proposed follow-ups for the
outer land agent.

Before completion, check symbols for this child epic and `sase-1vn.4`. Close this child
epic using its ordinary land workflow, then inspect `sase-1vn.4`: the existing
delegated-plan-to-phase cascade may already have closed it. If it remains open, its
scope is complete, and symbols are clear, the child land agent is responsible for
`sase bead close sase-1vn.4 --note "<verification evidence>"`, as requested by the
delegating phase. No phase worker closes an ancestor. Do not close `sase-1vn`, any other
ancestor plan, or `sase-12u`; provide evidence for the outer land agent's triage.

A check failure that reproduces identically on the clean base is recorded as a
`PROPOSED FOLLOW-UP:` note with its evidence and existing task ID if known, and does not
leave a completed phase open. New regressions must be fixed before closing.
