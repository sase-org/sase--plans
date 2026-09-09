---
tier: epic
title: Recover 0ak and make plan/finalizer provenance truthful
goal: "The approved monitor-kill lifecycle is complete on a released Rust core, commit
  finalization recognizes its own reconciliation commits, and every family surface shows
  the latest authoritative plan.

  "
phases:
  - id: core_monitor_cleanup
    title: Recover and publish the Rust monitor-cleanup contract
    depends_on: []
    size: medium
    description:
      "core_monitor_cleanup: recover the failed agent's schema-4 Rust cleanup change,
      reconcile it with current core, verify it, and publish the binding release."
  - id: python_monitor_integration
    title: Bind the committed Python cleanup path to the released core
    depends_on:
      - core_monitor_cleanup
    size: medium
    description:
      "python_monitor_integration: raise the core dependency floor, prove Rust/Python
      parity, and verify the committed monitor-stop lifecycle end to end."
  - id: finalizer_auto_commit_proof
    title: Preserve auto-commit proof across finalizer reconciliation
    depends_on: []
    size: small
    description:
      "finalizer_auto_commit_proof: retain the pre-reconciliation commit ledger so
      machine-owned commits prove clean transitions without weakening discard guards."
  - id: family_plan_preview_provenance
    title: Prefer the latest authoritative family plan everywhere
    depends_on: []
    size: small
    description:
      "family_plan_preview_provenance: make replacement plans win in ACE and editor
      previews while invalidating caches from bounded in-memory member state."
proposed_by: bbugyi200.athena.0av
status: done
bead_id: sase-s3
create_time: 2026-09-09 19:49:35
---

- **PROMPT:**
  [prompts/202608/0ak_failure_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/0ak_failure_recovery.md)
- **BEAD:**
  [sase-s3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s3/README.md)

# Plan: Recover 0ak and make plan/finalizer provenance truthful

## Outcome

Complete the approved `monitor_kill_lifecycle` implementation across both SASE and
`sase-core`, publish a binding whose cleanup wire matches Python schema 4, and raise the
Python dependency floor so production installs cannot silently remain on the schema-3
core. Fix the commit finalizer defect that falsely failed 0ak after it successfully
auto-committed an artifact-link index, and make family-level PLAN previews show the
latest replacement/accepted plan instead of the first rejected proposal. Preserve the
already-published main-repo commit; its inaccurate subject is historical metadata, not
an input to either failure, so do not rewrite shared `master` history.

## Confirmed diagnosis

- The screenshot's PLAN lane is wrong, but it did not cause the failure. The original
  family root retained `inspectable_monitor_indicator.md`, while the feedback planner,
  accepted code shell, and SDD commit all identify `monitor_kill_lifecycle.md`.
  `agent_family_preview_cache` resolves the root before concrete members and returns the
  first non-empty preview; its cache key contains only root association fields. The
  editor-helper family catalog uses the same root-first precedence. A rejected first
  proposal therefore wins indefinitely or until TTL expiry even after a replacement plan
  is accepted and implemented.
- Two verification monitors failed for unrelated state. `just check-full` first stopped
  on generated home-memory drift introduced by concurrent work. The retry reached the
  full test suite and reproduced the long-xdist-temp-path skills-rendering flake already
  tracked as `sase-rv`; the newer `master` fix `b05d2d5bf` makes those assertions
  wrap-aware. Neither failure implicates monitor cleanup behavior.
- The terminal `0ak--2` failure came from `builtin@commit`. Its accepted declaration
  included main, `sase-core`, and the plans-sidecar read index. Reconciliation
  successfully committed the plans index as `9b2c4855`, but `execute_commit_finalizer`
  captured `commit_results.json` only after reconciliation. The accepted plans
  obligation was then clean, the just-written marker looked old, and
  `_reject_discarded_dirty_work` falsely reported that the file vanished.
- Because finalization stopped at that false failure, the Rust cleanup change was never
  landed. The recoverable 0ak checkout contains the schema-4 planner/wire diff, while
  current `sase-core` is still schema 3. Main commit `c44138566` already expects Python
  schema 4 but still permits `sase-core-rs>=0.29.9`; today the facade falls back to the
  Python planner rather than exercising the intended shared Rust backend.
- The subject of `c44138566` describes the rejected plan. ACE does not derive the PLAN
  lane from Git subjects, and rewriting published `master` would add risk without
  repairing provenance. Correct plan links and new conventional commit subjects are the
  durable correction.

## Phase 1: Recover and publish the Rust monitor-cleanup contract

- **ID:** `core_monitor_cleanup`
- **Size:** `medium`
- **Depends on:** none

Use `/sase_repo` to open `sase-core`. Resolve the failed agent's recorded workspace
number from its stable metadata and use the audited repo workflow to inspect its dirty
linked-repo checkout; do not hard-code or search for an ephemeral workspace path.
Recover only the six-file schema-4 agent-cleanup diff attributable to 0ak, compare it to
the approved `monitor_kill_lifecycle` plan, and reapply/rebase it onto current
`sase-core` `master`. If the old checkout is unavailable, reimplement the same contract
from the plan and the Python schema-4 reference rather than weakening the boundary.

The Rust wire must expose `monitor_id`, `is_live_monitor`, a `monitor` kill kind, and
typed monitor-stop side effects; the planner must cascade only owned live monitor
descendants, deduplicate direct and cascaded selections, order stop intents before
ordinary cleanup, omit generic workspace release for monitor kills, retain terminal
monitor dismissal, and reject schema 3. Preserve parity exports and update the
unreleased changelog. Reconcile any intervening v0.29.12/v0.29.13 changes deliberately.

