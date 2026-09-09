---
tier: epic
title: Restore a green CI signal on sase master
goal: "CI on sase master passes reliably: the sase_core_rs import defect can no longer
  poison a pytest-xdist worker, and the two racy ACE TUI tests assert against real
  synchronization points instead of wall-clock luck.

  "
phases:
  - id: modguard
    title:
      Keep the core extension parent and submodule in sync across sys.modules patches
    depends_on: []
    size: medium
    description:
      "modguard: give the six test modules that swap or evict
      sys.modules['sase_core_rs'] a shared helper that moves the compiled submodule with
      its parent package, and pin the re-import invariant with a regression test."
  - id: apptitle
    title: Anchor the ACE title-refinement tests to the mount-loads sync point
    depends_on: []
    size: small
    description:
      "apptitle: replace the single pilot.pause() in the two on-mount title tests with a
      wait on _mount_state_loads_done, the barrier the title write actually sits behind."
  - id: detailrace
    title: Make the rapid-navigation detail test drive the debouncer deterministically
    depends_on: []
    size: small
    description:
      "detailrace: stop assuming two keypresses land inside the 150 ms debounce window;
      control the timer the way the existing debouncer tests do and assert on the
      coalesced result."
  - id: coreinit
    title: Harden the generated sase_core_rs package init upstream
    depends_on: []
    size: small
    description:
      "coreinit: fix the maturin star-import template in the sase-core repo so the
      package init binds the extension module explicitly instead of relying on an
      importlib side effect."
  - id: verify
    title: Confirm the restored signal end to end
    depends_on:
      - modguard
      - apptitle
      - detailrace
      - coreinit
    size: small
    description:
      "verify: run the exhaustive local gate, re-run the failing legs under adverse
      conditions, and confirm a full CI run is green."
proposed_by: bbugyi200.athena.u6
status: done
bead_id: sase-gg
create_time: 2026-09-09 19:50:05
---

- **PROMPT:**
  [prompts/202608/ci_green_restore.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ci_green_restore.md)
- **BEAD:**
  [sase-gg](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gg/README.md)

# Plan: Restore a green CI signal on sase master

## Background

`actstat` reports `sase-org/sase` red on master. Four consecutive CI runs failed, and
the failures are **four unrelated classes**, not one regression. Three are real defects
in this repo's dependency chain and test suite; one is a GitHub outage that needs no
code change.

Observed runs (all on master, 2026-08-06):

| Run         | Failing job                                                    | Class                               |
| ----------- | -------------------------------------------------------------- | ----------------------------------- |
| 31113741579 | `test (3.12)` coverage leg                                     | detail-panel race                   |
| 31113753459 | `build-core`                                                   | GitHub outage                       |
| 31114984919 | `test (3.12)` (27 tests), `test (3.13)`, plus 3 setup failures | import defect + title race + outage |
| 31116699976 | `build-core`                                                   | GitHub outage                       |
| 31118652934 | `build-core`, `published-core-minimum-smoke`                   | cancelled behind the outage         |

### Class 1 — GitHub Actions outage (no code change; do not "fix")

`build-core`, `published-core-minimum-smoke`, `perf-floors`, and `coverage-contexts`
failed during **step 1, "Set up job"** — before any repo step runs:

```
Getting action download info
Failed to resolve action download info. Error: Service Unavailable
Retrying in 21.635 seconds
Failed to resolve action download info. Error: Service Unavailable
Retrying in 13.266 seconds
##[error]Service Unavailable
```

This is GitHub's action-resolution service returning 503. The runner already retried
twice on its own. Nothing in `.github/workflows/ci.yml` causes it and nothing in this
repo can prevent it. Because every downstream job declares `needs: build-core`, one 503
reds the whole run — that cascade is correct behaviour, not a defect.

**Action: none.** Re-run the affected workflows once the other phases land. Do not add
retry scaffolding or file a bead for this class.

### Class 2 — `sase_core_rs` package init breaks on re-import (root cause, reproduced)

Run 31114984919's `test (3.12)` coverage leg failed **27 tests** at once, all with the
same error raised from inside the installed wheel:

```
E   NameError: name 'sase_core_rs' is not defined
.venv/lib/python3.12/site-packages/sase_core_rs/__init__.py:3: NameError
```

The shipped `__init__.py` is maturin's star-import template:

```python
from .sase_core_rs import *

__doc__ = sase_core_rs.__doc__
if hasattr(sase_core_rs, "__all__"):
    __all__ = sase_core_rs.__all__
```

Line 3 reads a bare `sase_core_rs` that the file never binds. It normally resolves only
because importlib, after loading a submodule, does
`setattr(parent_package, child_name, child_module)` — and the parent package's attribute
namespace _is_ this file's global namespace.

That `setattr` lives on the cache-miss path. When the submodule is already in
`sys.modules`, the import returns from cache, the `setattr` never runs, and the name is
unbound. So **re-executing `__init__.py` while `sase_core_rs.sase_core_rs` stays cached
raises `NameError`**, permanently breaking every later import in that process.

