---
tier: epic
title: '`sase task`: durable, session-aware background tasks'
goal: 'Every SASE background task is durably recorded, listable and inspectable from
  the CLI, attributed to the TUI session it came from, and launchable with `sase task
  run`; approving an epic runs its `sase bead work` bead creation as one of these
  visible tasks instead of an invisible detached process.

  '
phases:
- id: store
  title: Rust task store and bindings
  depends_on: []
  size: medium
  description: '''Rust task store and bindings'' section: add the locked, retention-bounded
    background-task record store and its `sase_core_rs` bindings in the sase-core
    repo.'
- id: sessions
  title: SASE session registry and display identity
  depends_on: []
  size: small
  description: '''SASE session registry and display identity'' section: register live
    TUI sessions, resolve session references, and render the short colored session
    chip shared by the CLI and TUI.'
- id: facade
  title: Python task facade, ids, logs, and the history limit
  depends_on:
  - store
  size: medium
  description: '''Python task facade, ids, logs, and the history limit'' section:
    wrap the Rust store in a `sase.tasks` facade with task ids, bounded log files,
    and the new `tasks.history_limit` config field.'
- id: runner
  title: Detached task supervisor and submit API
  depends_on:
  - facade
  size: medium
  description: '''Detached task supervisor and submit API'' section: run tasks under
    a detached supervisor that owns status transitions, log capture, kill, wait, and
    orphan reconciliation.'
- id: cli
  title: The `sase task` command surface
  depends_on:
  - runner
  - sessions
  size: medium
  description: '''The `sase task` command surface'' section: add `sase task list`,
    `sase task show`, and `sase task run` with rich human output, JSON output, and
    documentation.'
- id: tui
  title: Admin Center Tasks tab over the durable store
  depends_on:
  - runner
  - sessions
  size: medium
  description: '''Admin Center Tasks tab over the durable store'' section: mirror
    in-TUI tasks into the store, render store-backed and cross-session tasks in the
    Tasks tab, and keep the event loop free of store I/O.'
- id: epic
  title: Route approved-epic bead creation through the task runner
  depends_on:
  - runner
  size: small
  description: '''Route approved-epic bead creation through the task runner'' section:
    replace the invisible detached epic-launch fork with a submitted background task
    so approvals stay transparent.'
- id: verify
  title: End-to-end transparency exercise
  depends_on:
  - cli
  - tui
  - epic
  size: small
  description: '''End-to-end transparency exercise'' section: drive a real epic approval
    and a plain `sase task run` across CLI and TUI surfaces, then fix what the exercise
    surfaces.'
create_time: 2026-07-25 08:04:55
status: wip
bead_id: sase-95
---

