---
tier: tale
title: Guard sase dev update against swapping code under a running sase bead work
goal:
  A `sase dev update` fast-forward can no longer hot-swap the editable source tree underneath an in-flight `sase bead
  work`, `sase bead work` refuses to start while a swap is in progress instead of dying mid-launch, and the
  `priority_property` epic (`gh_bobs-org__bob-cli-4`) is relaunched successfully.
proposed_by: bbugyi200.athena.se
create_time: 2026-08-02 16:00:21
status: done
---

- **PROMPT:**
  [prompts/202608/dev_update_code_swap_guard.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/dev_update_code_swap_guard.md)

# Plan: Guard `sase dev update` against swapping code under a running `sase bead work`

## Background — what actually broke

On 2026-08-02 the epic plan `~/.sase/plans/202608/priority_property.md` was approved from ACE and launched as a detached
`sase bead work` task. The launch failed with:

```
Error: agent launch failed for epic gh_bobs-org__bob-cli-4: cannot import name 'StoredPromptRenderings' from
'sase.history.chat_prompt_sections' (/home/bryan/projects/github/sase-org/sase/src/sase/history/chat_prompt_sections.py)
```

The name exists in that file today, and existed in the commit the checkout was on when the error was reported. The
failure is **not** a missing symbol — it is a torn Python module graph.

### Evidence

| Time (EDT, 2026-08-02) | Event                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Source                                                                                                 |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| 14:43:36               | Commit `e3ca2c11c` "feat: expose stored prompt renderings" adds `StoredPromptRenderings` to `src/sase/history/chat_prompt_sections.py` and adds an import of it to `src/sase/ace/tui/widgets/artifacts/chats_detail.py`                                                                                                                                                                                                                                                                  | `git show e3ca2c11c`                                                                                   |
| 15:21:47.60            | Detached task `m9fmjfaq0k4r` starts `sase bead work ~/.sase/plans/202608/priority_property.md --yes-to-all …` (pid 3861058). The host checkout is still on `c8211ae5c`, which predates `e3ca2c11c`.                                                                                                                                                                                                                                                                                      | `~/.sase/tasks/tasks.jsonl`                                                                            |
| 15:21:48.19            | `bead_work` launch timer starts                                                                                                                                                                                                                                                                                                                                                                                                                                                          | `~/.sase/logs/tui_launch_timing.jsonl`                                                                 |
| ~15:21:49–54           | Stages `archive_plan_file` … `plan_snapshot` run. The prompt-render / prompt-archive machinery imports `sase.history.chat_prompt_sections` — the **pre-`e3ca2c11c`** file, with no `StoredPromptRenderings` — into `sys.modules`.                                                                                                                                                                                                                                                        | launch-timing stage list                                                                               |
| 15:21:54.87            | `sase dev update` runs `git merge --ff-only` on the host checkout, fast-forwarding `c8211ae5c → 72dd097de` (4 commits, including `e3ca2c11c`) and rewriting `src/sase/**` in place, then reinstalls the uv-tool editable packages.                                                                                                                                                                                                                                                       | `git reflog`, file mtimes, `~/.sase/logs/dev_update.jsonl` (journal entry `2026-08-02T15:21:58-04:00`) |
| ~15:22:02–04.6         | The `agent_launch` stage runs. `launch_bead_work_agents` performs its deferred `from sase.agent import launcher`; the launch path then does `from sase.ace.tui.actions.agent_workflow._ref_resolution import …`, which is compiled **fresh from the new on-disk source** and transitively imports the new `chats_detail`, whose module-level `from sase.history.chat_prompt_sections import StoredPromptRenderings` resolves against the **stale cached module object** → `ImportError`. | `src/sase/bead/cli_work_launch.py`, `src/sase/agent/launch_cwd_agents.py:71`                           |
| 15:22:04.60            | Launch aborts, zero agents spawned, preclaims rolled back, rollback published, resume command printed                                                                                                                                                                                                                                                                                                                                                                                    | `~/.sase/tasks/logs/m9fmjfaq0k4r.log`                                                                  |

