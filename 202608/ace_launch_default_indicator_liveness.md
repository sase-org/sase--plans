---
tier: tale
title: Keep the ACE top-bar launch-default indicator live as the @large pool rotates
goal:
  The ACE top-bar launch-default pill always names the provider/model that the next
  no-%model launch will actually use, updating within seconds after a pooled alias
  advances its round-robin cursor, without ever resolving models on the Textual UI
  thread.
size: medium
proposed_by: bbugyi200.athena.034
create_time: 2026-08-15 20:39:38
status: wip
---

# Plan: Keep the ACE top-bar launch-default indicator live

## 1. Observed symptom

A `sase` agent was launched from ACE with no `%model` directive, so it routed through
the configured `llm_provider.default_model` (`@large`). The agent detail panel correctly
showed `Model: CODEX(gpt-5.6-sol) @ xhigh ← @large`, meaning that launch consumed the
`codex/gpt-5.6-sol` member of the two-member `@large` pool.

The top-right ACE indicator — which is supposed to answer "what runs on the next
no-`%model` launch?" — kept showing `CODEX(gpt-5.6-sol)`. It should have flipped to
`CLAUDE(opus)`, because the round-robin cursor had advanced past the codex member.

Worse than a delay: the pill is frozen for the **entire lifetime of the ACE process**.
It never updates on its own, no matter how many launches rotate the pool.

## 2. Root cause

**The launch-default value is resolved exactly once per ACE process and then cached
forever. The 30-second poll re-renders the cached value but never re-resolves it.**

In `src/sase/ace/tui/widgets/llm_override_indicator.py`:

- `on_mount()` schedules one background resolution worker and then installs
  `self.set_interval(30.0, self.refresh)`.
- `refresh()` re-arms that worker **only** when the cache is still empty:

  ```python
  if self._cached_default is None and not self._cached_default_failed:
      if primary_override is None:
          self._schedule_default_resolution_if_needed()
  self._apply_content()
  ```

- `_apply_content()` → `_build_cached_default_content()` renders `self._cached_default`
  verbatim.

So once `_cached_default` is populated, every subsequent tick repaints the same tuple.
The only re-arm path in the codebase is `invalidate_cached_default()`, called from
`src/sase/ace/tui/actions/agent_workflow/_leader_mode.py` when Launch Control is
dismissed with `provider_routing_changed=True`. Nothing else — not an agent launch, not
a config edit, not a pool rotation — ever invalidates it.

### Confirming evidence

- **The resolver itself is correct and live.** With the machine-global cursor at
  `large.cursor == 1` in `~/.sase/llm_lb.json`, a direct non-consuming call to
  `resolve_effective_default_provider_model()` returns `('codex', 'gpt-5.6-sol')` and
  leaves the cursor untouched. The resolver already answers "what runs next" exactly as
  the indicator intends; only the widget's caching is wrong.
- **Reproduced in isolation.** Populating `_cached_default` with one value, swapping the
  resolver to return a different one, then driving three refresh ticks re-armed the
  worker `0` times, called the resolver `1` time total, and kept rendering the original
  `CODEX(gpt-5.6-sol)`.
- **It is a latent regression, not an original design flaw.** The cache landed in
  `fae4743b9` ("feat: defer LLM indicator default resolution", 2026-05-15) purely to
  keep the cold provider resolve off the UI thread. At that time the launch default was
  a _static_ value, so a process-lifetime cache was correct. Load-balanced alias pools
  landed later, in `5a23c297f` ("feat!: add load-balanced model alias pools",
  2026-07-21), which turned the launch default into a _moving_ value. The cache was
  never revisited.

### Why "just refresh it on the tick" is not the whole answer

The resolve must stay off the UI thread. `resolve_effective_default_provider_model()`
takes an exclusive `flock` on `~/.sase/llm_lb.lock` (via
`select_model_alias_pool_member`) and performs a self-cleaning read of the
temporary-override store. Under concurrent agent launches those are unbounded waits, and
`sase/memory/tui_perf.md` rules 1, 2, and 11 forbid blocking disk I/O, locks, and slow
bodies in timer/pump callbacks. Measured cost is ~38 ms cold and ~0.5 ms warm — cheap,
but lock waits are not bounded by that.

`tui_perf.md` rule 10 also states: _periodic ticks revalidate; recomputes get a longer
cadence_. So the fix is not "resolve every tick" — it is "revalidate cheaply every tick,
and recompute off-thread only when something actually changed".

### Why a launch-time event hook is not sufficient either

