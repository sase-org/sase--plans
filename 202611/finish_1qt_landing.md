---
tier: tale
title: Finish the sase-1qt landing repairs and close the backlog epic
goal:
  Required Rust cleanup policy, spawn-only fixture locks, and the integrated startup
  budget are verified; sase-1qt and its resolved tasks close normally and the backlog
  chain continues.
size: medium
proposed_by: bbugyi200.athena.sase-1qt.land
proposed_with: codex/gpt-6-astra
bead: sase-1qt
create_time: 2026-11-01 21:14:55
status: done
---

- **PARENT:**
  [202611/task_backlog_epic_5.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_5.md)
- **BEAD:**
  [sase-1qt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qt/README.md)

# Finish the remaining sase-1qt landing repairs

Complete the remaining defects found by the land audit, then close epic `sase-1qt` and
continue its approved task-backlog chain. This is one bounded coding turn across sase
and its linked sase-core checkout. The original three phases are complete except for the
defects below; do not redo their implementation or create another land agent.

## Scope and verified starting point

Read `sase bead read sase-1qt -r "Need the landing audit and follow-up dispositions"`
and
`sase artifact read plan:202611/task_backlog_epic_5.md "Need the original landing contract"`.
The final DECISIONS are `optional_scope = keep_five_seeds`: no adjacent feature or
memory changes. All three phase beads are closed. Five seed tasks remain ready:
`sase-1ia`, `sase-1l3`, `sase-1hy`, `sase-xo`, `sase-15h`; related `sase-1i9` is also
ready. The lander read every phase and task note and reviewed the implementations. Read
the retained audit and reproduction logs with
`sase artifact read file:explicit:10d848c95ce1680403f13082 "Need the landing evidence"`.

The audited tips are sase `d26baecb051927df38a8bfe4683b7d509371b1d1` and sase-core
`45c6a2601bb8fdaa48180ba97caa6339fe0a35c5`. The latter is the sase core revision pin and
contains the following changes:

- Prompt selection: sase `d26baecb05`, core `45c6a260`. The shared Rust selector
  excludes raw and finalizer prompts, preserves workflow-step preference, and ranks
  integer nanosecond mtimes with deterministic path ties. Python uses the required
  binding. Phase `.1` measured the host filesystem trigger and recorded athena 128/128
  and apollo 118/118 focused passes, including `sase-1i9`, with a forced-tie regression
  that fails on reverted code. Preserve this implementation.
- Drain cleanup: sase `3e6c19ee37`, core `f061f5f6`. Rust enforces recovery and caller
  deadlines. Exception cleanup shares a one-second budget; phase `.2` measured 15.6448
  seconds before and about 2.4 seconds after under injected index-lock contention, with
  9 bounded-drain tests and 20 core git-lock tests passing.
- Sudo fixture: core `b0a63dc3`. Atomic executable publication and a test-only spawn
  lock prevent fixture writer/spawner races, with an ETXTBSY-only retry backstop. Phase
  `.3` recorded 56 focused passes, 15 parallel repetitions and a passing core check.
  Preserve every argv, cwd, environment, timeout, cancellation and ledger assertion
  while correcting the lock scope and regression coverage below.

Intervening sase commits since epic creation/base `9066546900` were `a79207993c` (staged
handoff reclamation), `b8145cb51a` (clan running-member jump), and `82cf081fc4`
(publication observation module split). The first has no behavioral overlap; the split
preserves its facade and is footprint-neutral. Clan navigation still uses the existing
prompt loaders but adds startup footprint. There were no unrelated core commits after
`e60f3e16`. Recheck later drift before closeout.

The epic has **no `parent_bead`**. Its plan's predecessor link to backlog epic 4 is not
a parent-bead relationship. The audit found no `--epic-symbol` entries.

## 1. Restore the required Rust cleanup policy boundary