- **PROMPT:** [prompts/202607/background_tasks.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/background_tasks.md)
- **BEAD:** [sase-95](https://github.com/sase-org/sase--beads/blob/main/pages/sase-95/README.md)

# Plan: `sase task` — durable, session-aware background tasks

## Why

SASE already runs a lot of work in the background, but none of it is durable or addressable:

- The ACE **Tasks** tab (SASE Admin Center) renders `TaskQueue` (`src/sase/ace/tui/task_queue.py`), which is a
  **process-local, in-memory** registry. Quit the TUI and every task row, log, and outcome is gone. Nothing outside that
  one process — no CLI, no agent, no second TUI — can see it.
- Approving an epic spawns `spawn_detached_epic_launch()` (`src/sase/bead/epic_launch.py`), a fully detached worker that
  writes one log file nobody is told about. From the user's seat, approving an epic from Telegram or from a TUI gate
  panel produces silence until agents appear minutes later. When it fails, the failure is a notification and a path.
- The axe pane's 1–9 `bgcmd` slots (`src/sase/ace/tui/bgcmd.py`) are durable but capped at nine, keyed by slot rather
  than identity, and only reachable from one pane.

The result: no way to answer "what is SASE running for me right now, and what happened to the last thing it ran?"

This epic introduces one durable, session-aware background-task record that every surface shares, a `sase task` command
over it, and rewires epic approval to use it — so approving an epic is visible work instead of a silent fork.

## Design overview

### Concepts

**Background task** — a named unit of work SASE runs outside the foreground. Two kinds share one record type:

- `kind: "command"` — an argv executed by a detached supervisor process (`sase task run`, epic launch, anything a future
  surface submits). Survives TUI exit.
- `kind: "tui"` — an in-TUI Textual worker (ChangeSpec sync/mail/accept, agent launches, plugin operations). It cannot
  outlive its process, but it **mirrors** its record into the store so `sase task list` sees it and so its outcome
  survives the session that produced it.

**Session** — one running SASE TUI process. The canonical session id reuses the existing toast-session format
(`<YYYYMMDDTHHMMSSZ>-<pid>`, `sase.logs.toast_log.current_toast_session()`) so task rows and toast history join on one
id. Sessions get a short 4-character handle and a deterministic color for display.

**Task id** — a 12-character lowercase token from an unambiguous base32 alphabet, resolvable by unique prefix (minimum 3
characters), like a git short SHA. `sase task show k7m2` works.

### Storage layout

```
~/.sase/tasks/tasks.jsonl        # durable task rows (Rust-owned: locking, ordering, retention)
~/.sase/tasks/logs/<task_id>.log # append-only combined stdout/stderr, size-bounded
~/.sase/sessions/<session_id>.json  # live-session records (Python-owned)
```

The task **records** live in Rust core, per the repo's core-backend boundary: the CLI, the TUI, and any future frontend
must agree on ordering, retention, and legal status transitions. The **session registry** is deliberately Python: it is
OS process-liveness plumbing (pid + start-time checks, `/proc` reads) with no cross-surface domain rules, matching
existing precedent in `src/sase/ace/hooks/processes.py` and `src/sase/axe/maintenance.py`.

### Status lifecycle

```
pending ──▶ running ──▶ success        (exit code 0)
   │           │
   │           ├──────▶ error          (non-zero exit, spawn failure, or supervisor death)
   │           └──────▶ killed         (SIGTERM from `K` in the TUI or Ctrl-C on `--wait`)
   └──────────────────▶ error          (supervisor failed to start)
```

`pending` is honest about the window between "row written" and "supervisor took ownership". Terminal states are final:
the store rejects transitions back to `running`.

### Execution model

`sase task run` never asks a TUI to execute anything. It writes a row, forks a detached supervisor, and returns. The
supervisor owns the child process group, captures its output, and writes exactly one terminal status. This keeps tasks
alive across TUI restarts and makes `sase task run` work with no TUI running at all.

Targeting a TUI session therefore means **attribution**, not delegation: the task is stamped with a session id, that
session's Tasks tab shows it and counts it in the task indicator, and its completion toast lands there. This is the
reliable reading of "run a task in a particular TUI instance", and it degrades gracefully when the target session exits
mid-task.

### Session attribution

`--session/-s` accepts a full session id, a unique short-handle or id prefix, or one of `current`, `latest`, `none`. The
default is `current` → the current process's ACE session if it has one → otherwise `latest`, the most recently started
live ACE session → otherwise `none` (the task still runs, and lists with an em dash in the session column). This is what
makes an epic approved from Telegram land in the TUI the user is actually looking at.

### Non-goals

- No `sase task kill` / `sase task rm` subcommands in this epic. Killing stays in the TUI (`K`) and via Ctrl-C on
  `sase task run --wait`; `kill_task()` lands as API in the `runner` phase so a later CLI addition is trivial.
- The axe pane's 1–9 `bgcmd` slots are untouched. Folding them into this store is a good follow-up, not this epic.
- No remote or multi-host sessions.

---

## Rust task store and bindings

Work in the **sase-core** repo. Open it the sanctioned way and use the printed path for every read and write:

```bash
sase repo open sase-core -r "Add the background-task record store for the sase task command"
```

Model everything on `crates/sase_core/src/prompt_stash/` — it is the closest existing store (JSONL + `fs2` locks +
schema-versioned wire + atomic rewrite) and its conventions should be copied rather than reinvented.

**New module** `crates/sase_core/src/tasks/` with `mod.rs`, `wire.rs`, `store.rs`.

`BackgroundTaskWire` fields (`TASK_WIRE_SCHEMA_VERSION = 1`):

| Field                                       | Type                | Notes                                                      |
| ------------------------------------------- | ------------------- | ---------------------------------------------------------- |
| `task_id`                                   | string              | Generated by the Python caller; Rust never mints ids       |
| `label`                                     | string              | Human-facing name (`Epic launch · auth_rewrite`)           |
| `kind`                                      | string              | `command` \| `tui`                                         |
| `status`                                    | string              | `pending` \| `running` \| `success` \| `error` \| `killed` |
| `command`                                   | list[string]        | Empty for `kind: tui`                                      |
| `cwd`                                       | string              |                                                            |
| `project`                                   | optional string     | SASE project name                                          |
| `workspace_num`                             | optional int        |                                                            |
| `session_id`                                | optional string     | Origin/target session                                      |
| `session_label`                             | optional string     | Denormalized for dead-session display                      |
| `origin`                                    | string              | Producer label: `cli`, `ace`, `gate`, `epic-launch`        |
| `cl_name`                                   | optional string     | Ties a task to a ChangeSpec                                |
| `tags`                                      | list[string]        | Sorted, deduplicated                                       |
| `pid` / `pgid`                              | optional int        | Supervisor pid and the child's process group               |
| `exit_code`                                 | optional int        |                                                            |
| `phase`                                     | optional string     | Live progress line                                         |
| `message`                                   | optional string     | Terminal summary                                           |
| `created_at` / `started_at` / `finished_at` | RFC3339 UTC strings | Only `created_at` is required                              |
| `log_path`                                  | string              |                                                            |

**Public functions**

- `read_tasks_snapshot(path) -> TaskStoreSnapshotWire` — rows newest-first by `created_at` (insertion order breaks ties)
  plus store stats. A missing file is an empty snapshot, not an error.
- `append_task(path, task, history_limit) -> TaskAppendOutcomeWire { snapshot, pruned_task_ids }`.
- `update_task(path, update) -> TaskUpdateOutcomeWire { task, matched }` — partial update by id; only supplied fields
  change. Returns `matched: false` (never an error) for an id that retention already pruned, so a late supervisor write
  on a long-finished task is a no-op rather than a crash.
- `prune_tasks(path, history_limit) -> TaskPruneOutcomeWire` — standalone retention pass for when the configured limit
  shrinks.

**Rules the store enforces**

- Retention keeps every non-terminal row plus the newest `history_limit` terminal rows. Running work is never pruned
  just because it is old; a `history_limit` below 1 is clamped to 1.
- Terminal rows never transition back to `pending`/`running`. A second terminal write is idempotent-ish: last write wins
  for `message`/`exit_code`, but `finished_at` is preserved once set.
- Unknown or malformed rows are skipped on read and dropped on rewrite, never fatal — matching how
  `_toast_record_from_json` tolerates junk.
- Locking uses the `prompt_stash` shared/exclusive pattern with the same 2-second bounded timeout and jitter. Bounded
  waits are a hard requirement: the TUI calls into this store and must never block indefinitely.
- Writes go through a temp file plus rename.

**Bindings** — expose `read_tasks_snapshot`, `append_task`, `update_task`, and `prune_tasks` from
`crates/sase_core_py/src/lib.rs`, including the module-doc signature list that the other bindings maintain. Check
whether `tools/check_sase_core_rs_bindings` in the sase repo enumerates binding names and update it if so.

**Tests** (Rust, in `store.rs`): append/read round trip, retention at and beyond the limit, running rows surviving
retention, terminal-transition guards, unknown-field tolerance, concurrent writer contention, and lock timeout.

Do not touch `[workspace.package].version`, crate versions, or path-dependency pins — release-plz owns them. Use a
Conventional Commit (`feat(tasks): ...`).

Back in the sase repo, verify with `just rust-install` followed by `just rust-test`.

## SASE session registry and display identity

New package `src/sase/sessions/` (`__init__.py`, `registry.py`, `display.py`).

- `SessionIdentity(session_id, kind, pid, started_at, project, workspace_num, cwd, title)`.
- `current_session_id()` returns `sase.logs.toast_log.current_toast_session().session_id`. Do not fork the format or
  reimplement it — sharing the id is what lets a future "show me the toasts from the session that ran this task" work.
- `register_session(kind, ...)` writes `~/.sase/sessions/<session_id>.json`; `unregister_session()` removes it.
- `live_sessions()` reads every record, drops those whose pid is dead or whose process start time disagrees with the
  record (pid reuse is real), opportunistically deletes stale files, and returns the survivors newest-first. Reuse the
  liveness approach already in `src/sase/ace/hooks/processes.py` and `src/sase/axe/maintenance.py` rather than a bare
  `os.kill(pid, 0)`.
- `resolve_session_ref(ref) -> SessionIdentity | None` handles `current`, `latest`, `none`, a full id, or a unique
  short-handle/id prefix, raising a typed `SessionRefError` on ambiguity or miss with a message that lists the
  candidates.
- `short_session_handle(session_id)` — 4 lowercase base32 characters of a digest of the id.
- `session_color(session_id)` — deterministic pick from a small palette drawn from the existing ACE tribe/status colors,
  so rows from one session share a hue across CLI and TUI.
- `session_chip(identity_or_row) -> rich.text.Text` — the shared badge, e.g. `ace·sase#24 4f2a`. A dead session renders
  dim with a `†` marker; an unattributed task renders a dim `—`.

ACE registers its session during startup (next to the existing `current_toast_session()` call in
`src/sase/ace/tui/app.py`) with project, workspace number, and window title, and unregisters on the clean-quit path.
Registration must be cheap and non-fatal: a failed write logs at debug and is otherwise ignored, exactly like toast
recording.

**Tests**: id stability across calls, registry round trip, liveness pruning for a dead pid and for a reused pid,
`resolve_session_ref` for each form including the ambiguity error, and chip rendering for live/dead/absent sessions.

Public symbols here have no consumer until the `cli` and `tui` phases land. Rather than deleting or hiding them, add
`--epic-symbol <phase-bead-id>(<symbol>)` entries to the Symvision invocation in the `Justfile` for the ones Symvision
flags, and remove each entry when its consuming phase lands.

## Python task facade, ids, logs, and the history limit

New package `src/sase/tasks/` (`__init__.py`, `models.py`, `paths.py`, `ids.py`, `logs.py`, `store.py`) plus the new
config field. Keep every module well under the `toobig` thresholds (`just _lint-toobig`, 700/850/1000).

**Config** — add to `src/sase/default_config.yml`:

```yaml
tasks:
  # Maximum number of finished background tasks kept in ~/.sase/tasks. Running
  # tasks are never pruned. Lower values trim history sooner; the oldest
  # finished tasks (and their logs) are removed first.
  history_limit: 100
```

Add the matching object to `src/sase/config/sase.schema.json` (`"additionalProperties": false`, integer with
`"minimum": 1`, `"default": 100`, and a description that matches the YAML comment), a `get_task_history_limit()`
accessor modeled exactly on `get_configured_max_running_agents()` in `src/sase/config/core.py` (validate
`type(value) is int and value >= 1`, otherwise fall back to the default), and a row in `docs/configuration.md`.

**`models.py`** — frozen dataclasses mirroring the Rust wire, with `from_dict`/`to_dict` conversions that tolerate
unknown keys, following the `sase/core/*_wire*.py` conversion style.

**`ids.py`** — `new_task_id()` (12 characters from `secrets` over an unambiguous base32 alphabet, no `i`/`l`/`o`/`u`),
`short_task_id()` (first 6), and `resolve_task_ref(prefix, tasks)` requiring at least 3 characters and raising a typed
error that lists candidates on ambiguity.

**`logs.py`** — `task_log_path(task_id)`, `open_task_log(task_id)`, `read_task_log_tail(task_id, lines)` (reuse
`read_tail_seek` from `sase.axe.state`), and `delete_task_logs(task_ids)`. Bound log size with the helpers in
`sase.logs._bounded`, with an env override in the same style as `SASE_TUI_TOASTS_MAX_BYTES`.

**`store.py`** — the facade over the Rust bindings via `require_rust_binding`, mirroring
`src/sase/core/notification_store_facade.py`: `read_tasks()`, `get_task()`, `append_task()` (passes the configured
history limit and deletes the log files of returned pruned ids), `update_task()`, and `prune_tasks()`. Filtering helpers
(`status`, `session_id`, `project`, `tag`, free-text query over label/command/cl_name) live here so the CLI and TUI
filter identically.

**Tests**: config default, override, and invalid-value fallback; retention deleting log files; each filter; id
generation and prefix resolution including ambiguity; wire round-trip tolerance for unknown keys.

## Detached task supervisor and submit API

Two modules: `src/sase/tasks/runner.py` (submit/wait/kill/reconcile) and `src/sase/tasks/supervisor.py`
(`python -m sase.tasks.supervisor`, the detached child).

**`submit_task(argv, *, label, cwd, session_id, project, workspace_num, tags, origin, cl_name, env)`**

1. Validate a non-empty argv and an existing `cwd`; raise a typed `TaskSubmitError` otherwise.
2. Mint a task id and append a `pending` row.
3. Spawn the supervisor detached: `start_new_session=True`, all stdio to `DEVNULL`, `close_fds=True` — the same shape as
   today's `spawn_detached_epic_launch`.
4. If the spawn fails, mark the row `error` with the failure text and raise `TaskSubmitError`, so a failed submit is
   still visible in `sase task list` rather than vanishing.
5. Return the `BackgroundTask` record.

**Supervisor responsibilities**

- Mark `running` with its own pid and the child's pgid.
- Run the child with `stdout` and `stderr` merged into the task log, `stdin` at `DEVNULL`, its own process group, and
  these extra environment variables so any task can self-report: `SASE_TASK_ID`, `SASE_TASK_LOG_PATH`,
  `SASE_TASK_SESSION_ID`.
- Handle `SIGTERM`/`SIGINT` by terminating the child's process group (escalating to `SIGKILL` after a short grace
  period) and writing `killed`.
