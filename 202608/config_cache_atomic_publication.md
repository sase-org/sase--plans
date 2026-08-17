---
tier: tale
title:
  Finish isolating the process-global merged-config cache - atomic publication, honest
  single flight, and declared evidence retirement
goal: "`load_merged_config()` and `get_agent_owner_config_snapshot()` publish their
  (token, value) pair in one atomic store and refuse to publish a build whose cache
  generation was invalidated while it ran; a stale `sase-config-token-refresh` worker
  deregisters only its own registration instead of the live one; the pre-fix failure
  evidence for the `tests/test_config_cache*.py` class is retired by a declared `#
  fixed-at:` block naming bead `sase-mv` and commit `2959d3992` instead of being
  silently excluded by a file rename; and the residual non-main-thread config readers
  are re-measured on current master and either fixed at their owner or recorded with
  numbers. `just selection-health --fail-on-new-flake` names no
  `tests/test_config_cache*.py` node.

  "
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.6.6.1
bead: sase-ns.6.6.6.1
create_time: 2026-08-17 12:34:42
status: wip
---

- **PARENT:**
  [202608/backlog_top_five_gates_and_flakes.md](backlog_top_five_gates_and_flakes.md)
- **BEAD:**
  [sase-ns.6.6.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.6.1.md)

# Plan: Finish isolating the process-global merged-config cache

## Bead

This plan implements the remainder of phase bead `sase-ns.6.6.6.1` (`configcache`) of
epic `sase-ns.6.6.6`, plan `202608/backlog_top_five_gates_and_flakes.md`. That phase
owns task bead `sase-mv`, which is already **closed**. Read the
`Phase configcache — bead sase-mv` section of the epic plan first; this plan does not
restate its evidence.

Two earlier plans covered ground that is now landed or superseded. Do not redo them:

- `202608/config_cache_ambient_reader.md` (bead `sase-mv`) — implemented by `2959d3992`.
- `202608/config_cache_publication_isolation.md` (this bead, never approved) — its
  Defect 3 is now fixed by `2959d3992`. Its **Defect 1 and Defect 2 are still present in
  `src/sase/config/core.py` on current master** and are steps 1 and 2 below. That plan's
  root-cause analysis is still accurate for those two; re-read it if you want the long
  derivation.

## State of the phase on master `c4a29f213`

Landed by `2959d3992` "fix(ace-tui): stop leaked proc-observer threads between tests":

- `ProcObserver` instances are tracked and orphans are stopped from the autouse
  `_clear_config_caches` fixture (`tests/_conftest_runtime.py`
  `_stop_orphaned_proc_observers_if_loaded`) and from `AcePage` teardown, including the
  failed-enter path. This was the proximate poisoner.
- `_drain_config_token_refresh` no longer nulls a later test's live single-flight slot
  on join timeout.
- The four fragile `assert load_merged_config() is first` in-situ checks became deltas,
  and `test_load_merged_config_serves_stale_while_refreshing` now covers the stale-serve
  contract deterministically.
- The opt-in `--sase-detect-config-readers` probe (`tests/_config_reader_probe.py`) is
  permanent, with its own unit coverage.

Measured in this workspace on `c4a29f213`, so you do not have to re-derive it:

- Four consecutive full-lane records with **zero failures** at or after the fix:
  `20260817T143226Z-5abf9eb64e3c-305699`, `20260817T144608Z-5abf9eb64e3c-620023`,
  `20260817T153355Z-2959d3992cc0-1424366` (head is the fix commit, clean tree), and
  `20260817T160823Z-c3da174ea124-2014083`.
- `just selection-health --fail-on-new-flake` exits 1 naming exactly one node,
  `tests/test_models_panel_edit_outcomes.py::test_on_alias_edited_offers_commit_when_in_repo`,
  which is ready task bead `sase-oh` and is **not** this phase's. No
  `tests/test_config_cache*.py` node is named.

So the phase's symptom is gone. What is left is that the class is still resting on two
unfixed mechanisms and on an accidental gate exclusion, all three of which this plan
closes.

## Step 1 — Publish the merged-config and owner-snapshot caches atomically

`load_merged_config()` (`src/sase/config/core.py:628-658`) and
`get_agent_owner_config_snapshot()` (`src/sase/config/core.py:547-559`) each read and
write a `(token, value)` pair as **two independent, unlocked global assignments**, token
first:

```python
_merged_config_cache_token = token          # published first
_merged_config_cache_value = result         # published second
```

A concurrent reader that lands between those two statements sees the new token with the
old value. If its own token happens to equal the newly published one, the pair looks
consistent and it returns a config that does not correspond to it — and keeps returning
it until the token changes again. That is a production correctness bug, independent of
tests; the token cache next to it is fully serialized by
`_current_config_token_cache_lock`, and these two caches are protected by nothing.

