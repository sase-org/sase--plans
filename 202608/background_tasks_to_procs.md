---
tier: epic
title: Rename SASE Background Tasks to Procs
goal:
  SASE's durable background-execution feature is named **Proc** end to end — Rust wire
  types and bindings, the `sase.procs` Python package, the `sase proc` CLI (with `task`
  kept as a legacy alias), the ACE Procs tab and proc indicator, the
  `procs.history_limit` config key, the `~/.sase/procs/procs.jsonl` store, docs, memory,
  skills, and the project glossary — while task beads, asyncio/Textual worker tasks, and
  the Muse `task.lifecycle.*` provider protocol keep the word "task" untouched.
phases:
  - id: core
    title: Rename the Rust background-task core to procs
    depends_on: []
    size: medium
    description:
      "core: rename `crates/sase_core/src/tasks/` to `procs/` in ../sase-core, rename
      the wire structs and the `task_id` field to `Proc*`/`proc_id`, bump the wire
      schema to 2 while still deserializing legacy `task_id` records, and expose
      canonical `read_procs_snapshot`/`append_proc`/`update_proc`/`prune_procs` PyO3
      bindings alongside the existing legacy binding names."
  - id: store
    title: Move the Python package to sase.procs and migrate on-disk state and config
    depends_on:
      - core
    size: medium
    description:
      "store: `git mv src/sase/tasks src/sase/procs`, rename `BackgroundTask` to `Proc`
      and `task_id` to `proc_id` throughout, move the store to
      `~/.sase/procs/procs.jsonl` with a marker-guarded one-shot migration, rename the
      `tasks.history_limit` config key to `procs.history_limit` with the legacy key
      still honored, and update `tools/validate_sase_core_rs` plus the monitor
      cross-references."
  - id: cli
    title: Rename the sase task CLI command tree to sase proc
    depends_on:
      - store
    size: medium
    description:
      "cli: rename `sase task` to `sase proc` with `task` registered as a legacy alias
      and facade shim modules, rename the parser/handler/render modules and the `--json`
      envelope key, and update the CLI help text, tests, and `docs/cli.md`."
  - id: tui-runtime
    title: Rename the TUI tracked-task runtime to procs
    depends_on:
      - store
    size: medium
    description:
      "tui-runtime: rename `task_queue.py`, `task_mirror.py`, `task_subprocess.py`,
      `task_actions.py`, `widgets/task_indicator.py`, and the `_*_tasks.py` action
      mixins to their proc spellings, rename `TaskQueue`/`TaskMirror`/`TaskReporter`/
      `TrackedTask*`/`_submit_tracked_task` and every call site, without changing
      displayed text."
  - id: tui-pane
    title: Rename the ACE Tasks pane and Admin Center tab identifier to procs
    depends_on:
      - store
    size: medium
    description:
      "tui-pane: rename the `tasks_pane*` and `tasks_store_rows` modules and `TasksPane`
      to their proc spellings, move the Admin Center tab identifier from `tasks` to
      `procs` with persisted-state migration, and rename the `#tasks-*` DOM ids and
      their `styles.tcss` selectors, without changing displayed text."
  - id: labels
    title: Flip user-visible Task text to Proc and refresh snapshots
    depends_on:
      - cli
      - tui-runtime
      - tui-pane
    size: medium
    description:
      "labels: change every displayed string that names this feature — Admin Center tab
      label, pane title and hints, command palette entries, quit-confirm copy, status
      messages, CLI help — from Task to Proc, then regenerate the affected text and PNG
      snapshot goldens."
  - id: docs
    title: Rewrite documentation, memory, skills, and the glossary
    depends_on:
      - labels
    size: medium
    description:
      "docs: rewrite the background-task sections of docs/ace.md, cli.md,
      configuration.md, integrations.md, beads.md, sdd.md, notifications.md, plugins.md,
      monitors.md, and INSTALL.md, update `sase/memory/tui_perf.md` and the
      `sase_monitor` skill, add a `Proc` glossary entry to `sase/sase.yml`, and run
      `sase memory init`."
  - id: land
    title: Verify the migration and land the epic
    depends_on:
      - docs
    size: small
    description:
      "land: run the exhaustive verification lane over the combined tree, sweep for
      residual background-task spellings, confirm no emitter writes `task_id` or
      `~/.sase/tasks` again, and land the epic."
proposed_by: bbugyi200.athena.000
create_time: 2026-08-13 17:18:17
status: wip
---

- **PROMPT:**
  [prompts/202608/background_tasks_to_procs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/background_tasks_to_procs.md)

# Plan: Rename SASE Background Tasks to Procs

## Background

SASE's durable background-execution feature is currently called **Background Tasks**. It
consists of:

- a Rust-owned JSONL store and wire (`../sase-core/crates/sase_core/src/tasks/`),
- a Python facade package (`src/sase/tasks/`, 8 modules, ~1,390 lines),
- the `sase task` CLI command tree (`list`, `show`, `run`, `kill`),
- the ACE TUI's tracked-task runtime (`TaskQueue`, `TaskMirror`, `TaskReporter`, the
  `TaskActionsMixin`) and its Admin Center **Tasks** tab,
- the `tasks.history_limit` config key and the `~/.sase/tasks/tasks.jsonl` store.

The user wants this feature renamed to **Proc** (plural **Procs**). The rename is
worthwhile: "task" is the single most overloaded word in this codebase. It already means
three unrelated things, and only one of them is being renamed.

### The three meanings of "task" — only one is in scope

This is the most important thing for every phase worker to internalize. A worker that
renames the wrong "task" will break persisted bead data or the provider protocol.

**IN SCOPE — the durable background-execution feature.** Signals: `sase.tasks`,
`BackgroundTask`, `task_id`, `TASK_KINDS` (`command`/`tui`/`detached`), `tasks.jsonl`,
`sase task`, `TaskQueue`, `TaskMirror`, `TaskReporter`, `TasksPane`, `#task-indicator`,
`tasks.history_limit`, the prose phrases "background task" and "tracked task".

**OUT OF SCOPE — task beads.** A task bead is a work item in the beads system, a sibling
of epic and phase beads. Its size scale, triage gate, and launch workflow all use "task"
and all stay. Do not touch:

- `src/sase/bead/task_gate.py`, `_task_gate_actions.py`, `_task_gate_preview.py`,
  `_task_gate_response.py`, `_task_gate_spec.py`, `snooze_gate.py`
- `src/sase/bead/cli_work_task.py`, `src/sase/bead/task_launch.py` (this module
  _launches_ a task bead's worker; it happens to submit a background proc, so its **call
  sites** change but the module, its `_TASK_LAUNCH_SUBMIT_LOCK = "task-launch-submit"`
  lock name, and its bead-facing names do not)
- `src/sase/notification_gates/kind_validation/task_triage.py` and
  `task_triage_payload.py`; the `task_triage` gate kind and its
  `GateDebugModal.kind-task_triage` CSS selector
- `src/sase/scripts/sase_chop_bead_task_triage.py`, `_bead_task_triage_gates.py`,
  `_bead_task_triage_state.py`, and the `sase_chop_bead_task_triage` console script
- `src/sase/xprompts/skills/sase_new_task.md`, the `bd/work_task` xprompt, the
  `work_task_bead` tag, `sase bead work <task-id>`, `sase bead ready`, `sase bead +1`
- `sase/memory/sase_beads.md`, `sase/memory/sase.md`, `sase/memory/sase_sizes.md`, and
  the "File Discovered Work As Task Beads" section of `AGENTS.md` and its provider shims
- every `tests/**/*task_triage*`, `tests/test_bead/test_task_*`, and
  `tests/test_bead/test_work_task_rendering.py`

**OUT OF SCOPE — asyncio and Textual concurrency primitives.** `asyncio.Task`,
`asyncio.create_task`, `TaskGroup`, Textual `Worker`, and SASE's pump-free helpers in
`src/sase/ace/tui/util/pump_tasks.py` (`spawn_pump_free_task`,
`cancel_pump_free_tasks`). These name event-loop tasks, not procs. The
`sase/memory/tui_perf.md` bullet that mentions `spawn_pump_free_task()` stays as-is;
only its _separate_ tracked-background-task bullet changes.

**OUT OF SCOPE — the Muse/Antigravity provider protocol.** `task.lifecycle.proposed`,
`task.lifecycle.scheduled`, `task.lifecycle.output`, `task_kind`, and the `task_id` that
binds to a `call_id` are an external event vocabulary SASE consumes. Leave
`src/sase/llm_provider/_tool_call_muse.py`, `src/sase/llm_provider/agy.py`, and the
`docs/llms.md` Muse tables alone. The `docs/llms.md` sentences about provider tools
dispatching "background tasks" (lines ~328 and ~334) describe _provider_ behavior and
also stay.

