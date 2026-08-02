---
tier: tale
title: Serialize epic approvals with one project-scoped launch lock
goal:
  Concurrent epic approvals for one project serialize on a single lockfile instead of racing each other's sidecar
  clones, so a second approval waits for the in-flight launch rather than failing with an unusable-store error or
  hard-resetting the shared beads checkout out from under it.
proposed_by: bbugyi200.athena.rz
create_time: 2026-08-02 10:00:58
status: wip
---

- **PROMPT:**
  [prompts/202608/epic_approval_lock.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/epic_approval_lock.md)

# Plan: Serialize epic approvals with one project-scoped launch lock

## Problem

Approving two epics for the same project close together fails. The screenshot artifact
`.sase/artifacts/home/tmp/screenshots/20260802_092455.png` shows an ACE `Plan response: epic` task failing with:

```
approved epic plans store is unusable: could not materialize beads sidecar repository
sase-org/sase--beads from git@github.com:sase-org/sase--beads.git at
/home/bryan/projects/github/sase-org/sase/sase/repos/beads: SDD repository
/home/bryan/projects/github/sase-org/sase/sase/repos/beads needs integration but has tracked worktree
changes; commit or restore those changes and retry; machine-managed recovery failed: managed reset could
not be verified: worktree or index is not clean after reset.
```

while the task list directly above it still shows `detached Epic launch · stored_prompt_duality ... Working...`. The
second approval failed _because_ the first epic launch was still running.

## Root cause

Every epic approval for a project shares one set of sidecar clones hanging off the project's primary workspace
(`<primary>/sase/repos/beads`, `<primary>/sase/repos/plans`). The approval pipeline touches those clones in two
different processes, and neither one holds a lock that the other respects:

1. **Host preflight (ACE / gate / Telegram process).** `prepare_epic_launch` (`src/sase/_plan_approval_epic.py:50`)
   calls `require_epic_launch_store_health(cwd)` (`src/sase/bead/cli_work_from_plan_store.py:120`), which calls
   `resolve_beads_location(cwd, materialize=True)`. With `materialize=True` that runs `materialize_sdd_store` and
   `ensure_beads_sidecar_clone` (`src/sase/bead/cli_location.py:93-101`), which reaches
   `ensure_sidecar_sdd_clone(..., strict=True)` → `_pull_sdd_clone` (`src/sase/sdd/_store_link.py:75`) and integrates
   the shared beads clone — fetch, rebase, and on failure a machine-managed hard reset.
2. **Detached launch process.** `sase bead work <plan>` runs `work_from_plan_file`, which resolves and materializes the
   very same clones in `_resolve_context` (`src/sase/bead/cli_work_from_plan.py:117`) and only afterwards acquires
   `_epic_plan_launch_lock(store.repo_root)` (`src/sase/bead/cli_work_from_plan.py:216-220`) before archiving the plan
   and writing beads.

So the existing launch lock is acquired _after_ the materialization it needs to protect, and the host preflight never
acquires it at all. While launch A holds the lock and is writing uncommitted bead files into
`<primary>/sase/repos/beads`, approval B's preflight integrates that same clone, sees A's writes as tracked worktree
changes, escalates to a managed reset, and `verify_managed_reset` (`src/sase/sdd/_repository_recovery_git.py:163-179`)
then fails because `status_porcelain` is still dirty — A is still writing.

The visible symptom is the failed approval. The more serious problem is that the escalation path is a **hard reset of a
shared clone that another process is mid-transaction on**, so a losing race can discard an in-flight launch's
uncommitted bead state.

`materialization_lock(primary)` (`src/sase/sdd/_store_adoption.py:28`) does not help: it only serializes materialization
against other materialization, never against an in-flight launch that is already past materialization.

## Approach

Keep exactly one lock for the whole epic-approval pipeline, key it on something both processes can resolve _before_ any
materialization happens, and make both halves of the pipeline take it.

Reuse and widen the existing `epic_plan_launch_lock` in `src/sase/bead/cli_work_from_plan_store.py` rather than adding a
second lock, so the documented lock ordering (launch lock → store write lock → transient Git retry) stays intact and
there is only one thing to reason about.

