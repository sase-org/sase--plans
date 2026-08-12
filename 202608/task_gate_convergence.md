---
tier: epic
title: Converge task bead gates and settle them the moment a bead closes
goal: "Live TaskTriage/BeadSnooze notifications match the set of live task beads even
  after a project leaves the inventory or the chop's state file is lost, and closing a
  task bead from the CLI clears its gate notification immediately instead of up to five
  minutes later.

  "
phases:
  - id: gate_lookup
    title: Shared pending bead-gate lookup
    depends_on: []
    size: small
    description: "gate_lookup: add one scan-once resolver for pending task_triage and
      bead_snooze bundles that reports each gate's kind, request id, project, bead id,
      and producing chop, and move the existing per-bead triage scan onto it.

      "
  - id: chop_sweep
    title: Make the reconciler converge on gates it no longer tracks
    depends_on:
      - gate_lookup
    size: medium
    description: "chop_sweep: cancel pending gates stranded by a project that left the
      enabled inventory and by a lost or corrupt state file, without cancelling gates
      for a project that is merely unreadable this pass.

      "
  - id: close_settle
    title: Settle bead gates from sase bead close
    depends_on:
      - gate_lookup
    size: medium
    description:
      "close_settle: have the close command cancel each closed task bead's pending gate
      right after the store mutation commits, so the existing notifications inotify
      watch refreshes ACE at once, and keep the added cost off closes that cannot have
      gates."
proposed_by: bbugyi200.athena.yk
create_time: 2026-08-12 10:58:32
status: wip
---

- **PROMPT:**
  [prompts/202608/task_gate_convergence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_gate_convergence.md)

# Plan: Converge task bead gates and settle them the moment a bead closes

## Background: what is actually broken

The user observed 27 task bead gate notifications against 10 open task beads. That is
not a rendering problem — both numbers are correct, and the gap is real.

Measured on the live host at planning time:

- Task beads: 9 `ready`, 1 `in_progress`, 0 `open`, 0 `snoozed`.
- Notifications tagged `bead`+`task` with action `TaskTriage`: 28 undismissed.
- The chop's lane state (`~/.sase/axe/lumberjacks/checks/bead_task_triage.json`) holds
  gates under **three** project keys: `gh_bobs-org__bob-cli` (0 gates),
  `gh_sase-org__sase` (10 gates), and `gh_sase-org__sase-2` (19 gates).
- `sase project list` and `list_project_records(...)` return exactly three enabled
  projects: `gh_bbugyi200__actstat`, `gh_bobs-org__bob-cli`, `gh_sase-org__sase`.
  `gh_sase-org__sase-2` is not among them and has no directory under
  `~/.sase/projects/`.
- Spot-checking a stranded bundle confirms the owner:
  `interaction_requests/task_triage/bead-task-triage-sase-jm-f0b220fcd1b4-g1/request.json`
  carries `"producer": {"chop": "bead_task_triage", "project": "gh_sase-org__sase-2"}`.

9 live gates + 19 stranded gates = 28. The arithmetic closes exactly.

### Root cause

`_reconcile()` in `src/sase/scripts/sase_chop_bead_task_triage.py:458` iterates only
`project_stores` — today's enabled inventory — and touches `projects[project_name]` only
for those. A state entry whose project later leaves the inventory is never visited
again, so its pending gates are never inspected, never cancelled, and their
notifications never settle. Nothing else on the host reconciles them either.

The chop's own description in `src/sase/default_config.yml:735` promises "Gates are
canceled when their beads leave those states." That promise silently holds only while
the owning project key stays in the inventory.

### Secondary gap: the state file is a single point of failure

`_read_state()` (`src/sase/scripts/sase_chop_bead_task_triage.py:69`) returns `{}` on
`JSONDecodeError` or `OSError`. If that file is lost or truncated, every tracked gate is
forgotten at once: generations restart at 1, a fresh gate is created for every ready
bead, and the previous generation's pending gates leak in exactly the same way. The chop
currently has no way to notice a gate it produced but no longer remembers.

### On the provenance of `gh_sase-org__sase-2`

