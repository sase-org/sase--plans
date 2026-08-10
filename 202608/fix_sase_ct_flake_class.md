---
tier: epic
title:
  Fix the sase-ct flake class at its root - process-global state leaking between tests
goal:
  The tests behind sase-ct stop failing under the full parallel lane because the
  process-global state that leaks between tests is fixed by mechanism, a leak detector
  gate makes the class non-recurring, tests/reproducible_flake_baseline.txt shrinks to
  only nodes proven still broken, and sase-ct, sase-iy.5, sase-j4, sase-j5, and sase-j6
  are closed on evidence.
phases:
  - id: vcs-cache
    title: Fix the confirmed xprompt VCS-tag cache leak
    depends_on: []
    size: medium
    description:
      vcs-cache - give the caches derived from workspace-provider metadata a real
      invalidation entry point and restore them on teardown, so a test that fakes plugin
      metadata stops poisoning every later test in its worker.
  - id: leak-detector
    title: Build a global-state leak detector and inventory every leak in the suite
    depends_on: []
    size: medium
    description:
      leak-detector - build an opt-in pytest plugin that snapshots process-global state
      around every test, distinguishes cache warming from poisoning, and delivers a
      full-suite inventory artifact of every leak. Reports only; blocks nothing.
  - id: stale-nodes
    title: Stop the flake gate from flagging node IDs that no longer exist
    depends_on: []
    size: medium
    description:
      stale-nodes - bead sase-j5. Make the reproducible-flake gate skip or separately
      report recorded node IDs absent from the collected suite, so a renamed test stops
      manufacturing pressure to bump the baseline cutoff.
  - id: fix-leaks
    title: Fix every inventoried leak and root-cause the residual flakes
    depends_on:
      - vcs-cache
      - leak-detector
    size: large
    description:
      fix-leaks - fix every poisoning leak in the inventory by mechanism,
      deterministically reproduce and fix the bead-cluster and plan-approval nodes whose
      cause is not yet known, and flip the leak detector into a blocking gate so the
      class cannot recur.
  - id: retire
    title: Shrink the baseline, run the exit criteria, and close the beads
    depends_on:
      - vcs-cache
      - leak-detector
      - stale-nodes
      - fix-leaks
    size: medium
    description:
      retire - remove every fixed node from the flake baseline, run the four exit
      criteria non-vacuously on the combined tree, and close sase-j4, sase-j5, sase-j6,
      sase-ct, and sase-iy.5 on that evidence.
proposed_by: bbugyi200.athena.sase-j0.w1.f0
create_time: 2026-08-10 15:44:26
status: wip
---

- **PROMPT:**
  [prompts/202608/fix_sase_ct_flake_class.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fix_sase_ct_flake_class.md)

# Fix the `sase-ct` flake class at its root

## Why this plan exists

`sase-ct` has been closed and reopened eight times across two prior epics (`sase-h8`,
`sase-h8.10`) and 60 `+1`s. Every previous attempt treated the class as _timing
flakiness_ — TUI waits, contention, xdist scheduling — and every attempt ended in either
a wait-idiom fix that helped a few nodes or a bookkeeping move (baseline cutoff bumps,
new umbrella beads) that cleared the gate without fixing anything.

**That diagnosis was wrong.** The class is not timing. It is **deterministic
order-dependent failure caused by process-global state leaking from one test into the
next**, amplified into apparent randomness by `--dist=worksteal`.

This plan fixes the actual mechanism. It does **not** bump the flake-baseline cutoff
again, and it does **not** file another umbrella.

## The root cause, confirmed

### The mechanism

`sase/xprompt/_parsing_vcs_tags.py:14-16` holds three module-level caches:

```python
_VCS_TAG_PATTERN: re.Pattern[str] | None = None
_VCS_TAG_EMBEDDED_PATTERN: re.Pattern[str] | None = None
_VCS_REPLACE_PATTERN: re.Pattern[str] | None = None
```

