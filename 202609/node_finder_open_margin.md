---
tier: epic
title: Land the Node Finder open budget with real margin
goal: 'The unchanged 2,000-node Node Finder benchmark passes all four budgets on consecutive
  runs. The Node Finder PNG goldens no longer depend on the wall clock. The chain
  from sase-19i.7.3.3.3.3 up to sase-19i can then close.

  '
parent_bead: sase-19i.7.3.3.3.3
phases:
- id: finder-goldens
  title: Pin the Node Finder age clock in the visual goldens
  depends_on: []
  size: small
  description: 'finder-goldens: pin the unpinned local_now read in node_finder_rendering
    for the visual fixture, regenerate only the five drifting Node Finder goldens
    after inspecting each diff, and prove 7/7 stability across two runs minutes apart.

    '
- id: open-margin
  title: Release per-open garbage and widen the open margin
  depends_on: []
  size: medium
  description: 'open-margin: stop dismissed finder modals from keeping their snapshot
    rows alive until cycle collection, cut snapshot and first-paint drain work, and
    pass the unchanged official benchmark on three consecutive runs.'
proposed_by: bbugyi200.athena.0t7
create_time: 2026-09-27 15:26:21
status: wip
bead_id: sase-19i.7.3.3.3.3.3
---

- **PROMPT:** [prompts/202609/node_finder_open_margin.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/node_finder_open_margin.md)
- **BEAD:** [sase-19i.7.3.3.3.3.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19i/sase-19i.7.3.3.3.3.3.md)

# Land the Node Finder open budget with real margin

## Where the chain stands

These numbers were measured on 2026-09-27 at master `9814d8980e`, with host load between
13 and 24 on 64 cores.

The six nested epics `sase-19i` → `.7` → `.7.3` → `.7.3.3` → `.7.3.3.3` → `.7.3.3.3.3`
are all IN_PROGRESS, and every phase bead under them is closed. The land agent for
`sase-19i.7.3.3.3.3` was killed while it was still waiting, so that epic has never had a
land review.

The following items are already resolved on master. Verify them, but do not redo them:

- **Mypy.** The `build_agent_tree` errors in epic notes #1–#2 and phase `.2` note #1
  (`prefix_key` no-redef/arg-type from `215eb89f41`, plus the snapshot errors) are gone.
  `.venv/bin/mypy src/sase` is clean, because `919ba740e6` and `2660c5c7bb` removed the
  redefinition.
- **Proc lifecycle contract.** `test_validate_proc_lifecycle_contract` passes.
- **Epic symbols.** `sase bead epic-symbols` is empty for all six epics. The
  `describe_node_finder_row_from_facts` whitelist from `sase-19i.7.3.3` note #1 has been
  removed.
- **Symvision.** `just symvision` fails only on four
  `src/sase/integrations/usage_windows.py` external-reference pragmas from `315746c2ae`.
  That is not Node Finder work.

Two defects still block the close.

### The open p95 passes only by luck

I ran the official command
`.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -q` three times on
master:

| Run  | open p50 | open p95 | open max  |
| ---- | -------- | -------- | --------- |
| pass | 42.32 ms | 49.46 ms | 50.82 ms  |
| fail | 47.13 ms | 74.05 ms | 494.33 ms |
| fail | 43.50 ms | 57.16 ms | 58.85 ms  |

The other three cases passed every time. Keystroke-dispatch p95 was about 0.01 ms,
keystroke about 0.8 ms, keystroke-broad 6.6–8.1 ms, and highlight about 0.6 ms.

The phase commit `d5387e15d9` also failed twice, run from a temporary worktree on the
same host with the same venv:

- p50 47.59 ms, p95 52.87 ms
- p50 45.14 ms, p95 398.13 ms

Phase `.2` note #3 had already recorded 2 passes and 2 failures. Later commits did not
cause a regression. The epic never had real margin.

**Stage split.** I used a same-harness probe: the `_open_page` roster with 2,012 rows,
GC enabled, and 35–55 measured warm opens.

| Stage                        | p50      | p95      | Notes                               |
| ---------------------------- | -------- | -------- | ----------------------------------- |
| `build_node_finder_snapshot` | 26–28 ms | 31–37 ms |                                     |
| `NodeFinderModal(...)`       | 1.2 ms   |          |                                     |
| `push_screen`                | 1.7 ms   |          |                                     |
| Drain to first paint         | ~16 ms   | 20–25 ms | always exactly 28 `sleep(0)` rounds |
| Total open                   | 45–48 ms |          |                                     |

The modal reports `is_mounted` on the same round that its first window materializes.

**Tail.** Every 400–490 ms sample coincided with a generation-2 (full) cycle collection,
observed through `gc.callbacks`. A full `gc.collect()` on the bench heap of about 560k
tracked objects takes 335–365 ms. These pauses land about once per 20 opens, in either
the snapshot or the drain. The bench's p95 is the second-worst of 25 measured samples,
so two pauses in one run fail it no matter how fast the body is.

