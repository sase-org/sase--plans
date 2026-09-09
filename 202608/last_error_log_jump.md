---
tier: epic
status: done
title: Jump to the last registered error with the ,L leader chord
goal: "When a SASE agent launch (or chop) fails, the error toast names a leader chord,
  and pressing that chord opens the Admin Center Logs tab already scrolled to the exact
  log entry for that failure instead of leaving the user to hunt through log sources.

  "
phases:
  - id: keymap
    title: Leader `,L` opens the Logs tab
    depends_on: []
    size: small
    description: "keymap: register the new `jump_to_last_error` leader action on `L`
      across the keymap dataclass, default config, dispatcher, command catalog, footer,
      help modal, and docs, with a first behavior that opens the Admin Center Logs tab.

      "
  - id: registry
    title: Registered errors and error-anchored launch logs
    depends_on:
      - keymap
    size: medium
    description: "registry: stamp every launch-failure log entry with a stable error id,
      add the session-scoped registered-error pointer, make one helper both register the
      error and emit its toast so the chord hint can never appear without a target, and
      make `,L` select the registered error's log source.

      "
  - id: focus
    title: Logs pane focuses the registered error entry
    depends_on:
      - registry
    size: medium
    description:
      "focus: locate the registered error's anchor in a bounded tail scan, render a
      window containing it with the entry line highlighted, scroll the detail pane to
      it, and degrade with an in-pane notice when the entry has aged out."
proposed_by: bbugyi200.athena.07n
bead_id: sase-qw
create_time: 2026-09-09 19:50:41
---

- **PROMPT:**
  [prompts/202608/last_error_log_jump.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/last_error_log_jump.md)
- **BEAD:**
  [sase-qw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qw/README.md)

# Plan: Jump to the last registered error with the `,L` leader chord

## Problem

A failed agent launch shows a toast that reads
`Launch failed - see Logs in SASE Admin Center (#)`. Acting on it means pressing `#`,
walking to the Logs tab, picking the right source out of nine, and then scrolling a
500-line tail looking for the block that corresponds to the toast that just appeared.
Nothing in the toast identifies which entry it refers to.

The fix is a dedicated leader chord, `,L`, that jumps straight to the specific log entry
for the most recently _registered_ error, plus an invariant: an error is registered
exactly when a toast tells the user to press `,L`, so the chord always has a real
target.

## Current behavior (verified)

- `src/sase/ace/tui/actions/failure_messages.py` defines
  `LOG_PANEL_HINT = "see Logs in SASE Admin Center (#)"` and `with_log_panel_hint()`.
- There are exactly seven call sites of `with_log_panel_hint()`, in two files:
  - `src/sase/ace/tui/actions/agent_workflow/_launch_procs.py:110` — `"Launch failed"`.
  - `src/sase/ace/tui/actions/axe_chop_run.py:188,233,238,243,248,253` — chop launch
    exception plus the `failure` / `timeout` / `missing_script` / `check_error` /
    `action_failed` outcome statuses.
- Every one of those paths also writes a durable record through
  `sase.logs.log_launch_failure()` (`src/sase/logs/launch_log.py`), which appends a
  machine-readable line to `launch_failures.jsonl` and a human block to
  `launch_failures.log`. All three call sites run inside the ACE process (some on worker
  threads via `asyncio.to_thread`).
- The human block starts with a `=` rule and a header line shaped
  `[2026-06-17 14:30:00 UTC] single launch failure: alpha`. There is no per-entry
  identifier in either file today.
- The Logs tab (`src/sase/ace/tui/modals/logs_pane.py`) is a two-panel pane: a source
  `OptionList` on the left and a `Static` inside a `VerticalScroll` on the right. The
  detail body is built off the UI thread by `_build_log_pane_load_result()` and rendered
  by `render_log_detail()` (`logs_pane_render.py`), which reads a bounded 500-line tail
  via `LogSource.read_rendered_tail()`. There is no way to address an individual entry.
- Leader mode already has a very close analogue: `,n` (`jump_to_notification`). Follow
  its registration surface exactly.
