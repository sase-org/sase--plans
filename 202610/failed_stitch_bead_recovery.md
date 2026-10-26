---
tier: epic
title: Reopen beads whose completed work fails host finalization
goal: A failed host commit cannot leave this run's unlanded work marked done, and
  recovery evidence remains visible through beads, doctor, and ACE.
parent_bead: sase-1oh.4
phases:
- id: recovery-core
  title: Add atomic Rust recovery of closes owned by a failed run
  depends_on: []
  size: medium
  description: 'recovery-core: add close provenance, durable completion-failure facts,
    and an idempotent Rust mutation that conditionally reopens this run''s standing
    done closes with their ancestors and a STITCH FAILED note. Expose tested bindings;
    implement the failure-fact classification in the Rust stranded-work reducer.'
- id: host-recovery
  title: Connect terminal finalizer failures to durable bead recovery
  depends_on:
  - recovery-core
  size: medium
  description: 'host-recovery: pin the landed core API, carry close provenance through
    Python, and invoke recovery after terminal commit or declaration-recovery failure.
    Publish the bead mutation through the machine write lane, retain failure evidence,
    and cover success, partial landing, retries, and publication errors.'
- id: recovery-surfaces
  title: Verify reopened work blocks progress and appears in stranded surfaces
  depends_on:
  - recovery-core
  - host-recovery
  size: medium
  description: 'recovery-surfaces: feed reopened completion failures into the stranded-work
    collector, verify doctor and ACE plus epic dependency blocking with integration
    tests, and update the bead lifecycle reference. Record full acceptance evidence
    for the child land agent; do not close ancestor or source-task beads.'
proposed_by: bbugyi200.athena.sase-1oh.4
proposed_with: codex/gpt-6-astra
create_time: 2026-10-26 14:34:45
status: wip
bead_id: sase-1oh.4.1
---

- **PROMPT:** [prompts/202610/failed_stitch_bead_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/failed_stitch_bead_recovery.md)
- **PARENT:** [202610/backlog_epic_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_top_five_task_beads.md)
- **BEAD:** [sase-1oh.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1oh/sase-1oh.4.1.md)

# Reopen beads after failed host commits

## Scope and evidence

This is the implementation child epic for phase `sase-1oh.4` of
`plan:202610/backlog_epic_top_five_task_beads.md`. It repairs the defect reported in
`sase-yo`, including its merged duplicate's failed-declaration-recovery case. Read the
phase, source task, and parent plan through their audited commands before implementing.
The parent decisions `secret_scan = build` and `dormant_flakes = close` remain final;
neither changes this phase. No memory edits are authorized.

At the inspected bases, sase `60832648be` and sase-core `4121de03`:

- `finalizers/controller_run.py` records terminal failures but does not undo beads
  closed by the provider turn. Declaration recovery can fail before
  `execute_commit_finalizer` runs, so a stitch-subprocess-only hook misses a required
  trigger.
- `finalizers/commit_repair_stitch.py` already preserves commit-message paths and
  bounded operation stdout/stderr. Commit markers, checkpoints, and repository
  obligations already distinguish incomplete commits from successful work.
- Core `bead/events/wire.rs` records close actors, event IDs, and timestamps, but does
  not bind a close to a specific host run. The touch index is derived and can be stale;
  it is not mutation authority.
- Core `bead/mutation/close_remove.rs::open_issue` archives the previous close and
  reopens closed ancestors. Python `bead/stranded_facts.py` only selects `in_progress`
  candidates, so simply invoking ordinary open would make the repaired bead invisible to
  stranded-work surfaces.

There were no earlier implementation notes on `sase-1oh.4`. This split is warranted by
three distinct contracts: concurrent event-store mutation, finalizer evidence and
publication, and consumer integration. Each phase is a bounded medium implementation;
the land agent verifies their composition. No product files were changed while planning.

## Required behavior

