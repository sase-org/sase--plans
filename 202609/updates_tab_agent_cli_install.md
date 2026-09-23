---
tier: epic
title: Install agent CLIs from the Admin Center Updates tab
goal: 'A missing agent CLI can be found, previewed, and installed from the SASE Admin
  Center Updates tab, either one at a time or as a marked bulk set. Installs go through
  the same confirmed, shell-free installer that backs `sase agent-cli install`, which
  now also installs npm-packaged CLIs. The previewed bytes are exactly the bytes that
  run, and every outcome (success, not on PATH, failure) stays visible afterwards.

  '
phases:
- id: npm-installs
  title: Shared installer learns npm-packaged CLIs
  depends_on: []
  size: medium
  description: 'npm-installs: add a pure install-route classifier, npm-package install
    planning with npm/PATH/writability checks, plan-time PATH status, and a per-entry
    progress hook in sase.agent_clis; update `sase agent-cli install` rendering, JSON,
    help, hints, and docs.'
- id: tui-install
  title: Install flow in the Updates tab
  depends_on:
  - npm-installs
  size: medium
  description: 'tui-install: give missing agent-CLI rows an install verb on i / Space
    / I, a redesigned install detail panel, a digest-bearing confirm preview, a tracked
    sequential install proc, one combined flow for mixed plugin + CLI marks, install-aware
    history, result lines, and toasts, plus tests, docs, and goldens.'
- id: discover-bulk
  title: Discoverability and bulk-select accelerators
  depends_on:
  - tui-install
  size: medium
  description: 'discover-bulk: add the Available scope, the `*` mark-all-like-this
    key, a cross-scope hint when a filter matches nothing, the final detail call-to-action,
    docs, and goldens.'
proposed_by: bbugyi200.athena.0q6
create_time: 2026-09-23 11:58:13
status: done
bead_id: sase-171
---

