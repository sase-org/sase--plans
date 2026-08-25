---
tier: tale
title: Stop the prompt-format fake formatters from self-releasing under host load
goal:
  tests/ace/tui/widgets/test_prompt_format.py stops using short wall-clock ceilings as
  race participants, so a slow host can no longer unblock a fake formatter that the test
  still believes is in flight, and
  tests/ace/tui/widgets/test_prompt_format.py::test_newer_format_request_wins stops
  failing `just check-full`'s cost lane with `wait_for(<lambda>) timed out after 1.0s`.
size: small
proposed_by: bbugyi200.athena.8ptmrds1fsbc.f1
create_time: 2026-08-25 15:19:38
status: wip
---

# Plan

`tests/ace/tui/widgets/test_prompt_format.py::test_newer_format_request_wins` failed one
`just check-full` cost-lane run on 2026-08-25 with:

```
AssertionError: wait_for(test_newer_format_request_wins.<locals>.<lambda>)
timed out after 1.0s waiting for it to return True
```

The mechanism is fully understood and reproduces deterministically. It is a test defect,
not a production defect.

## What the investigation established

Measured at HEAD `bb429cf37` in a numbered workspace after `just install`. Every claim
below was run, not assumed.

1. **The recorded evidence is one real failure, not a pattern.** The selection-health
   store holds exactly two records naming the node.
   `20260825T181321Z-6b22f7e72d6d-135740-full-run.json` (mode `cost`, recorded
   2026-08-25T18:28:09Z) lists it as the run's _only_ failure. The other,
   `20260825T143552Z-70a9d101583f-62771-full-run.json`, has 1564 failures and is
   therefore already excluded from flake evidence by the gate's `max_failures_per_run`
   bar. The node is **not** in `tests/reproducible_flake_baseline.txt` and is **not**
   one of the four nodes `just selection-health --fail-on-new-flake` currently gates.

2. **The file is green on an idle host and stays green under the sanctioned soak.**
   `.venv/bin/python -m pytest tests/ace/tui/widgets/test_prompt_format.py` passes in
   ~6s, and
   `SASE_CONTENTION_REPEAT=3 just test-contention -- tests/ace/tui/widgets/test_prompt_format.py`
   is green 3/3 (`0 node(s) failed`), though the node's own call time stretches from
   ~0.5s to 2.3-3.0s there.

3. **The assertion window has enormous slack; the window _before_ it does not.** An
   instrumented copy of the node shows that on an idle host the second format result is
   already applied by the time `await pilot.press("ctrl+g", "f")` returns
   (`applied == press2 == 0.165s`), so `wait_for`'s 1.0s budget is normally consumed by
   zero polls. Under the pinned contention reproducer (26 workers on 2 CPUs) the window
   from the first fake formatter blocking to the test's release peaked at 0.687s across
   60 runs — 3.4x the idle figure, and one third of the 2s ceiling that gates it.

4. **The ceiling that actually decides the outcome is `release_first.wait(timeout=2)`
   inside the test's own fake formatter.** That call is not a hang guard; it is a race
   participant. When it expires, the first formatter stops being "blocked": it returns
   `"old result"`, and because the second `gf` has not been dispatched yet the request
   is still current, so production correctly applies it. The second request then formats
   `"old result"` and produces `"new old result"`, and the test's
   `wait_for(text == "new draft")` can never succeed no matter how large its timeout is.

5. **That mechanism reproduces deterministically and prints the recorded message.**
   Injecting `await asyncio.sleep(2.2)` between the two presses of a copy of the node
   fails 3/3 with exactly the recorded `wait_for(<lambda>) timed out after 1.0s`, and
   the instrumentation shows why: `released_cleanly=0.0`, `blocked_for=2.000`,
   `text='new old result'`, and the second formatter call receiving `'old result'`
   instead of `'draft'`.

6. **Four of the five gated tests in the file share the defect.** Injecting the same
   2.2s stall after every `assert await asyncio.to_thread(<event>.wait, 1.0)` handshake
   in a copy of the whole file fails `test_newer_format_request_wins`,
   `test_cursor_and_mode_changes_during_worker_are_preserved`,
   `test_edit_while_formatter_runs_is_responsive_and_discards_result`, and
   `test_rebuilt_pane_discards_old_widget_result` (8 passed, 4 failed). Only
   `test_focus_change_does_not_retarget_multi_pane_format` survives, and only because
   its assertions happen to hold either way.