**Why full collections are that frequent.** Reference counting does not free a popped
`NodeFinderModal`. I ran four open/pop cycles with GC disabled:

- Afterwards, 4 modals and 8,048 `NodeFinderRow` instances were still alive.
- One `gc.collect()` then freed 42,113 objects.

Each open therefore pushes about 10.5k cyclic-garbage objects through the young
generations into the old one. The snapshot's rows are garbage only because the dead
modal still references the snapshot and its view.

**Checked and ruled out:**

- A warm `build_node_finder_snapshot` opens no files. I patched `open`, `io.open`, and
  `Path.open` and saw zero calls. The file objects that showed up in allocation diffs
  came from background workers.
- `gc.freeze()` after startup shrank the full-collection pauses to 80–120 ms. That is
  still over budget, and it did not move the body. Process-wide GC policy is out of
  scope for this epic.

### Five Node Finder PNG goldens encode the wall clock

`src/sase/ace/tui/modals/node_finder_rendering.py` imports `local_now` from
`sase.core.time` and renders each row's age as `local_now() - started` (around line
361).

`pin_agents_visual_now` in `tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py`
patches `local_now` only in the modules it lists, and `node_finder_rendering` is not
among them. As a result, finder rows show the real age: `1733h08m` on 2026-09-27 for the
fixture's 2026-07-17 09:00 start.

Commits `d4c7b5ca9a` and `e75fa98d7b` each rebaselined the goldens, at different hours.
`.venv/bin/pytest -q -m visual tests/ace/tui/visual/test_ace_png_snapshots_agents_node_finder.py`
fails 5 of 7 nodes:

- Failing: hints, search, pending-prefix, query-hidden, and hidden-by-i.
- Passing: narrow and no-results.

The diffs are confined to the age digits. This is `sase-19i` note #5, and the defect
dates from the modal's introduction in `sase-19i.4` (`89ebc76256`).

## Phase `finder-goldens`

1. **Pin the rendering clock.** Add `sase.ace.tui.modals.node_finder_rendering` to the
   modules that `pin_agents_visual_now` pins.
2. **Check for other clock reads.** Search every Node Finder module for clock reads that
   reach rendered text: `local_now`, `datetime.now`, and `time.time`. The modules are:
   - `src/sase/ace/tui/models/node_finder.py` and `_node_finder_*.py`
   - `src/sase/ace/tui/modals/node_finder_*.py` and `_node_finder_modal_*.py`
   - `src/sase/ace/tui/actions/agents/_node_finder_*.py`

   Pin any you find the same way. Leave production age formatting unchanged.