**OUT OF SCOPE — immutable history and generic prose.** `CHANGELOG.md` (release-please
owns it), bead event streams under `sase/repos/beads/`, the plan archive under
`sase/repos/plans/`, and ordinary English "task" meaning "a unit of work" (for example
`docs/configuration.md`'s "External repositories are per-task repos").

### Compatibility precedent to copy

The `changespec` → `patch` and `vcs` → `stitch` renames are the reference
implementations. Read them before starting:

- **CLI alias**: `_COMMAND_REGISTRARS` in `src/sase/main/parser.py` maps both `patch`
  and `changespec` to `register_patch_parser`; `register_patch_parser` calls
  `add_parser("patch", aliases=["changespec"])`; `src/sase/main/entry.py` dispatches on
  `args.command in {"patch", "changespec"}`.
- **Facade modules**: `src/sase/main/parser_changespec.py`, `parser_vcs.py`, and
  `vcs_handler.py` are two-line star re-export shims keeping legacy import paths alive.
- **Symbol aliases**: `register_changespec_parser = register_patch_parser` at the bottom
  of `parser_patch.py`, exported through `__all__` with a `# legacy parser alias`
  comment.
- **Keymap / copy-group aliases**: `_LEGACY_APP_KEY_ALIASES`, `_migrate_key_aliases()`,
  and `_migrate_copy_group_aliases()` in `src/sase/ace/tui/keymaps/registry.py`.
- **Wire dual-key rule** (`../sase-core/README.md`, "JSON shape rules"): canonical
  records deserialize legacy keys and legacy records deserialize canonical keys, so
  mixed-version producers and consumers can overlap during a terminology migration.
- **Marker-guarded home-state migration**: `src/sase/agent/names/_migration.py` with
  `MIGRATION_MARKER_FILENAME` / `MIGRATION_SCHEMA_VERSION`.

### Cross-repo release mechanics

`../sase-core` is a linked repo. Open it with `sase repo open sase-core -r "<reason>"`
and use only the printed path. Never clone or web-fetch it.

Per `docs/rust_backend.md`, dev installs build `sase_core_rs` from the local
`../sase-core` checkout, so the `core` phase's binding changes are available to the
`store` phase immediately without a PyPI release. The `sase-core-rs` version window in
`pyproject.toml` is owned by the `sync-release-metadata` job — **no phase edits that
pin**. `release-plz` owns `../sase-core`'s crate versions and `CHANGELOG.md` — no phase
edits those either; use a Conventional Commit with a `!` or `BREAKING CHANGE:` footer
instead.

Because a released `sase` wheel can be paired with an older published `sase_core_rs`,
the `core` phase keeps the legacy PyO3 binding names as thin aliases and keeps legacy
`task_id` deserialization permanently on the read side. Retiring those aliases is a
follow-up, not epic scope.

### Naming decisions (apply uniformly)

| Concept                     | Old                                                                | New                                                                |
| --------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
| Feature name in prose       | Background Task / tracked task                                     | Proc                                                               |
| Rust module                 | `sase_core::tasks`                                                 | `sase_core::procs`                                                 |
| Rust record                 | `BackgroundTaskWire`                                               | `ProcWire`                                                         |
| Wire id field               | `task_id`                                                          | `proc_id`                                                          |
| Wire snapshot list key      | `tasks`                                                            | `procs`                                                            |
| PyO3 bindings               | `read_tasks_snapshot`, `append_task`, `update_task`, `prune_tasks` | `read_procs_snapshot`, `append_proc`, `update_proc`, `prune_procs` |
| Python package              | `sase.tasks`                                                       | `sase.procs`                                                       |
| Python record               | `BackgroundTask`                                                   | `Proc`                                                             |
| CLI                         | `sase task`                                                        | `sase proc` (alias `task`)                                         |
| Store                       | `~/.sase/tasks/tasks.jsonl`                                        | `~/.sase/procs/procs.jsonl`                                        |
| Logs                        | `~/.sase/tasks/logs/`                                              | `~/.sase/procs/logs/`                                              |
| Config key                  | `tasks.history_limit`                                              | `procs.history_limit`                                              |
| Admin Center tab id / label | `tasks` / "Tasks"                                                  | `procs` / "Procs"                                                  |
| TUI runtime                 | `TaskQueue`, `TaskMirror`, `TaskReporter`                          | `ProcQueue`, `ProcMirror`, `ProcReporter`                          |
| Indicator DOM id            | `#task-indicator`                                                  | `#proc-indicator`                                                  |

Local Python variables named `proc` for a `subprocess.Popen` handle already exist (~114
sites). Where a renamed record would shadow one in the same scope, rename the subprocess
local to `popen` or `process` rather than mangling the record name.

### Verification for every phase

Workspace directories are ephemeral, so always run `just install` first.

```bash
just install
just check
```

Run `just check-full` before any phase is considered done — this epic touches parsers,
config schema, keymaps, and TUI widgets, which is exactly the broadening set
`just check` warns about. Run it through `/sase_monitor` with a `--next` action, never
inline. The `tui-pane` and `labels` phases additionally need `just test-visual`. The
`core` phase runs `cargo test` and `just rust-install` in `../sase-core`.

Epic phase workers must not create beads. Record discovered follow-up work as a
`PROPOSED FOLLOW-UP:` note on your own phase bead.

---

## Phase `core`: Rename the Rust background-task core to procs

Open the repo first:
`sase repo open sase-core -r "Rename the background-task core to procs"`.

### Module and wire

- `git mv crates/sase_core/src/tasks crates/sase_core/src/procs`. The three files keep
  their names (`mod.rs`, `store.rs`, `wire.rs`).
- `crates/sase_core/src/lib.rs`: `pub mod tasks;` → `pub mod procs;` and update the
  `pub use tasks::{...}` re-export block to the new module and symbol names. Keep the
  file's existing alphabetical placement.
- `wire.rs` renames:
  - `TASK_WIRE_SCHEMA_VERSION` → `PROC_WIRE_SCHEMA_VERSION`, value `1` → `2`.
  - `BackgroundTaskWire` → `ProcWire`, with `pub task_id: String` →
    `pub proc_id: String`.
  - `TaskStoreStatsWire` → `ProcStoreStatsWire`, `TaskStoreSnapshotWire` →
    `ProcStoreSnapshotWire` (its `pub tasks: Vec<...>` field →
    `pub procs: Vec<ProcWire>`), `TaskAppendOutcomeWire` → `ProcAppendOutcomeWire`
    (`pruned_task_ids` → `pruned_proc_ids`), `TaskUpdateWire` → `ProcUpdateWire`
    (`task_id` → `proc_id`), `TaskUpdateOutcomeWire` → `ProcUpdateOutcomeWire` (its
    `pub task:` field → `pub proc:`), `TaskPruneOutcomeWire` → `ProcPruneOutcomeWire`
    (`pruned_task_ids` → `pruned_proc_ids`).
  - Leave `deserialize_present_option` and every `#[serde(...)]` attribute on the other
    fields exactly as they are.
- **Legacy read compatibility.** Add `#[serde(alias = "task_id")]` on
  `ProcWire::proc_id` and `ProcUpdateWire::proc_id`, and `#[serde(alias = "tasks")]` on
  `ProcStoreSnapshotWire::procs`, so a store written by the pre-rename version still
  loads. Serialization emits only the canonical keys. Add a unit test in `wire.rs` that
  deserializes a legacy-key JSON fixture into `ProcWire` and asserts the canonical keys
  round-trip out.
- `store.rs`: rename `append_task`/`update_task`/`prune_tasks`/`read_tasks_snapshot` →
  `append_proc`/`update_proc`/`prune_procs`/`read_procs_snapshot`, `TaskStoreError` →
  `ProcStoreError`, and every internal `task`/`tasks` identifier and doc comment. Do not
  change the locking, retention, or corrupt-line-tolerance behavior — this phase is a
  rename only. The store's own tests move with it; update their fixtures to the
  canonical keys and keep at least one legacy-key fixture.
- `mod.rs`: update both `pub use` blocks.

### PyO3 bindings

In `crates/sase_core_py/src/lib.rs`:

- Update the `use sase_core::tasks::{...}` import (line ~734) to
  `sase_core::procs::{...}` with the new symbol names.
- Rename the four `#[pyo3(name = ...)]` functions to `read_procs_snapshot`,
  `append_proc`, `update_proc`, `prune_procs`, and rename `task_store_error_to_pyerr`,
  `task_store_result_to_py`, and `background_task_from_pydict` to their proc spellings.
  The dict key the converter reads becomes `proc_id`, accepting `task_id` as a fallback
  so a stale Python caller keeps working.
- **Keep the four legacy binding names registered** as thin wrappers that call the
  canonical functions, each with a
  `// legacy binding alias; retire after the Python side ships` comment. This is what
  lets `store` land independently of a core release.
- Update the module docstring's binding inventory (lines ~80-83) to list the canonical
  names first and note the legacy aliases.
- Update the two inline binding tests (lines ~9015 and ~9042) that assert
  `["snapshot"]["tasks"][0]["task_id"]` to the canonical keys, and add one asserting the
  legacy alias binding still returns the canonical shape.
- Do **not** touch the bead bindings that carry "task" in their names
  (`bead_needs_task_ready_migration`, `bead_task_ready_migration_sql`,
  `add_task_plus_one`, `snooze_task`, `cancel_task_snooze`, `task_ready_migration_sql`)
  — those are task beads.

### Tests and docs in ../sase-core

- `crates/sase_core/tests/python_wire_parity.rs`: update the background-task fixture and
  its assertions; if the captured Python fixture pins `task_id`, regenerate it against
  the canonical keys and keep a legacy fixture for the alias path.
- `README.md`: update the wire inventory table row and add a bullet to the "JSON shape
  rules" list stating that `ProcWire` serializes canonical `procs` / `proc_id` keys and
  deserializes the legacy `tasks` / `task_id` keys for installed consumers, mirroring
  the existing `PatchWire`/`ChangeSpecWire` bullet.
- Do not edit `crates/sase_core/CHANGELOG.md`, `crates/sase_core_py/CHANGELOG.md`, or
  any `Cargo.toml` version.

### Verify

```bash
cargo test
cargo clippy --all-targets -- -D warnings
```

Then, from the sase workspace, `just rust-install && just check` to confirm the Python
side still imports and passes against the rebuilt extension through the legacy aliases.

Commit in `../sase-core` with a breaking-change Conventional Commit, e.g.
`refactor(core)!: rename the background-task store and wire to procs`.

---

## Phase `store`: Move the Python package to sase.procs and migrate on-disk state and config

### Package move

- `git mv src/sase/tasks src/sase/procs`. Files keep their names except `paths.py`,
  `ids.py`, `logs.py`, `models.py`, `runner.py`, `store.py`, `supervisor.py`,
  `__init__.py` — all stay, contents change.
- Rename throughout the package:
  - `BackgroundTask` → `Proc`; its `task_id` field → `proc_id`.
  - `TaskStoreSnapshot`/`TaskStoreStats`/`TaskAppendOutcome`/`TaskPruneOutcome`/
    `TaskUpdate`/`TaskUpdateOutcome` → `ProcStoreSnapshot`/`ProcStoreStats`/
    `ProcAppendOutcome`/`ProcPruneOutcome`/`ProcUpdate`/`ProcUpdateOutcome`.
  - `TASK_WIRE_SCHEMA_VERSION` → `PROC_WIRE_SCHEMA_VERSION`, value `2` to match `core`.
  - `ACTIVE_TASK_STATUSES`/`TERMINAL_TASK_STATUSES` → `ACTIVE_PROC_STATUSES`/
    `TERMINAL_PROC_STATUSES`; `COMMAND_TASK_KIND`/`TUI_TASK_KIND`/`DETACHED_TASK_KIND`/
    `TASK_KINDS` → the `_PROC_KIND`/`PROC_KINDS` spellings. **The kind string values
    (`"command"`, `"tui"`, `"detached"`) and the status values do not change.**
  - `ids.py`: `TaskRefError` → `ProcRefError`, `new_task_id`/`short_task_id`/
    `resolve_task_ref` → `new_proc_id`/`short_proc_id`/`resolve_proc_ref`,
    `TASK_ID_LENGTH`/`SHORT_TASK_ID_LENGTH`/`MIN_TASK_REF_LENGTH`/`TASK_ID_ALPHABET` →
    the `PROC_` spellings. Values are unchanged; ids stay 12-char base32.
  - `logs.py`: `append_task_log_text`/`delete_task_logs`/`open_task_log`/
    `read_task_log_tail`/`task_log_path` → the `_proc_` spellings.
  - `runner.py`: `TaskControlError`/`TaskSubmitError` → `ProcControlError`/
    `ProcSubmitError`; `kill_task`/`submit_task`/`submit_detached_task`/`wait_for_task`/
    `reconcile_running_tasks` → `kill_proc`/`submit_proc`/`submit_detached_proc`/
    `wait_for_proc`/`reconcile_running_procs`.
  - `store.py`: `TaskStoreLockTimeoutError` → `ProcStoreLockTimeoutError`;
    `append_task`/`filter_tasks`/`get_task`/`prune_tasks`/`read_tasks`/`update_task` →
    the proc spellings. Point `_call_binding` at the canonical Rust binding names from
    the `core` phase.
  - Rewrite the `__init__.py` facade imports and `__all__` to the new names, and its
    docstring to "Durable proc models, ids, logs, paths, and store facade."
- Do **not** add `sase.tasks` back as a facade shim. Nothing outside this repo imports
  it (verified: none of the five linked plugin repos reference `sase.tasks`), and a shim
  would keep the old spelling alive in autocomplete for no benefit.

### Paths and on-disk migration

- `paths.py`: `TASKS_SUBDIR = "tasks"` → `PROCS_SUBDIR = "procs"`,
  `TASK_STORE_FILENAME = "tasks.jsonl"` → `PROC_STORE_FILENAME = "procs.jsonl"`,
  `TASK_LOGS_SUBDIR` → `PROC_LOGS_SUBDIR` (value `"logs"` unchanged), and
  `tasks_dir`/`task_store_path`/`task_logs_dir` → `procs_dir`/`proc_store_path`/
  `proc_logs_dir`.
- Add `src/sase/procs/_migration.py` modeled on `src/sase/agent/names/_migration.py`:
  - `MIGRATION_SCHEMA_VERSION = 1`, `MIGRATION_MARKER_FILENAME = "procs_migration.json"`
    written into `sase_home()`.
  - When the marker is absent and `~/.sase/tasks/` exists: create `~/.sase/procs/`, move
    `tasks.jsonl` → `procs.jsonl` rewriting each record's `task_id` key to `proc_id`,
    move the `logs/` directory across, rewrite each record's `log_path` to the new
    directory, and leave the legacy `.lock` files behind (they are recreated on demand).
    Preserve the `.task-launch-submit.lock` and `.epic-launch-submit.lock` **names** —
    they belong to bead launches, not to this rename — but relocate them into
    `~/.sase/procs/`.
  - Skip cleanly and idempotently when the marker is present, when `~/.sase/tasks/` is
    absent, or when `~/.sase/procs/procs.jsonl` already exists. Never delete the legacy
    directory if any step fails; log and bail.
  - Run it lazily from the store facade's first read/write, guarded so concurrent CLI
    and TUI processes cannot both migrate (reuse `sase.logs._bounded.log_file_lock`).
- A running proc's recorded `log_path` points at the old directory until migration
  rewrites it; the migration must handle records for procs that are still `running`
  without disturbing their live writers — rewrite the record but keep the old path
  readable via a symlink, or defer those records and re-run on next start. Choose one
  and document it in the module docstring.

### Config key

- `src/sase/default_config.yml`: rename the `tasks:` block to `procs:`, keeping
  `history_limit: 100`, and reword the comment to "Maximum number of finished procs kept
  in `~/.sase/procs`. Running procs are never pruned."
- `src/sase/config/core.py`: rename `DEFAULT_TASK_HISTORY_LIMIT` →
  `DEFAULT_PROC_HISTORY_LIMIT` and `get_task_history_limit()` →
  `get_proc_history_limit()`. Read `procs.history_limit` first, fall back to
  `tasks.history_limit`, and keep `get_task_history_limit` as a legacy alias.
- `src/sase/config/layers.py`: add `"tasks": "procs"` to `DEPRECATED_TOP_LEVEL_KEYS` so
  `_collect_deprecated_keys()` warns users still setting the old block. Confirm
  `src/sase/config/_edit_plan.py` surfaces it through the existing `"deprecations"` map
  without further change.
- `src/sase/config/targets.py`: update its two task references.

### Call sites

Update every importer of `sase.tasks` (verified list): `src/sase/monitor/naming.py`,
`monitor/store.py`, `monitor/supervise.py` (docstring/comment cross-references only —
monitor ids and behavior are unchanged), `src/sase/bead/epic_launch.py`,
`epic_launch_handoff.py`, `task_launch.py`, `_task_gate_actions.py` (call sites only;
the bead-facing names in those last three stay), `src/sase/main/*` and
`src/sase/ace/tui/*` (those are `cli`, `tui-runtime`, and `tui-pane` — this phase only
needs the tree importable, so make the mechanical import/symbol updates there and leave
the module renames to those phases).

### Other

- `tools/validate_sase_core_rs`: replace the four binding names in its required list
  (lines ~123-126) with the canonical proc spellings.
- Rename `tests/test_tasks_facade.py` → `tests/test_procs_facade.py` and
  `tests/test_tasks_runner.py` → `tests/test_procs_runner.py`; update their contents.
- Add a regression test for the on-disk migration: a fixture `~/.sase/tasks/tasks.jsonl`
  with legacy `task_id` keys and a log file migrates to `~/.sase/procs/procs.jsonl` with
  `proc_id` keys, rewritten `log_path`, the marker written, and a second run a no-op.
- Add a regression test that `procs.history_limit` wins over `tasks.history_limit` and
  that the legacy key alone still works with a deprecation warning.

---

## Phase `cli`: Rename the sase task CLI command tree to sase proc

### Parser

- `git mv src/sase/main/parser_task.py src/sase/main/parser_proc.py`.
- Rename `register_task_parser` → `register_proc_parser`; inside, call
  `subparsers.add_parser("proc", aliases=["task"], ...)`.
- Rename the subparser dest `task_subcommand` → `proc_subcommand` (both
  `add_subparsers(dest=...)` and any `set_defaults`).
- Rename the positional `task_id` argument to `proc_id` on `kill` and `show`.
- Update the mirrored-constant comments at the top of the file
  (`ACTIVE_TASK_STATUSES | TERMINAL_TASK_STATUSES`, `TASK_KINDS`) to the proc spellings,
  and keep them spelled out rather than importing the store.
- Rewrite every `help=`, `description=`, and `epilog=` example to `sase proc ...`. Note
  in the group description that `sase task` remains a legacy alias. Keep the documented
  bare-default sentence, now reading "Running `sase proc` defaults to `sase proc list`."
- Update `--limit`'s help to reference `procs.history_limit`.
- Add `register_task_parser = register_proc_parser  # legacy parser alias`, exported
  through `__all__`, and add `src/sase/main/parser_task.py` as a facade shim in the
  exact shape of `src/sase/main/parser_changespec.py`:

  ```python
  """Legacy ``sase task`` parser facade for :mod:`sase.main.parser_proc`."""

  from sase.main.parser_proc import *  # noqa: F403
  from sase.main.parser_proc import __all__ as __all__
  ```

### Handler and render

- `git mv src/sase/main/task_handler.py src/sase/main/proc_handler.py` and
  `git mv src/sase/main/task_render.py src/sase/main/proc_render.py`.
- Rename `handle_task_command` → `handle_proc_command`, read `proc_subcommand`, and
  update the `noun, verb = ("task", "is") if hidden == 1 else ("tasks", "are")` line and
  every user-facing error/usage string to the proc spelling.
- In `proc_render.py`, rename `_task_json` → `_proc_json` and change the JSON envelope
  key from `"task"` to `"proc"` at both emit sites. This is a user-visible output
  change: state it in the commit body and in `docs/cli.md`.
- Keep `handle_task_command = handle_proc_command  # legacy handler alias` and add
  `src/sase/main/task_handler.py` and `task_render.py` as facade shims.

### Routing

- `src/sase/main/parser.py`: in `_COMMAND_REGISTRARS`, replace the `"task"` entry with a
  `"proc"` entry pointing at `("sase.main.parser_proc", "register_proc_parser")` and add
  a `"task"` entry pointing at the _same_ spec with a
  `# Legacy command alias for the proc parser.` comment. Both keys are required — the
  dict is deduplicated by spec when building the full parser and `parser_only_hint()`
  reads it to decide whether the narrow-parser fast path is safe. Keep the dict
  alphabetically sorted (`proc` sorts before `project`; `task` stays between `stitch`
  and `telemetry`).
- If `_COMPACT_ROOT_COMMANDS` carries a `task` entry, rename it to `proc` and reword the
  summary; `_validated_compact_root_commands()` asserts the name exists in the root
  subparser choices, so it must match the canonical name.
- `src/sase/main/parser_full_registrars.py`: import `register_proc_parser` from
  `sase.main.parser_proc` and update the `COMMAND_REGISTRARS_BY_NAME` entry, preserving
  the import block's ordering style.
- `src/sase/main/entry.py` (line ~407): change the guard to
  `if args.command in {"proc", "task"}:  # legacy command alias` and import
  `handle_proc_command` from `.proc_handler`.

### Tests

- `git mv tests/main/test_parser_task.py tests/main/test_parser_proc.py`,
  `tests/main/task_handler_helpers.py` → `proc_handler_helpers.py`, and each
  `tests/main/test_task_handler_*.py` → `test_proc_handler_*.py`; rewrite contents
  against `proc`/`proc_subcommand`.
- Add coverage that the legacy spelling still resolves: `parse_sase_args(["task"])` and
  `parse_sase_args(["task", "list"])` produce the same `proc_subcommand` values and
  defaults as the `proc` spellings, and that `sase.main.parser_task`,
  `sase.main.task_handler`, and `sase.main.task_render` remain importable and expose the
  legacy symbol names.
- `tests/main/test_parser_command_defaults.py`: replace `"sase task"` with `"sase proc"`
  in the expected list-group set; if the walker visits the alias choice as a separate
  path, include `"sase task"` too rather than special-casing the walker.
- Update `tests/test_command_catalog_build.py`, `tests/test_command_execution.py`, and
  `tests/test_timezone_display_cli.py`.

### Docs

- `docs/cli.md`: rewrite the five `sase task ...` table rows and the prose block at
  lines ~65-80 to the proc spelling, update the paths to `~/.sase/procs/procs.jsonl` and
  `~/.sase/procs/logs/`, retarget the `configuration.md#tasks` link to
  `configuration.md#procs`, and add one sentence noting `sase task` is still accepted as
  a deprecated alias. Also update the command list at line ~86. Leave the `ace.md`
  anchor links pointing at their current slugs — the `docs` phase renames those headings
  and fixes the anchors in the same commit.

---

## Phase `tui-runtime`: Rename the TUI tracked-task runtime to procs

**No displayed text changes in this phase.** That keeps the diff reviewable and leaves
the snapshot goldens untouched until `labels`. Verify by confirming `just test-visual`
passes against the existing goldens.

### Module renames

```
src/sase/ace/tui/task_queue.py                                   -> proc_queue.py
src/sase/ace/tui/task_mirror.py                                  -> proc_mirror.py
src/sase/ace/tui/task_subprocess.py                              -> proc_subprocess.py
src/sase/ace/tui/actions/task_actions.py                         -> proc_actions.py
src/sase/ace/tui/widgets/task_indicator.py                       -> proc_indicator.py
src/sase/ace/tui/actions/agents/_cleanup_tasks.py                -> _cleanup_procs.py
src/sase/ace/tui/actions/agents/_kill_tasks.py                   -> _kill_procs.py
src/sase/ace/tui/actions/agent_workflow/_launch_tasks.py         -> _launch_procs.py
src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_tasks.py
                                                                 -> _prompt_bar_save_xprompt_procs.py
src/sase/ace/tui/modals/plugins_browser_sase_update_tasks.py     -> plugins_browser_sase_update_procs.py
```

**Do not rename** `src/sase/ace/tui/util/pump_tasks.py` — those are asyncio tasks.

### Symbol renames

- `TaskQueue`/`TaskInfo` → `ProcQueue`/`ProcInfo`; `TaskMirror` → `ProcMirror`;
  `TaskReporter` → `ProcReporter`; `TaskIndicator` → `ProcIndicator`.
- `TaskActionsMixin` → `ProcActionsMixin`; `TrackedTaskResult`/`TrackedTaskCompletion`/
  `_TaskWorkerResult`/`_TaskCallbackConfig` → their `Proc` spellings.
- `_submit_tracked_task` → `_submit_tracked_proc`; `_submit_background_task` →
  `_submit_proc`; `_init_task_queue`/`_stop_task_mirror`/`_on_detached_task_count`/
  `_apply_detached_task_count`/`_update_task_indicator` → their proc spellings; the
  instance attributes `_task_queue`, `_task_workers`, `_task_completion_callbacks`,
  `_detached_task_count`, `_task_mirror` → `_proc_*`.
- `LaunchTaskMixin`/`CleanupTaskMixin`/`KillTaskMixin` and `LaunchTaskOutcome` → their
  `Proc` spellings.
- The `#task-indicator` DOM id → `#proc-indicator`, with the matching `styles.tcss`
  selector at line ~23. **Touch only that selector in `styles.tcss`** — the
  `QuitConfirmModal` and `TasksPane` blocks belong to `tui-pane`.

### Call sites

Update every importer. The verified set includes `src/sase/ace/tui/actions/base.py`,
`lifecycle.py`, `status.py`, `sync.py`, `axe.py`, `axe_bgcmd.py`, `agents_sync.py`,
`rename.py`, `proposal_rebase.py`, `_state_init_runtime.py`, the `actions/agents/*`
modules (`_approve.py`, `_dismissing.py`, `_loading_helpers.py`, `_marking.py`,
`_monitor_stop_flow.py`, `_notification_*.py`, `_revert.py`, `_tribe_assignment.py`,
`_wait_actions.py`), `actions/hints/_accept.py`, `_rewind.py`,
`actions/_artifacts_beads_*.py`, `actions/agent_workflow/_prompt_bar_stash.py`,
`_prompt_bar_save_xprompt_git.py`, `modals/plugins_browser_*.py`,
`modals/quit_confirm_modal.py`, `modals/runners_modal.py`,
`modals/notification_modal_action_support.py`, `modals/config_commit.py`,
`modals/__init__.py`, `widgets/__init__.py`, `commands/catalog.py`,
`src/sase/ace/handlers/mail.py`, `reword.py`, `src/sase/plan_approval_actions.py`,
`src/sase/_plan_approval_protocol.py`, `src/sase/main/plan_approve_handler.py`,
`src/sase/notification_gates/adapters.py`, and `src/sase/sessions/registry.py`.

Leave `src/sase/ace/tui/widgets/artifacts/commits.py` and `commits_pane.py` alone; their
"task" hits are unrelated.

### Keymaps

`src/sase/ace/tui/keymaps/registry.py` keeps `"task_queue"` inside
`_RETIRED_LEADER_KEYS` — that frozenset names a _historical user-config key_ that must
keep being dropped from stale overrides, so renaming it would resurrect the old binding.
Update only the explanatory comment above it to read "`task_queue` moved into the Admin
Center Procs tab".

### Tests

`git mv` and rewrite `tests/ace/tui/test_task_queue.py` → `test_proc_queue.py`,
`test_task_mirror.py` → `test_proc_mirror.py`, and
`tests/_agent_cleanup_task_helpers.py` → `_agent_cleanup_proc_helpers.py`. Update
`tests/ace/tui/test_agent_launch_non_blocking.py`, `test_agent_toggle_approve.py`,
`test_agent_tribe_assignment.py`, `test_failed_launch_stash.py`,
`test_launch_failure_logging.py`, `test_lazy_imports.py`, `test_log_panel_keymap.py`,
`test_notification_custom_gate.py`, `test_plugins_browser_pane_sase_update.py`,
`test_quit_confirm_modal.py`, `tests/ace/tui/actions/test_lifecycle_quit_confirm.py`,
`tests/ace/tui/_agent_wait_resume_helpers.py`, `tests/_plan_approval_tui_helpers.py`,
`tests/gate_conformance/_surfaces.py`, `tests/test_agent_launch_validation.py`,
`tests/test_launch_approval_tui.py`, and `tests/test_timezone_display_tui.py`.
`tests/ace/tui/test_lazy_imports.py` pins module paths — check it carefully.

---

## Phase `tui-pane`: Rename the ACE Tasks pane and Admin Center tab identifier to procs

**No displayed text changes in this phase either.** The tab still reads "Tasks" until
`labels`.

### Module renames

```
src/sase/ace/tui/modals/tasks_pane.py            -> procs_pane.py
src/sase/ace/tui/modals/tasks_pane_actions.py    -> procs_pane_actions.py
src/sase/ace/tui/modals/tasks_pane_render.py     -> procs_pane_render.py
src/sase/ace/tui/modals/tasks_pane_selection.py  -> procs_pane_selection.py
src/sase/ace/tui/modals/tasks_pane_store.py      -> procs_pane_store.py
src/sase/ace/tui/modals/tasks_store_rows.py      -> procs_store_rows.py
```

`TasksPane` → `ProcsPane`. Update `src/sase/ace/tui/modals/__init__.py`.

### Tab identifier

- `src/sase/ace/tui/modals/config_center_catalog.py`: change the tab id `"tasks"` →
  `"procs"` in the id tuple (line ~18) and in the catalog entry (lines ~131-137), rename
  `_tasks_pane_factory` → `_procs_pane_factory`, and point it at
  `ProcsPane(..., id="procs")`. **Leave the displayed label `"Tasks"` and the class-name
  string `"TasksPane"` for `labels`** — except the class-name string, which must track
  the real class name, so update it to `"ProcsPane"` here.
- `src/sase/ace/tui/modals/config_center_session.py`: `TasksSessionState` →
  `ProcsSessionState`, the `tasks:` field → `procs:`, and the `__all__` entry.
- The Admin Center's last-tab state is persisted (`~/.sase/admin_center_tab.json` and
  `~/.sase/ace_admin_center_last_tab.txt`). Add a read-side migration in
  `config_center_state.py` (or wherever the persisted value is loaded) that maps a
  stored `"tasks"` to `"procs"`, so a user who last had the tab open does not land on an
  unknown id. Cover it with a test.

