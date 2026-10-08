---
tier: epic
title: Plugin commands — sase listen as the first first-class command plugin
goal: "Plugins can mount top-level `sase <name>` commands through a metadata-declared
  `sase_commands` entry point. sase-listen uses it to ship `sase listen`, which behaves
  exactly like `sase-listen`, completes in bash/zsh/fish and the TUI `:` line, refreshes
  completion automatically on every plugin change, and is installed and managed from the
  Admin Center's Updates tab. Every moment a plugin adds or removes a command is
  announced clearly and consistently.

  "
decisions:
  compact_help:
    ask: Should compact `sase -h` list installed plugin commands in their own group?
    default: true
    why:
      Users look in `sase -h` first; the group only appears when a plugin command
      exists.
    answer: true
  memory_decision_record:
    ask:
      Add a decisions record that plugins may mount top-level commands, superseding
      listen's standalone stance?
    memory:
      - decisions
    requested: I agree with all of the requirements recommended in that research file.
    default: true
    answer: true
  memory_cli_rules:
    ask:
      Add a cli_rules.md note that plugin-mounted command subtrees are exempt from
      sase's CLI rules?
    memory:
      - cli_rules.md
    requested: I agree with all of the requirements recommended in that research file.
    default: true
    answer: true
phases:
  - id: mount
    title: Plugin command contract, discovery, and dispatch
    depends_on: []
    size: medium
    description: "mount: add the generic sase_commands contract, metadata-only discovery
      and validation, the pre-argparse dispatch fast path, helpful misses, the shared
      command chip, a fake-distribution test harness, and the hermetic test guard.

      "
  - id: listen-adapter
    title: sase-listen becomes a command plugin
    depends_on: []
    size: medium
    description: "listen-adapter: in the sase-listen repo, add the sase_command adapter
      and entry point, thread the program name through the CLI and hints, fix buildinfo
      and the feed-host remote command for plugin-only hosts, and rewrite the standalone
      stance.

      "
  - id: help-doctor
    title: Plugin commands in root help and sase doctor
    depends_on:
      - mount
    size: small
    description: "help-doctor: list plugin commands in sase -H (and sase -h per
      decision) with provenance and problem states, and add the plugins.commands doctor
      check.

      "
  - id: completion
    title: Plugin subtrees in completion with plugin-aware cache identity
    depends_on:
      - mount
    size: medium
    description: "completion: merge separately walked plugin parsers into a runtime
      completion spec, make the grammar and TUI spec caches key on the plugin command
      set and editable sources, and record omitted subtrees.

      "
  - id: lifecycle
    title: Command-aware plugin install, update, and uninstall
    depends_on:
      - mount
      - completion
    size: medium
    description: "lifecycle: fix the inventory groups, carry command names in the
      installed index, diff the command set around every plugin mutation, refresh
      completion in a fresh child process, and announce added or removed commands in CLI
      results and JSON.

      "
  - id: command-preview
    title: Pre-install command preview
    depends_on:
      - lifecycle
    size: small
    description: "command-preview: read an uninstalled plugin's declared sase_commands
      from its upstream pyproject.toml with a cache, expose it in plugin JSON and the
      install dry run, and flag collisions before install.

      "
  - id: updates-tab
    title: Commands in the Updates tab and plugin detail
    depends_on:
      - command-preview
    size: medium
    description: "updates-tab: render the command chip in Updates rows, the shared
      detail panel, install and uninstall confirmations, and the post-install toast
      receipt, then refresh the affected PNG goldens.

      "
  - id: listen-fast-start
    title: Lazy sase-listen command imports
    depends_on:
      - listen-adapter
    size: small
    description: "listen-fast-start: in the sase-listen repo, defer heavy imports into
      command handlers so building the parser and rendering help stay fast, guarded by
      an import-isolation test.

      "
  - id: research-macros
    title: Research macros prefer sase listen
    depends_on:
      - listen-adapter
    size: small
    description: "research-macros: in the sase-research-artifacts repo, make the audio
      macros select sase listen first, then sase-listen, then uvx sase-listen, and
      update docs and pinned-string tests.

      "
  - id: acceptance
    title: End-to-end acceptance, records, and docs
    depends_on:
      - help-doctor
      - updates-tab
      - listen-fast-start
      - research-macros
    size: medium
    description:
      "acceptance: verify parity, completion, freshness, Updates flows, and performance
      end to end with a real editable sase-listen, fix what breaks, and land the records
      and remaining docs."
proposed_by: bbugyi200.apollo.research.0n.linker.w0
decided_by: auto
create_time: 2026-10-08 15:26:15
status: wip
---

- **PROMPT:**
  [prompts/202610/plugin_commands.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/plugin_commands.md)

# Plan: Plugin commands, with `sase listen` as the first command plugin

## Context

The research report
`research:202610/sase_listen_plugin_commands/sase_listen_plugin_commands.md` is the
design basis. The user agreed with all of its adjusted requirements (R1–R9). This plan
adopts its recommended solution: a generic, metadata-declared `sase_commands`
plugin-command mechanism, with sase-listen as the first consumer. It adds a single
visual language so users always know when a plugin adds or removes a command.

Facts verified against the current tree, which the phases rely on:

- **Catalog and Updates tab.** `sase--plugin` is a GitHub repository _topic_. It is set
  on `sase-org/sase-listen`, and the catalog cache on this machine already lists
  `listen`. The Updates tab (`PluginsBrowserPane`) projects the catalog, so the row
  already appears under the Available/All scopes. No row-discovery work is needed (R1).