7. **Production is behaving exactly as designed.** `_prompt_format_result_is_current`
   (`src/sase/ace/tui/widgets/_prompt_format.py:182`) discards a worker result unless
   the request id, mount state, parent, and source text all still match, and the
   injected repro shows it discarding the stale result and applying the current one
   correctly at every step. No production change is warranted; the widget never sees the
   fake formatter's ceiling.

8. **The repo already owns the remedy and it is under-used.** `tests/_load_tolerant.py`
   defines `LOAD_TOLERANT_TIMEOUT = 60.0` and documents it as exactly this distinction —
   "These waits are deadlock detectors, not speed assertions." Only five test files
   import it. Separately, `tools/check_test_wait_helpers` (the `lint (test waits)` gate)
   already tells tests that a positive literal sleep "must use an inline '#
   sase-test-wait: <reason>' pragma, or be replaced by an observable wait", which is the
   same rule this file's five `await pilot.pause(0.05)` calls sit just outside of.

9. **The candidate fix was validated before this plan was written.** A candidate copy of
   the file carrying every edit in "Work to do" passes the injected-stall repro 12/12,
   passes normally 12/12, and passes 3/3 pinned-contention repeats at 26 workers on two
   CPUs.

## Work to do

All edits are in `tests/ace/tui/widgets/test_prompt_format.py` unless stated otherwise.

### 1. Make the fake formatters' release waits deadlock detectors

- Import the shared constant: `from tests._load_tolerant import LOAD_TOLERANT_TIMEOUT`
  (the same import line `tests/ace/tui/test_residual_freeze_soak.py` uses).
- Add one module-level helper the fake formatters call instead of waiting inline, e.g.:

  ```python
  def _wait_for_release(event: Event) -> None:
      """Block a fake formatter thread until the test releases it.

      This ceiling is a deadlock detector, not a speed assertion. A short one is
      a race participant: when it expires, the formatter the test still believes
      is in flight returns on its own, its result is applied as the current
      request, and every later assertion is made against a source the widget no
      longer holds.
      """
      if not event.wait(timeout=LOAD_TOLERANT_TIMEOUT):
          raise AssertionError("fake formatter was never released by the test")
  ```

- Replace all five inline waits with it: lines 95, 152, 239, 278 and 312 today
  (`release.wait(timeout=2)` and `release_first.wait(timeout=2)`).

### 2. Give the cross-thread handshakes the same load-tolerant ceiling

Every `assert await asyncio.to_thread(<event>.wait, 1.0)` in the file — lines 107, 170,
178, 252, 259, 292, 298, 326, 334 today — waits for a worker to reach or leave the
formatter thread. Pass `LOAD_TOLERANT_TIMEOUT` instead of `1.0`. Keep the `assert`.

### 3. Drop the sub-default `wait_for` budgets

Remove the explicit `timeout=1.0` from all nine `wait_for(...)` calls (lines 48, 80,
113, 136, 185, 207, 256, 295, 364 today) so they use `sase.ace.testing.wait_for`'s
documented 5.0s default. These waits cross a thread and a worker, which is the case that
module's docstring says must be waited on by end state; 5.0s is ~30x the worst window
measured under the contention reproducer, and nothing in the file justified tightening
below the house default.

### 4. Replace the fixed 50ms pauses with the observable end state

The five `await pilot.pause(0.05)` calls (lines 224, 260, 299, 335, 365 today) are fixed
sleeps standing in for "the format worker finished". They cannot fail the run, which is
the problem: they can pass vacuously before the worker they are meant to outlive has
even resumed. Add a second module-level predicate and wait on it instead:

```python
def _format_workers_finished(app: App[None]) -> bool:
    """Report that no prompt-format worker is still in flight."""
    return not any(worker.name.startswith("prompt-format:") for worker in app.workers)
```

`PromptFormatMixin.format_prompt_markdown` names its workers `prompt-format:<id>`
(`src/sase/ace/tui/widgets/_prompt_format.py:147`), and Textual's `WorkerManager` drops
a worker from the manager when it reaches a terminal state, so an empty match is the
real end state. Use `await wait_for(pilot, lambda: _format_workers_finished(app))` at
each of the five sites. This is what turns
`test_rebuilt_pane_discards_old_widget_result` and `test_newer_format_request_wins` into
real proofs that the stale result was discarded rather than merely late.

### 5. Name the predicates that guard the two-request orderings

