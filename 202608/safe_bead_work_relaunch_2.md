---
tier: tale
title: Preserve active bead workers during safe relaunch
goal: "Re-running sase bead work replaces only eligible waiting or failed owners whose
  recorded bead association belongs to that invocation, while preserving active work and
  launching only the missing or safely replaceable segments.

  "
size: medium
proposed_by: bbugyi200.athena.02v
create_time: 2026-08-15 17:37:25
status: done
---

- **PROMPT:**
  [prompts/202608/safe_bead_work_relaunch_2.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/safe_bead_work_relaunch_2.md)

# Plan: Preserve active bead workers during safe relaunch

## Problem and safety contract

`sase bead work` renders deterministic phase, land, and task agent names with
force-reuse directives. The current cleanup preview treats every live owner as `KILL`,
and the preparation path wipes every planned name regardless of the owner's status. An
epic retry can therefore terminate a phase or land agent that is already running, then
launch another agent for the same bead. The standalone-task path has a separate
live-assignee shortcut, so its behavior is inconsistent and cannot distinguish a
genuinely running worker from one parked in `WAITING`.

Use the bead association already carried by each rendered `%id(..., bead=...)` directive
and persisted in `agent_meta.json`; do not introduce a separate launch ID. Treat the
rendered logical assignment map as the authorization boundary:

- Every planned phase name must map to its phase bead, the current land name must map to
  the epic bead, and the legacy bare epic land name is an alternate owner for that same
  land slot. A task's deterministic name and any recorded current assignee must map to
  the task bead.
- Normalize owner names with the existing current-owner identity rules, but require the
  exact recorded bead ID and a compatible bead-store assignee before destructive
  cleanup. A missing bead association, a different bead, multiple live owners for one
  logical slot, or an inconsistent in-progress assignee is a hard collision: report it
  and make no cleanup or bead mutation.
- A live process may be terminated only when its authoritative raw state is `WAITING`
  (including a runner-slot row presented as `QUEUED`) and the bead association matches.
  A matching `FAILED` owner may be removed and relaunched; matching completed,
  interrupted, dismissed, or stale reservations may still be removed/released because no
  live work is being terminated. Require the same bead proof for any real agent
  artifact, even when it is terminal.
- Preserve and omit from the retry every matching owner that is starting, running,
  retrying, executing an approved/working plan, asking a question, awaiting plan review,
  or otherwise live in a state outside the waiting/failed replacement set. Unknown or
  contradictory live state fails closed instead of being wiped.
- Resolve family containers as one logical bead owner. A running or input-blocked
  concrete member preserves the whole family; a family whose effective current member is
  matching waiting/failed work can be cleaned member-by-member. Keep the populated epic
  clan container joinable, while applying the same rules to its concrete current
  phase/land members and the legacy land member.

Represent this decision as an immutable launch selection containing the logical slot,
expected bead, concrete owner generation/artifact, observed state, cleanup action,
whether it is preserved, and whether a replacement segment will launch. Build the
selection from one scan-backed view of registry ownership and agent markers so preview,
status, family membership, and bead association do not come from unrelated snapshots.

## Guarded cleanup and retry selection

Replace the unconditional name-only contract in `src/sase/bead/cli_work_cleanup.py` with
assignment-aware preview and preparation helpers. Validate all candidate owners before
exposing or performing any destructive action. Render `PRESERVE`, `KILL`, `REMOVE`, and
`RELEASE` decisions with the expected bead ID, so dry runs and confirmation prompts
explain both why an owner is retained and why a cleanup is authorized.

After confirmation, rescan and revalidate every owner before cleanup. Revalidation may
safely shrink the launch set—for example, a previously waiting agent that has begun
running is preserved and its segment is dropped. It must never broaden the destructive
set after the user saw the preview; a newly occupied name, changed generation, changed
bead association, or newly destructive target aborts before any wipe and asks the user
to rerun. Pass the expected artifact generation, bead, and eligible state into the
command-specific wipe path and check them again at the destructive boundary, so a
waiting-to-running transition is refused rather than delegated to the generic
unconditional force-reuse wipe. Keep generic user-entered force reuse behavior
unchanged.

Perform this selection and guarded cleanup before plan snapshots, readiness/preclaim
mutations, graph checkpoints, or launches. If final revalidation leaves no segments to
launch, print an idempotent success message and skip every mutation/publication/launch
step. Preserve the existing cleanup confirmation semantics: `--yes` does not authorize
destructive cleanup, while `--yes-to-all` and JSON mode do.