- **Closed command set.**
  - The dispatch order is in `src/sase/main/entry.py`: global options, legacy root
    rewrite, then the `bead` / `goal` / `completion candidates` / `completion ensure` /
    `run` fast paths, then `create_parser(only=parser_only_hint(argv))`.
  - `_COMMAND_REGISTRARS` (`src/sase/main/parser_registry.py`) is static.
  - An unknown root word builds the full parser (~0.9 s) and fails with argparse
    `invalid choice`.
- **Completion.**
  - `build_spec()` in `src/sase/completion/build.py` walks `create_parser()`. Its
    `_build_command` infers `default_child="list"` from the spec, and `kinds.py` assigns
    kinds through sase-specific name heuristics.
  - `runtime_identity()` and `source_fingerprint()` in
    `src/sase/completion/runtime_cache_identity.py` cover only `sase` and
    `sase-core-rs`.
  - The TUI `:` spec cache (`src/sase/completion/command_line_spec.py`) uses the same
    key.
  - Plugin install, update and uninstall never refresh completion. Only `sase update`
    does, through `_refresh_completions_in_child` in
    `src/sase/main/update_handler_completion.py`.
- **Plugins.**
  - `src/sase/plugins/inventory.py` `ENTRY_POINT_GROUPS` lists the retired
    `sase_xprompts` and omits `sase_macros` and `sase_pager_history`.
  - `InstalledInfo` (`installed.py`) keeps group names but drops entry-point names.
  - Plugin console scripts are never exposed on `PATH`. uv tool installs use `--with`
    without `--with-executables-from`.
- **sase-listen.**
  - `cli/app.py` `build_parser()` hard-codes `prog="sase-listen"`. Both CLIs use
    `add_subparsers(dest="command")`, so grafting listen into sase's argparse would
    break dispatch.
  - `cli/__init__.py` wraps `app.main` with stale-environment diagnostics (exit 3).
  - `src/sase_listen/cli.py` is dead code, shadowed by the package.
  - `buildinfo.upgrade_command()` and the feed-host SSH command both assume a standalone
    `sase-listen` tool.

No sase-core (Rust) work is needed. Discovering Python distributions, loading entry
points and handing off argv is host runtime glue under [[rust-core-required]]. The Rust
`CommandLineGrammar` is data-driven: its wire tolerates the merged spec, `kind` is
optional, and there is no `deny_unknown_fields`. There is no `sase-core-revision.txt`
bump.

No feature flag is needed. Each landed phase is complete on its own: a mounted command
without completion is not broken. No provider exists until a sase-listen release ships
the entry point, and an older sase ignores the unknown entry-point group.
`SASE_DISABLE_PLUGIN_COMMANDS` is a permanent operational switch, like every other
plugin group's, not a feature flag.

## Design principles

1. **The plugin parses; sase routes.** sase hands the untouched argv after the command
   word to the plugin's own `main(argv, prog)`. Parity with the standalone binary holds
   by construction, not by testing two parsers.
2. **sase owns everything around the command:** discovery, name reservation, collisions,
   help, completion, cache freshness, install lifecycle, and diagnostics.
3. **One visual language.** A plugin command is always rendered as the same _command
   chip_: the glyph `❯` followed by `sase <name>`. The chip uses the Updates accent
   (`#AF87FF`) in the TUI and bold magenta in the CLI. It appears everywhere: result
   panels, `sase plugin list` and `sase plugin show`, root help, Updates rows, detail,
   modals and toasts. Seeing `❯ sase listen` always means "a command a plugin gave you".
4. **Every moment of change is announced.** Before install (preview in the dry run and
   the confirm modal), at install (result panel and toast), and after install (help,
   completion, doctor). Removal is announced the same way.
5. **Never silently wrong.**
   - Collisions disable loudly and name both owners.
   - Broken plugins fail with a repair command.
   - Stale completion is impossible by construction, because the cache key includes the
     plugin command set.
6. **Zero cost for built-ins.**
   - A built-in command pays one dict lookup.
   - Warm completion never imports a plugin.
   - Only unknown root words pay the ~60–100 ms entry-point scan.

### Where users meet a plugin command

| Moment                 | CLI                                                                                                                                                       | TUI (Admin Center → Updates)                                                |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Browsing               | `sase plugin show listen`: `Commands ❯ sase listen` row (declared upstream when not installed)                                                            | Row chip `❯ listen`; same Commands row in detail                            |
| Before consent         | `sase plugin install listen -n`: "Adds command ❯ sase listen", plus collision warnings                                                                    | Install modal: "❯ Adds a new command: sase listen", plus collision warnings |
| Install result         | "Plugin Installed" panel callout: chip, summary, `Try it: sase listen --help`, completion-refresh line                                                    | Post-restart toast: `❯ sase listen  new command`                            |
| Daily use              | `sase -h` group (per decision) and `sase -H` footer; `<TAB>` completion                                                                                   | `:` command line completes `listen …`                                       |
| Uninstall              | "❯ sase listen command removed" in the result panel                                                                                                       | Uninstall modal "❯ Removes command: sase listen"; toast line                |
| Not installed / broken | `sase listen` → "provided by the listen plugin — `sase plugin install listen`"; load failures name the distribution and print `sase plugin update listen` | —                                                                           |
| Health                 | `sase doctor` `plugins.commands` check                                                                                                                    | —                                                                           |

Target CLI install result (exact styling may follow existing panels):