- In a `finally`, always write exactly one terminal status — `success` for exit code 0, `error` otherwise, with
  `exit_code`, `finished_at`, and a one-line `message`. **No path may leave a row stuck in `running`.**

**`reconcile_running_tasks()`** is the reliability keystone: any row in `pending`/`running` whose supervisor pid is no
longer alive is marked `error` with "supervisor exited without reporting". Call it at the top of `sase task list` and
`sase task show`, and from ACE's slow tick. Without it, a `kill -9` on a supervisor leaves a permanently "running"
ghost.

**`wait_for_task(task_id, *, timeout, on_line)`** polls with backoff and streams new log lines through `on_line`;
`kill_task(task_id)` signals the recorded pgid and marks the row `killed`.

**Tests**: success, non-zero exit, and killed paths driven with trivial commands; env-var propagation; log capture
including interleaved stderr; orphan reconciliation; submit failure for a bad cwd or unspawnable argv; and a supervisor
that is killed mid-run still leaving a terminal row (via reconciliation).

## The `sase task` command surface

New `src/sase/main/parser_task.py` and `src/sase/main/task_handler.py` (split rendering into a `task_render.py` if the
handler approaches the size lint), registered in `src/sase/main/parser.py` and dispatched in `src/sase/main/entry.py`
alongside the existing commands.