The import chain was verified directly against the installed interpreter: importing
`sase.ace.tui.actions.agent_workflow._ref_resolution` puts `sase.ace.tui.widgets.artifacts.chats_detail` into
`sys.modules`, and `chats_detail` is the only importer of `StoredPromptRenderings` anywhere in the tree. Meanwhile
`sase.bead.cli_work_handler` at import time loads **neither** module, and `sase.agents_sync.prompt_archive.validation`
loads `chat_prompt_sections` but **not** `chats_detail` — exactly the split that produces the tear.

### Root cause

`sase` is installed as a uv tool with an **editable** install pointing at the live host checkout. `sase dev update`
fast-forwards that checkout in place. Any `sase` process that outlives the fast-forward and then performs a deferred
import executes a mix of pre-swap modules (cached in `sys.modules`) and post-swap modules (compiled from the new files).
`sase bead work` is the worst possible victim: it is long-running (~16 s), it is dense with function-local imports, and
it does destructive, stateful work (archives the plan, creates the epic and phase beads, publishes the graph, preclaims
workspaces) **before** reaching the deferred imports in its final `agent_launch` stage.

Nothing serialises the two. `execute_dev_update` preflights only for a dirty tree and upstream ancestry
(`_preflight_actionable_roots` in `src/sase/dev_update/execute.py`), and `dev_update_blocking_reason`
(`src/sase/ace/tui/modals/plugins_browser_dev_update.py:170`) only checks plan actionability. There is no notion of "a
sase process is currently running against this source tree".

This is a general hazard, not a one-off: it can fire for any sufficiently long `sase` command, and it fires
destructively for epic launches.

## Goal

1. A `sase dev update` fast-forward never overlaps an in-flight `sase bead work`.
2. If a swap is already in progress, `sase bead work` fails immediately — before archiving the plan or creating any
   beads — with an actionable message, instead of dying 15 seconds later with an `ImportError` and a half-built epic.
3. The `priority_property` epic is relaunched.

## Repositories

All source changes are in the primary `sase` repo. The relaunch step in Step 7 touches the `bob-cli` project; open it
with the `/sase_repo` skill (`sase repo open bob-cli -r "<reason>"`) and use the printed path.

## Design

A reader/writer advisory lock over "the installed source tree is being swapped", implemented with `fcntl.flock` on a
single lock file, mirroring the existing epic-plan launch lock in `src/sase/bead/cli_work_from_plan_store.py`
(`epic_plan_launch_lock`, `_acquire_epic_plan_launch_lock`, `_write_lock_holder`).

- Lock file: `sase_subdir("locks") / "code-swap.lock"`.
- **Readers** (`sase bead work`) take `LOCK_SH | LOCK_NB` for the lifetime of the command.
- **Writer** (`sase dev update`) takes `LOCK_EX | LOCK_NB` across the merge _and_ the reconcile steps — the editable
  reinstall is part of the swap and must be covered too.
- Neither side blocks. Waiting is wrong here: a reader that waits has already imported pre-swap modules, and a writer
  that waits stalls the TUI. Both sides fail fast with a message naming the other holder.

Holder identity for human-readable messages lives in a sidecar directory `sase_subdir("locks") / "code-swap.holders/"`,
one `<pid>.json` per reader (`pid`, `op`, `command`, `started_at`), written on acquire and removed on release. Reporting
must filter entries through `is_process_running` (`src/sase/ace/hooks/processes.py:36`) so a crashed reader never
permanently blocks updates — the `flock` itself is the source of truth and is released by the kernel on process death;
the sidecar is advisory metadata only.

**Known residual race, state it in the docstring:** a reader that starts _during_ a swap can still import torn modules
before it reaches the lock. Step 5 narrows that window; closing it fully would require re-exec and is out of scope.

## Implementation

### Step 1 — new module `src/sase/dev_update/code_swap_lock.py`

Public surface:

- `code_swap_reader_lock(*, op: str, command: Sequence[str] | None = None)` — a context manager taking a non-blocking
  `LOCK_SH`. Yields a result carrying `acquired: bool` and, when not acquired, `blocked_by: str` (a rendered description
  of the writer). Registers and removes the holder sidecar file.
- `code_swap_writer_lock()` — a context manager taking a non-blocking `LOCK_EX`. When not acquired, `blocked_by` renders
  the live reader holders.
- `code_swap_readers_active() -> str | None` — cheap peek used by the TUI preview path (Step 4). Must not be treated as
  a guarantee; only the writer lock is authoritative.

Follow the existing conventions in `cli_work_from_plan_store.py`: `os.open` the lock file with `O_RDWR | O_CREAT`, retry
`InterruptedError`, and never let an `OSError` on the sidecar metadata break the operation (log at debug).

Provide an escape hatch env var (for example `SASE_DISABLE_CODE_SWAP_LOCK=1`) that makes both sides no-ops, so a wedged
lock can never brick either command.

### Step 2 — `sase bead work` takes the reader lock

`handle_bead_work` in `src/sase/bead/cli_work_entry.py:17` is the single entry point for every `sase bead work` variant.
Enter `code_swap_reader_lock(op="bead.work", command=sys.argv)` as the **first** thing it does, before any project or
store resolution.

If the lock was not acquired, abort immediately with a non-zero exit and a message along the lines of:

```
sase dev update is swapping the installed source tree (<blocked_by>).
No work was started. Re-run this command once the update finishes.
```

Nothing destructive has happened at that point — that is the entire point of putting it first. Use the error type the
surrounding code already uses for user-facing bead-work failures so the detached-task log and the ACE notification both
render it.

### Step 3 — `sase dev update` takes the writer lock

In `src/sase/dev_update/execute.py`, wrap `_merge_actionable_roots` **and** `_run_reconcile_steps` inside
`code_swap_writer_lock()`.

Acquire it as the last preflight, immediately after `_preflight_actionable_roots` succeeds and immediately before the
first merge, to keep the reader-starts-late window as small as possible.

When the writer lock is unavailable, return early through the existing `_failed_result` path with a reason that reads as
a **deferral**, not a failure — for example:

```
deferred: sase bead work (pid 3861058) is running against this checkout; re-run `sase update` when it finishes
```

Check how `DevUpdateResult` failures are rendered by `sase update` (`src/sase/main/update_handler.py`) and by the TUI
(`dev_update_failure_message`); if a plain failure string reads as alarming for what is a benign deferral, add a
dedicated `deferred` flag to `DevUpdateResult` rather than overloading the failure string. Either way the journal entry
written by `append_dev_update_journal` must record it, so `~/.sase/logs/dev_update.jsonl` shows deferrals.

### Step 4 — surface the deferral before the user commits to an update

`dev_update_blocking_reason` in `src/sase/ace/tui/modals/plugins_browser_dev_update.py:170` is consulted by
`plugins_browser_update.py`, `plugins_browser_sase_update.py`, and `plugins_browser_comprehensive_update.py` before
executing. Add the `code_swap_readers_active()` check there so the confirm/preview modal reports "a sase bead work is
running" up front instead of the user approving an update that then defers.

### Step 5 — narrow the deferred-import window in the launch path

Independent of the lock, stop `agent_launch` from being the first stage to touch fresh-from-disk modules. In the
`sase bead work` launch paths (`src/sase/bead/cli_work_from_plan.py`, `src/sase/bead/cli_work_handler.py`), after the
reader lock is held and **before** the first destructive stage (`archive_plan_file` / `epic_creation`), eagerly import
the launch chain:

- `sase.agent.launcher`
- `sase.ace.tui.actions.agent_workflow._ref_resolution`

Gate this on actually intending to launch (skip it for `--dry-run` and preview paths) so no new startup cost lands on
non-launching invocations. Record it as its own `preload_launch_imports` stage through the existing
`LaunchTimingRecorder` so the cost is visible in `~/.sase/logs/tui_launch_timing.jsonl`.