```text
╭──────────────────────────── Plugin Installed ────────────────────────────╮
│ ✓  sase-listen  0.1.2  (installed)                                       │
│   contributes  sase_commands                                             │
│                                                                          │
│ ❯ sase listen   new command                                              │
│   Turn Markdown into chaptered MP3 audio editions                        │
│   Try it:  sase listen --help                                            │
│                                                                          │
│ Installed listen in 9.4s · 86 dependencies resolved                      │
│ ✓ Shell completion refreshed (zsh) · already-open shells: exec $SHELL    │
│ Restart running sase agents to load the plugin.                          │
╰──────────────────────────────────────────────────────────────────────────╯
```

## The plugin command contract (frozen; shared by `mount` and `listen-adapter`)

A distribution declares one entry point per top-level command in the `sase_commands`
group. The entry-point **name** is the command name. The **value** is a module (or
`module:object`) exposing these duck-typed members:

| Member                                                | Required | Meaning                                                                                                                                                |
| ----------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `main(argv: Sequence[str], prog: str) -> int \| None` | yes      | Runs the command on the untouched argv after the command word. Returns an exit code (`None` means 0). `SystemExit` raised inside propagates unchanged. |
| `build_parser(prog: str) -> argparse.ArgumentParser`  | yes      | The full parser, used **only** for completion (never for dispatch). It has no side effects.                                                            |
| `SUMMARY: str`                                        | no       | One-line description for help and completion. When absent, sase uses the distribution's `Summary` metadata.                                            |
| `SASE_COMMAND_API: int`                               | no       | Contract version. Absent means 1. A value greater than sase supports is refused with "requires a newer sase — run `sase update`".                      |

The rules:

- **Cheap and sase-free.** The adapter module must be cheap to import and must not
  import `sase`. sase imports it only when the command runs, when help is printed, when
  doctor runs, or when a completion grammar is built on a cache miss.
- **Path completion.** An argparse action may set a public string attribute
  `action.sase_completion = "path"` or `"dir"` to request path completion. Otherwise
  only argparse `choices` produce candidates.
- **Name validation.** Names must match `^[a-z][a-z0-9-]{0,31}$`.
- **Reserved names.** These are reserved, and built-ins always win:
  - every `_COMMAND_REGISTRARS` key, including legacy aliases;
  - the legacy root words rewritten by `normalize_legacy_root_args`;
  - `help`.
- **Duplicate owners.** If two distributions claim one name, the command is disabled for
  both and every surface names both owners. sase never picks a winner by entry-point
  order.
- **Disable switches.** `SASE_DISABLE_PLUGINS` or `SASE_DISABLE_PLUGIN_COMMANDS` turns
  the group off (`is_plugin_disabled("commands")`).
- **Calling convention.** sase always passes `prog` as a keyword:
  `main(argv, prog=f"sase {name}")` and `build_parser(prog=f"sase {name}")`.
- **Process state at handoff.** sase sets `sys.argv = [f"sase {name}", *argv]` before
  calling `main`, so libraries that read `sys.argv` see a standalone-shaped invocation.
  The process exits with `main`'s return code.
- **Exemptions and limits.** Plugin subtrees are exempt from sase's CLI rules (R7). sase
  never post-processes them: no default-`list` delegation, no parser subclass, and no
  sase help formatter. A plugin may not patch or extend built-in commands.

## Phase: mount — Plugin command contract, discovery, and dispatch

Create a focused package `src/sase/plugin_commands/` with small modules, matching the
repo's lazy-facade style.

**Raw scan (`scan.py`).**

- Reads `importlib.metadata.entry_points(group="sase_commands")` only, without loading
  anything.
- Returns sorted records: name, value, distribution, version, location, and whether the
  install is editable. The location is the dist-info path; for editable installs it is
  the source root from `direct_url.json`.
- Has two modes: one honors the disable switches (used for dispatch and help), and one
  ignores them (used for lifecycle diffs and cache identity, which also records the
  switch state).
- Imports nothing from `sase.main.parser*`. The warm `completion ensure` contract
  (`tests/main/test_completion_ensure_contract.py`) forbids those imports, and the
  `completion` phase calls this module from that path.

**Validation (`registry.py`).**

- `discover_plugin_commands()` classifies each record as `mounted`, `shadowed` (reserved
  or built-in name), `conflict` (duplicate owners, with the other owners listed), or
  `invalid_name`.
- It returns a `PluginCommandSet` with `mounted` and `problems` views. The reserved set
  is computed from `_COMMAND_REGISTRARS` plus the legacy root words plus `help`.

**Adapter loading (`adapter.py`).**

- Loads one mounted command's adapter.
- Checks `SASE_COMMAND_API` and the required members.
- Resolves the summary from `SUMMARY`, or else the distribution `Summary`.
- Raises a typed `PluginCommandLoadError` that carries the distribution, version, and
  cause.

**Dispatch fast path (`dispatch.py`, wired in `src/sase/main/entry.py`).**

- **Placement.** Insert it after the `run` special case and before the
  `create_parser(...)` import.
- **Trigger.** It runs only when `root_command_index(argv[1:])` names a word that does
  not start with `-` and is not in `_COMMAND_REGISTRARS`.
- **Fall-through.** On no match it falls through untouched. The full-parser fallback
  output must stay byte-identical: `test_full_parser_fallback_output_is_unchanged`
  covers `[]`, `--help`, `-H` and `bogus`.
- **Pass-through.** Global options are consumed before the command word, so
  `sase -p listen …` works. Everything after the command word, including `-h`, `--` and
  same-spelled flags, belongs to the plugin.
