---
tier: epic
title: Durable agent-CLI update history in the Admin Center Agent CLIs sub-tab
goal: "Every sase-managed agent-CLI update run is recorded to a durable, bounded journal, and the Agent CLIs sub-tab
  renders that history beneath the selected CLI's detail panel — per-CLI by default, with a toggle to a run-grouped
  timeline across all CLIs — without any disk I/O on the keystroke path.

  "
phases:
  - id: journal
    title: Durable agent-CLI update run journal
    depends_on: []
    size: medium
    description:
      "journal: add the bounded JSONL run journal under ~/.sase/logs, define the run/entry records and the UpdateTrigger
      enum, record every run from the single execute_agent_cli_updates choke point with a best-effort writer that can
      never fail an update, and thread the trigger through the three call sites."
  - id: plumbing
    title: Pane load path, config, and session state
    depends_on:
      - journal
    size: small
    description:
      "plumbing: read a bounded tail of the journal inside the existing off-thread Updates load worker, carry it on
      PluginsLoadResult into pane state, add the two ace.updates config keys and the session-scoped history-scope flag,
      and mount the history Static with its TCSS so the render phase has a surface to paint into."
  - id: render
    title: History panel rendering and scope toggle
    depends_on:
      - plumbing
    size: medium
    description:
      "render: build the per-CLI and all-CLIs history renderables with their glyph/color palette, relative timestamps
      derived from the load clock, trigger badges, truncation footer, and empty/error states, and wire the H scope
      toggle with its check_action gating and repaint path."
  - id: polish
    title: Help, docs, and visual goldens
    depends_on:
      - render
    size: small
    description:
      "polish: document the panel and journal in the help modal, the ACE guide, the agent-providers guide, and the
      configuration reference, record the three new PNG goldens, and land a green just check."
proposed_by: bbugyi200.athena.sk
create_time: 2026-08-03 06:52:56
status: wip
---

- **PROMPT:**
  [prompts/202608/agent_cli_update_history.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_cli_update_history.md)

# Plan: Agent-CLI update history

## Why

The Agent CLIs sub-tab shows only _present_ state: installed version, latest version, install method, update command.
The only trace of a past update is `AgentCliBrowserMixin._agent_cli_results` — an in-memory dict keyed by provider name
that holds **one** outcome per CLI and is discarded when ACE exits. That is the worst possible retention for this data,
because the dominant trigger is `,U`, and a `,U` that changes SASE code **re-execs ACE**. The most common agent-CLI
update in practice is therefore the one whose record is guaranteed to be destroyed.

So today there is no way to answer: _when did Claude Code last actually update?_ _Did last night's `,U` update Codex or
silently skip it?_ _That npm failure — how long has it been failing?_

This epic adds a durable journal of every sase-managed agent-CLI update run and renders it beneath the selected CLI's
details.

## Scope

**In scope.** Recording runs from all three sase-managed trigger paths; the history panel in the Agent CLIs sub-tab; the
scope toggle; config, docs, help, and visual goldens.

**Out of scope.** A `sase agent-cli history` CLI subcommand. The journal module is deliberately shaped so one can be
added later (a pure reader returning typed records), but this epic ships no new CLI surface. Also out of scope: history
for plugin or SASE-core updates — those already have `~/.sase/logs/dev_update.jsonl` and the uv-tool receipt.

## Architecture decisions

### Where the code lives (Rust core boundary)

`sase/memory/rust_core_backend_boundary.md` asks that shared backend behavior move to `../sase-core`. This journal stays
in **Python**, in `src/sase/agent_clis/`, for three concrete reasons:

1. `sase-core` has no agent-CLI domain at all. `crates/sase_core/src/` contains no provider detection, latest-version
   probing, update planning, or execution — the entire producing domain
   (`sase/agent_clis/{detect,latest,operations}.py`) is Python. A Rust journal would be a sink with no Rust producer,
   requiring a new wire type mirroring `AgentCliUpdateResult` purely to cross the boundary and cross straight back.
2. The directly analogous artifact — `src/sase/dev_update/journal.py`, the durable JSONL journal of _SASE's own_ update
   runs — is Python and is written from this same comprehensive-update code path
   (`plugins_browser_comprehensive_update_execution.py` calls `append_dev_update_journal`). Splitting the two halves of
   one `,U` run across two languages would be strictly worse than either choice alone.
