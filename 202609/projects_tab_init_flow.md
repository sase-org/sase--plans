---
tier: tale
title:
  "Projects tab init flow: i/I gestures, InitPlanModal, and the streaming apply proc"
goal:
  "On the Admin Center Projects sub-tab, `i` initializes the marked or highlighted
  projects and `I` initializes every enabled project: each gesture plans off-thread via
  a `sase init … --check --json` session proc, toasts when everything is current, and
  otherwise shows an `InitPlanModal` preview whose confirmation submits exactly one
  streaming `sase init … --yes` proc that refreshes the pane in place."
size: medium
proposed_by: bbugyi200.apollo.sase-wm.2
bead: sase-wm.2
---

- **PARENT:**
  [202609/projects_tab_init.md](https://github.com/sase-org/sase--plans/blob/main/202609/projects_tab_init.md)
- **BEAD:**
  [sase-wm.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-wm/sase-wm.2.md)

# Plan: the `i`/`I` init gestures, `InitPlanModal`, and the streaming apply proc

Implements phase `flow` of epic `sase-wm`
(`sase/repos/plans/202609/projects_tab_init.md`). Phase `cli` (`sase-wm.1`) already
shipped `sase init -p NAME` (repeatable) and `sase init … --check --json`; this phase is
purely the Admin Center side plus one small `SessionProcReporter` extension.

**Read first**, through `/sase_memory_read`: `tui_perf.md` and `lint_and_test.md`. The
epic plan is the contract; re-read its `flow` section and its "Three hard constraints"
and "Decisions already made — do not relitigate" sections before starting.

## Non-goals for this phase

- **No `t` / "Run in terminal" button.** Phase `valve` (`sase-wm.3`) adds it. This phase
  only reserves the extension point: the modal dismisses a typed decision record whose
  `action` field is a one-member `Literal`, so `valve` widens the `Literal` instead of
  changing the callback contract.
- **No hint line, key help, docs, or PNG goldens.** Phase `polish` (`sase-wm.4`) owns
  all of those. Do not touch `hints_text()` in
  `src/sase/ace/tui/modals/project_management_rendering.py`, `docs/ace.md`,
  `docs/configuration.md`, or `docs/init.md`, and do not regenerate visual snapshots.
  Because pane rendering is unchanged, the existing Projects PNG goldens must stay
  green; if `just test-visual` disagrees, something in the pane changed that should not
  have.
- **No commit/no-commit control** for the memory step (explicit v1 non-goal). Apply uses
  blunt `--yes` semantics and the modal carries the warning.
- **Never edit** anything under `sase/memory/`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
  `OPENCODE.md`, `QWEN.md`, or `CHANGELOG.md`.
- **Do not create beads.** Record discovered follow-up work with
  `sase bead note sase-wm.2 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. Close
  only `sase-wm.2` when the work is verified — never the parent epic `sase-wm` or any
  other phase bead.

## Verified facts this plan is built on

Confirmed by reading the source in this repo; do not re-derive, but do re-check anything
you are about to depend on more deeply.

1. `sase init --check --json` emits one document via `emit_init_check_json`
   (`src/sase/main/init_check_json.py:78`):
   `{"schema_version": 1, "status": "current"|"drift"|"blocked", "projects": [...]}`
   (`src/sase/main/init_plan.py:140`). Each project row is
   `{name, display_name, status, unavailable_reason, planners: [...]}` plus an optional
   `error`. Each planner row is
   `{name, label, summary, actions, action_count, warnings, blockers, has_changes, runnable, requires_tty}`
   plus an optional `actions_truncated` (`serialize_init_plan`,
   `src/sase/main/init_plan.py:74`). Each action is
   `{path, operation, detail, new_content}` and, for binary payloads,
   `new_content_encoding: "base64"` (`_serialize_init_action`, same file).
2. Per-project `status` values come from `InitRunStatus`: `current`, `initialized`,
   `needs_attention`, `cancelled`, `failed`. A project the inventory held back is
   emitted as `status="failed"` **with** `unavailable_reason` set
   (`run_init_onboarding_all`, `src/sase/main/init_onboarding.py:616-628`).
3. `--check` exit codes conflate drift and blockers (both `1`), so the payload is the
   only truth. **But exit `1` does not guarantee a payload**: an inventory or
   name-resolution failure prints a red prose line and returns `1` _before_ any JSON is
   written (`init_onboarding.py:585-602`). Parsing must therefore fail loudly with the
   captured output, not crash.
4. `-p NAME` resolves against project name, display name, or alias, and rejects unknown
   or non-enabled names (`select_init_project_targets`,
   `src/sase/main/init_project_scope.py:147`). Passing the ProjectSpec directory key
   (`ProjectRecordWire.project_name`) always matches, and the payload's `name` field is
   that same key — so payload rows map back to pane records with no fuzzy matching.
5. Both `-p` and `--all` dispatch to `run_init_onboarding_all`
   (`src/sase/main/entry.py:280-283`), which prints `Project: <ref>` before each target
   (`_render_project_heading`, `init_onboarding.py:514`) and ends with
   `Initialization summary: <parts>` (`init_onboarding.py:711`). Both are usable as
   progress/outcome signals for a streamed apply.
6. Because the argv always carries `-p`/`--all`, the child's cwd is not
   scoping-significant; the coordinator `os.chdir`s per target itself
   (`_working_directory`, `init_onboarding.py:484`).
7. `_submit_session_worker` (`src/sase/ace/tui/actions/_proc_action_submission.py:189`)
   already implements dedup-key and exclusive-scope collision detection against live
   session workers, the durable projection, and pending durable scopes, and notifies
   with `duplicate_message` before returning `None`. No new collision machinery is
   needed.
8. `SessionProcReporter.run` (`src/sase/ace/tui/session_proc_reporter.py:158`) streams
   merged stdout+stderr into the proc log and returns a `CompletedProcess` whose
   `stdout` holds the full captured output. It has no way to suppress logging or observe
   lines — both of which this phase needs.
9. `tests/ace/tui/test_proc_producer_inventory.py` AST-scans every
   `_submit_session_worker` call (including the `submit = getattr(self.app, ...)`
   indirection) and keys sites by
   `(source_path, enclosing function name, kind, first positional arg, index)`. It also
   asserts `len(PRODUCTION_PRODUCERS) == 43`, which must become `45`.
10. `ace.keymaps.projects` in `src/sase/config/sase.schema.json` (~L2018) sets
    `"additionalProperties": false`, so new YAML keys without schema properties are a
    config-validation failure. `i` and `I` are unused in `ProjectsPaneKeymaps`
    (`src/sase/ace/tui/keymaps/app_keymaps.py:262`).

## Work

### 1. Keybindings, through the whole chain

Two new Projects actions: `initialize_project` (`i`) and `initialize_all_projects`
(`I`). Every link below must be updated or startup validation fails.

1. `src/sase/default_config.yml`, `ace.keymaps.projects` block: add
   `initialize_project: "i"` and `initialize_all_projects: "I"` after
   `set_current_project: "c"`. (Core-memory gotcha: keymap changes always update this
   file. The block has no `keep-sorted` markers.)
2. `ProjectsPaneKeymaps` (`src/sase/ace/tui/keymaps/app_keymaps.py:262-286`): add both
   fields with the same defaults. Field names must match the YAML keys exactly.
3. `_PROJECTS_BINDING_META` (`src/sase/ace/tui/keymaps/metadata.py:198-218`): add
   `("initialize_project", "initialize_project", "Initialize")` and
   `("initialize_all_projects", "initialize_all_projects", "Initialize All")`. This is
   what `build_projects_bindings` turns into live bindings.
4. `src/sase/config/sase.schema.json`, `ace.keymaps.projects.properties`: add both
   string properties with descriptions ("Initialize the selected (or marked) project(s)"
   / "Initialize every enabled project").
5. `_PROJECT_ONLY_ACTIONS` (`src/sase/ace/tui/modals/projects_pane.py:140-160`): add
   both action names so `check_action` makes them inert on the Repos/Workspaces
   sub-tabs.
6. `tests/test_keymaps_defaults.py::test_default_config_covers_all_projects_keymaps`:
   add both entries to the pinned dict.

`Enter` keeps meaning enable. Init is never implicit.

### 2. Scope and argv — `src/sase/ace/tui/modals/projects_pane_init.py`

A frozen, dependency-light module (no Textual imports) holding the scope contract.

```python
@dataclass(frozen=True, slots=True)
class InitScope:
    project_names: tuple[str, ...] = ()      # ProjectSpec directory keys
    display_names: tuple[str, ...] = ()      # parallel, for user-facing copy
    all_projects: bool = False
```

- Constructors `InitScope.for_projects(names, display_names)` and
  `InitScope.everything()` (not `all`, which shadows the builtin).
- `scope_key` → `"all"` for the all-projects scope, else `":".join(sorted(names))`.
  Sorting keeps the dedup key stable for the same set marked in a different order.
- `label` → `"all projects"`, the single display name, or `f"{n} projects"`.
- `cl_name` → the single project key, else `""`.
- `scope_flags` → `("--all",)` or the flattened `("-p", name)` pairs, in request order.
- `check_argv()` → `sase_argv("init", *scope_flags, "--check", "--json")`;
  `apply_argv()` → `sase_argv("init", *scope_flags, "--yes")`. Use `sase_argv` from
  `src/sase/ace/tui/actions/_durable_ops.py`.
- `init_cwd()` → `Path.home()`. Explicit and never scoping-significant (fact 6); the TUI
  must not manage cwd for project scoping.
- Timeouts, as module constants plus `check_timeout(count)` / `apply_timeout(count)`
  using `max(count, 1)`: `INIT_CHECK_STARTUP_SECONDS = 30.0`,
  `INIT_CHECK_PER_PROJECT_SECONDS = 25.0` (≈3× the measured ~8.4 s of planning work per
  project), `INIT_APPLY_STARTUP_SECONDS = 60.0`,
  `INIT_APPLY_PER_PROJECT_SECONDS = 180.0` (apply writes, commits, and pushes).

### 3. Payload parsing — `src/sase/ace/tui/modals/projects_pane_init_payload.py`

Typed, frozen, slotted mirrors of the phase-`cli` payload plus derived predicates. No
Textual imports; everything here runs on a worker thread.

```python
InitActionRow:   path, operation, detail, added, removed, diff_lines, diff_note
InitPlannerRow:  name, label, summary, has_changes, runnable, requires_tty,
                 warnings, blockers, actions, action_count, actions_truncated
InitProjectPlan: name, display_name, status, unavailable_reason, error, planners
InitCheckPayload: schema_version, status, projects, planned_at
```

Derived properties on `InitProjectPlan` (used by the modal and the toast copy):

- `unavailable` — `unavailable_reason` is not `None`.
- `held` — any planner has blockers.
- `requires_tty` — any planner with blockers has `requires_tty` (reserved for `valve`;
  used **now** to annotate blocker lines, so it is not dead code).
- `changed_runnable` — any planner with `has_changes and runnable`.
- `is_current` — not `unavailable`, not `held`, no planner has changes.

`parse_init_check_payload(stdout: str) -> InitCheckPayload` raising
`InitCheckPayloadError`:

- Try `json.loads(stdout)`; on failure retry the slice from the first `{` to the last
  `}` (Rich rendering is suppressed in `--json` mode, but be defensive about stray
  lines).
- If nothing parses, raise with a bounded tail (last ~10 lines, ~600 chars) of the
  captured output as the message — this is the path fact 3 creates, and the user must
  see the CLI's own red error line.
- Reject `schema_version != INIT_CHECK_JSON_SCHEMA_VERSION` (import the constant from
  `sase.main.init_plan`) with a message naming both versions and pointing at the `sase`
  binary on `PATH`. The TUI shells out, so a version skew is real, not theoretical.
- Validate types defensively (`status` in the three known values, `projects` a list of
  dicts) and coerce missing optional keys to safe defaults, so a slightly older/newer
  payload degrades instead of raising `KeyError`.
- `planned_at` is `sase.core.time.local_now()` captured at parse time.

### 4. Diffs, precomputed off the event loop — `src/sase/ace/tui/modals/projects_pane_init_diffs.py`

TUI perf rule 1 forbids disk I/O in render paths, and the epic requires "full unified
diffs with no second pass". So build every diff on the check worker thread, right after
parsing, and hand the modal pure text.

`attach_action_diffs(payload) -> InitCheckPayload` rebuilds the frozen tree with
`diff_lines`, `diff_note`, `added`, and `removed` filled in. Per action:

- `new_content_encoding == "base64"` → no diff, `diff_note = "binary content"`.
- `new_content is None` → no diff, `diff_note = "no file content in this plan"`
  (validate/deploy actions).
- Otherwise read the old text: only when the path is absolute and a readable file;
  anything else (relative path, missing file, `OSError`, `UnicodeDecodeError`) yields
  `""` for `create`, and `diff_note = "diff unavailable"` when reading an existing file
  failed. `delete` diffs old content against `""`.
- Produce `difflib.unified_diff(..., lineterm="")` with `fromfile`/`tofile` set to the
  display path; count `added`/`removed` from the body lines for the row diffstat.
- Cap at `MAX_DIFF_LINES = 400` per action and append an explicit `… N more diff lines`
  marker. Never truncate silently.

### 5. The check proc and the flow — `src/sase/ace/tui/modals/projects_pane_init_actions.py`

A new `ProjectsPaneInitActionsMixin`, mixed into `ProjectsPane` first in the MRO
(`class ProjectsPane(ProjectsPaneInitActionsMixin, ProjectManagementActionsMixin, …)`).
Declare the pane attributes and methods it borrows under `if TYPE_CHECKING:` exactly as
`ProjectManagementActionsMixin` does.

**`action_initialize_project`** — targets `self._target_records()`
(`src/sase/ace/tui/modals/project_list_controller.py:254`), which already means "the
marked set if any, else the highlighted row" and already sets its own status when it
returns empty. Then:

- Drop records that are not `state == "enabled"` or that are `system_managed` (the pane
  already excludes system-managed rows from `_records`; keep the guard anyway so the
  rule reads the same as `--all`'s inventory rule). Set a status message naming the
  skipped projects.
- If nothing survives, set a status **and** `notify(..., severity="warning")`, and
  submit nothing.
- Otherwise build `InitScope.for_projects(...)` from the survivors' `project_name` and
  `effective_project_name(record)`.

**`action_initialize_all_projects`** — ignores highlight, filter, and marks entirely and
builds `InitScope.everything()`. The modal's copy must say the scope is the canonical
`sase init --all` inventory, not the visible rows.

**`_start_init_check(scope)`**:

1. Set the visible status line **synchronously first**:
   `Checking initialization for {scope.label}…`.
2. `submit = getattr(self.app, "_submit_session_worker", None)`; if it is not callable,
   notify an error, restore the status, and return (same defensive shape as
   `memory_panel_publish_actions.py:116`).
3. Estimate the target count for the timeout: `len(scope.project_names)`, or for the
   all-projects scope the pane's own enabled-record count (cheap, already in memory),
   floored at 1.
4. Worker body (runs on a thread, takes the reporter):
   - `reporter.phase(f"Planning {scope.label}")`.
   - `completed = reporter.run(argv, cwd=..., timeout=..., log_lines=False)` — the raw
     JSON must not flood the Procs log (see §7).
   - `subprocess.TimeoutExpired` → failure result naming the timeout;
     `OSError`/`FileNotFoundError` → failure result naming the missing `sase` binary.
   - Exit code not in `(0, 1)` → failure result; log the captured tail through the
     reporter so the Procs tab entry is still useful (this is why `log_lines=False`
     needs the explicit-tail companion).
   - `parse_init_check_payload` then `attach_action_diffs`; `InitCheckPayloadError` →
     failure result carrying the message.
   - Success →
     `TrackedProcResult(success=True, message=<one-line summary>, payload=payload)`, and
     log that summary through the reporter.
5. Submit:
   ```python
   submitted = submit(
       "init-check",
       task,
       display_name=f"plan init · {scope.label}",
       cl_name=scope.cl_name,
       dedup_key=f"sase-init-check:{scope.scope_key}",
       exclusive_scopes=("sase-init",),
       duplicate_message="A project initialization is already running.",
       on_complete=self._on_init_check_complete,
   )
   ```
   The check claims the same global `sase-init` scope as the apply, which is what makes
   "activate `i` while a check _or_ an apply is in flight" warn exactly once and stack
   nothing. `submitted is None` means the machinery already notified; just set a status.
6. Keep the scope reachable from the completion callback (a small
   `self._init_scope_by_proc_id: dict[str, InitScope]` keyed by `submitted.proc_id`,
   cleaned up in the callback) rather than a single mutable "current scope" attribute —
   the exclusive scope makes concurrency impossible today, but a per-proc map cannot go
   stale.

**`_on_init_check_complete(completion)`** (UI thread):

- Failure or `payload is None` → status + `notify(..., severity="error")` with
  `completion.error or completion.message`.
- `payload.status == "current"` → **no modal**; toast only (the gesture's intent was
  "initialize", not "inspect"). Single project:
  `"{display} is initialized · {planner labels} are current"`, built from the payload's
  own planner labels so the list never lies about which planners ran. Multi/all:
  `"{n} projects are current · nothing to initialize"`. Also set the status line.
- Otherwise push `InitPlanModal` with `self.app.push_screen(modal, callback)`.

**`_on_init_plan_decision(decision, scope, payload)`** — `None` → status
`"Initialization cancelled"`. `decision.action == "apply"` → `_submit_init_apply`.

### 6. `InitPlanModal` — `src/sase/ace/tui/modals/init_plan_modal.py` (+ `_rendering.py`)

A `ModalScreen[InitPlanDecision | None]` built on `PluginActionConfirmModal`'s bones
(`src/sase/ace/tui/modals/plugin_action_confirm_modal.py:64-90`) but **not** on its
`PluginActionPreviewSection` model, which cannot carry the init glyph vocabulary,
per-planner warnings/blockers, or diffs. Do not leak "Plugin" naming into Projects.
Reuse `ConfirmKind` / `ButtonVariant` from `.confirm_dialog` and the shared
`confirm-dialog` / `confirm-dialog-panel` / `confirm-dialog--{kind}` CSS classes.

Split the file: the screen (bindings, `compose`, actions, scroll-hint sync) in
`init_plan_modal.py`; every pure `payload → RenderableType` builder in
`init_plan_modal_rendering.py`, so both stay well under the `toobig` limits and the
rendering is directly unit-testable without mounting an app.

Decision record, sized for `valve` to widen:

```python
InitPlanAction = Literal["apply"]          # valve adds "terminal"

@dataclass(frozen=True, slots=True)
class InitPlanDecision:
    action: InitPlanAction
```

Bindings: `escape`/`q`/`n` cancel, `y` confirm, `d` toggle diffs, `ctrl+d`/`ctrl+u`
scroll (copy the guarded `_can_scroll_down` / `_can_scroll_up` helpers). Reserve the `t`
slot for `valve` by leaving it unbound — no dead handler.

Content, top to bottom (mirrors the epic's mock):

1. **Aggregate line**, only for a multi-project or all-projects scope:
   `8 enabled · 5 need attention · 2 current · 1 unavailable`, counted from the payload
   (`changed_runnable or held` → needs attention). For the all-projects scope, one dim
   line stating the scope is the canonical `sase init --all` inventory — marks, filter,
   and highlight are ignored.
2. **The warning**, `bold yellow`, with prominence equal to the CLI prompt's
   (`init_onboarding.py:148-171`): "This can write files and may commit, push, create
   repositories, or deploy managed files." Add, when any memory planner has changes:
   "The memory step may commit and push generated project memory changes."
3. **`Would run`** + the exact apply argv verbatim via `shlex.join`.
4. **One section per project**, `held`/`changed_runnable` projects first in payload
   order, then a single dim summary line for current projects
   (`✓ Current  alpha, beta`). Each section carries:
   - a rule with the display name, plus the directory key when it differs, plus the
     project `status`;
   - `unavailable_reason` and `error`, red, verbatim;
   - one row per planner: the operation glyph, the uppercased planner label, the
     planner's **own** `summary`, and, when it has actions, ` · +A −R` from the
     precomputed diffstat. A planner with no changes and no blockers renders `✓` in
     `dim green`.
   - Glyphs are the CLI's vocabulary verbatim (`src/sase/main/init_preview.py:28-35`):
     `+` green create, `~` yellow update/overwrite, `−` red delete, `●` cyan
     validate/deploy. The row glyph is the highest-severity operation present (delete >
     overwrite > update > create > deploy > validate); each action line carries its own
     glyph.
   - action lines indented under the planner: glyph, path, dim detail; then the
     `actions_truncated` marker when the payload set it.
   - `warnings` yellow and `blockers` red, verbatim. A blocker on a planner with
     `requires_tty` gets a dim `(needs a terminal)` suffix — honest today, and the hook
     `valve` builds its button on.
5. **Diffs** when toggled on: the precomputed unified diff under each action, `+` green,
   `−` red, `@@` dim, plus `diff_note` when there is no diff.
6. A dim footer: the plan timestamp and "confirm re-plans fresh" — the apply invocation
   plans and applies in one process (`_run_init_onboarding_result`), so the preview is
   never applied stale.

Buttons and framing:

- Border title: `↻  Initialize sase` (single) or `↻  Initialize 5 runnable projects`.
- Primary button: the **same specific** copy plus ` (y)` — never "OK". "Runnable" counts
  projects with `changed_runnable`; held projects are excluded, and the modal says held
  projects are left unchanged.
- **Zero runnable projects** (everything blocked or unavailable): the modal still opens
  so the blockers are visible, but the primary button is disabled and labelled
  `Nothing runnable`, and `action_confirm` no-ops. Submitting `--yes` for a fully
  blocked scope would produce a proc that changes nothing and a misleading toast.
- `ConfirmKind.DANGER` when any runnable planner has an `overwrite` or `delete` action —
  danger styling only, no name-typing ceremony.
- Border subtitle: `y run · d diff · esc cancel`; flip to `d hide diffs` when diffs are
  on.
- Add the modal's CSS to `src/sase/ace/tui/styles.tcss` next to the
  `PluginActionConfirmModal` block (~L8546), reusing its width / `max-height` /
  scrollable-preview pattern and its `confirm-dialog--danger` border override.

### 7. `SessionProcReporter.run` — line hook and log suppression

`src/sase/ace/tui/session_proc_reporter.py`. Extend `run` with two keyword-only
parameters, both defaulting to today's behaviour:

```python
def run(self, argv, *, cwd=None, env=None, timeout=None, stream="stdout",
        log_lines: bool = True, on_line: LineCallback | None = None) -> ...
```

The internal callback logs when `log_lines` is true and then calls `on_line`. Wrap the
`on_line` call in `try/except Exception` and log the failure once through the reporter:
that callback runs on the subprocess reader thread, and an exception there would kill
streaming for the rest of the run. `subprocess_run_fn` and the other adapters keep
today's behaviour untouched.

### 8. The apply proc

**`_submit_init_apply(scope, payload)`** submits **exactly one** session worker:

```python
submitted = submit(
    "init-apply",
    task,
    display_name=f"init · {scope.label}",
    cl_name=scope.cl_name,
    dedup_key=f"sase-init:{scope.scope_key}",
    exclusive_scopes=("sase-init",),
    duplicate_message="A project initialization is already running.",
    on_complete=self._on_init_apply_complete,
)
```

`exclusive_scopes=("sase-init",)` is deliberately global: selected-project and
all-project scopes share chezmoi deployment, so they must never overlap. Cross-process
races stay covered by the chezmoi lock inside the coordinator (`defer_chezmoi_deploy`,
`init_onboarding.py:612`) — the TUI must not re-implement it.

Worker body:

- `reporter.set_command(argv)` is done by `reporter.run`; call
  `reporter.phase(f"Initializing {scope.label}")` first.
- Pass an `on_line` hook that matches the coordinator's own `Project: <ref>` headings
  (fact 5) and calls `reporter.phase(f"Project {i} of {total} · {ref}")`, where `total`
  is the payload's project count. No planner-level phase and no percentages — do not
  invent precision the output does not carry.
- Timeout from `apply_timeout(len(payload.projects))`.
- Map the outcome from the coordinator's own `Initialization summary: <parts>` line plus
  the exit code:
  - exit `0` → success;
  - non-zero with a summary line naming any `initialized` count → partial (`warning`);
  - otherwise → failure (`error`). When no summary line is present (timeout, kill,
    crash), fall back to the exit code and say so rather than guessing counts.

**`_on_init_apply_complete(completion)`** (UI thread):

- Toast that distinguishes success / current / partial / failure, echoing the
  coordinator's own summary parts where available, e.g.
  `Initialized 5 · 2 current · 1 needs attention — see Procs`.
- Refresh **in place**: capture `self._selected_project_name()` first, then reuse the
  pane's existing reload path (`action_reload_projects`, which calls `_load_records`,
  `_refresh_options(preferred_project=…)`, `_start_inventory_load`, and
  `_start_current_project_resolve`), then set the init status line so it is not
  overwritten by the reload's `"Reloaded"`. Filter text and the active sub-tab are pane
  state that reload does not touch, so they survive untouched.
- **Never** auto-switch to the Procs tab.

### 9. Proc-producer inventory

Add two `site(...)` entries to `src/sase/ace/tui/_proc_producer_sites_actions.py`
(`ACTION_PRODUCERS`), matching the AST scanner's key exactly — source path, **enclosing
function name** (`_start_init_check`, `_submit_init_apply`), `kind="session_worker"`,
and the literal first positional argument (`"init-check"`, `"init-apply"`):

- `projects.init_check` — `ui_only`, `domain_command="sase init --check --json"`,
  `identifiers=("scope_key",)`, `result_kind="init.check"`,
  `concurrency_keys=("sase-init",)`,
  `optimistic_ui="status line; preview modal or no-op toast"`,
  `restart_recovery="not durable; the read-only plan is session-local"`.
- `projects.init_apply` — `ui_only`, `domain_command="sase init --yes"`,
  `identifiers=("scope_key",)`, `result_kind="init.apply"`,
  `concurrency_keys=("sase-init",)`,
  `optimistic_ui="status line; streamed proc; toast; in-place reload"`,
  `restart_recovery="not durable; session-local init apply"`.

Then bump `len(PRODUCTION_PRODUCERS) == 43` to `45` in
`tests/ace/tui/test_proc_producer_inventory.py`.

### 10. Tests

New files under `tests/ace/tui/modals/` unless noted:

1. `test_projects_pane_init_scope.py` — scope→argv for every shape (single, multi, all);
   `scope_key` stability under reordering; `label` / `cl_name`; timeout scaling for 1,
   3, and 8 targets.
2. `test_projects_pane_init_payload.py` — parse a current payload, a drifted payload,
   and a blocked payload (drift vs blocked distinguished despite identical exit codes);
   `requires_tty` surfaced; `actions_truncated` preserved; an unavailable project row
   (`status="failed"` **plus** `unavailable_reason`) classified `unavailable`, not just
   failed; schema-version mismatch rejected with a version-naming message; non-JSON
   stdout rejected with the captured tail in the message.
3. `test_projects_pane_init_diffs.py` — a create action against a missing file (all
   added), an update against a real `tmp_path` file (correct `added`/`removed`), a
   base64 action (`binary content`, no diff), a `new_content is None` action, a relative
   or unreadable path (`diff unavailable`, no raise), and the `MAX_DIFF_LINES` marker.
4. `test_init_plan_modal.py` — render the modal for: a single-project update, a mixed
   all-projects payload (aggregate line, current projects on one summary line), a danger
   payload (overwrite/delete → `ConfirmKind.DANGER`), a TTY-blocked payload (blocker red
   and annotated, primary disabled when nothing is runnable), and the `d` toggle showing
   and hiding diffs. Assert on the rendered plain text plus the button label, so the
   `polish` phase's PNG goldens are not this phase's job.
5. `tests/ace/tui/test_projects_pane_init_flow.py` — the pilot, using `AcePage` and the
   `_patch_panes` helper style from `tests/ace/tui/test_projects_pane.py`, with
   `_submit_session_worker` recorded (monkeypatched on the app) rather than really
   spawning `sase`:
   - `i` on a drifted project submits exactly one `init-check` with the expected argv,
     dedup key, and `("sase-init",)` scope, and sets the status line synchronously;
   - the check completion pushes `InitPlanModal`, and confirming submits exactly one
     `init-apply` with `sase init -p <name> --yes`, `dedup_key="sase-init:<key>"`, and
     `exclusive_scopes=("sase-init",)`;
   - a second activation while one is in flight warns and submits nothing (drive this
     through the real `_submit_session_worker` collision path, not a stub, so the test
     covers fact 7);
   - a `status: "current"` payload opens no modal and toasts;
   - `I` ignores marks and the filter and submits `sase init --all --check --json`;
   - marking a disabled project and pressing `i` filters it out with a naming status
     message, and marking **only** disabled projects submits nothing;
   - entering the Projects tab submits nothing at all (init is never eager).

## Sequencing

Chain 1 → 2 → 3 → 4 → 7 (mechanical, independently testable), then 5 → 6 → 8 (the flow),
then 9, then 10. Run `just check` after the keymap chain lands — a mismatch between the
YAML keys, the dataclass fields, and the schema fails at startup, and catching it early
is cheaper than debugging it under a modal.

## Verification

1. `just install` first — this workspace is an ephemeral clone with its own virtualenv
   and may have sat unused while pinned dependencies changed.
2. `just check` (inline is fine; hand it to `/sase_monitor` if it runs long). Not
   `just check-full`: that is the `verify` phase's job, and only through a monitor.
3. `just test-visual` must still pass **unchanged** — this phase touches no pane
   rendering. Do not pass `--sase-update-visual-snapshots`.
4. Targeted runs while iterating:
   `just test tests/ace/tui/test_proc_producer_inventory.py tests/test_keymaps_defaults.py`
   plus the new files.
5. Symvision: everything added must be reachable. `requires_tty` is consumed by the
   blocker annotation and `InitPlanAction` by the decision record, so no `--epic-symbol`
   entry should be needed. If one becomes unavoidable, key the Justfile line to
   `sase-wm.3` (the `valve` phase that consumes the symbol), never to `sase-wm.2` —
   entries keyed to this phase go stale the moment it closes and turn other agents'
   `just check` red.

## Done when

`i` and `I` plan off-thread, preview truthfully or toast on no-op, submit exactly one
streaming apply proc on confirm, and refresh in place preserving selection, filter, and
sub-tab; entering the tab performs no init work; the preview never writes; planning and
apply never touch the event-loop thread; the proc-producer inventory test passes; and
`just check` passes.
