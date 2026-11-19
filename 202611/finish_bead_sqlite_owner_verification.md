---
tier: tale
title: Finish bead SQLite ownership regression coverage and verification
goal:
  Prove existing mirrors are left untouched, complete verification of the landed
  retirement, and close sase-1tc.4.
size: medium
proposed_by: bbugyi200.athena.sase-1tc.4
proposed_with: codex/gpt-6-astra
bead: sase-1tc.4
create_time: 2026-11-19 08:04:55
status: wip
---

- **PARENT:**
  [202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_cmdsub_retries_launches.md)
- **BEAD:**
  [sase-1tc.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.4.md)

# Finish the single-owner proof and verification for sase-1tc.4

Complete the remaining work for phase `sase-1tc.4` of `sase-1tc`. The production mirror
retirement already landed in `5b79f0f5e37237e07b48088c4079dc04082453e1`. Keep that
implementation and finish its missing legacy-mirror regression and verification. The
accepted epic decision is **mirror_fix = retire_mirror**.

The governing designs are `plan:202611/backlog_epic_cmdsub_retries_launches.md` and
`plan:202611/retire_bead_sqlite_mirror.md`; the underlying task is `sase-1kp`. No new
lock protocol, feature flag, memory edit, or backend redesign is authorized. Close only
the assigned phase, not the parent epic or underlying task.

## Why this remainder fits one coding agent

The previous agent's notes record completed production changes and passing focused
suites, followed by an unfinished check. Inspection at `6e1fc2536f` confirms the
implementation is committed, the tree is clean, and the remaining changes are bounded
regression-test improvements. A medium tale can finish them; there is no unfinished
multi-subsystem implementation to split into a child epic. Long verification belongs in
a monitor, with a concrete completion follow-up.

Verified evidence:

- `project.py` and `_project_store.py` no longer import SQLite or cache connections;
  exports use the required Rust binding. Legacy SQLite helpers live under
  `tests/test_bead/legacy_sqlite/`. Work-plan callers and performance fixtures have
  already been migrated. The ownership documentation already describes the Rust lock
  file and preservation of preexisting mirror bytes.
- ToolRun `3ead2af4b580392e4aa10ba628693c74` is terminal, **signaled**, with
  `terminal_cause=stop_requested`. Its recorded reason is
  `starter agent sase-1tc.4 ended without joining`. Twenty stages passed, but
  `test (scoped)` is incomplete. This is neither a green check nor evidence of a product
  regression. Do not join or poll this terminal run.
- `test_legacy_store_adoption_never_opens_beads_db` explicitly starts with no
  `beads.db`: its subprocess writes only JSONL and config. It does not cover the
  approved plan's adoption of an existing SQLite mirror or preservation of its inode.
- `test_new_store_never_opens_beads_db` covers real Rust mutations and exports, but
  neither single-owner scenario exercises `sync()`.
- `test_beads_db_has_single_rust_owner` contains the vacuous assertion
  `"_conn" not in text or "_conn_cache" not in text`, after already forbidding
  `_conn_cache`. It cannot independently prevent `_conn` from returning.
- The coordinated writer test checks unique child IDs and retained updates.
  `test_cli_work_contention_regressions.py` already separately proves exclusion under a
  held mutation lock and the configured lock deadline. Reuse that coverage.
- Phase note 3 records a clean-base reproduction of pytest fixture-order pollution when
  a `test_bead` file follows `test_autonomy_roles.py`. Retain that evidence; do not
  treat it as a new mirror-retirement defect without contrary evidence.
- `sase bead epic-symbols sase-1tc.4` currently reports no entries.

## 1. Confirm the baseline and ownership

Read the phase and `sase-1kp` through `sase bead read`, and both designs through
`sase artifact read`. Check whether this remainder has since landed before editing. Read
`lint_and_test.md` through `/sase_memory_read`.

Repeat `sase agent list` and `sase bead list -t plan` before editing. At planning time,
the other live work belonged to `sase-1tz.1.1`, covering notification coalescing, disk
inventory, workspace settlement, and monitor ownership; the next `sase-1tc` phase and
lander were waiting. No overlap with the proposed ownership regression files was
identified. If a live agent now owns a needed file, follow the parent epic's
coordination rule: record the overlap as a `PROPOSED FOLLOW-UP:` on this phase and a
coordination note on the owning epic.

## 2. Finish the missing regression contracts

Keep changes centered on `tests/test_bead/test_beads_db_single_owner.py` and
`tests/test_sqlite_single_owner.py`, with a small test helper if warranted.

1. Preserve the existing JSONL-only adoption test. Add a distinct existing-mirror case,
   or parameterize the fixture so both cases remain covered. Prepare a real WAL-mode
   SQLite `beads.db` in a separate completed subprocess using the existing test-only
   legacy helpers. Write its matching config and compatibility JSONL. Explicitly close
   the SQLite connection and wait for that subprocess to finish before running any Rust
   operation in the parent.