- **Load failure** (exit 1). Print
  `sase: the 'listen' command from sase-listen 0.1.2 failed to load: <cause>`, then the
  repair line `sase plugin update listen`. Load failures are caught; exceptions raised
  by the plugin's own `main` are not, which preserves standalone parity.
- **Conflict** (exit 1). Name every owner and suggest `sase plugin uninstall <name>` for
  one of them.

**Command chip (`chip.py`).** This is the single renderer for the `❯ sase <name>` chip.

- It has a plain-text form and a Rich form, with an optional state (`new`, `removed`,
  `problem`). Rich is imported lazily, so dispatch never pays for it.
- Every later phase uses it: help, CLI result panels, `sase plugin list` and `show`, and
  the Updates tab.

**Helpful misses (`hints.py`).** These apply when the word is neither built-in nor
mounted, and they use only the offline catalog cache
(`sase_subdir("plugins")/catalog_cache.json`):

- **Catalogued, not installed** (exit 2):
  `sase: 'listen' is provided by the listen plugin (sase-org/sase-listen).` and then
  `Install it: sase plugin install listen   (or the Updates tab in sase's Admin Center)`.
- **Installed, but too old to declare the entry point** (exit 2):
  `sase-listen 0.1.1 is installed but predates 'sase listen' — run sase plugin update listen`.
- **Disabled by a switch** (exit 2): name the environment variable that disabled it.
- **No catalog match:** fall through to the existing argparse error, unchanged.
- **Color:** rich color only when stderr supports color (`stream_supports_color`).
  Otherwise print plain text and never import rich.

**Hermetic tests and the shared harness.**

- **Hermetic guard.** Add `SASE_DISABLE_PLUGIN_COMMANDS=1` to the session-scoped autouse
  environment in `tests/conftest.py`. Built-in help, snapshot, kind-coverage and
  narrowing tests then stay independent of whatever the developer's venv has installed.
  Feature tests unset it explicitly.
- **Harness.** Add a reusable fake-distribution helper. It writes a temporary
  `<dist>.dist-info` (`METADATA`, `entry_points.txt`, optional `direct_url.json`) plus
  an adapter package, and prepends it to `sys.path` (and to `PYTHONPATH` for subprocess
  tests). Later phases reuse it.
- **Coverage:**
  - dispatch, exit codes and `sys.argv` shape;
  - `-p` global options, `-h` / `--` pass-through, and `SystemExit` propagation;
  - collisions with built-ins and legacy aliases;
  - duplicate owners, invalid names and the disable switches;
  - API mismatch, missing members and import failure;
  - each helpful-miss branch, with a seeded catalog cache in a temporary `SASE_HOME`;
  - built-in dispatch does not import `importlib.metadata` through this path, and the
    narrowing and fallback tests stay green.

**Docs.**

- In `docs/plugins.md`, add a **Command plugins** section covering:
  - the contract table and an adapter example;
  - name rules, collisions and the disable switch;
  - the `sase_completion` attribute;
  - the CLI-rules exemption and the in-process execution model.
- Add a `sase_commands` row to the group table and a `SASE_DISABLE_PLUGIN_COMMANDS` row
  to the disable-switch table.
- Mention plugin commands in `docs/cli.md`.

## Phase: listen-adapter — sase-listen becomes a command plugin

Work in the linked `sase-listen` repo (`sase repo open sase-listen`), and read its
`AGENTS.md` first. sase-listen must still never import `sase` or `sase_core_rs`, and it
keeps its standalone `sase-listen` console script (R4).

1. **Adapter and entry point.**
   - Add `[project.entry-points."sase_commands"] listen = "sase_listen.sase_command"` to
     `pyproject.toml`.
   - Add `src/sase_listen/sase_command.py` with `SASE_COMMAND_API = 1` and
     `SUMMARY = "Turn Markdown into chaptered MP3 audio editions"`.
   - `build_parser(prog)` delegates to `cli.app.build_parser(prog=prog)`.
   - `main(argv, prog)` delegates to the console-script wrapper
     `sase_listen.cli.main(argv, prog=prog)`, so the stale-environment guard (exit 3
     with a repair hint) also covers `sase listen`.
   - The adapter imports only stdlib at module top.
2. **Program name threading.**
   - The signatures become `cli.main(argv=None, *, prog="sase-listen")`,
     `app.main(argv=None, *, prog="sase-listen")` and
     `app.build_parser(prog="sase-listen")`.
   - Command modules receive `prog` explicitly when they register, so their epilogs read
     `Example: sase listen render …`.
   - `app.main` records the display name in a new leaf module
     `sase_listen/invocation.py` (`display_prog()`, `set_display_prog()`, and a
     `command(*parts)` helper). `build_parser` stays side-effect free.
3. **Hints follow the invoked name (R6).**
   - **What changes.** Route every user-facing command suggestion through `invocation`:
     usage and status error prefixes, "run this next" hints, the stale-env message, the
     starter-config comment, and `PENDING_HINT`. There are about 80 `sase-listen <cmd>`
     strings across about 20 files.
   - **What stays.** Keep product and distribution names unchanged:
     - "upgrade sase-listen on {host}", `--version` output, dist metadata, uv/pipx tool
       names, XDG paths, the audited-read reason string;
     - the wire strings: the feed-host remote command and the
       `"sase-listen" in item and "feed host" in item` coupling in `pipeline.py`.
   - **The agent guide.** `data/guide.md` is printed to agents by `guide`, so make it
     use a placeholder token that is substituted at print time.