### DOM ids and styles

Rename `#tasks-pane-title`, `#tasks-panels`, `#tasks-list-panel`, `#tasks-list`,
`#tasks-output-panel`, `#tasks-output-scroll`, `#tasks-output-content`, and
`#tasks-hints` to their `procs-` spellings in the widgets and in
`src/sase/ace/tui/styles.tcss` (lines ~6023 and ~6214-6266). Also rename
`QuitConfirmModal #quit-confirm-tasks` → `#quit-confirm-procs` and
`.quit-confirm-task-card` → `.quit-confirm-proc-card` (lines ~1406-1431) together with
`modals/quit_confirm_modal.py`. Update the `ConfigCenterModal TasksPane` selector to
`ProcsPane`.

Leave `GateDebugModal.kind-task_triage` at line ~1887 alone — task beads.

### Tests

`git mv` and rewrite `tests/ace/tui/test_tasks_pane.py` → `test_procs_pane.py`,
`test_tasks_pane_selection.py` → `test_procs_pane_selection.py`,
`test_tasks_pane_store.py` → `test_procs_pane_store.py`, `test_tasks_store_rows.py` →
`test_procs_store_rows.py`, `tests/ace/tui/_tasks_pane_helpers.py` →
`_procs_pane_helpers.py`, `tests/ace/tui/visual/_ace_config_center_tasks_helpers.py` →
`_ace_config_center_procs_helpers.py`, and
`tests/ace/tui/visual/test_ace_png_snapshots_config_center_tasks.py` →
`test_ace_png_snapshots_config_center_procs.py`. Update
`tests/ace/tui/test_config_center_tabs.py`, `test_config_center_resume.py`,
`test_admin_center_selection_resume.py`, and
`tests/ace/tui/visual/_ace_config_center_modal_helpers.py`.

