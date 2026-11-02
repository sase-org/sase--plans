---
tier: tale
title: Redo the lost sase-1qt landing repairs and close the backlog epic
goal:
  The required Rust cleanup-policy boundary, spawn-only sudo fixture locks with
  deterministic proofs, and a startup import footprint under the unchanged cap are on
  master. sase-1qt then closes normally without losing this turn's work.
size: medium
decisions:
  chain_continuation:
    ask:
      Should this tale propose the next backlog epic from its code step, as the backlog
      contract asks?
    choices:
      defer_to_owner:
        Repairs land safely; tale only refreshes the epic-6 draft and the owner proposes
        it
      propose_after_backup:
        Chain keeps moving; tale backs up diffs then proposes, the step that lost work
        last time
    default: defer_to_owner
    why:
      The gate bug that destroyed the last tale's work (sase-1pj.7.1) is still unfixed
    answer: defer_to_owner
proposed_by: bbugyi200.athena.sase-1qt.land
proposed_with: claude/opus
bead: sase-1qt
decided_by: auto
create_time: 2026-11-02 07:02:03
status: wip
---

- **BEAD:**
  [sase-1qt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qt/README.md)

# Redo the lost sase-1qt landing repairs and close the backlog epic

Epic `sase-1qt` (task backlog epic 5) was closed on 2026-11-02 03:39Z, but none of its
landing repairs are on master. The relaunched lander reopened it. This tale redoes the
repairs and closes the epic in the same turn. There is no other land agent, and nothing
resumes `sase-1qt.land` after this tale. Do not author another tale to finish this work.

## What happened and what stands

Read
`sase bead read sase-1qt -r "Need the landing audit, lost-repair evidence, and follow-up dispositions"`.
The notes that matter:

- **#2:** the land audit.
- **#3:** the as-built design and verification of the lost repairs. This is your primary
  design reference.
- **#4:** the earlier close summary.
- **REOPENED:** evidence of the loss, with an attachment.
- **FOLLOW-UP TRIAGE (relaunched lander):** this run's triage.

Also read the original contract:
`sase artifact read plan:202611/task_backlog_epic_5.md "Need the original landing contract and backlog rule"`.
Its final DECISIONS are `optional_scope = keep_five_seeds`: no adjacent feature or
memory changes.

**The seed fixes landed and stay landed. Do not redo them.**

- **Prompt selection** (`sase-1ia`, `sase-1l3`, `sase-1hy`, `sase-1i9`): sase
  `d26baecb05` and core `45c6a260`.
- **Bounded drain cleanup** (`sase-xo`): sase `3e6c19ee37` and core `f061f5f6`.
- **Sudo fixture publication lock and ETXTBSY backstop** (`sase-15h`): core `b0a63dc3`.

All three phase beads are closed. Those six task beads are closed and stay closed; do
not reopen them.

**The landing repairs were lost.** The earlier tale coder implemented and verified them,
closed the beads, then ran `sase plan propose` for `task_backlog_epic_6.md`. Its
provider was SIGTERMed. The EpicApproval gate then failed with "workspace claim is no
longer owned by the creating agent", so no finalizer ran. When the workspace was reused,
it was reset to `origin/master` and the linked sase-core clone was replaced. Only
`tests/agents_sync/test_publication_cleanup_policy.py` (17 tests) survived. It is
attached to the REOPENED note verbatim; restore it as your starting point for Part 1.

Current tips:

- sase `origin/master` is `5811e00b46`. Since the audited `d26baecb05`, only two docs
  commits landed (`dee9354664`, `5811e00b46`). `dee9354664`'s check recorded
  `tui_app_import_stays_under_startup_budget` as KNOWN-failing, which confirms the Part
  3 overrun is live on master.
- sase-core `origin/master` and `sase-core-revision.txt` are both `45c6a260`.
- Recheck drift in both repos before closeout and integrate anything relevant.

