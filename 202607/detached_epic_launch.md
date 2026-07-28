---
tier: epic
title: Detached background tasks and a single epic-launch path
goal: 'Approving an epic plan always launches `sase bead work` as one durable, session-independent
  "detached" background task — from the TUI, from Telegram, from the CLI, or with
  nothing interactive running at all — and that task is legible, killable, and beautiful
  on every surface that shows background tasks.

  '
phases:
- id: import_cycle
  title: Break the agents_sync to ace.tui import cycle
  depends_on: []
  size: small
  description: '''Phase import_cycle: Break the agents_sync to ace.tui import cycle''
    section: move the agent value types the sync backend needs out of the TUI package,
    restore `import sase.agents_sync` in non-TUI processes, and add a layering guard
    so a backend module can never import `sase.ace` again.'
- id: core_kind
  title: Accept the detached task kind in the Rust core
  depends_on: []
  size: small
  description: '''Phase core_kind: Accept the detached task kind in the Rust core''
    section: widen the background-task store''s kind validation to admit `detached`,
    cover it with Rust tests, and rebuild the Python binding.'
- id: cwd
  title: Resolve the epic launch workspace without provider env vars
  depends_on: []
  size: small
  description: '''Phase cwd: Resolve the epic launch workspace without provider env
    vars'' section: populate a plan gate''s `project_dir` from the shared runtime-neutral
    env contract and let the host claim an epic launch from `agent_project_file` alone.'
- id: runner
  title: Detached task submission, ownership, and filtering
  depends_on:
  - core_kind
  size: small
  description: '''Phase runner: Detached task submission, ownership, and filtering''
    section: add `submit_detached_task` plus the detached kind constant, teach orphan
    reconciliation to own detached rows, and add a kind filter to the shared task
    query.'
- id: launch
  title: Launch approved epics as one detached sase bead work task
  depends_on:
  - runner
  - cwd
  size: medium
  description: '''Phase launch: Launch approved epics as one detached sase bead work
    task'' section: make the epic launch a detached task whose command is literally
    `sase bead work <plan> --yes-to-all`, replace the output-regex worker with structured
    in-process metadata backfill and notification, and dedup concurrent launches of
    the same plan.'
- id: callers
  title: Collapse every epic approval onto the detached launch
  depends_on:
  - launch
  size: medium
  description: '''Phase callers: Collapse every epic approval onto the detached launch''
    section: delete the TUI tracked-task launch, the foreground CLI launch, and the
    agent-side subprocess launch so every approval path submits the same detached
    task and a launch that cannot be claimed fails loudly.'
- id: surfaces
  title: First-class detached tasks on the CLI and in the TUI
  depends_on:
  - runner
  size: medium
  description: '''Phase surfaces: First-class detached tasks on the CLI and in the
    TUI'' section: render and filter detached tasks in `sase task`, add `sase task
    kill` and `sase task run --detached`, fix the TUI detached indicator and Tasks-tab
    scope, and document the new kind.'
- id: verify
  title: End-to-end verification with and without a TUI
  depends_on:
  - callers
  - surfaces
  size: small
  description: '''Phase verify: End-to-end verification with and without a TUI'' section:
    approve a real epic plan with no TUI running and again from the TUI, and confirm
    one detached task per approval across every surface.'
create_time: 2026-07-26 07:20:14
status: done
bead_id: sase-9s
---

- **PROMPT:** [202607/prompts/detached_epic_launch.md](prompts/detached_epic_launch.md)

# Plan: Detached background tasks and a single epic-launch path

## Why: what actually broke

On 2026-07-26 the epic approved for the `l4.cld` agent (`sdd_clone_integration_race.md`) failed. Two independent defects
combined.

### Defect 1 — `sase bead work` cannot even start (the proximate crash)

The launch died with:

```
File "src/sase/workflows/commit/runtime_tags.py", line 45, in _resolve_runtime_commit_tags
  from sase.agents_sync.links import resolve_agent_commit_tag
File "src/sase/agents_sync/__init__.py", line 3 -> git_sync -> incoming_integration
  -> v2_importer -> v2_import_transactions.py", line 46, in <module>
  from sase.ace.tui.models.agent_types import AgentType
File "src/sase/ace/tui/__init__.py", line 3 -> .app -> .actions -> .actions.agents_sync
  from sase.agents_sync import (...)
ImportError: cannot import name 'get_agents_sync_status' from partially initialized
module 'sase.agents_sync' (most likely due to a circular import)
```