- **PROMPT:** [prompts/202609/updates_tab_agent_cli_install.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/updates_tab_agent_cli_install.md)
- **BEAD:** [sase-171](https://github.com/sase-org/sase--beads/blob/main/pages/sase-171/README.md)

# Plan: Install agent CLIs from the Admin Center Updates tab

## Context: what exists today

- **CLI installer.** `sase agent-cli install` (`src/sase/agent_clis/install.py`,
  `src/sase/agent_clis/cli_install.py`) runs only provider-declared **install scripts**.
  SASE fetches the script, shows its URL, SHA-256, command, and target, asks for
  confirmation, and runs it without a shell. Today only Muse Code declares a script. The
  npm-managed CLIs (Claude Code, Codex CLI, OpenCode, Qwen Code, Grok Build) are
  reported as skips (`declares no SASE-runnable install script; npm install -g …`).
  Antigravity (`manager: native`) can only be installed manually. Bulk install is not
  useful while only one CLI can be installed, so phase `npm-installs` adds npm packages
  to the shared installer.
- **Updates tab.** The Updates tab is `PluginsBrowserPane`
  (`src/sase/ace/tui/modals/plugins_browser_pane.py` plus its `plugins_browser_*.py`
  mixins). Agent-CLI rows are `UpdateRow(kind="agent-cli")`, built on the load worker
  thread by `build_update_rows()` in `plugins_browser_rows.py`. A not-installed CLI
  shows only in the **All** scope (the default scope is **Installed**), cannot be marked
  (`Select an updatable agent CLI to mark.`), and `i` does nothing for it.
- **Marks.** All marks share one row-key set (`self._marked`), and each row's capability
  decides what its mark means: `install` on installable plugins, `mark_update` on
  updatable CLIs. `i` installs the marked plugins or the highlighted one. `A` updates
  the marked CLIs, or every CLI that can be updated.
- **Confirm modal.** `PluginActionConfirmModal`
  (`src/sase/ace/tui/modals/plugin_action_confirm_modal.py`) can already render titled
  `PluginActionPreviewSection`s with counts, details, commands, and skips. The
  comprehensive update preview already uses it.
- **Backend boundary.** Agent-CLI planning and execution live only in the Python
  `sase.agent_clis` package, which the CLI and the TUI share. sase-core has no
  counterpart, so this epic changes no Rust code. The TUI must **call** the shared
  planner and executor and must never copy their logic. (The
  `rust_core_backend_boundary` rule is satisfied because the logic stays shared and
  frontend-neutral.)

## UX principles

1. **The action follows the row, with no new mode.** On a missing agent CLI the action
   is _install_, triggered by the same keys that install a plugin: `i` installs,
   `Space`/`I` marks. A user who already knows the plugin flow already knows this one.
2. **Preview first, and run exactly what was previewed.** Every install opens the
   confirm modal first, even a single one. The modal shows the exact command, the script
   URL, the full SHA-256, the target directory, and whether that directory is on `PATH`.
   The TUI runs the plan it already fetched inside a session proc, so the confirmed
   bytes are the bytes that run. Do **not** shell out to `sase agent-cli install --yes`,
   because that would fetch the script again.
3. **One marked set, one keypress.** `i` installs everything that is marked for install,
   whether plugins, CLIs, or both.
4. **Never silent.** Every key pressed on a missing CLI row gives visible feedback: a
   modal, a toast, or a mark.
5. **Discoverable.** Missing CLIs are one scope away (**Available**), and a filter that
   finds nothing in the current scope says where the matches are.
6. **Honest afterwards.** "Installed but not on PATH" and failures stay visible in the
   row, the detail panel, the toast, and the persisted history.

## Keymap (final state) and conflict audit

| Key                                     | Context                                       | Action                                                                                                 |
| --------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `i`                                     | highlighted missing agent CLI, nothing marked | Preview and install that CLI                                                                           |
| `i`                                     | any install marks exist                       | Preview and install **every** install-marked row (CLIs and plugins)                                    |
| `i`                                     | missing CLI that SASE cannot install          | Warning toast with the manual instructions and docs URL                                                |
| `Space` / `I`                           | missing installable CLI                       | Toggle its install mark; cursor moves to the next installable row in the same section                  |
| `*`                                     | any markable row (phase `discover-bulk`)      | Mark every visible row in the same section with the same action; unmark them if all are already marked |
| `Esc`                                   | —                                             | Clear every mark (existing), then close                                                                |
| `]` / `[`                               | —                                             | Cycle scopes, including **Available** (phase `discover-bulk`)                                          |
| `y` / `n` / `Esc` / `Ctrl+D` / `Ctrl+U` | confirm modal                                 | Confirm, cancel, scroll (existing modal)                                                               |

Keys already in use while this tab has focus:

- **Pane:**
  `j k [ ] i I space x m u A H a U r ctrl+d ctrl+u g G shift+g o v / ' escape`.
- **List widget:** `ctrl+d ctrl+u g G` plus the `OptionList` defaults.
- **Admin Center modal:** `escape q 0-9 tab shift+tab` and the opener (`#`).
- **Confirm modal:** `y n q escape g ctrl+d ctrl+u`.

The only new key is `*` (Textual `asterisk`). Rejected alternatives:

- `ctrl+a` (the precedent from the revive modal): it is a common tmux prefix, and it is
  _this user's_ tmux prefix, so it never reaches the TUI.
- `a`: sync agents.
- `m` / `M`: switch mode.
- `v` / `V`: verbose.

The app-level `*` (Agents saved-query picker) is a non-priority binding, so the focused
pane's binding shadows it. When the filter `Input` has focus it consumes `*` as text.

`src/sase/default_config.yml` does not change. The Updates pane's bindings are
hard-coded in `PluginsBrowserPane.BINDINGS`, not configurable keymaps (see the "Default
Keymap Config" gotcha).

## Visual design

Agent CLIs section in the **Available** / **All** scope (not-installed rows reuse the
plugin convention: `○` glyph and a `latest v…` label):

```text
── Agent CLIs ──
    ● Claude Code      v1.0.0 → v1.1.0  [self managed]  ↑
[✓] ○ Qwen Code        latest v0.8.0  [npm]
[✓] ○ Muse Code        latest v1.3.0-R3401.1  [script]
    ○ Antigravity CLI  not installed  [manual]
    ○ OpenCode         latest v1.18.0  [npm]  ⚠ not on PATH      ← installed this session, not on PATH
```

The marked-work line above the hints reads:
`Marked: 1 plugin install · 2 CLI installs · 1 CLI update (1 hidden by filter)`.

Detail panel for a missing installable CLI. The border uses the provider accent color,
and the title is `<Name> · not installed`:

```text
Provider         qwen
Binary           qwen
Status           not installed
Latest           0.8.0
Install via      npm · @qwen-code/qwen-code
Install command  npm install -g @qwen-code/qwen-code
Documentation    https://github.com/QwenLM/qwen-code

↓ i install now · Space mark for a bulk install
```

- **Script installers** replace the npm rows with `Install via  install script`,
  `Script  https://…`, `Runs  bash <downloaded script> — SHA-256 shown before it runs`,
  and `Target  ~/.local/bin (MUSE_INSTALL_DIR overrides)`.
- **Manual CLIs** show `How to install  …` and a yellow
  `○ SASE can't run this installer — follow the documentation above` in place of the
  call to action.
- **Installed this session but not on PATH:** the call to action becomes the yellow
  `⚠ Installed to ~/.local/bin, which is not on PATH — add export PATH="~/.local/bin:$PATH" to your shell startup file, then restart sase`.

Confirm modal (icon `↓`, panel title `Confirm agent CLI install`):

```text
╭ ↓  Install 2 agent CLIs ─────────────────────────────────────────────╮
│ Installs 2 agent CLIs, one at a time                                  │
│ ───────────── Qwen Code · npm · latest v0.8.0 ─────────────────────── │
│ - target ~/.npm-global/bin · on PATH                                  │
│ commands                                                              │
│  $ npm install -g @qwen-code/qwen-code                                │
│ ───────────── Muse Code · install script · latest v1.3.0-R3401.1 ──── │
│ - script https://dev.meta.ai/install.sh · 48213 bytes                 │
│ - sha256 <full 64-hex digest>                                         │
│ - target ~/.local/bin · not on PATH (SASE prints the export line)     │
│ commands                                                              │
│  $ bash ~/…/agent-clis/install-3f2a9c….sh                             │
│ Skipped                                                               │
│ - Antigravity CLI: no SASE-runnable installer — see https://…         │
│ - Runs without a shell · never edits your shell startup files         │
│              [ Confirm (y) ]    [ Cancel (n) ]                         │
╰ y confirm · n/esc cancel ─────────────────────────────────────────────╯
```

Toasts after a run:

- **All succeeded:** `Installed Qwen Code 0.8.0 · Muse Code 1.3.0-R3401.1`
  (information).
- **Any target not on PATH:** a warning that includes the exact `export PATH=…` line.
- **Any failure:** one line per CLI, at error severity.

History shows install runs with a green `↓`, the text `installed 0.8.0`, and the run
badge `i`. CLI-triggered runs keep the `CLI` badge, and update runs are unchanged.

## Phase `npm-installs`: Shared installer learns npm-packaged CLIs

Files: `src/sase/agent_clis/{models,install,detect,cli_install,__init__}.py`,
`src/sase/main/parser_agent_cli.py`, `src/sase/llm_provider/_hookspec.py` (docstring
only), and docs.

1. **Route enum.** Add `InstallRoute(StrEnum)` to `models.py` with `SCRIPT`, `NPM`,
   `MANUAL`, and `BUNDLED`.
2. **Pure classifier.** Add `AgentCliInstallOption` (frozen) to `install.py` with the
   fields `route`, `installable: bool`, `source: str | None` (script URL or npm
   package), `command_hint: str` (display text), and `reason: str | None` (why it is not
   installable, with the docs URL appended).
   `describe_agent_cli_install(status) -> AgentCliInstallOption` performs **no I/O**:
   - `manager: bundled` → `BUNDLED`, not installable.
   - `installs_from_script` → `SCRIPT`.
   - `install_manager == "npm"` with a `package` → `NPM`, with
     `command_hint="npm install -g <package>"`.
   - Anything else → `MANUAL`, with a reason derived from `install_hint` and the docs
     URL.

   The classifier ignores `status.installed`; callers combine the two. It must stay pure
   so the TUI can call it during row building and detail rendering (tui_perf rule 8:
   render paths never stat or glob).

3. **Planner dispatch.** `plan_agent_cli_install_status` dispatches on the classifier's
   route. The script branch stays byte-for-byte compatible. The new npm branch runs at
   plan time only (CLI process or TUI worker thread) and takes injectable
   `run_fn`/`writable_fn` parameters with real defaults:
   - Resolve `npm` with `resolve_executable("npm", env=env)`. If it is missing, return a
     skip: `npm is not on PATH; install Node.js (which ships npm) and retry`, plus the
     docs URL.
   - Probe `npm prefix -g` / `npm root -g` through a **public** helper promoted from
     `detect._probe_npm_environment` (for example `probe_npm_global_environment(...)`).
     Keep detection's internal callers on that helper, and do not use private functions
     across modules, because symvision flags it.
   - Check writability on the **nearest existing ancestor** of the global root, because
     a fresh user prefix may not have `lib/node_modules` yet. If it is not writable,
     skip with the same wording the update planner uses for a non-writable npm root,
     including the exact manual command. Never run sudo.
   - Set `argv=("npm", "install", "-g", <package>)`, `env_overlay=status.install_env`,
     and `install_dir=<npm prefix -g>/bin`.
4. **New entry fields.** `AgentCliInstallEntry` gains `route: InstallRoute` and
   `install_dir_on_path: bool | None`. The PATH flag is computed at plan time with the
   existing `_directory_on_path` for any resolved `install_dir` (script or npm), so
   previews can show `on PATH` / `not on PATH` before anything runs. Presenters branch
   on `route` instead of inspecting `argv`/`script`.
5. **Progress hook.** Add an optional parameter to `execute_agent_cli_installs`:
   `progress_fn: Callable[[int, int, AgentCliInstallEntry], None] | None = None`. It is
   called before each runnable entry with the 1-based index, the runnable total, and the
   entry. Nothing else changes: execution stays sequential, continues after a failure,
   records history, and finds npm binaries through `install_dir`.
6. **Install hint.** In `detect.py`, `_install_hint` for npm becomes `run \`sase
   agent-cli install <name>\` (npm install -g <package>)`. The script and manual wording
   stays. Update any tests that pin the old text.
7. **CLI presentation** (`cli_install.py`):
   - `_render_plan` gains an npm branch: `●` name, then `package:`, `command:`, and
     `target:` lines, each followed by an on-PATH / not-on-PATH marker. The script
     branch also gets the marker.
   - The prompt becomes `Run the install command(s) above? [y/N]`.
   - `_confirmation_required` uses generic wording (installing agent CLIs needs
     confirmation).
   - Dry-run JSON entries gain `method` and `install_dir_on_path`. The change is
     additive, so keep `schema_version` 1.
8. **Help text.** The parser's help, description, and epilog say that installs come from
   a provider-declared install script **or npm package**, and gain the example
   `sase agent-cli install claude codex -n`. No flags are added.
9. **Exports.** Export `AgentCliInstallOption`, `InstallRoute`, and
   `describe_agent_cli_install` from `sase.agent_clis`.
10. **Docs.** Update `docs/agent_providers.md` (the install and inventory sections), the
    `docs/cli.md` table row, the `docs/getting_started.md` sentence, the
    `docs/plugins.md` provider-metadata table (an npm `package` now also drives
    installs), and the `llm_install_metadata` hookspec docstring.

Tests go in `tests/agent_clis/test_install.py`, `test_cli_install.py`, and
`test_detect.py`:

- The classifier returns the right route for script, npm, manual, and bundled, and
  performs no I/O.
- A ready npm entry has the right argv, `install_dir` from the prefix, and on-PATH
  status.
- Skips for npm missing, for an unwritable root, and for a nonexistent root whose
  writable ancestor is used.
- `--force` reinstall of an installed npm CLI.
- `progress_fn` is called in order for runnable entries only.
- An executed npm entry is found in `install_dir` and reported on or off PATH.
- npm rendering, the new JSON fields, and the prompt wording.
- The npm install hint text.

**Done when:** with npm on PATH, `sase agent-cli install qwen -n` previews
`npm install -g @qwen-code/qwen-code` with its target and PATH status; Muse output is
unchanged apart from the PATH marker; and Antigravity is still a manual skip.

## Phase `tui-install`: Install flow in the Updates tab

### A. Row model (`plugins_browser_rows.py`)

- `_build_agent_cli_row` calls the pure `describe_agent_cli_install(status)`. When
  `not status.installed and option.installable`, it adds the **shared** `install`
  capability. A CLI row can never carry both `install` and `mark_update`, because
  updating requires an installed CLI.
- For a not-installed CLI row:
  - `source = option.route.value`, so its badge reads `[npm]` / `[script]` / `[manual]`.
  - The haystack keeps `not installed`, the route, and the package, so `/not installed`,
    `/npm`, and package names still filter.
  - `_agent_cli_version_label` returns `latest v<latest>` when the latest version is
    known, otherwise `not installed`, mirroring not-installed plugin rows.

### B. Row rendering and marks (`plugins_browser_rendering.py`)

- `_row_text` appends a yellow `  ⚠ not on PATH` when the session's last result for that
  CLI is a successful install with `install_dir_on_path is False` and the row still
  reads as not installed. This is an in-memory dictionary lookup, with no I/O.
- `_marked_plugin_names` returns only `plugin:` keys that carry `install`. A new
  `_marked_cli_install_names` returns `cli:` keys that carry `install`.
  `_marked_cli_names` (updates) is unchanged.
- `_advance_mark_selection(capability, *, section)` moves to the next row with the same
  capability **in the same section**, wrapping within that section. Existing plugin and
  CLI-update marking behaves exactly as before. CLI marking no longer jumps into the
  plugin sections.

### C. Detail panel (`plugins_browser_agent_clis.py`)

- `_agent_cli_detail_panel` branches on `status.installed`. Installed CLIs keep today's
  layout. Missing CLIs render a new `_agent_cli_install_panel(status)` exactly as shown
  under **Visual design**, including the three call-to-action variants: installable,
  manual, and installed but not on PATH.
- The target comes from the declared `install_dir` / `install_dir_env`. Nothing on this
  path does I/O: the option is pure and results are read from memory.
- The history panel stays visible for missing CLIs, because it may hold an earlier
  install record.

### D. History and result lines

- In `plugins_browser_agent_clis_history.py`, history becomes operation-aware:
  - A successful `INSTALL` shows a green `↓` and `installed <new>`.
  - A failed install keeps `!`.
  - An `ADMIN_CENTER` run whose executed entries are installs gets the badge `i`.
  - The titles become `History` / `History · all agent CLIs`.
  - The empty text mentions `i` to install and `A` to update. The per-CLI empty line
    reads `No recorded installs or updates for <Name>.`
- `agent_cli_result_line` (`plugins_browser_agent_clis_actions.py`) becomes
  operation-aware:
  - Success: `X: installed 1.2.3`, plus ` — <note>` when there is a note.
  - Failure: `X: install failed — <reason>`.
  - Skip: `X: skipped — <reason>`.

### E. CLI install actions

Add a new module, `src/sase/ace/tui/modals/plugins_browser_agent_clis_install.py`, with
an `AgentCliInstallActionsMixin` that `PluginsBrowserPane` mixes in. Add pane-module
seams `_plan_agent_cli_installs` and `_execute_agent_cli_installs`, like the existing
`_plan_agent_cli_updates`, so tests can monkeypatch them.

- **Planning.** `_begin_agent_cli_install_plan(names)` runs
  `pane_module._plan_agent_cli_installs(names, offline=self._offline, status_fn=lambda **_: statuses)`
  in a thread worker stored in `_agent_cli_install_plan_worker` (group
  `agent-cli-install-plan`). Route it in `on_worker_state_changed` like the other plan
  workers. While the worker runs, the hints line starts with
  `↓ preparing install preview…`.
- **Preview routing.**
  - An unknown name gets an error toast.
  - With no runnable entries: `plan.cleanup()`, then a warning toast listing every skip
    reason.
  - Otherwise, open the modal.
- **Modal.** A pure builder, `agent_cli_install_variant(plan) -> PluginActionVariant`,
  makes one `PluginActionPreviewSection` per runnable entry:
  - Title: the display name.
  - Counts: the route label, plus `latest v…` when known.
  - Details: `script <url> · <N> bytes`, `sha256 <full digest>`, and
    `target <dir> · on PATH | not on PATH (SASE prints the export line)`.
  - Commands: `command_text(argv, env_overlay)` from `sase.agent_clis.cli_update`.

  The variant has `argv=()`, the summary `Installs N agent CLI(s), one at a time`,
  `skipped` lines for entries that cannot run, and the detail
  `Runs without a shell · never edits your shell startup files`. The modal title is
  `Install <Name>` or `Install N agent CLIs`.

- **Cancel** calls `plan.cleanup()`.
- **Execution** uses:

  ```python
  _submit_session_worker(
      "agent-cli-install",
      task,
      dedup_key="agent-cli-install",
      exclusive_scopes=("agent-cli-update",),
      duplicate_message="An agent CLI install or update is already running.",
  )
  ```

  - Sharing the `agent-cli-update` scope serializes installs with `A` and with the
    comprehensive update, since all of them mutate the npm global tree and provider
    binaries.
  - If submit returns `None`, clean up immediately.
  - The task:
    - calls `reporter.phase("Installing agent CLIs")`;
    - uses `progress_fn` to set `reporter.phase(f"Installing {name} ({i}/{n})")`;
    - runs with `run_fn=reporter.command_runner()` and
      `trigger=UpdateTrigger.ADMIN_CENTER`, calling `plan.cleanup()` in a `finally`;
    - logs a `Results` section with `agent_cli_result_line`;
    - succeeds only if nothing is `FAILED`.

- **Completion.**
  - Store the results in `_agent_cli_results`.
  - Clear the marks on the attempted CLI keys and force a detail repaint.
  - Toast `agent_cli_install_summary(results)` at the severities listed under **Visual
    design**.
  - Call `_start_load(force=False)` if the pane is still mounted, and re-read UI state
    after the await (tui_perf rule 4).
- **Registration.** Register every new session-worker submit site in
  `src/sase/ace/tui/_proc_producer_sites_updates.py`; `test_proc_producer_inventory`
  enforces this.

### F. `i` routing (`plugins_browser_install.py::action_install`)

- Return early while any install plan worker is running.
- **Nothing marked:** act on the highlighted row.
  - A plugin keeps today's behavior.
  - A missing installable CLI starts the CLI flow with `(name,)`.
  - A missing CLI that SASE cannot install gets a warning toast with `option.reason`.
  - An installed CLI gets `X is already installed.`
- **Only plugins marked:** today's path, unchanged.
- **Only CLIs marked:** the CLI flow, applied to the marked names.
- **Both marked:** one combined flow.
  - **Planning.** One worker plans both the CLI install plan and the plugin install-many
    preview. The plugin half becomes a skip reason when sase is not a `uv tool` install
    or nothing can be installed.
  - **Modal.** One modal shows the CLI sections plus a `Plugins` section: the exact `uv`
    command, the plugin list, and `sase's TUI restarts after the plugins install`.
  - **Execution.** One registered session proc installs the **agent CLIs first**, so the
    restart can never interrupt an installer, and then the plugins.
  - **Completion.** Apply the CLI results and clear all install marks. Then hand the
    plugin outcome to `_handle_code_update_completion`, with the CLI summary lines added
    before its message. The TUI restarts only if the plugins changed.
  - **Cleanup.** Clean up the CLI plan on every exit path.
- **`check_action("install")`** returns True for rows with `install` **or for any
  agent-CLI row**. This way `i` on a CLI row always reaches the pane, gives visible
  feedback, and never falls through to the app-level `i` notifications binding.

### G. Marks and hints (`plugins_browser_agent_clis_actions.py`, `plugins_browser_status.py`)

- `action_toggle_mark` works as-is through the shared `install` capability. Its advance
  step now passes the row's section. `_unmarkable_message` for a missing CLI that SASE
  cannot install reads `<Name> can't be installed by SASE — <reason>`.
- `_hints` shows `i install (N)` counting **all** install marks, or `i install` when the
  highlighted plugin or CLI can be installed.
- `_marked_work_line` splits the counts into plugin installs, CLI installs, and CLI
  updates, using the format shown under **Visual design**.

### H. Tests, docs, and goldens

- **Tests.** Add `tests/ace/tui/test_plugins_browser_pane_agent_clis_install.py`, and
  extend `test_plugins_browser_rows.py`, `test_plugins_browser_pane_marks.py`,
  `test_plugins_browser_pane_agent_clis_history.py`, and the proc-producer inventory.
  Cover:
  - capability, badge, and label derivation for every route;
  - `i` on each kind of row (installable, manual, installed) producing its modal or
    toast;
  - the modal sections, which must include the full digest and PATH status;
  - cleanup on cancel, on a dedup rejection, and after execution, using a real temp
    script file whose deletion is asserted;
  - the session proc receiving `ADMIN_CENTER` and progress phases;
  - marked bulk install through `Space` then `i`, with marks cleared after completion;
  - a mixed set installing CLIs before plugins, restarting only when plugins changed,
    and still running the CLIs when plugins are unavailable;
  - the not-on-PATH toast severity and row suffix;
  - install history glyphs and badges;
  - `i` on a CLI row never reaching the app's notifications action.
- **Docs.** Update `docs/configuration.md#updates-tab` and the `docs/ace.md` "Updates
  Tab" section with the install flow, bulk marks, mixed sets, and safety model.
- **Goldens.**
  - Fixture change: in `tests/ace/tui/_plugins_browser_pane_helpers.py`, the fixture
    `_agent_cli_statuses()` gives Qwen `install_manager="npm"` and
    `package="@qwen-code/qwen-code"`. It also gains a missing script CLI and a missing
    manual CLI.
  - Add scenarios in a new
    `tests/ace/tui/visual/test_ace_png_snapshots_config_center_agent_cli_install.py`:
    the install detail panel, the install preview (npm + script + skipped, using a
    stubbed plan with a fixed digest and `/home/dev` paths), marked CLI installs with
    the aggregate line, and the mixed preview.
  - Existing Updates goldens that show the missing Qwen row will change (badge and
    label).
  - Run `just fix-tui-screenshots -- <selectors>` (through `/sase_monitor` if it runs
    long), then inspect every creation and each update group before accepting.

## Phase `discover-bulk`: Discoverability and bulk-select accelerators

1. **Available scope** (`plugins_browser_rows.py`, `plugins_browser_layout.py`,
   `config_center_session.py`).
   - `SCOPE_ORDER` becomes `outdated, installed, available, all`, with the label
     `Available`.
   - A row is in the Available scope when it is not installed:
     `_row_in_scope(row, "available")` returns `not row.installed`.
   - Update `scope_counts` and the `UpdateScope` literal.
   - The session default stays `installed`.
   - An empty Available scope reads `Everything is installed.`
   - The live count in the scope strip is the passive nudge that something is missing.
2. **`*` mark all like this** (`plugins_browser_pane.py` BINDINGS, plus the actions and
   layout mixins).
   - Binding: `("asterisk", "toggle_mark_all", "Mark all")`. `check_action` returns
     `_can_mark_highlighted()`.
   - The action takes the highlighted row's markable capability, then collects the
     **visible** rows (current scope and filter) in the **same section** with that
     capability. Using the section keeps Built-in and Community plugins apart.
   - If all of them are marked, it unmarks them; otherwise it marks them all.
   - It patches the affected rows in place with `_refresh_row`, refreshes the hints, and
     toasts `Marked 4 agent CLIs to install` / `Unmarked 4 agent CLIs`.
   - A row that cannot be marked gets the same warning toast as `Space`.
   - The hints show `* mark all` next to `I/space mark`.
3. **Cross-scope hint** (`plugins_browser_status.py::_status_message`). Only when a
   non-empty filter matches nothing in the current scope, count the matches in the other
   scopes and say, for example:
   `Nothing in Installed matches "codex" — 1 match in Available ([ / ] to switch scope)`.
4. **Detail call to action.** The installable variant becomes
   `↓ i install now · Space mark · * mark all missing`.
5. **Docs.** Update `docs/configuration.md#updates-tab` (the scope list, the keymap
   including `*`), the `docs/ace.md` "Updates Tab" section, and a pointer from
   `docs/agent_providers.md` to installing from the TUI.

Tests:

- Scopes: the four-scope cycle order, counts, empty messages, and the cross-scope hint.
- Marks: `*` stays within one section and respects scope and filter; it unmarks when
  everything is marked; Built-in and Community plugins stay separate.
- Bindings: the pane's BINDINGS have no duplicate keys; `*` on the focused pane runs
  `toggle_mark_all` and does **not** open the saved-query picker; typing `*` in the
  filter input inserts text.

Goldens: every Updates golden changes because the scope strip gains **Available**.
Inspect each update group. Add the new goldens `config_center_updates_available_scope`
and `config_center_updates_mark_all_clis`.

## Verification (every phase)

- Run `sase tool run check`. Do not run `check-full` unless explicitly instructed.
- Any phase that changes rendered TUI output runs
  `just fix-tui-screenshots -- <selectors>`, through `/sase_monitor` when long, and
  inspects the retained report and every golden change before finishing.
- Optionally, for the TUI phases, run a live `sase screenshot` of the Updates tab (Admin
  Center `#`, then `7`) to sanity-check the real layout.
- All slow work (script fetch, npm probes, installs) runs off the event loop: in plan
  workers or the session proc. Nothing new stats or runs subprocesses on a render or
  keystroke path.

## Decisions and non-goals

- **No feature flag.** Each phase lands complete, coherent behavior:
  - `npm-installs` is a complete CLI feature.
  - `tui-install` installs singles, bulk sets, and mixed sets end to end.
  - `discover-bulk` is additive polish.
- **Install only.** The TUI offers install only for CLIs that are not installed.
  Reinstall/repair of installed CLIs, uninstall, switching Antigravity to its script
  installer, and moving agent-CLI management into sase-core are all out of scope.