They are lazily built (`_get_vcs_tag_pattern`, `_parsing_vcs_tags.py:24-31`) from
`sase.workspace_provider.get_vcs_tag_pattern()`, which derives from
`get_all_workflow_metadata()` — itself a `functools.cache` over plugin discovery
(`sase/workspace_provider/_registry.py:46-49`). `sase/xprompt/_parsing.py:71-99` keeps a
second mirrored copy of the same three globals, synced by hand.

`tests/_workspace_provider_helpers.py` lets tests swap that metadata for a fake:
`patch_git_metadata` (only `git`), `patch_spy_metadata` (only `spy`), and
`patch_no_workspace_metadata` (**empty**). Each calls `_reset_xprompt_vcs_caches()`
(`tests/_workspace_provider_helpers.py:12-26`) to clear the derived globals **at
setup**.

**There is no teardown.** `monkeypatch` restores `get_all_workflow_metadata` when the
test ends, but the derived pattern globals keep whatever was compiled from the _fake_
metadata. Every later test in that worker process sees the poisoned pattern.

### Proof — instrumented

A `pytest_runtest_protocol` hookwrapper printing `_VCS_TAG_PATTERN.pattern` before and
after each test, run over `tests/test_removed_hg_workspace_workflow.py`:

```text
[LEAKPROBE] tests/test_removed_hg_workspace_workflow.py::test_retired_workflow_has_no_core_fallback
    before=None
    after='^#(?:)(?:!!|\\?\\?)?(?:\\([^)]*\\)|\\+|[_:][^\\s]*|)\\s'
[LEAKPROBE] tests/test_removed_hg_workspace_workflow.py::test_registered_fake_provider_uses_generic_paths
    before='^#(?:)(?:!!|\\?\\?)?(?:\\([^)]*\\)|\\+|[_:][^\\s]*|)\\s'
    after='^#(?:spy)(?:!!|\\?\\?)?(?:\\([^)]*\\)|\\+|[_:][^\\s]*|)\\s'
```

The file exits leaving a pattern that matches only `#spy`. Nothing restores it.

### Proof — deterministic reproduction

```bash
# Passes alone:
.venv/bin/python -m pytest -q \
  "tests/ace/tui/widgets/test_agent_display_xprompt.py::TestAgentXPromptRendering::test_agent_xprompt_highlights_warm_catalog_skills"
# -> 1 passed

# Fails every time when the poisoner runs first in the same process:
.venv/bin/python -m pytest -q \
  tests/test_removed_hg_workspace_workflow.py \
  "tests/ace/tui/widgets/test_agent_display_xprompt.py::TestAgentXPromptRendering::test_agent_xprompt_highlights_warm_catalog_skills"
# -> AssertionError: assert [('tmp', True)] == [('sase', True)]
```

The failure is exact and explainable. `known_xprompt_skill_names`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_xprompt_highlighting.py:18-27`) calls
`extract_vcs_workflow_tag("#git:sase Use /sase_plan")`. Against the poisoned `spy`-only
pattern that returns `None`, so the function falls through to
`Path(agent.project_file).parent.name` — which under `tmp_path` is `"tmp"`. Hence
`('tmp', True)` instead of `('sase', True)`.

The same poisoner also reproduces **two of the seven current baseline entries**:

```bash
.venv/bin/python -m pytest -q \
  tests/test_removed_hg_workspace_workflow.py \
  "tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_directory_key_spelling_also_resolves" \
  "tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name"
# -> 2 failed, 2 passed   (AssertionError: assert 'proj/thing' in {})
```

### Why it looked like a timing flake

`tools/run_pytest:159` sets `DEFAULT_PYTEST_DIST = "worksteal"`. Worksteal rebalances
tests across workers _dynamically, based on how fast each worker is going_. Which tests
share a worker process — and in what order — therefore changes from run to run with
machine load. A deterministic "test A poisons test B" bug becomes a test that fails only
under a loaded full parallel run and always passes in isolation.

**That is the entire `sase-ct` signature.** It is why isolation passes never proved
anything and why every contention-oriented fix only ever caught the subset whose
poisoner happened to be a wait idiom.

### The fix shape, already validated

Prototyped as an autouse fixture that snapshots the six mirrored globals before each
test and restores them if they changed, plus
`sase.history.prompt_metadata._workflow_names.cache_clear()`. With it loaded, the
poisoner-then-victim sequence for all three confirmed nodes:

```text
5 passed in 1.05s
```

## Scope boundaries

- **Do not bump `# effective-after:` in `tests/reproducible_flake_baseline.txt`.** It
  was already bumped to `2026-08-10T16:50:24Z` by `83bb8a6f7`. Bumping again is the move
  this plan exists to stop.