`src/sase/agents_sync/v2_import_transactions.py:46` imports `AgentType` from `sase.ace.tui.models.agent_types`.
Importing that leaf module initializes the `sase.ace.tui` package, whose `__init__.py` eagerly imports `AceApp`, which
imports `sase.ace.tui.actions.agents_sync`, which imports back into the half-initialized `sase.agents_sync`.

This is reproducible at HEAD and it is _import-order dependent_:

```bash
python -c "import sase.agents_sync"                       # ImportError
python -c "import sase.ace.tui; import sase.agents_sync"  # ok
```

The TUI imports `sase.ace.tui` first, so it never sees the cycle. Any plain `sase` CLI process that reaches
`agents_sync` first dies — which is every commit path, because `apply_auto_commit_tags_with_runtime` resolves runtime
commit tags through `sase.agents_sync.links`.

`sase bead work` crashed _after_ `✓ Validated` and `✓ Store` but _inside_ `commit_plan_file`, so the approved plan was
archived into the plans sidecar and never committed. That is the same dirty sidecar the axe
`workspace_sdd_clone_recovery` lumberjack then failed to recover twice.

Introduced by commit `6363f22db` ("fix: record dismissed identities during v2 import"). It is also a layering violation:
`sase.agents_sync` is backend, and backend must not import the TUI.

### Defect 2 — the epic launch never reaches the task runner

`src/sase/plan_gate.py:469` builds a plan gate's action data with:

```python
"project_dir": os.environ.get("CLAUDE_PROJECT_DIR"),
```

`CLAUDE_PROJECT_DIR` is a Claude-CLI-specific variable and it is **not set** in a SASE agent's environment (verified
empirically inside a live Claude agent run). Falsy values are dropped from the action-data mapping, so plan gates ship
with no `project_dir` at all — confirmed in the real `EpicApproval` notification for `l4.cld`, whose `action_data`
carries `agent_project_file`, `artifacts_dir`, and `bundle_path` but no `project_dir`.

Every host claim then declines, silently:

- `src/sase/ace/tui/actions/agents/_notification_epic_launch.py:42` returns `False` on its first guard, so no tracked
  task is ever created.
- `src/sase/_plan_approval_epic.py:119` (`epic_launch_cwd`) returns `None`, so `can_claim_epic_launch()` is `False` and
  even `epic_launch_mode="detached"` cannot claim.

With no host owner, `epic_launch_owner` is never set, and `src/sase/axe/run_agent_exec_plan_accept.py:509` falls back to
`_run_epic_launch_subprocess()` — the invisible in-agent launch that `sase-95.7` was written to remove. Nothing appeared
in `sase task list` or the Tasks tab; the failure surfaced only as an axe notification. The task store confirms it: two
`Plan response: epic` rows on 2026-07-26 both reported "Epic approved", and there is no epic-launch row for either.

The fallback also runs in the _agent's_ ephemeral workspace instead of the primary checkout, which is why the sidecar it
corrupted was a numbered workspace's clone.

Note that `resolve_epic_launch_cwd()` already prefers `agent_project_file` and only consults `project_dir` as a fallback
— so the data needed to resolve the workspace was present the whole time. The guards in front of it were simply too
strict.

### What the user asked for

1. Diagnose and fix the failure.
2. Support epic approvals with no TUI running.
3. Do it with a new "detached" kind of SASE background task.
4. Every epic approval, from any surface, launches `sase bead work` as a detached task.
5. Excellent support for detached tasks on every surface that shows background tasks.
6. Intuitive, reliable, beautiful.

## Design

### The detached task kind

`sase.tasks` records two kinds today:

| kind      | owner                           | lifetime               | scope                   |
| --------- | ------------------------------- | ---------------------- | ----------------------- |
| `tui`     | the TUI process that mirrors it | dies with that TUI     | that session            |
| `command` | `sase.tasks.supervisor`         | survives its submitter | attributed to a session |

Add a third:

| kind       | owner                   | lifetime                 | scope                           |
| ---------- | ----------------------- | ------------------------ | ------------------------------- |
| `detached` | `sase.tasks.supervisor` | survives every submitter | **global — no session owns it** |

