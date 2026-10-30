---
tier: tale
title: Fix agent_state_refresh timeouts and scheduler wedging
goal:
  The agent_state_refresh job finishes the 2,470-hood sase project inside its budget,
  and no killed or frozen tick can pin rotation to one project.
size: medium
proposed_by: bbugyi200.athena.168
proposed_with: claude/opus
create_time: 2026-10-30 10:32:31
status: wip
---

# Plan: Fix `agent_state_refresh` timeouts and scheduler wedging

## Problem

The 2026-10-30 10:07 scheduler error digest shows the `checks/agent_state_refresh` job
timing out (120 s, SIGKILL, no output) on every tick since 09:22, plus two one-off
`publication/agent_publication` errors at 09:07.

### Diagnosis (measured on this host)

- **Root cause: quadratic manifest reads.** The `gh_sase-org__sase` agents sidecar owner
  manifest (`users/bbugyi200/machines/athena/manifest.json`) has 2,470 hoods (481 KB).
  `_collect_published_rows` in `src/sase/agents_sync/publication_state_repair.py` reads
  that manifest once, then calls `_read_validated_snapshot` for every hood, and that
  helper re-reads and re-validates the whole manifest just to look up one entry. One
  read costs about 52 ms, so collection alone is 2,471 × 52 ms ≈ 130 s, before any
  repair work and outside the 90 s internal work budget (the deadline is only checked in
  the resolve/plan loops). `py-spy dump` of a live tick showed it inside
  `read_owner_manifest` ← `_read_validated_snapshot` ← `_collect_published_rows`. With
  manifest reads memoized, a read-only preview of the same project finished in about 38
  s (collection about 4 s, local-evidence inventory about 30 s), and full
  `load_validated_publication` takes about 6 s.
- The same per-hood re-read pattern exists in `_plan_manifest_and_pages_degraded`, which
  loops over every manifest hood, and in `_plan_state_payload`, which re-reads once per
  changed hood (at most 5, so minor).
- **Why it never recovers and starves other projects.** The tick is SIGKILLed before
  `_run_locked` in `src/sase/agents_sync/agent_state_refresh.py` persists anything.
  `~/.sase/axe/lumberjacks/checks/agent_state_refresh.json` still says
  `last_selected = gh_bobs-org__bob-cli` and has no `gh_sase-org__sase` checkpoint, so
  the sase_core fair-rotation policy picks `gh_sase-org__sase` again on every tick.
  actstat, tailnet, and bob-cli never get refreshed again. Each killed tick also holds
  the sase agents-sidecar lock for two minutes (sase publication drains report
  `lock_busy`).
- **Latent sibling bug (reproduced by simulation).** `_run_locked` maps every report
  without `error`/`skip_reason` to a SUCCESS outcome with
  `next_cursor=report.next_cursor`. Several repair paths freeze the window on purpose
  with `next_cursor=None, pass_complete=False` and no error:
  - every changed row withheld or failed (the `if not payload and not updated:` branch)
  - payload over the byte budget
  - `apply_payload_batched_atomic` failure

  sase_core rejects that request with "successful incomplete outcome requires
  next_cursor". The tick then returns `scheduling_failed` without persisting, and the
  same project is reselected forever, which is the same starvation.

- **Rust policy facts** (probed through `advance_agent_run_state_schedule`):
  - A FAILURE outcome keeps the cursor, increments `failure_count`, sets
    `retry_after_ms` to `now + 60 s × 2^(n-1)` (bounded), and rotation moves on.
  - SUCCESS requires `next_cursor` unless `pass_complete`.

  No sase-core change is needed. The policy stays in Rust; this plan only changes which
  outcome Python reports and when it persists it.

### Out of scope: the two `agent_publication` errors

These came from a one-time drain of a legacy backlog. Nothing needs to change in code:

- The primary checkout fast-forwarded at 09:06. The editable install then picked up the
  new background publication job, and that job drained old outbox rows.
- actstat's request for hood `sase-xe` (created 2026-09-11) was retired terminally by
  design ("no publishable runs").
- bob-cli's `bbugyi200.athena.161` requests were created before 66b15496f9 started
  recording `artifacts_dir`. They were quarantined after three attempts.
- Every publication run after 09:07:30 reported no further errors for these rows, and
  current enqueue paths record `artifacts_dir`.

Do not touch the publication outbox or its code in this plan.

## Changes

### 1. Linear published-row collection (`src/sase/agents_sync/publication_state_repair.py`)

- Add a helper that validates one hood's snapshot against an already-read manifest
  entry, for example `_validated_snapshot_for_entry(repo_root, owner, hood, entry)`. It
  must raise the same
  `AgentsSyncFormatError(f"hood {hood!r} has no published snapshot")` when the entry is
  missing and otherwise return `previous_snapshot(repo_root, owner, hood, entry)`, so
  digest and identity validation and error text stay identical.
- `_collect_published_rows`: compute `entries = manifest.by_hood()` once from the
  manifest it already reads, and use the helper. The owner manifest must be read exactly
  once per collection. Keep per-hood exception isolation, `bad_hoods`, and diagnostics
  unchanged.
- `_plan_manifest_and_pages_degraded`: read the on-disk owner manifest once before the
  loop and use the helper for every unchanged, non-skipped hood. Changed hoods are
  already in `healthy`. Do not validate unchanged hoods against the rewritten manifest's
  new digests; use the on-disk entries exactly as `_read_validated_snapshot` does today.
- `_plan_state_payload`: read the owner manifest once before the hood loop and use the
  helper. Keep the deadline check and per-hood failure handling.