Follow the CLI rules: excellent `-h/--help`, alphabetically sorted subcommands and options, a short alias for every
public long option, and colored output wherever color aids reading. Bare `sase task` delegates to `sase task list`
through the central `_default_list_subcommands()` — do not re-implement the delegation — and the group description
documents that default.

**`sase task list`**

```
-a, --all              Include tasks from every session (default: this session plus unattributed)
-j, --json             Emit a stable JSON envelope
-n, --limit N          Show at most N tasks (default: the configured tasks.history_limit)
-p, --project NAME     Only tasks for this project
-q, --query TEXT       Case-insensitive substring filter over label, command, and ChangeSpec name
-r, --running          Only tasks that are pending or running
-s, --session REF      Only tasks from this session (id, prefix, current, latest, none)
-S, --status STATUS    Only tasks with this status (repeatable)
-t, --tag TAG          Only tasks carrying this tag
```

The default limit is the configured `tasks.history_limit`, so a bare `sase task` shows everything stored and `-n`
narrows it. Render a Rich table styled like `sase repo list` / `sase plan list`: status glyph (reuse the Tasks-tab icons
`● ✓ ✗` plus a distinct killed glyph), short id, label, colored session chip, project, relative start time, duration,
and exit code for failures. Terminal rows render dim; the session chip color is the visual thread tying a row to the TUI
it came from. Empty state is a friendly panel that names `sase task run`, not a bare blank line.

