---
tier: epic
title: Survivable bead-store locking for concurrent bead work launches
goal: 'Task-bead worker launches succeed while an epic launch (or any other bead writer)
  is in flight, and an epic launch that blocks on a lock reports what it is waiting
  for instead of stalling silently for minutes.

  '
phases:
- id: store_lock
  title: Fair, configurable store-lock waits in sase-core
  depends_on: []
  size: medium
  description: 'store_lock: replace the 2s hardcoded try-lock poll in the Rust bead-mutation,
    task-store, and prompt-stash locks with a capped-backoff wait whose bound is a
    long, env-overridable default, and record holder identity so an expired wait names
    its blocker.

    '
- id: work_timing
  title: Durable stage timing for sase bead work
  depends_on: []
  size: medium
  description: 'work_timing: promote the bead-work launch timer to a durable telemetry
    sink, instrument the currently unmeasured plan-to-epic span, and warn on slow
    stages so a multi-minute launch is attributable after the fact.

    '
- id: launch_lock
  title: Bounded and instrumented plan-launch and store-write locks
  depends_on:
  - work_timing
  size: medium
  description: 'launch_lock: give the unbounded epic-plan launch flock a deadline,
    holder identity, and an actionable expiry error, and record every store-write-lock
    wait through the durable timing sink instead of only warning on fail-open.

    '
- id: launch_retry
  title: Contention-resilient task and epic bead launches
  depends_on:
  - store_lock
  size: small
  description: 'launch_retry: classify store-lock expiry as a distinct retryable failure,
    retry bead-work preclaims within a bounded budget, and report contention to the
    operator as a wait rather than a bare exit-code-1 error.

    '
- id: contention_tests
  title: Concurrency regression coverage for bead launches
  depends_on:
  - store_lock
  - launch_lock
  - launch_retry
  size: small
  description: 'contention_tests: add regression tests that drive concurrent bead
    mutations and a task launch overlapping an epic launch, asserting no lock expiry
    and no half-claimed beads.'
proposed_by: bbugyi200.athena.r5
create_time: 2026-08-01 09:03:42
status: wip
bead_id: sase-da
---

