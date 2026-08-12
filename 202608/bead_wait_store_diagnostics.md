---
tier: epic
title: Bead waits must never block silently on an unresolvable bead store
goal: "A bead-gated agent wait that cannot be decided — because the project's bead store
  is unresolvable, or because a waited-on bead is absent from a readable store — is
  reported as a distinct, attributable condition on the waiter, its chops, `sase
  doctor`, and ACE, instead of blocking forever behind an ordinary WAITING status.

  "
phases:
  - id: store
    title: One project bead-store resolver with an explicit availability result
    depends_on: []
    size: medium
    description: "store: collapse the three disagreeing project bead-store resolvers
      into one candidate-ordered lookup that returns an explicit unavailable result with
      the paths it tried, and stop the cwd walk-up from binding a project to an
      unrelated store.

      "
  - id: signal
    title: Structured blocked reasons through the wait barrier
    depends_on:
      - store
    size: medium
    description: "signal: give every blocked bead dependency a reason
      (store_unavailable, absent_from_store, not_closed), persist it on the waiting
      marker through the sase-core waiting wire, and make the wait_checks and
      bead_store_refresh chops report the undecidable cases instead of counting them as
      ordinary no-ops.

      "
  - id: surface
    title: Doctor check, notification, and waiter rendering
    depends_on:
      - signal
    size: medium
    description:
      "surface: add a doctor check for live waiters whose bead dependencies cannot be
      decided, notify once per waiter past a dwell threshold, and render the reason in
      the ACE wait section and `sase agent list`."
proposed_by: bbugyi200.athena.y4
create_time: 2026-08-12 07:29:03
status: wip
---

# Plan: Bead waits must never block silently on an unresolvable bead store

## Background: what actually happened

Two epic phase workers for the `sase-js` epic have been in `WAITING` for hours with
their dependencies already satisfied:

- `sase-js.6` declares `%w(bead=sase-js.4)` and `%w(bead=sase-js.5)`.
- `sase-js.7` declares the same two bead waits.
- Both `sase-js.4` and `sase-js.5` are `closed` in the bead store that owns the
  `sase-js` epic. So are `sase-js.1`, `sase-js.2`, `sase-js.3`, and `sase-js.8`.
- `sase-js.9` and `sase-js.land` are queued behind `sase-js.6` / `sase-js.7`, so the
  whole epic tail is stalled.

The waiters are not blocked because a dependency is unfinished. They are blocked because
the wait barrier could not decide the question at all, and the code treats "I cannot
decide" exactly like "not yet".

### The failing chain

1. `dependency_resolution_status` in
   `src/sase/core/wait_dependency_resolution/_resolution.py` takes `closed_bead_ids`.
   When that argument is `None` it appends _every_ wait bead to `blocked_on`:

   ```python
   if wait_bead_items and closed_bead_ids is None:
       for bead_id in wait_bead_items:
           if isinstance(bead_id, str) and bead_id:
               _append_blocked_dependency(blocked_on, bead_id)
   ```

   The resulting `WaitDependencyStatus("waiting", ...)` is indistinguishable from a
   genuine pending dependency. This fail-closed behavior arrived with the original bead
   wait feature and is correct as a _safety_ property — it must not change. What is
   missing is any way to tell the two situations apart.

2. `closed_bead_ids_for_project` in `src/sase/bead/store_locator.py` returns `None`
   whenever `canonical_beads_dir_for_project` yields no directory. Both the
   `wait_checks` chop and the runner's own fallback funnel through it, so both reach the
   same verdict.

3. `canonical_beads_dir_for_project` delegates to `_canonical_project_beads_dir` in
   `src/sase/bead/workspace.py`, which resolves the project's primary workspace and then
   asks `resolve_sdd_store(primary, 1)`. For the affected project that call reports
   in-tree storage, because `_resolve_sdd_storage` in
   `src/sase/sdd/_store_resolution.py` finds no materialized `sdd-store.json` record
   under the primary workspace and silently falls back to the VCS provider's default
   storage policy. The in-tree branch then checks `<primary>/sdd/beads`, which does not
   exist, and returns `None`. The project's real bead store is untouched and healthy;
   nothing in the resolution path says so.

### Why nobody was told

Every layer that could have surfaced this reported success or benign inactivity:

- The `bead_store_refresh` chop ends with `reason = "no_canonical_stores"` and
  `status=no_op`, even while it knows `projects_waiting=1`. A project with live bead
  waiters and no resolvable store is not a no-op; it is the exact failure being
  described here.
- The `wait_checks` chop reports `reason=dependencies_not_ready` and logs nothing about
  bead dependencies. Its `_terminal_blockers` helper only inspects agent-name and
  artifact-identity dependencies; `wait_for_beads` entries never produce a log line, an
  evidence row, or an `unknown_outcome` counter bump.
- `refresh_bead_wait_store` in `src/sase/axe/run_agent_wait_deps.py` — the runner's
  ten-minute self-heal — returns silently when `canonical_beads_dir_for_project` is
  `None`, so the healing path is disabled by exactly the condition it should heal.
