---
tier: tale
title: Fix the running_field / agent_launch_claims import cycle
goal:
  The disk_pressure chop and every module on the cycle import cleanly from a fresh
  interpreter.
size: small
decisions:
  chop_guard:
    ask: Should the regression guard fresh-import every sase_chop_* script module?
    choices:
      all_chops:
        ~37 subprocess probes (~12 s serial); any job's import crash fails CI, not a
        digest
      cycle_only:
        Probe only this cycle's modules; the next latent job cycle surfaces only in
        digests
    default: all_chops
    why:
      Scheduled jobs fail silently into hourly digests; a per-job import smoke test is
      cheap
    answer: all_chops
proposed_by: bbugyi200.athena.1f5
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 08:46:55
status: wip
---

# Fix the `sase.running_field` ↔ `sase.core.agent_launch_claims` import cycle

## Problem

Every hourly run of the `housekeeping/disk_pressure` chop has failed since
`ff0baf13e9 perf(axe): cheap hooks-lane jobs and fewer spawns` (2026-11-21). So
disk-pressure housekeeping is not running at all. The job crashes at import time:

```
ImportError: cannot import name 'allocate_and_claim_workspace_from_content' from partially
initialized module 'sase.core.agent_launch_claims' (most likely due to a circular import)
```

The import chain is:

```
sase.scripts.sase_chop_disk_pressure
 → sase.core.disk_footprint → disk_footprint_inventory → disk_footprint_inventory_collect
 → sase.workspace_provider.inventory
 → sase.core.agent_launch_claims            (starts executing)
 → sase.running_field._model                (runs package __init__ first!)
 → sase/running_field/__init__.py            (eagerly imports _claim, _hold, _query, ...)
 → sase.running_field._claim
 → from sase.core.agent_launch_claims import allocate_and_claim_workspace_from_content  ✗ partially initialized
```

The cycle has existed since `c563582e20` (May 2026). In that commit,
`sase.core.agent_launch_claims` started importing `WorkspaceClaim` from
`sase.running_field._model`. Until yesterday it was hidden: on every real entry point,
`sase.running_field` was already fully imported before `agent_launch_claims`, through
the eager package re-exports of `sase.monitor`, `sase.procs`, and `sase.agent.hijack`.
`ff0baf13e9` made those packages lazy (PEP 562), which removed that ordering and exposed
the cycle. `git bisect` confirms `ff0baf13e9` as the first bad commit for
`import sase.scripts.sase_chop_disk_pressure`.

Currently failing imports from a fresh interpreter on master:

- `sase.core.agent_launch_claims`
- `sase.workspace_provider.inventory`
- `sase.workspace_provider.occupancy_conflicts`
- `sase.scripts.sase_chop_disk_pressure`, which backs the `sase_chop_disk_pressure` and
  `sase_job_disk_pressure` console scripts

All other 55 `[project.scripts]` entry-point modules import cleanly today. The failure
is deterministic, so this is not a flake.

## Fix

### 1. Make `sase.running_field` resolve its re-exports lazily

Rewrite `src/sase/running_field/__init__.py` with the same PEP 562 pattern `ff0baf13e9`
used for `src/sase/monitor/__init__.py`:

- Keep the module docstring that describes the RUNNING field format. Add one sentence:
  package-level names resolve lazily, so that importing a leaf submodule such as
  `sase.running_field._model` does not pull in the claim/release graph. This also
  prevents the cycle with `sase.core.agent_launch_claims`.
- `from __future__ import annotations`, `from typing import TYPE_CHECKING`, and
  `from sase._lazy_exports import lazy_dir, lazy_getattr`.
- `_LAZY_EXPORTS: dict[str, tuple[str, str]]` that maps each of the 23 names currently
  in `__all__` to its `(submodule, attr)` source. The sources are exactly the current
  eager imports:
  - `_claim`: `claim_next_axe_workspace`, `claim_workspace`
  - `_claim_labels`: `OPERATIONAL_LEASE_CLAIM_PREFIX`,
    `is_operational_lease_claim_workflow`, `operational_lease_claim_workflow`
  - `_hold`: `hold_workspace_claim`
  - `_model`: `ClaimResult`, `WorkspaceClaim`, `WorkspaceClaimError`
  - `_occupant`: `describe_workspace_occupant`, `format_workspace_occupant`,
    `pid_is_alive`
  - `_query`: `get_claimed_workspaces`
  - `_release`: `release_workspace`
  - `_rename`: `update_running_field_cl_name`
  - `_transfer`: `transfer_workspace_claim`
  - `_workspace`: `claim_next_axe_workspace_dir`, `find_runner_numbered_workspace`,
    `get_first_available_axe_workspace`, `get_first_available_workspace`,
    `get_workspace_directory`, `get_workspace_directory_for_num`,
    `runner_has_placeholder_workspace`