4. **`buildinfo.upgrade_command()`.**
   - Parse `sys.prefix/uv-receipt.toml` with `tomllib`. When the first requirement is
     `sase`, return `sase plugin update listen`. Otherwise keep the existing branches.
   - Listen's own doctor upgrade hints follow the new output.
   - Update `test_buildinfo.py` for both modes.
5. **Feed host on plugin-only hosts.**
   - Keep `REMOTE_CALL_ENV`. The remote shell prefers the standalone binary, then the
     plugin:
     `if command -v sase-listen …; then exec sase-listen ARGS; elif command -v sase …; then exec sase listen ARGS; else echo 'sase-listen: not installed' >&2; exit 127; fi`.
   - Tighten "too old to receive" detection so it requires `invalid choice: 'receive'`.
     Any `invalid choice` currently matches, so a missing plugin would be misreported.
   - Map exit 127 to a new actionable error:
     `install it there: sase plugin install listen (or uv tool install sase-listen)`.
   - Update the fake-ssh harness in `test_feedhost.py` to execute both forms.
6. **Cleanup.** Delete the dead `src/sase_listen/cli.py`.
7. **Docs and stance.**
   - Rewrite the standalone stance in `AGENTS.md`, `CONTRIBUTING.md`,
     `docs/background.md`, `docs/getting-started.md`, `docs/sase-integration.md`,
     `README.md`, `docs/index.md`, `docs/multi-machine.md`, `docs/troubleshooting.md`
     and `docs/cli.md`.
   - The new stance: "never imports sase; the only `sase_*` entry point is
     `sase_commands`; `sase--plugin` topic set". Document the two install paths:
     - `sase plugin install listen` gives `sase listen`;
     - `uv tool install sase-listen` gives the standalone binary for people who don't
       use sase.
   - Note in `docs/background.md` that the plan decision "standalone tool, not a sase
     plugin" (`plan:202610/sase_listen.md`) is superseded. Follow the repo's changelog
     convention.
8. **Tests (no sase needed).**
   - Contract: the entry point is declared, the adapter has the right shape and imports
     only stdlib, and `main(["--help"], prog="sase listen")` prints
     `usage: sase listen`.
   - Parity between `prog="sase-listen"` and `prog="sase listen"`, normalizing only the
     program name. Cover:
     - bare invocation (exit 2) and nested `-h` for all 12 commands;
     - `--version`, unknown flags, and `config --json`;
     - a lint error and an offline tone-engine render;
     - `BrokenPipeError` handling.
   - `sase_completion = "path"` is set on these path-like slots:
     - `render source`, `script source`, `lint script`;
     - `--cover`, `-o/--output`, `-H/--html`, `--source`, `audition --text`.
9. **Verify** with `sase tool run check` inside that checkout. Use Conventional Commit
   style, because release-please cuts the release.

## Phase: help-doctor — Plugin commands in root help and sase doctor

**Full help (`sase -H`).**

- Extend `FullRootHelpAction` (`src/sase/main/parser_root_help.py`) to print a **Plugin
  commands** footer after argparse's help, in both plain and colored forms.
- Each row shows the command chip, the summary, and the distribution and version.
- Problem rows (`shadowed`, `conflict`, `invalid_name`, load failure) render with `⚠`
  and a one-line reason that points at `sase doctor`.
- A closing line reads:
  `Manage plugins with sase plugin list or the Updates tab in sase's Admin Center.`
- The footer is omitted when no plugin commands exist.

> [!decision] compact_help Add a **Plugin commands** group to compact `sase -h` (both
> plain and colored formats) between "Common commands" and "Examples". Each row shows
> the name, the summary, and a dim `· <distribution>`. The group appears only when at
> least one command is mounted; problem states stay in `-H` and doctor.

> [!decision] compact_help = no Leave compact `sase -h` unchanged; plugin commands
> appear only in the `sase -H` footer.

**Doctor.**

- Add a `plugins.commands` check to `src/sase/doctor/checks_plugins.py`, using
  `_check_plugins_resources` as the template:
  - **OK** lists the mounted commands with their owners.
  - **WARN** covers shadowed and invalid names.
  - **ERROR** covers conflicts, load failures, API mismatches and missing members.
  - Every non-OK result names the distribution and carries a next step
    (`sase plugin update <name>`, `sase plugin uninstall <name>`).
- A `deep=True` variant also calls `build_parser` to catch parser-construction failures.

**Tests.** Use the fake-distribution harness. Root-help tests stay hermetic through the
autouse switch. `-H` output must still contain every argparse choice.

## Phase: completion — Plugin subtrees in completion with plugin-aware cache identity

**Runtime spec.**

- Keep `build_spec()` builtin-only. That keeps hermetic the snapshot gate
  (`tests/completion/snapshots/cli_spec.json`, `tools/sync_completion_spec`,
  `test_snapshot.py`), `test_kind_coverage`, the 60-character summary test, and the
  run-policy contract.
- Add `build_runtime_spec()`, which returns the builtin spec plus one root child per
  mounted plugin command, and the list of omitted commands.
- Each child comes from `adapter.build_parser(prog=f"sase {name}")`, walked with a
  plugin mode of the `_build_command` walker:
  - **no** `default_child="list"` inference;
  - kinds come only from the public `sase_completion` attribute (`path` →
    `ValueKind.PATH`, `dir` → `ValueKind.DIR`) and argparse `choices`;
  - **none** of the `NAME_TABLE`, `PATH_OVERRIDES` or hint heuristics apply;
  - the default run policy applies.
- The root child's summary is `"<summary> · <distribution>"`.
- If loading the adapter or building its parser fails, skip that subtree, log it, and
  return it as an omission.