3. Every other durable SASE JSONL log (`repo_open_log.py`, `skills/use_log.py`, `memory/read_log.py`, everything under
   `sase/logs/`) is Python on the shared `sase.logs._bounded` primitives, which already provide the flock-guarded,
   size-bounded append this feature needs.

If the agent-CLI domain is later migrated to `sase-core`, the journal should migrate with it as one unit. Do not split
it out early.

### Recording happens at the choke point, not at call sites

There are exactly three sase-managed trigger paths, and all three converge on
`sase.agent_clis.operations.execute_agent_cli_updates`:

| Trigger                   | Path                                                                   |
| ------------------------- | ---------------------------------------------------------------------- |
| `,U` comprehensive update | `plugins_browser_comprehensive_update_execution._execute_provider_leg` |
| `A` in the Admin Center   | `plugins_browser_agent_clis._submit_agent_cli_update_task`             |
| `sase agent-cli update`   | `agent_clis.cli_update.handle_agent_cli_update_command`                |

Recording lives **inside** `execute_agent_cli_updates`. A fourth trigger added later is journaled automatically; a
call-site-by-call-site approach would silently miss it. Call sites contribute only the `trigger` label.

### No disk I/O on the keystroke path

`sase/memory/tui_perf.md` rule 1 (never block the event loop), rule 7 (debounce detail panels), and rule 8 (render paths
never stat) all bear on this panel, because it repaints on every `j`/`k`.

The design satisfies all three without new machinery: the journal is read **once per pane load**, inside the existing
threaded `load_plugins_catalog_for_pane` worker, and carried on `PluginsLoadResult` exactly like `agent_cli_statuses`
already is. Every subsequent repaint renders from an in-memory tuple. Selection changes continue to flow through
`DetailPanelDebouncer`. There is no mtime cache to invalidate and no new refresh path.

Freshness comes for free: `_on_agent_cli_update_complete` already calls `_start_load(force=False)`, so an `A` update's
run appears after that reload; a `,U` that restarts ACE re-reads the journal on the next pane load. Do **not** add an
optimistic in-memory insert — it would need run-id dedup against the reload for no user-visible benefit.

### One record per run, not per CLI

A single `,U` or `A` updates several CLIs. That batch is the honest unit of history, so the journal appends **one JSONL
line per run**, holding one entry per CLI. This keeps appends atomic under the existing flock (a run can never be half
recorded, and line-based rotation can never split one), keeps the file small, and preserves the run grouping the
all-CLIs view needs. The per-CLI view flattens runs at read time.

### Only runs that did something are recorded

`,U` runs on a cadence and is usually a no-op. Journaling every no-op would bury the real events.

**Rule:** a run is written **iff at least one entry has status `updated` or `failed`** — i.e. a command actually ran and
reached a terminal outcome. Within a written run, `already_current` and `skipped` entries are recorded too, because they
are the context for what the run chose not to do.

This rule is a property of the journal writer, is unit-tested directly, and must not be re-implemented as a UI filter.

## Data model

`src/sase/agent_clis/history.py`, schema version 1:

```jsonc
{
  "schema_version": 1,
  "run_id": "9f2c1ab40e77", // uuid4().hex[:12]
  "timestamp": "2026-08-03T06:41:14-04:00", // local, tz-aware, seconds precision
  "epoch": 1785753674.0, // machine clock for the UI; never re-derived from `timestamp`
  "trigger": "comprehensive", // comprehensive | admin_center | cli | unknown
  "all_clis": true,
  "elapsed_seconds": 12.41, // measured wall time of the whole batch
  "counts": { "updated": 1, "already_current": 2, "failed": 0, "skipped": 1 },
  "entries": [
    {
      "name": "claude",
      "display_name": "Claude Code",
      "status": "updated",
      "old_version": "2.1.220",
      "new_version": "2.1.221",
      "command": ["npm", "install", "-g", "@anthropic-ai/claude-code@latest"],
      "reason": null,
      "elapsed_seconds": 9.02,
      "output_tail": null,
    },
  ],
}
```

