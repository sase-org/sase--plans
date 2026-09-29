---
tier: tale
title: Keep gate-turn reclaim away from in-flight gate creation
goal:
  The housekeeping reclaim chop and explicit cancel never settle a gate turn as lost
  while its creation transaction is still running, and a live creator's own settlement
  no longer fails on a 5s follow-up lock timeout, so %auto plan gates stop killing epic
  phase agents after their plan is already approved.
size: medium
proposed_by: bbugyi200.athena.0ty
create_time: 2026-09-29 12:02:04
status: wip
---

# Keep gate-turn reclaim away from in-flight gate creation

## Problem

Epic phase agent `sase-1ck.4` (`%auto` + `#plan`) failed right after its tale plan was
approved and archived. The run died with:

```
GateError: timed out waiting 5s for lock on .../<gate-turn>/.gate_followup.lock; held by pid <housekeeping child>
```

### Root cause (confirmed from artifacts and `~/.sase/logs/workspace_claims.jsonl`)

1. `create_gate_turn()` (`src/sase/gate_turn/transaction.py`) creates the pending
   gate-turn member under the per-lane creation lock, then calls `create_gate(spec)`.
   The member's `gate_bundle_path` is only written by `_record_with_gate_result()`
   **after** `create_gate()` returns.
2. For an `%auto` gate, `create_gate()` → `_resolve_auto_gate()`
   (`src/sase/notification_gates/service.py`) runs the selected option commands inline
   (for a PlanApproval: approve + commit/archive the plan, ~25s in this incident) and
   then calls `settle_gate_turn(..., creator_live=True)`.
3. While that ran, the hourly housekeeping `gate_turn_reclaim` chop scanned the member.
   `_reclaim_one()` (`src/sase/gate_turn/reclaim.py`) saw a pending member with no
   bundle path and immediately settled it `lost` / "gate bundle unreachable". Under host
   load that non-creator settlement held `.gate_followup.lock` for ~14s (claim hold,
   decision record, index writes, claim restore).
4. The creator's `settle_gate_turn(creator_live=True)` hit
   `FOLLOWUP_LOCK_TIMEOUT_SECONDS` (5s, `src/sase/gate_turn/handoff.py`), raised, and
   the whole agent run failed — after the plan approval and archive commit had already
   happened. The phase bead is left IN_PROGRESS, all downstream phases wait on it, and
   the gate turn misleadingly shows `TALE APPROVED` + `lost`.

The same race hit `0t1--gate` (PlanApproval, 2026-09-26): housekeeping settled an
in-flight auto-approved tale as `lost`; that creator survived only because the lock
freed within 5s. The race window is every `%auto` gate's inline command execution, so it
will keep recurring as epics run many `%auto #plan` phases.

The missing-bundle check has no notion of "creation still running". The precise signal
already exists: every gate-turn creation (including request-id replays) runs inside
`_create_gate_turn_transaction()`, which holds the lane lock from before member creation
until after the auto-settle / bundle-path recording. A process crash releases the flock,
so "lane lock free + still no bundle" really does mean the creation is gone.

This is host-side process coordination (flock files), so it stays in Python; no
`sase-core` change is needed.

## Changes

### 1. Shared lane-lock helper — new `src/sase/gate_turn/lane_lock.py`

- Move the lane-lock path derivation out of `transaction.py` (`_gate_lane_lock_path`)
  into this module. The flock file it resolves to MUST stay byte-identical to today's:
  `log_file_lock(base)` flocks `base.with_name(f".{base.name}.lock")` where
  `base = sase_projects_dir() / project / "artifacts" / "ace-run" / f".gate-turn-{key}"`
  and `key = sha256(f"{project}\0{lane}".encode()).hexdigest()[:32]`. Keeping the path
  stable matters because dev-update code swaps leave old-code runners alive next to
  new-code housekeeping; both must exclude each other on the same file.
- Provide:
  - a blocking context manager used by the creation transaction (it may keep delegating
    to `log_file_lock(base)`), and
  - a non-blocking `try_gate_lane_lock(project_name, lane)` context manager that yields
    `True` when it acquired the same flock file (`LOCK_EX | LOCK_NB`,
    `O_CREAT | O_RDWR`, mode `0o600`, parent dir created) and `False` when another open
    file description holds it; it releases on exit.
- Switch `_create_gate_turn_transaction()` to the blocking helper. Update
  `tests/gate_turn/test_transaction_gate_intent.py`, which currently monkeypatches
  `transaction_module._gate_lane_lock_path`, to patch the new seam instead.

### 2. Reclaim never settles an in-flight creation as lost — `src/sase/gate_turn/reclaim.py`

- Route both early `lost` branches of `_reclaim_one()` ("gate bundle unreachable" and
  "gate bundle unreadable") through one public helper, e.g.
  `settle_lost_unless_creating(record, reason)` (public name + `__all__` entry, since
  `cancel.py` will import it; Symvision flags cross-module use of `_private` names).