**`sase task show ID`**

```
-A, --all-lines        Print the whole retained log
-f, --format FORMAT    markdown (default) or json
-F, --follow           Stream new output until the task reaches a terminal state
-l, --log-lines N      Log lines to show (default: 200)
-o, --output-only      Print only the captured log (pipe-friendly, no chrome)
```

`ID` resolves by unique prefix. The default view is a header panel — label, status, full and short id, session chip,
project, cwd, the exact command, tags, created/started/finished timestamps, duration, exit code — followed by the log
tail. `--follow` on an already-terminal task prints and exits immediately rather than hanging.

**`sase task run -- COMMAND...`**

```
-c, --cwd DIR          Working directory (default: the current directory)
-j, --json             Emit the created task as JSON
-l, --label TEXT       Human-facing task label (default: derived from the command)
-p, --project NAME     Project to attribute the task to (default: inferred from cwd)
-q, --quiet            Print only the task id
-s, --session REF      Session to attribute the task to (default: current, then latest, then none)
-t, --tag TAG          Tag the task (repeatable)
-w, --wait             Stream output and exit with the task's exit code
```

Everything after `--` is the command. Without `--wait` it prints the new task id plus a one-line hint
(`monitor with: sase task show <id> --follow`); with `--wait` it streams the log and propagates the exit code, and
Ctrl-C kills the task. Each subcommand's help carries a short `examples:` epilog.