Both `timestamp` and `epoch` are stored. The ISO string is for humans reading the file; `epoch` is what the UI uses, so
a malformed or exotic timestamp can never break relative-time rendering.

`output_tail` is retained **only for `failed` entries**, truncated to 2 000 characters. Successful npm output is noise;
failure output is the one thing worth keeping. (`dev_update.jsonl` already retains 12 000-character tails, so this is
the conservative side of existing precedent.) `reason` is truncated to 500 characters.

## Trigger vocabulary

`UpdateTrigger` is a `StrEnum` in `src/sase/agent_clis/models.py` (it appears in a public function signature, so it
belongs with the other domain enums; the run records themselves live in `history.py`).

| Value           | Set by                                     | Rendered as |
| --------------- | ------------------------------------------ | ----------- |
| `comprehensive` | `_execute_provider_leg`                    | `,U`        |
| `admin_center`  | `_submit_agent_cli_update_task`            | `A`         |
| `cli`           | `handle_agent_cli_update_command`          | `CLI`       |
| `unknown`       | default; unrecognized value read from disk | `—`         |

Rendering the _keymap that caused it_ is the point: the user asked for the history of updates "triggered via the `,U`
keymap", and the badge answers that directly.

---

# Phase `journal`: Durable agent-CLI update run journal

## Deliverables

**`src/sase/agent_clis/models.py`** — add `UpdateTrigger(StrEnum)` with the four values above; export it from `__all__`
and from `src/sase/agent_clis/__init__.py`.

**`src/sase/agent_clis/history.py`** (new):

- `AGENT_CLI_UPDATE_JOURNAL: str | None = None` — test override, mirroring `dev_update.journal.DEV_UPDATE_JOURNAL`.
- `ENV_MAX_BYTES = "SASE_AGENT_CLI_UPDATE_JOURNAL_MAX_BYTES"`.
- `HISTORY_SCHEMA_VERSION = 1`, `MAX_REASON_CHARS = 500`, `MAX_OUTPUT_TAIL_CHARS = 2_000`.
- `agent_cli_update_journal_path() -> Path` →
  `AGENT_CLI_UPDATE_JOURNAL or sase_subdir("logs") / "agent_cli_updates.jsonl"`.
- Frozen dataclasses `AgentCliUpdateRunEntry` and `AgentCliUpdateRun` matching the JSON above. Give `AgentCliUpdateRun`
  a `executed_entries` property returning entries whose `command` is not `None`, and an `executed_count` property; the
  render phase consumes both rather than re-deriving them.
- `should_record_run(results) -> bool` — the "did something" rule, exported and directly unit-tested.
- `build_agent_cli_update_run(results, *, trigger, elapsed, now=None, run_id=None) -> AgentCliUpdateRun` — pure, fully
  injectable clock and id so tests are deterministic.
- `record_agent_cli_update_run(results, *, trigger, elapsed, path=None, now=None, run_id=None) -> AgentCliUpdateRun | None`
  — returns `None` when `should_record_run` is false **or** when the append fails. Wrapped in a bare
  `except Exception: log.debug(..., exc_info=True); return None`. Appends via
  `sase.logs._bounded.append_jsonl_record(..., max_bytes=max_bytes_from_env(ENV_MAX_BYTES, DEFAULT_MAX_BYTES), sort_keys=True)`.
- `read_agent_cli_update_runs(*, limit=200, path=None) -> tuple[AgentCliUpdateRun, ...]` — reads the file, decodes the
  **last `limit`** well-formed records, returns them **newest first**. Skips blank lines, undecodable JSON, non-mapping
  payloads, records whose `schema_version` is not 1, and records missing required fields — one bad line never discards
  the file. A missing file returns `()`. An unreadable file raises `OSError`, which the caller in `plumbing` converts to
  a pane-level error string. Unknown `trigger` values decode to `UpdateTrigger.UNKNOWN` rather than failing.

**`src/sase/agent_clis/operations.py`** — `execute_agent_cli_updates` gains two keyword-only parameters:

```python
def execute_agent_cli_updates(
    plan: AgentCliUpdatesReady | AgentCliNothingToUpdate,
    *,
    run_fn: RunnerFn = run_command,
    trigger: UpdateTrigger = UpdateTrigger.UNKNOWN,
    record_fn: RecordFn | None = record_agent_cli_update_run,
) -> tuple[AgentCliUpdateResult, ...]:
```