- `,L` was previously bound to a retired `log_panel` action. `log_panel` stays in
  `_RETIRED_LEADER_KEYS` (`src/sase/ace/tui/keymaps/registry.py`) and its guard tests
  (`tests/test_keymaps_defaults.py::test_leader_mode_drops_log_panel_key`,
  `tests/ace/tui/test_log_panel_keymap.py::test_stale_log_panel_override_is_filtered_out`)
  must keep passing. This epic introduces a **new** action id on the same key; it must
  not resurrect `log_panel`.

## Decisions made up front

Implementing agents should treat these as settled and not re-litigate them.

**Action id.** `jump_to_last_error`, bound to `L` in `leader_mode`. Never reuse
`log_panel`.

**Where the code lives (Rust core boundary).** This stays in Python. The launch-failure
log format is already a deliberate Python-side contract — see the module docstring of
`src/sase/logs/launch_log.py`, which records the decision to keep this logging in
`sase.logs` rather than the Rust core. The new session pointer is ACE-process
presentation state (which toast the user just saw), and the pane rendering is Textual
glue. Nothing here belongs in `../sase-core`.

**Session-scoped, not durable.** The "last registered error" pointer lives in memory in
the ACE process. The _entry itself_ stays durable in `launch_failures.log` / `.jsonl`
(and gains a stable id, which is the durable half). Seeding the pointer from disk at
startup was considered and rejected: `,L` would then point at a days-old failure that
the user never saw a toast for, which breaks the one invariant that makes the chord
trustworthy.

**The invariant is structural, not documentary.** A single helper both registers the
error and emits the toast. `LOG_PANEL_HINT` / `with_log_panel_hint` are deleted, so
there is no way to write the "go look at the logs" hint without registering a target.

**No feature flag.** Each phase ships a coherent, honest behavior: phase `keymap` gives
`,L` = "open the Logs tab" and says exactly that in the footer and docs; phase
`registry` upgrades it to "open the registered error's source" and the toast promises no
more than that; phase `focus` delivers the entry-level jump. No phase ships a promise it
does not keep, so there is no unready user-reaching path to gate. (Per
`sase/memory/sase_flags.md`, a flag is for a disabled beta or an early landed path that
does not yet work.)

**Symvision.** Order the phases as listed and every public symbol gains a real non-test
consumer inside the phase that introduces it, so no `--epic-symbol` whitelist entry
should be needed. If a phase does need one, add it to the Symvision invocation in the
`Justfile` and remove it in the phase that wires up the consumer.

**TUI perf rules that apply** (`sase/memory/tui_perf.md`): all log file I/O stays inside
the pane's existing `run_worker(..., thread=True, exclusive=True)` load — never add a
read to an action handler or a render path. `call_after_refresh` callbacks must stay
thin and synchronous. The existing `ProgrammaticSelectionGuard` around `OptionList`
selection must keep wrapping every programmatic highlight change.

## Verification for every phase

Run `just install` first — these workspaces are ephemeral and dependencies drift. Then
`just check` after making changes. Before the epic's combined tree lands, run
`just check-full` through `/sase_monitor` (never inline), with a `--next` action.

---

## Leader `,L` opens the Logs tab

Register the new leader action everywhere a leader action must be registered, and give
it a first, honest behavior: open the SASE Admin Center on the Logs tab. Use `,n`
(`jump_to_notification`) as the reference implementation for every surface below.

### Changes

1. `src/sase/ace/tui/keymaps/mode_keymaps.py` — add `"jump_to_last_error": "L"` to
   `LeaderModeKeymaps.keys`.
2. `src/sase/default_config.yml` — add `jump_to_last_error: "L"` under
   `ace.keymaps.modes.leader_mode.keys`. This is mandatory:
   `tests/test_command_catalog_guards.py::test_leader_mode_dataclass_default_matches_default_config_yml`
   fails if the dataclass and the YAML drift.
3. `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py` — add a dispatch branch in
   `_dispatch_leader_key`, next to the other global (non-tab-scoped) actions such as
   `models_panel` / `update_sase`:

   ```python
   if key == leader_keys["jump_to_last_error"]:
       LeaderModeMixin._remember_leader_key(self, key, remember=remember)
       self.action_jump_to_last_error()  # type: ignore[attr-defined]
       self._refresh_current_tab()  # type: ignore[attr-defined]
       return True
   ```

   It is deliberately global: a launch can fail from any tab.