Fix it without putting a lock on the read path, which is hot and budgeted (see the cost
gate note below):

1. Replace `_merged_config_cache_token` / `_merged_config_cache_value` with a single
   module global holding an immutable pair, and the same for
   `_agent_owner_config_cache_token` / `_agent_owner_config_cache_value`. A read is one
   attribute load; a publish is one attribute store. Both are atomic against other
   threads, so no reader can observe a mismatched pair.
2. Refuse to publish a build that a `clear_config_cache()` invalidated while it ran.
   Capture `_config_cache_generation` before the merge and publish only if it is
   unchanged afterwards. This is production-honest — the result is known stale — and it
   is what stops a reader that began before a test boundary from installing its result
   into the next test's generation.
3. Update `clear_config_cache()` (`src/sase/config/core.py:214-228`) to reset the new
   slots, and `tests/_conftest_runtime.py` plus any test that pokes the old global
   names. `grep -rn '_merged_config_cache_\|_agent_owner_config_cache_' src tests`
   currently finds references only inside `src/sase/config/core.py`; re-run it, because
   the names are also plausible targets for a `monkeypatch.setattr`.

**Do not change caching semantics.** Stale-while-revalidate, the `CONFIG_DIR` binding
added by `3a22ff04f`, and the generation-in-token behaviour all stay exactly as they
are. This step removes a data race; it must not remove or add a cache hit.

On coverage, be honest about what is testable. The torn-read window is two adjacent
bytecode stores; you cannot park a thread between them from Python, so **no test can
reproduce the old bug and no test should be written trying**. Sub-step 1 is justified by
construction — one store cannot be observed half-done — and its regression guard is
structural. Sub-step 2 is genuinely testable. So:

- In `tests/test_config_cache.py`, a deterministic generation-guard test: patch
  `merge_config_sources` with a builder that blocks on a `threading.Event` the test
  owns, start `load_merged_config()` on a second thread, call `clear_config_cache()`
  from the main thread while it is parked, release it, and assert the next
  `load_merged_config()` returns the post-clear config and that the discarded build
  never became the cached value. Repeat the shape for
  `get_agent_owner_config_snapshot()` with `_build_agent_owner_config_snapshot`.
- A structural test that the pair is one slot: after a load, the new global is a 2-tuple
  whose token equals `current_config_token()` and whose value is the object
  `load_merged_config()` returns; after `clear_config_cache()` it is `None`. That is
  what stops a later refactor from splitting the pair apart again.
- Put both in `tests/test_config_cache.py` — that file is 235 lines after the
  `c715bacbc` split and has room. `tests/_config_cache_helpers.py` already holds the
  shared writers and bounded-wait pollers; add any new helper there rather than
  duplicating it, and mark any bounded wait with the repo's wait pragma the way
  `644177a88` did for the drain sleeps.

## Step 2 — Let only the live refresh worker deregister itself

`_refresh_current_config_token()` (`src/sase/config/core.py:150-167`) correctly refuses
to publish a token when its `cache_epoch` is stale, and then clears the worker slot
**unconditionally**, outside that guard:

```python
with _current_config_token_cache_lock:
    if cache_epoch == _current_config_token_cache_epoch:
        ...                                        # correctly skipped when stale
    _current_config_token_refresh_thread = None    # runs even when stale
```

A worker that missed its drain window and finally runs inside a later test therefore
deregisters **that test's live worker**, so the next expired read starts a second one —
which is exactly the contract
`tests/test_config_cache_token.py::test_current_config_token_refresh_is_single_flight`
asserts. `2959d3992` fixed the mirror-image bug on the test-harness side
(`_drain_config_token_refresh` no longer nulls a live slot); the production side is
untouched.

Clear the slot only when it still holds this thread:

```python
if _current_config_token_refresh_thread is threading.current_thread():
    _current_config_token_refresh_thread = None
```

Add a regression test next to the existing token tests
(`tests/test_config_cache_token.py`) that registers a stale worker, bumps the epoch,
lets a live worker register, runs the stale worker's publish path, and asserts the live
registration survives.
`tests/test_config_cache_teardown.py::test_prior_refresh_worker_cannot_publish_after_drain`
is the model for driving a stale worker deliberately.

## Step 3 — Re-measure the residual non-main-thread config readers

`sase-mv`'s step-1 full-lane probe run (monitor `qdw2tbd135ka`, pre-fix tree) recorded
29852 ambient config calls with **520 poisoning reads** from two thread families. The
`sase-ace-proc-observer` family is what `2959d3992` fixed. The second family — the
`asyncio_0..6` Textual thread-worker pool, reaching config through
`sase.ace.tui.widgets.llm_override_indicator` → `build_launch_model_setting_snapshot` →
`get_llm_provider_config` (see
`src/sase/ace/tui/widgets/llm_override_indicator.py:140-168`, a
`run_worker(..., thread=True)`) — was left in place and recorded on this bead by
`sase-ns.6.6.6.1--…` on 2026-08-17T14:23:38Z.