Note the deliberate divergence from `sase notify list`, which uses `-l/--limit`: `sase task list` uses `-n/--limit`
because `-l` is more valuable as `--log-lines` on `show`, and consistency within the `task` group matters more here.

**Docs** — add the three commands to the appropriate table in `docs/cli.md`, and document the storage layout, the
session-attribution rule, and `tasks.history_limit` where the CLI reference points (extend the ACE **Tasks Tab** section
in `docs/ace.md` and cross-link it).

**Tests**: parser wiring and the bare-`sase task` delegation notice, rendering of each state (including an empty store
and a dead session), the JSON envelope shape, `--limit`/filter combinations, unknown and ambiguous id errors, and
`run --wait` exit-code propagation.

## Admin Center Tasks tab over the durable store

Touches `src/sase/ace/tui/task_queue.py`, `src/sase/ace/tui/actions/task_actions.py`,
`src/sase/ace/tui/modals/tasks_pane.py`, the task indicator, the help modal, and the visual snapshots.

**Mirroring.** In-TUI tasks write store rows: an append on submit (`kind: "tui"`, `origin: "ace"`, current session), an
update on phase/status change, and a final update with the outcome on completion. Log lines flush incrementally (track
the last flushed index; never rewrite the whole log).

**All store I/O from the TUI goes through one daemon writer thread with a queue**, exactly like
`src/sase/logs/toast_log.py`. `_submit_tracked_task()` runs on the UI thread, and a locked file append there would
violate the "never block the event loop" rule. Reuse the toast-log pattern rather than inventing a new one.