Run the focused Rust planner/wire/PyO3 parity suites and `just check` in `sase-core`.
Commit through the SASE finalizer workflow, land the core change, and publish the next
`sase-core-rs` release through the repository's normal release path. Record the exact
published version for the dependent Python phase; do not claim completion from an
editable build alone.

## Phase 2: Bind the committed Python cleanup path to the released core

- **ID:** `python_monitor_integration`
- **Size:** `medium`
- **Depends on:** `core_monitor_cleanup`

Review main commit `c44138566` against the approved monitor lifecycle plan and the
published Rust wire. Keep its already-committed Python/TUI/CLI behavior, correcting any
parity defects found during integration. Raise the `sase-core-rs` minimum in
`pyproject.toml` to the version published by Phase 1 and refresh `uv.lock`, so supported
installs cannot satisfy the package with schema 3 and silently take the compatibility
fallback.

Run `just install` to build against the current linked core, then exercise the focused
cleanup facade/parity, monitor-stop persistence, named-agent kill, TUI cleanup, and
owner-cleanup integration tests added by 0ak. Assert the installed binding reports wire
schema 4 and the Rust path handles live-monitor selection/cascade without falling back
to Python. Run `just check`; because this restores a cross-repository backend contract,
run `just check-full` only through `/sase_monitor` and inspect any failure rather than
attributing it to this change automatically.

## Phase 3: Preserve auto-commit proof across finalizer reconciliation

- **ID:** `finalizer_auto_commit_proof`
- **Size:** `small`
- **Depends on:** none

In the built-in commit executor, snapshot the run-owned commit-result ledger before
`prepare_commit_dirty_state` performs machine-owned reconciliation. Keep a separate
post-reconciliation snapshot for ordinary `sase stitch create` marker checks. When an
accepted dirty repository becomes clean because reconciliation auto-committed bead,
plan-status, Q&A, or artifact-link state, compare against the pre-reconciliation ledger
so the newly written marker proves the transition. Preserve the fail-closed cases: a
clean transition with no new attributable marker, an unchanged/stale marker, a marker
for a different checkout, or unpublished machine-owned state must still fail.

Add a focused reconciliation regression matching 0ak: submit a declaration while a
plans-sidecar artifact-link index is dirty, let preparation auto-commit it and update an
existing run/repository marker, then prove finalization succeeds without invoking a
manual stitch for that now-clean repo. Retain and extend the stale-marker/discarded-work
tests, and add an integration-level artifact-link case so marker timing cannot regress
behind mocks. Run the focused finalizer protocol, reconciliation, live-E2E, and
artifact-link auto-commit tests plus `just check`.

## Phase 4: Prefer the latest authoritative family plan everywhere

- **ID:** `family_plan_preview_provenance`
- **Size:** `small`
- **Depends on:** none

Define one explicit family-plan precedence contract: inspect concrete sequential shells
from newest to oldest and use the latest resolvable direct/archived/SDD plan; use the
aggregate root only as a compatibility fallback when no concrete member resolves. This
makes an accepted feedback-round plan beat the rejected first proposal while preserving
legacy one-shell families and bead fallbacks. Apply equivalent ordering in the ACE
family preview cache and the editor-helper catalog so completion labels and the SASE
CONTEXT PLAN lane cannot disagree.

Expand the in-memory ACE cache key with the ordered member association state (or an
equivalent cheap immutable token) so attaching a feedback/code continuation invalidates
the old preview immediately. Keep all filesystem and plan parsing in the existing
off-thread warm-up; render and keystroke paths may only traverse bounded in-memory
family state. Add regressions with an initial `inspectable_monitor_indicator` planner
and a newer accepted `monitor_kill_lifecycle` planner/code shell, covering immediate
cache invalidation, TUI PLAN rendering, editor catalog parity, root fallback, malformed
newer plans, and unchanged bead behavior. Run focused family-preview, prompt-panel,
editor helper, and TUI performance-contract tests plus `just check`.

## Landing and incident acceptance

The epic land step must integrate the four phases on current `master`, rerun
`just install`, `just check`, and the diff-selected monitor/finalizer/family-preview
tests. If the combined tree touches a broadening trigger or scoped selection escalates,
run `just check-full` through `/sase_monitor` with a follow-up that diagnoses failures.

Acceptance requires all of the following:

- A normal supported install loads a schema-4 `sase_core_rs` cleanup binding and uses
  Rust for the monitor kill plan.
- Killing an owning agent stops its live monitor through the canonical monitor service,
  suppresses follow-up, releases the claim exactly once, and does not stop unrelated
  monitors; starting a monitor still leaves it alive after the starter handoff.
- A machine-owned artifact-link auto-commit between declaration and commit execution is
  accepted as attributable proof, while genuine discarded dirty work remains fatal.
- A multi-round family such as 0ak renders `monitor_kill_lifecycle.md` in the SASE
  CONTEXT PLAN lane and exposes the same title in editor/catalog completion surfaces.
- The existing `c44138566` commit remains untouched. New stitches and the approved epic
  plan carry accurate monitor-cleanup/finalizer/plan-preview provenance.

## Non-goals

- Do not rewrite or force-push published `master` only to repair the old commit subject.
- Do not change the deliberate monitor handoff lifetime contract or stop monitors
  globally.
- Do not suppress the finalizer's discarded-work guard; repair the missing proof window.
- Do not fix the already-addressed skills-rendering flake or mutate generated home
  memory as part of this epic.
- Do not add synchronous repository, plan, or artifact reads to the TUI event loop.
