---
tier: tale
title: Break the sase.agent to sase.monitor import cycle crashing the AXE lumberjack
goal:
  Every module under `src/sase/agent/` imports cleanly in a cold interpreter because no
  module there imports the `sase.monitor` package at module scope, so the `run_every`
  lumberjack stops crash-looping on the partially-initialized
  `sase.agent.launch_admission_store` import, and a static import-boundary gate plus
  cold-import probes fail closed if the edge is ever reintroduced.
size: medium
proposed_by: bbugyi200.athena.0k8
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0k8](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0k8.md)
  - [bbugyi200.athena.sase-zs.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zs.1/README.md)
  - [bbugyi200.athena.sase-zs.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zs.2/README.md)
  - [bbugyi200.athena.sase-zs.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zs.3/README.md)
  - [bbugyi200.athena.sase-zs.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zs.4/README.md)
- **COMMITS:**
  - [63653c5](https://github.com/sase-org/sase/commit/63653c5ee1593d8deef0aa6890639af8f79ccfdb)
    — fix(sdd): reuse primary sidecar clone references
  - [4ea8a15](https://github.com/sase-org/sase/commit/4ea8a1531b366d644522ddd7795d6cc8cb878ea1)
    — fix(sdd): retry remote clone timeouts
  - [186c543](https://github.com/sase-org/sase/commit/186c543d0e79e0518a7da20a96754dd7295ea3da)
    — feat(github): add retryability classifier facade
  - [ecea389](https://github.com/sase-org/sase/commit/ecea389efd48ff04d3ab054c99496748299f3f9d)
    — feat(sdd): stream network git progress

# Plan: Break The `sase.agent` -> `sase.monitor` Import Cycle

## Context

The AXE `run_every` lumberjack crash-loops on startup (3 failures within 60s, exit code
1), which raises an `axe`/`crash-loop` notification every restart episode. The failure
is a hard `ImportError`, not a data or environment problem:

```
sase axe lumberjack run run_every
  -> sase/axe/lumberjack.py  run() -> _run_tick()
  -> sase/axe/_chop_lifecycle_runner.py  finalize_launched_chop_runs()
  -> sase/axe/_chop_lifecycle_typed_admission.py  typed_admission_reconciliation()
     -> _read_admission_receipt()
        -> (function-local) from sase.agent.launch_admission_store import ...
           -> sase/agent/launch_admission_store.py line 18:
              from sase.monitor.transaction import write_json_marker_atomic
              -> sase/monitor/__init__.py line 22   -> .resume
              -> sase/monitor/resume.py line 38     -> .followup
              -> sase/monitor/followup.py line 40   -> .continuation_delivery
              -> sase/monitor/continuation_delivery.py line 12
                                                    -> .continuation_admission
              -> sase/monitor/continuation_admission.py line 17:
                 from sase.agent.launch_admission_store import ADMISSION_DIRNAME, ...
ImportError: cannot import name 'ADMISSION_DIRNAME' from partially initialized module
'sase.agent.launch_admission_store' (most likely due to a circular import)
```

### Root cause

`sase.agent.launch_admission_store` reaches into the `sase.monitor` package for one
generic helper, `write_json_marker_atomic`. Python cannot import a submodule without
first executing its package `__init__`, so that single line pulls in the whole
`sase/monitor/__init__.py` re-export block, which transitively reaches
`sase.monitor.continuation_admission`, which imports back from the still-executing
`sase.agent.launch_admission_store`. Because `ADMISSION_DIRNAME` and its sibling
constants are defined _after_ the offending import line, the name does not exist yet and
the import fails.

The cycle only detonates when `sase.agent.launch_admission_store` is the module that
_enters_ it. `import sase.monitor` on its own succeeds, because by the time
`continuation_admission` runs, `launch_admission_store` starts from scratch and its
`sase.monitor.transaction` import is satisfied from a `sase.monitor` entry that is
already in `sys.modules`. That is exactly why this survived in the tree: the ordinary
monitor and TUI entry points import `sase.monitor` first and never see it. The AXE
lumberjack tick does not import `sase.monitor` at all (793 `sase.*` modules are loaded
at that point and `sase.monitor` is not among them), so its function-local import of
`launch_admission_store` enters the cycle from the failing side on every single tick.

Verified against a clean tree at `14c665e15`:

- `python -c "import sase.agent.launch_admission_store"` fails with the traceback above.
- `python -c "import sase.agent.direct_typed_launch"` fails the same way (same line-29
  `sase.monitor.transaction` import).
- `python -c "import sase.axe.lumberjack; import sase.agent.launch_admission_store"`
  reproduces the lumberjack crash exactly.
- `import sase.monitor`, `import sase.axe`, `import sase.monitor.continuation_admission`
  and `import sase.axe.chop_typed_admission` all succeed, which is why no existing test
  catches this.

### The direction the layering already runs

`sase.monitor` imports `sase.agent` at module scope in seven places
(`continuation_admission`, `followup` x3, `handoff`, `supervise`, `spawn`), so
`sase.agent` is the lower layer and must not import the `sase.monitor` package back.
There are exactly five module-scope violations of that today, and every one of them
imports the same single name:

| File                                           | Line |
| ---------------------------------------------- | ---- |
| `src/sase/agent/launch_admission_store.py`     | 18   |
| `src/sase/agent/launch_admission.py`           | 63   |
| `src/sase/agent/launch_admission_engine.py`    | 56   |
| `src/sase/agent/launch_condition_workspace.py` | 14   |
| `src/sase/agent/direct_typed_launch.py`        | 29   |

`write_json_marker_atomic` has no monitor-specific knowledge: it creates the parent
directory, writes JSON to a `.<name>.<pid>.<ns>.tmp` sibling, fsyncs, and `os.replace`s
it into place. It lives in `sase/monitor/transaction.py` only for historical reasons —
it was first written for the `.monitor_go` / `.monitor_started` markers. Relocating it
to a neutral low-level module removes every `sase.agent` -> `sase.monitor` module-scope
edge in one move and leaves no residual cycle.

### Note on the existing band-aid

`src/sase/axe/chop_typed_admission.py` lines 17-21 already carry a comment explaining
that a top-level `launch_admission_store` import "would cycle", and both that module and
`src/sase/axe/_chop_lifecycle_typed_admission.py` import `launch_admission_store` from
inside functions to dodge it. Deferring the import did not remove the cycle; it moved
the cycle's entry point into a code path that runs on every lumberjack tick and turned a
latent problem into a crash loop. That comment is also factually wrong in its premise —
`sase.axe.__init__` does not import `chop_typed_admission` at all. The band-aid must go
with the cycle, otherwise the next reader re-derives the same wrong conclusion.

## Invariants

- `write_json_marker_atomic` keeps its exact signature, atomic-publish semantics, and
  temp-sibling naming. This is a module move, not a behavior change: no caller's on-disk
  output may differ.
- One definition only. Do not leave a re-export shim in `sase.monitor.transaction`; a
  second import path is how this edge grows back.
- `sase.monitor.transaction`'s other exports (`MONITOR_GO_MARKER`,
  `MONITOR_LAUNCH_BARRIER_TIMEOUT_SECONDS`, `MONITOR_STARTED_MARKER`,
  `MONITOR_START_ACK_TIMEOUT_SECONDS`, `monitor_go_path`, `monitor_lane_lock_path`,
  `monitor_started_path`) stay exactly where they are.
- `sase.monitor_state` and `sase.monitor_status` are separate top-level modules, not the
  `sase.monitor` package. `sase.agent` imports them legitimately and the new gate must
  not flag them.

## Implementation

1. Add `src/sase/core/atomic_json.py` and move `write_json_marker_atomic` there verbatim
   from `src/sase/monitor/transaction.py`. It imports only `json`, `os`, `time`, and
   `pathlib`, so it adds no package-init cost to anything. Give it a module docstring
   saying it is the shared atomic JSON marker writer for launch-admission and monitor
   coordination, and rewrite the function docstring, which currently claims the helper
   is "Shared by the `.monitor_go` launch barrier and the `.monitor_started` startup
   acknowledgement" — it is now also the launch-admission sidecar/receipt writer and the
   condition-workspace marker writer. Export it in `__all__`.

   Put it in its own module rather than in `src/sase/core/atomic_temp.py`: that module
   is the reaper for stale `.<name>.*.tmp` siblings and carries no JSON knowledge, and
   keeping writer and reaper separate matches how the rest of `sase/core/` is split.

2. Delete `write_json_marker_atomic` from `src/sase/monitor/transaction.py`, including
   its `__all__` entry. Its `json`, `os`, and `time` imports are used by nothing else in
   that module — the three remaining path helpers use only `hashlib.sha256`, `Path`, and
   `sase.core.paths.sase_projects_dir` — so all three go too, leaving `transaction.py`
   as pure path/constant surface.

3. Repoint all eight call sites at `sase.core.atomic_json`:
   - The five `sase/agent/` modules listed in the table above.
   - `src/sase/monitor/supervise.py` — it imports the name inside a grouped
     `from .transaction import (...)`; leave the other three names in that group alone.
   - `src/sase/monitor/diagnostics.py` line 15.
   - `src/sase/axe/chop_typed_admission.py` line 295, a function-local import inside
     `_resolve_clan_dispatch_payload` — it no longer needs to be function-local for
     cycle reasons, so hoist it to module scope with the rest.

   `tests/test_launch_condition_workspace.py` and
   `tests/test_launch_condition_workspace_admission_failures.py` patch
   `sase.agent.launch_condition_workspace.write_json_marker_atomic`, i.e. the importing
   module's own attribute. Those patch targets stay correct and must not be changed.

4. Remove the stale cycle comment at `src/sase/axe/chop_typed_admission.py` lines 17-21
   and hoist the `launch_admission_store` imports in both AXE modules to module scope,
   so the dependency is declared once instead of being re-derived in six function
   bodies:
   - `src/sase/axe/chop_typed_admission.py` lines 51, 255, 314 (`admission_dir`,
     `UNITS_DIRNAME`, `read_json`).
   - `src/sase/axe/_chop_lifecycle_typed_admission.py` lines 77, 164, 178
     (`admission_dir`, `read_json`, `RECEIPT_FILENAME`, `SIDECAR_FILENAME`).

   Leave `_read_typed_admission_payload`'s function-local
   `sase.agent.launch_request_response` import alone — it sits inside a deliberate
   `try/except` and is not part of this cycle.

5. Fold `src/sase/monitor/continuation_admission.py` line 142's function-local
   `from sase.agent.launch_admission_store import read_journal` into the module-scope
   import block at line 17. That import is already redundant — `_read_journal_entries`
   defers a name from a module the file has already imported eagerly at line 17 — and
   leaving it behind implies a constraint that does not exist.

## Tests

Add `tests/test_agent_monitor_import_boundary.py`, following the shape of the two
existing precedents in this repo:
`tests/workspace_provider/ test_primary_writable_store_import_boundary.py` for the AST
gate and `tests/test_chop_import_budget.py` for the subprocess probes.

1. **Static boundary gate.** Walk every `src/sase/agent/**/*.py`, parse it with `ast`,
   and collect module-scope `Import`/`ImportFrom` nodes (anything not nested inside a
   function or class body) that resolve to the `sase.monitor` package. Resolve relative
   imports too, so a future `from ..monitor import x` is caught. Match only
   `sase.monitor` exactly or a `sase.monitor.` prefix, so `sase.monitor_state` and
   `sase.monitor_status` stay legal. Assert the resulting set is empty, with a failure
   message naming the offending files and modules.

   Use no allowlist. The docstring should state the rule and its reason: `sase.monitor`
   imports `sase.agent` at module scope, so the reverse edge at module scope is a cycle,
   and the sanctioned escape hatch — if one is ever genuinely needed — is a
   function-local import.

2. **Cold-import probes.** Two `subprocess.run([sys.executable, "-c", ...])` probes, in
   the style `_run_probe` already uses in `tests/test_chop_import_budget.py`:
   - `import sase.agent.launch_admission_store` succeeds in a fresh interpreter and
     `ADMISSION_DIRNAME` is readable off it. This is the exact statement that fails
     today.
   - `import sase.axe.lumberjack` followed by `import sase.agent.launch_admission_store`
     succeeds and leaves `sase.monitor` out of `sys.modules`. This is the lumberjack
     crash path verbatim, and the `sys.modules` assertion is what would catch a
     reintroduced eager edge that the AST gate cannot see (for example, one added under
     `sase.core`).

Both probes were run against a scratch copy of `src/sase` carrying the module move and
the `sase/agent` + `sase/axe` call-site repointing, and both pass:
`import sase.monitor`, `import sase.axe`, and `import sase.agent.direct_typed_launch`
keep working, and the lumberjack path's `launch_admission_store` import drops from
pulling in 300 extra `sase.*` modules (~180ms) to 2. Hoisting the AXE lazy imports to
module scope (step 4) was verified in the same scratch copy, including that it leaves
`tests/test_chop_import_budget.py`'s `sase.chops.sdk` closure unchanged.

## Verification

Run `just install` first — ephemeral workspaces can have drifted deps — then
`just check` inline. This change edits the import graph across `sase/agent`,
`sase/monitor`, `sase/axe`, and adds a module under `sase/core`, which is squarely in
the broadening set, so also run `just check-full` before landing and run it only through
the `/sase_monitor` skill with the `TESTING` / `TESTED` status pair, never inline.

Expect `symvision` to stay clean without a pragma: `write_json_marker_atomic` keeps all
eight non-test importers at its new home, and no symbol is orphaned by the move.

## Non-goals

- **Do not make `sase/monitor/__init__.py` lazy.** A PEP 562 `__getattr__` would also
  break the cycle and would save the ~180ms/300-module hit that any first touch of
  `sase.monitor` costs, but it is a much larger change to a package with a long
  `__all__`, and it would hide rather than fix the layering inversion. If it is worth
  doing, it is separate work.
- **Do not try to break the `sase.monitor` <-> `sase.axe` cycle.** It is real
  (`monitor/logs.py`, `monitor/proc_adapter.py`, `monitor/resume.py`,
  `monitor/host_completion_state.py` all import `sase.axe` at module scope) but it
  resolves correctly today and is not what crashes the lumberjack.
- **No Rust core change.** Per the core-backend-boundary rule, the litmus test is
  whether another frontend would need the behavior to match the TUI.
  `write_json_marker_atomic` is a local filesystem primitive with no wire or API
  surface, and this change moves a Python module without altering behavior, so nothing
  belongs in the sibling core repo.
- **No feature flag.** Nothing user-reaching changes and there is no old branch that has
  to stay reachable; the only removed surface is one internal Python import path, whose
  in-repo callers are all updated in the same change.

## Done when

- `sase axe lumberjack run run_every` starts and ticks without the `ImportError`, and no
  new `axe`/`crash-loop` notification is raised for that lane.
- `grep -rn "^from sase\.monitor" src/sase/agent/` returns only `sase.monitor_state` and
  `sase.monitor_status` hits.
- `write_json_marker_atomic` is defined exactly once, in `src/sase/core/atomic_json.py`.
- `tests/test_agent_monitor_import_boundary.py` passes, and fails if the
  `sase.agent.launch_admission_store` line-18 import is restored.
- `just check` is clean, and `just check-full` is clean before landing.