Because the visual test _function_ names change, `git mv` the PNG goldens to match the
names the suite derives (`config_center_tasks_tab_120x40.png` →
`config_center_procs_tab_120x40.png`, `config_center_home_resume_tasks_*.png` →
`config_center_home_resume_procs_*.png`) **without re-rendering them** — the pixels are
unchanged in this phase. Confirm with `just test-visual`. Leave
`agents_task_bead_notes_120x40.png` and `custom_gate_task_triage_120x40.png` alone.

---

## Phase `labels`: Flip user-visible Task text to Proc and refresh snapshots

Now change what users read. Sweep for displayed strings, not identifiers.

- Admin Center tab label `"Tasks"` → `"Procs"` in `config_center_catalog.py`.
- `ProcsPane` title, region headers, hint bar, and empty-state copy.
- Command palette entries in `src/sase/ace/tui/commands/catalog.py`: "Open tasks panel"
  → "Open procs panel", and any "Background Tasks" phrasing.
- `quit_confirm_modal.py` and `actions/lifecycle.py`:
  `"N background tasks will be stopped"` → `"N procs will be stopped"`, and the
  confirmation body copy.
- `widgets/proc_indicator.py` and `actions/status.py`: tooltip and status-line text.
- `modals/runners_modal.py`: the **Background Tasks** section heading → **Procs**.
- Any toast/notification text produced by the `plugins_browser_*`, `sync`, `axe`,
  `mail`, `reword`, and `proposal_rebase` actions that says "background task".
