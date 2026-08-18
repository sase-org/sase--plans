---
tier: tale
title: Suppress task-bead gates while a live agent is working the bead
goal:
  A task bead with a live agent working it never receives a new TaskTriage, FlagTriage,
  or BeadSnooze gate notification, and any gate already pending for it is canceled, for
  the whole lifetime of that agent.
size: medium
proposed_by: bbugyi200.athena.06o
create_time: 2026-08-18 15:47:31
status: wip
---

# Plan: Suppress task-bead gates while a live agent is working the bead

## Symptom

The owner received a second, identical `TaskTriage` gate notification (Telegram) for a
task bead they had already launched an agent for. The two notifications are two
generations of the same gate, five minutes apart, not a re-delivery of one message.

## Evidence (incident trace, athena, 2026-08-18)

Reconstructed from `~/.sase/interaction_requests/task_triage/` and
`~/.sase/procs/procs.jsonl` for bead `sase-q1`:

| Time (local) | Event                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------ |
| 14:45:08     | `bead_task_triage` creates `bead-task-triage-sase-q1-e8a9d175d981-g1`                      |
| 14:46:39     | Owner answers `launch` from Telegram; response records `task_launch_task_id: a6hsfg6b77hn` |
| 14:46:50     | Proc `a6hsfg6b77hn` starts `sase bead work sase-q1 --yes-to-all` in workspace `sase_25`    |
| 14:46:52     | Proc exits 1: `Error: issue not found: sase-q1`                                            |
| 14:50:13     | `bead_task_triage` creates `...-g2` — **the duplicate the owner saw**                      |
| 14:55:18     | `bead_task_triage` cancels `g2` (`task_triage_presentation_changed`) and creates `...-g3`  |

The persisted lane state (`~/.sase/axe/lumberjacks/checks/bead_task_triage.json`) still
shows `sase-q1 -> ...-g3`, i.e. generation 3 of one bead's gate.

## Root cause

`sase_chop_bead_task_triage._reconcile` has exactly one "this bead is already being
worked" signal:

```python
in_flight_launches = active_task_launch_bead_ids()   # src/sase/scripts/sase_chop_bead_task_triage.py:331
```

`sase.bead.task_launch.active_task_launch_bead_ids` returns bead IDs that have an
**active proc** whose command is `sase bead work <id>`. That proc lives for a couple of
seconds: it preclaims the bead, checkpoints, spawns the agent runner, and exits. After
it exits, the chop's only remaining protection is the bead's stored status — and
`_gateable_beads` (`src/sase/scripts/_bead_task_triage_state.py`) drops a bead once it
reads `in_progress`.

That backstop is not reliable, for two independent reasons:

1. **Store lag.** `sase bead work` runs in a _leased_ checkout (`sase_25` above) and
   writes `status=in_progress` there, then commits and pushes. The chop reads the
   _canonical_ store, `canonical_beads_dir_for_project(...)` → the project's **primary**
   workspace. Until that primary checkout integrates the push, the chop still sees
   `ready` and re-gates. Nothing suppresses gating during that window.
2. **Failed launch.** In this incident the launch never got as far as preclaiming: the
   leased checkout did not yet have `sase-q1` in its store. The bead stayed `ready`, the
   proc died in three seconds, and every later tick was free to re-gate.

So the suppression window is bounded by the _launch proc's_ lifetime, not by the
_agent's_ lifetime. `docs/notifications.md` even states the assumption this defect
breaks: "because the launch changes the stored status, the next reconciliation cancels
that stale gate."

## Requirement

From the owner, verbatim in intent: as soon as an agent is launched to work a task bead,
and for as long as that agent is running, SASE must not send task-bead gate
notifications for that bead.

## Design

Replace the chop's single narrow signal with a composite **work-in-flight** view built
from two independent sources:

- **launching** — an active `sase bead work <bead>` proc. Today's signal. Covers the few
  seconds between the owner's answer and the agent runner writing its metadata.
- **working** — a live agent artifact record whose `agent_meta.bead_id` names the bead.
  New. Covers the agent's entire run, independent of bead-store sync and independent of
  how the agent was launched (gate answer, ACE Beads pane, or a hand-typed
  `sase bead work`).