3. **Regenerate only this file's goldens:**
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_node_finder.py`.
   Run it through `/sase_monitor` if it outruns the turn.
4. **Inspect every updated golden and its diff.** Only the age cells may change, and
   they must become the pinned value (12:00 minus each fixture start). Any other
   difference is a real rendering change. Investigate it instead of accepting it. Leave
   all other goldens untouched.

**Acceptance:**

- The check-only targeted command above passes 7/7 twice. Start the second run at least
  two minutes after the first, which proves the minute digits no longer drift.
- Record the inspection and both runs on the phase bead.
- `sase tool run check` passes. Leave `just check-full` unrun.

## Phase `open-margin`

This phase covers:

- the finder snapshot, model, and modal code:
  `src/sase/ace/tui/actions/agents/_node_finder_*.py`,
  `src/sase/ace/tui/models/node_finder.py` and `_node_finder_*.py`, and
  `src/sase/ace/tui/modals/node_finder_modal.py` and `_node_finder_modal_*.py`
- any presentation helpers that a profile shows they actually call

### Step 1: release per-open state

When the modal is dismissed or unmounted, drop its references to all per-open state:

- the snapshot and its rows
- the current view
- the per-modal view memo
- option and line caches
- preview state

The memo and the identical-rebuild skip only ever serve the live modal, so dropping them
after dismissal loses nothing.

Also find anything that pins the modal after `pop_screen`, such as closures,
bound-method callbacks, timers, debouncers, or workers. Cancel or clear each one.

The goal: with GC disabled, after `pop_screen` and a pause, no `NodeFinderRow` from that
open is still alive, and the modal's own leftover cyclic garbage is as small as
practical.

Add a focused regression test next to the modal tests. It opens and dismisses the finder
with GC disabled, re-enabling GC in `finally`. It then asserts that the snapshot or its
rows were released without calling `gc.collect()`, using a weakref or an instance count.

### Step 2: cut the body

The two budgets to cut are the snapshot (26–28 ms p50) and the drain (about 16 ms over
28 rounds).

1. **Re-profile first.** Wait until startup work is idle, profile with GC enabled, and
   charge only the frames that `print_callees` attributes to the open path.
2. **Test these candidates against the profile.** Keep or reject each based on what it
   shows.
   - **Lazy per-row fields.** Some per-row fields are needed only by the 64-row window,
     the hints, or the preview. Compute them lazily or when the window materializes,
     with identical filter, hint, and count results.
   - **Fewer tracked allocations per row.** Each open currently allocates about 2,010
     `NodeFinderRow` objects plus about 2,100 tuples.
   - **Invisible first-paint work.** Remove compose and mount work in `NodeFinderModal`
     and its widgets that is not visible at first paint.
3. **Keep first-frame work in place.** Anything visible in the first frame stays ahead
   of the first-paint predicate.

### Constraints

- **Benchmark untouched.** Leave `tests/ace/tui/bench_node_finder.py` byte-for-byte
  unchanged. That includes `_SAMPLES`, `_WARMUP_SAMPLES`, `_percentile`, the fixture,
  the timers, the first-paint predicate, and the budgets.
- **No GC tricks.** Do not use `gc.disable()`, `gc.collect()`, `gc.freeze()`, threshold
  changes, or any other device that moves collections out of the measured span. Do not
  change process-wide GC policy. If you believe policy work is warranted, record it as a
  `PROPOSED FOLLOW-UP:`.
- **No cross-open caching.** The snapshot still reads current in-memory owner state on
  every open. Never cache that state across opens.
- **Preserve behavior exactly:**
  - tree order and hidden-reason precedence
  - dismissed exclusions
  - panel, banner, fold, query, and `I`-hidden behavior
  - the `◆` here row
  - hint assignment, counts, and navigation identities
  - `NAMED_PROC`, gate, and monitor classification
  - the per-modal view memo (cap 8)
  - the identical-rebuild skip, including the guard that makes a cursor move followed by
    a refilter recenter
  - the first-paint predicate
  - the 150 ms Tier 1 debounce
  - broad, narrow, and highlight scoring, order, and context rows
- **Keep the other three budgets green.**

### Acceptance

1. **Release test.** The release regression test passes.
2. **Probe comparison.** In one session, run a probe of at least 40 warm opens before
   the change and again after it.
   - Open p50 must be at least 8 ms lower after the change.
   - Record both distributions (snapshot, drain, total).
   - Record the count of generation-2 collections per 40 opens, before and after.
3. **Official benchmark.** The unchanged official command
   `.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -q` exits 0 on three
   consecutive runs.
   - A failing run resets the count. Report every run, not just the passes.
   - For each run, record p50, p95, and max for all four cases, plus the `uptime` load.
4. **Focused tests.** The focused Node Finder tests pass: model, modal, snapshot,
   preview, ladder/reveal, and the `agent_groups` tree suites.
5. **Check.** `sase tool run check` passes. Leave `just check-full` unrun.

A miss that also reproduces on a tree without this phase's diff stays in this phase
until the benchmark passes.

## Handoff

`parent_bead: sase-19i.7.3.3.3.3` hands this epic back to the chain. The user asked for
the whole chain to be closed.

This epic's land agent runs its own land first. It then resumes the interrupted landing
of `sase-19i.7.3.3.3.3` and climbs through the ancestors, in this order:

1. `sase-19i.7.3.3.3`
2. `sase-19i.7.3.3`
3. `sase-19i.7.3`
4. `sase-19i.7`
5. `sase-19i`

It closes each ancestor that is still complete and sets that ancestor's linked plan file
to `status: done`. It stops only at an ancestor that is genuinely incomplete, records
the blocker there, and reports it.

These items are already triaged for those landings. Recheck them, but do not rediscover
them.

### `sase-19i.7.3.3.3.3`

- **Epic notes #1–#2:** fixed on master (see above).
- **Phase `.1` #3:** contended-host spikes blamed on bead-store scans. Decline. The GC
  evidence behind `open-margin` explains those spikes.
- **Phase `.2` #1:** clean-base `just check` red. Mypy and the proc-lifecycle test now
  pass. The remaining Symvision `usage_windows.py` pragma errors come from `315746c2ae`.
  Route them through `/sase_new_task` unless an owner already tracks them.
- **Phase `.2` #3:** the thin margin, which is `open-margin`.

### Integration since `215eb89f41`

The commits that touch finder code are:

- refactors: `65bd149a0c` (snapshot split), `86fac2f103` (model split), `faaf69757b`
  (modal mixins), and `2660c5c7bb` (tree split)
- the `named_proc` rename, `d4c7b5ca9a`
- the wall-clock golden rebaselines `d4c7b5ca9a` and `e75fa98d7b`, which
  `finder-goldens` fixes

The active epic `sase-1bc` (agent sub-tabs) does not touch finder code yet. How the
finder treats nodes on other agent tabs belongs to its phase `sase-1bc.6` (cross-tab
navigation), not to this chain. Say so in the close note.

### `sase-19i.7.3.3.3`, `.7.3.3`, `.7.3`, and `.7`

Their notes route all remaining work to the open budget, which this epic finishes. The
`sase-19i.7.3.3` note #1 whitelist entry is already gone.

### `sase-19i`

- **Note #5:** fixed by `finder-goldens`.
- **Note #4:** the `sase-19i` landing still owes the final non-causal triage of every
  `PROPOSED FOLLOW-UP:` note on phases `sase-19i.1` through `sase-19i.6`.