The pool cursor does **not** advance when ACE launches an agent. It advances inside the
agent runner process, at the moment of the real LLM invocation, via
`resolve_launch_selection(..., consume=True)` from the workflow executor's prompt step
(see the module docstring in `src/sase/llm_provider/launch_selection.py` and the
regression suite in `tests/test_pooled_alias_single_consumption.py`). A refresh fired at
launch time would race the consume and read the pre-launch cursor. Polling is the
correct mechanism; an event hook would only be an unreliable latency optimization.

### Backend boundary

No Rust core change is required. The rotation cursor and its state file are Python-only
(`src/sase/llm_provider/load_balancing.py`, `~/.sase/llm_lb.json`); the sibling Rust
core has no load-balancing state — its only `round_robin` references are in
`sase_xprompt_lsp` and merely consume a `selector_mode` string it is handed. This change
is TUI refresh cadence plus a display-only read of state files whose owners already live
in this repo's Python `llm_provider` package, so it stays on the presentation side of
the boundary described in `CLAUDE.md`.

## 3. Design

Three parts. All of them mirror patterns that already exist in this repo.

### 3.1 A cheap, lock-free change token for the launch default

Add `src/sase/llm_provider/launch_default_peek.py`, modeled directly on the two existing
peek modules (`temporary_override_peek.py`, `provider_disable_peek.py`), including their
`_PEEK_STAT_FLOOR_SECONDS = 0.5` monotonic stat floor and their "any failure degrades to
a constant" posture.

```python
def peek_launch_default_change_token() -> tuple[object, ...]:
    """Return a cheap token that changes when the launch default might have."""
```

The token is built from, in order:

1. `sase.config.current_config_token()` — covers edits to `llm_provider.default_model`
   and to alias definitions. Already time-gated and revalidated off-thread, so it is
   safe to call from a timer callback.
2. `(st_mtime_ns, st_size)` of the pool rotation state file (`llm_lb.json`) — this is
   the one that moves on every pooled launch.
3. `(st_mtime_ns, st_size)` of the temporary-override state file (`llm_override.json`) —
   covers an override placed on `@large` itself.
4. `(st_mtime_ns, st_size)` of the provider-disable state file
   (`llm_provider_disables.json`) — a disable changes which pool members are available.

A missing file contributes `None`; any `OSError` or unexpected exception makes the whole
token a fixed sentinel so a broken stat degrades to "no change detected" rather than a
refresh storm. Reads are `os.stat` only — no parsing, no locks.

To avoid duplicating filename literals, expose the two paths that are currently private
through small public accessors and have the existing private helpers delegate to them:

- `load_balancing.rotation_state_path()` (currently `_state_path()`).
- `provider_disable_peek.provider_disable_state_path()` (currently `_state_path()`).

`temporary_override_state.state_path()` is already public and is used as-is.

**Known, accepted limitation:** an `(mtime_ns, size)` token can in principle miss a
rewrite that lands in the same nanosecond at the same size. Both existing peek modules
already accept this tradeoff, and `llm_lb.json` is written through `os.replace` of a
fresh temp file, so a collision is not realistically reachable. Document it in the
module docstring rather than inventing a heavier-weight signal.

### 3.2 Widget: revalidate on tick, recompute off-thread on change

Rework `LLMOverrideIndicator` so the periodic tick is cheap and lock-free, and the
expensive resolve runs off-thread only when warranted.

- **Make the synchronous per-tick override read lock-free.** The widget currently calls
  `get_active_temporary_override()` — the locking, self-cleaning store read — up to
  three times per tick on the UI thread (in `refresh()`, `_apply_content()`, and
  `_schedule_default_resolution_if_needed()`). Add a `peek_active_temporary_override()`
  wrapper next to the existing `peek_active_alias_overrides()` (it is the same lookup,
  keyed by `launch_model_setting_override_key(DEFAULT_MODEL_FIELD)`), and use it for the
  widget's display reads. This matches `ProviderDisablesIndicator`, which already peeks.
  Expiry is still evaluated against the current clock on every peek call, so the
  countdown and expiry behavior are unchanged.

  Leave `AliasOverridesIndicator` on the authoritative self-cleaning read — its module
  docstring documents that choice deliberately. Do not change it.

- **Tighten the cadence.** Replace `set_interval(30.0, self.refresh)` with a
  module-level `_POLL_INTERVAL_SECONDS = 5.0`. This is only affordable because the tick
  is now pure peek + `os.stat`; state this rationale in a comment so a future reader
  does not "optimize" the peek back into a locking read.