- An `if TYPE_CHECKING:` block that keeps the current imports so type checkers and IDEs
  still resolve the names.
- Keep `__all__` unchanged, plus `__getattr__`, `__dir__`, and the
  `_PACKAGE_GETATTR = __getattr__` / `_PACKAGE_DIR = __dir__` aliases. Copy the monitor
  package's shape exactly, including those aliases, so symvision treats it the same way.

This fixes the bug at its root. `sase.running_field._model` is a leaf module (only `re`
and `dataclasses`), and `workspace_provider/inventory.py`, `occupancy_conflicts.py`, and
`main/project_handler_lifecycle.py` already import it directly, expecting it to be
cheap. After this change it actually is. The 40+ `from sase.running_field import X`
callers and the existing `patch("sase.running_field.X")` tests keep working, because
`lazy_getattr` resolves names on first access. The `sase.monitor` and `sase.procs`
facades already depend on this behavior.

A prototype of exactly this `__init__` was checked in a throwaway worktree. All four
failing imports above then succeed, and every name in `sase.running_field.__all__` still
resolves.

Do not use the alternative of a function-local import in `agent_launch_claims.py`. It
fixes only this one edge and leaves `running_field/_model` imports expensive and cyclic.

### 2. Regression tests in `tests/test_chop_import_budget.py`

Use the existing `_run_probe` helper (fresh `sys.executable -c` subprocess):

- **Cycle test.** One probe that imports, in a fresh interpreter,
  `sase.core.agent_launch_claims` _first_, and then `sase.workspace_provider.inventory`
  and `sase.workspace_provider.occupancy_conflicts`. Add separate probes that each
  import one of those modules first, so that every edge order is covered. Each probe
  then asserts that `sase.running_field.claim_workspace` still resolves.
- **Leaf-stays-leaf test.** `import sase.running_field._model` and assert that
  `sase.running_field._claim` and `sase.core.agent_launch_claims` are **not** in
  `sys.modules`.
- **Disk-pressure entry point.** `import sase.scripts.sase_chop_disk_pressure` succeeds.
  The `chop_guard` test below already covers this; include it explicitly if you choose
  `cycle_only`.

> [!decision] chop_guard = all_chops Add a parametrized test that discovers every
> `sase_chop_*` module under `sase.scripts` (via
> `pkgutil.iter_modules(sase.scripts.__path__)`, filtered by the `sase_chop_` prefix;
> assert the discovered list is non-empty) and runs `_run_probe(f"import {module}")` for
> each one. That way any future import-time crash in a scheduled job fails CI instead of
> appearing in an hourly error digest. Keep the file under 500 lines; if it would exceed
> that, put the new tests in a new `tests/test_chop_entrypoint_imports.py`.

> [!decision] chop_guard = cycle_only Skip the parametrized all-chops probe and add only
> the cycle, leaf, and disk-pressure probes above.

Confirm the new cycle/disk-pressure tests **fail** on the unfixed tree (stash step 1 or
run them first) before applying step 1, then pass after.

### 3. Unblock test collection: `tests/test_held_settle_evidence.py`

This is an unrelated break on current master that `just check` will hit.
`2ede612ca7 test: split stale running cleanup monitor claims tests` moved
`_make_leaked_monitor` and `_run_cleanup_sweep` into
`tests/_stale_running_cleanup_monitor_claims_retention.py`. However,
`tests/test_held_settle_evidence.py` (line ~32) still imports them from
`tests.test_stale_running_cleanup_monitor_claims`, so the whole module fails to collect
with `ImportError`. Point that import at
`tests._stale_running_cleanup_monitor_claims_retention`. It is a one-line change, and
both helpers exist there.

## Verification

1. Fresh-interpreter smoke checks, run from the repo root with the project venv:
   ```bash
   for m in sase.core.agent_launch_claims sase.workspace_provider.inventory \
            sase.workspace_provider.occupancy_conflicts sase.scripts.sase_chop_disk_pressure \
            sase.running_field; do python -c "import $m" || echo "FAIL $m"; done
   ```
2. Run the new and touched tests: `tests/test_chop_import_budget.py`,
   `tests/test_held_settle_evidence.py`, and the running-field / workspace-claim tests
   (`pytest -k "running_field or workspace_claim"`).
3. Before finishing, read the `lint_and_test` reference memory
   (`sase memory read lint_and_test.md -r "..."`) and run `just check` as it directs. If
   symvision complains about the lazy facade, read the `symvision` reference memory and
   compare with how `src/sase/monitor/__init__.py` passes.

No Rust/`sase-core` changes are needed. This is purely Python import structure. No docs
or memory notes describe `sase.running_field`'s import behavior, so no docs or memory
updates are needed.

## Out of scope

- The installed `sase` tool runs from the primary checkout as an editable install. Once
  this lands on master and the primary checkout syncs, the next hourly `disk_pressure`
  run picks up the fix. No reinstall is needed.
