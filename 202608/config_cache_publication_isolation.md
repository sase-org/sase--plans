---
tier: tale
title:
  Make the process-global merged-config cache publish atomically so its nodes stop
  failing the flake gate
goal:
  "`tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`
  and its sibling nodes in that file pass in isolation, under file-scoped contention,
  and in a full parallel lane, because ACE TUI teardown no longer abandons a
  config-polling `sase-ace-proc-observer` thread into the next test, the merged-config
  and owner-snapshot caches publish atomically instead of as two unsynchronized writes,
  and a leftover refresh worker can no longer deregister a live one. `just
  selection-health --fail-on-new-flake` names at most the `sase-n4`-owned fakey
  usage-limit node.

  "
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.6.6.1
bead: sase-ns.6.6.6.1
create_time: 2026-08-17 09:33:44
status: wip
---

- **PARENT:**
  [202608/backlog_top_five_gates_and_flakes.md](backlog_top_five_gates_and_flakes.md)
- **BEAD:**
  [sase-ns.6.6.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.6.1.md)

# Plan: Make the process-global merged-config cache publish atomically

## Bead

This plan implements phase bead `sase-ns.6.6.6.1` (`configcache`) of epic
`sase-ns.6.6.6`, plan `202608/backlog_top_five_gates_and_flakes.md`, which owns task
bead `sase-mv`. Read the `Phase configcache — bead sase-mv` section of that epic plan
before starting; this plan does not restate its evidence, only what has been learned
since.

## What is failing, and why it cannot be a single-threaded bug

`tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`
fails in the whole-suite parallel lane and passes in isolation. Line 251 is the failing
assertion:

```python
now[0] += config_core._CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS + 0.01

assert load_merged_config() is first          # <-- fails: not the same object
second = _wait_for_new_merged_config(first)
```

Trace the single-threaded path and this assertion cannot fail. At line 241 the autouse
`_clear_config_caches` fixture has just drained the token cache, so
`current_config_token()` takes the synchronous first-fill branch
(`src/sase/config/core.py:188`), publishes token `T_a`, and sets the deadline to the
_patched_ monotonic value `10.75`. `load_merged_config()` then stores the pair
`(_merged_config_cache_token=T_a, _merged_config_cache_value=first)`. At line 251 the
patched clock reads `10.76`, so `current_config_token()` takes the
stale-while-revalidate branch: it starts a refresh worker and returns `T_a` _while still
holding `_current_config_token_cache_lock`_ (`src/sase/config/core.py:199-211`). The
worker cannot publish before that call returns, and even when it does publish it only
touches the token cache. `load_merged_config()` compares the `T_a` it already captured
against `_merged_config_cache_token`, matches, and returns `first`.

So the assertion can only fail if, between lines 241 and 251, **another thread in the
same process** either replaced `_merged_config_cache_value`, or nulled it via
`clear_config_cache()`. The same argument explains the other nodes in this file:
`test_load_merged_config_caches_plugin_layer` asserts `call_count["n"] == 1`, which only
breaks if `_plugin_configs_cache` was reset mid-test, and
`test_load_merged_config_eventually_invalidates_on_file_mtime_change` has the identical
shape as the named node.

This matters because it says the fix is not in the test and not in the token cache. The
token cache is already correct: it is fully serialized by
`_current_config_token_cache_lock`, and `3a22ff04f` (bead `sase-nv`) already bound it to
the `CONFIG_DIR` object it was computed against and added an epoch guard so a leftover
worker cannot publish into a successor's generation.

## Root cause: an abandoned poller, against a cache with no publication discipline

Two things have to be true for these nodes to fail, and both are defects this phase
owns. A foreign thread has to be running config work inside the test — that is **Defect
3**, and it is the proximate cause. And the process-global caches it runs against have
to be unguarded enough for that to do damage — that is **Defects 1 and 2**.
`src/sase/config/core.py` protects `_current_config_token_cache_*` with an `RLock` and
an epoch, and protects nothing else.

Fix Defect 3 and the named node stops failing. Fix Defects 1 and 2 and the cache stops
being corruptible by any concurrent reader, in production as well as in tests. Do both:
Defect 3 alone leaves a data race that a future poller will find again, and Defects 1
and 2 alone do not stop a legitimate concurrent rebuild from replacing the object the
test holds.

### Defect 1 — the merged-config cache publishes a pair as two unsynchronized writes

