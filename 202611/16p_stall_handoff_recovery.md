---
tier: epic
title: Unstick the 16p session and close its handoff-slot failure chain
goal:
  The 16p recovery work is finished (sase-core rustfmt fix landed, broken LaunchApproval
  withdrawn, remaining epics relaunched), and one interrupted gate command can no longer
  block later handoffs, orphan a monitor, leave an actionable lost gate, or discard an
  accepted final declaration.
decisions:
  lost_intent_outcome:
    ask:
      Should a turn whose accepted declaration postdates its lost gate intent end DONE
      after finalizers?
    choices:
      complete: Finalizers run; the turn ends DONE with a lost-gate warning notification
      fail_after_finalize:
        Finalizers run; the run still ends FAILED with the lost-gate error
    default: complete
    why:
      The agent saw the failed gate itself, and FAILED invites retries of finished work
    answer: complete
  prune_window:
    ask:
      How long should already-handled pending actions stay in the store before writes
      prune them?
    choices:
      week:
        Prune handled entries after 7 days; the 13 MB store drops below 1,000 entries
      month:
        Prune handled entries after 30 days; about 2,250 handled entries (~5 MB) remain
    default: week
    why:
      Notification rows compact after 3 days, so week-old handled entries annotate
      nothing
    answer: week
phases:
  - id: core-fmt
    title: Land the lost sase-core rustfmt fix
    depends_on: []
    size: small
    description:
      "core-fmt: re-run sase-core's pinned formatter on the staged_handoff modules, pass
      its check gate, and commit."
  - id: recover-epics
    title: Finish the recovery plan 16p abandoned
    depends_on: []
    size: medium
    description:
      "recover-epics: withdraw launch-de85665f, re-inventory, then redo steps 4, 5, and
      7 of the athena-restart recovery plan."
  - id: handoff-slot
    title: Reclaim dead-owner reservations and admit monitors before spawning
    depends_on: []
    size: medium
    description:
      "handoff-slot: reclaim staging reservations whose owner died; run handoff
      admission before legacy monitor start spawns."
  - id: gate-entry-atomic
    title: Make gate entry signal-safe and withdraw lost gates
    depends_on: []
    size: medium
    description:
      "gate-entry-atomic: turn termination signals into compensation during gate entry;
      withdraw notifications of lost gate turns; fix wording."
  - id: intent-lost-finalize
    title: Honor accepted declarations made after a lost gate intent
    depends_on:
      - gate-entry-atomic
    size: medium
    description:
      "intent-lost-finalize: let a later accepted final declaration supersede a lost
      gate intent so finalizers still run."
  - id: pending-action-prune
    title: Bound the shared pending-action store
    depends_on:
      - core-fmt
    size: medium
    description:
      "pending-action-prune: prune old already-handled entries in sase-core's
      pending-action store without reviving handled notifications."
proposed_by: bbugyi200.athena.16r
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 12:02:51
status: wip
---

- **PROMPT:**
  [prompts/202611/16p_stall_handoff_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/16p_stall_handoff_recovery.md)

# Unstick the 16p session and close the handoff-slot failure chain behind it

## Goal

Agent session `16p` was implementing the approved plan
`202611/recover_epics_after_athena_restart.md` (decisions `core_fmt=fix`,
`gc_fix=settle`). It is stalled: no session member is alive, nothing will resume it, and
its one finished deliverable (a rustfmt fix for sase-core) was lost. This epic:

1. finishes the work 16p left undone: land the sase-core rustfmt fix, withdraw the
   broken LaunchApproval, and run the recovery plan's remaining steps; and
2. fixes the four host defects that turned one interrupted `sase launch request` into a
   dead session and a lost commit, plus the store bloat that made the command slow.

## What happened (evidence)

All times are 2026-11-01 UTC.

- **14:47** — `16p--code` (muse, `@medium`) starts implementing the recovery plan. It
  finishes steps 1–3 (bead closes and notes, new beads `sase-1qi` and `sase-1qj`).