`wait_for`'s timeout message renders `predicate.__qualname__`, which is why the recorded
failure says only `wait_for(<lambda>)`. In `test_newer_format_request_wins` and
`test_edit_while_formatter_runs_is_responsive_and_discards_result`, replace the text
lambda with a named local predicate (e.g. `def newer_result_is_applied() -> bool:`) so a
future miss names what was missing. Leave the other lambdas alone; they are not the ones
that have burned a run.

### 6. Retire the one recorded failure

Append a block to `tests/reproducible_flake_baseline.txt` in the same shape as the
existing `sase-rm.9` block — a comment naming this fix and the mechanism, then:

```
# fixed-at: <UTC instant the fix lands> tests/ace/tui/widgets/test_prompt_format.py::test_newer_format_request_wins
```

Use `date -u +%Y-%m-%dT%H:%M:%SZ` at commit time (or later). Do **not** add the node as
a plain baseline entry: it is not gated today, and the file's header calls plain entries
"debt to remove, not suppressions to grow". The retirement exists so the single
2026-08-25 record cannot later pair with an unrelated future failure and re-open this
investigation from zero. A `# fixed-at:` entry that stops retiring anything is reported
as removable, not as a gate failure, so this cannot rot into a hidden suppression.

## Verification

- `just install` first — numbered workspaces go stale.
- `.venv/bin/python -m pytest tests/ace/tui/widgets/test_prompt_format.py -q` — 12
  passed.
- Prove the fix against the mechanism, not just the symptom: copy the edited file to a
  gitignored scratch path (`.pytest_cache/` is the suite's scratch space), inject
  `await asyncio.sleep(2.2)` after every `to_thread(<event>.wait, ...)` handshake, and
  run it. Before this plan's edits that injection fails 4 tests; after them it must pass
  12/12. Delete the scratch copy afterwards.
- `SASE_CONTENTION_REPEAT=3 just test-contention -- tests/ace/tui/widgets/test_prompt_format.py`
  — expect a green tally. This was green before the fix too, so it is a no-regression
  check, not proof.
- `just check` inline.
- `just check-full` through `/sase_monitor` (never inline) with a `--next` action.
  **Expect its final `flake baseline` stage to still fail.** At plan-authoring time
  `just selection-health --fail-on-new-flake` already exits 1 naming four unrelated
  nodes
  (`tests/ace/tui/test_agents_pane_mount.py::test_agents_pane_mounts_activates_and_loads`,
  `tests/fakey/test_pipe_e2e.py::test_default_pipe_creates_family_member_with_fork_and_shared_workspace`,
  `tests/main/test_parser_command_help.py::test_agents_help_renders_sorted_subcommands`,
  `tests/test_config_schema.py::test_default_config_matches_public_schema`). That red is
  pre-existing and out of scope. Confirm only that the four names are unchanged and that
  this plan's node is not among them; do not chase or baseline them here.

## Non-goals

- Do not change `src/sase/ace/tui/widgets/_prompt_format.py`. Its staleness check is
  correct, and evidence item 7 shows it discarding and applying results exactly as
  designed throughout the deterministic repro.
- Do not "fix" this by raising only `wait_for`'s timeout. Evidence items 4 and 5 show
  the expected text becomes unreachable once a formatter self-releases, so a larger
  budget buys a slower failure and nothing else.
- Do not add the node to `tests/reproducible_flake_baseline.txt` as a plain entry.
- Do not sweep the other ~59 short unasserted `Event.wait(timeout=...)` ceilings in
  `tests/`; that is filed separately as **sase-tu** (below).
- Do not touch `tests/_load_tolerant.py`, `sase/memory/*.md`, `AGENTS.md`, or the
  generated provider shims.
- `CHANGELOG.md` is generated by release-please; describe the change in the commit
  subject/body instead.

## Follow-up filed separately

A repo-wide grep finds ~59 further sites in `tests/` where a fake worker, lock holder,
or subprocess is held by an unasserted `Event.wait(timeout=1.0)`-to-`2.0` ceiling — the
same shape as evidence item 4, including
`tests/ace/tui/test_y_keymap_non_blocking.py:105`,
`tests/ace/tui/test_agents_diff_badge_deferred.py:516`,
`tests/ace/tui/test_prompt_catalog.py:515`, and
`tests/ace/tui/test_axe_force_refresh.py:265`. Auditing those against
`LOAD_TOLERANT_TIMEOUT` is a distinct, wider judgment call — not every ceiling is wrong
— so it is filed as its own task bead, **sase-tu** (`sase bead show sase-tu`), rather
than smuggled into this one. That bead carries `related` links to the four already-filed
single-node flake beads that share the shape (sase-d1, sase-gs, sase-qp, sase-t7).