**Reading.** The Tasks pane renders a merged view: in-memory tasks stay authoritative for live output, and store rows
not present in memory are rendered from their log files. Store reads happen off the event loop and are cached by store
mtime — the existing 0.25-second timer keeps driving the spinner and in-memory updates, but re-reads the store only when
its mtime changed and at a slower cadence (about 1 second), per the "periodic ticks revalidate, recomputes get a longer
cadence" rule. Nothing in a render or keystroke path may stat, read, or lock the store.

**Scope filter.** The pane defaults to this session plus unattributed tasks; `a` toggles "all sessions". The pane title
states the active scope (`Tasks · this session [2 running · 5 done]`), and rows from other sessions carry the colored
session chip from the `sessions` phase. Because `a` is a new conditional binding, follow the footer convention in
`src/sase/ace/CLAUDE.md` and update the `?` help popup — both are mandatory for any Tasks-tab keybinding change.

**Kill.** `K` keeps cancelling the Textual worker for in-memory tasks and calls `kill_task()` off-thread for
store-backed ones, with the same confirmation modal.

**Indicator.** The top-bar task indicator counts running tasks attributed to this session, including detached ones, so
an epic launch started from Telegram shows up in the TUI's count.

**Docs and snapshots.** Update the Tasks Tab section of `docs/ace.md` (new columns, scope toggle, durability, the
relationship to `sase task`). Refresh the PNG goldens for
`tests/ace/tui/visual/test_ace_png_snapshots_config_center_tasks.py` with
`just test-visual --sase-update-visual-snapshots`, and only for genuinely intended visual changes.

**Tests**: extend `tests/ace/tui/test_tasks_pane.py` for merged rendering, the scope toggle, session chips, dead-session
rendering, and killing a store-backed task; add mirroring tests that assert no store I/O happens on the event loop
(assert through the writer-thread queue).

## Route approved-epic bead creation through the task runner

Today `prepare_epic_launch(mode="detached")` calls `spawn_detached_epic_launch()`, which forks
`python -m sase.bead.epic_launch --worker ...` with all output redirected to a private log file. Replace the fork with a
submitted background task; keep every behavior that follows it.

- Add `submit_epic_launch_task(plan_file, *, cwd, artifacts_dir, cl_name, session_id)` to
  `src/sase/bead/epic_launch.py`. It calls `submit_task()` with argv
  `[sys.executable, "-m", "sase.bead.epic_launch", "--worker", "--plan-file", ..., "--cwd", ...]`, label
  `Epic launch · <plan stem>`, tags `["epic", "launch"]`, `origin="epic-launch"`, and the approval's `cl_name`. Call the
  submit API directly rather than shelling out to `sase task run` — it is the same code path the CLI uses, without a
  PATH dependency or an extra process hop.
  `sase task run --label 'Epic launch · <plan>' -- sase bead work <plan> --yes-to-all` remains the equivalent hand-run
  form, and the plan's docs should say so.