A provider may close its beads during its turn. After terminal host finalization
failure, every still-current `done` close attributable to that run must be undone when
the run still has work that was not landed. Reopen through the canonical bead mutation
engine, preserve close history, reopen closed ancestors, and append one `STITCH FAILED:`
note per affected close episode.

The note must name the failed run, affected repository/workspace locations, preserved
commit-message path when available, failure code, and bounded tail of the hook output.
For declaration failure before a stitch exists, explicitly say that no stitch/message
file was created and record the declaration diagnostic and evidence artifact location.
Do not invent a hook failure for that case. Use existing output sanitization and bound
the tail; never include an entire transcript or secret value.

Recovery must not alter a bead closed before this run, a human's or other run's close,
`canceled`/`superseded` resolutions, a newer close episode, or a bead already reopened
or reclaimed by a successor. Cover all beads the run closed, including additional task
beads and delegated plan closes, rather than only `SASE_BEAD_ID`.

Success, clean/no-code completion, an authentic handoff, an active repair, and an
unrelated finalizer failure after all work has demonstrably landed must leave closes
alone. An intermediate failed attempt followed by successful retry must not reopen
anything. In a multi-repository run, one successful commit cannot prove the whole run
landed; outstanding work obligations still require recovery. Exclude unrelated,
pre-existing dirt using the existing host baselines and accepted obligations.

The recovery action never turns the failed finalizer into a success and never retries
the product commit. Existing commit/push repair stays responsible for that work.

## Shared implementation contract

1. **Provenance belongs on the close event.** Add optional host-run provenance to close
   writes and events, using the existing stable run/agent identity and artifact
   identity. Do not use the controller's newly generated UUID or the finalizer turn
   nonce as the sole identity: declaration recovery remints the nonce. Keep the original
   run identity available even when no declaration was accepted. Human and historical
   closes without this provenance remain readable and are never guessed into automatic
   recovery from an assignee or timestamp alone.
2. **Rust owns eligibility and the state transition.** Provide a typed request/result
   for discovering and conditionally recovering standing closes from the canonical event
   store. Check provenance, `done`, current status, and the exact close event under the
   mutation lock. Perform reopen, ancestor updates, close-history archival, structured
   failure facts, and note append in one mutation. Retrying the same run/close-event
   recovery must not duplicate notes or history, including after a partial publication
   failure. Preserve unrelated fields and assignees.
3. **Persist structured failure facts.** Store the recovered close-event ID, run
   identity, failure timestamp, and evidence locations with the bead's canonical
   event-derived state. Use additive optional wire fields and existing event operations
   where possible. The readable note is for humans; consumers must not infer state by
   searching its text. Preserve these facts through replay, cached reads, JSONL
   projections, and Python codecs. A new successful close or a new claim supersedes the
   active failure; historical evidence remains in notes/history.
4. **Reopened means `open`.** Reuse normal reopening/ancestor semantics; do not
   fabricate a running agent by writing `in_progress`. Extend stranded collection to
   include open beads carrying an active completion failure, and feed the failure's run
   facts when the bead had no assignee or a different historical assignee. Ordinary
   unassigned draft beads remain excluded.
5. **Use existing liveness precedence.** Rust classifies recovered open work as stranded
   once no live owner, recovery, handoff, delegation, or settle grace applies. Reuse the
   existing `failed` cause unless a distinct cause is necessary to state the behavior.
   Explicit persisted failure evidence remains useful if the old run's artifact index
   entry has aged out; an unavailable index must not erase this known failure.
   Unknown/live successor ownership must still be handled conservatively.
6. **The host supplies evidence, not lifecycle policy.** Gather exact repository
   commit/publication proofs through the existing finalizer ledger, checkpoint, and
   reconciliation code. Do not use exit code alone, clean git status alone, or any
   commit marker from another repository/attempt as proof. Pass normalized facts to core
   wherever a new shared eligibility rule is needed.

## Phase recovery-core