Planning could not determine what created that project key. No such directory exists
now, and nothing in this repo generates `-<n>`-suffixed project keys (project names are
derived from directory names under `~/.sase/projects/`, verified experimentally against
`list_project_records`). The stranded gates were created in one burst at
`2026-08-11T18:06`, which is consistent with a project directory that existed briefly
and was later removed.

**This plan deliberately does not depend on that answer.** The reconciler must converge
no matter how a project key enters and leaves the inventory. Finding the creator is
worth a separate task bead; it is not a blocker here, and the sweep below cleans up the
existing 19 stranded gates on the first tick after landing without any manual migration.

## Background: why the second half is small

The user asked for `sase bead close` to "send some kind of event that the TUI can pick
up automatically." **That event channel already exists and works.** Do not invent a new
one.

- ACE starts an inotify watcher over `~/.sase/notifications`
  (`src/sase/ace/tui/actions/_startup_watchers.py:59-62`).
- A change there maps to the `notifications` dirty surface and calls
  `_schedule_notification_poll(source="watcher")` — an immediate, small-surface refresh
  that runs _even when auto-refresh is disabled_
  (`src/sase/ace/tui/actions/event_refresh/_watcher.py:63-70`).
- `cancel_gate` (`src/sase/notification_gates/executor.py:428`) calls
  `_settle_gate_notification` (`:573`), which marks the pending action handled and
  dismisses the notification row — writing that watched file.

So cancelling the gate _is_ the event. The missing link is that `handle_bead_close`
(`src/sase/bead/cli_crud.py:562`) never touches gates at all, so a CLI close leaves the
row up until the chop's next five-minute tick (the `checks` lumberjack runs at
`interval: 300`).

ACE's own beads pane already does the right thing: it calls `cancel_task_triage` after
the mutation and requests a notification refresh
(`src/sase/ace/tui/actions/_artifacts_beads_common.py:97-114`). The CLI is the outlier.

### The performance trap to avoid

`_find_pending_task_triage` (`src/sase/bead/_task_gate_actions.py:174`) scans _every_
bundle directory under `interaction_requests/task_triage/` on each call. On the live
host that is **506 directories** (of which only 29 are pending), so roughly 1,000
`stat()` calls per lookup. Bundles are never reaped, so this only grows.

Calling that helper once per closed bead would make a bulk close O(beads x bundles). The
user explicitly asked that this not hurt `sase bead close`, which is why `gate_lookup`
exists as its own phase: one scan serves every bead in the close, and the scan is
skipped entirely when the close cannot have produced a gate.

## Phase gate_lookup: Shared pending bead-gate lookup

Add one resolver both later phases consume. Suggested home:
`src/sase/bead/gate_lookup.py` (a new module in the bead package, since both consumers
reason in terms of beads).

Shape:

- A frozen dataclass describing one pending gate: `kind`, `request_id`, `project`,
  `bead_id`, and `producer_chop` (`None` when the envelope names no producer).
- One function that scans the bundle directories for a requested set of gate kinds and
  returns every _pending_ gate it can identify, in one pass.

Requirements:

- Accept the kinds to scan; callers pass `TASK_TRIAGE_KIND`, `BEAD_SNOOZE_KIND`, or
  both. Never hardcode a single kind.
- A bundle is pending when neither `response.json` nor `cancellation.json` exists. Reuse
  the existing terminal-file convention rather than inventing a new predicate.
- Only pending bundles get their `request.json` parsed. On the live host that is 29
  parses out of 506 directories — keep that ratio.
- Read identity from the trusted envelope: `payload.project`, `payload.bead_id`,
  `request_id`, `kind`, and top-level `producer.chop`. Skip any bundle whose envelope is
  unreadable, malformed, or missing those fields; a bad bundle must never raise out of
  the scan.
- A missing kind directory yields nothing, not an error.
- Return results in a stable, deterministic order so callers and tests do not depend on
  filesystem ordering.

Then move `_find_pending_task_triage` onto the new resolver so exactly one scan
implementation exists. `cancel_task_triage`'s signature and behavior must not change —
its two ACE callers (`_artifacts_beads_common.py:97`, `_artifacts_beads_launch.py:82`)
keep working untouched, and the refactor incidentally removes a redundant scan from
those paths too.