`load_merged_config()` (`src/sase/config/core.py:628-658`) reads and writes
`_merged_config_cache_token` and `_merged_config_cache_value` with no lock at all:

```python
token = current_config_token()
if _merged_config_cache_value is not None and _merged_config_cache_token == token:
    return _merged_config_cache_value
...
_merged_config_cache_token = token          # published first
_merged_config_cache_value = result         # published second
```

`get_agent_owner_config_snapshot()` (`src/sase/config/core.py:547-559`) has the
identical shape for `_agent_owner_config_cache_*`. Three concurrent failures fall out,
and only the third is a test-only symptom:

1. **Torn read.** A reader that observes the new token but the old value returns a
   config that does not correspond to the token it matched, and — because the pair now
   looks consistent — keeps returning it until the token changes again. This is a
   production correctness bug, not a test artifact.
2. **Stale clobber.** A slow builder for token `T_old` that finishes after a fast
   builder published `T_new` overwrites the slot with `(T_old, old_result)`. Every
   subsequent reader holding `T_new` then misses and rebuilds, so the cache degrades
   into a permanent miss. This is the shape of a merge-count explosion, and the
   repository already budgets that count: `tests/perf/baselines/test_cost_baseline.json`
   pins `config_load_merged` at 4924.
3. **Object replacement.** Any concurrent reader that rebuilds swaps
   `_merged_config_cache_value` for a different dict object. The failing nodes assert
   object identity, so this is exactly what turns them red.

### Defect 2 — a leftover refresh worker deregisters the live one

`_refresh_current_config_token()` (`src/sase/config/core.py:150-167`) correctly refuses
to publish a token when its `cache_epoch` no longer matches, but then clears the worker
slot **unconditionally**:

```python
with _current_config_token_cache_lock:
    if cache_epoch == _current_config_token_cache_epoch:
        ...                                        # correctly skipped when stale
    _current_config_token_refresh_thread = None    # runs even when stale
```

`_drain_config_token_refresh()` (`tests/_conftest_runtime.py:96-109`) joins a worker
with a 2 s timeout and, on timeout, leaks it while clearing the slot. Under the
documented 26-worker-on-2-CPU contention harness a worker can miss that window. When it
finally runs inside a later test, it clears that test's _live_ worker registration, so
the next expired read starts a second worker. That is precisely the contract
`tests/test_config_cache.py::test_current_config_token_refresh_is_single_flight`
asserts, and that node appears five times in the failure history below.

### Defect 3 — `ProcObserver.stop()` abandons a config-polling daemon after 1 second

This is the poisoner the phase description asks for, and it was found by measurement,
not by reading. Instrumenting every call into `load_merged_config`,
`get_agent_owner_config_snapshot`, `clear_config_cache`, and `current_config_token` with
the calling thread's name, over a serial run of `tests/ace/` plus
`tests/test_config_cache.py` in this workspace, recorded **5121 config calls made from
non-main threads**, from exactly two thread families:

| thread family            | `current_config_token` | `load_merged_config` | `get_agent_owner_config_snapshot` | `clear_config_cache` |
| ------------------------ | ---------------------- | -------------------- | --------------------------------- | -------------------- |
| `sase-ace-proc-observer` | 1413                   | 475                  | 938                               | 0                    |
| `asyncio_0`…`asyncio_6`  | 1563                   | 690                  | 70                                | 3                    |

`ProcObserver` (`src/sase/ace/tui/proc_observer.py:322-346`) is created and started by
every ACE TUI app instance (`src/sase/ace/tui/actions/_proc_action_observer.py:37-40`).
Its loop is `while not self._stop.wait(POLL_SECONDS): self.poll_once()` with
`POLL_SECONDS = 0.5`, and `poll_once()` → `_build_snapshot()` reads config — that is
where those 1413/475/938 calls come from. Its only stopper is:

```python
def stop(self, *, timeout: float = 1.0) -> None:
    with self._lock:
        thread = self._thread
        self._thread = None
        self._stop.set()
    if thread is not None and thread.is_alive():
        thread.join(timeout=timeout)          # <-- abandons the thread on timeout
```

Setting `_stop` prevents the _next_ poll, but a poll already in flight must finish
first. The join caps that wait at one second and then returns, leaving a **daemon**
thread that nothing else will ever stop. Under the documented 26-worker-on-2-CPU
contention harness a thread can miss a one-second window simply by not being scheduled.
The abandoned observer then finishes its poll — calling `current_config_token()` and
`load_merged_config()` — inside whatever test is running next in that xdist worker.