- CLI help strings already flipped in `cli`; re-grep to confirm none were missed.

Then regenerate goldens:

```bash
just test-visual --sase-update-visual-snapshots
just test-visual
```

Inspect `.pytest_cache/sase-visual/` diffs before accepting. Expect changes in the
config-center procs tab, config-center home resume, and quit-confirm snapshots only —
any other golden moving means a selector or label was changed too broadly.

---

## Phase `docs`: Rewrite documentation, memory, skills, and the glossary

### Documentation

- `docs/ace.md` — the largest surface. Rename the headings
  `### Background Task Indicator` (~3022), `## Tasks Tab` (~4828),
  `### Task Status Icons` (~4871), and `### Durable Background Tasks` (~4882) to their
  Proc spellings, and rewrite their bodies including the `~/.sase/tasks/tasks.jsonl`
  path. Also update the quit/restart section (~2329-2340), the tracked-queue references
  (~2301, ~2796, ~2853, ~4233), and the `sase task` mentions. Leave every task-bead
  passage (~94, ~308-383, ~582-589, ~1045-1053, ~1760, ~2681-2685, ~3335-3377)
  untouched.
- `docs/cli.md` — fix the `ace.md#tasks-tab` and `ace.md#durable-background-tasks`
  anchors to the new heading slugs.