`detached` is not "another way to spawn a subprocess"; `command` already detaches. The distinction that earns a new kind
is **ownership**: a detached task belongs to no interactive session, so it is _always_ in scope on every surface. That
is exactly the property "an epic approved from Telegram at 3am must be visible in whatever TUI I open at 9am" requires.

Concretely a detached row always has `session_id = None` and `session_label = None`. That is not a loss of information —
it is honest, and the store's existing scope rules already treat unattributed rows as globally visible:

- `sase task list` includes them by default (`include_unattributed`).
- The TUI Tasks tab includes them by default (`_in_scope` admits `task.session_id is None`).

_Where the work came from_ is recorded in `origin` (`ace`, `telegram`, `cli`, `axe`, `api`), which is what `origin` is
for. This is also a correctness fix: `submit_epic_launch_task()` currently pins the launch to whatever ACE session
`_epic_launch_session_id()` happens to resolve, which hides it from every other surface.

### The epic launch becomes exactly what the user described

Today the launch is a two-level process sandwich — the supervisor runs `python -m sase.bead.epic_launch --worker`, which
runs `sase bead work` — and the wrapper recovers its results by **regex-scraping its own task log** (`_EPIC_ID_RE`,
`_PLAN_LINK_RE`, `_ARCHIVED_PLAN_RE`) with a 2-second `_LOG_SETTLE_SECONDS` poll to race the supervisor's log drain.

Replace all of it. The detached task's command becomes, literally:

```
sase bead work <plan> --yes-to-all
```

That is one process under the supervisor, and it is the same string the resume hint tells the user to run.
`sase bead work` already _returns_ what the wrapper was scraping — `work_from_plan_file()` yields a result carrying
`epic_id` and `archived_plan_path` — so the planner-metadata backfill and the completion notification move into that
process, where the values are structured facts rather than parsed text. The regexes, the settle loop, the log re-reader,
and the `--worker` entry point all go away.

The approval-linking inputs (which planner artifacts dir to back-fill, which ChangeSpec to name in the notification) are
passed as explicit `sase bead work` options so the behavior is discoverable in `--help`, not carried by a hidden env
var.

### One path, no silent fallbacks

Three code paths can launch an epic today:

1. TUI tracked task — `_notification_epic_launch.submit_epic_launch_task()`
2. Foreground CLI — `sase plan approve` with `epic_launch_mode="foreground"`
3. In-agent subprocess — `_run_epic_launch_subprocess()`

All three go away. The single path is `prepare_epic_launch()` → `submit_epic_launch_task()` → a detached task, reached
from the shared neutral gate executor (`PlanGateAdapter.on_response`) that _every_ surface already funnels through: the
TUI, `sase gate`, `sase plan approve`, auto-approve, and the Telegram inbound lumberjack.

Because that executor runs in whichever process applies the response, and `submit_task()` spawns its supervisor with
`start_new_session=True`, the launch outlives its submitter. That is what makes "no TUI running" work: the Telegram
lumberjack (or a bare `sase gate` invocation) submits and exits, and the supervised `sase bead work` keeps going.

`EpicLaunchMode` collapses from `detached | foreground | skip` to `detached | skip`. And a host that _cannot_ claim the
launch must now raise `PlanApprovalActionError("epic_launch_failed", ...)` with a resume hint instead of returning
`False` and letting an invisible fallback take over. Silence is what turned this bug into a 20-minute forensic dig
through `~/.sase/axe/recent_errors.json`.

### Back-compat during the transition

An agent process launched by an older build still reads `epic_launch_owner` from the response and will launch the epic
itself if the field is absent. So epic approvals must keep writing `epic_launch_owner: "host"` unconditionally, even
though current agent code no longer consults it. Dropping the field would double-launch every in-flight epic across an
upgrade.

## Phase import_cycle: Break the agents_sync to ace.tui import cycle

Unblocks every other phase, and is independently shippable — without it `sase bead work` cannot run at all.

`AgentType` is a two-member `Enum` of domain values (`RUNNING = "run"`, `WORKFLOW = "workflow"`) that happens to live in
the TUI's model package. The sync backend legitimately needs it (`DismissedIdentity` is keyed by it), so the type is in
the wrong place, not the import.