Reproduced deterministically against the current wheel:

```
$ .venv/bin/python -c "
import sys, importlib
import sase_core_rs
del sys.modules['sase_core_rs']          # evict parent, keep submodule cached
importlib.import_module('sase_core_rs')"
NameError: name 'sase_core_rs' is not defined
```

Six test modules create exactly that window. Each swaps or evicts the **parent** package
while leaving the compiled submodule cached:

- `tests/test_core_health.py` (`_install_fake_extension`, `_force_import_failure`)
- `tests/test_core_rust.py`
- `tests/test_core_vcs_log.py`
- `tests/test_core_git_query.py`
- `tests/test_core_agent_scan_facade.py`
- `tests/core_agent_scan_helpers.py`

All use `monkeypatch.setitem(sys.modules, RUST_EXTENSION_MODULE_NAME, fake)` or
`monkeypatch.delitem(sys.modules, RUST_EXTENSION_MODULE_NAME, raising=False)`. Both
record the _parent's_ prior value only. Whichever teardown ordering leaves the parent
absent while `sase_core_rs.sase_core_rs` is still cached poisons the worker, and the
next ~27 tests scheduled onto it all fail at import.

**Why it is intermittent, and why it is not a flake to retry.** `pytest-randomly` is not
installed, so collection order is stable — but pytest-xdist assigns tests to workers
dynamically, so _which_ tests share a process with the poisoning sequence changes run to
run. Running the six modules together in one process in collection order passes today;
the defect only surfaces under a worker split that CI happens to produce. It will keep
recurring until fixed.

### Class 3 — `test_on_mount_refines_title_to_resolved_version` races the mount loads

```
E   AssertionError: assert 'sase ace (v0.15.0)' == 'sase ace (v0.15.0+9.gdeadbee.dirty)'
tests/ace/tui/test_app_title.py:136
```

The test monkeypatches `resolved_app_version`, mounts `AceApp`, awaits **one**
`pilot.pause()`, and asserts the refined title. But the title write is the _last_
statement of `_run_mount_state_loads`
(`src/sase/ace/tui/actions/_startup_loads.py:146-148`), behind four sequential
`asyncio.to_thread` hops — prompt-stash counts, changespecs from disk, last selection,
and the query save. A single event-loop pause cannot join four worker threads. On an
idle dev machine it usually wins; on a loaded CI runner it loses.

The loader already publishes the correct barrier: `self._mount_state_loads_done` is set
in that coroutine's `finally` (`_startup_loads.py:150`). Existing code waits on it —
`src/sase/ace/testing/ace_page.py:168`,
`tests/ace/tui/test_startup_stopwatch_live_update.py:234`, and
`tests/ace/tui/test_residual_freeze_soak.py:307`. This test simply does not.

Its sibling `test_on_mount_keeps_initial_title_when_resolver_returns_none` has the same
single-pause shape. It never fails, because it asserts the title is _unchanged_ — so it
passes for the wrong reason whenever the loader has not finished. Fixing only the
failing test would leave a test that cannot detect the regression it exists to catch.

### Class 4 — `test_rapid_navigation_loads_only_the_final_detail` races the debouncer

```
E   AssertionError: assert ['default:row-1...'] == ['default:row-2...']
tests/ace/tui/test_artifacts_files_detail.py:261
```

`pane.selected_entry is rows[2]` passed, so navigation was correct — the _detail loader_
fired for the intermediate row. The test is:

```python
await page.press("j", "j")
await page.wait_for(lambda _state: bool(calls))
assert pane.selected_entry is rows[2]
assert calls == [rows[2].id]
```

`DetailPanelDebouncer` coalesces on a 150 ms timer (`DEFAULT_DEBOUNCE_S`,
`src/sase/ace/tui/util/debounce.py`). The test assumes both keypresses land inside one
window. When the gap exceeds 150 ms the first `j`'s timer fires for row-1, and
`bool(calls)` — which is true as soon as _any_ load starts — returns immediately on that
intermediate value.

The failing leg is the coverage leg, the slowest lane in CI (24m56s versus 15m04s
uncovered), which is exactly where a >150 ms inter-keypress gap is plausible. Both
observed occurrences of this test failing were on that leg.

The predicate is under-specified: it waits for _a_ call rather than for the coalesced
result. Retrying, or widening the assertion to accept row-1, would discard the contract
the test exists to prove.

## Phases

### modguard — Keep the core extension parent and submodule in sync across sys.modules patches

Make this repo immune to the Class 2 defect regardless of which wheel is installed. This
phase alone unblocks CI; it does not wait on `coreinit`.

1. Add one shared helper — near `RUST_EXTENSION_MODULE_NAME`'s test-side users, in a
   `tests/` helper module importable by all six call sites — that patches or evicts
   `sys.modules[RUST_EXTENSION_MODULE_NAME]` **and**
   `sys.modules[f"{RUST_EXTENSION_MODULE_NAME}.{RUST_EXTENSION_MODULE_NAME}"]` as a
   unit, through `monkeypatch` so teardown restores both together. With the compiled
   submodule absent, any re-import re-executes `__init__.py` on the cache-miss path, the
   `setattr` runs, and the name binds.
