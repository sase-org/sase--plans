---
tier: epic
title: Stop pytest writing to the production bead store, and purge the leaked beads
goal: 'A sase test run can no longer mutate any bead store outside its own pytest
  sandbox, the tests that were silently targeting the shared plans sidecar resolve
  to isolated stores instead, a standing check proves the production store is byte-identical
  across a full suite run, and the leaked pytest-fixture beads are removed from the
  shared store.

  '
phases:
- id: sandbox
  title: Pytest sandbox containment for bead-store resolution
  depends_on: []
  size: medium
  description: '''Pytest sandbox containment'' section: publish the pytest sandbox
    root, stop checkout-marker and primary-workspace discovery from escaping it inside
    pytest, and repair the shared bead test fixtures that currently resolve to the
    real sidecar store.

    '
- id: guard
  title: Deny-by-default bead-store write guard
  depends_on:
  - sandbox
  size: medium
  description: '''Deny-by-default bead-store write guard'' section: extend the existing
    pytest state-write guard with a bead-store rule, wire it into every bead write
    chokepoint including the Rust CLI fast path, and repair every remaining test the
    armed guard refuses.

    '
- id: soak
  title: Standing soak check and documentation
  depends_on:
  - guard
  size: small
  description: '''Standing soak check and documentation'' section: add a repeatable
    check that a full suite run leaves the production bead store byte-identical, and
    document the guard and its environment contract.

    '
- id: purge
  title: Verify and purge the leaked fixture beads
  depends_on:
  - soak
  size: small
  description: '''Verify and purge the leaked beads'' section: re-identify the leaked
    pytest-fixture beads against the live store, remove them atomically, and confirm
    the open backlog and ready queue contain only real work.

    '
create_time: 2026-07-25 10:56:23
status: done
bead_id: sase-9l
---

- **PROMPT:** [202607/prompts/bead_store_pytest_isolation.md](prompts/bead_store_pytest_isolation.md)

# Plan: Stop pytest writing to the production bead store, and purge the leaked beads