Wrap the existing loop with a `time.monotonic()` stopwatch, and after it call
`record_fn(results, trigger=trigger, elapsed=elapsed)` when `record_fn` is not `None`. `record_fn=None` disables
journaling for tests that must not touch state. The function's return value is unchanged.

Import direction is `operations → history → models`; `history.py` must not import `operations.py`.

**Call sites** — pass `trigger=`:

- `plugins_browser_agent_clis.py:568` → `UpdateTrigger.ADMIN_CENTER`
- `plugins_browser_comprehensive_update_execution.py:83` → `UpdateTrigger.COMPREHENSIVE`
- `cli_update.handle_agent_cli_update_command` → `UpdateTrigger.CLI`

Both TUI sites call through the `pane_module._execute_agent_cli_updates` alias, which tests monkeypatch. The one
existing stub (`tests/ace/tui/test_plugins_browser_pane_agent_clis.py:326`) is `lambda *_args, **_kwargs:`, so it
absorbs the new keyword; no `callable_accepts_keyword` guard is needed. Confirm no other stub is added.

## Test isolation

`tests/conftest.py::redirect_sase_home` exists but is not autouse. Any test that reaches the real
`record_agent_cli_update_run` would otherwise write to the user's `~/.sase/logs/`. Two guards:

1. New journal tests pass an explicit `path=tmp_path / "agent_cli_updates.jsonl"`.
2. Existing `execute_agent_cli_updates` tests in `tests/agent_clis/` pass `record_fn=None`, **or** an autouse fixture in
   `tests/agent_clis/conftest.py` monkeypatches `history.AGENT_CLI_UPDATE_JOURNAL` to a `tmp_path` file. Prefer the
   fixture — it cannot be forgotten by a future test.

## Tests

`tests/agent_clis/test_history.py`:

- `should_record_run` is true for a run containing `updated`, true for one containing `failed`, false for all-current,
  false for all-skipped, false for empty results.
- A recorded run round-trips through `record` → `read` with every field preserved, entries in plan order.
- `read_agent_cli_update_runs` returns newest first and honors `limit`.
- A file containing a blank line, `not json`, `[]`, `{"schema_version": 99}`, and one valid record yields exactly the
  valid record.
- An unknown `trigger` string decodes to `UpdateTrigger.UNKNOWN`.
- `reason` longer than 500 chars is truncated; `output_tail` is kept only for `failed` entries and truncated to 2 000.
- An append failure (monkeypatch `append_jsonl_record` to raise) returns `None` and does not propagate.
- Rotation: with `SASE_AGENT_CLI_UPDATE_JOURNAL_MAX_BYTES` set very low, appends keep the file bounded and `read` still
  returns whole records.

`tests/agent_clis/test_operations.py` (extend): `execute_agent_cli_updates` calls `record_fn` exactly once with the
results, the given trigger, and a non-negative elapsed; and `record_fn=None` suppresses it entirely.

---

# Phase `plumbing`: Pane load path, config, and session state

## Load path

**`src/sase/ace/tui/modals/plugins_browser_loading.py`** — extend `PluginsLoadResult`:

```python
agent_cli_history: tuple[AgentCliUpdateRun, ...] = ()
agent_cli_history_error: str | None = None
```

In `load_plugins_catalog_for_pane`, read the journal in its own `try/except` so a history failure degrades that panel
alone and never affects the catalog, statuses, or uv-tool probe — the same independent-degradation shape the existing
`core_versions` and `agent_cli_statuses` legs use. Read `limit=200`.

**`src/sase/ace/tui/modals/plugins_browser_pane.py`**:

- `__init__`: `self._agent_cli_history: tuple[AgentCliUpdateRun, ...] = ()`,
  `self._agent_cli_history_error: str | None = None`.
- Module-level alias `_read_agent_cli_update_runs = read_agent_cli_update_runs`, matching the existing
  `_plan_agent_cli_updates` / `_execute_agent_cli_updates` aliasing convention so tests and visual fixtures can stub it.