## Partial epic and task launches

Teach the epic rendering helpers in `src/sase/bead/work.py` and the orchestration in
`src/sase/bead/cli_work_handler.py` / `src/sase/bead/cli_work_plan.py` to render a
selected subset of logical segments while retaining the full dependency graph:

- Omit preserved phase and land segments from the prompt, environment tuple, expected
  name set, launch count, and rollback scope. Keep waits on preserved agent names and
  phase beads so replacement downstream phases and the land agent still observe work
  already in flight.
- Recompute first-segment duties after filtering: the first launched segment receives
  the clan summary environment, prompt and environment ordering remain one-to-one, and
  retries join an existing clan instead of redeclaring it. Preserve the authored total
  phase count for land-model routing.
- Treat a matching live legacy bare-name land agent as the existing land slot. Preserve
  it and omit the new `.land` segment; clean a matching waiting/failed legacy owner
  before launching `.land`; reject simultaneous or mismatched legacy/current land
  ownership rather than creating a second land worker.
- Preclaim only selected phase assignments. Expose the Rust core's existing optional
  epic-agent argument through `src/sase/core/bead_mutation_facade.py` and
  `src/sase/bead/_project_mutations.py`, and pass a land assignee only when the land
  segment itself is launching. Thus preserved phase/land beads retain their status and
  assignee without redundant preclaim events, while zero-spawn rollback restores only
  mutations made for this reduced launch set.

Route standalone tasks in `src/sase/bead/cli_work_task.py` through the same classifier
instead of treating every live assignee as already running. A matching running or
input-blocked task owner remains an idempotent `already_running` result; a matching
waiting/failed deterministic owner (or recorded alternate assignee) is guardedly cleaned
and replaced; an unrelated same-name or assignee owner is refused. Keep the task prompt,
one-shot force-reuse bead authorization, preclaim, and rollback aligned to the final
replacement owner.

Return a structured epic launch outcome, analogous to `TaskWorkResult`, that
distinguishes `dry_run`, `declined`, `already_running`, and `launched` and records the
actual launched and preserved names. Thread it through bead-ID JSON output and the
plan-file resume/result/notification path so a fully preserved retry is reported as an
idempotent success rather than a declined launch, and partial retries never claim that
all planned agents were relaunched.

## Verification

Add focused unit and orchestration coverage beside the existing bead-work tests:

- Exercise the assignment-aware decision matrix for plain agents and families: matching
  `WAITING`/queued and `FAILED`, matching terminal/stale reservations,
  running/starting/retrying/question/plan-review states, unknown state, missing or
  mismatched bead IDs, mismatched bead assignees, and multiple live logical owners.
- Replace the existing regressions that expect live phase/land workers to be wiped.
  Verify a running phase is neither killed nor rendered/preclaimed again; a matching
  waiting phase and failed phase are replaced; downstream name/bead waits remain; a
  running land or legacy land is preserved; and mixed partial retries keep prompt,
  environment, expected-name, clan-summary, preclaim, launch-count, and rollback order
  aligned.
- Simulate the preview-to-cleanup race by changing a matching owner from waiting to
  running and assert final revalidation preserves it without calling the wipe helper.
  Also assert a newly destructive or newly mismatched owner aborts before any cleanup or
  bead mutation.
- Verify an all-running epic performs no snapshot, readiness update, preclaim,
  checkpoint/push, or agent launch and returns the truthful `already_running` outcome in
  human, JSON, plan-file resume, and completion-notification surfaces.
- Cover task parity: matching running owners remain idempotent, matching waiting/failed
  owners relaunch, alternate assignees are handled safely, and mismatched bead owners
  fail without cleanup.
- Add facade coverage proving an empty phase subset and `land_agent_name=None` pass
  through to the existing Rust binding without mutation; no `sase-core` source change is
  required.

Run `just install` first, then the focused bead-work, launch-rendering, result, and core
facade tests while iterating. Finish with `just check`; use `just check-full` through
the required monitor workflow only if the implementation touches the broadening set or
the scoped selector escalates.

Acceptance requires that no live non-waiting agent ever reaches the termination helper,
no owner lacking the exact launch bead association is cleaned, preserved work is not
reassigned or relaunched, partial launch metadata remains positionally correct, and a
normal fresh epic/task launch retains its current behavior.