- **Re-arm on the tick when, and only when, one of these holds:**
  1. `self._cached_default is None` and no resolve is in flight (the existing cold-start
     case);
  2. `self._cached_default_failed` is set — periodic re-arm now lets transient failures
     self-heal instead of latching `unavailable` for the process lifetime;
  3. the change token differs from the token captured at the last _completed_ resolve;
  4. an override pill was active on the previous tick and is not active now. Override
     expiry is clock-based, so no state file mtime changes when an override lapses;
     without this rule the pill would fall back to a potentially stale cached default.

- **Never clear `_cached_default` on the periodic path.** Keep rendering the last known
  value while the worker runs, so the pill does not flash `...` every few seconds.
  `invalidate_cached_default()` keeps its current clear-then-re-resolve behavior — it is
  an explicit user action after a routing change, where a brief placeholder is honest.

- **Capture the token at resolve start and store it on success.** Record the token
  _before_ the worker runs and commit it in `on_worker_state_changed` only on
  `WorkerState.SUCCESS`, so a resolve that raced a concurrent write is retried on the
  next tick instead of being wrongly marked current.

- **Keep the existing worker mechanics.**
  `run_worker(..., thread=True, exclusive=True, group=_DEFAULT_WORKER_GROUP)` already
  gives last-request-wins coalescing, which matches `tui_perf.md` rule 5. Keep
  `_build_content()` and `_build_default_content()` as-is for the existing
  synchronous-snapshot tests.

- **No activity gate.** `tui_perf.md` rule 13 covers deferring non-urgent work during
  navigation; the tick here is a handful of `os.stat` calls plus a worker spawn, so
  gating it would add state for no measurable benefit. Noted explicitly so the omission
  reads as a decision, not an oversight.

### 3.3 Tooltip: explain the rotation

Once the pill starts flipping between `CLAUDE(opus)` and `CODEX(gpt-5.6-sol)` on its
own, a silently changing label could read as a glitch. Make the existing hover tooltip
explain it.

Have the worker return a small frozen dataclass instead of a bare `(provider, model)`
tuple, sourced from
`build_launch_model_setting_snapshot(DEFAULT_MODEL_FIELD, consume=False)`, which already
exposes `referenced_alias`, `selector_mode`, and `selector_members`. Carry `provider`,
`model`, `referenced_alias`, `selector_mode`, and the member count.

When `selector_mode == "round_robin"`, add one line to the inactive-override tooltip,
for example:

```
Launch default: CLAUDE(opus)
@large rotates across 2 models; CLAUDE(opus) is next.
No temporary override active.
Press ,m for Launch Control.
```

Non-pool defaults keep today's three-line tooltip unchanged.

**Deliberately rejected:** adding a visible rotation marker (e.g. a `⇄` glyph) to the
pill itself. The top bar already carries seven indicators, the pill is the widest of
them, and the tooltip plus Launch Control already own the detail view. Keep the pill
text exactly as it renders today.

## 4. Non-goals

- Changing _which_ model a launch selects, or any rotation/consumption semantics.
  `tests/test_pooled_alias_single_consumption.py` must keep passing unmodified.
- Refreshing the indicator from agent-launch events (see §2 — it would race the consume
  and the 5 s poll already covers it).
- Migrating `AliasOverridesIndicator` to a lock-free peek. Its self-cleaning read is
  documented as intentional; if it is worth revisiting, that is a separate task bead,
  not part of this change.
- Any Rust core (`sase-core`) change.
- Hand-editing `CHANGELOG.md` — this repo generates it via release-please.

## 5. Files to change