Three changes make it correct:

1. **Re-key the lock on the project's primary-workspace anchor** instead of `store.repo_root`. `store.repo_root` is only
   known _after_ `_resolve_context` has already materialized the clones, which is the exact ordering bug. The primary
   workspace directory is resolvable from any cwd with no I/O against the sidecar clones, is identical for the host
   preflight (`cwd` is the primary workspace, from `resolve_epic_launch_cwd`) and for the detached launch (it runs with
   that same cwd), and is per-project so unrelated projects never serialize against each other.
2. **Acquire it before `_resolve_context`** in `work_from_plan_file`, so materialization/integration is inside the same
   critical section as the bead writes it races with today.
3. **Acquire it in `require_epic_launch_store_health`**, so a host preflight waits for an in-flight launch instead of
   hard-resetting the clone out from under it.

### Deliberate decision: the preflight waits briefly, then defers rather than failing

The detached launch is the thing that actually mutates the store, so it keeps the long, generous wait (900 s) and its
existing "timed out, here is the resume command" failure. The host preflight is only a fail-fast check that runs on an
ACE worker thread; blocking it for 15 minutes would tie up a Textual worker and leave the user staring at `Working...`.

So the preflight waits on a shorter bound (default 120 s) and, if the lock is still held, **skips the health check and
submits the detached launch anyway** — logging which holder it deferred to. Nothing is lost: the detached launch re-runs
the identical health check under the lock, and reports failures through the existing epic-launch completion notification
with a resume command. This preserves fail-fast for the normal uncontended case and removes the false failure under
contention.

The user-visible effect is what was asked for: a second epic approval waits for the first instead of racing it, and if
it must give up it says so with a resume command rather than corrupting the shared clone.

### Not a Rust core change

The `rust_core_backend_boundary` rule was considered. The epic-approval pipeline, SDD store materialization, and all of
its sibling locks (`epic_plan_launch_lock`, `store_git_write_lock`, `materialization_lock`) live in Python in this repo;
`sase-core`'s `store_lock.rs` is `pub(crate)` and is not exposed through `sase_core_rs`. Every frontend reaches epic
approval through the same two Python entry points changed here (`prepare_epic_launch` and `sase bead work`), so
serialization holds for any future frontend without a wire change. Do not add a Rust binding for this.

## Implementation

### 1. Public workspace anchor resolution — `src/sase/bead/cli_location.py`

Add a public helper next to the existing private context resolution:

```python
def resolve_workspace_anchor(cwd: Path | None = None) -> Path | None:
    """Return the primary workspace that owns this cwd's shared SDD clones."""
```

It resolves `cwd` (defaulting to `Path.cwd()`), calls the existing `_resolve_workspace_context`, and returns
`context.primary` or `None` when no workspace context is discoverable. Do not change `_resolve_workspace_context`
itself; its `pytest_path_is_sandboxed` guard and marker/scan fallbacks must keep their current behavior.

### 2. Anchor + re-keyed lock — `src/sase/bead/cli_work_from_plan_store.py`

Add:

```python
def epic_launch_lock_anchor(cwd: str | Path | None = None) -> Path:
    """Return the canonical serialization anchor for one project's epic launches."""
```

It returns `resolve_workspace_anchor(cwd)` when available, otherwise the expanded/resolved `cwd`, always
`.resolve(strict=False)`-normalized so two processes that reach the same directory by different paths agree.

Change `_epic_plan_launch_lock_path` to take that anchor and always return
`sase_subdir("locks") / "epic-plan-launches" / f"{sha256(os.fsencode(anchor))}.lock"`. Delete the `rev-parse --git-dir`
branch: it exists only to place the lock inside the store repo's Git directory, which is precisely the store this lock
must be acquirable _without_ touching. Keeping the lock under `~/.sase/locks/` also guarantees it can never land in a
store worktree.

Change the `epic_plan_launch_lock` signature and semantics:

```python
@contextmanager
def epic_plan_launch_lock(
    anchor: Path,
    *,
    plan_file: str | Path | None = None,
    timeout: float | None = None,
    op: str = "epic plan launch",
    raise_on_timeout: bool = True,
) -> Iterator[bool]:
```