In `src/sase/core/agent_publication_recovery_publication.py`,
`default_publication_cleanup_timeout` currently uses `optional_rust_binding`, catches
binding errors and returns a Python `1.0`. `src/sase/agents_sync/publication_cleanup.py`
adds another fallback and independently caps with `_FALLBACK_CLEANUP_TIMEOUT_SECONDS`.
This violates the required Rust backend boundary. A lander probe with the binding
missing returned `1.0` at both layers; with a mocked Rust default of `0.25`, an explicit
Python timeout of `0.8` still produced `0.8`.

- Use `require_rust_binding` through a thin core adapter. Remove the Python policy
  mirrors and optional-binding path for this operation. Any override can shorten the
  Rust-owned cap but cannot enlarge it. Keep timeout validation/policy in Rust if new
  deterministic behavior is needed; Python may own monotonic clocks and signals.
- Handle policy lookup failure without replacing an already-active publication
  exception. In `_commit_publication_locked.py`, budget construction currently occurs
  outside the protected cleanup `try`; move it inside the cleanup error boundary so
  missing/stale binding diagnostics attach to the original exception and no cleanup runs
  with an invented default. Direct adapter calls must expose the required-binding error.
  Preserve the queued request, timeout attribution and released sidecar lock.
- Keep one shared shrinking deadline across both cleanup families and all git
  subprocess/recovery attempts. Do not raise the one-second cap or the five-second
  elapsed assertions. Add meaningful regressions for missing/raising binding,
  original-exception preservation and an override that cannot exceed the Rust cap.
- In core, remove the newly introduced `core_default_publication_cleanup_timeout` alias
  from `crates/sase_core_py/src/prelude.rs`; use the domain module path in `vcs/mod.rs`,
  per that checkout's instructions. Do not expand this into a general prelude cleanup.

Open sase-core with `/sase_repo` and read its `AGENTS.md` before edits. If a binding
contract changes, update Rust policy/wire/binding tests and Python callers together; let
host completion move `sase-core-revision.txt` to the actual containing commit. Never
fabricate a SHA or wait for this turn's own commit before closing the epic.

## 2. Finish the sudo fixture race fix and its proof

In `crates/sase_gateway/src/sudo_runner/handoff.rs`, both `sudo_status` and
`spawn_internal_root_executor` call `retry_on_etxtbsy(|| command.status())`. The
`cfg(test)` guard in `platform.rs::retry_on_etxtbsy` encloses the entire closure, so
these calls retain the fixture spawn lock while waiting for the process to finish. That
contradicts the intended per-spawn-only lock and can delay other fixture writers or
spawns past their timing budgets.

- At these two sites, guard/retry only `Command::spawn`, then wait outside the guard.
  Preserve stdio, error mapping, exit-status semantics and child reaping. Review all
  remaining uses, including test `.output()` calls, for the same wait-under-lock
  mistake. Keep the ETXTBSY-only bounded backstop and never retry an executed command.
- Add a synchronized regression proving another fixture publication/spawn can proceed
  while one of these children is still alive. Coordinate with pipes/channels or
  handshake files, with bounded cleanup on failure; do not rely on fortunate sleeps.
- Repair
  `tests/etxtbsy.rs::exec_while_fork_child_holds_write_descriptor_fails_with_etxtbsy`.
  It currently retains the write handle in the parent, while its `Command::spawn` has
  already execed `/bin/sleep` and closed CLOEXEC descriptors. It therefore proves only
  that a parent writer blocks exec. Build an isolated, synchronized child-held
  descriptor reproducer: confirm the child holds the writer, close the parent's copy,
  assert ETXTBSY, release the child and prove execution succeeds. If using raw fork,
  keep the child branch async-signal-safe and always reap it.
- Replace the two sleeps in `etxtbsy_retry_recovers_once_writer_closes` with a handshake
  that releases the writer only after observing the first failed attempt; keep the
  assertion that a retry really happened. A heavily delayed test thread must not make
  the first attempt succeed and fail the regression itself.