- **15:48:46** — For step 4 the agent runs
  `timeout 120 sase launch request -f /tmp/launch_request.json -o json | tail -n 30` (8
  slots).
- **15:50:22** — Under heavy post-restart load (load1 30–40), the CLI only now writes
  the staging reservation `.sase_handoff_staging.json` (pid 1853074) and the gate intent
  marker.
- **15:50:45** — Gate-turn member `16p--gate-0` is created and the creator's workspace
  claim moves to it. Notification `7196dff3-8edf-4857-914f-e439b8574bfd` is appended.
  `register_pending_action` starts rewriting the 13 MB
  `~/.sase/pending_actions/actions.json` and leaves `.actions.json.1853074.tmp` (3.6
  MB).
- **15:50:46** — `timeout` sends SIGTERM. Python's default SIGTERM handling skips every
  `except BaseException` and `finally` compensation. The reservation, the intent marker,
  a live notification and pending action, and a bundle journal stuck at
  `notification_prepared` are all left behind.
- **15:52** — A retry fails with
  `pending handoff already exists: .sase_handoff_staging.json`. Nothing checks whether
  the reserving pid is alive. The agent carries on without the gate and notes "Batch B
  withheld".
- **16:11:56** — `sase monitor start -J 8dff86de…` (sase-core check gate) spawns monitor
  `6jfm1m9h6xj8` (`16p--mon`). Only after that does the handoff marker write fail on the
  same stale reservation. The CLI crashes, the agent is not ended, and the monitor runs
  with `parent_node_ids: []`.
- **16:17:51** — The monitor exits 101. The only failure is the known flake `sase-1gz`.
  Its follow-up is blocked as `monitor result is missing its exact starter parent node`
  (`not-launchable`).
- **16:19** — The reclaimer settles `16p--gate-0` as `lost` ("gate bundle unreachable").
  The LaunchApproval notification stays unread and its pending action stays `available`,
  so it can still be approved.
- **16:21:13** — The agent submits a final declaration: commit the 4-file rustfmt fix in
  sase-core. The host **accepts** it.