That closes the causal chain for the failing assertion. If an abandoned observer polls
between lines 249 and 251, its `current_config_token()` sees the test's advanced fake
clock (`10.76 >= 10.75`), starts the refresh worker, which publishes `T_b`; the same
poll's later `load_merged_config()` then rebuilds against `T_b` and replaces
`_merged_config_cache_value` with a different dict. Line 251 gets that object and
`is first` fails. The same intrusion explains
`test_load_merged_config_caches_plugin_layer`'s `call_count == 1` failure, and the three
`clear_config_cache` calls recorded from `asyncio_*` threads are a second, rarer
instance of the same class.

The ACE loader pool (`sase-loader_*`,
`src/sase/ace/tui/models/_loaders/_json_cache.py:87`) was the first suspect and is
**not** the culprit: it is alive at every test boundary after
`tests/ace/tui/actions/test_agent_artifact_startup_contracts.py` creates it — 3884
consecutive boundaries in one serial run — but its workers sat idle and made no config
call in the instrumented run. Do not spend time on it.

## Evidence

Failure history for this class from `~/.sase/test-selection/gh_sase-org__sase`, counted
in this workspace: **110 full-run failure records naming a `tests/test_config_cache.py`
node**, across nine distinct nodes in that one file. Post-`3a22ff04f` records, which the
existing `# fixed-at:` block does not retire:

| recorded_at (UTC)   | head        | node                                                              |
| ------------------- | ----------- | ----------------------------------------------------------------- |
| 2026-08-17T08:52:15 | `99b4e43a1` | `test_load_merged_config_caches_plugin_layer`                     |
| 2026-08-17T09:09:32 | `b6246f1cf` | `test_selector_change_eventually_invalidates_merged_config`       |
| 2026-08-17T10:46:16 | `cf7eeee03` | `test_load_merged_config_caches_plugin_layer`                     |
| 2026-08-17T11:40:51 | `ded7f1a5f` | `test_drain_config_token_refresh_joins_worker_and_advances_epoch` |

Nine distinct nodes in the file have failed under the full lane, which is what makes
"one defect, in the cache mechanism" the right reading rather than nine test bugs:
`test_selector_change_eventually_invalidates_merged_config`,
`test_load_merged_config_caches_plugin_layer`,
`test_load_merged_config_caches_default_layer`,
`test_load_merged_config_eventually_invalidates_on_file_mtime_change`,
`test_clear_config_cache_forces_reload`,
`test_clear_config_cache_resets_config_token_time_gate`,
`test_current_config_token_refresh_is_single_flight`,
`test_explicit_invalidation_wins_race_with_background_refresh`,
`test_drain_config_token_refresh_joins_worker_and_advances_epoch`.

Two prior agent shells on this bead (`sase-ns.6.6.6.1` and `sase-ns.6.6.6.1--2`) reached
the same mechanism independently and left `PROGRESS:` notes on the bead. Their trees
were never committed and are not recoverable from this workspace. Their notes carry two
warnings worth honouring: a naive "only the clearing thread may publish" restriction
broke first-fill and drove `config_load_merged` from ~18k to 32810 in the cost lane, and
`tests/test_config_cache.py` is close to the file-size gate (they measured 1021 lines
against a 1000-line limit after adding regressions, and split helpers out to stay
under).

## Scope

Change `src/sase/config/core.py`, `src/sase/ace/tui/proc_observer.py`, the test harness
in `tests/_conftest_runtime.py`, and the tests that cover them. Do not change the
failing tests' assertions: object identity after a clock advance is the contract being
defended, not the problem.

Step 3 is the one that turns the named node green; steps 1 and 2 are what stop the next
poller from doing the same thing. Land them together.

### 1. Publish each derived cache through one immutable slot

Replace each `(token, value)` global pair with a single frozen slot object published by
one reference assignment, so no reader can observe a torn pair:

```python
@dataclass(frozen=True, slots=True)
class _ConfigCacheSlot:
    token: tuple[Any, ...]
    value: Any

_merged_config_slot: _ConfigCacheSlot | None = None
_agent_owner_config_slot: _ConfigCacheSlot | None = None
_derived_config_build_lock = threading.RLock()
```

and give both accessors double-checked locking that re-reads the token under the build
lock, so a slow builder can never publish a token that is already stale:

```python
def load_merged_config() -> dict[str, Any]:
    slot = _merged_config_slot                 # one atomic read, no lock
    token = current_config_token()
    if slot is not None and slot.token == token:
        return slot.value
    with _derived_config_build_lock:
        token = current_config_token()         # re-read: never publish a stale token
        slot = _merged_config_slot
        if slot is not None and slot.token == token:
            return slot.value
        result = merge_config_sources(...)
        _merged_config_slot = _ConfigCacheSlot(token, result)
        return result
```

Four properties this must preserve, each of which a reviewer should be able to check:

- **The hit path stays lock-free.** It is one global read plus a
  `current_config_token()` call, which is what it is today. This is what keeps the
  `config_load_merged` cost budget intact; concurrent duplicate builds now collapse into
  one, so the count should fall slightly or stay flat, never rise. If it rises, the
  first-fill path has been over-restricted — that is the exact regression the prior
  shell hit.
- **`_derived_config_build_lock` is an `RLock` shared by both derived caches**, because
  `load_merged_config()` calls `get_agent_owner_config_snapshot()` inside its own build
  (`src/sase/config/core.py:652`) and would otherwise self-deadlock.
- **Lock order is build lock, then token lock, never the reverse.**
  `current_config_token()` takes `_current_config_token_cache_lock` while the build lock
  is held. `clear_config_cache()` already holds the token lock, so it must _not_ take
  the build lock; it clears the slots with plain assignments
  (`_merged_config_slot = None`). That is safe: clearing is a single reference write,
  and the generation bump means a build still in flight publishes a token no later
  reader will match.
- **`_default_config_cache` and `_plugin_configs_cache` are built under the same build
  lock**, so `test_load_merged_config_caches_plugin_layer`'s `call_count == 1` contract
  cannot be broken by two builders racing the `is None` check.

### 2. Only the registered worker may deregister itself

In `_refresh_current_config_token()`, make the slot clear conditional:

```python
if _current_config_token_refresh_thread is threading.current_thread():
    _current_config_token_refresh_thread = None
```

### 3. Stop `ProcObserver` from abandoning a live poll

Make observer teardown deterministic instead of time-boxed. Two parts, both needed:

- **Do not abandon the thread on a slow schedule.** `stop()`'s one-second join is the
  abandonment. Raise it to a budget that a bounded poll cannot plausibly exceed even
  under 13x CPU oversubscription (order of tens of seconds), rather than one second.
  Keep it finite so a genuinely wedged poll cannot hang app teardown forever, and log at
  debug level when the join does time out so the abandonment stops being silent.
- **Make the in-flight poll end sooner.** `poll_once()` should return early once
  `self._stop` is set, checked before `_build_snapshot()` and before delivering the
  snapshot, so a stop that arrives mid-poll does not have to wait out the whole poll.

Then make a recurrence loud rather than silent: extend the autouse isolation fixture in
`tests/_conftest_runtime.py` so that a test which leaves a live `sase-ace-proc-observer`
fails, in the spirit of the existing `_drain_config_token_refresh()` and of
`tests/test_config_cache_isolation.py::test_no_live_refresh_worker_after_drain_window`.
A leaked config-polling thread must be attributed to the test that leaked it, not
silently inherited by the next one.

The instrument that produced the Defect 3 table, if it needs to be rebuilt: a pytest
plugin that wraps those four functions and records every non-main-thread call with the
active nodeid. Two traps cost this investigation a full run each, so do not rediscover
them. First, a module-level `global x = ...` inside `sase.config.core` does **not** go
through the module object's `__setattr__`, so hooking `ModuleType.__setattr__` records
nothing. Second, `from sase.config.core import load_merged_config` binds at import time,
so the wrapper must also be re-pointed across every already-imported module in
`sys.modules`, not just set on `sase.config.core`.

### 4. Regression coverage

Add tests that fail before the change and pass after. Keep `tests/test_config_cache.py`
under the 1000-line file gate — extract helpers to a `tests/_config_cache_helpers.py`
module if needed, which is the route the prior shell took and measured.

- A concurrent builder cannot make `load_merged_config()` return a different object
  while the token is unchanged, and cannot leave a slot whose token and value disagree.
- A builder whose token went stale mid-build does not clobber a fresher publication.
- A stale-epoch refresh worker cannot deregister a live worker (drives defect 2).
- `ProcObserver.stop()` joins a poll that outlives the old one-second budget, and
  `poll_once()` returns early once `_stop` is set.