- Then delete `_read_validated_snapshot`, or make it a thin wrapper if a caller still
  needs it. Symvision flags unused private symbols.
- Behavior must be unchanged apart from the number of manifest reads. Keep the file's
  net line count flat or lower; it is already well past the `toobig` thresholds.

### 2. A killed tick backs off its project (`src/sase/agents_sync/agent_state_refresh.py`)

- In `_run_locked`, after resolving `target` and `checkpoint` and **before** calling
  `repair`, record a pessimistic pre-charge:
  - Call `advance_agent_run_state_schedule` with the same `wires`, `now_ms=moment`,
    `last_selected=outcome_identity`, and a FAILURE outcome for the selected identity
    (`next_cursor=None`, `pass_complete=False`).
  - Apply the returned `updated_checkpoint` and `last_selected` to a copy of
    `persisted`, then `_write_state` it.
  - If that policy call raises, return `scheduling_failed` exactly as the existing code
    does, without running repair.
  - If only the pre-charge write fails, append a diagnostic and continue; the final
    write reports persistence failure as today.
- After `repair` returns, compute the real transition exactly as today, from the
  **original** `wires` and not the pre-charged state, and persist it. A tick that
  completes must persist the same state as the current code: no double-counted
  `failure_count`, and the same cursor and `last_selected`.
- Factor "apply a schedule transition to `_PersistedState`" into one small helper shared
  by the pre-charge and the final write, instead of duplicating the checkpoint-update
  block.
- Effect: if the process is SIGKILLed mid-repair, the next tick sees that project with
  `failure_count + 1`, a future `retry_after_ms`, its cursor kept, and `last_selected` =
  that project. Rotation moves on to the other projects, and the project is retried with
  bounded backoff.

### 3. Frozen windows retry with backoff instead of wedging the scheduler

- In `_run_locked`, classify a report with no `error`/`skip_reason` but
  `next_cursor is None and not pass_complete` as a FAILURE outcome (cursor kept,
  backoff) instead of SUCCESS.
  - Append the diagnostic
    `f"{target.project_key}: repair window did not advance; retrying with backoff"`.
  - Return `reason="window_not_advanced"`.
  - Do not add to `errors`. Project failures today are reported through `reason` and
    `failed`, not `errors`; keep that convention.
- Leave the repair service's frozen-window reports and the
  `sase agent sync --repair-states` CLI contract unchanged.

### 4. Docs and descriptions

- `docs/contribute/agents-sidecar-sync.md`, section "Scheduled state convergence and
  owner-local backfill": state that a tick killed by the job timeout counts as a failed
  attempt (the project backs off, keeps its cursor, and rotation continues), and that a
  window that cannot advance (byte limit, every changed row withheld or failed) retries
  with backoff.
- `src/sase/default_config.yml`, `axe` → `checks` → `agent_state_refresh` description:
  add the same two facts briefly. Keep `timeout: "2m"`.
- Update the module docstrings of `src/sase/agents_sync/agent_state_refresh.py` and
  `src/sase/scripts/sase_chop_agent_state_refresh.py` where they describe crash
  semantics.
- Run `just fix` so any generated docs that derive from config descriptions are
  refreshed.

## Tests

- `tests/agents_sync/test_publication_state_repair.py`: add a regression test showing
  that collection reads the owner manifest a constant number of times regardless of hood
  count.
  - Seed two sidecars with the existing `_SeedBuilder`, one with 2 hoods and one with 6.
  - Wrap `sase.agents_sync.v2_io.read_owner_manifest` with a counting monkeypatch. The
    repair module imports it at call time, so patching the module attribute works.
  - Run a preview on each sidecar and assert both counts are equal and small.
  - Existing tests such as `test_corrupt_hood_does_not_block_sibling` (degraded path)
    and `test_byte_budget_freezes_cursor` must pass unchanged.
- `tests/agents_sync/test_agent_state_refresh.py`:
  - **Killed tick**: a `repair_one` stub raises a test-local `BaseException` subclass,
    which escapes the `except Exception` isolation the same way a SIGKILL stops work.
    Assert that it propagates and that the state file shows the selected project with
    `failure_count == 1`, `retry_after_ms > now`, its prior cursor kept, and
    `last_selected` = that project. Assert that the next tick, with a stub that returns
    success, selects a different project.
  - **Frozen window**: a stub returns a report with `next_cursor=None`,
    `pass_complete=False`, and no error. Assert `reason == "window_not_advanced"`,
    `errors == 0`, and persisted state with `failure_count == 1` and the prior cursor
    kept, and that the next tick rotates to another project. Today this returns
    `scheduling_failed` and persists nothing.
  - **Completed ticks unchanged**: the existing rotation, backoff, cursor, and
    end-of-pass tests must pass unmodified.

## Verification

- `sase tool run check`. Run `just install-venv` first if the workspace venv is stale.
- Read-only live smoke test from the workspace venv:
  `time .venv/bin/sase agent sync --repair-states -p sase --state-limit 100 --json`.
  Preview makes no network calls, no sidecar writes, and no checkpoint changes.
  - Before the fix, collection alone took about 130 s; expect well under 60 s.
  - If it still takes over 60 s, profile it and file a follow-up with `/sase_new_task`
    (the roughly 30 s scoped inventory loader is the known candidate). Do not widen this
    plan's scope.
- Do **not** run `--apply`, hand-edit
  `~/.sase/axe/lumberjacks/checks/agent_state_refresh.json`, or reinstall SASE globally.
  The global install is editable from the primary checkout. Once this lands there, the
  next `checks` tick selects `gh_sase-org__sase` and should finish inside the 90 s
  budget. The other projects then resume normal rotation without manual cleanup.