- `on_worker_state_changed` SUCCESS branch: adopt both fields with `getattr(result, ..., default)`, exactly like
  `agent_cli_statuses` and `agent_cli_error`. Stubbed loaders that omit them must keep working.
- `compose`: inside `#agent-clis-detail-scroll`, after `#agent-clis-detail`, yield
  `Static("", id="agent-clis-history", markup=False)`.

**`src/sase/ace/tui/styles.tcss`**, beside the existing `#agent-clis-detail` rule:

```tcss
PluginsBrowserPane #agent-clis-history {
    width: 100%;
    height: auto;
    color: $text;
    margin-top: 1;
}
```

The surrounding `VerticalScroll` already has `height: 1fr` and the pane already binds `ctrl+d`/`ctrl+u`/`g`/`G` to
scroll it, so a long history scrolls with no new bindings.

## Config

**`src/sase/default_config.yml`**, in the existing `ace.updates` block:

```yaml
agent_cli_history: true
agent_cli_history_max_rows: 8
```

**`src/sase/config/sase.schema.json`** — add both under `ace.updates.properties` beside `post_update_toast_max_commits`
(`boolean`, and `integer` with `minimum: 0`), with descriptions.

Load them in the pane the way `_load_incoming_commits_config` already loads `ace.updates.incoming_commits`: a small
frozen config dataclass read once in `__init__`, coercing bad values to the defaults rather than raising.
`agent_cli_history: false` hides the panel entirely (render nothing, skip the journal read in the load worker).
`agent_cli_history_max_rows: 0` means unlimited.

## Session state

**`src/sase/ace/tui/modals/config_center_session.py`** — add to `UpdatesSessionState`:

```python
agent_cli_history_all: bool = False
```

Session-scoped, matching `TasksSessionState.all_sessions`. The scope choice survives closing and reopening the Admin
Center within one ACE process and resets on restart.

## Tests

`tests/ace/tui/test_plugins_browser_pane_agent_clis.py` (extend):

- A load result carrying runs populates `pane._agent_cli_history`; one omitting the field leaves it `()`.
- A journal read that raises inside the loader sets `agent_cli_history_error` and leaves `agent_cli_statuses` intact.
- With `agent_cli_history: false`, the loader does not call `_read_agent_cli_update_runs`.
- Config coercion: non-integer / negative `agent_cli_history_max_rows` falls back to the default.

---

# Phase `render`: History panel rendering and scope toggle

## New module

`src/sase/ace/tui/modals/plugins_browser_agent_clis_history.py` — pure renderable builders, no widget access, so they
are unit-testable against a captured `Console` the way other Rich builders in this package are. `AgentCliBrowserMixin`
imports them; keep `plugins_browser_agent_clis.py` from growing a second responsibility.

## The clock

Relative times are computed against `self._now` — the load-result timestamp the pane already stores (visual fixtures pin
it to `_NOW = 1_700_000_000.0`). Never call `time.time()` in a render path. This keeps PNG goldens deterministic and
satisfies tui_perf rule 8.

Format, reusing the shape of `_relative_time` in `tasks_pane_render.py`: `just now`, `42s ago`, `17m ago`, `3h ago`,
`2d ago`; at or beyond 7 days switch to an absolute `Aug 01 14:02` derived from `epoch` in the local timezone. Runs with
an `epoch` in the future (clock skew) render as `just now`, never a negative age.

## Palette

Deliberately subordinate to the detail panel above it: the detail panel owns the loud provider-colored border; the
history panel borrows the provider color only for its title.

| Element               | Style                                                 |
| --------------------- | ----------------------------------------------------- |
| Panel border          | `#5F5F87`                                             |
| Panel title           | `bold <provider accent>` (`_agent_cli_color(status)`) |
| Panel subtitle        | `dim`                                                 |
| `updated` glyph `▲`   | `bold #00D700`                                        |
| `failed` glyph `✖`    | `bold #FF5F5F`                                        |
| `already_current` `·` | `dim`                                                 |
| `skipped` glyph `○`   | `dim`                                                 |
| old version           | `dim`                                                 |
| arrow `→`             | `dim`                                                 |
| new version           | `bold #00D700`                                        |
| relative time         | `dim`                                                 |
| trigger badge         | `bold #AF87FF` (the Updates sub-tab accent)           |
| elapsed               | `dim`                                                 |
| failure reason        | `#FF5F5F`                                             |

