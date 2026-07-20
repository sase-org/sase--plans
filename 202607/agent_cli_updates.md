---
tier: epic
title: Agent CLI updates across the SASE CLI and TUI
goal: 'SASE can check and update every installed supported agent CLI (Claude Code,
  Codex, OpenCode, Qwen Code, Antigravity) through one shared, provider-metadata-driven
  service layer, exposed as a new `sase agent-cli` command group and as a redesigned
  Admin Center Updates tab split into Core / Plugins / Agent CLIs sub-tabs with a
  new `A` keymap.

  '
phases:
- id: core
  title: Agent CLI update service layer and provider metadata
  depends_on: []
  size: small
  description: '''Agent CLI update service layer and provider metadata'' section:
    enrich the sase_llm install-metadata hook with update/docs fields, add the src/sase/agent_clis/
    package (detection, latest-version oracle, plan/execute operations, typed runner),
    and consolidate doctor setup hints onto the enriched metadata.'
- id: cli
  title: sase agent-cli command group
  depends_on:
  - core
  size: small
  description: '''sase agent-cli command group'' section: add the `sase agent-cli`
    group with `list` and `update` subcommands (Rich tables, JSON envelopes, dry-run
    previews, docs-URL-bearing error output) wired through the shared service layer.'
- id: tui
  title: Updates tab split and Agent CLIs sub-tab
  depends_on:
  - core
  size: small
  description: '''Updates tab split and Agent CLIs sub-tab'' section: split the Admin
    Center Updates tab into Core / Plugins / Agent CLIs sub-tabs, add the new Agent
    CLIs master/detail browser with granular update marks, bind the pane-wide `A`
    (update agent CLIs) keymap, and refresh docs, hints, help modal, and visual snapshots.'
- id: smoke
  title: End-to-end agent CLI update exercises
  depends_on:
  - cli
  - tui
  size: small
  description: '''End-to-end agent CLI update exercises'' section: exercise the new
    CLI surface (list, JSON, dry-run, error paths) and the TUI sub-tabs read-only
    against the real environment; no real update executions.'
  model: haiku
create_time: 2026-07-19 19:42:44
status: done
bead_id: sase-7s
---

# Plan: Agent CLI updates across the SASE CLI and TUI

## Context and product goal

SASE can already update itself and its plugins (`sase update`, `sase plugin update`, and the Admin Center Updates tab),
but the agent CLIs it drives — Claude Code, Codex CLI, OpenCode, Qwen Code, and the Antigravity CLI — must today be
updated by hand, each with its own vendor-specific mechanism. This epic adds first-class "update my agent CLIs" support:

- **Shared core**: one service layer, driven by provider-declared metadata, used identically by the CLI and the TUI. No
  presentation layer reimplements update logic.
- **CLI**: a new `sase agent-cli` command group (`list`, `update`).
- **TUI**: the Admin Center Updates tab splits into three sub-tabs — **Core**, **Plugins**, and **Agent CLIs** — and
  gains a pane-wide `A` (update agent CLIs) keymap. The new Agent CLIs sub-tab offers granular, mark-based control
  mirroring the Plugins install-marks UX.
- **Honest reliability**: SASE only automates update paths it can positively identify (npm-managed installs and CLIs
  with a native self-update command). Anything ambiguous or unsupported degrades to a clear message carrying the
  vendor's **canonical install/update docs URL** and the exact suggested command — never a guess, never `curl | bash`.

The existing `u` keymap (full SASE core + plugins update) keeps its exact behavior; only its documentation is clarified
so it is not confused with the new agent-CLI flow.

## Design overview

### Where the logic lives (and the Rust boundary)

All existing update machinery — `src/sase/updates/`, `src/sase/uv_tool/`, `src/sase/plugins/operations.py`,
`src/sase/dev_update/` — is pure Python in this repo, and the Rust core contains no update orchestration. Agent CLI
updating is host-environment tooling of the same kind, so it follows that precedent: a new pure-Python package
`src/sase/agent_clis/`, no `sase-core` changes.

### Provider metadata is the single source of truth

The pluggy `sase_llm` registry (`src/sase/llm_provider/registry.py`, hookspec in `_hookspec.py`) already exposes
per-provider `llm_autodetect_cli_name` and `llm_install_metadata` (`manager` / `package` / `scope`; today implemented by
claude, codex, qwen, fakey — **opencode and agy are gaps this epic fills**). Rather than inventing a parallel registry,
the existing `llm_install_metadata` hook is enriched with optional, tolerantly-parsed fields (the registry's
`_install_metadata()` normalizer and the hook docstring are updated accordingly; absent fields keep today's behavior for
third-party providers):