`sase-1qt` has no `parent_bead`; the predecessor plan link to epic 4 is not a parent. It
has no `--epic-symbol` entries.

## Environment first

The workspace venv may lack a usable `sase_core_rs`; import fails with "run
`just install-venv`". Open sase-core with `/sase_repo` and read its `AGENTS.md`. Then
rebuild the venv against the linked core with `just install-venv`. If that will take
minutes, hand it to `/sase_monitor`.

The wheel cache only accepts a clean core tree, so build before editing core, or rebuild
after core edits. Never use `just install` or `just install-dev`; global install is
human-only. Never run bare `cargo`. Read the `lint_and_test` memory before verification,
`tui` before Part 3, and `ace` before editing under `src/sase/ace/`.

## 1. Required Rust cleanup-policy boundary (sase, plus a core alias cleanup)

**Current defect.** Two layers fall back to a Python 1.0 default:

- `src/sase/core/agent_publication_recovery_publication.py::default_publication_cleanup_timeout`
  uses `optional_rust_binding`.
- `src/sase/agents_sync/publication_cleanup.py` wraps it in another `try/except`
  fallback to `_FALLBACK_CLEANUP_TIMEOUT_SECONDS`. It also replaces a non-positive
  timeout with that fallback and caps every override at the Python constant.

This violates the required-Rust boundary (decision `rust-core-required`). The as-built
design from note #3 passed verification; reproduce it:

- **Adapter.** `default_publication_cleanup_timeout` goes through `require_rust_binding`
  in the core adapter. The recovered tests patch
  `sase.core.agent_publication_recovery.require_rust_binding`, so the lookup must be
  reachable through that module.
- **Errors.** A missing, stale, or raising binding raises, both from the adapter and
  from `PublicationCleanupBudget`. Non-numeric, non-positive, or non-finite values raise
  `RuntimeError`.
- **Overrides.** Delete `_FALLBACK_CLEANUP_TIMEOUT_SECONDS` and every Python policy
  mirror. An override can only shorten the Rust-owned cap (`min`). A non-positive
  override raises `ValueError`.
- **Cleanup boundary.** In `src/sase/agents_sync/_commit_publication_locked.py`, build
  the budget inside the protected cleanup boundary. A policy-lookup failure is attached
  as a note to the original exception, and no cleanup runs with an invented default.
  Preserve all of the following:
  - drain timeout attribution stays primary;
  - the request stays queued with `attempts == 1`;
  - the sidecar lock is released.
- **Unchanged.** Keep one shared shrinking deadline across both cleanup families and all
  git subprocess and recovery attempts. Do not raise the one-second cap or any
  `elapsed < 5` assertion.
- **Core alias.** In the linked core, remove the
  `core_default_publication_cleanup_timeout` alias from
  `crates/sase_core_py/src/prelude.rs`, and use the domain module path in
  `crates/sase_core_py/src/vcs/mod.rs`. No binding contract changes, so no pin move is
  owed. If one does change, let host completion move `sase-core-revision.txt`; never
  invent a SHA.
- **Tests.** Restore the recovered test file and make all 17 tests pass. Adjust imports
  only where the module layout from Part 3 requires it, never the assertions.

## 2. Spawn-only sudo fixture lock and deterministic proofs (sase-core)

Work under `crates/sase_gateway/src/sudo_runner/`.

**Current defect.** `handoff.rs` (`sudo_status` and `spawn_internal_root_executor`)
calls `retry_on_etxtbsy(|| command.status())`. The `cfg(test)` fixture spawn lock in
`platform.rs::retry_on_etxtbsy` covers the whole closure, so the lock is held through
the child wait.

**As-built fix to reproduce:**

- **Retry helper.** Type `retry_on_etxtbsy` as a spawn (`FnMut() -> io::Result<Child>`).
  The guard covers one `Command::spawn` attempt and is released before any wait or
  backoff. A spawn that returned `Ok` is never retried, and only ETXTBSY is retried.