**Consumers that switch to the runtime spec:**

- `sase completion spec` (both `-d` and the structural view);
- `sase completion bash|zsh|fish`;
- `install_scripts.expected_scripts_for_shells` (so drift detection compares like with
  like);
- runtime grammar generation behind `sase completion ensure`;
- the TUI command-line spec subprocess.

The snapshot tool stays on `build_spec()`.

**Cache identity.**

- In `runtime_identity()`, add a `plugin_commands` record: sorted
  `{name, value, distribution, version, location}` from the disable-ignoring scan, plus
  the disable-switch state.
- In `source_fingerprint()`, add a stat walk (mtime and size of `*.py`) of each
  _editable_ provider's top-level package directory. Resolve it from `direct_url.json`
  and the entry-point module root, without importing the plugin.
- Bump `CACHE_FORMAT_REVISION` in `runtime_cache_models.py`.
- The shell grammar cache and the TUI spec cache share this key, so any change to the
  plugin set gives new shells and new TUI sessions a fresh grammar. That holds for every
  route: managed installs, `sase update`, and hand-run `uv` changes.
- Warm `completion ensure` must keep its forbidden-import list and CPU budget, and must
  never import a plugin.

**Omissions.**

- Record omitted subtrees in the grammar manifest, so a broken plugin is not re-imported
  on every new shell (the key is unchanged, so the cache hits).
- Report them as WARN, with a next step, in the completion doctor checks
  (`completion_check_specs`).

**Long-lived TUIs.** When the `:` command line opens, recompute the spec key in its
existing thread worker (`src/sase/ace/tui/command_line/grammar.py`). Reload the grammar
only if the key changed. Read the `tui_perf.md` memory note first; nothing may block the
event loop.

**Docs.** In `docs/completion.md`, document:

- that plugin subtrees are merged;
- the freshness model: new shells and new TUI sessions update automatically;
  already-open shells keep their loaded functions until `exec $SHELL`; exported files
  from `sase completion zsh > file` stay unmanaged;
- the `sase_completion` attribute.

**Tests.**

- The runtime spec includes the fake plugin subtree, with choices and path kinds and no
  `default_child`.
- The builtin snapshot is unchanged with the fake plugin installed.
- The identity key changes on install, uninstall, version change, disable switch, and an
  editable source edit.
- A warm ensure with the plugin installed imports neither the plugin nor the parser
  modules.
- An omission is recorded once and not retried.
- The emitters for bash, zsh and fish include the subtree. Extend the existing smoke
  tests.

## Phase: lifecycle — Command-aware plugin install, update, and uninstall

**Inventory.**

- In `src/sase/plugins/inventory.py`, add `sase_commands` (a provider group: inspected,
  never loaded), `sase_macros` and `sase_pager_history` to `ENTRY_POINT_GROUPS`.
- Keep the retired `sase_xprompts` only as a recognition signal (via
  `legacy_xprompt_syntax.RETIRED_PLUGIN_GROUP`), so legacy-only distributions are still
  detected. Do not display it as a capability.
- Update the test string allowlist if needed, and fix the group count in
  `docs/plugins.md`.
- Add `InstalledInfo.commands: tuple[str, ...]`, built in `build_installed_index`, and
  `installed.commands` in `plugin_entry_json`.

**Command chip.** Use the `mount` phase's chip helper (`sase.plugin_commands.chip`)
everywhere. In `sase plugin list`, show the chip in the groups/capabilities cell for
plugins that provide commands.

**Command diff.**

- `sase.plugin_commands.snapshot` captures `{name → (distribution, version, location)}`
  through the disable-ignoring scan.
- `diff_command_snapshots(before, after)` returns
  `CommandChanges(added, removed, updated)`.
- Take _before_ ahead of every uv mutation. Take _after_ right after it, calling
  `importlib.invalidate_caches()` first; the existing post-install `_installed_groups`
  index already proves in-process metadata reads work here.

**Shared post-change effects.**

- Move `_refresh_completions_in_child`, `CompletionRefreshReport` and
  `render_completion_refresh` into a public `src/sase/completion/refresh_child.py`.
  `sase update` keeps identical behavior through it.
- Add `src/sase/plugins/post_change.py`, which takes the uv install and the _before_
  snapshot and returns `PluginChangeEffects(command_changes, completion_refresh)`.
- It refreshes completion in a **fresh child process** (the tool's own
  `sase completion refresh --json`) whenever the command snapshot changed. It is
  best-effort, like `sase update`: a failure never fails the mutation and is reported
  with a retry command (`sase completion refresh`).
- Call it after every real change, before `restart_after_plugin_change`, from:
  - `cli_install`, `cli_update` and `cli_uninstall` (this also covers TUI single
    installs, which run that CLI as a durable proc);
  - the required-plugins gate command in `_required_gate_spec.py`, keeping its persisted
    preview byte-stable;
  - the TUI's in-process batch and combined install workers.
- Carry the effects in `InstallManyOutcome` and in the ops-result payload, so the
  `updates-tab` phase can render them.

**Results and JSON.**

- The install, update and uninstall success panels (`render_results.py`) gain the
  command callout shown in the target mock:
  - **new:** chip, summary, and `Try it: sase <name> --help`;
  - **removed:** chip and "command removed";
  - **updated:** for added or removed names only.
- They also gain the completion line: refreshed shells, or the failure with its retry
  command, plus the already-open-shell tip.
- JSON payloads gain `command_changes` (`added` / `removed` / `updated`, each
  `{name, distribution, version}`) and `completion_refresh`. Follow the repo's schema
  version convention for additive fields.