- **16:21:52** — At `provider_done`, `raise_if_gate_intent_lost`
  (`src/sase/llm_provider/_invoke.py`) raises `GateIntentLostError` **before finalizers
  run**. The run is marked FAILED and the accepted commit never happens. The workspace
  is then recycled, and the sase-core clone with the fix is deleted. The diff survives
  only at
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202611/01/20261101111157/diagnostics/worktree_recovery.diff`.

sase-core master CI (latest run on `dc5383f6`) still fails `cargo fmt --check`. That
blocks the daily core cut, and some recovery continuations wait on it.

## Defects addressed

1. **Stale staging reservations block forever.** `acquire_staging_reservation` in
   `src/sase/agent/_staged_handoff_admission.py` records a pid, but
   `occupying_handoff_names` never checks whether that pid is alive.
2. **Legacy `sase monitor start` spawns before admission.** In
   `src/sase/main/monitor/start.py`, the CLI calls `start_monitor` before checking the
   handoff slot. `plan propose`, `questions`, and `pipe` call `handoff_guard` first.
3. **Gate entry is not signal-safe.** On SIGTERM/SIGHUP, `run_gate_entry`,
   `create_gate_turn`/`publish_gate_turn`, and `notification_gates` `create_gate` skip
   their compensation. A lost gate turn also leaves its published notification
   actionable.
4. **A lost gate intent discards a later accepted final declaration.** The intent check
   runs before finalizers, whatever the agent did after the intent was lost.
5. **The pending-action store never prunes.** It holds 6,982 entries, 6,427 of them
   `already_handled` and 4,177 older than 30 days. Every gate registration rewrites it
   whole under an exclusive flock. Rust has `cleanup_stale_pending_actions`, but nothing
   calls it, and it would also drop still-`available` entries.

## Phases

### Phase core-fmt: Land the lost sase-core rustfmt fix

- **ID:** core-fmt
- **Size:** small
- **Depends on:** none

The fix that `16p--code` declared was never committed. Re-create it and land it.

1. Open sase-core with `/sase_repo`, refresh onto `origin/master`, and read its
   `AGENTS.md`. If `origin/master` already passes the pinned formatter check, because
   someone else landed a fix, stop and report that instead.
2. Run sase-core's pinned-toolchain formatter (its `scripts/check.sh fmt` / `just fmt`
   entry point). Expected files:
   `crates/sase_core/src/staged_handoff/{mod.rs, recovery.rs,tests.rs}` and
   `crates/sase_core_py/src/staged_handoff/mod.rs`. Formatting only, no logic changes.
   Cross-check the result against the preserved `worktree_recovery.diff` named above.
3. Run sase-core's check gate through `sase tool run check` from the sase-core checkout.
   It can outlive the provider's 10-minute sync ceiling. If it does, hand the same run
   to `/sase_monitor` (`sase monitor start -J <run> …`) exactly as the CLI suggests.
   Never wrap it in `timeout`. A failure in
   `federation_worker::listener_creates_private_socket_and_rejects_symlink` is the known
   flake `sase-1gz`: +1 it, do not fix it here.
4. Land through `/sase_final` with a commit decision for sase-core. No sase pin change:
   no binding changes.

### Phase recover-epics: Finish the recovery plan 16p abandoned

- **ID:** recover-epics
- **Size:** medium
- **Depends on:** none

Continue `202611/recover_epics_after_athena_restart.md` from where `16p` stopped. Its
guardrails still apply and its decisions are final. Step 6 belongs to phase `core-fmt`.

1. **Withdraw the broken LaunchApproval** `launch-de85665f-93a1-4f5e-86e9-5556b4b30b22`
   (notification `7196dff3-8edf-4857-914f-e439b8574bfd`). Never approve it: its gate
   turn is `lost` and its bundle journal never reached `complete`. Reject it through
   `sase launch reject` with a reason. If the CLI refuses a never-completed bundle,
   dismiss the notification through `sase notify`'s supported mutation. Confirm the
   pending action is no longer `available`. Never hand-edit `actions.json`,
   `notifications.jsonl`, or bundle files.
2. Do **not** resume monitor `6jfm1m9h6xj8`. Its starter already consumed that result.
3. **Re-inventory from current state.** Hours have passed, so some phases may now be
   live, closed, or recovered by other agents. Redo the plan's step-1 inventory and the
   step-2 dry-run guard for the step-4 continuations (`sase-1qh.7`, `sase-1pi.1`,
   `sase-1om.8`, `sase-1nv.5.1.4`, `sase-1oe.4`, `sase-1q2.9.2`, `sase-1pt.9.2`,
   `sase-1n6.10`) and the step-5 Batch B epics. Confirm that steps 1–3 actually landed
   (closes of `sase-1pj.6.1.6`, `sase-1pj.7.1.2`, `sase-1pu.5.1.4`, `sase-1nv.1`,
   `sase-1nf.14`, notes, and the beads `sase-1qi`/`sase-1qj`). Drop segments that no
   longer need a continuation.
4. **Re-run step 4** with the remaining segments: one LaunchApproval, each segment
   preflighted with `sase macro expand`, and core-CI mentions citing phase `core-fmt`'s
   current state. Submit with exactly `sase launch request -f <file> -o json`, bare and
   in the foreground. No `timeout`, no pipe, and no background job: a slow request is
   still publishing, and killing it recreates this incident. The turn ends when the
   descriptor prints.
5. **Resumed turn:** run step 5 (Batch B) and step 7 (verify and report). The report
   also summarizes this 16p failure chain and names the withdrawn request.

### Phase handoff-slot: Reclaim dead-owner reservations and admit monitors before spawning

- **ID:** handoff-slot
- **Size:** medium
- **Depends on:** none

1. **Record owner identity.** `acquire_staging_reservation` writes `process_identity`
   (`sase.core.process_identity.process_identity_token`, as `begin_gate_intent` already
   does) next to `pid`.
2. **Reclaim provably dead reservations under the handoff lock.** A reservation whose
   owner pid is not alive, or whose recorded identity no longer matches, is no longer
   occupying. Move it aside as evidence (a timestamped
   `.sase_handoff_staging.stale-*.json` in the artifacts dir) and log it. Do this in one
   helper that `admit_stage`, `acquire_staging_reservation`, and
   `write_pending_handoff_marker` all reach through `occupying_handoff_names`. Fail
   closed: a live owner, an unreadable or malformed file, or a reservation with no pid
   keeps blocking. This is host process observation fed into the Rust admission request,
   so it stays in Python, like `gate_intent_guard`.
3. **Better refusal text.** "pending handoff already exists" names the reservation's
   owner pid and whether it is alive, so an agent can tell a stale block from a live
   one.
4. **Admit before spawning.** In the legacy (non-staged) branch of
   `src/sase/main/monitor/start.py`, call `handoff_guard(command="sase monitor start")`
   when `SASE_AGENT` is set, before `start_monitor`. If the post-spawn
   `maybe_handoff_monitor_from_agent` still fails (a race), stop the monitor that was
   just started through the existing monitor stop path, print that it was stopped, and
   exit non-zero. No orphan monitor may survive a failed handoff.
5. Tests:
   - a dead-pid reservation is reclaimed and evidence is written;
   - a live pid keeps blocking;
   - an identity mismatch is reclaimed;
   - malformed and no-pid reservations keep blocking;
   - monitor start refuses before spawning when the slot is occupied (assert no
     supervisor process or record);
   - the race path stops the monitor.

### Phase gate-entry-atomic: Make gate entry signal-safe and withdraw lost gates

- **ID:** gate-entry-atomic
- **Size:** medium
- **Depends on:** none

1. **Signal-safe critical section.** For the agent-side gate entry (reservation through
   prepare and publish in `src/sase/gate_turn/entry.py`, covering `create_gate_turn` and
   `publish_gate_turn`), turn SIGTERM, SIGHUP, and SIGINT into a dedicated
   `BaseException` subclass. The existing compensation then runs: gate-turn `failed`
   settlement plus claim restore, `_compensate_published_gate`, reservation release, and
   `clear_gate_intent`. Restore the previous handlers before `_complete_gate_entry`
   performs the handoff kill. Defer signals that arrive during compensation until it
   finishes. Reuse one context manager so plan, questions, and pipe entry points can
   adopt it later. Do not widen this phase to them.
2. **Lost gate turns withdraw their published gate.** Add one helper that takes a gate
   turn record settled `lost` and does three things:
   - finds its bundle by `gate_kind`/`gate_id` under `interaction_requests/`, even when
     the record has no `bundle_path` (16p's case);
   - if the journal never reached `complete` or no response exists, applies the same
     compensation as `_compensate_published_gate`: unregister the pending action,
     dismiss the notification, journal `failed`;
   - leaves answered gates alone.

   Call it from `settle_lost_unless_creating`'s callers (`_reclaim_pending.py`,
   `cancel.py`). Phase `intent-lost-finalize` reuses it.

3. **Stop the CLI and skills implying a blocking wait.** `sase launch request --help`
   says "Agent-side callers wait for the response". Reword it: from an agent, the
   command creates the gate and ends the turn, so run it bare in the foreground. Read
   reference memory `generated_skills` first. Then add one sentence to the source
   templates under `src/sase/macros/skills/` that tell agents to run turn-ending gate
   commands (at least `sase_run`; check `sase_gate`, `sase_sudo`, and `sase_questions`):
   never wrap them in `timeout`, a pipe, or a background job. Redeploy per that note.
4. Tests:
   - SIGTERM injected after notification append leaves no reservation, no intent marker,
     no `available` pending action, a dismissed notification, a `failed` journal, and a
     restored claim;
   - lost settlement withdraws an incomplete bundle's notification and leaves an
     answered one alone;
   - help-text and skill-render snapshots are updated.

### Phase intent-lost-finalize: Honor accepted declarations made after a lost gate intent

- **ID:** intent-lost-finalize
- **Size:** medium
- **Depends on:** gate-entry-atomic

`raise_if_gate_intent_lost` (`src/sase/llm_provider/gate_intent_guard.py`) runs at
`provider_done` in `invoke_agent`, before finalizers. Change only the success path:

1. Before raising, load the latest accepted current final submission with the loaders
   behind `_has_accepted_current_final_submission`. If it exists and its `accepted_at`
   is later than every remaining intent's `timestamp`, the lost intents are superseded.
   Then:
   - write `gate_intent_lost.json` with `superseded_by_final_submission: true` and the
     submission digest;
   - discard the intents;
   - settle any matching non-terminal gate-turn member `lost` through the
     `gate-entry-atomic` helper, which also withdraws its notification;
   - post a warning notification naming the request id and kind;
   - continue to finalizers.
2. Unchanged: an intent with no later accepted declaration (the gate was the turn's last
   act) still raises `GateIntentLostError`, and the exception-path wrapper
   `gate_intent_lost_error_for` is untouched.
3. Tests:
   - a later accepted declaration runs the finalizer commit and writes evidence and a
     warning;
   - an earlier or missing declaration still raises;
   - the live-intent wait path is unchanged.

> [!decision] lost_intent_outcome = complete The run finishes DONE and the warning
> notification is the only signal.

> [!decision] lost_intent_outcome = fail_after_finalize After finalizers succeed, record
> the lost-gate error as the run's failure (`done.json` `outcome: failed`, with
> `finalizers_completed: true`). Verify that auto-restart and healer sweeps skip runs
> whose accepted declaration already executed.

### Phase pending-action-prune: Bound the shared pending-action store

- **ID:** pending-action-prune
- **Size:** medium
- **Depends on:** core-fmt

The Rust pending-action store is core backend logic. Implement this in sase-core through
`/sase_repo`, in `crates/sase_core/src/notifications/pending_actions.rs`. `core-fmt`
lands first so master CI is green for this change.

1. In the shared write path (`write_pending_action_store_unlocked`, or one prune step
   that every mutating entry point runs under its lock), drop entries whose state is
   `already_handled` and whose `handled_at_unix` (falling back to `updated_at_unix`) is
   older than the decided window. Never drop `available` entries. Leave
   `cleanup_stale_pending_actions` unused.
2. Before landing, verify that pruning never flips an old notification back to
   actionable. Check what `pending_action_state_for_notification`,
   `pending_action_state_from_store`, the gateway, and Telegram report for a handled
   notification whose entry is gone. If that state would become `available`, make it
   fall back to the notification's own handled or response evidence first.
3. Add Rust tests for retention boundaries, survival of `available` entries, and the
   missing-entry state of pruned notifications.
4. On the sase side, follow `docs/rust_backend.md`. The `sase-core-revision.txt` pin
   moves only if a binding that sase calls changes. The first store write after deploy
   shrinks the file.

> [!decision] prune_window = week Retention is 7 days.

> [!decision] prune_window = month Retention is 30 days.

## Verification

- `core-fmt`: the sase-core master CI `fmt` job is green on the landed commit.
- `recover-epics`: `sase notify show --id 7196dff3-…` reports the request dismissed or
  rejected, and its pending action is not `available`. Every recovered phase and land
  has a live owner per the plan's step-7 report.
- Code phases: `just check` passes in sase, and sase-core's gate passes for
  `pending-action-prune`. Each phase's new tests encode its 16p failure mode, so
  reverting the fix fails them.