- Helper behavior:
  1. `with try_gate_lane_lock(record.project_name, record.lane) as acquired:` — if not
     acquired, a creation or replay transaction is running on this lane: do nothing and
     report "creating" (reclaim returns `None`, same as a still-pending gate; the next
     tick re-evaluates).
  2. While holding the lane lock, re-read the member from disk with
     `read_gate_turn_marker(record.project_name, record.artifacts_dir)`. The chop acts
     on a snapshot read several seconds earlier, so the snapshot record can be stale. If
     the fresh record is missing or already terminal, do nothing. If the fresh record
     now has a bundle path that is a directory and `load_and_verify_bundle()` succeeds,
     do nothing and report "reachable" (the next pass classifies it normally).
  3. Otherwise settle the **fresh** record `lost` with the given reason, still inside
     the lane lock, and report "settled".
- Lock order is lane lock → `.gate_followup.lock`, identical to the creation transaction
  (lane lock, then `settle_gate_turn`'s follow-up lock). No code path takes the
  follow-up lock and then the lane lock, so this cannot deadlock. Reclaim only ever
  _tries_ the lane lock, so it never blocks on a slow creator.
- Keep `GateTurnReclaimSummary` / chop payload keys unchanged (the chop output contract
  tests pin them); deferral counts as neither `lost` nor an error.

### 3. Same guard for explicit cancel — `src/sase/gate_turn/cancel.py`

`cancel_gate_turn()` documents that it mirrors reclaim's checks and has the same
unconditional "bundle unreachable → lost" branch (reachable from `sase gate cancel` and
the TUI kill action). Use the shared helper there:

- "creating": return the record unchanged (still pending; the CLI/TUI already treat an
  unchanged record as "nothing cancelled").
- "reachable": continue down the normal cancel path with the fresh record (a bundle now
  exists, so cancel it properly instead of calling it lost).
- "settled": return the settled record.

### 4. A live creator waits longer for its own settlement lock — `handoff.py` / `settlement.py`

Even with reclaim fixed, a creator-live settle can still briefly contend on
`.gate_followup.lock` — e.g. `_create_gate_turn_transaction()` settles an auto gate a
second time after `_resolve_auto_gate()` already made it terminal, and the reconcile
phase of the same chop holds that lock while diagnosing terminal gates (including a
possible ~7s snapshot refresh). A creator-live settle runs in the agent runner after the
gate's selected commands may already have produced irreversible side effects, so a 5s
timeout turns a benign wait into a failed run.

- Give `with_gate_followup_lock()` an optional `timeout` keyword defaulting to
  `FOLLOWUP_LOCK_TIMEOUT_SECONDS` (5s, unchanged for chop/TUI/answer paths).
- Add `CREATOR_LIVE_FOLLOWUP_LOCK_TIMEOUT_SECONDS = 60.0` next to it and have
  `settle_gate_turn()` pass it when `creator_live=True`. This covers every creator-live
  caller (`transaction.py`, `notification_gates/service.py`,
  `gate_turn/agent_handoff.py`).
- Leave `publish_gate_turn_terminal_state()`, reclaim, reconcile, and resume on the 5s
  default.

### 5. Docs

Update the `gate_turn_reclaim` descriptions in `docs/notifications.md` (the "hourly
`gate_turn_reclaim` housekeeping job settles pending turns..." paragraph) and
`docs/axe.md` (the "`gate_turn_reclaim` job is the backstop" paragraph): reclaim skips a
turn whose gate creation is still running (its session's gate-creation lock is held), so
an `%auto` gate that is still executing its selected commands is never marked lost; a
turn whose creator died before recording a bundle is still settled lost on a later pass.

## Tests

Add to `tests/gate_turn/test_reclaim.py` (reuse `gate_turn_home` / `make_gate_turn`
fixtures) and new focused test modules as needed. Holding the lane lock from the test
via a separate `open()` + `fcntl.flock` is enough: flock locks belong to open file
descriptions, so the helper's non-blocking attempt fails even inside the same process.

- Reclaim leaves a pending member with no bundle path untouched (`None`, still pending,
  no `done.json`, no follow-up lock contention) while the lane lock is held.
- The same member is settled `lost` / "gate bundle unreachable" once the lane lock is
  free (crashed-creator behavior preserved).
- The "gate bundle unreadable" branch is also deferred while the lane lock is held.
- A stale snapshot record without a bundle path whose on-disk meta now points at a valid
  bundle is not settled lost.
- Incident reproduction: drive `create_gate_turn()` for an `%auto` gate and, from inside
  the gate's option execution (monkeypatch the execution seam), run
  `reclaim_pending_gate_turns()` against a freshly loaded snapshot. Assert reclaim did
  not settle the member and the creation returns with the member settled `answered`.
- `lane_lock` pins the flock file path format (`..gate-turn-<32 hex>.lock` under the
  project's `artifacts/ace-run/`) and matches what the creation transaction locks.
- `cancel_gate_turn()` returns an in-flight member unchanged, cancels normally when the
  fresh record has a reachable bundle, and still settles a truly bundle-less member
  lost.
- `settle_gate_turn(creator_live=True)` acquires the follow-up lock with the 60s
  timeout; non-creator-live settlement keeps 5s.

## Validation

Follow the `lint_and_test` reference memory (`sase memory read lint_and_test.md`) before
finishing, including `just check` run through `sase tool run`. Fix any Symvision
findings for the new module and helper per the `symvision` reference memory.