- `display_name` — human tool name ("Claude Code", "Codex CLI", …).
- `docs_url` — the vendor's canonical install/update documentation page. Required for every built-in provider; surfaced
  in every error message, skip reason, and the new sub-tab.
- `self_update_argv` — extra argv after the binary for a native self-update command (e.g. `["update"]` for Claude Code,
  `["upgrade"]` for OpenCode), when one exists.
- `version_argv` (default `["--version"]`) and an optional version-parse regex override.
- `latest_version_package` — an npm package to use as a latest-version oracle even when the local install is not
  npm-managed (e.g. `@anthropic-ai/claude-code` tracks native installs).

Expected per-provider values (the implementer must verify each command and docs URL against the vendor's current
documentation and `<cli> --help` before landing; `docs_url` values below are best-known candidates, and the doctor
`_PROVIDER_SETUP_HINTS` strings are the in-repo reference for install commands):

| Provider | Display name    | Manager/strategy        | Self-update                          | Docs URL (verify)               |
| -------- | --------------- | ----------------------- | ------------------------------------ | ------------------------------- |
| claude   | Claude Code     | npm or native installer | `claude update` (native installs)    | code.claude.com/docs setup page |
| codex    | Codex CLI       | npm (brew possible)     | none known                           | developers.openai.com/codex/cli |
| opencode | OpenCode        | native installer or npm | `opencode upgrade`                   | opencode.ai/docs                |
| qwen     | Qwen Code       | npm                     | none known                           | QwenLM qwen-code docs           |
| agy      | Antigravity CLI | curl installer          | verify via `agy --help`; else manual | antigravity.google CLI docs     |
| fakey    | Fakey           | bundled with SASE       | excluded (updates via `sase update`) | n/a                             |

This uses uniform, data-driven hooks — no runtime-specific branching in logic, consistent with the "Uniform Agent
Runtimes" convention.

As part of this, the hardcoded `_PROVIDER_SETUP_HINTS` dict in `src/sase/doctor/checks_providers.py` is consolidated:
doctor derives install/docs hints from the enriched provider metadata (keeping auth strings and a fallback for providers
without metadata), so install knowledge lives in exactly one place.

### Update strategy resolution (realistic, never guessing)

For each provider with a CLI, the service layer resolves an **install method** and from it an **update strategy**, using
this precedence:

1. **Not installed** — executable not resolvable (honoring `SASE_<PROVIDER>_PATH` overrides, same resolution as
   `sase doctor`). Never installed by this feature; listed with the install hint + docs URL.
2. **npm-managed** — the resolved executable lives under npm's global prefix (probe `npm root -g` / `npm prefix -g` once
   per run, cached) or `npm ls -g --json <package>` confirms the declared package. Strategy:
   `npm install -g <package>@latest`. If the npm global root is not writable by the current user, do **not** attempt
   (and never sudo): report a manual skip with the exact command and docs URL.
3. **Homebrew-managed** — executable resolves under a brew prefix (`/opt/homebrew`, `/usr/local/Cellar`,
   `/home/linuxbrew/...`). Detection lands in this epic; automated `brew upgrade` may be marked
   manual-with-exact-command in v1 if reliability is in doubt — the skip message carries the command and docs URL either
   way.
4. **Self-managed** — the provider declares `self_update_argv` and the executable is not npm/brew-managed (typical for
   native-installer installs of Claude Code and OpenCode). Strategy: run `<executable> <self_update_argv>`.
5. **Bundled** — fakey; shown as bundled, excluded from update operations.
6. **Unknown** — anything else. No command is run; the row/result explains what was found and points at the canonical
   docs URL.

Version handling: installed versions come from `<executable> --version` probes (subprocess with timeout, first-semver
parse with per-provider override), generalizing the existing `sase doctor` deep-provider probe. Latest versions come
from an npm-registry oracle for providers declaring `latest_version_package`, cached with a TTL snapshot following the
`sase/updates/cache.py` pattern, with offline mode reading cache only. When no oracle exists, latest is honestly
"unknown" — updates still run (self-update commands are idempotent) and results report old → new from post-run
re-probes.

### One shared plan/execute pipeline

`src/sase/agent_clis/operations.py` mirrors `sase/plugins/_operations_update.py`:

- `plan_agent_cli_updates(names, all_clis, refresh, offline)` → a typed plan union (ready / nothing-to-update /
  unknown-name), where the ready plan carries per-CLI entries: exact argv, strategy, and skip reasons (not installed,
  bundled, already up to date, manual with docs URL, unwritable npm prefix, …).