Tests: exercise pending vs answered vs cancelled bundles, both kinds in one scan, a
malformed `request.json`, a bundle missing `payload`, an absent kind directory, and a
bundle with no `producer`. `tests/test_bead/task_gate_test_helpers.py` and
`tests/test_bead/snooze_gate_test_helpers.py` already build real bundles — build on them
rather than hand-writing envelope JSON.

## Phase chop_sweep: Make the reconciler converge on gates it no longer tracks

Two independent sweeps in `src/sase/scripts/sase_chop_bead_task_triage.py`, both inside
the existing lane lock and state write.

### Sweep 1: project entries that left the inventory

`_enabled_project_stores` (`:175`) currently drops a project silently when
`canonical_beads_dir_for_project` raises or returns `None`. That distinction matters:

- A project that is **gone from the inventory** should have its pending gates cancelled
  and its state entry dropped.
- A project that is **in the inventory but unreadable this pass** must be left strictly
  alone. Cancelling its gates would cause a cancel/recreate flap — a burst of duplicate
  notifications — every time a store is transiently unavailable.

So `_enabled_project_stores` must report both the stores it resolved _and_ the project
names it saw in the inventory but skipped. The sweep set is then:

```
state_projects - reconciled_projects - skipped_projects
```

For each swept project, cancel every pending gate it names (using the recorded `kinds`
map, defaulting to `TASK_TRIAGE_KIND` for legacy version-2 entries, exactly as the main
loop already does at `:498`), then delete the whole entry including its `generations`.

Design decisions to honor:

- **A disabled project is swept.** A disabled project should not keep nagging for
  triage. Re-enabling it re-gates on the next tick, and because the entry's generations
  are gone the new gates start at `g1` with fresh request ids — no collision with the
  cancelled ones.
- Use a distinct cancellation reason (for example `project_no_longer_enabled`) so the
  bundle records _why_, separately from the existing `task_bead_no_longer_ready`.
- If the whole project inventory fails to load, `_reconcile` already returns early with
  `project_inventory_unavailable` (`:465-469`). Keep that fail-closed behavior — never
  sweep on an empty inventory read.

### Sweep 2: bundles this chop produced but no longer tracks

This is the backstop that makes the chop self-healing against state loss. After the
per-project work, scan both kind directories with the `gate_lookup` resolver and cancel
any pending gate where all of the following hold:

- `producer_chop == "bead_task_triage"` — never touch gates another producer owns.
- Its `request_id` is not one this pass expects to be live (build that set from the
  state the pass is about to write, so gates just created or just kept are excluded).
- Its project is not in the skipped-because-unreadable set, for the same flap reason as
  Sweep 1.

Cost: one directory scan of each kind per tick — roughly 1,000 `stat()` calls and ~30
JSON parses every five minutes. Negligible against the chop's existing per-gate
`poll_gate` work and its `2m` timeout.

### Reporting

Extend `_summary` (`:403`) with the new counters so axe snapshots show convergence
happening rather than it being invisible. Keep the existing counter names and shape; add
rather than rename. The pass should no longer report `no_triage_changes` when it swept
something.

When new summary or log text names a swept project, follow the project-naming
convention: resolve a display name through `sase.project_display_names` and fall back to
the key only when none is known — which will be the common case for a project that no
longer exists.

Also update the chop's `description` block in `src/sase/default_config.yml:738-746` so
the documented contract matches the new behavior.

Tests belong with the existing suites — `tests/test_axe_chop_bead_task_triage.py` and
the shared `tests/_axe_chop_bead_task_triage_helpers.py`. Cover at minimum:

1. A state entry for a project absent from the inventory has its pending gates cancelled
   and its entry dropped.
2. A project in the inventory whose store read fails keeps its gates and its entry.
3. An empty/failed inventory read sweeps nothing.
4. A pending `bead_task_triage`-produced bundle with no state entry is cancelled.
5. A pending bundle produced by something else, and a bundle for a gate the pass just
   created, are both left alone.
6. A swept-then-re-enabled project gets a fresh `g1` gate rather than reusing a
   cancelled request id.

Regression check for the reported symptom: state naming a project key that resolves to
no store, holding gates for beads that are simultaneously live under a _different_
project key, converges to exactly one gate per live bead.

## Phase close_settle: Settle bead gates from sase bead close

In `handle_bead_close` (`src/sase/bead/cli_crud.py:562`), after the
`with bead_store_mutation(...)` block exits, settle the pending gate for each
just-closed task bead.