The "working" source is exactly the shape `bead_claim_checks` and `sase doctor` already
use: `scan_agent_artifacts(projects_root, options)` for `ace-run` artifact directories,
then `sase.agent.names.is_process_alive({"pid": ..., "stopped_at": ...}, artifact_dir)`.
`agent_meta.json` for a task worker carries `bead_id` equal to the task bead ID
(verified on this machine: e.g.
`{'name': 'sase-q3.1', 'bead_id': 'sase-q3.1', 'pid': 1653127}`), and it is written
early in `bootstrap_agent_run`, before any dependency or slot wait — so a queued agent
counts as working its bead, which is what the requirement asks for.

### Decisions, with rationale

- **Do not add a feature flag.** Per `sase/memory/sase_flags.md`'s rule, flags cover
  user-reaching behavior that is not ready (disabled beta, early-landed path, reachable
  deprecation). This is a defect fix that lands complete; there is no old branch to keep
  reachable and nothing for the owner to choose. Diagnosability is handled with an
  explicit log line instead (below).
- **Scope the new signal by project, leave the existing launch signal alone.** Artifact
  records carry `project_name`, so `(project_name, bead_id)` matching is free and
  strictly correct. `active_task_launch_bead_ids()` stays bead-ID-only: changing it is
  behavior the owner did not ask for, and its callers and test helpers would all churn.
  Record the asymmetry in the new module's docstring so a future change moves both
  together.
- **Fail open on scan failure.** Wrap the scan in `try/except`, log a warning, and treat
  the result as empty — matching the existing handling of
  `active_task_launch_bead_ids()` at line 331 ("keep today's triage behavior"). Failing
  _closed_ would silently stop all task triage for every project during a persistent
  scan outage, which is a worse failure than one extra gate.
- **Do not add new result counters.** Many tests assert the counters dict exactly. Feed
  the new signal into the existing `deferred` / `canceled` / `skipped` counters and
  widen their documented meaning.
- **Keep this in Python, in this repo.** Per `CLAUDE.md`'s Rust-core boundary note, the
  litmus is whether another frontend needs the behavior to match the TUI. This is
  host-side chop reconciliation, and its siblings — `sase/bead/task_triage_policy.py`,
  `sase/bead/task_launch.py`, `sase_chop_bead_claim_checks.py` — all live here. The
  filesystem scan itself already crosses into Rust via `scan_agent_artifacts`.

### Behavior matrix

For a bead the chop would otherwise act on:

| Chop branch                                                   | launch proc active  | live agent | New behavior                                                                                                                                                       |
| ------------------------------------------------------------- | ------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Pending gate, bead still gateable (line 385)                  | yes                 | —          | `skipped`, gate left pending (**unchanged** — the launch may still fail, and the pending ask is the owner's fallback)                                              |
| Pending gate, bead still gateable (line 385)                  | no                  | yes        | **Cancel** the gate with reason `bead_work_in_flight`, drop it from lane state, count `canceled`; the bead then falls through to the creation loop and is deferred |
| Pending gate, bead no longer gateable / suppressed (line 430) | yes, and suppressed | —          | `skipped` (**unchanged**)                                                                                                                                          |
| Pending gate, bead no longer gateable / suppressed (line 430) | no                  | yes        | Falls through to today's cancel (`task_bead_no_longer_ready`) — already correct                                                                                    |
| Creating a new gate (line 458)                                | yes                 | —          | `deferred` (**unchanged**)                                                                                                                                         |
| Creating a new gate (line 458)                                | no                  | yes        | `deferred` (**new**)                                                                                                                                               |

Canceling a pending gate when a live agent owns the bead is the self-healing half of the
requirement: it clears asks that are already stale, including ones stranded by this very
defect, and covers launches that bypass gate cancellation (a hand-typed
`sase bead work`, and ACE's `_submit_task_bead_launch`, which cancels only the
`task_triage` kind and so leaves a `FlagTriage` gate pending on a due flag bead).

Because the suppression key is the bead ID, it applies uniformly to all three kinds this
chop owns — `TaskTriage`, `BeadSnooze`, and `FlagTriage` — with no per-kind branching.

### Known residual gap (accept, do not chase)

There is a sub-second window after the launch proc exits and before the agent runner
writes `agent_meta.json` in which neither signal is true. The chop ticks every 300s, so
landing inside that window is vanishingly unlikely, and the next tick recovers. Do not
add a grace period or a timestamp heuristic for it.

## Implementation

### 1. New module: `src/sase/bead/work_liveness.py`

Owns the shared agent-liveness read for bead work.

```python
"""Which beads currently have work in flight, and what proves it."""
```

Public surface:

- `AGENT_BEAD_SCAN_OPTIONS: AgentArtifactScanOptionsWire` — the `ace-run`-only, marker-
  light options constant. Move the identical literal out of
  `sase_chop_bead_claim_checks._SCAN_OPTIONS` and have that chop import this one, so the
  two callers cannot drift apart.
- `agent_record_is_alive(record: AgentArtifactRecordWire) -> bool` — the
  `pid`/`stopped_at` → `is_process_alive(..., artifact_dir)` predicate, extracted so
  `bead_claim_checks._claim_owner_is_alive` and this module share one definition.
  (`bead_claim_checks` builds `_ClaimArtifact` from records first; either have it call
  this on the raw record before building, or keep its thin wrapper delegating here —
  implementer's choice, but there must be exactly one liveness rule.)
- `beads_with_live_agents(projects_root: Path | None = None) -> frozenset[tuple[str, str]]`
  — scan, keep `ace-run` records with a non-empty `agent_meta.name` and
  `agent_meta.bead_id`, take the newest record per `(project_name, agent_name)` by
  `record.timestamp` (same rule as `bead_claim_checks._latest_owner_records` — a stale
  record must not outvote a fresh one), keep the alive ones, return
  `{(project_name, bead_id)}`. Default `projects_root` to `sase_projects_dir()`.
- `BeadWorkInFlight` frozen dataclass with `launching: frozenset[str]`,
  `working: frozenset[tuple[str, str]]`, and methods `is_launching(bead_id)`,
  `is_worked(project, bead_id)`, `covers(project, bead_id)`.
- `bead_work_in_flight(log_warning: Callable[[str], None]) -> BeadWorkInFlight` — builds
  both halves, each guarded independently so one failing source does not blank the
  other. Prefer passing the chop's `runtime.log.warning`; do not import `ChopLogger`
  here.

Docstring must state: the launch half is keyed by bead ID only (matching today's
behavior) while the agent half is project-scoped, and both must move together if that is
ever changed.

### 2. Rewire `src/sase/scripts/sase_chop_bead_task_triage.py`

- Replace the `active_task_launch_bead_ids` import with `bead_work_in_flight` (keep a
  module-level indirection so tests can monkeypatch it on the chop module, exactly as
  `patch_active_launches` does today).
- Replace the `in_flight_launches` block at lines 329–336 with one `work` value.
- Line 385: `if work.is_launching(bead_id): skipped += 1; continue`.
- Line 385 branch, after the launch check: compute
  `worked = work.is_worked(project_name, bead_id)`. Skip the fingerprint shortcut when
  `worked` is true, and pass the cancel reason `bead_work_in_flight` (taking precedence
  over `bead_status_changed` and `task_triage_presentation_changed`). Everything after
  the cancel — dropping the state entry and re-adding the issue to `live_tasks` — is
  unchanged.
- Line 430: `if bead_id in suppressed and work.is_launching(bead_id)` (unchanged
  semantics, launch-only).
- Line 458: `if work.covers(project_name, bead_id): deferred += 1; continue`.
- When a bead is deferred or canceled because a live agent owns it, emit one
  `runtime.log.info` line naming the bead and the owning agent name. This is the
  diagnosability substitute for a feature flag: a misfiring liveness read must be
  greppable in `~/.sase/axe/lumberjacks/checks/chops/bead_task_triage/runs/*.log`.
  Return the agent name from `beads_with_live_agents` (make the value a small record, or
  return a mapping) so the log line can name it.
- Update the module docstring and the `deferred` counter's meaning to say "work in
  flight (an active launch proc **or** a live agent)".

### 3. Test-helper defaults

In `tests/_axe_chop_bead_task_triage_helpers.py`:

- Add `patch_live_agent_beads(monkeypatch, pairs=frozenset())`, mirroring
  `patch_active_launches`.
- Call it from `patch_project` so every existing triage test stubs the scan out by
  default and never touches the real artifact tree. **Without this the whole
  `test_axe_chop_bead_task_triage_*` suite starts scanning `~/.sase/projects`.**

### 4. Tests

New module `tests/test_axe_chop_bead_task_triage_agent_liveness.py` (keeps the existing
launch-deferral module focused on procs):

1. Ready task bead + live agent record → not gated, `deferred=1`, no lane-state entry.
2. Pending gate + live agent → gate canceled with reason `bead_work_in_flight`,
   lane-state entry dropped, `canceled=1` and `deferred=1` on the same tick; the bead is
   **not** re-gated.
3. Pending gate + live agent + changed presentation → still one cancel with
   `bead_work_in_flight`, not `task_triage_presentation_changed`, and no replacement
   gate.
4. Pending gate + active launch proc and **no** live agent → `skipped=1`, gate untouched
   (regression guard for today's behavior).
5. Same bead ID, different project's live agent → **not** suppressed; the gate is
   created.
6. Live agent goes away → the next tick gates normally with the next generation suffix.
7. Due flag task bead + live agent → the `FlagTriage` gate is deferred, proving the
   suppression is kind-agnostic.
8. `beads_with_live_agents` raising → warning logged, gating proceeds unchanged.

New module `tests/test_bead/test_work_liveness.py` for the helper itself:

- Builds `(project, bead_id)` pairs from fake records; skips records with no `bead_id`,
  no `agent_meta`, or a non-`ace-run` workflow dir.
- A dead pid and a set `stopped_at` both read as not alive.
- Two records for one `(project, agent_name)` → only the newest `timestamp` decides.
- `BeadWorkInFlight.covers` is true for either half and false otherwise.

### 5. Documentation and chop description

- `src/sase/default_config.yml`, the `bead_task_triage` chop `description` (the sentence
  ending "A gateable bead with a detached launch still in flight is deferred instead of
  re-gated"): restate it as deferral while **either** a detached launch is in flight or
  a live agent is working the bead, and note that a pending gate is canceled when a live
  agent owns the bead.
- `docs/notifications.md`, "Task Triage Notification": correct the now-wrong sentence "A
  direct `sase bead work <task-id>` command does not settle an older gate itself;
  because the launch changes the stored status, the next reconciliation cancels that
  stale gate." Replace the status-change rationale with the liveness rule, and extend
  the final "except while the task bead's detached launch is still in flight" clause to
  cover a live agent. Mention that the rule covers `BeadSnooze` and `FlagTriage` too.
- Do **not** touch `sase/memory/*.md`, `AGENTS.md`, or any generated provider shim: no
  memory edit was authorized for this work.

## Out of scope (report, do not implement)

The incident's proximate trigger was a _failed_ launch, and it was invisible to the
owner: proc `a6hsfg6b77hn` exited 1 with `Error: issue not found: sase-q1` because the
leased checkout's bead store predated the bead, and the only feedback the owner got was
another identical triage gate five minutes later. This plan deliberately does **not**
change that: a failed launch means the bead genuinely is not being worked, so re-gating
it is correct. The real gaps are (a) `sase bead work` failing on a stale leased checkout
instead of refreshing it, and (b) the silence when a gate-answered launch dies. The
implementer should record these as `PROPOSED FOLLOW-UP:` notes rather than widening this
plan.

## Verification

```bash
just install
just check
```

`just check`'s scoped lane should select the chop, bead, and notification suites. If the
scoped selection escalates or looks unusual, hand `just check-full` to `/sase_monitor`
with a `--next` action rather than running it inline.

Targeted while iterating:

```bash
just test tests/test_axe_chop_bead_task_triage_agent_liveness.py \
          tests/test_axe_chop_bead_task_triage_launches.py \
          tests/test_bead/test_work_liveness.py \
          tests/test_axe_chop_bead_flag_triage.py \
          tests/test_axe_chop_bead_task_triage_gate_lifecycle.py \
          tests/test_bead/test_gate_lookup.py
```

No test pins the `default_config.yml` chop description prose
(`tests/test_axe_lumberjack_config.py` only asserts `bead_task_triage` is present in the
`checks` lane), so the wording change is guarded by review, not by a fixture. Run
`just fmt` after editing `docs/notifications.md` so prettier reflows the prose to the
configured markdown print width.

## Done when

- A task bead with a live agent working it receives no new `TaskTriage`, `FlagTriage`,
  or `BeadSnooze` gate, for the agent's whole run, regardless of what the canonical bead
  store says.
- A gate already pending for such a bead is canceled with reason `bead_work_in_flight`.
- A bead whose launch proc is still running keeps today's behavior exactly.
- A bead whose agent is gone is gated again on the next tick.
- `just check` passes.