- `execute_agent_cli_updates(plan)` → per-CLI results (updated / already-current / failed / skipped) with old → new
  versions from post-run re-probes, the command that ran, an output tail on failure, and the docs URL. Commands run
  sequentially (never two package-manager invocations concurrently), each with a generous timeout, captured output, no
  shell.
- A typed subprocess runner module mirrors `sase/uv_tool/runner.py` (typed error hierarchy, `check=False` capture,
  timeout mapping) for npm/self-update invocations.

Both the CLI handlers and the TUI workers call exactly these functions.

## Phase details

### Agent CLI update service layer and provider metadata

Deliverables (all under `src/sase/` unless noted):

- `agent_clis/` package: models (`AgentCliStatus`, plan/result types), `detect.py` (executable resolution shared with
  doctor helpers, version probing, install-method detection), `latest.py` (npm-registry latest oracle + TTL cache),
  `runner.py` (typed subprocess runner), `operations.py` (plan/execute as designed above).
- Hookspec + registry: enriched `llm_install_metadata` fields with tolerant normalization; implementations updated in
  `llm_provider/{claude,codex,qwen,fakey}.py` and **added** to `llm_provider/{opencode,agy}.py`, including verified docs
  URLs and self-update argv.
- Doctor consolidation: `checks_providers.py` setup hints derived from provider metadata (auth strings stay local;
  fallback preserved); `checks_runtime_node.py` and deep provider version checks keep working unchanged.
- Unit tests with fake runners/probes covering: strategy precedence (npm beats self-managed; unknown never runs
  commands), not-installed/bundled/manual skips carrying docs URLs, version parsing, npm-prefix writability guard,
  offline/TTL cache behavior, and plan → execute → re-probe reporting. No test may invoke a real package manager.

### sase agent-cli command group

- New parser module `src/sase/main/parser_agent_cli.py` registering the `agent-cli` group (alphabetically ordered in the
  top-level command list) with children `list` and `update`; dispatch from `src/sase/main/entry.py`; handlers in
  `src/sase/agent_clis/cli_list.py` / `cli_update.py` mirroring `sase/plugins/cli_update.py` structure (console-free
  operations layer + Rich rendering).
- `sase agent-cli list`: colored Rich table — display name, binary, installed version, latest version, install method,
  and an `↑` update-available marker consistent with `sase plugin list`; not-installed rows show the install hint;
  verbose adds executable paths and docs URLs. Options: `-j/--json` (versioned schema), `-o/--offline`, `-r/--refresh`,
  `-v/--verbose`.
- `sase agent-cli update [<name> ...]`: update the named CLIs, or every updatable installed CLI with `-a/--all`; bare
  invocation without names or `-a` is a usage error (exit 2), matching `sase plugin update`. Options: `-a/--all`,
  `-j/--json`, `-n/--dry-run` (print the exact per-CLI commands and skip reasons without running anything),
  `-o/--offline`, `-r/--refresh`. Exit codes: 0 all succeeded or already current, 1 any failure, 2 usage. Failure and
  skip output always includes the provider's canonical docs URL.
- CLI conventions per the repo's CLI rules: excellent `-h` output with examples, subcommands/options sorted
  alphabetically, a short alias for every public long option, and the bare-group → `list` default (auto-wired by
  `_default_list_subcommands()`; document it in the group help like `sase plan`).
- Cross-reference: `sase update`'s epilog gains a one-line pointer to `sase agent-cli update` (agent CLIs are not
  covered by `sase update`).
- Tests: parser/handler tests with stubbed operations covering table/JSON output, dry-run, exit codes, unknown-name
  errors (exit 2, docs pointer), and offline behavior.

### Updates tab split and Agent CLIs sub-tab

Restructure `src/sase/ace/tui/modals/plugins_browser_pane.py` (and its mixin modules) to host three pane-local sub-tabs
under the existing Updates tab, copying the `ProjectsPane` pattern (`PanelTabStrip` + nested `ContentSwitcher`, `]` /
`[` cycling, clickable strip, `check_action` gating, filter-input key forwarding). `[`/`]` are currently unused on this
pane, so nothing conflicts with the Admin Center's `tab`/`shift+tab` and digit keys.

- **Core** (default sub-tab): the all-current banner, the SASE core versions panel, and core update status/hints. The
  `,U` leader flow and the updates-indicator click-through (`auto_update_on_load`) land here and behave exactly as
  today.
- **Plugins**: the existing plugin filter input, master/detail browser, and all current plugin actions (`i`, `I`/`space`
  marks, `x`, `m`, `U`) move here unchanged; `check_action` gates them to this sub-tab.