- Change `_run_detached_worker()` to let `sase bead work` inherit stdout/stderr — the supervisor captures them into the
  task log, so the epic launch becomes **live** output instead of a file read after the fact. Resolve the log path for
  the completion notification's `extra_files` from `SASE_TASK_LOG_PATH`. Keep parsing, `update_epic_launch_metadata()`,
  and `notify_workflow_complete()` exactly as they are, and keep accepting `--log-path` so a worker spawned by an older
  version that is still in flight during an upgrade behaves.
- `prepare_epic_launch()` keeps its `detached` / `foreground` / `skip` contract and its return-value semantics. On a
  submit failure it still raises `PlanApprovalActionError("epic_launch_failed", ...)` carrying the
  `sase bead work <plan> --yes-to-all` resume hint. `mode="foreground"` is unchanged.
- Attribution: when the process resolving the gate is an ACE session (TUI notification panel) the task attaches to it;
  from Telegram or `sase gate` it attaches to the newest live ACE session, else runs unattributed. This is what makes a
  Telegram approval visible in the TUI the user is looking at.
- Update the epic choice's `consequence_text` in `src/sase/plan_approval_choices.py` — it currently says
  "`sase bead work` (background task)" — so it names where the task is now visible.

**Tests**: the submitted argv and label; the notification's `extra_files` pointing at the task log; the failure path
raising with the resume hint; and `foreground` mode untouched. Extend the existing epic-launch tests rather than
creating a parallel suite.

## End-to-end transparency exercise

Exercise the whole feature against real state, then fix what it surfaces.

1. Approve a throwaway epic plan from a TUI gate notification panel, and again through `sase gate` with no TUI in the
   resolving process. In both cases confirm the launch appears in `sase task list`, streams under
   `sase task show <id> --follow`, and shows in the Tasks tab with the right session chip.
2. Run a plain `sase task run -- <command>` with and without `--wait`, with `--json`, and with a command that fails,
   confirming the exit code propagates and the failure renders clearly.
3. Kill a running task from the Tasks tab (`K`) and confirm the row becomes `killed` in both surfaces.
4. Force retention: set `tasks.history_limit` low, generate more finished tasks than that, and confirm the oldest rows
   and their log files are gone while running tasks survive.
5. `kill -9` a supervisor and confirm reconciliation converts the ghost row to `error` rather than leaving it running.

Report what was exercised and what was fixed.

## Testing and verification

`sase_<N>` workspaces are ephemeral and may have drifted dependencies, so **run `just install` first** in every phase.

- Every phase: `just check` and `just test`.
- After Rust changes: `just rust-install` (or `just install`) before Python tests, plus `just rust-test`.
- The `tui` phase: `just test-visual`, refreshing goldens only for intended changes.
- New Python public symbols with no consumer until a later phase get `--epic-symbol <bead_id>(<symbol>)` entries in the
  `Justfile` Symvision invocation, removed as each consuming phase lands.

## Risks

| Risk                                              | Mitigation                                                                                                                                                                  |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Store lock contention blocking the TUI event loop | Bounded 2-second lock waits in Rust; all TUI store writes go through one daemon writer thread; no store I/O in render or keystroke paths                                    |
| Rows stuck in `running` after a crash             | Supervisor writes a terminal status in a `finally`; `reconcile_running_tasks()` sweeps orphans from the CLI and ACE's slow tick                                             |
| Pid reuse making a dead session look live         | Liveness checks compare process start time, not just pid existence                                                                                                          |
| Visual snapshot churn from Tasks-tab changes      | Regenerate goldens in the `tui` phase only, and review the diff artifacts under `.pytest_cache/sase-visual/` before accepting                                               |
| Epic-approval regression                          | `prepare_epic_launch()` keeps its contract, the worker keeps its parsing/metadata/notification duties, and the existing epic-launch tests are extended rather than replaced |