2. Route all six modules listed above through it. Keep each test's existing observable
   behaviour — this is a containment change, not a rewrite.
3. Add a regression test asserting the invariant the six modules kept breaking: after
   the helper's teardown, `import sase_core_rs` still succeeds and
   `sase_core_rs.sase_core_rs` is bound. Assert the _helper's_ contract, not the wheel's
   robustness, so this test stays green on both the current wheel and the `coreinit`
   one.
4. Prove the fix reaches the reported failure. Construct the poisoning order directly —
   evict the parent via the old pattern, confirm the `NameError`, then confirm the new
   helper does not produce it. A green ordinary run is not evidence here; the defect
   needs a specific worker split.

Do not paper over this with an autouse fixture that re-imports the module after every
test. That hides the leak instead of removing it and slows every test.

### apptitle — Anchor the ACE title-refinement tests to the mount-loads sync point

In `tests/ace/tui/test_app_title.py`, replace the single `await pilot.pause()` with a
bounded wait on `app._mount_state_loads_done` in **both** on-mount tests —
`test_on_mount_refines_title_to_resolved_version` and
`test_on_mount_keeps_initial_title_when_resolver_returns_none`. Follow the established
idiom at `src/sase/ace/testing/ace_page.py:168`: poll with `pilot.pause()` against a
deadline and fail with a clear message on timeout, rather than looping unbounded.

Fixing the sibling is deliberate: it currently passes for the wrong reason, and leaving
it would keep a blind spot in the exact behaviour this file covers.

Do not raise a sleep or add a retry decorator. The barrier already exists; the tests
only need to use it.

### detailrace — Make the rapid-navigation detail test drive the debouncer deterministically

Rewrite `test_rapid_navigation_loads_only_the_final_detail` so the coalescing contract
is proven without depending on how fast the runner delivers two keypresses.

The repo already has the deterministic pattern: capture or control the timer instead of
racing it — see `tests/ace/tui/test_detail_panel_debouncer.py` (fake app recording
`set_timer` callbacks) and
`tests/ace/tui/test_plugins_browser_pane_sase_update.py:196-201` (monkeypatched
`app.set_timer`). Either shape is acceptable:

- widen the pane debouncer's delay for the test so both presses provably land in one
  window, assert `calls == []` immediately after the presses to prove the intermediate
  row was suppressed, then let the pending callback run and assert
  `calls == [rows[2].id]`; or
- intercept `set_timer` so the test fires the coalesced callback itself.

Preserve the assertion that the intermediate row-1 detail is **never** loaded — that is
the whole point of the test. Do not relax the assertion to `calls[-1] == rows[2].id`,
and do not replace the predicate with a sleep.

### coreinit — Harden the generated sase_core_rs package init upstream

Fix the true root cause where it lives. The wheel is built from the sibling Rust core
repo, so per the Rust-core boundary rule this change does not belong in the sase repo.

Open the core repo with the `/sase_repo` skill and use only the path it prints — do not
clone it, and do not read it over the web. In its Python package init (the source of the
installed `sase_core_rs/__init__.py`), bind the extension module explicitly instead of
relying on the importlib `setattr` side effect:

```python
from . import sase_core_rs
from .sase_core_rs import *
```

Keep the existing `__doc__` and `__all__` re-exports byte-compatible; this is a
robustness fix, not an API change. Verify with the reproduction above: after
`del sys.modules['sase_core_rs']`, re-importing must succeed.

Then follow that repo's normal release path so a corrected wheel is published. This
phase is independent of `modguard` — `modguard` protects this repo now, `coreinit` stops
the defect at its source for every other consumer.

### verify — Confirm the restored signal end to end

1. Run `just install`, then `just check-full` — the whole-repo lint gates plus the full
   suite. `check-full` is required here rather than `just check`: this epic touches
   shared test helpers, so a diff-scoped selection is not a sufficient gate.
2. Exercise the two repaired TUI tests under adverse timing, not just a quiet run.
   Confirm `apptitle` still passes when the mount loads are slow and `detailrace` still
   passes when the two presses are spaced beyond 150 ms — under the original bug each of
   those conditions produced the reported failure.
3. Re-run the previously failing legs, including the 3.12 coverage leg, and confirm a
   full CI run is green.

If a `Set up job` / `Service Unavailable` failure reappears, that is Class 1. Re-run the
job; do not treat it as a regression from this epic and do not change workflow files in
response.

## Out of scope

- Retry or resilience scaffolding in `.github/workflows/ci.yml` for the GitHub outage.
  The failure precedes every repo-controlled step.
- Broader refactoring of the six `sys.modules`-patching test modules beyond routing them
  through the shared helper.
- Changing `DEFAULT_DEBOUNCE_S`. 150 ms is a product decision about j/k navigation feel;
  the bug is in the test's synchronization, not the value.