1. Move the agent value types the backend needs out of `sase.ace.tui.models` into a UI-free module —
   `src/sase/core/agent_types.py` is the natural home, next to the existing `sase.core.agent_identity_facade`. Move
   `AgentType` at minimum; move `AgentChildLinkage` too if it is referenced from non-TUI code, and leave purely
   presentational types (for example `LinkedRepoMetadata`) where they are unless they are already imported by backend
   modules.
2. Re-export from `src/sase/ace/tui/models/agent_types.py` and `src/sase/ace/tui/models/__init__.py` so the ~dozens of
   TUI call sites and the public `sase.ace.tui.models` surface keep working unchanged. Check `sase/memory/symvision.md`
   via `/sase_memory_read` before adding re-exports — a re-export that Symvision reads as an unused symbol needs the
   documented treatment.
3. Point `src/sase/agents_sync/v2_import_transactions.py:46` at the new module.
4. Sweep for the same violation elsewhere: any `src/sase/agents_sync/**` module importing `sase.ace`, and any other
   backend package doing the same. Fix what you find the same way.

Regression coverage — both matter, because a unit test that runs after the TUI has already been imported will pass
against the broken code:

- A test that imports `sase.agents_sync` in a **fresh interpreter**
  (`subprocess.run([sys.executable, "-c", "import sase.agents_sync"])`) and asserts a clean exit. Do the same for
  `sase.agents_sync.links`, the module the real crash reached through `runtime_tags`.
- A layering guard that walks the `src/sase/agents_sync/**` AST and fails on any `sase.ace` import, module-level or
  deferred. If the repo already has an import-layering test, extend it rather than adding a parallel one.

Add a `sase bead work` smoke check that the command reaches plan validation without an `ImportError`, so a future
backend-to-TUI import is caught by the suite instead of by a failed epic approval.

## Phase core_kind: Accept the detached task kind in the Rust core

The task store is a Python facade over `sase_core_rs`, and the Rust side is the gatekeeper for `kind`. Open the
`sase-core` linked repo with `/sase_repo` first; do not edit it through any other path.

In `crates/sase_core/src/tasks/store.rs`, `normalize_and_validate_task()` currently rejects anything outside
`command | tui`:

```rust
if !matches!(task.kind.as_str(), "command" | "tui") {
```

1. Admit `"detached"`. Prefer a named constant or a small helper next to `validate_status()` so the accepted kinds are
   declared in one place, matching how statuses are already validated.
2. Add Rust tests: a `detached` row appends and round-trips; an unknown kind is still rejected with the existing
   `unknown kind` message; a `TaskUpdateWire` that sets `kind: "detached"` on an existing row validates.
3. Leave `TASK_WIRE_SCHEMA_VERSION` at 1. Widening an accepted enum value is additive: older readers deserialize a
   `detached` row into `BackgroundTaskWire` fine, because `kind` is a plain `String` and only _writes_ are validated.
4. Rebuild the binding so the Python side can exercise the new kind, and run the core's own checks before handing off.

Commit the core change in the `sase-core` repo. The Python phases below assume a rebuilt binding is available.

## Phase cwd: Resolve the epic launch workspace without provider env vars

Two small, independent fixes that together make the host able to claim a launch.

**Runtime-neutral `project_dir`.** `src/sase/plan_gate.py:469` reads `CLAUDE_PROJECT_DIR` directly.
`src/sase/env_contracts.py` already defines the right contract for this — `PROVIDER_PROJECT_DIR_ENV_VARS`, which covers
`CODEX_PROJECT_DIR`, `SASE_ACTIVE_PROJECT_DIR`, `CLAUDE_PROJECT_DIR`, `QWEN_PROJECT_DIR`, `GEMINI_PROJECT_DIR`, and
`OPENCODE_PROJECT_DIR`. Resolve `project_dir` from the first of those that is set. `SASE_ACTIVE_PROJECT_DIR` _is_
exported into every SASE agent environment regardless of runtime, so this alone makes `project_dir` present for the
first time. Reuse or extend an existing helper if one resolves that tuple already
(`src/sase/llm_provider/commit_finalizer_config.py` consumes it) rather than open-coding a third lookup. This is also
required by the "Uniform Agent Runtimes" convention in `CLAUDE.md`: reading a Claude-only variable makes Codex, Gemini,
and Qwen agents second-class.

