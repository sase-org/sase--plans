---
tier: tale
title: Fix sase update failure from stale ephemeral plugin paths
goal: 'sase update succeeds again on the dev machine, and sase never silently records
  an ephemeral workspace path as a plugin install source nor fails opaquely when a
  receipt-pinned local path has disappeared.

  '
create_time: 2026-07-18 22:41:48
status: done
---

- **PROMPT:** [202607/prompts/fix_sase_update_stale_plugin_paths.md](prompts/fix_sase_update_stale_plugin_paths.md)
- **AGENTS:**
  - [bbugyi200.athena.e8](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.e8/README.md)
  - [bbugyi200.athena.e8--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.e8.md#member-code)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._display_detail.9cceb35a-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._display_detail.9cceb35a-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._marking.2877bedf-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._marking.2877bedf-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._notification_mo.418eb0a2-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._notification_mo.418eb0a2-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._unread.2e7e99ec-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.actions.agents._unread.2e7e99ec-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.modals.model_picker_modal.49f651de-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.modals.model_picker_modal.49f651de-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.modals.statistics_pane.6fe343e1-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.modals.statistics_pane.6fe343e1-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.util.stall_watchdog.c1d8ffc6-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.util.stall_watchdog.c1d8ffc6-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets._file_completion_open.3b43b67f-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets._file_completion_open.3b43b67f-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.artifacts.commits_pane.3abf0379-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.artifacts.commits_pane.3abf0379-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.artifacts.plans_data.282e52fa-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.artifacts.plans_data.282e52fa-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.artifacts.plans_pane.cc493c74-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.artifacts.plans_pane.cc493c74-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.directive_completion.16db5c71-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.directive_completion.16db5c71-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.prompt_panel._agent_cla.e0b4df69-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.prompt_panel._agent_cla.e0b4df69-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.prompt_panel._agent_dis.e3a4d712-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.prompt_panel._agent_dis.e3a4d712-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.ace.tui.widgets.prompt_panel._agent_dis.f038031a-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.ace.tui.widgets.prompt_panel._agent_dis.f038031a-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.agent.launch_request.81c67a28-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.agent.launch_request.81c67a28-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.agent.multi_prompt_launcher.8e655e93-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.agent.multi_prompt_launcher.8e655e93-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.agent.names._lookup.c2144d73-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.agent.names._lookup.c2144d73-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.axe.chop_runner_script.620a4ae7-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.axe.chop_runner_script.620a4ae7-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.axe.config.8f537af3-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.axe.config.8f537af3-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.bead.cli_work_from_plan.85ac2829-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.bead.cli_work_from_plan.85ac2829-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.config.core.42c0c9ed-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.config.core.42c0c9ed-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.core.agent_artifact_index_lifecycle.ce47e900-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.core.agent_artifact_index_lifecycle.ce47e900-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.core.agent_cleanup_facade.6a506bfa-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.core.agent_cleanup_facade.6a506bfa-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.core.wait_dependency_resolution._index.5fb08746-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.core.wait_dependency_resolution._index.5fb08746-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.history.chat.635e50ab-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.history.chat.635e50ab-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.integrations.agent_list_entries.2a6546d6-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.integrations.agent_list_entries.2a6546d6-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.memory.inventory.9b600620-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.memory.inventory.9b600620-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.notification_gates.debug.48460b6d-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.notification_gates.debug.48460b6d-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.notification_gates.models.366491c2-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.notification_gates.models.366491c2-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.notification_gates.registry.d4699587-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.notification_gates.registry.d4699587-0/README.md)
  - [bbugyi200.athena.split_file.src.sase.plan_approval_actions.46e9c10a-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.src.sase.plan_approval_actions.46e9c10a-0/README.md)
  - [bbugyi200.athena.split_file.tests.ace.tui.models.test_agent_associated_plan.a5e26426-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.tests.ace.tui.models.test_agent_associated_plan.a5e26426-0/README.md)
  - [bbugyi200.athena.split_file.tests.ace.tui.models.test_agent_groups_grouping_.d6ab407a-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.tests.ace.tui.models.test_agent_groups_grouping_.d6ab407a-0/README.md)
  - [bbugyi200.athena.split_file.tests.ace.tui.test_agent_neighbor_navigation.8b2a5c29-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.tests.ace.tui.test_agent_neighbor_navigation.8b2a5c29-0/README.md)
  - [bbugyi200.athena.split_file.tests.ace.tui.test_agent_unread_done_navigation.a6f7c380-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.tests.ace.tui.test_agent_unread_done_navigation.a6f7c380-0/README.md)
  - [bbugyi200.athena.split_file.tests.ace.tui.test_artifacts_plans.1a8be42b-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.tests.ace.tui.test_artifacts_plans.1a8be42b-0/README.md)
  - [bbugyi200.athena.split_file.tests.ace.tui.test_jump_hints_for_folded_banners.61434913-0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.split_file.tests.ace.tui.test_jump_hints_for_folded_banners.61434913-0/README.md)
- **COMMITS:**
  - [4e2a948](https://github.com/sase-org/sase/commit/4e2a9488a052ef061e555f6ae387138bf31d5429) — refactor(ace): split agent detail display helpers
  - [185469e](https://github.com/sase-org/sase/commit/185469e09dc7c42908e9b0ecd2bc4674eb0eaa65) — fix(plugins): preflight stale local install sources
  - [08d1c79](https://github.com/sase-org/sase/commit/08d1c79850994aba686be83647fb835590a313fd) — refactor(ace): split agent marking actions
  - [98579a0](https://github.com/sase-org/sase/commit/98579a05f72436211127e0c16764b295e0553327) — refactor: split notification plan modal helpers
  - [efd19c9](https://github.com/sase-org/sase/commit/efd19c925cc6614c9bbc54ebe3ae098bb3cb3185) — refactor(ace): split unread agent actions
  - [33d3e6f](https://github.com/sase-org/sase/commit/33d3e6f21943f50b16e16af6fe6e73f0b652bdd9) — refactor(tui): split model picker modal
  - [453a84a](https://github.com/sase-org/sase/commit/453a84af8aa14cc449ee160f58e9634c7c55d73c) — refactor: split statistics pane rendering
  - [360b691](https://github.com/sase-org/sase/commit/360b6918247f45d51697f09534929bd02b7594de) — refactor(tui): split stall watchdog internals
  - [57b57f3](https://github.com/sase-org/sase/commit/57b57f30fb4cd401cb80f46023ba15b3cb0809dc) — refactor(ace): split manual file completion handlers
  - [3a570cb](https://github.com/sase-org/sase/commit/3a570cb6cba177997f9c78ae2b44351ccd6f7bf6) — refactor(ace): split commits pane responsibilities
  - [63dc8c6](https://github.com/sase-org/sase/commit/63dc8c6d24788356a7cad223b7d812ead3cb2685) — refactor(ace): split plans data module
  - [c537447](https://github.com/sase-org/sase/commit/c5374479cf9beacd99b319bf736627ec3c63af78) — refactor(ace): split plans pane responsibilities
  - [30e1577](https://github.com/sase-org/sase/commit/30e15776174a32f74e93e4d71d98c441b58d4ad9) — refactor(ace): split directive completion token parsing
  - [fcafb33](https://github.com/sase-org/sase/commit/fcafb33b1645cd3e813a4dcb7e07f4f5f6a5a4ea) — refactor(tui): split agent clan aggregation helpers
  - [df4777e](https://github.com/sase-org/sase/commit/df4777eaa8943267bcf75c54f3282783ed8d3344) — refactor(tui): split clan display implementation
  - [d9ea0c3](https://github.com/sase-org/sase/commit/d9ea0c3552e3860ce95e3c34a76b265e598b6ce4) — refactor(tui): split agent display header rendering
  - [6ae0dd2](https://github.com/sase-org/sase/commit/6ae0dd2b73f07602463bbb5eb745db3e75193d26) — refactor(agent): split launch request responsibilities
  - [689f39b](https://github.com/sase-org/sase/commit/689f39b2ad91b3353f253a4de930719cd7ae5eca) — refactor(agent): split multi-prompt launcher
  - [2b4526f](https://github.com/sase-org/sase/commit/2b4526f959b4e5a05b8d47e198f49b74f3fcd5ad) — refactor(agent-names): split lookup implementation
  - [b78b34b](https://github.com/sase-org/sase/commit/b78b34b3643933289e22364eec49009879c2198a) — refactor(axe): split chop runner script responsibilities
  - [dd7fc1a](https://github.com/sase-org/sase/commit/dd7fc1ace88c8ddc7fbac0d4b18316a227728310) — refactor(axe): split configuration module
  - [b2d8cef](https://github.com/sase-org/sase/commit/b2d8cefd2eecae5ec52aa29696d034b835b0a966) — refactor(bead): split plan-file work CLI module
  - [552e24c](https://github.com/sase-org/sase/commit/552e24c3116f93d40f09a284b2fffd3bb2f1b11c) — refactor(config): split core config responsibilities
  - [3cfd6d6](https://github.com/sase-org/sase/commit/3cfd6d632d65dc615d8dd7a40ca641a942e99a61) — refactor(core): split agent artifact index lifecycle
  - [ae6ff5d](https://github.com/sase-org/sase/commit/ae6ff5d22acb9edf43eb73132c292f9c09bef11d) — refactor: split agent cleanup facade
  - [da17707](https://github.com/sase-org/sase/commit/da1770780936971499bd2421a2b2181a85086eaa) — refactor: split wait dependency index queries
  - [c659638](https://github.com/sase-org/sase/commit/c6596383a14fd9171a97d595056ae9f35eb4141c) — refactor(history): split chat history module
  - [7ae8298](https://github.com/sase-org/sase/commit/7ae829839047fe2c0c603358e818268656ca1240) — refactor(integrations): split agent list entry module
  - [0a36657](https://github.com/sase-org/sase/commit/0a36657fca6544acb16041f1f75029f79fe83257) — refactor(memory): split inventory into focused modules
  - [1b9659f](https://github.com/sase-org/sase/commit/1b9659f362e90eb5ef5dc3b617d7a8193a017d16) — refactor: split notification gate debug module
  - [81c5553](https://github.com/sase-org/sase/commit/81c5553093b770851199c0213fdef19739f15983) — refactor(notification-gates): split model definitions
  - [a07f029](https://github.com/sase-org/sase/commit/a07f029125a9b5b5c663e1d6d798611e5474b817) — refactor(notification-gates): split registry modules
  - [6bc0292](https://github.com/sase-org/sase/commit/6bc02923d46585db29dd8dc4d51de338cd9c6441) — refactor: split plan approval actions by responsibility
  - [98abf03](https://github.com/sase-org/sase/commit/98abf033fc78b7f39e8beae2acc8bde0bd1651f1) — test: split associated plan model tests
  - [7b7caa7](https://github.com/sase-org/sase/commit/7b7caa77e8f95e0161ec3eef580c499244b4a509) — test: split agent grouping mode tree tests
  - [83d3143](https://github.com/sase-org/sase/commit/83d3143a41a7b2d80c0d1364e4b4c830345834fa) — test: split agent neighbor navigation tests
  - [41ef643](https://github.com/sase-org/sase/commit/41ef64379fede7505a6e6d926e473234aedcb142) — test(ace): split unread done navigation tests
  - [cdd9073](https://github.com/sase-org/sase/commit/cdd907377a7ae0e111418eccfa0a5f12f87508a6) — test(ace): split artifacts plans test module
  - [4824443](https://github.com/sase-org/sase/commit/4824443e599f04c7bef532399d09f94b3fefd183) — test: split folded banner jump hint coverage

# Plan: Fix `sase update` failure from stale ephemeral plugin paths

## Problem

`sase update` currently fails on the dev machine. The dev-update journal (`~/.sase/logs/dev_update.jsonl`) shows the
"Reinstall uv-tool editable Python packages" reconcile step dying with:

```
error: Distribution not found at:
file://<workspace-store>/.../sase/repos/external/gh/bbugyi200/bugyi-chops
```

Root cause chain:

1. `sase plugin install <query>` passes any local path through verbatim (`resolve_install_spec` →
   `Requirement.from_spec` → `url=<path>` → `--with <path>`), and uv records that path in `uv-receipt.toml` as a
   `directory` requirement.
2. The `bugyi-chops` plugin was installed from an **external-repo checkout inside an ephemeral numbered workspace** (a
   `sase/repos/external/gh/bbugyi200/bugyi-chops` checkout under the managed workspace store at
   `~/.local/state/sase/workspaces/...`). Workspace-local repo checkouts are ephemeral by design; that checkout has
   since been cleaned up.
3. The uv receipt is the source of truth for the injected set (epic decision D2), and every receipt-reconstructing uv
   command re-injects the full `--with` set. So every `sase update` (both the dev reconcile step built in
   `sase/dev_update/plan.py` and the managed leg built via `sase/main/update_routing.py::managed_update_argv`) replays
   the dead path and uv fails with a raw, non-actionable error. `sase plugin update` shares the same reconstruction and
   fails the same way.

Context worth knowing: `bugyi-chops` is **not published on PyPI** (the sase-6v.8 bead intended publication but it never
happened), which is why it was installed from a local checkout in the first place. The GitHub repo
`bbugyi200/bugyi-chops` is public and carries the `sase--plugin` topic, so `sase plugin install --git bugyi-chops` (→
`git+https://github.com/bbugyi200/bugyi-chops`) is a durable install source today.

## Design

Two code fixes plus a machine remediation. No new CLI subcommands or flags.

### Fix A — reject ephemeral install sources at `sase plugin install` time (prevention)

Add a small, pure "ephemeral path" predicate and apply it when planning plugin installs:

- **Detection helper.** Expose a public accessor for the managed workspace-store root from
  `sase/workspace_provider/store.py` (today `_default_state_root()` is private; honor the `SASE_WORKSPACE_ROOT` env
  override the same way `_resolve_root` does). Build a predicate (suggested home: a new `sase/uv_tool/preflight.py`
  module) that answers "does this local path live inside the managed workspace store?" using normalized absolute-path
  containment. As a secondary signal, also treat paths containing the managed repo-clones subdirectories (the
  `sase/repos/external` and linked-repo clone segments from `sase/linked_repos.py`) as ephemeral, so checkouts
  materialized under a host checkout are caught even under a nonstandard store root.
- **Wiring.** In `sase/plugins/_operations_install.py`:
  - `plan_install`: when the resolved spec is a local-path source (an `editable` path or a `url` that is a filesystem
    path) and the predicate flags it, return an error outcome (reuse the existing `UvToolError`-rendering path so `-j`
    and exit codes keep their current shapes) instead of `InstallReady`. The message must name the offending path, say
    that workspace-local checkouts are ephemeral, and point at durable alternatives: `sase plugin install --git <name>`
    or a durable checkout path.
  - `plan_install_many`: flag the same condition per query as an `InstallSkipped` with that reason, consistent with the
    existing skip reporting.
- Passing a durable path (outside the workspace store) must keep working unchanged.

### Fix B — preflight missing local paths on receipt reconstruction (diagnosability)

Add a preflight that turns today's raw uv failure into an actionable, typed error:

- **Helper.** In the same `sase/uv_tool/preflight.py`, a function that, given the requirements actually being injected
  (a `ReconstructedRequirements` set or an iterable of `Requirement`s), returns the entries whose local-path source
  (editable path, or `url`/`directory` that is a filesystem path) no longer exists on disk. Non-path sources (index,
  git, http URLs) are never checked.
- **Actionable message** (shared template): plugin name, the dead path, and the remediation: run
  `sase plugin uninstall <name>`, then reinstall from a durable source (e.g. `sase plugin install --git <name>`).
- **Wiring points** — preflight the _reconstructed set actually passed to uv_, not the raw receipt, so removing the
  broken plugin remains possible:
  - Dev leg: in `sase/dev_update/plan.py::_reconcile_steps`, when building the `uv_tool_install` step, run the preflight
    on the receipt's reconstructed set; on a hit, emit the step **unavailable** (`command=()`,
    `reason=<actionable message>`). The executor already fails the run with an unavailable step's reason
    (`sase/dev_update/execute.py`), so the message flows into the update panel, JSON, and the journal without new
    plumbing.
  - Managed leg: in `sase/main/update_handler.py`, before running the managed argv (both call sites that use
    `managed_update_argv`), preflight and fail with a `UvToolError` carrying the same message. This also covers the ACE
    plugins-browser dev-update path if it flows through the same helpers; verify
    `sase/ace/tui/modals/plugins_browser_dev_update.py` and give it the same guard if it executes independently.
  - `sase plugin update` (and `--all`) in `sase/plugins/_operations_update.py`: same preflight on the reconstructed set
    before running uv.
  - `sase plugin uninstall` must **not** be blocked when the missing-path plugin is the one being removed: its
    reconstruction already excludes the target, so preflighting the reconstructed set naturally allows it. Add a test
    locking this in — it is the remediation path.

### Machine remediation and end-to-end verification (this machine, dev install)

After the code changes pass checks, repair the live environment and actually update sase:

1. `sase plugin uninstall bugyi-chops` — rebuilds the uv tool env without the dead path (works today because the
   reconstruction excludes the removed plugin).
2. `sase plugin install --git bugyi-chops` — reinstalls from the durable public GitHub source. Verify the new receipt
   entry is a `git+https://...` source and that no receipt requirement points into the workspace store anymore
   (`~/.local/share/uv/tools/sase/uv-receipt.toml`).
3. `sase update -n` to preview, then `sase update` — expect exit 0, the previously failing "Reinstall uv-tool editable
   Python packages" step green, and the rust-core/LSP rebuild + health-check steps to run. Spot-check `sase --version`
   and that the `bugyi-chops` chop entry points are still installed.

## Testing

- Unit tests for the ephemeral-path predicate (store-root containment, `SASE_WORKSPACE_ROOT` override, repo-clones
  subdir signal, durable paths negative).
- `plan_install` / `plan_install_many` guard tests using the existing fakes in `tests/test_plugin_operations_install.py`
  (blocked ephemeral path, allowed durable path, JSON/exit-code shape).
- Reconcile-step preflight tests alongside the existing dev-update suite in `tests/dev_update/` (missing path →
  unavailable step with actionable reason; intact paths → unchanged behavior).
- Managed-leg and `plugin update` preflight tests (typed error, message content).
- Uninstall-of-broken-plugin regression test (must still plan successfully).
- Full `just check` before finishing, per repo policy.

## Risks and non-goals

- **Non-goal:** publishing `bugyi-chops` to PyPI (tracked by the sase-6v lineage) and any auto-fallback from PyPI to git
  during catalog installs.
- **Non-goal:** auto-dropping or rewriting stale receipt entries. Silent membership changes uninstall plugins behind the
  user's back; we fail with instructions instead.
- **Non-goal:** warning about receipt paths that are ephemeral but still exist. Fix A prevents new ones from being
  recorded, making that state unreachable going forward.
- **Risk:** overly broad ephemeral detection could block legitimate durable-path installs (e.g. unusual store-root
  configs). Mitigate with the env-aware root resolution and tests for durable-path acceptance.
- **Risk:** the preflight must never crash version/update rendering on odd receipts; keep it pure over parsed
  `Requirement`s and covered by unit tests.
