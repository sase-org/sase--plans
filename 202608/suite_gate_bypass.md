---
tier: tale
title: Close the ungoverned suite-gate bypass that oversubscribed the host
goal:
  "A parallel pytest run can no longer take worker capacity the shared token pool cannot
  see: an exemption is honoured only when a real ancestor lease corroborates it, and a
  deliberate SASE_TEST_GATE_DISABLED bypass is bounded by the host budget and announces
  itself."
proposed_by: bbugyi200.athena.v7
create_time: 2026-08-07 19:14:02
status: wip
---

# Plan: Close the ungoverned `SASE_TEST_GATE_DISABLED` bypass in the pytest suite gate

## Problem

The development host reached a load average of **97.6 on 64 cores** while CPU pressure
was effectively zero. The load was entirely I/O wait caused by memory oversubscription
from concurrent pytest fleets, and the mechanism that exists to prevent exactly this —
the host-global worker-token pool in `tests/_suite_gate.py` — was bypassed.

### Evidence captured during the incident

Load and process state (64 logical CPUs):

```
load average: 97.60, 107.99, 85.88
procs_running 12      procs_blocked 29
ps state counts: 643 S, 376 I, 40 D, 12 R
```

Pressure stall information — this is not a CPU shortage:

```
/proc/pressure/cpu     some avg10=0.36  full avg10=0.00
/proc/pressure/io      some avg10=48.36 full avg10=43.32
/proc/pressure/memory  some avg60=12.94 avg300=27.59
```

Memory and swap:

```
Mem:   62Gi total, 44Gi used, 10Gi buff/cache
Swap:  63Gi total, 25Gi used
vmstat si: 54096 -> 70444 KiB/s sustained swap-in
```

Concurrent pytest controllers, one per sibling workspace clone, all running the fast
lane at the same time:

| Controller             | Workers | Gate environment                                          |
| ---------------------- | ------- | --------------------------------------------------------- |
| contention harness run | `-n 64` | `SASE_PYTEST_WORKERS=64`, `SASE_TEST_GATE_DISABLED=1`     |
| full-lane run          | `-n 24` | governed                                                  |
| scoped/gear run        | `-n 4`  | `SASE_TEST_GATE_DISABLED=1` + `SASE_TEST_GATE_GOVERNED=1` |
| fast-lane run          | `-n 4`  | `SASE_TEST_GATE_DISABLED=1` + `SASE_TEST_GATE_GOVERNED=1` |

The shared pool state at the same moment:

```
/tmp/sase-pytest-tokens-<uid>/pool.lock -> {"capacity": 32, "explicit": false}
32 token files exist; 24 were held.
```

Roughly 100 pytest worker processes were alive, totalling ~32 GiB RSS with ~25 GiB
pushed to swap.

### Root cause

The pool worked correctly for every run that used it. `capacity` was 32 and the governed
runs collectively held 24 tokens — they fit. The `-n 64` controller held **zero** tokens
while consuming 64 workers' worth of memory. At the 0.74–0.85 GiB of live worker RSS
that `_MEMORY_KIB_PER_WORKER` is calibrated against, that surplus alone is ~48–54 GiB on
a 62 GiB host, which is precisely the swap storm observed.

The bypass lives in `tools/run_pytest._parallel_worker_grant()`:

```python
def _parallel_worker_grant() -> tuple[int, WorkerTokenLease | None]:
    budget, capacity_is_explicit = configured_token_budget()
    exact_worker_count = _configured_worker_count()
    gate_disabled = os.environ.get("SASE_TEST_GATE_DISABLED") == "1"

    if exact_worker_count is not None:
        if gate_disabled:
            return exact_worker_count, None   # unbounded, no lease, no tokens
        ...
    else:
        floor, ceiling = automatic_worker_range(budget)
        if gate_disabled:
            return ceiling, None              # bounded at 28, still no tokens
```