- `docs/configuration.md` — rename the `### tasks` section (~2649) to `### procs`,
  update its body and the TOC entry (~41), the Admin Center tab lists (~155, ~172-178),
  and the tracked-task sentences at ~223 and ~358-362. Document that
  `tasks.history_limit` is a deprecated alias. Leave the model-routing passages
  (~1435-1449) and the "per-task repos" sentence (~1789) alone.
- `docs/integrations.md` — rename the `## Durable Background Tasks` heading (~131), the
  background-task count sentence (~167), and the cross-link at ~178.
- `docs/beads.md` (~289-293, ~560, ~1834-1835), `docs/sdd.md` (~150-153),
  `docs/notifications.md` (~309), `docs/plugins.md` (~378) — these describe _task beads_
  launching _background procs_. Change only the background-task half of each sentence.
- `docs/monitors.md` (~10, ~13) — the phrase "provider-native background-task tools"
  describes provider tooling; reword to make clear it is not SASE's Procs feature, or
  leave it if already unambiguous.
- `INSTALL.md` (~59) — "tracked background task (watch it on the **Tasks** tab)" → Procs
  tab.
- `docs/perf_runbook.md` and `docs/architecture.md` — sweep for background-task
  mentions.
- Do **not** touch `docs/llms.md`; every hit there is the Muse protocol or provider
  behavior.