Implements recommendation 1 of the 2026-07-25 beads leverage research report ("Stop the test suite writing to the
production bead store, and purge the leaked beads", finding F1).

## Context and verified evidence

The sase pytest suite creates beads in the **real, shared** bead store — the plans sidecar clone at
`sase/repos/plans/beads` inside whatever workspace runs the tests, which auto-commits and pushes to
`git@github.com:sase-org/sase--plans.git`. The corruption is durable, shared across every workspace, and still active.

Verified against the live store while writing this plan (`sase bead show`, plus a scan of
`sase/repos/plans/beads/issues.jsonl`, 2,025 records):

| Bead      | Title                         | Design (PLAN) recorded                                | Originating test                                                                       |
| --------- | ----------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `sase-97` | `Created Epic`                | in-checkout pytest tmp path                           | `tests/test_bead/test_cli_golden.py` `create` case                                     |
| `sase-9a` | `Created`                     | `plan.md`                                             | `tests/test_bead/test_cli_auto_commit.py::test_handle_bead_create_auto_commit_message` |
| `sase-9b` | `Epic`                        | `plan.md`                                             | `tests/test_bead/test_cli_changespec.py::test_create_plan_accepts_changespec_and_bug`  |
| `sase-9c` | `Epic`                        | `sdd/plans/202605/roadmap.md`                         | `test_create_plan_stores_sibling_workspace_plan_path_relative_to_primary`              |
| `sase-9d` | `Epic`                        | absolute path into a sibling workspace's pytest cache | `test_create_plan_preserves_external_absolute_plan_path`                               |
| `sase-9e` | `Created`                     | `plan.md`                                             | same as `sase-9a`, later run                                                           |
| `sase-9f` | `Epic`                        | `plan.md`                                             | same as `sase-9b`, later run                                                           |
| `sase-9g` | `Epic`                        | `sdd/plans/202605/roadmap.md`                         | same as `sase-9c`, later run                                                           |
| `sase-9h` | `Epic`                        | absolute path into a sibling workspace's pytest cache | same as `sase-9d`, later run                                                           |
| `sase-9i` | `Epic` (`model: claude/opus`) | `plan.md`                                             | `test_create_with_model_persists`                                                      |
| `sase-9j` | `Created Epic`                | in-checkout pytest tmp path                           | same as `sase-97`, later run                                                           |

That is **eleven** live leaked beads, not the five the research report snapshotted — three separate suite runs leaked,
the most recent roughly two hours after the report was written. All eleven are `open`, `tier: epic`, `type: plan`, so
`sase bead ready` offers them to any agent running `#bd/next`. Two older fixture beads (`sase-8q`, `sase-8s`) were
previously closed rather than removed, and their IDs are permanently consumed.

`git log` on the plans sidecar confirms the leak is committed and pushed: `chore(beads): create sase-9f` through
`chore(beads): create sase-9j` are real commits on the shared sidecar.

## Root cause (verified, and different from the report's hypothesis)

The research report attributes the leak to test helpers not clearing inherited `SASE_SDD_PLANS_DIR` /
`SASE_SDD_BEADS_DIR`. **That is not the cause.** Those variables are only ever _written_ (`src/sase/sdd/env.py:44-47`)
for launched agents; no bead-store resolution path reads them. Do not build the fix around clearing them.

The actual cause is a resolution path that the test fixtures do not patch:

1. `sase.bead.cli_common.resolve_beads_location()` is the single resolver behind `get_project()`,
   `find_beads_location()`, `auto_commit_bead_store()`, and the Rust CLI fast path.
2. It calls `_resolve_workspace_context()`, whose **first** strategy is `_resolve_workspace_context_from_marker()` — a
   direct call to `sase.workspace_provider.marker.find_marker_from_cwd`, which walks up from the CWD looking for a
   managed checkout's `.sase/checkout.json`.
3. The bead test fixtures (`tests/test_bead/conftest.py::project_dir`, `test_cli_golden.py::_run_cli`, and the per-file
   copies in `test_cli_changespec.py`) isolate by `monkeypatch.chdir(tmp_path)` plus
   `monkeypatch.setattr("sase.bead.workspace.resolve_primary_workspace", lambda: None)`. That patch is consulted
   _third_. The marker strategy runs first and is never patched.
4. When pytest's `basetemp` lives **inside** a managed checkout, the walk-up from `tmp_path` finds the checkout's real
   marker, resolution yields the workspace's real plans sidecar, and every fixture write lands in production.

Reproduced read-only from a workspace checkout, with `SASE_HOME` redirected and `PYTEST_CURRENT_TEST` set — that is,
with the suite's own isolation fully in force:

```text
resolve_beads_location(cwd=<in-checkout tmp dir>)  with project_dir-style patch
  -> <workspace>/sase/repos/plans/beads   (storage: sidecar_repos)   # the production store
```

This also explains the timeline. The leaked designs reference `.localtmp/…` and `.pytest_cache/tmp/…`, both
**in-checkout** pytest basetemps. Relocating pytest scratch off `/tmp` (in-flight work under bead `sase-96`) is what
moved `basetemp` inside the checkout and turned a latent hole into an active one.

Two further facts that shape the fix:

- **Containment alone is insufficient.** With marker discovery neutralized, the same probe still resolved to a store
  outside the sandbox (`/home/bryan/sdd/beads`) via the legacy walk-up fallback. Every unbounded ancestor walk in the
  resolver is a candidate leak path, so a deny-by-default write guard is the load-bearing fix and containment is the
  ergonomic one.
- **The Python `BeadProject` API is not the only write path.** `src/sase/main/bead_fast_path.py` hands `write_beads_dir`
  straight to the Rust `bead_cli_execute` binding and mutates entirely in Rust, bypassing `BeadProject`. Guarding only
  `BeadProject` would miss the fast path — which is exactly the path `sase bead create` takes.

## Precedent to follow

Closed bead `sase-8g.11` ("Keep tests out of production state") built `src/sase/core/state_write_guard.py`: a
pytest-context detector plus `assert_test_state_write_isolated()` / `best_effort_test_state_write_allowed()`, wired into
telemetry, axe state, axe logs, and the session registry. It refuses writes at or below the OS account's real `~/.sase`.
**Bead stores are not under `~/.sase`** — they live in per-workspace sidecar clones — so that rule never covered them.
Extend that module rather than inventing a parallel one; its tests are in `tests/test_state_write_guard.py` and its
user-facing description is `docs/development.md:47-54`.

Upstream `gastownhall/beads` shipped the same class of guard on 2026-07-24 (`50003b803`, "refuse test DBs on production
dolt servers (AD-01)").

## Shared design contract

All phases build against this contract; the earlier phases must land these exact names so the later ones can rely on
them.

**Environment variables**

| Name                                 | Set by                                   | Meaning                                                                                                                                                                            |
| ------------------------------------ | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SASE_PYTEST_SANDBOX_DIR`            | the pytest harness (`tests/conftest.py`) | Absolute path of the sandbox root. Every bead-store write a test performs must land at or below it.                                                                                |
| `SASE_ALLOW_UNSANDBOXED_BEAD_WRITES` | a specific test, deliberately            | Set to `1` to opt one process out of the guard. Expected to have **zero** uses when this epic lands; it exists so a future genuine exception does not require reverting the guard. |

**Guard API** — added to `src/sase/core/state_write_guard.py`, reusing its existing `pytest_context_detected()`:

```python
def assert_bead_store_write_sandboxed(
    beads_dir: str | os.PathLike[str],
    *,
    operation: str,
    environ: Mapping[str, str] | None = None,
) -> None:
    """Raise when a pytest process would mutate a bead store outside its sandbox."""
```

Semantics, deny-by-default:

- Not a pytest process (`pytest_context_detected()` is false) → return; production behavior is unchanged.
- `SASE_ALLOW_UNSANDBOXED_BEAD_WRITES=1` → return.
- `SASE_PYTEST_SANDBOX_DIR` unset or empty → **refuse**. A pytest process that cannot prove where its sandbox is has not
  earned a write. This is what makes the guard cover test-spawned subprocesses.
- Target resolves at or below the sandbox root → allow. Otherwise → **refuse**.
- Refusal raises (do not warn-and-continue). A silently skipped bead write produces a confusing downstream assertion
  failure; a raise names the offending store, the operation, and the sandbox root, and says how to fix the test.
- Resolve both sides with `Path(...).expanduser().resolve(strict=False)` before comparing, matching the existing
  `_resolve_path` / `_is_at_or_below` helpers in the module.

Reuse `_PytestStateIsolationError` or a sibling `RuntimeError` subclass so `pytest.raises(RuntimeError)` keeps working
across both guards.

## Pytest sandbox containment

Phase `sandbox`. Stops the dominant leak vector and keeps the suite green before the guard is armed.

1. **Publish the sandbox root.** In `tests/conftest.py`, add a session-scoped autouse fixture that sets
   `SASE_PYTEST_SANDBOX_DIR` in `os.environ` (not `monkeypatch`, which is function-scoped) to
   `str(tmp_path_factory.getbasetemp())`, and clears it on teardown. It must be a real environment variable so
   subprocesses a test spawns inherit it. Under xdist each worker publishes its own basetemp, which is correct: a worker
   only ever writes beneath its own.
2. **Bound checkout-marker discovery inside pytest.** `find_marker_from_cwd` must not return a marker found _above_
   `SASE_PYTEST_SANDBOX_DIR` when `pytest_context_detected()` is true. Prefer implementing the bound in
   `src/sase/workspace_provider/marker.py` so it also applies to test-spawned subprocesses, rather than as a monkeypatch
   that only covers in-process calls. Markers a test plants inside its own `tmp_path` must keep working —
   `tests/test_bead/test_cli_auto_commit.py`, `tests/main/test_bead_fast_path.py`, and
   `tests/workspace_provider/test_checkout_marker.py` depend on that and are the regression check for this step.
3. **Apply the same bound to primary-workspace resolution.** `sase.bead.workspace.resolve_primary_workspace()` and
   `sase.bead.cli_common._resolve_workspace_context()` reach the same real workspace by other routes (the
   `~/.sase/projects` scan and the workspace-provider plugin). Under pytest, a resolved primary that is outside the
   sandbox must be treated as "not found" so resolution falls through instead of silently selecting production.
4. **Repair the leaking fixtures.** Make them isolate positively rather than by patching one of three strategies:
   - `tests/test_bead/conftest.py::project_dir` — the shared fixture behind most bead CLI tests.
   - `tests/test_bead/test_cli_golden.py::_run_cli`.
   - `tests/test_bead/test_cli_changespec.py::project_dir` and the three tests there that build their own primary
     (`test_create_plan_stores_sibling_workspace_plan_path_relative_to_primary`,
     `test_create_plan_preserves_external_absolute_plan_path`, `test_create_with_model_persists`).

   Preferred shape: one shared helper that plants a checkout marker in the test's own tmp workspace and points every
   resolution strategy at it, so the resolver takes its normal production path but lands inside the sandbox. That is
   strictly better than patching `resolve_primary_workspace`, which is what failed here.

5. **Verify** by re-running the probe from the root-cause section: from an in-checkout tmp directory, with the fixtures
   applied, `resolve_beads_location()` must return a path under the sandbox. Then run
   `just test tests/test_bead tests/main/test_bead_fast_path.py tests/workspace_provider` and confirm the production
   store's `issues.jsonl` is byte-identical before and after (`git status` in `sase/repos/plans` must stay clean and no
   new `chore(beads):` commit may appear).

## Deny-by-default bead-store write guard

Phase `guard`. The load-bearing fix. Land it armed.

1. **Implement** `assert_bead_store_write_sandboxed()` in `src/sase/core/state_write_guard.py` per the shared design
   contract, with unit tests in `tests/test_state_write_guard.py` covering: non-pytest process allowed; sandboxed target
   allowed; unsandboxed target refused with the store path, operation, and sandbox root in the message; missing
   `SASE_PYTEST_SANDBOX_DIR` refused; explicit override allowed.
2. **Wire every write chokepoint.**
   - `src/sase/core/bead_mutation_facade.py` — every mutating entry point: `init_store`, `create`, `update`,
     `claim_for_agent_launch`, `claim_for_agent_wait`, `release_agent_claim`, `preclaim_epic_work`, `close`,
     `remove_many`, `add_dependency`, `mark_ready_to_work`, `unmark_ready_to_work`, `export_jsonl`. (`remove` delegates
     to `remove_many`; guarding the delegate is sufficient.) Guard before the `require_rust_binding(...)` call, passing
     the operation name for the error message. A small private helper keeps this from becoming thirteen copies of the
     same three lines.
   - `src/sase/main/bead_fast_path.py` — guard `context.write_beads_dir` before invoking `bead_cli_execute`, for
     mutating verbs only. The Rust dispatch in `crates/sase_core/src/bead/cli.rs:65-96` is authoritative: mutating verbs
     are `create`, `open`, `update`, `close`, `dep`, `rm`; read verbs are `list`, `show`, `search`, `ready`, `blocked`,
     `stats`. Confirm that list against the checkout before relying on it. Reads must stay unguarded so read-only tests
     keep passing.
3. **Sweep and repair.** With the guard armed, run the fast, slow, and visual suites. Every refusal is a test that was
   writing to production; fix each one by isolating it, not by setting the override. 48 test files touch bead APIs, so
   budget for a real sweep, but most route through the fixtures phase `sandbox` already repaired. Record in the phase
   bead notes any test that genuinely needs `SASE_ALLOW_UNSANDBOXED_BEAD_WRITES` and why — the expected count is zero.
4. **Keep the Rust boundary intact.** This guard is a Python-test-harness concern, not shared domain behavior: it keys
   off `PYTEST_CURRENT_TEST`, which the Rust core has no notion of (there is no pytest reference anywhere in
   `crates/sase_core/src`), and no non-pytest frontend needs to match it. It belongs in this repo alongside its
   `sase-8g.11` precedent. Do not add it to `sase-core`. If a future Rust-only frontend gains its own bead writes, an
   AD-01-style guard there is a separate follow-up.

## Standing soak check and documentation

Phase `soak`. `sase-8g.11` guarded metrics and axe logs and this surface still regressed; without a standing check it
will regress again.

1. **Soak check.** Add a `tools/` script plus a `Justfile` recipe that records the production bead store's
   `issues.jsonl` digest and the sidecar's git HEAD, runs the suite, and fails loudly if either moved. Keep it out of
   the default `just check` path if it is slow; the value is that it is one documented command, runnable on demand and
   from CI.
2. **Unit-level regression.** Add a test asserting that a `resolve_beads_location()` result outside the sandbox is
   refused by the wired chokepoints — so the wiring itself, not just the primitive, is covered.
3. **Documentation.** Extend the pytest safety-boundary paragraph in `docs/development.md:47-54` to cover bead stores
   and state the sandbox rule. Add `SASE_PYTEST_SANDBOX_DIR` and `SASE_ALLOW_UNSANDBOXED_BEAD_WRITES` to the environment
   table in `docs/configuration.md` alongside the existing `SASE_TEST_GATE_*` entries.

## Verify and purge the leaked beads

Phase `purge`. Last, so the store cannot be re-polluted between cleanup and the fix landing.

1. **Re-identify — do not trust this plan's ID list.** The store moves; more beads may have leaked since. Scan
   `sase/repos/plans/beads/issues.jsonl` for the fixture signature: non-closed, `tier: epic`, `type: plan`, no children,
   generic title (`Epic`, `Created`, `Created Epic`), and a `design` that is either a temp/pytest path (matching
   `pytest`, `popen-gw`, `.pytest_cache`, `localtmp`, `/tmp`) or one of the fixture literals `plan.md` /
   `sdd/plans/202605/roadmap.md`. Real epics have descriptive titles and `design` paths under
   `sase/repos/plans/YYYYMM/`; that is the discriminator.
2. **Verify each candidate individually** with `sase bead show <id>` immediately before removing it, and confirm it has
   no children and no dependents. Do not remove anything whose title or design looks like real work.
3. **Remove atomically** with a single `sase bead rm <id> [<id> ...]` — multi-remove is supported and validates every ID
   before mutating. The removal auto-commits and pushes to the shared sidecar, which is intended.
4. **Confirm.** `sase bead ready` must return only real work; re-run the signature scan and expect zero matches. Record
   the removed IDs in the phase bead notes.

The eleven IDs known at planning time, as a starting point only: `sase-97`, `sase-9a`, `sase-9b`, `sase-9c`, `sase-9d`,
`sase-9e`, `sase-9f`, `sase-9g`, `sase-9h`, `sase-9i`, `sase-9j`.

## Out of scope

- `sase bead doctor` validation of PLAN paths pointing into temp directories (research report recommendation 4). It is
  the right standing detector for this failure mode but is separate, additive work.
- Reopening or reclaiming the IDs consumed by the previously closed fixture beads `sase-8q` and `sase-8s`.
- Read-path hermeticity. This epic stops pytest _writing_ to production stores. Tests can still _read_ one, which makes
  them non-hermetic but does not corrupt anything; treat it as a follow-up.
- The in-flight `sase-96` work relocating pytest scratch. This epic must be correct whether `basetemp` is inside or
  outside the checkout, not depend on which.

## Definition of done

- A full suite run leaves `sase/repos/plans/beads/issues.jsonl` byte-identical and adds no `chore(beads):` commit to the
  plans sidecar.
- A test that attempts a bead write outside the sandbox fails with an actionable error naming the store and the sandbox
  root.
- `SASE_ALLOW_UNSANDBOXED_BEAD_WRITES` has zero uses in the tree.
- `sase bead ready` contains no pytest fixture beads.
- `just check` passes. Run `just install` first — workspace virtualenvs go stale between sessions.