`#00D700`, `#FF5F5F`, `#AF87FF`, and `#5F5F87` are all already in this pane's palette.

## Per-CLI view (default)

Rows are the selected CLI's **executed** entries (`command is not None`, i.e. `updated` or `failed`) across all runs,
newest first — `already_current` and `skipped` entries are not rows here, because the detail panel directly above
already states the current version and the update command. Columns via `Table.grid(padding=(0, 2))`:

```
╭─ Update history ───────────────────────────────────────────────────────╮
│ ▲  2.1.220 → 2.1.221    2h ago    ,U     9.0s                          │
│ ▲  2.1.180 → 2.1.220    Aug 01    A     11.4s                          │
│ ✖  2.1.180              Jul 30    ,U     2.0s  npm ERR! EACCES: permis… │
╰─ 3 of 12 runs · H all CLIs ────────────────────────────────────────────╯
```

Failure reasons are truncated with `Text.truncate(..., overflow="ellipsis")` so a row never wraps.

## All-CLIs view (`H`)

Grouped by run, newest first — the run is the unit, so the grouping must be visible:

```
╭─ Update history · all agent CLIs ──────────────────────────────────────╮
│ 2h ago · ,U · 9.0s                                                     │
│   ▲ Claude Code   2.1.220 → 2.1.221                                    │
│   ▲ Codex CLI     0.146.0 → 0.147.0                                    │
│   · 2 already current · ○ 1 skipped                                    │
│                                                                        │
│ Aug 01 14:02 · A · 11.4s                                               │
│   ✖ OpenCode      1.18.11   npm ERR! EACCES: permission denied         │
╰─ 2 of 12 runs · H this CLI ────────────────────────────────────────────╯
```

Each entry line uses that provider's accent (`self._agent_cli_colors`) for its display name, so the two views share one
color language with the master list. Non-executed entries collapse into the single dim summary line, which is omitted
when there are none.

## Counting and truncation

`agent_cli_history_max_rows` caps **rows** in the per-CLI view and **runs** in the all-CLIs view. The subtitle always
states what was elided — `3 of 12 runs` — because silent truncation would misrepresent the history. With `max_rows: 0`,
the subtitle drops the `N of M` prefix.

## States

| State                               | Rendering                                                                                                       |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| No CLI selected                     | render nothing (the detail placeholder already covers it)                                                       |
| `agent_cli_history` config false    | render nothing                                                                                                  |
| Journal read failed                 | panel body: `Could not read update history:\n<error>` in `#FF5F5F`                                              |
| Journal empty                       | `No sase-managed agent CLI updates recorded yet.` + `Press A to update agent CLIs, or ,U to update everything.` |
| Selected CLI has no rows, others do | `No recorded updates for <display name>.` + `12 runs recorded for other CLIs — press H to see them.`            |
| All-CLIs view, journal empty        | same empty text as above                                                                                        |

The fifth row is the case that makes the panel feel intelligent rather than broken, and it is why the scope toggle is
discoverable without a hints-line entry.

## Scope toggle

Add to `PluginsBrowserPane.BINDINGS`: `("H", "toggle_history_scope", "History scope")`. `H` is currently unbound in this
pane. In `check_action`, return `False` for `toggle_history_scope` when `_active_subtab != "agent-clis"`.

`action_toggle_history_scope` flips `self._session_state.agent_cli_history_all`, then calls
`self._render_agent_cli_history(force=True)`. It repaints **only** the history Static — not the detail panel, and not
the option list.

The hint lives in the panel subtitle (`H all CLIs` / `H this CLI`) and in the help modal, **not** in
`_agent_cli_hints()`. That line is already at nine segments and clips at `height: 1`; adding a tenth would truncate an
existing hint at 120 columns.

## Wiring

- `_render_agent_cli_history(*, force: bool = False)` in `AgentCliBrowserMixin`, mirroring `_render_agent_cli_detail`:
  dedup on `(name, scope)` unless `force`, query `#agent-clis-history`, `update(...)`.