- **Agent CLIs**: a new master/detail browser over `AgentCliStatus` rows — display name tinted with the provider's
  `llm_cli_status_color`, installed → latest versions, install method badge, `↑` update marker, and a mark glyph.
  `space` toggles a per-CLI update mark (mirroring the Plugins marks UX; `escape` clears marks first, then closes). The
  detail panel shows versions, resolved executable, install method, the exact planned update command (or the manual
  command + reason), the canonical docs URL, and the last update outcome. Loading happens in the pane's existing
  threaded load worker path (never on the event loop), honoring `o` offline and `r` refresh, with the TTL-cached latest
  oracle.
- **`A` (update agent CLIs)**, bound pane-wide and active on all three sub-tabs (uppercase `A` is free; only `I`, `U`,
  `G` are taken): plans updates off-thread for the marked subset (when on the Agent CLIs sub-tab with marks) or every
  updatable installed CLI otherwise, then opens a confirm-with-preview modal in the `PluginActionConfirmModal` style
  showing each CLI's exact command and every skip with its reason and docs URL. On confirm the execution runs as a
  tracked background task (`_submit_tracked_task`, dedup key `agent-cli-update`, visible in the Tasks tab), then toasts
  a summary (updated old → new / already current / failed with docs pointer) and refreshes the sub-tab. No ACE restart
  is involved — agent CLIs are external binaries; freshly launched agents pick up new versions.
- **`u` unchanged**: still `update_sase` with identical behavior and gating, active from all three sub-tabs.
  Documentation is clarified everywhere it appears (pane hints, help modal, leader-mode help): `u` updates SASE core +
  plugins; `A` updates agent CLIs.
- Documentation/beauty sweep: `_TAB_DESCRIPTIONS["updates"]` and the `ConfigCenterModal` docstring mention agent CLIs;
  per-sub-tab hints lines; the `?` help modal Updates section documents `[`/`]`, `A`, and the sub-tabs (help popup sync
  is mandatory in this repo). The sub-tab strip reuses the Updates accent, matching the Projects sub-tab styling. No
  `default_config.yml` keymap changes are needed (pane bindings are class-level), but this must be re-verified after
  binding work.
- Tests: unit tests mirroring `tests/ace/tui/test_plugins_browser_pane*.py` for sub-tab switching/gating, marks, the `A`
  plan→confirm→execute flow (stubbed operations), and `u` regression on every sub-tab. Visual PNG snapshots: new goldens
  for each sub-tab (including a marked Agent CLIs state and the `A` confirm preview) plus re-goldening the existing
  `config_center_plugins_*` / `config_center_updates_*` snapshots that now include the sub-tab strip; deterministic
  fixtures stub agent-CLI probing (no real subprocesses).

### End-to-end agent CLI update exercises

Read-only, real-environment verification (no real updates are executed, since that would mutate the host):

- `sase agent-cli`, `sase agent-cli list`, `list -j`, `list -v -o` — sane tables/JSON, bare-group delegation notice,
  offline behavior.
- `sase agent-cli update -n -a` and `update -n <name>` — dry-run previews show exact commands and docs-URL-bearing skip
  reasons; `sase agent-cli update` (bare) and an unknown name exit 2 with actionable messages.
- `sase agent-cli --help` / subcommand help — alphabetical, aliased, with examples.
- `sase doctor` provider checks still pass with the consolidated hints.
- The TUI visual snapshot suite and the pane unit tests pass; a manual `sase ace` pass over the three sub-tabs confirms
  navigation (`[`/`]`, digits, `tab`), marks, and that `A`'s confirm modal renders correctly before cancelling out.

## Risks and mitigations

- **Vendor churn** (install mechanisms, self-update commands, docs URLs change): all such knowledge lives in provider
  metadata with docs-URL fallbacks; unknown situations degrade to manual guidance rather than wrong commands.
- **Detection ambiguity** (symlinked binaries, version managers like nvm/volta shims): the precedence rules only
  automate positively identified methods; everything else is `unknown` → manual with docs. nvm-style per-version npm
  prefixes resolve naturally via `npm root -g` from the user's environment.
- **TUI responsiveness**: all probing/planning/execution stays on threaded workers and tracked tasks per the established
  Updates-pane patterns; the periodic update indicator is deliberately **not** extended to agent CLIs in this epic (its
  network cost is bounded to the Updates tab being open or explicit CLI runs), noted as future work.
- **Snapshot churn**: the sub-tab split re-goldens existing Updates-tab PNGs; keeping widget IDs stable where possible
  limits unit-test fallout.
- **Concurrent running agents**: updating a CLI while agents run is safe on Linux (running processes keep their loaded
  binary); new launches use the new version. The confirm modal does not need to warn about this.