- ACE's wait section enriches bead targets through `bead_statuses_for_project`, which
  also collapses "store unavailable" and "bead absent from store" into `None`. The panel
  renders the waiter as ordinarily waiting.
- `sase doctor` has no check that connects live waiters to bead-store reachability.

### The disagreement worth fixing

`src/sase/doctor/checks_beads.py` already resolves a project's bead store far more
generously than the wait barrier does. Its `_candidate_beads_dirs` tries
`resolve_sdd_kind_dir(workspace, 1, "beads")`, `<workspace>/sdd/beads`,
`<workspace>/.sase/sdd/beads`, and `<workspace>/sase/repos/beads`, for both the current
checkout and the project record's workspace. The wait barrier tries exactly one path.
Two resolvers with different answers to "where are this project's beads" is the
structural defect underneath the incident: doctor can report a healthy store for a
project whose waiters are permanently stuck.

The same file's walk-up fallback has the mirror-image problem.
`_current_or_primary_beads_dir` in `src/sase/bead/workspace.py` walks up from the
current directory looking for any `sdd/beads`, so a command run inside one project's
workspace can silently bind to an unrelated project's store when the canonical lookup
fails. That is how an unrelated store's beads can appear under `sase bead list` for this
project today.

### Design commitment

The barrier keeps failing closed. An unresolvable store must never auto-satisfy a wait,
and a bead that is absent from a readable store must never be assumed closed — a phase
bead is often created moments before its waiter is evaluated. Everything below is about
making an undecidable wait _attributable and recoverable_, not about guessing.

## One project bead-store resolver with an explicit availability result

Replace the three disagreeing resolvers with a single lookup, and make "unavailable" a
value rather than an absence.

In `src/sase/bead/store_locator.py`:

- Add a frozen result type — a `BeadStoreLookup` with the resolved `beads_dir` when one
  was found, the ordered `candidates` that were tried, and the reason resolution failed
  (`no_project_record`, `no_candidate_exists`, `unreadable`). Return it from a new
  `resolve_project_bead_store(project)`.
- Reimplement `canonical_beads_dir_for_project` on top of it so existing callers keep
  their current signature.
- Give `closed_bead_ids_for_project` and `bead_statuses_for_project` sibling functions
  that return the lookup alongside the data, so callers can distinguish "store
  unavailable" from "store readable, bead absent". Keep the existing `None`-returning
  functions as thin wrappers; the callers converted in the `signal` and `surface` phases
  move to the richer form.

In `src/sase/bead/workspace.py`:

- Extend `_canonical_project_beads_dir` to try the same ordered candidate list doctor
  already uses — the resolved SDD `beads` kind root, `<primary>/sase/repos/beads`,
  `<primary>/.sase/sdd/beads`, `<primary>/sdd/beads` — instead of returning `None` as
  soon as the storage mode's single expected path is missing. This is a read-only
  widening: no materialization, no clone, no network.
- Constrain `_current_or_primary_beads_dir` so the cwd walk-up stops at the resolved
  primary workspace root rather than continuing to the filesystem root. A checkout that
  belongs to a known project must never bind to a store outside it.

In `src/sase/doctor/checks_beads.py`, replace the hand-rolled `_candidate_beads_dirs`
project branch with the shared resolver so doctor and the wait barrier can no longer
disagree. Keep doctor's extra cwd-relative candidates, which serve a different purpose.

Do not change SDD store materialization, adoption, or the provider-policy fallback in
`src/sase/sdd/_store_resolution.py`. That silent downgrade is real and it is what erased
this project's sidecar configuration, but repairing materialization is a separate change
with its own risk; this phase makes the consequence visible and the `surface` phase
makes doctor report it.

Tests: a project whose beads live under each supported layout resolves to that store; a
project with no store anywhere returns an unavailable lookup that names every candidate
it tried; a workspace nested under an unrelated store no longer resolves to it.

## Structured blocked reasons through the wait barrier

Make the barrier record _why_ each bead dependency is unsatisfied, and carry that reason
to the surfaces that watch waiters.

In `src/sase/core/wait_dependency_resolution/_types.py` and `_resolution.py`:

- Add a per-dependency blocked reason for bead entries: `store_unavailable`,
  `absent_from_store`, `not_closed`, and `malformed` for a non-string or empty entry.
  Expose it on `WaitDependencyStatus` as an ordered mapping keyed by bead id, leaving
  `blocked_on` byte-identical so no existing consumer changes behavior.
- Take the richer store lookup rather than a bare `closed_bead_ids` collection, and keep
  the existing keyword argument working for callers that already pass a set. Resolution
  outcomes must not change: every case that blocks today still blocks.

In `src/sase/axe/run_agent_wait_deps.py` and `src/sase/axe/run_agent_wait.py`:

- Thread the reasons out of `initial_dependencies_resolved` and
  `waiting_marker_dependencies_resolved`, and persist them on `waiting.json` as a
  `bead_wait_diagnostic` object next to `wait_for_beads`, refreshed on each fallback
  pass with the timestamp of first observation so dwell time is derivable.