**Claim on either signal.** `resolve_epic_launch_cwd()` in `src/sase/bead/epic_launch.py` ignores `project_dir` entirely
when `agent_project_file` is set — but the guards in front of it bail on a missing `project_dir` first. Fix both:

- Make `project_dir` optional in `resolve_epic_launch_cwd()`, and raise a clear error only when _neither_ signal is
  available.
- `epic_launch_cwd()` in `src/sase/_plan_approval_epic.py` should proceed when `agent_project_file` _or_ `project_dir`
  is present.
- Do the same in `src/sase/_plan_approval_artifacts.py` if its `project_dir` use has the same over-strict shape.
  (`_plan_action_project_name()` already prefers `agent_project_file` — use it as the model.)

Tests: a plan gate built with only `SASE_ACTIVE_PROJECT_DIR` set carries a `project_dir`; `epic_launch_cwd()` resolves
the primary workspace from `agent_project_file` alone with no `project_dir` in the action data; and with neither signal
it returns `None` (so the loud failure in phase `callers` has something to report).

The resolved cwd stays workspace 1, the project's primary checkout. That is already what `resolve_epic_launch_cwd()`
returns and it is the right answer — running the launch in an agent's ephemeral numbered workspace is what corrupted a
sidecar clone during the incident.

## Phase runner: Detached task submission, ownership, and filtering

All in `src/sase/tasks/`.

1. **Kind constants** in `models.py`: name the three kinds (`COMMAND_TASK_KIND`, `TUI_TASK_KIND`, `DETACHED_TASK_KIND`)
   and a `TASK_KINDS` frozenset, mirroring how `ACTIVE_TASK_STATUSES` and `TERMINAL_TASK_STATUSES` are already declared.
   Export them from `sase.tasks`. Have `src/sase/ace/tui/task_mirror.py` use `TUI_TASK_KIND` for its `MIRROR_KIND`
   rather than a second literal.

2. **`submit_detached_task()`** in `runner.py`. Factor the shared body out of `submit_task()` so both go through one
   implementation and there is no second copy of the record-then-spawn-supervisor sequence. Differences: `kind` is
   `detached`, and `session_id` is unconditionally `None` (do not accept a session parameter — the whole point is that
   no session owns it). Signature:
   `submit_detached_task(argv, *, label, cwd, origin, project=None, workspace_num=None, tags=(), cl_name=None, env=None)`.

3. **Reconciliation must own detached rows.** `_is_orphaned()` reads:

   ```python
   if task.kind != "command":
       return False
   ```

   A `detached` row whose submitter died between `append_task()` and the supervisor `Popen` has no pid, so today it
   would sit `pending` forever and never reconcile. Change the guard to "supervisor-owned kinds" (`command` or
   `detached`), keeping the `_UNCLAIMED_GRACE_SECONDS` window and keeping `tui` rows excluded — those are owned by their
   own live process. Test this directly: a pid-less `detached` row older than the grace window reconciles to `error`; a
   freshly created one does not.

4. **Kind filter** in `store.py`: add `kind: str | Collection[str] | None` to `read_tasks()` and `filter_tasks()`,
   following the existing `status` parameter exactly (including its `_status_set` normalization shape). The CLI and TUI
   both need it in phase `surfaces`.

5. `submit_detached_task()` must reject an empty argv and a non-existent cwd with `TaskSubmitError`, same as
   `submit_task()` — the shared helper gives that for free, so just assert it.

Extend `tests/test_tasks_runner.py` and `tests/test_tasks_facade.py`: a detached submit records `kind="detached"` and
`session_id=None` even when a live session exists; the supervisor drives it to a terminal status; `kill_task` works on
it; `read_tasks(kind=...)` filters correctly.

## Phase launch: Launch approved epics as one detached sase bead work task

The core of the feature. Read `sase/memory/cli_rules.md` via `/sase_memory_read` before adding the `sase bead work`
options below.

**Rewrite `src/sase/bead/epic_launch.py`.** `submit_epic_launch_task()` becomes:

```python
submit_detached_task(
    build_epic_launch_argv(plan_file, artifacts_dir=..., cl_name=...),
    label=f"Epic launch · {Path(plan_file).stem}",
    cwd=cwd,
    origin=origin,          # "ace" | "telegram" | "cli" | "axe"
    project=...,            # inferred from cwd, so the row is attributable
    tags=("epic", "launch"),
    cl_name=cl_name,
)
```