Note for the implementer: these imports are not free (the `_ref_resolution` chain pulls in the ACE widget layer). If the
measured cost is material on the launch path, keep the preload but say so in the commit message; do not silently drop
the step.

### Step 6 — tests

Match the layout already present in `tests/dev_update/` and `tests/test_bead/`.

- New `tests/dev_update/test_code_swap_lock.py`: reader acquires; writer is refused while a reader holds; reader is
  refused while a writer holds; two readers coexist; a dead holder's sidecar entry is ignored; the disable env var makes
  both sides no-ops.
- Extend `tests/dev_update/test_execute.py`: `execute_dev_update` performs **no** merge command when the writer lock is
  unavailable, and the returned reason names the blocking reader.
- New coverage under `tests/test_bead/` (alongside `test_cli_work_contention_regressions.py`): `handle_bead_work` exits
  non-zero and mutates **no** store state when a writer holds the lock — assert that the plan file is not archived and
  no bead is created, since "fails before doing damage" is the property that matters.
- A test that the Step 5 preload actually imports the launch chain (assert the modules land in `sys.modules`).

Run `just install` first (workspace virtualenvs go stale), then `just check`.

### Step 7 — relaunch the `priority_property` epic

Do this only after Steps 1–6 are implemented, `just check` passes, and the change is committed.

The failed launch left durable state behind: epic `gh_bobs-org__bob-cli-4` and its five phase beads (`…-4.1` … `…-4.5`)
were created, the dependency graph was committed and published, the plan was archived into the bob-cli plans sidecar at
`sase/repos/plans/202608/priority_property.md`, and the plan-to-bead link was committed. The rollback only restored the
six work preclaims and the epic's `is_ready_to_work` state. So this is a **resume**, not a fresh launch.

1. Open the project: `sase repo open bob-cli -r "Relaunch the priority_property epic after the dev-update race fix"`.
2. Verify the state before doing anything: `sase bead show gh_bobs-org__bob-cli-4` and confirm the five phase beads
   exist, no agents are running against them, and the archived plan carries the `BEAD` link.
3. Run the resume command sase itself printed in `~/.sase/tasks/logs/m9fmjfaq0k4r.log`, with the plans-sidecar path
   resolved inside the checkout from step 1:

   ```bash
   sase bead work <bob-cli-checkout>/sase/repos/plans/202608/priority_property.md --yes
   ```

   `sase bead work` detects the already-linked epic and takes the resume path (`_resume_linked_epic` in
   `src/sase/bead/cli_work_from_plan.py`) rather than recreating beads.

4. If step 2 shows the epic is **not** in a resumable state, stop and report what was found instead of forcing a fresh
   launch — a second epic would be worse than a delayed one.
5. Confirm the launch: five phase agents across four waves plus `gh_bobs-org__bob-cli-4.land`, per the wave plan
   recorded in the task log.

Also note: `~/.sase/plans/202608/priority_property.md` still exists alongside the archived copy. Check whether the live
copy is expected to survive archiving; if it is residue from the aborted run, file a task bead with `/sase_new_task`
rather than deleting it as part of this change.

## Out of scope

- Re-exec-on-swap for readers that start mid-update (the residual race noted in Design).
- Extending the reader lock beyond `sase bead work` to every long-running `sase` command. Do that once this guard has
  proven itself; `sase bead work` is where the failure is destructive.
- Anything about `StoredPromptRenderings` itself — that code is correct and needs no change.

## Verification

- `just install && just check` passes.
- Manual: hold the writer lock from a scratch process, run `sase bead work <plan>`, confirm it exits immediately with
  the deferral message and that the plan file is untouched and no bead was created.
- Manual: hold a reader lock, run `sase update`, confirm no `git merge --ff-only` is executed and the journal records a
  deferral.
- Epic `gh_bobs-org__bob-cli-4` is running.