- Call it from `_render_agent_cli_detail` (so debounced highlight moves repaint both together), from
  `_render_agent_clis` with `force=True`, and from `action_toggle_history_scope`.
- Do **not** call it from `_on_agent_cli_highlighted` directly; the existing `DetailPanelDebouncer` path is the only one
  that should reach it on `j`/`k`.

## Tests

`tests/ace/tui/test_plugins_browser_pane_agent_clis_history.py` (new), rendering to a captured `Console` and asserting
on plain text:

- Per-CLI view lists only that CLI's executed entries, newest first, and excludes another CLI's entries.
- `already_current` and `skipped` entries produce no per-CLI rows.
- All-CLIs view groups by run, shows the trigger badge per run, and collapses non-executed entries into the summary line
  — which is absent when every entry executed.
- Trigger badges render `,U`, `A`, `CLI`, and `—` for the four enum values.
- Relative time boundaries: 30 s, 90 s, 2 h, 2 d, 8 d (absolute), and a future `epoch` (`just now`).
- Truncation: with `max_rows=2` and 5 eligible rows, exactly 2 render and the subtitle reads `2 of 5 runs`; `max_rows=0`
  renders all and drops the prefix.
- Each of the six states in the table above.
- `H` toggles `_session_state.agent_cli_history_all`, repaints the history Static, and leaves `_agent_cli_detail_name`
  untouched; `check_action("toggle_history_scope", ())` is `False` on the core and plugins sub-tabs.

---

# Phase `polish`: Help, docs, and visual goldens

## Help modal

`src/sase/ace/tui/modals/help_modal/binding_common.py`, `ADMIN_CENTER_UPDATES_SECTION` — add after the `Space` row:

```python
("H", "CLI history: this / all"),
```

`src/sase/ace/CLAUDE.md` caps help descriptions at 32 characters, which is why this reads `CLI history` rather than
`Agent CLI history`. Verify the 57-character box width still renders correctly.

## Docs

**`docs/ace.md`, "Updates Tab"** — extend the Agent CLIs sentence and add a paragraph: every sase-managed update run
(`,U`, `A`, or `sase agent-cli update`) is appended to `~/.sase/logs/agent_cli_updates.jsonl`; runs in which no command
reached a terminal outcome are not recorded; the sub-tab renders that history below the selected CLI's details, `H`
toggles between this CLI and a run-grouped view of all CLIs, and `ace.updates.agent_cli_history` /
`agent_cli_history_max_rows` control it.

**`docs/agent_providers.md`, "Inventory and updates"** — note that `sase agent-cli update` is journaled to the same
file, and name the `SASE_AGENT_CLI_UPDATE_JOURNAL_MAX_BYTES` bound.

**`docs/configuration.md`, "Updates tab"** — document both new keys with their defaults, matching the surrounding
`post_update_toast_*` entries.

## Visual goldens

`tests/ace/tui/visual/test_ace_png_snapshots_config_center_plugins.py` — three new `120x40` goldens, using
`_patch_plugins_catalog` extended (in `_ace_config_center_plugins_helpers.py`) to seed a fixed history built from
`_NOW`-relative epochs so the relative times are stable:

| Golden                                          | Scenario                                                        |
| ----------------------------------------------- | --------------------------------------------------------------- |
| `config_center_agent_clis_history_120x40`       | Claude Code selected, per-CLI view with an update and a failure |
| `config_center_agent_clis_history_all_120x40`   | after `H`, run-grouped all-CLIs view                            |
| `config_center_agent_clis_history_empty_120x40` | empty journal, empty-state copy                                 |

The fixture history must include at least one `failed` entry with a long reason (proving truncation), one run with
non-executed entries (proving the summary line), and two distinct triggers (proving both badges).

Record with `just test-visual --sase-update-visual-snapshots`, then re-run without the flag to confirm exact-pixel
equality. Inspect `.pytest_cache/sase-visual/` on any mismatch.

## Verification

`just install` first (ephemeral workspace), then `just lint`, `just test`, `just test-visual`, and finally `just check`.
Check `sase/memory/symvision.md` if the new public symbols in `history.py` and `plugins_browser_agent_clis_history.py`
trip the unused-symbol lint — they should all be reachable from `agent_clis/__init__.py` or `AgentCliBrowserMixin`.