- **PROMPT:** [202608/prompts/bead_store_lock_contention.md](prompts/bead_store_lock_contention.md)
- **BEAD:** [sase-da](https://github.com/sase-org/sase--beads/blob/main/pages/sase-da/README.md)

# Plan: Survivable bead-store locking for concurrent bead work launches

## Problem

Two related failures were reported, both reproduced from the 2026-08-01 12:34–12:47 UTC incident.

### 1. Task-bead worker launches fail under ordinary concurrency

`sase bead work sase-d8 --yes-to-all` exited 1 with:

```
Error: lock_timeout: timed out after 2000ms waiting for bead mutation lock
  <beads_dir>/beads.db for store <beads_dir>
```

The bound comes from `BEAD_MUTATION_LOCK_TIMEOUT: Duration = Duration::from_secs(2)` in
`crates/sase_core/src/bead/mutation.rs` (constant block near the top of the file, consumed by `with_bead_mutation_lock`
/ `lock_bead_mutation_with_timeout`). It is hardcoded with no config or environment override.

The wait itself is a userspace poll: `FileExt::try_lock_exclusive` in a loop with a 10–30 ms jittered sleep. That is
unfair — a freshly arriving writer races equally with one that has already waited seconds — so tail latency is dominated
by lost races, not by how long the lock is actually held.

Measured against a copy of the real store (2541 beads, 485 event streams, 7.9 MB event store), running N concurrent
writers each performing 15 mutations:

| concurrent writers | median mutation | p95     | longest successful wait | expiries |
| ------------------ | --------------- | ------- | ----------------------- | -------- |
| 2                  | 92 ms           | 938 ms  | 1459 ms                 | 0 / 30   |
| 3                  | 92 ms           | 111 ms  | 1499 ms                 | 1 / 45   |
| 4                  | 93 ms           | 234 ms  | 1436 ms                 | 3 / 60   |
| 8                  | 92 ms           | 1169 ms | 1961 ms                 | 16 / 120 |

The critical section is ~92 ms, yet successful waits already reach 1.96 s at 8 writers. **No slow lock holder is
required to break the 2 s ceiling — ordinary healthy concurrency does it.** An epic launch performs ~19 bead mutations
and then fans out to 8 agents, which is exactly this regime.

The same 2 s / 10 ms / 20 ms idiom is duplicated in `crates/sase_core/src/tasks/store.rs` and
`crates/sase_core/src/prompt_stash/store.rs`, so both share the defect.

### 2. The epic launch stalls, and nothing records where

Detached task `bp80vzzxsfsn` (`sase bead work <plan>.md --yes-to-all`) ran 12:34:52 → 12:47:27 UTC, 12 m 35 s.
Reconstructed from commit timestamps and durable launch telemetry:

| time (UTC)          | evidence                                                         |
| ------------------- | ---------------------------------------------------------------- |
| 12:34:52            | task starts                                                      |
| 12:34:54            | plans repo: `Archive approved plan clan_summary_view_hints`      |
| 12:34:54 → 12:45:29 | **10 m 35 s gap — no commit, no logged git op from any process** |
| 12:45:29            | plans repo: `Link approved epic plan to its bead`                |
| 12:46:17            | beads repo: `checkpoint approved epic graph sase-d9`             |
| 12:46:28 → 12:47:24 | 8 agents spawn (`agent_launch_spawn` records)                    |
| 12:47:27            | task finishes                                                    |

The work inside that 10 m 35 s gap was measured directly and is fast: `validate_plan_file` 2 ms, 1 epic + 7 phase
creates + 11 dependency adds 2.8 s total, `project_plan_header_sections` 218 ms. `~/.sase/logs/tui_git_ops.jsonl`
records no failing, push/fetch, or slow git operation in the gap (that log only records those categories), so the
process was doing neither network git nor slow git. Concurrent ACE tasks in the same window
(`kill 1 + dismiss 1 agents`, `kill my_feature`, `comprehensive-update updates`, `launch foo`) all hung and errored,
then cleared as the epic launch finished — a system-wide serialization event, not epic-specific compute.

The span between those two commits contains exactly two waits, and **both are effectively invisible**:

- `epic_plan_launch_lock` (`src/sase/bead/cli_work_from_plan_store.py`) takes `fcntl.flock(fd, LOCK_EX)` with **no
  timeout at all**, no holder record, and no logging. A wedged epic launch blocks every later plan launch on that store
  forever.
- `store_git_write_lock` (`src/sase/sdd/_git_contention.py`) waits up to
  `DEFAULT_WORKTREE_MUTATION_LOCK_TIMEOUT_SECONDS = 180.0`, then **fails open** with a warning that is not present in
  any log retained from the incident.

The exact wait that consumed the 10 m 35 s is **not attributable from the surviving logs**, and that is itself the
defect to fix: `agent_launch_spawn` and `tui_agent_launch` write durable per-stage records (which is how the
12:46:28–12:47:24 fan-out above was reconstructed), but `sase bead work` builds its `LaunchTimingRecorder` with
`durable=False` and debug-only logging (`_make_bead_work_timer` in `src/sase/bead/cli_work_handler.py`), and the entire
plan-archive → epic-creation → plan-link span in `src/sase/bead/cli_work_from_plan.py` has no stages at all. The one
operation that stalls for twelve minutes is the one that records nothing.

## Approach

Fix the proven cause (phase `store_lock`), remove the unbounded wait and make the remaining waits observable (phases
`work_timing`, `launch_lock`), stop turning contention into a hard launch failure (phase `launch_retry`), and lock the
behaviour down with tests (phase `contention_tests`).

`work_timing` deliberately does not depend on `store_lock`: it is pure Python and should land in parallel so the next
stall is diagnosable even before the Rust change ships.

### Note for the phase agents that touch sase-core

Phase `store_lock` changes the sibling Rust core repo. Open it with `/sase_repo`
(`sase repo open sase-core -r "<reason>"`) and use the printed path as the only path for reads and writes. Dev installs
track the local checkout, so `just install` (or `just rust-install`) rebuilds `sase_core_rs` from it; see
`docs/rust_backend.md` for the published-wheel version window and when it must be widened. Do not reimplement store-lock
behaviour in Python.

## Fair, configurable store-lock waits in sase-core

Repo: `sase-core`. Files: `crates/sase_core/src/bead/mutation.rs`, `crates/sase_core/src/tasks/store.rs`,
`crates/sase_core/src/prompt_stash/store.rs`.

1. Add one shared helper module (for example `crates/sase_core/src/store_lock.rs`) that owns the file-lock wait for all
   three stores, replacing the three copies of the poll loop:
   - a `LockMode` (shared/exclusive) and a wait function that loops on `try_lock_*` until a deadline, returning the
     elapsed wait on success and a typed timeout carrying the elapsed wait, the lock path, and the recorded holder on
     expiry;
   - **capped exponential backoff with jitter** instead of a flat 10–30 ms: start at ~5 ms, grow by roughly 1.5–2×, cap
     at ~250 ms, and never sleep past the deadline. This keeps handoff fast when uncontended and stops a long wait from
     burning CPU in a tight poll.
   - a small env-parsing helper that reads a float number of seconds, ignoring unset, unparseable, and non-positive
     values in favour of the default.
2. Raise the bounds and make them overridable:

   | lock          | env var                           | old default | new default |
   | ------------- | --------------------------------- | ----------- | ----------- |
   | bead mutation | `SASE_BEAD_MUTATION_LOCK_TIMEOUT` | 2 s         | **600 s**   |
   | task store    | `SASE_TASK_STORE_LOCK_TIMEOUT`    | 2 s         | 120 s       |
   | prompt stash  | `SASE_PROMPT_STASH_LOCK_TIMEOUT`  | 2 s         | 120 s       |

   600 s for bead mutations is the bound the user asked for: a launch must be able to wait out a full epic launch. The
   other two stores are on interactive paths and were not implicated in this incident, but 2 s is indefensible for them
   too.

   Leave `effort_override.rs` and `runner_limit_override.rs` at their 250 ms bounds — those are sub-second interactive
   toggles where failing fast is the correct behaviour. Do not migrate them in this phase.

3. **Holder identity.** While the lock is held, best-effort write `{pid, operation, acquired_at}` to a sibling holder
   file (for the bead store, something like `<beads_dir>/.bead-mutation-lock.holder`), remove it on release, and include
   its contents in the expiry error message. Requirements:
   - never fail or slow a mutation because the holder file could not be written or removed;
   - add the file to the bead store's `.gitignore` and make sure `sase bead doctor` does not report it as unexpected
     store content.
4. **Do not print from the library.** The core is loaded inside the ACE TUI process, so stray stdout/stderr would
   corrupt the display. Instead surface the wait to callers as data: include the elapsed lock wait (`lock_wait_ms` or
   equivalent) in the mutation outcome wire so the Python layer can log it, and leave the holder file as the out-of-band
   signal an external observer can read.
5. Keep the error `kind` string `"lock_timeout"` unchanged — Python classifies on it, and phase `launch_retry` depends
   on that classification being stable.
6. Tests in `sase-core`: a wait that outlives the old 2 s bound and still acquires once the holder releases;
   env-override parsing including the rejected forms; an expiry message that names the holder pid; and backoff that
   respects the deadline rather than overshooting it.

Run the sase-core repo's own checks, then `just install && just check` in this repo so the Python side is exercised
against the rebuilt extension.

## Durable stage timing for sase bead work

Repo: `sase`. Files: `src/sase/bead/cli_work_handler.py`, `src/sase/bead/cli_work_from_plan.py`,
`src/sase/bead/cli_work_entry.py`, `src/sase/bead/cli_work_task.py`, `src/sase/agent/launch_timing.py`.

1. Build the bead-work recorder with `durable=True` in `_make_bead_work_timer` (and in the equivalent fallback recorder
   constructed inside `launch_task_bead_work`) so records reach the durable JSONL sink the agent-launch path already
   uses, with `operation: "bead_work"`. Records must land regardless of `SASE_BEAD_WORK_TIMING`; that env var keeps its
   existing role of promoting the same records to info-level logs.
2. Instrument the currently unmeasured span. Thread the existing recorder into `work_from_plan_file` /
   `_work_from_plan_file_locked` rather than creating a second one, and add stages covering at least: acquiring the
   plan-launch lock, `require_plan_store_health`, `archive_plan_file`, the archived-plan commit, `BeadProject` open,
   epic + phase + dependency creation, header projection, and the plan-link commit. These are the stages that would have
   named the 10 m 35 s gap.
3. Add a slow-stage signal: when a stage exceeds a threshold (30 s is a reasonable default), log a warning naming the
   stage, its elapsed time, and the bead/plan it belongs to, and mark the record so it is greppable. A twelve-minute
   launch should leave an obvious trail.
4. Keep the recorder's failure mode best-effort — telemetry must never break a launch.
5. Tests: assert the durable sink receives a `bead_work` summary with the new stage names for both the epic-from-plan
   and task-bead paths, and that a stage over the threshold emits the warning.

## Bounded and instrumented plan-launch and store-write locks

Repo: `sase`. Files: `src/sase/bead/cli_work_from_plan_store.py`, `src/sase/sdd/_git_contention.py`.

1. `epic_plan_launch_lock` must stop blocking forever. Replace the bare `fcntl.flock(fd, LOCK_EX)` loop with a
   deadline-bounded `LOCK_EX | LOCK_NB` poll using the same capped-backoff shape as phase `store_lock`, defaulting to a
   generous bound (900 s is appropriate — it must outlast a legitimate slow epic launch) and overridable through an env
   var such as `SASE_EPIC_PLAN_LAUNCH_LOCK_TIMEOUT`.
   - Write holder metadata (`pid`, plan file, `started_at`) into the lock file while held, and truncate it on release.
   - On expiry raise `PlanFileWorkError` naming the holding pid and plan, with a resume hint, so the ACE Tasks panel
     shows which launch is blocking instead of an indefinite `Working...`.
   - Preserve the documented lock ordering: launch lock first, then the store write lock.
2. Record every `store_git_write_lock` acquisition through the phase `work_timing` sink: the waited milliseconds, the
   `op` label, the repo root, and whether it was acquired or failed open. Warn above a threshold on acquisition (not
   only on fail-open), and include the recorded holder when one is available. The existing fail-open behaviour and the
   `SASE_SDD_STORE_WRITE_LOCK_TIMEOUT` override stay as they are — this phase makes the wait visible, it does not change
   the policy.
3. Tests: a held plan-launch lock expires with an error naming the holder rather than hanging; the lock is still
   released on both success and exception paths; store-write-lock waits appear in the durable timing records with their
   `op` label.

## Contention-resilient task and epic bead launches

Repo: `sase`. Files: `src/sase/bead/cli_work_task.py`, `src/sase/bead/cli_work_handler.py`,
`src/sase/bead/_project_mutations.py` (or a small shared helper next to it).

1. Classify a core `lock_timeout` error as its own retryable condition instead of letting it escape as an opaque
   `Error: lock_timeout: ...` that exits 1. Add a narrow retry around the bead-work preclaim mutations
   (`proj.update(...)` in `launch_task_bead_work`, `proj.mark_ready_to_work` and `proj.preclaim_epic_work` in
   `launch_epic_bead_work`) with a bounded attempt budget and backoff. With phase `store_lock` landed this is
   belt-and-braces, so keep the budget small and do not retry anything that already mutated the store.
2. Make the operator-facing message a wait, not a crash: while retrying, report that the launch is waiting for the bead
   store and name the holder from the holder file when it is readable. When the budget is genuinely exhausted, fail with
   an actionable message that names the holder and gives the exact resume command.
3. Verify the failure path leaves no half-claimed bead: a launch that gives up during preclaim must roll back exactly as
   the existing `rollback_task_work_launch` / `rollback_work_launch` paths do.
4. Tests: a simulated `lock_timeout` on the first preclaim attempt succeeds on retry; an exhausted budget rolls back the
   claim and reports the holder.

## Concurrency regression coverage for bead launches

Repo: `sase`. Files: `tests/test_bead/` (new module alongside the existing claim/sync tests).

1. Add a concurrency regression test that runs several concurrent bead mutations against a seeded store and asserts zero
   `lock_timeout` failures. Seed enough beads that a mutation is not trivially fast, keep the writer count and iteration
   count modest so the test stays quick and deterministic under CI parallelism, and drive the timeout through the new
   env var so the test pins behaviour rather than wall-clock luck.
2. Add a test for the reported scenario: a task-bead launch that overlaps an in-flight epic launch completes rather than
   exiting 1, and leaves the task bead correctly claimed.
3. Add a test that a plan launch blocked on the launch lock fails with the holder-naming error after its bound rather
   than hanging, using a short env-var-supplied bound.
4. These tests must not touch the real `~/.sase` state or the project's real bead store; use temporary stores as the
   existing bead tests do.

## Out of scope

- The `beads.db` lock file is also the store's tracked (empty) database file, so a git operation that replaces its inode
  while the lock is held can split the lock. The code comments already acknowledge this. It is a real hazard but a
  separate change — file a task bead rather than folding it in here.
- Rebuilding the legacy SQLite compatibility mirror from `issues.jsonl` currently fails with a foreign-key violation on
  the real store (`rebuild_from_jsonl` → `import_from_jsonl` → `db.create_issue`). It did not cause this incident and
  the mutation paths no longer depend on the mirror, but it should be filed as a task bead.
- Reducing the number of bead mutations an epic launch performs, or batching them into one transaction, is a worthwhile
  optimization but is not required to fix the reported failures.