- **Do not add node IDs to the baseline.** The only permitted edit to that list is
  _removal_, and only in the `retire` phase, only for nodes with a fix commit.
- **Do not treat an isolation pass as evidence of a fix.** Passing alone while failing
  under the full lane is the definition of the class. Evidence is: the poisoner-then-
  victim reproduction fails before your change and passes after it.
- **Do not file another umbrella bead.** Node-specific beads only.
- Recorded selection-store failure data has node IDs but no tracebacks — do not plan on
  mining it for stack traces.

## Phase `vcs-cache` — Fix the confirmed xprompt VCS-tag cache leak

Depends on: nothing.

### What to build

The underlying defect is that derived caches have **no invalidation link to their
source**: anyone who clears `get_all_workflow_metadata` today silently leaves six stale
derived globals behind. Fix that, do not just patch the test helper.

1. Add a single public invalidation entry point in `sase.workspace_provider` — e.g.
   `reset_workflow_metadata_caches()` — that clears `get_all_workflow_metadata` and
   every cache derived from it: both mirrored copies of `_VCS_TAG_PATTERN`,
   `_VCS_TAG_EMBEDDED_PATTERN`, `_VCS_REPLACE_PATTERN`, plus
   `sase.xprompt._parsing_vcs_refs._VCS_UNDERSCORE_NORMALIZER`,
   `_LAUNCH_XPROMPT_AT_REF_RE`, and `sase.history.prompt_metadata._workflow_names`.
2. Make `tests/_workspace_provider_helpers.py` restore state on **teardown** as well as
   setup. Converting `patch_*_metadata` to fixtures, or having them register a
   `monkeypatch`-scoped finalizer, are both acceptable; the requirement is that no test
   can leave a derived pattern compiled from fake metadata.
3. Add an autouse fixture in `tests/conftest.py` as the backstop, next to the existing
   `_restore_working_directory` / `_isolate_sase_home` fixtures it should read like.
   Snapshot-and-restore is cheap; do not make it unconditionally clear caches, which
   would cost a plugin-discovery rebuild per test.

You may instead collapse the duplicated mirroring in `sase/xprompt/_parsing.py:71-99`
down to a single cache owned by the registry. That is a cleaner fix and is in scope —
but `tests/test_special_cases.py:29,54` patches `sase.xprompt._parsing._VCS_TAG_PATTERN`
by name, so the names must keep working.

### Exit criteria

- Both reproduction commands in "Proof — deterministic reproduction" pass.
- A new regression test asserts that after a `patch_spy_metadata` /
  `patch_no_workspace_metadata` test tears down, `extract_vcs_workflow_tag("#git:x ")`
  still resolves. This test must fail on the parent commit.
- `just check` green.

Do **not** remove the two `test_prompt_bar_xprompt_selector_requests.py` nodes from the
baseline here — that happens in `retire`, after the full-lane criteria have run.

## Phase `leak-detector` — Build a global-state leak detector and inventory the suite

Depends on: nothing. Runs in parallel with `vcs-cache`; it only reports.

### What to build

A pytest plugin, opt-in behind a flag (e.g. `--sase-detect-global-leaks`), that
snapshots process-global state before and after each test and reports tests that mutate
it without restoring. The working prototype:

```python
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_protocol(item, nextitem):
    before = _snapshot()
    yield
    after = _snapshot()
    if after != before:
        report(item.nodeid, before, after)
```

`_snapshot()` must cover, at minimum:

- Every module-level global in loaded `sase.*` modules whose name starts with `_` and
  whose value is `None`, a compiled pattern, `dict`, `set`, `list`, or `frozenset` (~42
  such globals exist today).
