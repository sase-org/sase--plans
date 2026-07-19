---
tier: tale
title: Fix sase update failure from stale ephemeral plugin paths
goal: 'sase update succeeds again on the dev machine, and sase never silently records
  an ephemeral workspace path as a plugin install source nor fails opaquely when a
  receipt-pinned local path has disappeared.

  '
create_time: 2026-07-18 22:41:48
status: wip
prompt: 202607/prompts/fix_sase_update_stale_plugin_paths.md
---

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