Placement and shape:

- **After** the `with` block, never inside it. The store lock, the auto-commit, and the
  push all live in that context manager; gate I/O must not extend the lock. ACE's beads
  pane sets the precedent (`_artifacts_beads_common.py:94-101`).
- Resolve the project key with `infer_project_name_from_cwd()`
  (`src/sase/bead/project_name.py:67`) — the gate payload's `project` is the ProjectSpec
  key, and that helper is marker-first and cheap.
- Cover **both** kinds. A closed `ready` task owns a `task_triage` gate; a closed
  `snoozed` task owns a `bead_snooze` wake gate. Closing a snoozed task today leaves its
  wake gate pending. There is no per-bead lookup helper for the snooze kind — that is
  precisely what `gate_lookup` supplies.
- Cancel with a reason that records the cause (for example `bead_closed`).

Performance requirements, in priority order:

1. **Skip entirely when nothing can have a gate.** Only task-type beads get gates. Build
   the candidate id set from the mutation outcome's `closed_ids` and
   `cascade_closed_ids`, filtered to `IssueType.TASK` using issues already in hand from
   `mutation.project.close(...)`. If the set is empty — every plan/phase close, and
   every already-closed no-op — do no filesystem work at all.
2. **Scan once, not once per bead.** One `gate_lookup` pass resolves every candidate. A
   twenty-bead `--phases` close must cost one scan, not twenty.
3. **Never fail or stall the close.** Gate settlement is best-effort: a `GateError` with
   code `already_answered` is an expected no-op (the human may have answered the gate
   concurrently), and any other gate/filesystem failure must be swallowed or logged, not
   propagated. The bead is already closed and committed by this point; the chop's next
   tick remains the backstop.

Relative cost: `sase bead close` already takes a store lock, writes a git commit, and
pushes to a remote. One directory scan and at most a few small JSON writes are a
rounding error beside that, and phase 1's skip check keeps even that off the common
path.

### TUI impact: no new cost

Nothing needs to change in ACE, and nothing should be added to its refresh path.

- The write lands in `~/.sase/notifications`, which is already watched.
- `ArtifactWatcher` coalesces inotify bursts before dispatching.
- `_schedule_notification_poll` (`_notification_deadlines.py:92`) explicitly coalesces
  timer, watcher, and mutation bursts — if a poll is already scheduled or running it
  sets a pending flag instead of launching another.

So a twenty-bead close producing twenty notification writes still costs ACE at most one
extra notification poll, on a surface that is deliberately small. Verify this rather
than assume it; do not add a second refresh trigger.

Tests: extend the `tests/test_bead/test_cli_close_*.py` family. Cover a ready task close
cancelling its `task_triage` gate, a snoozed task close cancelling its `bead_snooze`
gate, a bead with no gate being a clean no-op, a plan/phase close performing no bundle
scan at all, a multi-bead close performing exactly one scan, and an `already_answered`
gate not failing the command. Assert the observable contract — the bundle gains
`cancellation.json` and the notification row is dismissed — rather than asserting on
internal call sequences.

## Explicit non-goals

- **Reaping terminal bundles.** 506 `task_triage`, 709 `plan`, and 255 `epic_plan`
  bundle directories have accumulated with no reaper. That is a real and growing cost,
  but it is a separate concern from convergence; file it as a task bead.
- **Other bead verbs.** `sase bead snooze`, `sase bead open`, and
  `sase bead update -s ready` all change which gate a bead should own. They keep the
  chop's five-minute cadence. The user asked for `close`; widening to every verb belongs
  in follow-up work, and `gate_lookup` leaves the door open for it.
- **Hunting what created `gh_sase-org__sase-2`.** Worth a task bead; not required for
  convergence.

## Verification

Run `just install` first — workspace directories are ephemeral and dependencies drift.
Then `just check` per phase, and `just check-full` on the epic's combined tree before
landing, since this touches a chop script, the bead CLI, and shared gate plumbing.

Post-land, the live symptom is the acceptance test: within one five-minute `checks`
tick, the stranded `gh_sase-org__sase-2` gates should cancel and the task bead gate
notification count should fall to the number of live `ready` plus `snoozed` task beads.