Open `sase-core` through `/sase_repo` and read its `AGENTS.md`. Work in the bead domain:
`crates/sase_core/src/bead/{wire.rs,events/,mutation/,read_model/}` and
`crates/sase_core_py/src/beads/`. Add a focused mutation module instead of enlarging the
existing close implementation with host-specific orchestration.

- Add backward-compatible close provenance and failure-fact types, defaults, and
  validation. Preserve the existing close API for callers that provide no provenance;
  append an optional binding argument or introduce a request wrapper without breaking
  existing positional calls. Stamp implicit delegated closes with the same origin.
- Implement the atomic recovery contract above using the established mutation view,
  replay fallback, event minting, read-model publication, and ancestor helpers. Return
  recovered, already-applied, and skipped close IDs with useful reasons.
- Extend the stranded-work wire/reducer for explicit failure facts and open recovery
  candidates. No Python classification fallback. Keep older wire inputs valid.
- Add and register PyO3 bindings with actual round-trip tests. Update schema mirrors
  only if a schema version truly needs to change; additive defaults are preferable.
- Test multiple closes, another actor/run, historical unproven closes, old closed-bead
  attachment, non-done resolution, newer close, ancestor history, duplicate recovery,
  concurrent reclose/reassignment, cache/replay parity, and active-marker clearing. Test
  liveness/grace/delegation precedence and ordinary open drafts in the reducer.

Run targeted core and binding tests with the repository's `just test` entry point and
`sase tool run check` in core. Never use bare cargo, change crate versions, or edit
changelogs. This phase lands the additive core API before its Python caller. Include the
actual binding names and core verification evidence in the phase note.

## Phase host-recovery

After core lands, ratchet sase's source pin with `just ratchet-core-revision`; exit 2
means a bump was applied. Verify the chosen revision contains the preceding phase and
does not regress a concurrently advanced pin. Read `docs/contribute/rust-backend.md` for
the local build and host-owned commit rules.

Implement thin model/codecs/facade wiring in `sase.core.bead_mutation_facade`,
`sase.bead.model`, and the existing issue codecs. Carry the host-run origin through
`BeadProject.close` and the CLI close path; cover other host close call sites, including
stitch bead actions. No user-facing CLI option or change to agent close instructions is
needed. Capture provenance before close without requiring `/sase_final` to have run.

Add a focused finalizer recovery coordinator and call it at the terminal controller
failure boundary, including failures thrown by declaration recovery before an executor
starts. `controller_run.py`, `controller_context.py`, `commit_execution.py`,
`commit_repair_stitch.py`, and the existing result/checkpoint helpers are the main
integration points. Inspect authentication failure exits outside the main controller try
block so they cannot silently bypass recovery where trustworthy run evidence exists.

Use the stable originating run for monitor/prepared completion too: a monitor has its
own artifacts and may mint a new nonce. Follow the prepared intent's source identity,
and preserve pending-recovery ownership semantics. Do not reopen work merely because
normal finalization is skipped for a valid handoff.

Persist the recovery request/outcome in durable run artifacts and the finalizer journal.
Apply and publish it through the existing machine-writable bead-store and repository
transaction APIs (`bead/background_store.py`, `bead_store_mutation`, and SDD
publication). Never mutate the user-owned primary snapshot or invoke the failed product
hook to commit the recovery note. Refresh the normal projections and schedule sidecar
convergence. Check actual publication outcomes; a false return is not success. If
recovery storage or publication also fails, retain the original finalizer error plus an
actionable recovery diagnostic/request in run artifacts for retry. Repeated invocation
must finish publication without duplicating the canonical recovery event. Do not discard
dirty work or reset its repository while acquiring a machine write context.

Start with two failing regressions using a real temporary bead store and finalizer
harness: close plus before-commit failure, and close plus declaration-recovery failure.
Assert bead state and persisted note, not just a mock invocation. Add success and retry
controls, multiple closes, mixed-repository partial landing, exact repository/attempt
proof matching, already-landed commit plus later hook/plugin failure, clean completion,
handoff/prepared completion, and failure of the recovery publication itself. Reuse
`tests/finalizers_protocol_harness_test_helpers.py` and
`tests/finalizers_live_e2e_test_helpers.py`; isolate all repos/remotes in temporary
paths.