2. Record the mirror's device/inode and bytes before adoption. Under the stdlib SQLite
   connection guard, use the real `BeadProject` and Rust binding to open, read, mutate,
   allocate a child, export, sync, close, and reopen the store. Assert canonical data
   reflects the mutation and allocation, while the old mirror inode and bytes remain
   unchanged. The mirror is deliberately stale after the mutation; reads must still
   report canonical state.
3. Where covering old `-wal`/`-shm` preservation, retain valid fixture sidecars in a
   controlled subprocess snapshot, or test untouched sentinel files separately. Do not
   leave a legacy SQLite connection alive in the same process as Rust, and do not
   present synthetic sidecars as a reproduction of a live WAL database. Use a disposable
   git repository for real sync, preserving the existing exclusions for `beads.db*`; no
   network or live-store operations are needed.
4. Exercise sync in the new-store guarded scenario too. Keep the guard active across
   initialization/adoption, reads, mutations, export, sync, and context exit; assert
   zero Python SQLite opens of `beads.db`. Do not mock Rust operations.
5. Replace the vacuous `_conn` check with an actual check that the relevant classes have
   no `_conn` member/property, without banning unrelated names by substring. Keep the
   repository-wide SQLite opener allowlist and every other Rust-owned filename guard.
   Verify that the guard still rejects the retired connection path; do not build a new
   general-purpose static analyzer.

Retain the existing concurrent-writer regression and run the existing mutation-lock
contention tests. Add no timing sleeps, weakened deadlines, skips, or suppressions.
Ensure subprocesses are joined and cleaned up on failures.

Production changes should only be needed if these tests expose an in-scope defect.
Preserve the lock inode and Rust lock protocol. Any shared domain repair belongs in
`sase-core`, opened first through `/sase_repo`; never reintroduce a Python fallback. No
Rust change is expected. Read the relevant reference memories before any unanticipated
documentation, CLI, or TUI edit.

## 3. Complete verification and leave usable evidence

Run focused coverage for the new tests, existing lock contention, storage/lifecycle,
explicit export failure propagation, stale-JSONL event authority, sync exclusions, local
child allocation, work-plan/rendering, CLI dependency removal, and legacy JSONL/schema
fixtures. Include `tests/test_autonomy_roles.py` in a separate pytest invocation from
the grouped `tests/test_bead` files because of the recorded preexisting ordering hazard.
Keep the 1,500-row legacy-read case.

Smoke-test the changed performance fixture builders on small disposable corpora:
`bench_bead._write_project`, `bench_bead._write_work_plan_project`, and
`bench_artifacts_first_paint._write_bead_corpus`. Verify expected counts and dependency
shape through the normal Rust-backed reads, with no leftover `fixture.sqlite`
connection. No full performance sweep or TUI screenshot update is needed.

Run `just fix` or at least `just fmt`, then the normal `sase tool run check`. Ensure the
diff-scoped selection includes the already-landed retirement, rather than selecting only
this tale's test edits: inspect `tools/select_tests --explain` with
`--base 5b79f0f5e37237e07b48088c4079dc04082453e1^`, and use the supported
`SASE_CHECK_BASE` setting for the check. Preserve all normal selection thresholds and
budgets; do not run `just check-full`.

Choose `/sase_monitor` before starting long verification. The previous check ran for
over twenty minutes before it was stopped. Stage a monitored check with a follow-up that
reads its terminal result, diagnoses failures, records evidence, and closes this phase
when its requirements are met. Wait for the monitor-start command itself to exit and
report `Staged`, then finish through `/sase_final`. Do not end a turn with a raw
verification subprocess still running, cancel a running check to move it, or leave the
phase open merely because a monitor is needed. A staged successor must own the remaining
completion work.

For failures, inspect retained ToolRun evidence. Fix failures caused by this phase. If a
failure reproduces identically on the appropriate clean base, record the command,
revisions, matching failure, and any existing tracking bead in a `PROPOSED FOLLOW-UP:`
note, then complete the phase anyway as the user instructed. Use an isolated checkout
inside the allowed workspace for base comparison. Do not claim the prior
`blocked_unpublished` core-floor observation is still current without rechecking it,
bypass binding validation, or expand this tale into a release/infrastructure repair.
Create no task beads.

## Completion

Record the landed implementation SHA, new test changes, exact passing tests and ToolRun
result, and any independently proved baseline failures on `sase-1tc.4`. The original
pre-fix witness proved main-file POSIX lock loss; it did not prove SIGBUS, corruption,
or loss of locks on other inodes. The post-fix proof is that supported Python paths
cannot open the mirror alongside Rust, while real mutations and lock exclusion still
work. Do not claim a manually opened legacy mirror becomes safe with the unchanged Rust
lock.

Append `TASK RESOLVED: sase-1kp — <implementation and verification evidence>` to this
phase. Immediately before closing, run `sase bead epic-symbols sase-1tc.4` and resolve
or re-key any entries to a still-open owner. Close only `sase-1tc.4` using
`sase bead close sase-1tc.4 --note "<what was verified>"`. Leave `sase-1tc` and
`sase-1kp` for the epic lander. Finish through `/sase_final`; never change the assigned
phase status by hand or reopen it as part of normal work.