4. `src/sase/ace/tui/actions/base.py` — add the action next to `action_open_log_panel`:

   ```python
   def action_jump_to_last_error(self) -> None:
       """Open the Logs tab on this session's most recent registered error."""
       self._open_config_center("logs")
   ```

   The `registry` phase replaces the body; keep the docstring's promise aligned with
   what the phase actually does.

5. `src/sase/ace/tui/commands/_mode_commands.py` — add
   `"jump_to_last_error": "Jump to the last error in Logs"` to `_LEADER_LABELS`. Do
   **not** add a `_LEADER_TABS` entry: no tab restriction.
6. `src/sase/ace/tui/widgets/_keybinding_modes.py` — in `update_leader_bindings`, add
   `bindings.append((k("jump_to_last_error"), "last error"))` in the global section
   alongside `models_panel` and `update_sase`.
7. Help modal — add a row to the leader section of all three binding files, matching how
   `models_panel` / `update_sase` appear in each:
   - `src/sase/ace/tui/modals/help_modal/agents_bindings.py`
   - `src/sase/ace/tui/modals/help_modal/patches_bindings.py`
   - `src/sase/ace/tui/modals/help_modal/axe_bindings.py`

   Label: `"Jump to last error log"`.

8. `docs/ace.md` — add a `,L` row to each of the three Leader Mode tables (the Patches,
   Agents, and Axe tab sections; each begins at a "Leader Mode (comma prefix)" heading).
   Keep the existing column alignment. Description for this phase:
   `Open the Admin Center Logs tab`.

No config-schema change is required: `ace.keymaps.modes.*.keys` in
`src/sase/config/sase.schema.json` accepts arbitrary string-valued keys.

### Tests

- `tests/test_keymaps_defaults.py` — assert
  `reg.leader_mode.keys["jump_to_last_error"] == "L"`, and keep the existing
  `log_panel`-is-retired assertions untouched.
- `tests/ace/tui/test_log_panel_keymap.py` — add a case that dispatching the `L` leader
  key pushes a `ConfigCenterModal` whose `_initial_tab` is `"logs"`. Reuse the file's
  existing `_ActionApp` harness.
- `tests/ace/tui/test_leader_keymap_dispatch.py` — cover the new dispatch branch.
- `tests/ace/tui/test_leader_keybinding_footer.py` — assert the footer shows the
  `last error` binding.
- `tests/test_command_catalog_guards.py` passes as-is once step 2 is done; do not weaken
  it.

### Done when

`,L` from any tab opens the Admin Center on the Logs tab, the command palette lists
"Jump to the last error in Logs", the leader footer and Help modal show it, and
`just check` is green.

---

## Registered errors and error-anchored launch logs

Give every launch-failure entry a stable id, record the most recent one as this
session's registered error, and rewrite the toasts so the hint and the registration are
produced by one call.

### 1. Error ids in the launch log

`src/sase/logs/launch_log.py`:

- `log_launch_failure(...)` gains a keyword-only `error_id: str | None = None` and
  returns the `str` id it used (minting one when the caller passed none). It must keep
  its never-raises contract: if the write fails, still return the id.
- The JSONL record gains an `"error_id"` field.
- The human block header line gains the anchor as a trailing bracketed token:

  ```
  [2026-06-17 14:30:00 UTC] single launch failure: alpha  [err_260617_143000_7f3a9c]
  ```

  Append it to the existing header line rather than adding a new line, so the anchor and
  the entry's most identifying text land on the same row (the `focus` phase highlights
  and scrolls to exactly this line). This does not disturb
  `logs_pane_render._TIMESTAMP_RE`, which only matches a leading timestamp.

### 2. The registered-error pointer

New module `src/sase/logs/error_registry.py`:

```python
@dataclass(frozen=True)
class RegisteredError:
    error_id: str      # "err_<%y%m%d_%H%M%S>_<6 hex>"
    source_id: str     # a LogSource.id, e.g. "launch_failures"
    anchor: str        # exact substring present in the entry, "[<error_id>]"
    summary: str       # the toast prefix, for display
    registered_at: str # local timestamp, for display


def new_error_id() -> str: ...
def error_anchor(error_id: str) -> str: ...          # f"[{error_id}]"
def register_error(*, error_id: str, source_id: str, summary: str) -> RegisteredError: ...
def last_registered_error() -> RegisteredError | None: ...
def clear_registered_errors() -> None: ...           # test seam
```