Run the changed finalizer, declaration, bead lifecycle, and monitor tests plus
`sase tool run check` in sase. Re-run core's check only if this phase changes core.

## Phase recovery-surfaces

Wire structured failure facts into `bead/stranded_facts.py` and the Rust facade. Select
`in_progress` candidates as before plus `open` recovery candidates; load each relevant
run once. Keep broad event scans and blocking I/O out of the TUI render path. The
failure provenance supplies the run association without changing bead assignees.

Extend collector and integration tests so the real recovered bead feeds the shared
report. Verify:

- After terminal failure and settle grace, the bead is not closed and the report names
  it as stranded, with the parent blocked where applicable.
- `sase doctor` reports a WARNING and ACE's existing stranded flag receives that same
  verdict, including a source task that originally had no assignee.
- Epic/phase dependency checks see the reopened bead before they can release a waiting
  dependent or lander on the failed worker's terminal result. Exercise the actual
  waiter/release path, not just a list of closed IDs. If the current waiter only watches
  agent termination, make the narrow dependency guard necessary to honor this contract.
- Re-running the failed finalizer produces one note; a later claimed/live recovery owns
  the work; a new verified close removes the active stranded verdict.
- Ordinary draft/open, unrelated closed, canceled, and successful runs are unaffected.

Relevant tests include `tests/test_stranded_collector*.py`,
`tests/test_stranded_work_facade.py`, `tests/doctor/test_checks_beads.py`,
`tests/ace/tui/test_artifacts_beads_stranded.py`, and `tests/test_wait_epic_follow_*`.
Read `ace`, `tui`, and the appropriate TUI child memory before changing ACE code. Prefer
existing surface vocabulary; update and inspect focused PNG goldens only if visible
rendering changes.

Update the Stranded beads section of `docs/reference/work-units/bead-lifecycles.md` to
explain automatic reopening, the durable note, and the recovery action. Read
`docs_writing` and follow the reference-page budget; do not add bead IDs to
reader-facing prose or modify memory notes.

Run focused surface/integration regressions and `sase tool run check`. Record the
specific before-commit and declaration-failure test names for the lander's acceptance.

## Verification and completion rules

For every phase, read `lint_and_test` before finishing; run `sase tool run check` in
each changed repo. Use `/sase_monitor` for long commands and wait for the handoff
command itself to exit. Do not run `check-full`, global installs, manual commits, or
change timeouts/assertions to hide failures. Read `symvision` before addressing its
failures. Any temporary epic-symbol exemption must name a still-open owner; scan before
closing.

A failure reproduced identically on the clean base is recorded as `PROPOSED FOLLOW-UP:`
on the worker's assigned phase, citing an existing task when known; it does not keep an
otherwise completed phase open. Workers never create beads or close ancestors. Only the
host creates this child plan's phase graph on approval.

The child land agent must verify the integrated regression tests on the landed code, the
core pin, canonical replay of recovery events, publication retry idempotence, and the
stranded/doctor/ACE/dependency acceptance above. Run the required checks for any landing
changes and record the command results on `sase-1oh.4`.

Finish this child epic through its normal land flow. The delegated completion target is
**only `sase-1oh.4`**: before its close run `sase bead epic-symbols sase-1oh.4`,
resolve/re-key any entries, and use
`sase bead close sase-1oh.4 --note "<verified results>"` if delegation has not already
closed it. Do not close outer epic `sase-1oh` or its other ancestors. The current user
instruction also limits the source-task close: leave `sase-yo` for the outer epic's land
agent, and provide its verification evidence on `sase-1oh.4` instead of closing it from
a child phase. Preserve any original failure evidence and all unlanded work.