- Add the error value to the bare `result.is_ok()` assertion identified in the
  `sase-15h` notes, without weakening it. Verify every executable writer still uses
  `publish_executable`/`write_executable`; manifest, argv and marker data stay data.

## 3. Integrate the startup footprint with intervening changes

The new cleanup code is eagerly imported during TUI startup. This is a real epic
integration failure, not an unrelated follow-up. The lander measured:

| Tree or contribution                           | Normalized source bytes |
| ---------------------------------------------- | ----------------------: |
| Base `9066546900`                              |              12,161,158 |
| Epic cleanup additions in the imported closure |                  +3,036 |
| Intervening clan navigation in that closure    |                  +2,269 |
| Epic prompt-selection adapter                  |                     -93 |
| Current `d26baecb05`                           |              12,166,370 |
| Unchanged strict `<` cap                       |              12,162,484 |

The accounting is from `tools/tui_import_closure --diff 9066546900`, with zero module
count change. Cleanup contributors are `agents_sync.git` (+1,381),
`_git_sync_ops_payload` (+507), `git_sync_transaction` (+461),
`core.agent_publication_recovery_publication` (+236), `prompt_archive.git_ops` (+222),
`core.agent_publication_recovery` (+138), and `git_lock_retry` (+91).

The phase check `a17ced1a91cff09ea4654b4a30fdf7e6` already measured 12,164,194 source
bytes against this cap on base `a79207993c` plus the cleanup patch. Its KNOWN witness
`02ee4a669bcb10115e81429c94035534` was an older, different assertion (2861 modules
against strict `<2861`). It does not establish that this footprint overrun is unrelated.
Do not change either recorded exit code or classification retroactively.

Defer publication/git cleanup implementation off the first-paint import path with
use-site imports or thin facades. Re-measure to demonstrate the new closure is below the
unchanged cap while the real publication callers still execute correctly. Keep the
deferred-module list, CPU bound and total-module guard. Do not raise allowances, remove
assertions or hide source from the metric. Read the `tui` memory before changes to this
performance-sensitive path, and `ace` before edits under `src/sase/ace/`.

## 4. Verify the final integrated tree

The lander's checkout initially had an unbuilt editable Rust package. To reproduce the
unrelated accounting leak and measure source footprint it installed the historical
`b0a63dc3` cache wheel locally. That wheel deliberately lacks the new bindings. Refresh
the checkout's venv against the actual linked core before product verification; do not
use the historical cache entry as an after-wheel. Use `/sase_monitor` for known-long
environment setup or verification commands and wait for the handoff command to exit.

- Run focused Rust policy/binding tests if changed and the full sudo_runner module,
  including its timing tests, repeatedly under parallel load. Never use bare cargo.
- Run the bounded-drain module and git-lock adapter tests; keep elapsed, attempts,
  queued-request, later-retry and lock-release assertions.
- Run the prompt cache tests and the four primary prompt-render files plus siblings
  named by `sase-1ia`, `sase-1l3`, `sase-1hy`, and `sase-1i9`. Preserve phase `.1`'s
  recorded cross-host evidence; if prompt behavior changes, repeat the original plan's
  scratch-snapshot apollo verification and clean up the scratch tree.
- Run the full `tests/ace/tui/test_app_import_budget.py` and
  `tests/test_tui_import_closure.py`; retain the before/after attributed closure report.
- Read `lint_and_test` memory, format, then run `sase tool run check` in each changed
  repository. This is `just check`, not `just check-full`. Do not run check-full. Fix
  epic-caused failures. Independently proven existing failures are triaged using project
  policy and recorded accurately, without changing command outcomes.

The drain-accounting test-order leak was triaged by the lander to active global-state
epic `sase-j7` as a `DISCOVERED ISSUE`, as recorded on both epics. No duplicate task was
created. Do not silently make it part of these repairs. The cold module has 9 failures/3
passes; importing `sase.core.git_lock_facade` before `pytest.main` gives 12/12 on
identical source. The scanner test patches every binding lookup and a late facade import
retains that fake after teardown. This is independent of cleanup.