Delete `_run_detached_worker()`, `_run_launch_with_inherited_output()`, `_run_launch_into_log()`,
`_read_launch_output()`, `_read_retained_log()`, `_notify_detached_completion()`, `parse_epic_launch_output()`,
`_EpicLaunchOutput`, the three module-level regexes, both `_LOG_SETTLE_*` constants, `TASK_LOG_PATH_ENV`, the `--worker`
argparse entry point, and `_epic_launch_session_id()`. Keep `build_epic_launch_argv()` (now taking the approval-linking
options) and `update_epic_launch_metadata()` (now called in-process by `sase bead work`).

**Give `sase bead work` the approval-linking options.** Add two options to the `work` subparser in
`src/sase/main/parser_bead.py`, following the repo's short+long flag convention:

- one naming the planner artifacts directory to back-fill,
- one naming the ChangeSpec to attribute the completion notification to.

When they are supplied and a launch succeeds, `sase bead work` calls `update_epic_launch_metadata()` with the `epic_id`
and `archived_plan_path` that `work_from_plan_file()` already returns, then sends the `notify_workflow_complete`
epic-launch notification. On failure it sends the failure notification with the resume hint. Both stay best-effort: a
notification or metadata write that fails must not change the command's exit code, exactly as the old wrapper behaved.

This is the reliability payoff — the epic id now comes from the value the command computed, not from a regex over a log
file being drained by another process.

**Dedup concurrent launches.** Before submitting, check the store for an active (`pending` or `running`) detached row
already launching the same plan; if one exists, return it instead of submitting a second. The gate response file is
written with `open("x")` so only one responder normally wins, and `sase bead work` itself detects an already-linked epic
and resumes — but a re-applied response should not spawn a duplicate supervisor. Key the check on the resolved absolute
plan path (via a tag or a `command` match) plus the `epic`/`launch` tags.

**Health check placement.** `prepare_epic_launch()` in `src/sase/_plan_approval_epic.py` already calls
`require_epic_launch_store_health(cwd)` before submitting and raises
`PlanApprovalActionError("epic_launch_failed", ...)` with a resume hint on a `SddMaterializationError` or
`SddRepositoryHealthError`. Keep that: failing before submit gives the approver a synchronous error instead of a task
that dies seconds later. Recovering a sidecar that a _previous_ crash left dirty stays the axe
`workspace_sdd_clone_recovery` lumberjack's job and is out of scope here.

Rewrite `tests/ace/tui/test_notification_epic_launch.py` and the epic-launch cases in
`tests/test_plan_approval_actions.py` against the new shape. Assert the submitted command is the literal
`sase bead work <plan> --yes-to-all …` argv, that `kind == "detached"` and `session_id is None`, and that a second
submit for the same plan returns the first task rather than creating another.

## Phase callers: Collapse every epic approval onto the detached launch

Remove the alternatives so there is exactly one way an epic gets launched.

1. **Delete the TUI tracked-task launch.** Remove `src/sase/ace/tui/actions/agents/_notification_epic_launch.py`
   entirely, along with its `_EpicLaunchTaskPayload`, its `_epic_launch_dedup_key()`, and its
   `_finish_successful_epic_launch()` toast. Drop the `submit_epic_launch_task` calls and the `host_owns_epic_launch`
   branching in `_notification_plan_gate.py:90` and `_notification_modals.py:322`, so both simply pass
   `epic_launch_mode="detached"`. The user-visible replacement is better, not worse: instead of a TUI-owned row that
   dies with the TUI, the Tasks tab shows the detached row that any surface can see.

2. **Delete the foreground CLI launch.** `src/sase/main/plan_approve_handler.py:161` passes
   `epic_launch_mode="foreground"`; switch it to `"detached"`. Remove `run_epic_launch_foreground()` from
   `epic_launch.py`, drop `"foreground"` from `EpicLaunchMode` in `src/sase/_plan_approval_protocol.py:28`, from the
   mode validation in `_plan_approval_epic.py`, from `src/sase/plan_gate.py:273` and its `epic_launch_mode` enum at
   `plan_gate.py:574`, and from the `on_response` mode check in `src/sase/notification_gates/adapters.py:123`.
   `sase plan approve` should print the new task id and the `sase task show <id> --follow` hint, so a CLI approval is at
   least as informative as it was when it blocked.