- Every `functools.cache` / `lru_cache` wrapper reachable from loaded `sase.*` modules
  (~42 exist today) — compare `cache_info()`.
- `os.environ` keys, `sys.path`, and the working directory, so the detector subsumes
  what `_restore_working_directory` and `_clear_agent_env_vars` handle ad hoc.

Use cheap fingerprints, not deep copies: this runs across ~28k tests.

Expect noise from caches legitimately warming from cold. Distinguish **warming** (`None`
-> value, or a cache growing) from **poisoning** (value -> _different_ value, or a cache
shrinking/being cleared). Report the second class; count and summarize the first.
Getting this distinction right is the core of the phase — a detector that reports every
cache warm-up is useless.

### What to deliver

Run it over the full suite and produce a durable inventory artifact — use
`/sase_artifact_file` — listing every poisoning leak: the leaking test node ID, the
global it leaves changed, and the before/after fingerprints. Include the count of
warming-only mutations so the next phase knows what was filtered out.

Deliberately check whether any leak plausibly explains the four `tests/test_bead/`
snooze/plus-one baseline nodes and
`tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`.
Their causes are **not** yet known — see "What is not yet root-caused" below.

### Exit criteria

- The detector, run over `tests/test_removed_hg_workspace_workflow.py` on the **parent**
  commit of `vcs-cache`'s fix, reports the `_VCS_TAG_PATTERN` leak. That is its
  calibration: a detector that misses the one leak we have already proven is not
  working.
- A full-suite run completes and the inventory artifact exists.
- The detector is not yet wired into any blocking gate.
- `just check` green.

## Phase `stale-nodes` — Stop the flake gate from flagging node IDs that no longer exist

Depends on: nothing. This is bead `sase-j5`.

`reproducible_flake_nodeids` (`tests/_test_selection_health.py:183`) calls a node a
reproducible flake when it fails across two disjoint change sets with no independent
interleaved pass. When a flaky test is **renamed or deleted**, its old node ID can never
appear in a passing run again — there is no test left to run under that name — so it
stays flagged forever until someone bumps the baseline cutoff past every historical
failure. That is a gate that manufactures pressure to do the exact bookkeeping move this
epic is trying to end.

Concrete instance:
`tests/test_run_pytest_main.py::test_main_cost_mode_arms_only_the_cost_recorder` was
renamed to `test_main_cost_mode_arms_cost_and_health_recorders` by `1417de7db`; the old
ID had a 14:33Z failure and could never pass again. `83bb8a6f7`'s cutoff bump cleared it
incidentally, which means it will silently recur on the next rename.

### What to build

The gate should skip — or report separately as _stale_, not as a live reproducible flake
— any recorded node ID absent from the currently collected suite. A stale-node report
should be visible in `--explain` output rather than silently dropped, so dead IDs are
still removable debt rather than invisible.

Also address, in the same phase, the ordering-oracle gap noted on `sase-j5`: of 201
distinct record heads in the store at authoring time, `git_commit_order_oracle` failed
to resolve exactly 1 (a commit from a workspace not present in that checkout).
Unresolved heads fall back to sequence order and sort last
(`_ordered_flake_candidate_runs`, `tests/_test_selection_health.py:244`). At 1/201 this
is not distorting verdicts today, but it is the mechanism by which cross-workspace
records get mis-ordered. Making unresolved heads visible in `--explain` is sufficient;
re-designing the oracle is not in scope.

### Exit criteria

- A test proves a recorded-but-uncollectable node ID is not reported as a live
  reproducible flake.
- `just selection-health --fail-on-new-flake` still exits 0 and still flags live nodes —
  verify non-vacuity as described in `retire`, not by assuming.
- `just check` green.

## Phase `fix-leaks` — Fix every inventoried leak and root-cause the residual flakes

Depends on: `vcs-cache`, `leak-detector`.

### What to do

1. Read the inventory artifact from `leak-detector`. For each poisoning leak, fix it by
   mechanism, preferring the `vcs-cache` shape: a real invalidation entry point on the
   production side, plus teardown in whatever test helper introduced the leak. An
   autouse blanket restore is an acceptable backstop, not the primary fix.