- It now yields `bool` (whether the lock was acquired) instead of `None`.
- `raise_on_timeout=True` (the default, used by the detached launch) keeps today's behavior exactly: raise
  `PlanFileWorkError` naming the holding pid, its plan file, its start time, and the resume command.
- `raise_on_timeout=False` yields `False` after the bound expires instead of raising, and logs a warning naming the
  holder and the `op` that gave up.
- Record `op` alongside `pid` / `plan_file` / `started_at` in the holder JSON, and include it in
  `_format_epic_plan_launch_holder`, so an ACE Tasks row tells the user whether a preflight or a launch is holding.
- Keep the existing bounded, jittered `LOCK_EX | LOCK_NB` backoff loop, the holder write/truncate on release, and
  `SASE_EPIC_PLAN_LAUNCH_LOCK_TIMEOUT` / 900 s as the default bound for the launch path.

Add a second bound for the preflight path:

```python
ENV_EPIC_APPROVAL_PREFLIGHT_LOCK_TIMEOUT = "SASE_EPIC_APPROVAL_PREFLIGHT_LOCK_TIMEOUT"
DEFAULT_EPIC_APPROVAL_PREFLIGHT_LOCK_TIMEOUT_SECONDS = 120.0
```

parsed by the same validation shape as `_epic_plan_launch_lock_timeout` (non-finite, zero, negative, and unparseable
values fall back to the default).

Add a re-entrancy guard in the same style as `_held_store_write_locks` in `src/sase/sdd/_git_contention.py:38`: a
`ContextVar[frozenset[Path]]` of held anchors, and raise a `RuntimeError` naming the anchor if the same execution
context tries to acquire an anchor it already holds. `flock` is per open-file-description, so a second `open()` +
`flock()` in one process would silently self-deadlock for the full timeout; failing loudly is the point. No current
caller nests, so this guard is purely a tripwire for future callers.

### 3. Take the lock before materialization — `src/sase/bead/cli_work_from_plan.py`

Restructure `work_from_plan_file` so a single `ExitStack` spans from before store resolution to the end of the function:

- Enter the stack immediately after plan validation succeeds.
- When `dry_run` is `False`, enter `_epic_plan_launch_lock(_epic_launch_lock_anchor(), plan_file=source_path)` on that
  stack inside the existing `timer.stage("plan_launch_lock")` stage, **before** the `timer.stage("store_context")` block
  that calls `_resolve_context`. Import `epic_launch_lock_anchor` alongside the lock in the existing
  `from sase.bead.cli_work_from_plan_store import (...)` block, following that block's `_`-prefixed alias convention.
- Remove the current acquisition at `src/sase/bead/cli_work_from_plan.py:216-220`. It must be _moved_, not duplicated —
  a second acquisition of the same anchor now trips the re-entrancy guard added above.
- Leave the dry-run path unlocked: `_resolve_context(dry_run=True)` passes `materialize=False` and writes nothing, so it
  neither needs nor deserves to block a real launch.
- Keep `_work_from_plan_file_locked` and every stage inside it unchanged; it simply now runs with the lock already held
  and with the materialization it depends on covered by the same critical section.

### 4. Take the lock in the host preflight — `src/sase/bead/cli_work_from_plan_store.py`

Rewrite `require_epic_launch_store_health(cwd)` to:

```python
with epic_plan_launch_lock(
    epic_launch_lock_anchor(cwd),
    timeout=_epic_approval_preflight_lock_timeout(),
    op="epic approval preflight",
    raise_on_timeout=False,
) as acquired:
    if not acquired:
        return  # an in-flight launch owns the shared clones; it re-checks under the lock
    location = resolve_beads_location(cwd, materialize=True)
    if location is not None and location.store is not None:
        require_plan_store_health(location.store)
```

Note the nesting order: the launch lock is outermost, and `require_plan_store_health` acquires `store_git_write_lock`
inside it — the ordering the module docstring already documents. Do not reverse it.