- **Status helper.** Add `status_with_etxtbsy_retry`, which spawns under the guard and
  waits outside it. Use it at both `handoff.rs` sites, preserving stdio, error mapping,
  exit-status semantics, and child reaping.
- **Test `.output()` calls.** Replace test closures that call `.output()` with a
  test-only `output_with_etxtbsy_retry`. Review every remaining caller for the same
  wait-under-lock pattern.
- **Regression `tests/spawn_lock.rs`.** Keep a real sudo fixture child alive on a
  handshake file, and prove that another fixture publication plus spawn completes
  meanwhile. Use a bounded `recv_timeout`, a self-expiring child, and drop-guard
  release; no sleeps. Mutation check: temporarily hold the lock through the wait and
  confirm the regression fails, then revert.
- **`tests/etxtbsy.rs`, exec-while-held test.**
  `exec_while_fork_child_holds_write_descriptor_fails_with_etxtbsy` must have a real
  child hold the inherited non-CLOEXEC writer, confirmed through a ready marker. Then
  close the parent's copy and assert ETXTBSY. Finally release and reap the child and
  prove the exec succeeds. Keep any raw-fork child branch async-signal-safe.
- **`tests/etxtbsy.rs`, retry test.** In `etxtbsy_retry_recovers_once_writer_closes`,
  close the writer inside the retry loop only after observing the first failed attempt.
  Use no sleeps or threads, and still assert that a retry happened.
- **Assertion messages.** Make the bare `result.is_ok()` assertions in sudo_runner
  `cli.rs` and `worker.rs` print the `Err` (9 sites), without weakening them.
- **Invariants.** Every executable writer still goes through
  `publish_executable`/`write_executable`; manifest, argv, and marker data stay data.
  Keep every argv, cwd, environment, timeout, cancellation, and ledger assertion.

## 3. Startup footprint under the unchanged cap (sase)

**Current defect.** The epic's eager cleanup imports pushed the TUI first-paint closure
over the strict cap. The base `9066546900` measured 12,161,158 bytes, `d26baecb05`
measures 12,166,370, and the cap is `< 12,162,484`. The as-built fix reached 11,893,483
bytes (2796 `sase.*` modules) by deferring these edges:

- **`src/sase/agents_sync/__init__.py`:** a PEP 562 lazy facade through
  `sase._lazy_exports`, keeping the same public names.
- **`src/sase/ace/tui/actions/agents_sync.py`:** import `sync_agents` at its use site.
- **`src/sase/artifact_links/projection/_stitch_rules.py`:** import
  `agents_sync.git.run_git` at its use site.
- **`src/sase/llm_provider/commit_finalizer_git_progress.py`:** import
  `AGENTS_SYNC_AUTO_COMMIT_TYPE` at its use site.
- **Deferred-module guard:** add `agents_sync.git`, `git_sync`, `git_sync_transaction`,
  `publication`, and `publication_cleanup`.

Keep the allowances, CPU bound, and total-module guard unchanged. Do not hide source
from the metric or remove assertions. Confirm that monkeypatch targets naming
`sase.agents_sync.<attr>` still work through the lazy facade (for example
`tests/ace/tui/test_log_panel_keymap.py`). Confirm that real publication callers still
run. Measure with `tools/tui_import_closure --diff d26baecb05` and attach the attributed
report to the epic note.

## 4. Verification

Use foreground `sase tool run` commands, or `/sase_monitor` for long ones. Never run
`just check-full`.

- **sase-core:**
  - the full sudo_runner module, then repeated runs under parallel load (the as-built
    run did 12 concurrent processes for 5 rounds alongside busy loops);
  - the focused Rust policy and binding tests if anything there changed;
  - `sase tool run check`.