- Module-level `_last: RegisteredError | None` guarded by a `threading.Lock`:
  registration happens both on the Textual event loop and from `asyncio.to_thread`
  workers.
- `register_error` is pure in-memory and must stay cheap enough to call from the event
  loop — no disk I/O.
- `error_anchor` is the single definition shared by the writer (`launch_log`) and the
  reader (the pane, in the `focus` phase). Nothing else may hand-build the anchor.
- Mint ids with `uuid4().hex[:6]` for the random suffix; the timestamp prefix keeps them
  sortable and greppable.
- Export the public names through `src/sase/logs/__init__.py` alongside the existing
  `log_launch_failure` re-export.

### 3. One helper registers and toasts

Rewrite `src/sase/ace/tui/actions/failure_messages.py`. Delete `LOG_PANEL_HINT` and
`with_log_panel_hint` outright (leaving them would let a caller print the hint without a
target, which is exactly the bug this epic closes). Replace with:

```python
def notify_registered_error(
    app: Any,
    prefix: str,
    *,
    error_id: str,
    source_id: str = "launch_failures",
    severity: str = "error",
) -> RegisteredError:
    """Register this session's latest error and toast the chord that reaches it."""
```

It calls `register_error(...)`, resolves the chord with
`leader_key_display(registry, "jump_to_last_error")` from
`getattr(app, "_keymap_registry", None)` (falling back to a literal `",L"` if the
registry is missing, so a toast can never raise), and calls
`app.notify(f"{prefix} - press {chord} for the log entry", severity=severity)`.

Because registration and the hint are the same call, a toast pointing at `,L` and a
registered target cannot get out of sync.

### 4. Rewire the seven call sites

`src/sase/ace/tui/actions/agent_workflow/_launch_procs.py`:

- In the payloadless-failure branch of the completion handler, mint
  `error_id = new_error_id()` on the UI thread, pass it into
  `_schedule_payloadless_launch_failure_log(...)` (and on into
  `_log_payloadless_launch_failure` → `log_launch_failure(error_id=...)`), and replace
  the `with_log_panel_hint("Launch failed")` notify with
  `notify_registered_error(self, "Launch failed", error_id=error_id)`.
- Minting up front matters: the log write is scheduled off-thread while the toast fires
  immediately, so the id must be created before either.

`src/sase/ace/tui/actions/axe_chop_run.py`:

- Chop launch exception path (~line 177): mint the id, pass it through the
  `asyncio.to_thread(log_launch_failure, ..., error_id=error_id)` call, then
  `notify_registered_error(self, f"Failed to launch chop '{chop_name}': {e}", error_id=error_id)`.
- `_log_chop_failure_outcome(outcome)`: return the `error_id` it used.
- `_notify_chop_outcome(self, outcome)`: gains a keyword-only
  `error_id: str | None = None`. The five failure statuses (`failure`, `timeout`,
  `missing_script`, `check_error`, `action_failed`) route through
  `notify_registered_error` when `error_id` is not None, and fall back to a plain
  `self.notify(...)` without any chord hint when it is None (defensive only — the caller
  always supplies one for those statuses). Non-failure statuses are unchanged.
  `check_error` keeps `severity="warning"`.

### 5. `,L` targets the registered error's source

- `src/sase/ace/tui/actions/base.py`:

  ```python
  def action_jump_to_last_error(self) -> None:
      """Open the Logs tab on this session's most recent registered error."""
      error = last_registered_error()
      if error is None:
          self.notify("No error registered in this ACE session")
      self._open_config_center("logs", log_error_target=error)
  ```

  Opening the Logs tab even with no target is deliberate — it is still the useful
  destination.

- `_open_config_center` gains `log_error_target: RegisteredError | None = None` and
  forwards it to `ConfigCenterModal`.
- `src/sase/ace/tui/modals/config_center_modal.py` — `ConfigCenterModal.__init__` gains
  the same keyword-only argument, stored as `self._log_error_target`.