### Memory and skills

The user explicitly requested "every memory", which is the required permission for these
edits.

- `sase/memory/tui_perf.md` — rewrite bullet 3 (~30-35): "Run slow user-initiated
  operations as tracked procs", `_submit_tracked_proc()` / `_submit_proc()`
  (`src/sase/ace/tui/actions/proc_actions.py`), "proc indicator/Procs tab",
  `LaunchProcMixin` / `CleanupProcMixin`. **Leave bullet 1's `spawn_pump_free_task()` /
  `cancel_pump_free_tasks()` and the line-81 "asyncio task" reference exactly as they
  are.**
- `src/sase/xprompts/skills/sase_monitor.md` — the two "background-task" mentions
  describe _provider-native_ tools that do not work in SASE. Reword so they cannot be
  misread as SASE Procs, e.g. "provider-native monitor, background-execution, or
  scheduled wake-up tools".
- Leave `sase/memory/sase.md`, `sase_beads.md`, `sase_sizes.md`, `gotchas.md`,
  `xprompts.md`, `README.md`, and `src/sase/xprompts/skills/sase_new_task.md` alone —
  every hit is a task bead.

### Glossary

Add a `Proc` entry to the `memory.glossary` map in `sase/sase.yml`, alphabetically
between `Patch` and `Sase Project`, with
`aliases: [procs, background task, background tasks]` and a definition along the lines
of: "A durable background process SASE records, supervises, and can stream or kill.
Procs live in `~/.sase/procs/procs.jsonl` with combined output logs, come in `command`,
`tui`, and `detached` kinds, and are surfaced by `sase proc` and ACE's Procs tab.
Distinct from a task bead, which is a work item, and from an asyncio task." Listing
"background task" as an alias is what makes future agents resolve the old name to the
new concept.

### Regenerate

```bash
sase memory init
```

This regenerates `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, the
memory README, and the glossary note. Commit the regenerated files. Do not hand-edit
them.

---

## Phase `land`: Verify the migration and land the epic

1. `just install`, then `just check-full` through `/sase_monitor` with a `--next`
   action, plus `just test-visual`.
2. Residue sweep. Each of these must return only out-of-scope hits (task beads, asyncio,
   Muse, `CHANGELOG.md`, `sase/repos/beads/`, `sase/repos/plans/`):

   ```bash
   grep -rniI 'background task\|background-task' src tests docs sase/memory *.md
   grep -rniI 'sase\.tasks\|sase task \|tasks\.jsonl\|~/\.sase/tasks' src tests docs sase/memory *.md
   grep -rniI 'BackgroundTask\|TaskQueue\|TaskMirror\|TaskReporter\|TasksPane' src tests
   grep -rniI 'tasks\.history_limit' src tests docs
   ```

3. Confirm no emitter can write the old shapes: `grep -rn 'task_id' src/sase/procs`
   returns only the legacy-read fallback, and `grep -rn '"tasks"' src/sase/procs`
   returns nothing.
4. Verify the legacy surfaces still work end to end: `sase task list` and
   `sase task run -- true` succeed, `sase proc list --json` emits a `"proc"` envelope
   key, and a config file containing only `tasks: {history_limit: 5}` still applies with
   a deprecation warning.
5. Verify the on-disk migration on a real home: back up `~/.sase/tasks`, run any
   `sase proc` command, and confirm `~/.sase/procs/procs.jsonl` carries `proc_id` keys,
   the logs moved, running procs still stream, and the marker file was written.
6. Confirm `../sase-core` is committed and pushed with its breaking-change Conventional
   Commit, and that `tools/validate_sase_core_rs` passes against the locally built
   extension.
7. Land the epic's combined tree.