- **sase:**
  - the bounded-drain module plus the new policy module, repeated under concurrent load
    with per-case timings;
  - the git-lock adapter tests;
  - the prompt cache tests and the four prompt-render files plus the siblings named by
    `sase-1ia`, `sase-1l3`, `sase-1hy`, and `sase-1i9`;
  - the full `tests/ace/tui/test_app_import_budget.py` and
    `tests/test_tui_import_closure.py`;
  - format, then `sase tool run check`.
- **Failures.** Fix any epic-caused failure. Record independently proven existing
  failures accurately without changing command outcomes. Two are already known and must
  not be fixed here:
  - the drain-accounting test-order leak, owned by `sase-j7`;
  - the cold-isolation keymap failure, task `sase-1qu`.

## 5. Closeout (same turn, before host commit)

1. Re-read `sase-1qt` and its notes. Recheck sase and sase-core drift since
   `5811e00b46`/`45c6a260` and integrate anything relevant.
2. Record a `LANDING REPAIRS (redo)` note on `sase-1qt` covering:
   - the boundary, lock scope, deterministic evidence, and closure numbers with the
     attached report;
   - the checks with their ToolRun IDs;
   - the limits.
3. Append a short correction note to `sase-15h` and `sase-xo` (they stay closed). Their
   close notes cited landing repairs that did not land on 2026-11-02; this tale redid
   them. Cite the redo note.
4. Run `sase bead epic-symbols sase-1qt`. For every entry, wire it up, privatize it,
   pragma it, or delete it. Re-key one only to a still-open bead that needs it.
5. Close the epic with
   `sase bead close sase-1qt --note "<verification, drift integration, follow-up outcomes>"`.
   Carry every follow-up disposition from the FOLLOW-UP TRIAGE notes:
   - wheel-cache proposal declined;
   - drain-accounting leak routed to `sase-j7`;
   - import-budget overrun resolved here as epic integration work;
   - lost-work data loss recorded as a DISCOVERED ISSUE on `sase-1pj.7.1`;
   - keymap cold failure filed as `sase-1qu`.

   Never use `--force`.

6. Run `just symvision`. Open the plans repo with `/sase_repo`. Set `status: done` in
   `202611/task_backlog_epic_5.md`, and in `202611/finish_1qt_landing.md`, whose scope
   this tale completes. Declare that repo in `/sase_final`.
7. Re-run `sase bead read sase-1qt -r "Need the parent link"`. No parent is expected; if
   one appeared, handle it per the landing rules and stop on ambiguity.

## 6. Backlog continuation

The approved contract asks the lander to keep the backlog chain moving. A draft of the
next epic already exists outside the repo at
`~/.sase/plans/202611/task_backlog_epic_6.md` (seeds `sase-1oy`, `sase-1em`, `sase-zo`,
`sase-1pw`, `sase-1ce`); it was never approved. Refresh it against the current tracker:

- Re-list non-closed tasks with `sase bead list -t task -n 0`.
- Close only verified duplicates or obsolete beads, per the contract.
- Re-rank with the selection-health full-run records.
- Keep at most 5 phases and 5 tasks, the same continuation rule, and the
  `optional_scope` owner opt-out.
- Validate with `sase plan validate`.

> [!decision] chain_continuation = defer_to_owner
>
> Do not run `sase plan propose`, `/sase_run`, or any other gate-creating command from
> this tale. Leave the refreshed, validated draft at its path. Note on `sase-1qt` that
> it awaits the owner's proposal, then finish normally through `/sase_final` so host
> completion commits sase, sase-core, and the plans repo.

> [!decision] chain_continuation = propose_after_backup
>
> After closeout and before proposing:
>
> 1. Attach these to `sase-1qt` as one note: a patch of each repo's full working-tree
>    change, including untracked files (sase, linked sase-core, and the plans repo).
> 2. Confirm that each attachment applies cleanly with `git apply --check` on a scratch
>    copy.
> 3. Propose the refreshed epic 6 with `sase plan propose` as the very last action.
