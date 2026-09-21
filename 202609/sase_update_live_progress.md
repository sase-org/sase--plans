---
tier: epic
title: Live, streaming progress for sase update
goal: '`sase update` shows every step as it happens: a live timeline on terminals
  and an append-only log when output is piped. It streams the tail of slow subprocess
  output (cargo, uv), shows each checkout''s commits and diffstat as it fast-forwards,
  expands the failing step''s output on error, survives Ctrl-C cleanly, and always
  leaves a full log file behind. The JSON and quiet contracts stay stable.

  '
phases:
- id: stream-runner
  title: Streaming subprocess runner
  depends_on: []
  size: medium
  description: 'stream-runner: add a line-streaming subprocess primitive with sanitization,
    timeout, and process-group interrupt handling, and plumb an optional on_output
    sink through run_dev_update_command, run_recorded_command, and run_uv without
    changing their existing behavior.'
- id: progress-model
  title: Progress event protocol, timeline model, and renderers
  depends_on: []
  size: medium
  description: 'progress-model: create the sase.update_progress package: the UpdateProgress
    event protocol plus a null sink, a thread-safe timeline model, the rich Live renderer,
    the plain append-only renderer, the full-run log file sink, and a fan-out sink.'
- id: instrument-backends
  title: Emit progress events from dev-update, uv, and mode-switch backends
  depends_on:
  - stream-runner
  - progress-model
  size: medium
  description: 'instrument-backends: thread an optional progress sink through plan_dev_update,
    execute_dev_update (fetch, preflight, merge, reconcile), and execute_mode_switch.
    Emit step events with friendly titles and result details, and route subprocess
    output into the running step.'
- id: wire-live-update
  title: Wire the live timeline into the sase update live path
  depends_on:
  - instrument-backends
  size: medium
  description: 'wire-live-update: add the -v/--verbose flag and renderer selection,
    declare the step timeline, run the managed uv upgrade as a streamed step with
    live package rows, add restart and completion rows, print the final frame and
    a deduplicated summary, expand output on failure, handle Ctrl-C with exit 130,
    and add log_path to the JSON output.'
- id: wire-mode-switch-dry-run-docs
  title: Mode switch, dry-run, and documentation
  depends_on:
  - wire-live-update
  size: small
  description: 'wire-mode-switch-dry-run-docs: move --to mode switches onto the same
    live session after confirmation, show a transient timeline while --dry-run plans
    its fetches, and update the plugins.md and cli.md docs with the new output and
    the --verbose flag.'
proposed_by: bbugyi200.apollo.1d
create_time: 2026-09-21 07:49:16
status: wip
bead_id: sase-158
---