`prepare_epic_launch` (`src/sase/_plan_approval_epic.py:47-64`) needs no change: a deferred preflight simply returns
without raising, and the existing `SddMaterializationError` / `SddRepositoryHealthError` handling still converts a real
health failure into `epic_launch_failed` with a resume command.

### 5. Documentation

- `docs/configuration.md`, "SDD Git Operations" table (around line 2459): add rows for
  `SASE_EPIC_PLAN_LAUNCH_LOCK_TIMEOUT` (currently undocumented) and `SASE_EPIC_APPROVAL_PREFLIGHT_LOCK_TIMEOUT`.
- `docs/sdd_storage.md`, "Concurrency and Recovery" (line 169): add a short paragraph stating that epic approvals for
  one project serialize on a primary-workspace-keyed lock under `~/.sase/locks/epic-plan-launches/`, that the lock
  covers sidecar materialization as well as bead writes, and that a contended host preflight defers its health check to
  the detached launch instead of failing the approval.

## Tests

Extend `tests/test_sdd_git_contention.py` (it already owns `epic_plan_launch_lock` coverage) and
`tests/test_plan_approval_actions.py`. Update the existing `epic_plan_launch_lock` tests for the `bool` yield and the
anchor-keyed path.

1. **Anchor identity.** `epic_launch_lock_anchor` returns the same path for the primary workspace and for a numbered
   workspace of the same project, and different paths for two different projects.
2. **Cross-process exclusion on the anchor.** Keep the existing subprocess/multiprocessing exclusion test, re-pointed at
   an anchor; keep the "distinct anchors do not serialize" counterpart.
3. **Deferral, not failure.** With the lock held by another process,
   `epic_plan_launch_lock(..., raise_on_timeout=False, timeout=<small>)` yields `False` and does not raise; with
   `raise_on_timeout=True` it still raises `PlanFileWorkError` naming the holder and the resume command.
4. **Preflight defers under contention.** With the anchor lock held by another process,
   `require_epic_launch_store_health` returns without raising **and without calling `resolve_beads_location`**
   (monkeypatch it to record calls). This is the direct regression test for the screenshot.
5. **Preflight runs when uncontended.** With no holder, `require_epic_launch_store_health` calls
   `resolve_beads_location(materialize=True)` and propagates a raised `SddMaterializationError` unchanged.
6. **Lock is held across materialization.** In `work_from_plan_file`, monkeypatch `_resolve_context` to assert the
   anchor lock file is already flock-held (a probe process, or a non-blocking `flock` attempt from a child, must fail)
   when it runs. This is the regression test for the ordering bug.
7. **Re-entrancy guard.** Acquiring the same anchor twice in one execution context raises rather than deadlocking; use a
   short timeout so a regression fails fast instead of hanging the suite.
8. **Release on failure.** The lock is released when the launch raises, so a later launch acquires immediately (the
   existing exception-path test, retained).
9. **End-to-end approval regression.** With a simulated holder on the anchor, `prepare_epic_launch` still returns a
   submitted `BackgroundTask` and raises no `PlanApprovalActionError`; the existing
   `tests/test_plan_approval_actions.py:415` and `tests/ace/tui/test_notification_plan_gate.py:226` monkeypatches of
   `require_epic_launch_store_health` must keep working.

Drive every wait bound in tests through the env vars so the suite pins behavior instead of wall-clock luck, and keep all
timeouts sub-second.

## Verification

```bash
just install    # required: workspaces are ephemeral and deps may have changed
just check
```

`just check` must pass, including Symvision — `resolve_workspace_anchor`, `epic_launch_lock_anchor`, and the new
`ENV_EPIC_APPROVAL_PREFLIGHT_LOCK_TIMEOUT` constant must all have real callers, not just test callers.

## Out of scope

Non-epic-approval writers of the shared sidecar clones (an ACE bead refresh, `sase bead sync`, an ad-hoc `sase bead`
command run from the primary workspace) can still integrate the beads clone while a launch is in flight. The freshness
TTL and failed-integration cooldown in `_pull_sdd_clone` make that window narrow, and closing it means teaching
`ensure_beads_sidecar_clone` to respect the launch lock — a broader change than the reported failure needs. File it as a
follow-up task bead if it is observed in practice; do not widen this change to cover it.