2. For each of the five nodes below, reproduce the failure deterministically (poisoner
   first, victim second, same process), fix it, and show the same command passing:
   - `tests/test_bead/test_snooze_gate.py::test_bead_snooze_gate_preview_carries_the_real_snooze_note`
   - `tests/test_bead/test_snooze_lifecycle.py::test_cancel_snooze_returns_the_bead_to_ready`
   - `tests/test_bead/test_snooze_lifecycle.py::test_plus_one_target_wakes_the_bead_with_the_preset_note`
   - `tests/test_bead/test_snooze_lifecycle.py::test_snooze_round_trips_through_every_persistence_surface`
   - `tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`

   plus `sase-j6`'s node,
   `tests/test_bead/test_plus_one_presentation.py::test_post_close_plus_one_badge_marker_search_and_json_agree`.

3. Flip the detector to a **blocking gate**: wire it into `just check-full` and CI so a
   newly introduced leak fails the build. This is what makes the class non-recurring and
   is the load-bearing deliverable of the phase — without it `sase-ct` comes back.

### What is not yet root-caused — read this before you start

The bead cluster and the plan-approval node are **not** explained by the VCS-tag leak.
Measured, so you do not repeat it:

- All of `tests/test_bead/` plus `tests/test_plan_approval_actions.py` pass serially
  (1684 passed) and pass under `-n 12 --dist=worksteal` scoped to those files, three
  runs in a row. Their poisoner lives **outside** those files.
- The `test_removed_hg_workspace_workflow.py` poisoner does **not** reproduce them:
  `poisoner -> test_post_close_plus_one_badge_marker_search_and_json_agree` passes, and
  `poisoner -> snooze cluster` passes.

Two useful signals from the selection store:

- In the run recorded `2026-08-10T19:18:31` (head `5f6d8ea64`), the two
  **proven-poisoned** VCS-tag nodes failed _in the same run as_
  `test_post_close_plus_one...`. Consistent with one worker being poisoned and taking
  unrelated victims with it — worth checking whether a single leak explains both.
- The four snooze nodes failed together as a clean set of exactly 4 at
  `2026-08-09T13:32:43`, the same one-poisoned-worker signature. Their last recorded
  failure is 2026-08-09, i.e. they are older than the `sase-iy` fixes.

If a node turns out **not** to be a global-state leak, root-cause it on its own terms
and say so explicitly. Do not force it into this diagnosis. If a node cannot be
reproduced at all after honest effort, leave it in the baseline with a note recording
exactly what you tried — **that is an acceptable outcome and is much better than
removing it on a guess.**

### Exit criteria

- Every poisoning leak in the inventory is either fixed or has a node-specific bead
  filed through `/sase_new_task` explaining why it is not fixable here.
- Every node in step 2 either has a deterministic before/after reproduction, or a
  written record of what was tried and why it stayed unreproducible.
- The leak detector runs as a blocking gate and passes.
- `just check-full` green.

## Phase `retire` — Shrink the baseline, run the exit criteria, and close the beads

Depends on: `vcs-cache`, `leak-detector`, `stale-nodes`, `fix-leaks`.

Re-read the parent epic plan before starting; open the plans sidecar with `/sase_repo`
rather than reaching for a path:

```
plans:202608/retire_sase_ct_umbrella.md
```

### Step 1 — Shrink the baseline

Remove from `tests/reproducible_flake_baseline.txt` every node with a fix commit from
`vcs-cache` or `fix-leaks`. Leave every node without one, and add a comment on each
survivor pointing at its bead. Do not touch the `# effective-after:` line. The header
says entries are "debt to remove, not suppressions to grow" — this is the first change
in the epic's history that actually removes any.

### Step 2 — Run the four exit criteria on the combined tree

Run `just install` first; workspaces go stale.

1. `just check-full` green end to end.
2. `just test-visual` green.
3. `just test-contention` on the residue node set, zero node failures:
   ```bash
   just test-contention -- tests/test_agent_group_revival_e2e.py \
     tests/ace/tui/test_commits_pane_filters.py \
     tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py \
     tests/ace/tui/test_plugins_browser_pane_sase_update_dev.py
   ```
   `just test-contention` deliberately starves the host and other agents share this
   machine — keep it scoped to those files.