## 5. Complete this epic's closeout and backlog continuation

This tale has no land agent and nothing automatically resumes `sase-1qt.land`. The coder
must perform this step in the same turn as the final code, before host commits. Do not
wait for this turn's SHA, push or CI result, and do not author another tale just to
close these already-verified repairs.

1. Re-read `sase-1qt`, all descendants and notes, the five seed tasks and `sase-1i9`,
   and the original linked plan. Recheck commits that landed after the audited tips and
   reconcile relevant drift. Record the repaired boundary, lock scope, deterministic
   race evidence, startup closure numbers, checks, and limits on the epic.
2. Close each of `sase-1ia`, `sase-1l3`, `sase-1hy`, `sase-xo`, and `sase-15h` with
   `sase bead close <id> --note "<its fixing commits and verified before/after evidence>"`.
   Also close `sase-1i9` normally on the recorded apollo pass plus current focused pass.
   Do not close any independently owned adjacent task such as `sase-15e`/`sase-15f`.
3. Include every follow-up disposition from the lander's `FOLLOW-UP TRIAGE` note in the
   close note: decline the historical wheel as a cache bug, retain the accounting leak's
   recorded route to `sase-j7`, and resolve the import-budget proposal as this epic's
   own integration work. Duplicate accounting proposals in phase `.2` notes 2 and 3 are
   one issue. Its later verification notes refer to the same proposals. Phase `.3`
   explicitly proposed no new follow-up.
4. Run `sase bead epic-symbols sase-1qt`. For every entry, wire it up, privatize it, add
   a justified non-test consumer pragma, or delete it under Symvision policy. Re-key
   only when a named still-open later bead actually needs the exemption.
5. Close normally with
   `sase bead close sase-1qt --note "<verification, drift integration, follow-up outcomes>"`.
   If symbols remain, finish their cleanup and retry. If a phase is incomplete, finish
   or reopen it; never use `--force` to advance this successful landing.
6. Run `just symvision`. Open the plans repo with `/sase_repo`, read the artifact with
   `sase artifact read plan:202611/task_backlog_epic_5.md "Need the completed plan for closeout"`,
   and set `status: done` in `202611/task_backlog_epic_5.md` under the printed plans
   checkout. Declare that changed repo in `/sase_final`.
7. Run `sase bead read sase-1qt -r "Need the parent link"` again. At audit time it has
   no parent, so no ancestor close is owed. If that link changed, handle it
   deliberately: a phase parent closes only after its scope is verified; a plan parent
   requires its prior landing note, every descendant/note, linked-plan readiness and
   post-child drift rechecked. Retire its symbols, close normally, run symvision, mark
   its plan done and repeat direct plan ancestors. Stop on ambiguity/incompleteness and
   note the blocker on that parent. Never force a successful ancestor close.
8. Preserve the original approved backlog-chain contract: re-list every non-closed task
   using `sase bead list -t task -n 0`. Close verified duplicates only as `superseded`
   with their canonical bead named and noted; close obsolete tasks only with
   fixing-commit, removed-code or passing-test evidence. Never close a task owned by an
   active epic or a snoozed task early; dormant tasks stay open. Rank the five most
   important remaining tasks by fleet-wide agent/CI cost, landing cost,
   security/data-loss risk and corroboration, then cost to fix. Use selection-health
   full-run recurrence records, excluding runs with many failures and failures in their
   own changed files. Prior leads from the original plan were `sase-1q3`, `sase-zo`,
   `sase-1em`, `sase-1oy`; these are leads, not a fixed selection. Write a new epic with
   at most five phases and five tasks, preserving this same backlog-continuation rule
   and an owner opt-out for optional feature/memory work. Use `/sase_memory_write`
   before proposing any memory edit. Complete `/sase_plan`'s validate-with-explain,
   edit, revalidate and propose loop. Propose only after this epic's closeout is
   finished, since proposal hands off the turn. If no tasks remain, finish normally
   through `/sase_final`.
