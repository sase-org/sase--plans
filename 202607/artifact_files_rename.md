---
tier: tale
title: Rename the agent artifact-file concept to "artifact files"
goal: 'The ARTIFACTS lane of the Agents-tab metadata panel labels its explicit-artifact
  field "Files", and every code, CLI, skill, config, test, and doc reference to that
  concept says "artifact file(s)" — cleanly disambiguated from run-directory artifacts
  and the Artifacts tab — with zero change to any persisted on-disk format.

  '
create_time: 2026-07-17 10:44:38
status: wip
prompt: 202607/prompts/artifact_files_rename.md
---

# Plan: Rename the agent artifact-file concept to "artifact files"

## Product context

On the Agents tab of the `sase ace` TUI, the agent metadata panel's SASE CONTEXT section has a ranked **ARTIFACTS** lane
with three fields: `Commits:`, `Deltas:`, and `Artifacts:`. The last field lists the explicit files an agent registered
during its run (via `sase artifact create` / the `sase_artifact` skill) plus synthesized defaults (chat, plan, media).
Having a field named "Artifacts" inside a lane named "ARTIFACTS" is confusing, and the bare word "artifact" is badly
overloaded across the codebase.

This change renames the **field** to `Files:` and renames the **underlying concept** to "artifact file(s)" everywhere it
is referenced — UI labels, Python symbols and modules, the CLI command, the generated skill, config keys, tests, and
docs.

## Concept taxonomy — read this before touching anything

The word "artifact" names three unrelated things. Only the first is being renamed.

1. **Artifact files (RENAME TARGET)** — the explicit/default files registered for an agent run. Pure-Python model:
   `AgentArtifact` dataclass + a global JSONL index at `~/.sase/artifacts/index.jsonl` with stored copies under
   `~/.sase/artifacts/agents/`. Core modules: `src/sase/core/agent_artifact_types.py`, `agent_artifact_explicit.py`,
   `agent_artifact_defaults.py`, `agent_artifact_helpers.py`, `agent_artifact_facade.py`. Surfaces: the `Artifacts:`
   lane field, the "Agent Artifacts" modal (key `a`), `sase artifact create`, the `sase_artifact` skill. Roughly 35
   source files plus ~17 test files.

2. **Run-directory artifacts (DO NOT RENAME)** — the per-run directory
   `projects/<project>/artifacts/<workflow>/<timestamp>/` plus its marker files and the SQLite scan index
   (`~/.sase/agent_artifact_index.sqlite`). This is the Rust-backed backbone of the whole agent list:
   `agent_artifact_paths.py`, `agent_artifact_index_lifecycle.py`, `agent_artifact_index_lock.py`, all `agent_scan_*`
   wire modules, every `artifacts_dir`/`artifact_dir` field, the `SASE_ARTIFACTS_DIR` env var, the
   `sase agent artifacts layout` and `sase agent index` CLIs, and every `AgentArtifact*` symbol in the sase-core Rust
   repo (including the frozen mobile API v1 contract fields `artifact_dir`/`has_artifact_dir`). None of this changes.
   sase-core, sase-telegram, sase-github, and sase-nvim require **no edits**.

3. **Other "artifacts" (DO NOT RENAME)** — the TUI **Artifacts tab** and its subtabs (plans/commits/bugs:
   `src/sase/ace/tui/widgets/artifacts/`, `artifact_tabs.py`, `cycle_artifacts_subtab*` keymaps), the ARTIFACTS **lane
   header string itself** (it stays "ARTIFACTS" — it umbrellas commits + deltas + files), plan artifacts
   (`plan_search/`, sase-core `plan/`), commit artifacts, the xprompt-DSL `artifact:` step-output keyword (`git.yml`,
   `json.yml`, `file.yml`), and build/demo/perf artifacts in the Justfile.

**Disambiguation invariant:** after this change, `agent_artifact*` / `AgentArtifact*` names refer exclusively to
run-directory machinery (concept 2), while the file concept uses `artifact_file*` / `ArtifactFile*`. A reference is in
scope only if it flows through the `AgentArtifact` file model or its UI/CLI/skill surfaces. If it touches run
directories, the SQLite index, or the scan wire, leave it alone.

## Naming scheme

Canonical term: **artifact file(s)**. Apply this mapping and extend the same rule to every remaining in-scope symbol,
module, and string:

| Current                                                                  | New                                                                    |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `AgentArtifact`                                                          | `ArtifactFile`                                                         |
| `AgentArtifactAssociation`                                               | `ArtifactFileAssociation`                                              |
| `AgentArtifactKind` / `AGENT_ARTIFACT_KINDS`                             | `ArtifactFileKind` / `ARTIFACT_FILE_KINDS`                             |
| `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` (JSONL, value 1)                   | `ARTIFACT_FILE_INDEX_SCHEMA_VERSION` (value unchanged)                 |
| `store_explicit_agent_artifact` / `store_default_agent_artifact`         | `store_explicit_artifact_file` / `store_default_artifact_file`         |
| `list_agent_artifacts` / `list_explicit_agent_artifacts`                 | `list_artifact_files` / `list_explicit_artifact_files`                 |
| `synthesize_default_agent_artifacts` / `persist_default_agent_artifacts` | `synthesize_default_artifact_files` / `persist_default_artifact_files` |
| `AgentArtifactPath` (prompt-panel dataclass)                             | `ArtifactFilePath`                                                     |
| `core/agent_artifact_{types,explicit,defaults,helpers,facade}.py`        | `core/artifact_file_{types,explicit,defaults,helpers,facade}.py`       |
| `agent/agent_artifacts_cache.py` / `AgentArtifactCache`                  | `agent/artifact_files_cache.py` / `ArtifactFileCache`                  |
| `ace/tui/modals/agent_artifacts_modal.py`                                | `ace/tui/modals/artifact_files_modal.py`                               |
| `ace/tui/models/agent_artifacts.py`                                      | `ace/tui/models/artifact_files.py`                                     |
| `prompt_panel/_agent_artifacts.py`                                       | `prompt_panel/_artifact_files.py`                                      |
| `COLOR_ARTIFACT_PATH` / `COLOR_ARTIFACT_BASENAME`                        | `COLOR_ARTIFACT_FILE_PATH` / `COLOR_ARTIFACT_FILE_BASENAME`            |

Field names that mirror run-directory identity (e.g. the `agent_artifacts_dir` attribute on the file model, which points
at the concept-2 run dir) keep their names. `prompt_panel/_agent_artifacts_lane.py` renders the whole ARTIFACTS lane
(concept 3 header), so the module may keep its lane-oriented name; only its `Artifacts:` field label and count phrase
change.

## Changes by area

### 1. TUI labels and widgets

- `prompt_panel/_agent_artifacts_lane.py`: field label `"  Artifacts:\n"` → `"  Files:\n"`; lane-header count phrase
  `count_phrase(n, "artifact")` → `count_phrase(n, "artifact file")` so the header reads e.g.
  `2 commits · 5 files · 1 artifact file` without colliding with the Deltas file count. The `"ARTIFACTS"` header and
  `COLOR_ARTIFACTS_SUBHEADER` stay.
- Agent artifact-files modal: title `"Agent Artifacts"` → `"Artifact Files"`; keep key `a` and all bindings. Update the
  help-modal entry ("Artifacts pane (or marked set)" → "Artifact files pane (or marked set)"), keybinding-footer hints
  ("artifacts", "artifacts (marked)" → "artifact files" variants), agent-onboarding copy, and the command-palette
  metadata label.
- Rename the A1 TUI support modules per the naming scheme: modal, models, prompt-panel widget,
  `actions/agents/_panel_artifacts.py`, `_artifact_provider.py`, `graphics/_viewer_artifacts.py`, and the
  `event_refresh` artifact-path classifiers — each only where it operates on the file model (verify against the
  invariant first).

### 2. Core model and callers

- Rename the five core modules and all symbols per the table; keep `sase.core.artifact_file_facade` as the compatibility
  barrel the way `agent_artifact_facade` is today.
- Update all in-repo callers: the axe finalize path (`run_agent_exec_plan_artifacts.py`, runner finalize modules call
  `store_explicit_agent_artifact` / `persist_default_agent_artifacts`), plan/var handlers, and TUI consumers. Callers
  that also touch the run-dir index (`update_agent_artifact_index_for_marker_mutation`) keep those concept-2 calls
  unrenamed.

### 3. CLI