3. **Delete the in-agent launch.** In `src/sase/axe/run_agent_exec_plan_accept.py`, the epic branch always returns
   `"epic_approved"`. Remove `_run_epic_launch_subprocess()`, `_EpicLaunchResult`, `_EPIC_ID_LINE`,
   `_EPIC_OUTPUT_TAIL_LINES`, and the `epic_launch_owner` check at line 506. Keep `_notify_epic_launch_failure()` and
   the `epic_launch_error` metadata write — the SDD-store-unusable pre-checks at lines 418 and 474 still legitimately
   fail an approval before any launch is possible, and they are the caller's only way to report it.

4. **Fail loudly when the host cannot claim.** `can_claim_epic_launch()` must stop being an advisory boolean that
   enables a fallback. With the fallback gone, an unresolvable cwd has to raise
   `PlanApprovalActionError("epic_launch_failed", plan, ...)` carrying the `sase bead work <plan> --yes-to-all` resume
   hint. Propagate it: the TUI shows an error toast, `sase plan approve` exits non-zero with the hint, and the gate
   executor raises `GateError("epic_launch_failed", ...)` so Telegram reports it back into the chat.

5. **Keep writing `epic_launch_owner: "host"`** for every epic approval, in both `plan_approval_actions.py:161` and
   `plan_gate.py:280`, unconditionally rather than conditioned on a claim. An in-flight agent from a pre-upgrade build
   treats a missing field as "launch it yourself" and would double-launch the epic. Comment it as transitional so it can
   be removed once no pre-upgrade runner can still be waiting.

Update `tests/test_plan_approval_actions.py`, `tests/test_plan_approve_cli.py`, `tests/test_plan_rejection_response.py`,
`tests/test_epic_approval.py`, and the notification-modal tests. Add a case asserting an unresolvable cwd raises with a
resume hint instead of quietly proceeding — that is the regression that turned this incident into an invisible failure.

## Phase surfaces: First-class detached tasks on the CLI and in the TUI

Independent of `launch` and `callers`; only needs `runner`.

**`sase task` CLI.**

- `src/sase/main/task_render.py`: distinguish detached rows. Add a KIND column to `task_table()` (or a compact glyph
  beside the status glyph — pick one and keep it consistent between the table and `task_detail()`), so `detached`,
  `command`, and `tui` are legible at a glance. `task_detail()` already prints a `Kind` row; make the detached case say
  so unmistakably. A detached row's session cell already renders as the dim `—` em dash from `session_chip()`, which
  reads correctly for "no session owns this" — pair it with the kind marker so `—` never looks like missing data. Keep
  the existing palette and `STATUS_DISPLAY` glyphs; do not introduce a competing color scheme.
- `src/sase/main/parser_task.py` + `task_handler.py`: add a kind filter to `sase task list` (a `--detached` shorthand
  plus a repeatable `--kind {command,tui,detached}`), threading it into `read_tasks(kind=...)`. Detached rows must
  remain in the default scope for every session — verify `_ListScope.matches()` keeps admitting them, and make
  `_ListScope.label()`/`_hidden_hint()` honest about it (a detached task is never one of the "tasks from other sessions"
  that `--all` reveals).
- Add `sase task kill <ID>`. It exists in the store API (`kill_task`) and in the TUI, but not on the CLI — and a
  detached task is precisely the one you cannot reach from a TUI you did not start. Accept the same id-or-prefix
  resolution as `sase task show`, report a task that was already terminal as a no-op rather than an error, and support
  `--json`.
- Add `sase task run --detached` so the new kind is reachable from the CLI, not only from the epic-launch caller. It is
  mutually exclusive with `--session`.
- `_task_json()` gains an additive `detached` boolean. `TASK_JSON_SCHEMA_VERSION` stays 1 — the field is additive and
  `kind` was always in the payload.
- Update the `empty_task_panel()` hint if it should now mention detached work.

**TUI.**

- `src/sase/ace/tui/task_mirror.py`: `_refresh_detached_count()` returns early when `context.session_id is None` and
  filters rows by `session_id=context.session_id`. Detached rows have no session, so as written it would count **zero**
  of them — the indicator that exists specifically to show "an epic launch approved from Telegram" would show nothing.
  Count active rows this process does not own: every `detached` row globally, plus this session's `command` rows. Keep
  the `DETACHED_POLL_SECONDS` cadence and keep every store read on the writer thread — read `sase/memory/tui_perf.md`
  via `/sase_memory_read` before touching this file.