- An ordering regression in the `tests/test_config_cache_isolation.py` style: a poisoner
  test that leaves an observer mid-poll, then a victim test that asserts
  `load_merged_config()` still returns the same object across a clock advance. That file
  already runs a two-test poisoner/victim pair through `pytester.runpytest_subprocess`,
  so follow its shape rather than inventing one.

### 5. The flake baseline, only if the gate is still red on history

Remove nothing from `tests/reproducible_flake_baseline.txt`. If the fix lands and
`just selection-health --fail-on-new-flake` is still red purely on pre-fix evidence, add
a `# fixed-at: <UTC timestamp> <node id>` block naming bead `sase-mv` and the fix
commit, following the convention already documented in that file's header. Do not add
node-ID entries; those are debt, not a fix.

## Boundary check

This does not cross the Rust core backend boundary. `_merged_config_slot` and
`_current_config_token_cache_*` are CPython process-global caches in front of a Python
merge; the change is thread-safety on that in-process cache, not new backend or domain
behavior. No wire, binding, or `../sase-core` API changes.

## Verification

1. `just install` first — this workspace is ephemeral.
2. `just test tests/test_config_cache.py tests/test_config_cache_isolation.py` green.
3. `SASE_CONTENTION_REPEAT=3 just test-contention tests/test_config_cache.py` green on
   every repeat. This was already green before the change, so it is a no-regression
   check, not evidence the fix worked.
4. A whole-suite contention run is the check that matters, because the plan's own
   evidence says file-scoped contention does not reproduce this:
   `SASE_CONTENTION_REPEAT=1 just test-contention` with no path argument. Report the
   per-node tally for `tests/test_config_cache.py`.
5. `just check` green.
6. `just check-full` through `/sase_monitor` with a `--next` action — never inline.
   `tests/_conftest_runtime.py` is in the broadening set, so the scoped lane is not
   sufficient here. The pre-existing suite-cost budget failure tracked by `sase-j0` is
   expected and is not this plan's to fix; any _new_ `config_load_merged` regression is.
7. `just selection-health --fail-on-new-flake` names at most the `sase-n4`-owned
   `tests/fakey/test_usage_limit_e2e.py` node.

## Handoff obligations

The implementing agent owns phase bead `sase-ns.6.6.6.1` and must:

- Leave a `sase bead note sase-ns.6.6.6.1` recording what changed and what verified it,
  whether or not the work lands. If it cannot finish, that note is the handoff and must
  say what was tried and what the next agent should do.
- Record discovered work as
  `sase bead note sase-ns.6.6.6.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. Do
  not create beads.
- Close **only** `sase-ns.6.6.6.1`, with
  `sase bead close sase-ns.6.6.6.1 --note "<what was verified>"`. Never close epic
  `sase-ns.6.6.6` or any ancestor.
- Never ask the user for approval directly. This work is an objective fix — a leaking
  cache and a broken single-flight guard — so it does not need approval. If some part of
  it turns out to need an owner decision, leave
  `sase bead note sase-ns.6.6.6.1 'TASK NEEDS APPROVAL: ...'` and stop that part.
- If the root cause turns out to be owned by in-progress epic `sase-j7` ("Fix the
  sase-ct flake class at its root"), record
  `sase bead note sase-j7 'DISCOVERED ISSUE: ...'` and say so in the phase note — but
  still fix what can be fixed without waiting on it.

## Out of scope

- `tests/fakey/test_usage_limit_e2e.py` — the other node holding the flake gate red,
  owned by in-progress epic `sase-n4`.
- The suite-cost budget gate (`sase-j0`) and the process-global leak class at its root
  (`sase-j7`).
- The ACE loader pool (`sase-loader_*`). It leaks across the whole session but made no
  config call under instrumentation, so it is not this defect. If its `executor.map()`
  abandonment looks like a product bug worth its own work, record it as a
  `PROPOSED FOLLOW-UP:` note.
- The `asyncio_*` default-executor threads, beyond noting them. They account for 3 of
  the recorded `clear_config_cache` calls and are the same class of defect, but their
  owner is the asyncio loop teardown, not this bead. If steps 1-3 leave them able to
  poison a test, record a `PROPOSED FOLLOW-UP:` note rather than widening this plan.
- The other four phases of epic `sase-ns.6.6.6` (`goldens`, `supervise`, `forksafe`,
  `saseinstall`). This plan touches `tests/reproducible_flake_baseline.txt` only
  additively, so it does not conflict with `supervise`, which removes its own entries.