- **PROMPT:** [prompts/202609/sase_update_live_progress.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_update_live_progress.md)
- **BEAD:** [sase-158](https://github.com/sase-org/sase--beads/blob/main/pages/sase-158/README.md)

# Plan: Live, streaming progress for `sase update`

## Problem

`sase update` is silent for most of its runtime. On the editable (dev) path, the command
plans (one `git fetch` per checkout, sequentially), fetches again, fast-forwards, and
then runs minutes-scale reconcile steps: uv reinstall, a Rust prebuild-cache consume, a
possible cargo rebuild of `sase-core-rs` with a timeout of up to 3600 s, and a health
check. Nothing is printed until the end. The managed path shows only one spinner around
`uv tool upgrade sase`. Every subprocess runs with `capture_output=True`, so the user
can't tell a slow cargo build from a hang. When something fails, all they get is a
one-line reason. The finished panel also shows a flat "executed commands" table and
doesn't say where time went.

## Design principles

1. **One mental model: a timeline of steps.** Every run is an ordered list of
   human-titled steps. Each step is `pending ○`, `running ⠋`, `done ✓`, `warned ⚠`,
   `failed ✗`, `skipped –`, or `interrupted ■`. The user sees the whole journey up front
   (pending rows) and knows where they are.
2. **Show the right amount of output.** A running step shows its elapsed clock and a
   3-line dim tail of its live output (`│ Compiling sase_core v0.14.2`). Finished steps
   collapse to a one-line result detail. A failed step expands to its last 20 output
   lines. `-v/--verbose` streams all output.
3. **stdout stays for results, stderr carries progress.** Live and plain progress go to
   the stderr console. The final summary panels stay on stdout, as they do today. So
   `sase update > out.txt` still shows live progress on the terminal, and pipes stay
   clean.
4. **Stable contracts.** `-j/--json` emits no progress at all. Its payload is unchanged
   apart from an additive `log_path` key, so `UPDATE_JSON_SCHEMA_VERSION` stays the
   same. `-q/--quiet` still prints only its one-line summary. Exit codes are unchanged,
   plus a new 130 for Ctrl-C.
5. **Reliable first.** Progress is purely observational. The default sink is a no-op,
   and no rendering failure can fail an update (renderer exceptions are caught and the
   renderer degrades to the plain sink). Subprocess output is sanitized and inserted as
   `rich.text.Text` (never markup), so a `[red]` in cargo output can't inject styles or
   crash the renderer.
6. **Beautiful but restrained.** Use the existing sase update grammar: glyphs backed by
   words, and green/cyan/yellow/red/dim styling where color adds meaning but is never
   load-bearing. Use a rounded cyan panel. The title carries the install mode, and the
   right-aligned subtitle carries the overall elapsed clock.

This work is presentation plus the Python orchestration that already lives in this repo
(`sase.dev_update`, `sase.uv_tool`, `sase.mode_switch`, `sase.main.update_*`). It adds
no shared domain behavior, so nothing crosses into `sase-core`.

## Target experience

Live TTY frame, editable install, mid-rebuild (80 columns):

```text
╭─ sase update · dev install ───────────────────────────────────────────────╮
│ ✓ Inspect install                  uv tool · 5 editable · 1 managed  0.4s │
│ ✓ Check for updates                2 behind · 1 current              2.1s │
│     ✓ sase          behind 4 · origin/master                              │
│     ✓ sase-core     behind 1 · origin/master                              │
│     · sase-github   current                                               │
│ ✓ Fast-forward checkouts                                             0.9s │
│     ✓ sase          a1b2c3d → 9f8e7d6 · 4 commits · +212 −40              │
│     ✓ sase-core     77aa12b → 01bc9e3 · 1 commit · +12 −3                 │
│ ✓ Reinstall editable Python packages                                 6.3s │
│ ⚠ Install prebuilt Rust core       cache miss · stamp-missing        0.2s │
│ ⠼ Rebuild Rust core into uv-tool venv                                0:31 │
│     │    Compiling sase_core v0.14.2                                      │
│     │    Compiling sase_core_py v0.14.2                                   │
│     │    Compiling sase_xprompt_lsp v0.14.2                               │
│ ○ Verify sase-core-rs imports                                             │
│ ○ Restart scheduler                                                       │
│ ○ Refresh shell completions                                               │
╰──────────────────────────────────────────────────────────────── 0:44 ─────╯
```

Managed install: one `Upgrade sase + plugins via uv` step. Its tail streams uv's own
lines (`Resolved 84 packages in 1.2s`, `Prepared 3 packages`, …). Each `+ pkg==ver` line
that appears turns into a live child row (`✓ sase-github 0.3.2 → 0.4.0`, pairing with
any earlier `- pkg==old` line).

Completion: the live region disappears (transient) and a static final frame is printed
to stderr in its place. The static frame keeps the step rows, children, and durations,
drops the tails, and adds a `slowest: <step> (<dur>)` footer. Then the existing stdout
result panels print, minus the redundant executed-commands table and slowest line
whenever a timeline was shown. After that come the restart and completion lines.

Failure: the failed step is red `✗` with its reason as detail and its last 20 output
lines expanded under it. Later pending steps render as `– skipped`. The error panel ends
with `Full log: ~/.../logs/update/<timestamp>.log`.

Ctrl-C: the running step's process group gets SIGINT and is killed after a grace period.
The step renders `■ interrupted`. The code-swap lock is released by its context manager,
the final frame and `Interrupted — full log: …` are printed, and the command exits 130.

Plain mode (stderr not a terminal, or `TERM=dumb`): append-only lines with no cursor
movement:

```text
sase update · dev install
[00:00] → Inspect install
[00:00] ✓ Inspect install — uv tool · 5 editable · 1 managed (0.4s)
[00:02] ✓ Check for updates — 2 behind · 1 current (2.1s)
[00:03] ✓ Fast-forward sase — a1b2c3d → 9f8e7d6 · 4 commits · +212 −40
[00:10] → Rebuild Rust core into uv-tool venv
[00:41] ✓ Rebuild Rust core into uv-tool venv (30.8s)
```

Plain mode prints a start line only for steps that are still running after 2 s, so a
hang is always visible without doubling every line. `-v` adds `    │ <line>` for every
output line in both modes (in live mode, verbose output is printed above the live region
through `Live.console.print`).

## Architecture

```text
backends (dev_update / uv_tool / mode_switch)
   │  UpdateProgress events (step_start / step_output / step_finish / declare)
   ▼
FanOutProgress ──► TimelineModel (thread-safe) ──► LiveTimelineRenderer | PlainTimelineRenderer
            └────► UpdateLogSink (full transcript file)
```

- Backends only know the `UpdateProgress` protocol. The default is `NULL_PROGRESS`, so
  the Admin Center Updates tab and other callers are untouched. (Hooking the TUI up to
  the same events is a possible follow-up, out of scope here.)
- Subprocess output reaches the model through the new streaming runner's `on_output`
  callback. The callback is bound to the current step by
  `progress.output_sink(step_id)`.

## Phase: stream-runner — Streaming subprocess runner

Add a streaming primitive and plumb it through the existing runners. Behavior with no
sink stays byte-for-byte the same.

- New module `src/sase/dev_update/stream_command.py` exposes
  `run_streaming(argv, *, cwd, env, stdin, timeout, on_line) -> subprocess.CompletedProcess[str]`:
  - `Popen` with `stdout=PIPE, stderr=PIPE`, `start_new_session=True`, and
    `PYTHONUNBUFFERED=1` merged into the child env (keep a caller-supplied env
    otherwise). Two pump threads read bytes and decode with `errors="replace"`. They
    split on `\n` and treat `\r` as a line reset (keep the last segment) so uv/cargo
    progress redraws don't flood. Each finished line goes through a `sanitize_line`
    helper that strips ANSI/OSC escape sequences and other control characters, and
    `on_line(stream, line)` is called with it. Pump threads block signals the same way
    `sase.tool.executor._pump_child_stream` does.
  - It accumulates full stdout/stderr text (unsanitized) and returns a
    `CompletedProcess` so `run_with_git_lock_retry` keeps working unchanged.
  - Timeout: SIGTERM the process group, wait 1 s, SIGKILL, then raise
    `subprocess.TimeoutExpired` with partial output (callers already map this to rc 124
    / `UvCommandFailedError(timeout=...)`).
  - `KeyboardInterrupt` in the waiting thread: send SIGINT to the process group, wait up
    to 2 s, SIGKILL, join the pumps, and re-raise. A callback exception is swallowed
    after the first failure (the sink gets disabled for the rest of that command) so
    progress can never break a command.
  - Reuse or align with `sase.noninteractive_subprocess` process-group helpers instead
    of duplicating kill logic where practical.
- `run_dev_update_command(...)` gets `on_output: OutputSink | None = None`. `None` keeps
  the current `subprocess.run(capture_output=True)` path. When set, it uses
  `run_streaming`, keeping git's non-interactive env and `stdin=DEVNULL`.
  `OutputSink = Callable[[str, str], None]` (`stream`, `line`) lives in
  `sase.dev_update.models`. Extend the `DevCommandRunner` Protocol with the optional
  keyword.
- `run_recorded_command(...)` gets `on_output=None` and forwards it only when set, using
  the same pattern as `env`/`timeout`. That way injected test fakes that don't accept it
  keep working when no progress is active.
- `run_uv(argv, *, run_fn=..., timeout=..., on_output=None)`: when set, it runs through
  `run_streaming` and feeds the same `parse_uv_output` on the combined text. Error
  mapping stays identical.
- Tests (`tests/dev_update/test_stream_command.py`, plus extending the uv runner tests)
  use real tiny `python -c` children. Cover: interleaved stdout/stderr lines delivered
  in order per stream, `\r` collapsing, ANSI stripping, invalid UTF-8, timeout →
  `TimeoutExpired` plus the group killed (grandchild `sleep` is gone), a raising
  callback not failing the command, `KeyboardInterrupt` delivered via a callback that
  raises it → child group terminated and the exception propagates, and git
  lock-retry-compatible return type.

## Phase: progress-model — Progress event protocol, timeline model, and renderers

New package `src/sase/update_progress/` (presentation-only; imports only rich and
stdlib; no sase backend imports, so it can't create cycles):

- `events.py`:
  - `StepStatus = Literal["pending","running","done","warned","failed","skipped","interrupted"]`.
  - `StepSpec(id, title, parent_id=None)`.
  - `UpdateProgress` Protocol: `declare(specs)`,
    `start(id, *, title=None, detail=None)`, `output(id, stream, line)`,
    `finish(id, status, *, detail=None)`, and `output_sink(id) -> OutputSink`.
  - `NULL_PROGRESS` no-op.
  - A `step(progress, id, title)` context manager that marks `failed` on exception (and
    `interrupted` on `KeyboardInterrupt`) and re-raises.
  - Starting an undeclared id auto-appends it, so backends never crash on a missing
    declaration.
  - Finishing a parent auto-finishes still-running children as `skipped`.
  - `finalize(status_for_pending="skipped")` marks leftovers.
- `timeline.py`: `TimelineModel`, which implements the protocol under a
  `threading.Lock`. It keeps ordered steps with children, `started_at`/`ended_at` from
  an injected clock, `detail`, and a per-step `deque(maxlen=200)` tail. `snapshot()`
  returns immutable row data for renderers. Pure and deterministic under a fake clock.
- `render_live.py`: `LiveTimelineRenderer(console, model, *, verbose, clock)`, a context
  manager around `rich.live.Live(transient=True, refresh_per_second=10, console=err)`.
  - The frame is a `Panel(box=ROUNDED, border_style="cyan")` titled
    `sase update · <mode>` (mode set via `set_header`), with the elapsed clock as a
    right-aligned subtitle.
  - The table has columns glyph · title (ellipsis) · detail (dim, ratio 1, ellipsis) ·
    duration (right). Running steps use a `rich.spinner.Spinner("dots")` glyph and a
    ticking `m:ss` clock. Child rows are indented 4. The live tail is the last 3 lines,
    rendered as `Text` prefixed with a dim `│`.
  - Row budget: if the frame would exceed `console.height - 2`, collapse finished
    children of finished parents into a `+N more` line first, then tails. The running
    step and failures are never hidden.
  - `print_final(expand_failures=True)` prints the static frame: no spinner, tails
    dropped, failed steps with their last 20 lines, and a `slowest:` footer when more
    than one step ran.
  - Verbose mode prints each output line above the live region via
    `live.console.print(Text(...))`.
  - Any exception during rendering or refresh stops Live and switches to the plain
    renderer for the rest of the run.
- `render_plain.py`: `PlainTimelineRenderer(console, model, *, verbose, clock)`:
  - Prints the header line.
  - Prints `[mm:ss] ✓ <title> — <detail> (<dur>)` on finish.
  - Prints a `[mm:ss] → <title>` start line from a lightweight 1 s ticker thread once a
    step has been running 2 s (only once per step).
  - Prints failure tails (last 20 lines) and verbose lines.
  - No color codes unless the console is a terminal (rich handles this); respects
    `NO_COLOR`.
- `log_sink.py`: `UpdateLogSink`.
  - Writes a plain-text transcript to
    `sase_subdir("logs") / "update" / "update-<UTC %Y%m%dT%H%M%SZ>-<pid>.log"`: header
    (argv, sase version, mode), every step start/finish with timestamps, the full
    command line and cwd of each streamed command (fed through a
    `command(id, argv, cwd)` event, a no-op on other sinks), and all output lines.
  - Best-effort: any `OSError` disables the sink silently.
  - Retention keeps the newest 20 `update-*.log` files.
  - Exposes `path` (or `None`).
- `fanout.py`: `FanOutProgress(*sinks)`, which forwards every event and isolates each
  sink's exceptions.
- `session.py`: `select_renderer(err_console, *, as_json, quiet, verbose)` returns
  `"live" | "plain" | "none"`. It picks live when
  `err.is_terminal and not err.is_dumb_terminal`, plain otherwise, and none for
  json/quiet. (The log sink runs in every mode, json and quiet included.) Also add an
  `UpdateProgressSession` context manager that owns model + renderer + log sink and
  exposes `.progress`, `.log_path`, `.print_final()`, and `.shown` (whether a timeline
  was displayed).
- Tests (`tests/update_progress/`): model transitions and auto-behaviors under a fake
  clock; thread-safety smoke (pump-thread outputs while the main thread finishes);
  `Console(record=True, width=80/40, force_terminal=True)` frame exports for the target
  layout, including narrow-width ellipsis, the row budget, markup-like output
  (`[red]x[/]` rendered literally), and the failure expansion; plain renderer lines with
  fake clock and ticker; log sink content, retention, and an unwritable-dir fallback;
  fan-out isolation; renderer fallback when a render raises.

## Phase: instrument-backends — Emit progress events from dev-update, uv, and mode-switch backends

Add `progress: UpdateProgress = NULL_PROGRESS` keyword parameters. Nothing changes when
it's left at the default, and result dataclasses and journal records are untouched.

- Step ids are stable strings: `inspect`, `check`, `check:<root>`, `merge`,
  `merge:<root>`, `reconcile:<index>`, `managed`, `managed:<pkg>`, `restart`,
  `completions`, and `switch:<index>` for mode switch. Titles use the repo display name
  (checkout directory basename, disambiguated by parent dir when basenames collide), not
  the full path. Full paths go to the log.
- `plan_dev_update(..., progress=...)`: wrap the root refresh loop in the `check` step
  with one `check:<root>` child per root. Child detail is `behind N · <upstream>`,
  `current`, `dirty — skipped`, `fetch failed; using cached ref` (warned), and so on,
  derived from the same classification used for the plan. The parent detail summarizes
  `N behind · M current · K skipped`. Update `PlanDevFn` and `call_plan_dev_update` to
  forward `progress` only when not null, so existing fakes keep working.
- `execute_dev_update(plan, *, run, clock, progress=...)`:
  - Declare the plan's steps up front: `merge` with each actionable root as a child,
    plus one row per reconcile step. That's what makes the pending rows visible.
  - The execute-time fetch and preflight run under each root's `merge:<root>` child with
    detail `fetching…` / `checking…`, not as new rows. The user already saw "Check for
    updates", and a second fetch row would confuse.
  - On success the child detail becomes `old7 → new7 · N commits · +I −D` from the
    existing head/diffstat/commit-log capture.
  - Preflight failures finish the child `failed` with the existing reason.
  - The code-swap deferral finishes `merge` as `warned` with the deferral reason.
- `run_reconcile_steps(..., progress=...)`:
  - Map step kinds to titles: `uv_tool_install` → "Reinstall editable Python packages",
    `rust_prebuild_install` → "Install prebuilt Rust core",
    `rust_dev_install`/`rust_install_uv_tool` → "Rebuild Rust core into uv-tool venv",
    `rust_health_check` → "Verify sase-core-rs imports", `rust_lsp_install` → "Install
    xprompt LSP". Fall back to `step.label`.
  - Prebuild details are `cache hit` (done) or `cache miss · <reason>` (warned).
  - A rebuild skipped due to a cache hit becomes `skipped · prebuilt artifacts used`.
  - A failure deferred to the health check shows `warned` with the pending failure.
  - Repair runs appear as child rows of the health check.
  - Each command runs with `on_output=progress.output_sink(step_id)` and emits
    `command(id, argv, cwd)` for the log.
- `execute_mode_switch(plan, *, run_uv_fn, run_command_fn, progress=...)`: declare one
  row per available command using its existing label, and stream output into it.
  Failures finish the row `failed` before raising the existing restore-hint error.
- The managed uv leg lives in the handler (next phase). Here, only make sure `RunUvFn`
  call sites can pass `on_output` by adding an optional keyword to the `RunUvFn` type
  alias's protocol (convert to a Protocol) and forwarding it only when set.
- Tests: extend `tests/dev_update/` executor, reconcile, and plan tests with a recording
  progress fake. Assert event order, statuses, and details for: all-current, a
  successful ff + rebuild, prebuild hit skipping the rebuild, preflight dirty failure,
  code-swap deferral, health-check repair, and streamed output reaching the right step.
  Add mode-switch execution event tests. Existing tests must pass unchanged, proving the
  null default is inert.

## Phase: wire-live-update — Wire the live timeline into the `sase update` live path

- Parser (`src/sase/main/parser_update.py`): add `-v/--verbose` ("Stream full output of
  every step (git, uv, cargo) as it runs"), keeping options alphabetical
  (`-j -n -q -t -v -y`). Add a `sase update -v` epilog example. `-v` with `-j` or `-q`
  is accepted and only affects the log. Follow the CLI rules note.
- `handle_update_command`: build an `UpdateProgressSession` from
  `select_renderer(err, as_json=..., quiet=..., verbose=...)`. Keep the session
  injectable for tests (`progress_session_factory=None` → default). Run the whole
  live/mode-switch flow inside it.
- **Self-update safety:** the update fast-forwards (editable) or replaces (uv) the very
  code the running process imports. Import every module the rest of the run needs
  (`sase.update_progress.*`, the renderers, `update_render`, `update_restart`, rich
  submodules such as `rich.live`/`rich.spinner`) at module top level in the handler
  modules, never lazily after execution starts. Add a test that asserts these modules
  are in `sys.modules` after importing `sase.main.update_handler`.
- `handle_live_update`:
  - `inspect` step around receipt load, dev routing, and preflight. Detail is
    `uv tool · N editable · M managed`, and the header mode is `dev install` /
    `managed install` / `mixed install`.
  - Pass `progress` into planning and execution.
  - Managed leg: replace the `out.status` spinner with a `managed` step titled "Upgrade
    sase + plugins via uv". Pass `on_output` to `run_fn` only when a session is active.
    A small line watcher parses `+/- name==ver` lines (reuse `_CHANGE_RE` from
    `sase.uv_tool.runner` via a public helper) into `managed:<pkg>` child rows as they
    stream. The step's final detail is `N upgraded · M current` from the
    `UpdateSummary`.
  - `restart` step around `restart_after_update`: detail is the restart message, or
    `skipped · no code changed`/`not running`. `completions` step around the completion
    refresh.
  - On a timeline-shown run, call `session.print_final()` before printing the stdout
    summary panels, and pass `timeline_shown=True` into `render_dev_update_result` so it
    omits the executed-commands table and slowest line (keep them when no timeline was
    shown, e.g. plain-less quiet or tests).
  - Failure paths (`fail_update`, the dev failure branch, `UvToolError` from uv): finish
    the active step `failed` with the error text, print the final frame with the failure
    tail, then the existing error output plus a dim `Full log: <path>` line when a log
    exists.
  - `KeyboardInterrupt` anywhere inside the session: mark the running step
    `interrupted`, print the final frame, print `Interrupted — full log: <path>` to
    stderr, append the dev journal with a new restart reason `interrupted` where a plan
    exists, and return 130. No traceback.
  - JSON: add `log_path` (string or null) to live-update and mode-switch payloads
    (additive; keep the schema version) and document it.
- Tests (`tests/main/test_update_command_*.py`):
  - Live and plain frames via recorded consoles with a fake clock and fake runners that
    emit output lines.
  - Assert stdout summary content is unchanged in non-timeline mode.
  - Assert JSON has no progress text on either stream and includes `log_path`.
  - Assert quiet prints one line.
  - Failure expansion, and interrupt → exit 130 with a final frame.
  - Managed live package rows from streamed `+`/`-` lines.
  - Existing update tests must stay green (adjust only those asserting an empty stderr,
    making them explicit about renderer mode).

## Phase: wire-mode-switch-dry-run-docs — Mode switch, dry-run, and documentation

- `handle_mode_switch`: the confirmation prompt and plan preview stay **before** any
  live region starts. Never prompt inside Live. After confirmation, run
  `execute_mode_switch(..., progress=session.progress)` inside the session with header
  `sase update · switch to <dev|pypi>`, plus the same `restart` and `completions` rows,
  final frame, failure expansion (the restore hint stays in the error panel), and
  interrupt handling as the live path.
- `handle_dry_run`: planning fetches every checkout, so show a **transient** live
  timeline (`inspect`, `check` with per-root children) on terminals only. The existing
  dry-run panel remains the only persistent output, printed after the region is torn
  down. No log file for dry runs. Plain mode prints nothing extra for dry runs, to keep
  previews terse.
- Docs:
  - `docs/plugins.md` "Updating sase and plugins": replace the typical-output sample
    with the new live-frame/final-frame sample and a plain-mode sample. Document `-v`,
    stderr progress vs stdout results, the log file location and retention, Ctrl-C
    behavior (exit 130, checkouts never left mid-merge beyond what git itself
    guarantees), and `log_path` in JSON.
  - `docs/cli.md` `sase update` row: mention `-v` streams full step output.
- Tests: mode-switch session events and the prompt-before-live ordering (a fake confirm
  asserting no Live is active); dry-run transient timeline leaves only the dry-run panel
  in recorded output.

## Verification (every phase)

Run `just fmt` then `just check` (prefer `sase tool run check`). No TUI PNG goldens are
affected. At the end, manually sanity-check the real command in a terminal with
`sase update -n`, and with `sase update` on an editable install when safe, both piped
(`sase update 2>&1 | cat`) and live.