| File                                                   | Change                                                                                                                                                                                         |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/sase/llm_provider/launch_default_peek.py`         | **New.** `peek_launch_default_change_token()`, stat-floor cache, docstring covering the mtime-collision limitation.                                                                            |
| `src/sase/llm_provider/load_balancing.py`              | Promote `_state_path()` to public `rotation_state_path()`; keep internal callers working.                                                                                                      |
| `src/sase/llm_provider/provider_disable_peek.py`       | Promote `_state_path()` to public `provider_disable_state_path()`; keep internal callers working.                                                                                              |
| `src/sase/llm_provider/temporary_override_peek.py`     | Add `peek_active_temporary_override()` (lock-free counterpart of `get_active_temporary_override()`).                                                                                           |
| `src/sase/llm_provider/temporary_override.py`          | Re-export `peek_active_temporary_override` in `__all__` alongside `peek_active_alias_overrides`.                                                                                               |
| `src/sase/ace/tui/widgets/llm_override_indicator.py`   | Peek-based display reads, `_POLL_INTERVAL_SECONDS = 5.0`, token-gated worker re-arm, richer worker payload, pool-aware tooltip.                                                                |
| `tests/llm_provider/test_launch_default_peek.py`       | **New.** Token unit tests.                                                                                                                                                                     |
| `tests/test_llm_override_indicator.py`                 | Extend with re-arm/gating/tooltip coverage.                                                                                                                                                    |
| `tests/test_launch_default_indicator_pool_rotation.py` | **New.** Composed regression test for the reported bug.                                                                                                                                        |
| `docs/ace.md`                                          | Short paragraph near the top-bar pill documentation (~the "Temporary overrides" area, where the gold `default` pill is described) covering the launch-default pill and its live pool rotation. |

## 6. Implementation steps

1. Promote the two private state-path helpers to public accessors and repoint their
   existing internal callers. Confirm `sase/memory/symvision.md` rules are satisfied —
   each new public symbol must have a real caller.
2. Add `peek_active_temporary_override()` to `temporary_override_peek.py` and re-export
   it from `temporary_override.py`.
3. Add `launch_default_peek.py` with `peek_launch_default_change_token()`.
4. Rework `LLMOverrideIndicator` per §3.2 and §3.3.
5. Update `docs/ace.md`.
6. Write the tests in §7.
7. Run `just check`. Before landing, run `just check-full` through `/sase_monitor`
   (never inline), passing a `--next` action so the follow-up agent acts on the result.

## 7. Tests

**`tests/llm_provider/test_launch_default_peek.py`** (unit, isolated `SASE_HOME`)

- Token is stable across repeated calls when nothing changes.
- Token changes after a real `select_model_alias_pool_member(..., consume=True)`
  advances `llm_lb.json`. Use the stat floor deliberately: either monkeypatch
  `time.monotonic` or the floor constant so the test is not wall-clock flaky.
- Token changes when the override state file changes, and when the provider-disable
  state file changes.
- Missing state files yield a stable token rather than raising.
- A stat that raises `OSError` degrades to the fixed sentinel, not an exception.

**`tests/test_llm_override_indicator.py`** (extend the existing file)

- Cache re-arms when the token changes; does **not** re-arm when it is unchanged (the
  tick-cost guarantee).
- Re-arms while `_cached_default_failed` is set, so failures self-heal.
- Re-arms on the tick where an active override lapses (§3.2 rule 4).
- `_cached_default` is not cleared on the periodic re-arm path — the pill keeps the
  previous label rather than flashing `...`.
- The token is committed only on `WorkerState.SUCCESS`, so an errored resolve is retried
  on the next tick.
- Pool tooltip renders the rotation line for `round_robin`, and non-pool defaults keep
  the existing three-line tooltip.
- No test may call the real resolver on the UI thread — assert via a monkeypatched
  resolver that counts calls.

**`tests/test_launch_default_indicator_pool_rotation.py`** (new, composed regression —
the reported bug, end to end)

Against an isolated `SASE_HOME` and the shipped two-member `@large` pool (reuse the
fixtures behind `tests/test_pooled_alias_single_consumption.py` and
`tests/llm_provider/_load_balanced_alias_helpers.py`):

1. Resolve and cache the launch default; assert the pill renders the member the cursor
   currently points at.
2. Advance the pool exactly once with a real consuming resolution.
3. Drive one indicator tick and let the worker complete.
4. Assert the rendered pill now names the **other** pool member, and that the cursor was
   not advanced by the indicator itself (the display path must stay `consume=False` —
   this is the guard against the fix accidentally stealing a pool slot from a real
   launch).

## 8. Definition of done

- With ACE running, launching a no-`%model` agent flips the top-bar pill to the other
  `@large` member within ~5 seconds of the launched agent's first LLM invocation, and it
  keeps alternating across further launches.
- The indicator never advances the rotation cursor itself.
- No model resolution, state-store lock, or blocking disk read happens on the Textual UI
  thread; the periodic tick is peek + `os.stat` only.
- `just check` passes, and `just check-full` (run through `/sase_monitor`) passes before
  landing.

## 9. Risks

- **Refresh storm from a mis-scoped token.** If the token accidentally included
  something that changes every tick, the widget would spawn a worker every 5 seconds.
  Mitigated by the explicit "does not re-arm when unchanged" test.
- **Cross-process write races.** Another process can write `llm_lb.json` while the
  worker resolves. Mitigated by committing the token only on success, so the next tick
  retries.
- **Textual worker cancellation churn.** `exclusive=True` cancels an in-flight worker on
  re-arm. Because re-arms are token-gated they are rare, and the existing
  `_cached_default` keeps the pill painted meanwhile.
- **Test flakiness from the 0.5 s stat floor and the 5 s interval.** Drive ticks by
  calling the widget's methods directly with an injected clock; never wait on real
  wall-clock intervals.