- Rename the command group `sase artifact` → `sase artifact-file` (subcommand `create` and its `-p/--path`,
  `-n/--label`, `-k/--kind` options unchanged). Rename `main/parser_artifact.py` and `main/artifact_handler.py`
  accordingly and update help text to "artifact file" phrasing (e.g. "Move a file into SASE artifact-file storage for
  the current agent"). Follow the CLI rules memory: alphabetical listing, short aliases, polished `-h` output. No
  back-compat alias is needed — the only caller is the regenerated skill.
- `SASE_ARTIFACTS_DIR` and `SASE_AGENT` env checks are concept 2 and stay as-is.

### 4. Config and schema

- `src/sase/default_config.yml`: rename keymap `open_agent_artifacts` → `open_artifact_files` (key stays `a`) and the
  `artifact_viewer:` block → `artifact_file_viewer:`. Mirror both in `src/sase/config/sase.schema.json`,
  `ace/tui/keymaps/types.py`, `bindings.py`, and command metadata. The `cycle_artifacts_subtab*` /
  `pick_artifacts_project` keys are concept 3 and stay.
- No known user config overrides these keys (the chezmoi-managed `sase.yml` has no artifact references), so no
  legacy-key alias is required.

### 5. Skill + chezmoi regeneration

- Rename the skill source `src/sase/xprompts/skills/sase_artifact.md` → `sase_artifact_file.md`: frontmatter
  `name: sase_artifact_file`, body switched to `sase artifact-file create …` and "artifact file" phrasing.
- Audit the other skill templates (`sase_var.md`, `sase_chats.md`, `sase_agents_status.md`) and update wording only
  where they refer to the file concept or the old skill name.
- Regenerate and deploy per the generated-skills memory: `sase skill init --force`, then `chezmoi apply`. Open the
  chezmoi linked repo through the `/sase_repo` skill and remove the now-stale generated `skills/sase_artifact/`
  directories for every provider (claude, codex, gemini, antigravity-cli, jetski, qwen, opencode); also remove the stale
  deployed copies from the live home locations so no runtime still advertises the old skill. Commit the chezmoi repo
  through the sanctioned sase commit flow.

### 6. Docs

- `README.md`: update the Agents-tab prose that means artifact files (currently "diffs, chats, and artifacts"-style
  wording) to say "artifact files".
- Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or provider instruction shims — grep confirmed none reference the
  file concept, and those files must never be changed without the user's explicit in-conversation permission. If an edit
  there ever seems necessary, stop and ask the user instead.

### 7. Tests

- Rename/update the artifact-file test set, including: `tests/test_artifacts.py`, `tests/test_agent_artifact_e2e.py`,
  `tests/main/test_artifact_handler.py` (CLI), `tests/agent/test_agent_artifacts_cache.py`, the modal tests
  (`tests/ace/tui/modals/test_agent_artifacts_modal_*.py` + helpers), prompt-panel and display tests asserting the
  literal `"Artifacts:"` label (`tests/ace/tui/widgets/test_agent_context.py`,
  `test_agent_display_artifact_metadata.py`, `test_prompt_panel_*_artifacts.py`), and the actions tests
  (`test_agent_artifact_async_discovery.py`, image/open/marked tests).
- Leave the concept-2/3 tests untouched: `test_agent_artifact_index_*`, `test_agent_artifact_layout`, marker-audit
  tests, `test_artifacts_{bugs,plans,…}` tab tests, `test_artifact_pane_tab_navigation.py`.
- PNG visual suite: `tests/ace/tui/visual/test_ace_png_snapshots_agents_artifacts.py` renders the affected panel;
  regenerate goldens with `--sase-update-visual-snapshots` only for the intentional label change, per the build-and-run
  memory.

## Persisted-format stability rules (hard requirements)

- `~/.sase/artifacts/` root, `index.jsonl`, `index.lock`, and the stored-copy layout keep their paths and names.
- The JSONL row shape stays byte-compatible: `{"schema_version": 1, "artifact": {...}}` with all existing inner key
  names (including `agent_artifacts_dir`). If any dataclass field rename would leak into serialization, map keys
  explicitly instead. No schema-version bump.
- Run-directory layout, marker filenames, the SQLite index, and the sase-core wire are untouched by definition (out of
  scope).

## Verification

- `just install`, then `just check` (lint + mypy + symvision + tests). No symvision pragmas or epic-symbol entries
  reference the renamed symbols today; renames must keep every public symbol's real consumers wired so no new whitelist
  is needed.
- Run the targeted suites directly (artifact-file tests, modal tests, prompt-panel tests) plus `just test-visual` for
  the PNG suite.
- Launch `sase ace`, select an agent with registered artifact files, and confirm: the ARTIFACTS lane shows `Files:` with
  an "N artifact file(s)" header count, the `a` modal opens titled "Artifact Files", and `sase artifact-file create`
  works end-to-end from an agent run (the e2e test covers this flow).

## Risks

- **Mis-scoping is the main risk.** `grep -l agent_artifact` matches ~139 files in `src/`, but only ~35 are the file
  concept; the rest are run-directory machinery whose rename would break the Rust parity boundary. Always classify
  against the taxonomy before renaming a hit.
- The config-key renames (`open_agent_artifacts`, `artifact_viewer`) would break any unknown out-of-tree config that
  sets them; none are known to exist.
- Visual snapshot churn on the agents panel is expected and intentional; accept only diffs caused by the label change.