A partial re-measurement was started in this workspace on `c4a29f213`
(`-n 2 --sase-detect-config-readers` over `tests/ace/tui` plus every
`tests/test_config_cache*.py` file and `tests/test_llm_override_indicator.py`) and had
not finished when this plan was written. **Re-run it yourself; do not trust a number
this plan does not print.** The cheap form is the same command; the authoritative form
is the full lane through `/sase_monitor`:

```bash
sase monitor start \
  --command 'just test -- -p tests._config_reader_probe --sase-detect-config-readers' \
  --reason 'Re-measure residual ambient config readers after the proc-observer fix' \
  --timeout 40m \
  --next 'Read .pytest_cache/sase-config-readers.json and continue step 3 of plan 202608/config_cache_atomic_publication.md for bead sase-ns.6.6.6.1.'
```

Then act on `poisoning_reads` and `cross_test_threads` in the report:

1. **Zero poisoning reads** — say so with the number in the bead note and go to step 4.
2. **A bounded, owned leak** — for instance the `LLMOverrideIndicator`
   default-resolution worker still running after its app exited. Fix it where it is
   started, preferring a test-lifecycle fix (awaiting or cancelling the
   `_DEFAULT_WORKER_GROUP` workers in `AcePage.__aexit__`,
   `src/sase/ace/testing/ace_page.py:241`) over a production change, and add a focused
   regression test in the style of `tests/test_proc_observer_isolation.py`.
3. **A reader with no owner you can fix inside this plan** — Textual's default executor
   is shared and its threads outlive individual apps, so this is a real possibility.
   Record it as
   `sase bead note sase-ns.6.6.6.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`
   with the measured counts and the captured stack, and go to step 4. **Do not create a
   bead** — the epic's land agent triages these. Say in the close note which of these
   three branches you took.

Steps 1 and 2 stand on their own and do not depend on this step reproducing anything.

## Step 4 — Retire the class's pre-fix evidence by declaration, not by accident

`tests/reproducible_flake_baseline.txt` has no `# fixed-at:` block for this class. The
gate is currently quiet for it only because the split in `c715bacbc` moved the named
node out of `tests/test_config_cache.py`, so `selection_health` reports it as
`1 recorded node ID(s) no longer collectable (renamed or deleted test); excluded as stale`.
That is an accident of a file rename, not a declared fix, and it does not cover the
class members that kept their node IDs. Counted in this workspace from
`~/.sase/test-selection/gh_sase-org__sase`, these still-collectable node IDs carry
pre-fix records inside the gate's window, including some recorded **after** the
`sase-nv` `# fixed-at:` instant that block already declares:

| node ID (as recorded)                                                                             | records | latest record       |
| ------------------------------------------------------------------------------------------------- | ------- | ------------------- |
| `tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`           | 14      | 2026-08-17T09:09:32 |
| `tests/test_config_cache.py::test_load_merged_config_caches_default_layer`                        | 13      | 2026-08-16T21:46:49 |
| `tests/test_config_cache.py::test_clear_config_cache_forces_reload`                               | 10      | 2026-08-16T23:03:29 |
| `tests/test_config_cache.py::test_load_merged_config_eventually_invalidates_on_file_mtime_change` | 7       | 2026-08-16T21:50:13 |
| `tests/test_config_cache.py::test_load_merged_config_invalidates_on_include_local_toggle`         | 5       | 2026-08-16T10:40:50 |
| `tests/test_config_cache.py::test_load_merged_config_caches_plugin_layer`                         | 4       | 2026-08-17T10:46:16 |
| `tests/test_config_cache.py::test_owner_snapshot_reuses_parsed_overlay_until_token_changes`       | 2       | 2026-08-16T11:06:12 |
| `tests/test_config_cache.py::test_drain_config_token_refresh_joins_worker_and_advances_epoch`     | 1       | 2026-08-17T11:40:51 |

Add one block to `tests/reproducible_flake_baseline.txt` following the convention
already in its header and used by the `sase-nv`, `sase-md`, and `sase-ob` blocks: a
comment naming bead `sase-mv`, this phase bead, and commit `2959d3992`, then one
`# fixed-at: 2026-08-17T15:32:01Z <node id>` line per node. `2026-08-17T15:32:01Z` is
`2959d3992`'s committer instant (`git show -s --format=%cI 2959d3992`).

Two rules for this step:

- **Only declare node IDs that actually retire evidence.** `selection_health` prints
  `# fixed-at: entries ... retired nothing in the current window and can be removed` for
  dead directives. Run the gate after editing and delete every entry it names. The table
  above is a starting set counted at plan time, not a specification.
- **Add nothing to the node-ID section.** Those entries are suppressions and are debt; a
  `# fixed-at:` directive is a retirement of pre-fix evidence and is not.

One honest wrinkle to carry into the bead note rather than paper over:
`tests/test_config_cache_teardown.py::test_prior_refresh_worker_cannot_publish_after_drain`
has a single failure record at `2026-08-17T15:40:08Z`, eight minutes **after** the fix
instant, from workspace `sase_14` running head `1482fc1dc` — an ancestor of the fix.
That is exactly the stale-tree limitation the baseline header documents. It is one
record, so it does not reach the gate today. Do **not** back-date the `fixed-at` instant
to cover it and do **not** add it to the node-ID section; if it ever reaches the gate it
is live evidence and belongs to a new investigation.

## Verification

1. `just install`, then `just check` inline. If it takes long, hand it to
   `/sase_monitor` with a `--next` action.
2. The class passes in isolation and under file-scoped contention:

   ```bash
   .venv/bin/python -m pytest -p no:randomly \
     tests/test_config_cache.py tests/test_config_cache_token.py \
     tests/test_config_cache_selector.py tests/test_config_cache_teardown.py \
     tests/test_config_cache_isolation.py
   SASE_CONTENTION_REPEAT=3 just test-contention tests/test_config_cache.py
   ```

3. `just check-full` through `/sase_monitor` with a `--next` action, never inline. Every
   node in `tests/test_config.py` and the five `tests/test_config_cache*.py` files must
   be green. The `sase-j0` suite-cost budget failure is pre-existing and is not this
   phase's to fix — but the `config_load_merged` count budget (`count: 4924` in
   `tests/perf/baselines/test_cost_baseline.json`) **is** a direct check on step 1: a
   generation guard that refuses too many publishes shows up there as a merge-count
   explosion. An earlier, broken attempt at first-fill coalescing on this bead pushed
   that count to 32810, so treat any large move in it as a step-1 regression, not as
   noise.
4. `just selection-health --fail-on-new-flake`, recorded honestly. The bar for this
   phase is that **no `tests/test_config_cache*.py` node is named**. The gate may still
   exit 1 on
   `tests/test_models_panel_edit_outcomes.py::test_on_alias_edited_offers_commit_when_in_repo`
   (task bead `sase-oh`) and on the `sase-n4`-owned fakey node if new records arrive;
   neither is this phase's, and neither may be presented as this phase failing or as a
   reason to widen the baseline.

## Closing the bead

1. `sase bead epic-symbols sase-ns.6.6.6.1` before closing. Resolve any leftover
   `--epic-symbol` entries or re-key the Justfile line to a still-open bead. There were
   none at plan time.
2. `sase bead close sase-ns.6.6.6.1 --note "<what you verified>"`. The note must name:
   which of step 3's three branches you took and the measured poisoning-read count, the
   `check-full` monitor id, and the `selection-health` state in full.
3. Close **only** this phase bead. Do not close `sase-ns.6.6.6`, `sase-ns.6.6`,
   `sase-ns.6`, or `sase-ns`. `sase-mv` is already closed; do not reopen it.
4. Record discovered work as
   `sase bead note sase-ns.6.6.6.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.
   Do not create beads.

## Non-goals

- Reopening or re-fixing `sase-mv`. It is closed on two green full lanes plus two more
  since.
- `tests/test_models_panel_edit_outcomes.py::test_on_alias_edited_offers_commit_when_in_repo`
  (ready task bead `sase-oh`) and
  `tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error`
  (epic `sase-n4`). Neither belongs to this phase even though both touch the same gate.
- The `sase-j0` suite-cost budget gate and the `sase-j7` process-global leak epic.
- Adding any node ID to the suppression section of
  `tests/reproducible_flake_baseline.txt`, or adding a retry or a sleep to any
  config-cache test.

## Risks

- **The read path is hot.** `load_merged_config` is called thousands of times per suite
  and its count is budgeted. Keep the read to a single global load and a tuple compare;
  do not introduce a lock, a property, or a function call on that path.
- **The generation guard could suppress a legitimate publish** if the generation is
  bumped by something other than `clear_config_cache()`. It is not today —
  `grep -n '_config_cache_generation' src` shows one increment site — but re-check
  before relying on it, and prefer a missed publish (one extra merge) to a wrong
  publish.
- **Step 3 may find nothing to fix.** Textual's default executor is process-shared and
  its threads legitimately outlive any single app, so branch 3 (record and move on) is
  an expected outcome, not a failure.