- `src/sase/ace/tui/modals/config_center_catalog.py` — `_logs_pane_factory` passes
  `error_target=modal._log_error_target` to `LogsPane`.
- `src/sase/ace/tui/modals/logs_pane.py` — `LogsPane.__init__` gains
  `error_target: RegisteredError | None = None`, stores it as `self._error_target`, and
  when it is set and its `source_id` matches a known source, overrides the
  bookmark-derived `self._selected_index` with that source's index. The rest of the
  existing load path then selects the right source unchanged (the worker's
  `restore_selection_by_identity` receives that index's source id as `prior_identity`).

Plumb the whole `RegisteredError`, not just the source id — the `focus` phase needs the
anchor and adds no further plumbing.

### Tests

- `tests/logs/test_launch_log.py` — `log_launch_failure` returns an id, the id appears
  in the JSONL record, the id appears bracketed on the human header line, and an
  explicitly passed `error_id` is used verbatim.
- New `tests/logs/test_error_registry.py` — id shape, `error_anchor` round-trip,
  last-write-wins, `clear_registered_errors`, and thread-safety under concurrent
  `register_error` calls.
- New `tests/ace/tui/test_registered_error_toasts.py` — for each of the two failure
  modules: the toast text contains the configured chord, and `last_registered_error()`
  is non-None with a matching `summary` afterwards. Include a case with a custom leader
  keymap (e.g. `jump_to_last_error: "E"`) proving the toast names the _configured_
  chord, not a hardcoded `,L`.
- A guard test asserting the old hint literal `see Logs in SASE Admin Center` no longer
  appears anywhere under `src/`, and that the string `for the log entry` appears in
  exactly one `src/` file (`failure_messages.py`) — that is the structural invariant.
- `tests/ace/tui/test_log_panel_keymap.py` — `,L` with a registered error passes it as
  `log_error_target`; `,L` with none still opens the Logs tab and notifies.
- `tests/ace/tui/_logs_pane_helpers.py` — extend the `LAUNCH_LOG_BODY` fixture's header
  line with a `[err_...]` anchor so it matches the real format.
- Every test that registers an error must `clear_registered_errors()` in teardown; add
  an autouse fixture rather than relying on per-test cleanup.

### Done when

A failed launch toasts `Launch failed - press ,L for the log entry`, `,L` opens the Logs
tab with the Launch & Fan-out Failures source selected, the entry in
`launch_failures.log` carries its id, and `just check` is green.

---

## Logs pane focuses the registered error entry

Land the actual jump: find the anchor, render a window containing it, highlight the
entry line, and scroll to it.

### 1. Focused rendering

`src/sase/ace/tui/modals/logs_pane_render.py`:

- Extract the existing header construction in `render_log_detail` (title, description,
  path + mtime + counts, `─` rule) into a private `_detail_header(...) -> Text` used by
  both renderers. Keep `render_log_detail`'s current signature and return type — other
  callers and tests depend on it.
- Add `_MAX_FOCUS_SCAN_LINES = 5000` beside the existing `_MAX_TAIL_LINES = 500`.
- Add:

  ```python
  @dataclass(frozen=True)
  class FocusedLogDetail:
      text: Text
      focus_line: int | None   # 0-based line index within `text`
      found: bool


  def render_focused_log_detail(
      source: LogSource,
      anchor: str,
      max_lines: int = _MAX_TAIL_LINES,
      scan_lines: int = _MAX_FOCUS_SCAN_LINES,
  ) -> FocusedLogDetail: ...
  ```

  Algorithm:
  1. `lines = source.read_tail(scan_lines).splitlines()`. The read is `O(tail)` via
     `read_tail_seek`, so the enlarged scan stays cheap.
  2. `hit` = the **last** index whose line contains `anchor`. If there is none, return
     `FocusedLogDetail(text=<render_log_detail(source, max_lines) plus a dim italic notice that the entry is no longer in the last {scan_lines} lines — the log may have rotated>, focus_line=None, found=False)`.
  3. `start = hit - 1` when the preceding line is a separator rule (non-empty and its
     stripped characters are a subset of `{"="}`), else `start = hit`. Then clamp with
     `start = max(0, min(start, len(lines) - max_lines))` so a recent entry yields
     exactly the normal tail window and an older one yields a window opening at its own
     block.
  4. Body window is `lines[start : start + max_lines]`, rendered line by line with the
     existing `styled_log_line`, except the anchor line, which is rendered as an
     inverse-gold bar (`style=f"bold #1F1B00 on {GOLD}"`) so it is unmistakable against
     the red/cyan palette already in use.
  5. Header notes the focus (e.g. a trailing `· focused on <error_id>` in the same dim
     style as the existing `· N lines` suffix).
  6. `focus_line = len(header.plain.splitlines()) + (hit - start)`. Compute the header
     height, never hardcode it.

### 2. Pane wiring

`src/sase/ace/tui/modals/logs_pane_models.py` — `LogPaneLoadResult` gains
`focus_line: int | None = None` and `focus_found: bool | None = None` (defaults keep
existing constructions valid).

`src/sase/ace/tui/modals/logs_pane.py`:

- `_build_log_pane_load_result(...)` gains an `error_target: RegisteredError | None`
  parameter. When it is set and the selected source's id matches
  `error_target.source_id`, build the detail with
  `render_focused_log_detail(source, error_target.anchor)` and carry `focus_line` /
  `focus_found` into the result; otherwise behave exactly as today. **All of this runs
  in the existing thread worker.**
- `_start_load` passes `self._error_target` into the worker task.
- `_apply_load_result`: when `result.focus_line is not None`, schedule the scroll with
  `self.call_after_refresh(...)` so the new content height is laid out before
  `_force_scroll_detail_to(result.focus_line)` clamps against `max_scroll_y`. Keep the
  callback thin and synchronous (perf rule 2). When it is `None`, keep the current
  `reset_scroll` behavior.
- Clear `self._error_target = None` once a focused load has been applied, so pressing
  `r` returns the pane to its ordinary tail view and a later `,L` re-focuses.

### 3. Hints

`src/sase/ace/tui/modals/logs_pane.py::_hints` — no new key is added, so the hint string
only needs updating if the phase adds one. Prefer surfacing focus state through the
detail header (step 1.5) rather than the hint line.

### Tests

- `tests/ace/tui/test_logs_pane_render.py` — anchor found in the recent tail
  (`focus_line` points at the anchor line and the window equals the plain tail); anchor
  older than `_MAX_TAIL_LINES` but inside the scan window (window opens at the entry's
  separator); anchor absent (`found is False`, notice text present,
  `focus_line is None`); duplicate anchors resolve to the last occurrence.
- `tests/ace/tui/test_logs_pane.py` — constructing `LogsPane(error_target=...)` selects
  the target source and scrolls the detail so the focused line is visible (assert
  `scroll_y > 0` for a target placed below the fold, and that a subsequent
  `action_refresh` returns to an unfocused render).
- An end-to-end test: register an error, dispatch the `L` leader key, and assert the
  pushed `ConfigCenterModal` carries the target through to a `LogsPane` that focuses it.
- If the ACE PNG snapshot suite covers the Logs tab, add or update a golden for the
  highlighted entry (`just test-visual`, `--sase-update-visual-snapshots` to accept);
  otherwise skip — do not add a snapshot test just for this.

### Docs

Update the `,L` row in all three `docs/ace.md` Leader Mode tables to the final behavior:
`Jump to the log entry for the most recent error toast`. Add a short paragraph to the
Admin Center Logs section of `docs/configuration.md` (near the existing Admin Center tab
list) describing the registered-error jump and its session scope.

### Done when

A failed launch toast, followed by `,L`, lands the user on the Logs tab with the Launch
& Fan-out Failures source selected and the detail pane scrolled to that failure's
highlighted header line — and an entry that has aged out of the log degrades to the
ordinary tail plus an explanatory notice instead of a silent no-op. `just check` is
green, and `just check-full` has been run through `/sase_monitor` before the epic lands.

## Out of scope

- Surfacing an unread-error indicator in the footer or tab strip (a natural follow-up:
  make the leader footer show `,L last error` only when one is registered, via a
  `CommandContext` field and an `availability.py` predicate).
- Registering errors from anything other than the launch/chop failure paths. The
  mechanism is general — `source_id` is any `LogSource.id` — but this epic wires only
  the seven existing hint call sites.
- Persisting the pointer across ACE restarts.