`budget` is computed and then discarded. Neither branch takes a token, so the demand is
invisible to every other run's view of the pool.

The deeper flaw is that `SASE_TEST_GATE_DISABLED=1` currently carries two irreconcilable
meanings, and the code cannot tell them apart:

1. **A statement of exemption.** A process that actually holds a lease sets the variable
   via `WorkerTokenLease._set_descendant_environment()` so its own child pytest and
   xdist workers do not double-acquire. The ancestor's lease already paid for this
   demand.
2. **A claim of exemption.** Anybody can export the same variable at top level and take
   the whole machine while the pool believes nothing is happening.

`tools/run_pytest` already names this distinction in a comment on the scoped gear path
("the belt-and-braces flag would be a claim of exemption rather than a statement of
one"), but `_parallel_worker_grant()` accepts the claim.

The distinction cannot be drawn from the environment as the code stands today, because
only _some_ leases mark their descendants governed:

- `_parallel_worker_grant()` and `engage_scoped_gear()` construct their lease with
  `governed=True`, so descendants get both `SASE_TEST_GATE_DISABLED=1` and
  `SASE_TEST_GATE_GOVERNED=1`.
- `configure_suite_gate()` constructs its lease **without** `governed=True`, so its
  descendants get `SASE_TEST_GATE_DISABLED=1` alone — indistinguishable from a top-level
  bypass.

Closing that gap is what makes the fix possible.

## Goal

Make an unaccounted parallel pytest run impossible to start by accident, while keeping a
deliberate, explicit bypass available.

After this change:

- A run whose exemption is _corroborated_ by a real ancestor lease keeps working exactly
  as it does today, with no waiting and no token acquisition.
- A top-level `SASE_TEST_GATE_DISABLED=1` still bypasses the pool (it never waits), but
  its width is bounded by the host budget and it announces itself.
- Asking for more workers than the host budget permits fails with the existing,
  actionable `_fit_request_to_budget` error instead of silently succeeding.

## Approach

### 1. Make every real lease mark its descendants governed

In `tests/_suite_gate.py`, `configure_suite_gate()` currently builds its lease without
`governed=True`. Change it so that it does. Once every genuinely-held lease sets
`SASE_TEST_GATE_GOVERNED=1`, that variable becomes a reliable signal meaning "an
ancestor's lease already accounts for my demand".

With no remaining caller that wants an ungoverned lease, delete the `governed`
constructor parameter and the `self._governed` branch in
`_set_descendant_environment()`, so a lease unconditionally sets both variables. Fewer
states beats a flag with one live value.

### 2. Split "corroborated exemption" from "deliberate bypass"

Add one helper to `tests/_suite_gate.py`, exported for `tools/run_pytest`:

```python
def descendant_exemption() -> bool:
    """Report whether an ancestor's lease already accounts for this demand."""
    return (
        os.environ.get(_GOVERNED_ENV) == "1"
        or "PYTEST_XDIST_WORKER" in os.environ
    )
```

Update `_is_gate_exempt()` to be expressed in terms of it:

```python
def _is_gate_exempt() -> bool:
    return descendant_exemption() or os.environ.get(_DISABLED_ENV) == "1"
```

`_is_gate_exempt()` keeps honouring the bare disable flag — `configure_suite_gate()` is
the in-pytest safety net and a caller who disabled the gate should not have it
re-acquire underneath them. The width bounding happens in `tools/run_pytest`, which is
the only place that _chooses_ a width.

### 3. Bound and announce the deliberate bypass in `tools/run_pytest`

Rewrite `_parallel_worker_grant()` so the two meanings take different paths:

```python
def _parallel_worker_grant() -> tuple[int, WorkerTokenLease | None]:
    budget, capacity_is_explicit = configured_token_budget()
    exact_worker_count = _configured_worker_count()

    if descendant_exemption():
        # An ancestor's lease already paid for this width. Grant it untouched
        # and take no tokens: acquiring here would double-count the demand.
        if exact_worker_count is not None:
            return exact_worker_count, None
        return automatic_worker_range(budget)[1], None

    bypassing = os.environ.get("SASE_TEST_GATE_DISABLED") == "1"

    if exact_worker_count is not None:
        floor = ceiling = exact_worker_count
        exact = True
    else:
        floor, ceiling = automatic_worker_range(budget)
        exact = False

    if bypassing:
        # A deliberate, uncorroborated bypass. It never waits for tokens, but
        # it is still bounded by the host budget and it says so, because the
        # pool cannot see it and every other run's budget assumes it is absent.
        _, granted = _fit_request_to_budget(
            floor,
            ceiling,
            budget,
            exact=exact,
            capacity_is_explicit=capacity_is_explicit,
        )
        print(
            f"suite gate bypassed: running {granted} ungoverned workers "
            f"against a {budget}-token host pool "
            "(SASE_TEST_GATE_DISABLED=1)",
            file=sys.stderr,
        )
        return granted, None

    lease = WorkerTokenLease(...)  # unchanged
    granted = lease.acquire(floor, ceiling, exact=exact)
    lease.make_inheritable()
    return granted, lease
```

The implementing agent should use whatever expression of `_fit_request_to_budget` reads
cleanly — the sketch above is about behaviour, not about that call's exact shape.
`_fit_request_to_budget` is currently private to `tests/_suite_gate.py`; export it (or a
thin wrapper) so `tools/run_pytest` can reuse it rather than reimplementing the clamp
and the error text.

The important behaviours are:

- An **exact** over-budget request (`SASE_PYTEST_WORKERS=64` with a 32-token budget)
  raises the existing `pytest.UsageError`, whose message already prescribes the correct
  remedy: "Reduce SASE_PYTEST_WORKERS/-n or increase SASE_TEST_GATE_SLOTS deliberately."
- An **automatic** bypass clamps to the budget instead of taking
  `automatic_worker_range` at face value.
- Either way the bypass prints one line to stderr naming the width and the pool.

This keeps the sanctioned escape hatch that `_timeout_message()` advertises ("set
`SASE_TEST_GATE_DISABLED=1` only to bypass the pool deliberately") while removing its
ability to oversubscribe the host by 3x.

### 4. Give the legitimate over-budget case a supported route

A contention or scaling benchmark genuinely needs to run wider than the standing budget.
Under this change its route is the one the error message already names: set
`SASE_TEST_GATE_SLOTS` explicitly to the intended host capacity. That makes the capacity
change explicit and pool-visible (`configured_token_budget()` returns
`capacity_is_explicit=True`, and `_synchronize_capacity` propagates it), so concurrent
runs see the enlarged pool instead of being blindsided by invisible demand.

Document this in the `SCOPED_WORKER_REMEDY`-adjacent constants in `tools/run_pytest` or
in the bypass banner text, so the next person to hit the new `UsageError` is told what
to do without reading the source.

## Files to change

- `tests/_suite_gate.py` — `configure_suite_gate()` lease becomes governed; drop the
  `governed` parameter and its branch; add `descendant_exemption()`; re-express
  `_is_gate_exempt()`; export `_fit_request_to_budget` (or a wrapper) for reuse.
- `tools/run_pytest` — rewrite `_parallel_worker_grant()` per above; import the new
  helpers; add the bypass banner and the remedy text.
- `tests/test_suite_gate.py`, `tests/test_run_pytest_workers.py`,
  `tests/test_suite_gate_integration.py` — extend as below.

## Tests

Add cases that pin the newly-separated behaviours. The existing fixtures in
`tests/_run_pytest_fixtures.py` already clear `SASE_PYTEST_WORKERS` and
`SASE_TEST_GATE_DISABLED`, so they are the right starting point.

1. **The incident, as a regression test.** With a budget of 32, `SASE_PYTEST_WORKERS=64`
   and `SASE_TEST_GATE_DISABLED=1` and no `SASE_TEST_GATE_GOVERNED`,
   `_parallel_worker_grant()` raises `pytest.UsageError` mentioning
   `SASE_TEST_GATE_SLOTS`. This is the case that produced load 97.6 and must fail
   loudly.
2. **Corroborated exemption still grants freely.** The same environment plus
   `SASE_TEST_GATE_GOVERNED=1` returns 64 with no lease and takes no tokens.
3. **`PYTEST_XDIST_WORKER` is also corroboration.** Same as (2) with
   `PYTEST_XDIST_WORKER=gw0` instead.
4. **Automatic bypass clamps.** No `SASE_PYTEST_WORKERS`, `SASE_TEST_GATE_DISABLED=1`,
   budget below `_DEFAULT_AUTOMATIC_CEILING`: the grant equals the budget, not the
   ceiling.
5. **Explicit slots unlock the wide run.** `SASE_TEST_GATE_SLOTS=64` with
   `SASE_PYTEST_WORKERS=64` succeeds, confirming the supported route for benchmarks.
6. **Every held lease marks descendants governed.** After `configure_suite_gate()`
   acquires, both `SASE_TEST_GATE_DISABLED` and `SASE_TEST_GATE_GOVERNED` are set, and
   `unconfigure_suite_gate()` restores both. This is the invariant the whole fix rests
   on; without it, descendants of the in-pytest gate would be misread as top-level
   bypasses and would start queueing for tokens they already hold.
7. **The scoped gear still refuses under a governed ancestor.** `engage_scoped_gear()`
   continues to return `REFUSED_GATE_DISABLED` when `SASE_TEST_GATE_DISABLED=1` is
   inherited, unchanged by this work.

## Verification

- `just check` for the lint gates and the diff-scoped lane.
- `just check-full` before landing: this change touches the suite gate itself, which is
  the broadening set by definition — every parallel test run in the repo goes through
  the modified code path.
- Confirm by hand that a full lane still acquires a lease and that
  `/tmp/sase-pytest-tokens-<uid>/pool.lock` reports the expected capacity afterwards.

## Risks and trade-offs

- **The escape hatch becomes bounded.** `SASE_TEST_GATE_DISABLED=1` currently grants any
  width; afterwards it is capped at the host budget unless `SASE_TEST_GATE_SLOTS` is
  raised. This is the intended behaviour change and the point of the tale, but it will
  break any in-flight branch that relies on the unbounded form — the contention harness
  observed during this incident is exactly such a caller, and it must move to
  `SASE_TEST_GATE_SLOTS`. Reviewers should confirm they want bounding rather than a hard
  refusal of the ungoverned bypass.
- **Making `configure_suite_gate()`'s lease governed changes descendant environments.**
  Any code that distinguishes the two flags today must be re-checked;
  `tests/_test_selection_gear.py` reads `SASE_TEST_GATE_DISABLED` and is the one known
  reader, and its behaviour is unchanged because it keys off the disable flag that both
  lease kinds already set.
- **No change to the budget maths.** `_DEFAULT_HARD_TOKEN_LIMIT`,
  `_MEMORY_KIB_PER_WORKER`, and `_RESERVED_CPU_DIVISOR` are deliberately untouched. The
  pool's arithmetic was correct during the incident; only its enforcement was optional.

## Out of scope

- A second, unrelated I/O contributor was observed on the same host:
  `/etc/cron.hourly/hourly_backup` had been running for 51 minutes with an `rm -rf`
  against the `/mnt/hercules` array, pinning `md0` at 98.8% utilisation for 19 of them,
  so an hourly job was overrunning its hour. Only a single instance was running, and it
  sat on a different device from the swap storm, so it contributed negligibly to the
  load average. It is host configuration rather than repository code and belongs in a
  separate task bead against the dotfiles repo.
- Retuning worker memory reservation or the hard token cap.
- Any change to test selection, the scoped gear's ceiling, or `just selection-health`.