**Tests.** Use injected uv runners and the fake-distribution harness:

- the before/after diffs for install, update and uninstall, including a batch with mixed
  plugins;
- the refresh runs only when the command set changes, and a refresh failure is
  non-fatal;
- the required gate's output stays byte-stable;
- panel text and JSON shapes;
- `sase update` refresh behavior is unchanged.

## Phase: command-preview — Pre-install command preview

**Reading the declaration (`src/sase/plugins/declared_commands.py`).**

- For an uninstalled catalog entry, fetch the repository's `pyproject.toml` with the
  existing `gh` helper (`gh api repos/<full_name>/contents/pyproject.toml` with the raw
  media type). Parse it with `tomllib`.
- Read `project.entry-points.sase_commands` and return
  `DeclaredCommands(status: declared | none | unknown, names, source)`.
- The status is `unknown` when there is no pyproject, `entry-points` is dynamic, `gh` is
  missing, the fetch fails, or the mode is offline. Unknown renders nothing, so sase
  never makes a false claim.

**Caching.** Store results in `sase_subdir("plugins")/declared_commands_cache.json`,
keyed by `full_name`. An entry is invalidated when the catalog's `updated_at` changes or
after 7 days.

**Problems before consent.** Validate declared names with the `mount` rules. Flag
collisions before install:

- with reserved or built-in names: "would be shadowed";
- with currently installed command owners: "would conflict with <dist> — both would be
  disabled".

**Where the preview surfaces.**

- `plugin_entry_json` gains `declared_commands` (status, names, source, problems), used
  by `sase plugin show -j` and `sase plugin list -j`.
- The `sase plugin install -n` dry-run panel shows "Adds command ❯ sase listen" and any
  collision warnings.
- The TUI install preview worker (`plan_install_preview`) attaches the preview to its
  plans.

An installed plugin's authoritative `InstalledInfo.commands` always wins over the
preview.

**Tests** use a fake `gh` runner: declared, none, dynamic, missing file, parse error,
cache hit and miss, invalidation on `updated_at`, collision flags, and dry-run output.

## Phase: updates-tab — Commands in the Updates tab and plugin detail

**Prerequisites and shared surfaces.**

- Read the `tui.md`, `tui_perf.md` and `tui_screenshot.md` memory notes first.
- **Detail panel.** `build_detail_panel` (`src/sase/plugins/render_catalog.py`) is
  shared by `sase plugin show` and the TUI detail. Add a **Commands** row to
  `_detail_rows`:
  - **installed:** one chip per command, with its summary or its problem state;
  - **not installed:** the declared preview, in dim, as "added on install · declared in
    pyproject.toml";
  - **unknown:** the row is omitted.
- **Row chip.** In `_row_text` (`plugins_browser_rendering.py`), plugin rows show the
  chip after the version label. It is bold accent when installed, and dim when it comes
  from a cached preview. Never fetch per row; the detail view fetches the preview lazily
  in a worker, in its own group, like the latest-version workers.

**Confirmations.**

- **Single install modal:** a leading detail line "❯ Adds a new command: sase listen",
  plus collision warnings in yellow.
- **Batch install modal:** per-item suffixes such as "(adds ❯ sase listen)".
- **Uninstall modal:** "❯ Removes command: sase listen".
- **Update modal:** a commands line only when the upstream preview differs from the
  installed set.

**Toast after restart.**

- Add per-plugin `commands_added` and `commands_removed` to `UpdateVersionTransition`.
  Bump the receipt `FORMAT_VERSION` from 3 to 4, and keep decoding v3 receipts (which
  carry no commands).
- Build the fields from the lifecycle effects for both single installs (through
  `update_receipt` in the CLI payload) and batch installs.
- In `post_update_toast.py`, render `❯ sase listen  new command` (green) and
  `❯ sase <name>  removed` (yellow) under the plugin's line.
- Leave the `plugins.required` gate preview unchanged.

**Goldens and tests.**

- Add PNG goldens for:
  - an installed plugin with a command (detail and row chip);
  - an install modal with the command preview and a collision warning;
  - a toast with a new command.
- Refresh existing plugin goldens that the Commands row changes, using
  `just fix-tui-screenshots` with selectors (through `/sase_monitor` if long), and
  inspect every changed golden.
- Pane tests cover the lazy preview worker and the modal lines.

## Phase: listen-fast-start — Lazy sase-listen command imports

Work in the linked `sase-listen` repo.

**The change.** Move the heavy imports into command handler bodies or into handler-only
modules, keeping each command's `register` (parser construction) lightweight. The heavy
modules are numpy, PIL, mutagen, httpx, lxml, trafilatura, pdfminer, google-genai and
`pipeline`. `build_parser()` and `--help` must not import them.

**What must not change.**

- The stale-environment guard must still map dependency `ImportError`s to exit 3. It
  wraps both the import and `app.main`.
- `--help` output for every command must be byte-identical before and after (golden
  comparison test).

**Guard and measurement.**

- Add a subprocess import-isolation test: after `build_parser("sase listen")` and after
  `--help`, none of those modules are in `sys.modules`.
- Report before/after timings for `sase-listen --help` in the phase notes.
- Verify with `sase tool run check`.

## Phase: research-macros — Research macros prefer sase listen

Work in the linked `sase-research-artifacts` repo
(`sase repo open sase-research-artifacts`), and read its `AGENTS.md`.

**CLI selection.** In `xprompts/research_audio.md`, step "Choose the CLI" becomes:

1. use `sase listen` when `sase listen render --help` advertises `--generated-cover`;
2. else use `sase-listen` with the same probe;
3. else use `uvx sase-listen`.

The step keeps the "do not silently drop the option" and `audio.ok=false` handoff rules.
Later steps refer to the chosen CLI through one placeholder name, for example
`<listen> guide --edition …`, and render with `sase tool run -- <listen> render …`.

**The linker card.** Update the recovery hint in `xprompts/research_swarm.md` the same
way. `ls` is still a stub in sase-listen, so record a `PROPOSED FOLLOW-UP:` on this
phase's bead rather than claiming that the hint works.

**Docs and tests.**

- Update `docs/macros.md` and `README.md` to recommend `sase plugin install listen`,
  with `uv tool install sase-listen` as the alternative.
- Update the pinned strings in `tests/test_macro_loading.py`.
- Verify with `sase tool run check`.

## Phase: acceptance — End-to-end acceptance, records, and docs

**Setup.** Install the linked sase-listen checkout editable into this workspace's sase
dev venv (`uv pip install -e <path printed by sase repo open sase-listen>`) for the
duration of the phase. Uninstall it before finishing. Never mutate the user's real uv
tool environment or real shell rc files: use temporary `HOME`, `SASE_HOME` and XDG
directories for every freshness and install check. Unset `SASE_DISABLE_PLUGIN_COMMANDS`
in the shells you use for these checks.

**Checks to perform:**

| Area        | Evidence                                                                                                                                                                                                                                                                                                                                                             |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Parity      | `sase-listen` vs `sase listen`, normalizing only the program name: bare (exit 2), nested `-h`, `--version`, unknown flags, `config --json`, a lint error, an offline tone-engine render; stdout, stderr and exit codes; broken pipe; Ctrl-C (exit 130 during render). No paid TTS.                                                                                   |
| Dispatch    | `sase -p <project> listen --help`; `--` pass-through; the helpful miss with listen uninstalled and a seeded catalog cache; the "too old" hint against a fake old distribution.                                                                                                                                                                                       |
| Completion  | `sase completion zsh                                                                                                                                                                                                                                                                                                                                                 | bash | fish`contain the listen subtree. Real-shell probes through the existing bash/zsh smoke harnesses:`sase listen <TAB>`, `render --edition <TAB>`, `--progress <TAB>`, path slots. The TUI command-line spec contains `listen`. |
| Freshness   | With warm caches: install, uninstall, reinstall, a version change, the disable switch, and an editable source touch. Each one changes the key, so a new `sase completion ensure zsh` and the TUI spec pick up the change. The refresh child runs, and a forced refresh failure is reported but not fatal.                                                            |
| Updates     | `sase plugin show listen` shows the Commands row, both declared and installed. Headless pane tests show the row chip, modal lines and toast. If a disposable uv tool environment can be built in temporary directories (`UV_TOOL_DIR`, `UV_TOOL_BIN_DIR`), also exercise `sase plugin uninstall listen` and a reinstall there. If not, record why on the phase bead. |
| Performance | Built-ins and warm `ensure` never import `sase_listen`. `sase listen --help` overhead stays within about 100 ms of standalone.                                                                                                                                                                                                                                       |

Fix the small defects you find. Record anything larger as `PROPOSED FOLLOW-UP:` notes on
this phase's bead.

**Docs.**

- In `docs/plugins.md` "Available Plugin Packages", add `sase-listen` and
  `sase-research-artifacts`, and note that `sase-nvim` carries no catalog topic.
- Add a short "migrating from the standalone `sase-listen`" note:
  `sase plugin install listen`, then `sase listen doctor`, then
  `uv tool uninstall sase-listen`. Migrate the feed host last, after the feed-host
  fallback is released.

> [!decision] memory_decision_record Use `/sase_memory_write` to add a `decisions`
> strand, "Plugins May Mount Top-Level Commands", with keyword slug `plugin-commands`.
> It records:
>
> - **Claim:** a metadata-declared `sase_commands` entry point lets the plugin own its
>   subtree and parse it; sase owns discovery, collisions, help, completion and
>   lifecycle.
> - **Why**, and the rejected alternatives: grafting into argparse; hard-coding
>   `listen`; PATH exposure through `--with-executables-from`; doing it in sase-core.
> - **Cost:** environment weight and resolver coupling for opt-in command plugins, and
>   the entry-point scan on unknown words.
> - **Reopens when:** a non-Python frontend needs the mount table, or plugins need to
>   extend built-ins.
>
> It states that it supersedes the "standalone tool, not a sase plugin" decision in
> `plan:202610/sase_listen.md`, and links [[rust-core-required]]. Run `sase memory init`
> afterwards.

> [!decision] memory_cli_rules Use `/sase_memory_write` to add one line to
> `cli_rules.md`: these rules govern sase's built-in commands; plugin-mounted subtrees
> (`sase_commands`) are recommended to follow them but are exempt, and sase never
> post-processes them (no default-`list` delegation). Run `sase memory init` afterwards.

Finish with `sase tool run check` in this repo.

## Rollout (manual, after the releases ship)

On each machine (athena, apollo, the Mac):

1. Install from the Updates tab, or run `sase plugin install listen`.
2. Run `sase listen doctor`.
3. Run `uv tool uninstall sase-listen`.

Retire the feed host's standalone tool last, and only after the sase-listen release that
contains the feed-host fallback. The open Mac rollout bead (`sase-1gc`) is the natural
place to track the Mac step. Plugin executables stay off `PATH` (no
`--with-executables-from`), because `sase listen` covers that need.