- `src/sase/ace/tui/modals/tasks_pane.py` and `tasks_pane_render.py`: mark detached rows in `task_row_label()` and
  `output_header()`. `TaskInfo` already carries the store `kind` in `task_type` (set by
  `tasks_store_rows._store_task_row()`), so no new plumbing is needed. Confirm `_in_scope()` keeps detached rows visible
  in both scope modes, and that `action_kill_task()` can kill a detached row from any session — it is `store_backed`, so
  `kill_store_task()` already handles it; add the test.
- Follow `src/sase/ace/CLAUDE.md`: if you add or change a Tasks-tab keybinding, update the `?` help modal, and respect
  the footer convention (only conditional keymaps belong in the footer).

**Docs.** Update the `sase task` sections of `docs/cli.md` and the Tasks-tab section of `docs/ace.md` to introduce the
three kinds and what "detached" guarantees. Mention that an approved epic launches as a detached task, and check
`docs/sdd.md` for epic-approval prose that now describes the wrong mechanism.

Add visual snapshot coverage if the Tasks tab already has PNG goldens under `tests/ace/tui/visual/snapshots/png/`;
otherwise plain render assertions on `task_row_label()` and `task_table()` are enough. Run `just test-visual` if you
touch anything the goldens cover.

## Phase verify: End-to-end verification with and without a TUI

Prove the two things the incident disproved: that an epic approval launches at all, and that it does so with nothing
interactive running.

1. **No TUI.** With no `sase ace` process running, approve a small epic plan through the non-TUI path
   (`sase plan approve`, or a gate response applied the way the Telegram inbound job applies one). Confirm: a `detached`
   row appears in `sase task list` from a plain shell with no `--all`; its command is the literal
   `sase bead work <plan> --yes-to-all …`; it reaches `success`; `sase task show <id>` shows the real launch output; the
   epic bead and its phase beads exist; the planner's `agent_meta.json` gained `epic_bead_id`, `epic_started_at`,
   `plan_committed`, and `sdd_plan_path`; and the completion notification arrived.
2. **From the TUI.** Approve another epic from the ACE Agents tab. Confirm exactly **one** detached row is created (not
   a TUI-mirrored row plus a detached one), that the Tasks tab shows it with its detached marker, that the top-bar task
   indicator counts it, and that the planner agent transitions to its epic-approved state without launching anything
   itself.
3. **Survives the TUI.** Approve an epic, quit the TUI while the launch is running, and confirm the task keeps running
   and still reaches a terminal status — then reopen the TUI and confirm the finished row is still visible in the Tasks
   tab.
4. **Loud failure.** Force an unclaimable launch (an action data set with neither `agent_project_file` nor
   `project_dir`) and confirm the approver gets an `epic_launch_failed` error carrying the resume command, with no
   silent in-agent fallback.
5. **Kill path.** Start a launch and `sase task kill` it; confirm the row goes to `killed` and no orphaned
   `sase bead work` process survives.

Record what you observed. If a phase above left a gap, file it as a bead rather than widening this phase.

## Cross-cutting requirements

- `just install` then `just check` before finishing any phase. Workspace directories are ephemeral, so `just install` is
  not optional.
- The `core_kind` phase edits the `sase-core` repo; open it with `/sase_repo` and run that repo's checks there. Per the
  Rust core boundary in `CLAUDE.md`, task-store validation is core backend logic and belongs there, not in a Python-side
  workaround.
- Do not add a runtime-specific branch anywhere in this work. The `CLAUDE_PROJECT_DIR` read being fixed in phase `cwd`
  is exactly the anti-pattern the "Uniform Agent Runtimes" convention forbids.
- Consult `/sase_memory_read` for `cli_rules.md` (new CLI options in `runner`, `launch`, and `surfaces`), `symvision.md`
  (re-exports in `import_cycle`, deletions in `callers`), and `tui_perf.md` (`task_mirror.py` in `surfaces`).
- Deletions are part of the deliverable. Phases `launch` and `callers` remove three launch paths, an output-parsing
  layer, and a whole module. Leaving dead code behind reintroduces the ambiguity this plan exists to remove.