- Print a single line when the reason set changes — not on every two-second poll — so an
  agent's own log says `bead store unavailable for project <name>` rather than repeating
  a bare `Waiting for beads: ...`.
- Make `refresh_bead_wait_store` log at warning level when the canonical store cannot be
  resolved, instead of returning silently. It still must not raise; a waiting runner
  survives refresh failures.

Because ACE reads waiting markers through the Rust-backed agent scan, the new field has
to exist on the waiting wire type before any TUI surface can consume it. Add
`bead_wait_diagnostic` to the waiting marker wire in the sibling `sase-core` repo
(`crates/sase_core`), release the binding, and mirror it in
`src/sase/core/agent_scan_wire_markers.py`. Treat a missing field as absent, never as an
error: markers written by an older host must keep loading.

In `src/sase/scripts/sase_chop_wait_checks.py`:

- Extend `_terminal_blockers` — or add a bead-specific sibling — so a waiter blocked by
  `store_unavailable` or `absent_from_store` produces a log line naming the waiter, the
  project, the bead ids, and the candidate paths that were tried.
- Add counters for those two cases and return `status=warn` with evidence rows when
  either is non-zero, rather than folding them into `unresolved` and reporting
  `dependencies_not_ready`.

In `src/sase/scripts/sase_chop_bead_store_refresh.py`, stop reporting
`no_canonical_stores` as a no-op when `projects_waiting` is greater than zero. That
combination is a warning with the affected project names as evidence.

Tests: each blocked reason is produced for its own condition and persisted to
`waiting.json`; a marker without the new field still resolves; `wait_checks` returns
`warn` with evidence for an unresolvable store and stays `no_op` for an ordinary pending
dependency; `bead_store_refresh` warns only when waiters exist.

## Doctor check, notification, and waiter rendering

Close the loop so this condition is impossible to sit on unnoticed.

Add a `project.bead_waiters` check in `src/sase/doctor/checks_beads.py`, registered
alongside the existing specs in `bead_check_specs`. It scans live agent artifacts for
waiting markers carrying `wait_for_beads`, and reports `ERROR` for any live waiter whose
bead dependencies cannot be decided, listing the agent name, the bead ids, the reason,
and the candidate store paths that were tried. It reports `OK` when every live waiter's
beads resolve, and `SKIP` when there are no bead waiters. Its next step points at the
concrete recovery: restore a bead store for the project that contains those beads, or
release the waiter from ACE's wait modal. A waiter whose beads are merely `not_closed`
is healthy and must not be reported.

Add a notification, appended through `sase.notifications.store.append_notification`,
raised by the `wait_checks` chop when a waiter has been blocked by an undecidable bead
dependency for longer than a dwell threshold — thirty minutes, taken from the first
observation stamped on the marker in the `signal` phase. Key it on the waiter's artifact
directory so it fires once per waiter and not once per chop tick, and clear it when the
waiter resolves or exits.

Render the reason where waiters are watched:

- In `src/sase/ace/tui/models/agent_wait_beads.py`, carry the store lookup through the
  cache so an unresolvable store is cached as its own state rather than as an
  indistinguishable miss, and keep the existing TTL behavior for real statuses.
- In `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py`, badge an
  undecidable bead target with the existing `_UNRESOLVABLE_WAIT_TARGET_GLYPH` and its
  style, matching how unresolvable agent wait targets already render. Do not invent a
  new glyph vocabulary.
- In `src/sase/integrations/_agent_list_entry_builder.py` and
  `src/sase/agents/cli_list.py`, carry the diagnostic onto the agent list entry and its
  JSON output so `sase agent list -j` shows why a waiter is stuck without opening ACE.

Tests: doctor reports `ERROR` for a live waiter with an unresolvable store and `OK` when
the same waiter's beads resolve; the notification fires once past the threshold and not
before; the ACE wait section badges an undecidable bead target; `sase agent list -j`
carries the reason. Extend the existing ACE visual snapshot coverage only if the wait
section's rendered width changes.

## Non-goals

- Auto-resolving a bead wait when the store is unavailable or the bead is missing. The
  barrier stays fail-closed.
- Repairing SDD store materialization, re-materializing sidecar clones, or changing the
  provider-policy fallback in `src/sase/sdd/_store_resolution.py`.
- Re-pointing, merging, or migrating project records. Recovering the currently stalled
  `sase-js` waiters is an operator action: restore a bead store for the project that
  contains the `sase-js` beads, or release `sase-js.6` and `sase-js.7` from ACE's wait
  modal. This plan makes that situation loud; it does not perform it.

## Verification

Run `just check` in the workspace for each phase. Run `just check-full` before landing
the epic's combined tree, since the `signal` phase changes a wire type shared with
`sase-core` and the `surface` phase touches ACE rendering.

The end-to-end acceptance for the whole epic: a project whose canonical bead store
cannot be resolved, with one live bead-gated waiter, produces a `warn` from
`wait_checks`, an `ERROR` from `sase doctor`, one notification, and an unresolvable
badge on the waiter's wait section — and the waiter still does not start.