4. `.venv/bin/python tools/check_test_wait_helpers` exits 0, **and**
   `just selection-health --fail-on-new-flake` passes **non-vacuously**.

Non-vacuity is a required check, not a nicety — `sase-h8` was misled by a gate that
passed because it was judging nothing. Confirm the eligible-record count is comfortably
above `MIN_GATED_FULL_RUNS` (2) and report the number you actually measured. Other
agents write to the record store concurrently, so measure immediately before you close
and report what you saw, not what this plan predicted.

**If a criterion fails, fix it or file it and report. Do not close `sase-ct` on a
criterion you did not meet.** `sase-h8.5` closed `done` with zero commits and looked
identical to a real closure from the outside.

### Step 3 — Close the beads

Close all of the following. This is an explicit deliverable of this plan, not a
follow-up:

- **`sase-j4`** — `test_agent_xprompt_highlights_warm_catalog_skills`. Close `done`
  citing the `vcs-cache` fix commit and the reproduction that now passes.
- **`sase-j5`** — stale node IDs in the flake gate. Close `done` citing `stale-nodes`.
- **`sase-j6`** — `test_post_close_plus_one_badge_marker_search_and_json_agree`. Close
  `done` if `fix-leaks` fixed it; if it stayed unreproducible, leave it open with the
  investigation recorded and say so in the `sase-ct` close note.
- **`sase-ct`** — close `done` as a retired umbrella. Use the epic plan's verbatim
  retirement text (`plans:202608/retire_sase_ct_umbrella.md`), and **replace its "fixed
  by mechanism" paragraph with this epic's actual finding**: the class was
  process-global state leaking between tests, amplified by `--dist=worksteal`, not TUI
  timing. Name the leak detector gate as the reason it will not recur. Re-check the `+1`
  and reopen counts against `sase bead show sase-ct` at close time (they were `[+60]`
  `[↺8]` at authoring) and correct them if they moved.
- **`sase-iy.5`** — close, recording the measured non-vacuity numbers from step 2, the
  four exit-criteria results, and every bead ID closed here.

Confirm `8501a19ac` (which routes retired-umbrella duplicates to new tasks in
`/sase_new_task`) is still an ancestor of the tree you close from. If that were ever
reverted, the window between closing `sase-ct` and re-landing it is exactly when the
next `+1` reopens it a ninth time.

### Step 4 — Close out the epic plan and leave the neighbors clean

- Set `status: done` in the frontmatter of `202608/retire_sase_ct_umbrella.md` (via
  `/sase_repo`).
- Do **not** close `sase-h8` or `sase-h8.10`. Note on each that `sase-ct` was retired
  here, by what criteria, and that it will not reopen — their land agents should not be
  left waiting on a bead that will never come back.

### Step 5 — Report to the owner without acting

- `sase-iu` is `READY` and `sase-iv` is a byte-identical `OPEN` duplicate of it. Both
  name `tests/test_contract_manifest.py` nodes that now pass on master. One should
  probably be closed as a duplicate of the other and both as already-fixed — the owner's
  call, not this epic's.
- Any bead filed during `fix-leaks`, by ID.

## Watch out for

- **The temptation to bump the cutoff.** If `just selection-health --fail-on-new-flake`
  is red near the end, the fix is to fix the node or file it — not to move
  `# effective-after:`. That move has already been made twice (`607b72bb0`, `83bb8a6f7`)
  and is why this is the third attempt.
- **A vacuous green gate.** A cutoff late enough to flag nothing passes and proves
  nothing. Verify the gate still flags live nodes.
- **Cache-warming noise drowning the detector.** If `leak-detector` reports thousands of
  mutations, the warming/poisoning distinction is wrong, not the suite.
- **Concurrent agents.** They share this host and write to the selection record store.
  Numbers drift between measurement and commit; re-measure right before you commit and
  report the numbers you actually saw.
