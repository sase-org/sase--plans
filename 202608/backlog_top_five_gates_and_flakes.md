---
tier: epic
status: done
title:
  Task backlog top five - clear the two red verification gates and the three
  reproducible test hazards behind them
goal: "The reproducible-flake gate has no non-epic-owned node holding it red, the serial
  ACE PNG visual lane is green on master, the last known fork-after-threads hazard in
  the test suite is gone, and an agent typing a bare `sase` can no longer be silently
  answered by a different checkout's build. Task beads sase-mv, sase-ny, sase-lk,
  sase-o5, and sase-o6 are closed on evidence.

  "
phases:
  - id: configcache
    title:
      Isolate the process-global merged-config cache so its nodes stop failing the flake
      gate
    depends_on: []
    size: large
    description: "configcache: fix the process-global merged-config cache leak behind
      tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config,
      the only non-epic-owned node still holding `just selection-health
      --fail-on-new-flake` red, and verify its sibling nodes in the same file.

      "
  - id: goldens
    title: Rebaseline the eleven stale ACE PNG goldens that fail the serial visual lane
    depends_on: []
    size: medium
    description: "goldens: confirm and rebaseline exactly the eleven ACE PNG goldens
      that fail deterministically in a serial `-n 0` visual run on clean master, without
      touching any other golden in the suite.

      "
  - id: supervise
    title:
      Deflake the three monitor-supervise pipe-EOF nodes and retire their baseline debt
    depends_on: []
    size: large
    description: "supervise: root-cause and fix the residual full-parallel-lane failures
      of the three tests/monitor/test_monitor_supervise.py pipe-EOF nodes that survived
      the 2026-08-15 BoundedLogPipe.close fix, then retire their committed baseline
      entries.

      "
  - id: forksafe
    title: Replace the last known fork-after-threads lock holders in the test suite
    depends_on: []
    size: medium
    description: "forksafe: replace the four multiprocessing fork sites in
      tests/test_sdd_git_contention.py with the in-process lock-holder seam that already
      fixed the identical hazard on tests/test_plan_approval_actions.py.

      "
  - id: saseinstall
    title:
      Stop a stale global sase build from silently answering workspace memory-drift
      checks
    depends_on: []
    size: medium
    description: "saseinstall: make it impossible for a bare `sase` resolved from a
      separate uv tool checkout to silently answer a memory-drift check about this
      workspace, which is the mechanism behind five phantom-drift reproductions across
      six agent shells.

      "
proposed_by: bbugyi200.athena.sase-ns.6.6.land--1
parent_bead: sase-ns.6.6
bead_id: sase-ns.6.6.6
create_time: 2026-09-09 19:50:03
---

- **PROMPT:**
  [prompts/202608/backlog_top_five_gates_and_flakes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/backlog_top_five_gates_and_flakes.md)
- **BEAD:**
  [sase-ns.6.6.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.6.md)

# Plan: Task backlog top five - clear the two red verification gates and the three reproducible test hazards behind them

## Background: how these five were chosen

A backlog-triage sweep on 2026-08-17 (workspace `sase_12`, clean master `cf7eeee03`,
after a from-scratch `just install`) reviewed all 25 ready task beads for the `sase`
project. Two were closed during that sweep as no longer relevant:

- `sase-o4` (stale Symvision `--epic-symbol` entries for closed bead `sase-nb`) was
  already fixed by `ec2cc1912`, which landed 46 seconds after the bead was filed.
  Verified: `grep -n epic-symbol Justfile` now names only in-progress `sase-n4` /
  `sase-n4.5` beads.
- `sase-dc` (retry-E2E PNG snapshots "fail only under full-suite contention") had its
  premise disproved by measurement — see the `goldens` phase below. Closed as superseded
  into `sase-ny`.

The five phases here are the remaining beads ranked by measured impact. Two of them are
verification gates that are red right now, for everyone.

### Gate 1: the reproducible-flake gate (phase `configcache`)

`just selection-health --fail-on-new-flake` exits 1 on current master with exactly two
exceeding nodes:

```
flake baseline gate: 2 reproducible flake(s) exceed tests/reproducible_flake_baseline.txt
  tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error
  tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config
```

The first belongs to in-progress epic `sase-n4` and is out of scope here. The second is
`sase-mv`, at `+18` corroborations. It is therefore the **only** node an ordinary agent
can do anything about, and while it is red every `just check-full` in the org fails at
this gate. Newest failure record when this plan was written: `2026-08-17T08:58:10Z`, one
hour earlier.

### Gate 2: the serial ACE PNG visual lane (phase `goldens`)

Measured directly during the sweep, serial so contention is excluded:

```
.venv/bin/pytest -q -n 0 -m visual \
  tests/ace/tui/visual/test_ace_png_snapshots_artifacts_split.py \
  tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py \
  tests/ace/tui/visual/test_ace_png_snapshots_models_panel_navigation.py \
  tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py
-> 11 failed, 12 passed in 57.56s
```

All seven `sase-ny` nodes failed **and** all four `sase-dc` nodes failed. That is what
retired `sase-dc`: its whole thesis was that those four pass in the dedicated visual
lane, and they do not.

### The three hazards (phases `supervise`, `forksafe`, `saseinstall`)

Ranked by recurrence cost rather than by severity of any single incident. `sase-lk` has
32 durable failure records and was reopened after its own fix; `sase-o5` is the last
known instance of a fork-after-threads pattern that already cost 12 records over 11 days
on a sibling node; `sase-o6` is the most likely explanation for five independent
phantom-drift reproductions across six agent shells.

## Rules that apply to every phase

These come from the backlog-triage loop this epic serves. They are not optional and they
are not per-phase judgment calls.

1. **Mark the bead in progress before doing any work**, so a parallel sweep does not
   pick it up: `sase bead update <bead-id> --status in_progress`.
2. **Never ask the user for approval directly.** If a phase concludes its bead needs an
   owner decision, leave the note and stop that part of the work:
   `sase bead note <bead-id> "TASK NEEDS APPROVAL: <what needs deciding and why>"`. Be
   lenient about this. Objective improvements — deleting dead code, fixing a leaking
   fixture, regenerating a golden that a landed commit made stale — do **not** need
   approval. Reserve the note for changes that alter user-facing behavior, weaken a
   gate's ability to catch regressions, touch the user's machine outside this repo, or
   edit `sase/memory/*.md` / `AGENTS.md` / a generated provider shim.
3. **Leave a note on the bead recording the outcome**, whether or not the work landed:
   `sase bead note <bead-id> "<what changed, and what verified it>"`. If the phase could
   not finish, the note must say what was tried, what was learned, and what the next
   agent should do — that note is the handoff.
4. **Close the bead only if the work is actually finished**:
   `sase bead close <bead-id> --note "<what was verified>"`. If it is not finished,
   leave it open with the step-3 note. Do not close a bead to make a phase look done.
5. **Close your own phase bead and nothing else**:
   `sase bead close <phase-bead-id> --note "<what was verified>"`. Never close the epic.
6. **Do not create beads.** Record discovered work as
   `sase bead note <phase-bead-id> "PROPOSED FOLLOW-UP: <one-line summary — detail>"`.
   The land agent triages those.
7. **Verification is `just install` then `just check`.** Hand `just check-full` to
   `/sase_monitor` with a `--next` action; never run it inline.

### One shared file, deliberately not a dependency

`configcache` and `supervise` both edit `tests/reproducible_flake_baseline.txt`.
`configcache` removes nothing and may add a `# fixed-at:` block; `supervise` removes its
own node-ID entries. The collision is textual, not semantic — whichever lands second
rebases. Neither blocks the other, which is why all five phases declare `depends_on: []`
and can run in parallel.

## Phase `configcache` — bead `sase-mv`

### What is actually failing

`tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`
fails under the whole-suite parallel lane and passes in isolation. The failing assertion
is line 251:

```python
now[0] += config_core._CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS + 0.01

assert load_merged_config() is first          # <-- fails: not the same object
second = _wait_for_new_merged_config(first)
```

The test advances a monkeypatched `sase.config.core.time.monotonic` past the refresh
interval and then asserts that the _next_ `load_merged_config()` still returns the old
object — i.e. that invalidation is asynchronous and single-flight, not immediate. Under
the full lane it has already been replaced by the time line 251 runs.

### Evidence

- 19 full-run failure records in `~/.sase/test-selection/gh_sase-org__sase` for the
  config-cache class since 2026-08-15, 11 of them since 2026-08-16T12:00Z, across
  distinct heads `37fe22b81`, `5184f5ab0`, `985aae20c`, `708c25452`, `78a9130f7`,
  `30c9ba23b`, `3a37168cc`, `23c953bc7`, `bbc24e472`, `fc1ad39e7`, `b6246f1cf`.
- `sase-mv`'s own `+1` history names three distinct nodes in this same file and class:
  the titled `test_owner_snapshot_reuses_parsed_overlay_until_token_changes`, the
  gate-named `test_selector_change_eventually_invalidates_merged_config`, and
  `test_load_merged_config_caches_plugin_layer`. Treat them as **one defect**.
- Green in isolation and under file-scoped contention:
  `SASE_CONTENTION_REPEAT=3 just test-contention tests/test_config_cache.py` gives 18
  passed per repeat, 0 failures across 3 repeats. So the poisoner is elsewhere in the
  suite, not inside this file.

### What has already been tried, so do not redo it

`3a22ff04f` "fix(config): isolate config cache from test-owned CONFIG_DIR" (bead
`sase-nv`, 2026-08-16T23:02:36Z) bound `current_config_token()` to the `CONFIG_DIR`
object it was computed against, made `_clear_config_caches` a yield fixture that drains
`sase-config-token-refresh`, and only calls `cache_clear` on the original functools
helpers. That retired nine sibling nodes. **One failure record postdates it**
(`b6246f1cf`, 2026-08-17T09:09:32Z), so `CONFIG_DIR` rebinding is not this node's path.

### Scope

Find what still lets another test's activity advance or invalidate this process-global
cache mid-test, and isolate it by mechanism. Candidates worth ruling in or out, in rough
order of likelihood:

1. The background refresh thread (`sase-config-token-refresh`) surviving from an earlier
   test and completing inside this test's window.
2. `patch("sase.config.core.time.monotonic", ...)` being process-global: another
   worker's test in the same process observing or racing the patched clock, or this
   test's advance being visible to a concurrently running refresh.
3. Real wall-clock elapsing past `_CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS` for a
   _different_ code path that does not go through the patched monotonic.

Use `tests/_global_state_leak_detector.py` (built by phase `sase-j7.2`) as the
instrument for finding a poisoning test. Fix the mechanism; do not add a retry, a sleep,
or a baseline entry.

### Verification

- The named node plus `test_owner_snapshot_reuses_parsed_overlay_until_token_changes`
  and `test_load_merged_config_caches_plugin_layer` all pass in isolation, under
  `SASE_CONTENTION_REPEAT=3 just test-contention tests/test_config_cache.py`, and in a
  full parallel lane.
- `just selection-health --fail-on-new-flake` names at most the `sase-n4`-owned fakey
  usage-limit node. If the fix landed but historical records still hold the gate red,
  declare a `# fixed-at: <UTC timestamp> <node id>` entry in
  `tests/reproducible_flake_baseline.txt` naming this bead and the fix commit, following
  the convention already in that file's header — that directive retires only pre-fix
  evidence for that node.
- `just install && just check` green.

### Escalation

If the root cause turns out to be owned by in-progress epic `sase-j7` ("Fix the sase-ct
flake class at its root"), record it as `sase bead note sase-j7 "DISCOVERED ISSUE: ..."`
and say so in the phase note — but still fix this node if it can be fixed without
waiting on that epic.

## Phase `goldens` — bead `sase-ny`

### The eleven nodes

Seven this bead already owned:

```
tests/ace/tui/visual/test_ace_png_snapshots_artifacts_split.py::test_artifacts_split_mode_png_snapshot[narrow-size0-artifacts_split_narrow_120x40]
tests/ace/tui/visual/test_ace_png_snapshots_artifacts_split.py::test_artifacts_split_mode_png_snapshot[even-size1-artifacts_split_even_120x40]
tests/ace/tui/visual/test_ace_png_snapshots_artifacts_split.py::test_artifacts_split_mode_png_snapshot[wide-size2-artifacts_split_wide_120x40]
tests/ace/tui/visual/test_ace_png_snapshots_artifacts_split.py::test_artifacts_split_mode_png_snapshot[narrow-size3-artifacts_split_narrow_80x24]
tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py::test_help_panel_filter_png_snapshot
tests/ace/tui/visual/test_ace_png_snapshots_models_panel_navigation.py::test_models_panel_bucket_drilled_in_png_snapshot
tests/ace/tui/visual/test_ace_png_snapshots_models_panel_navigation.py::test_models_panel_mixed_builtin_bucket_png_snapshot
```

Four inherited from `sase-dc`, closed as superseded on 2026-08-17:

```
tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_retry_countdown_png_snapshot
tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_loader_plan_family_retry_countdown_png_snapshot
tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_running_fallback_png_snapshot
tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py::test_real_fakey_completed_retry_chain_png_snapshot
```

### Why the four are the same class, measured not assumed

Material-diff ratios from the serial run, against the alpha-aware colour-distance-8
threshold:

| nodes                        | material diff                |
| ---------------------------- | ---------------------------- |
| `artifacts_split` ×4         | 9.78%, 8.63%, 8.29%, 9.24%   |
| `help_panel` filter          | 0.0448% (681 / 1,520,532 px) |
| `models_panel_navigation` ×2 | 0.0909%, 0.0911%             |
| `retry_e2e` ×2 measured      | 0.3235%, 4.5214%             |

Every one is a concentrated, material, text-or-chrome-shaped diff. None is the scattered
sub-pixel anti-aliasing that would put it in the **retired** `sase-dl` renderer-variance
class. The `help_panel` number reproduces `sase-ny`'s original measurement to the pixel.

### Scope

For each of the eleven, inspect `.pytest_cache/sase-visual/<node>/<snapshot>/` (actual,
expected, diff, source) and confirm the change is explained by a landed UI commit. Named
candidates, none owned by an active epic: `3f5378aeb` "feat(artifacts): conform pane
contract capabilities" for the four `artifacts_split` nodes; `83e2ceea6` "feat(ace):
unify monitor gear iconography across nodes and the top bar" and `3c9df1182`
"feat(ace-tui)!: unify the Artifacts keymap across Patch and its siblings" for the
help-panel and models-panel chrome. The four `retry_e2e` nodes have no attributed commit
yet — attribute them before accepting them.

Then rebaseline **only** confirmed-intentional goldens with
`--sase-update-visual-snapshots` on the specific node IDs.

### Hard constraints

- **Never run `just test-visual-update` broadly.** Closed bead `sase-lo` records that it
  rewrites every golden in the suite and silently absorbs unrelated drift.
- Any node whose diff is _not_ explained by a landed UI change must be investigated, not
  accepted. If it turns out to be scattered renderer noise, it belongs to the retired
  `sase-dl` class: leave that node red, say so in the bead note, and do not rebaseline
  it.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_artifacts.py`-style failures that
  error during setup are **not** in scope — that is `sase-my` (a stale monkeypatch
  target after the Files pane split, an `AttributeError` before any PNG comparison).

### Verification

A serial `just test-visual` (or the equivalent `-n 0 -m visual` run over
`tests/ace/tui/visual/`) shows all eleven green, and no golden outside the eleven has a
modified mtime or appears in `git status`. `just install && just check` green.

## Phase `supervise` — bead `sase-lk`

### The three nodes

```
tests/monitor/test_monitor_supervise.py::test_run_supervisor_escalates_term_ignoring_chatty_child
tests/monitor/test_monitor_supervise.py::test_run_supervisor_times_out_after_partial_line
tests/monitor/test_monitor_supervise.py::test_run_supervisor_completes_when_grandchild_holds_stdout
```

Added by epic `sase-ku` phase `sase-ku.2` (`afa8178ce`). They pass in isolation and in
the monitor-only suite, and fail intermittently in other agents' full parallel lanes.

### What has already been tried

A fix landed on 2026-08-15 under this bead: `BoundedLogPipe.close()` now joins for
`close_drain_seconds` plus a 0.1s scheduling allowance instead of a fixed 5s, and
ingests immediately-readable bytes before honoring the close deadline. It was verified
with 15 serial + 10 parallel repeats of all three nodes.

**It was not sufficient.** The durable store holds 32 failure records for these nodes, 7
of them after 2026-08-15 and 3 after 2026-08-16T12:00Z, newest `20260817T011249Z`. A
`+1` on 2026-08-17T01:22:56Z from an unrelated session reports
`test_run_supervisor_times_out_after_partial_line` failing on the 13-worker full lane on
a tree that already contained the fix, passing on immediate isolated rerun.

So this phase starts from "the drain-deadline theory is incomplete", not from scratch.
Read the existing fix first and work out what it does not cover — most plausibly that
the pipe genuinely has not EOF'd because a descendant still holds the write end, and the
close path cannot distinguish that from a slow drain.

### Baseline debt this phase owns

`tests/reproducible_flake_baseline.txt` currently carries an active node-ID entry for
`test_run_supervisor_escalates_term_ignoring_chatty_child` (and, separately,
`test_run_supervisor_kills_the_whole_process_group_on_timeout`, which this bead does
**not** name — leave that one alone unless the same fix demonstrably covers it, and say
so explicitly if it does). Removing this bead's own entry is part of finishing it;
baseline entries are debt to remove, not suppressions to grow.

### Verification

- All three nodes pass under
  `just test-contention tests/monitor/test_monitor_supervise.py` with
  `SASE_CONTENTION_REPEAT` at 3 or more, and in a full parallel lane.
- The `sase-lk` entry is removed from `tests/reproducible_flake_baseline.txt`, or a
  `# fixed-at:` directive naming this bead and the fix commit retires the pre-fix
  evidence — whichever the file's header convention calls for.
- `just install && just check` green.

### If it cannot be fixed

This bead has already defeated one fix. If a bounded effort does not produce a
mechanism-level fix, that is a legitimate outcome: leave the bead **open**, record what
was ruled out and what evidence would settle it, and do not close it. Do not widen the
baseline to make the gate quiet.

## Phase `forksafe` — bead `sase-o5`

### The hazard

Forking from a multi-threaded process inherits only the calling thread, so any lock or
buffer another thread held at fork time is inherited in an indeterminate state. Under
xdist every worker carries a live execnet receiver thread, so **every**
`multiprocessing.get_context("fork")` in a test is a fork-after-threads hazard.

This is proven, not theoretical: it was the root cause of
`tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`
(bead `sase-nz`), which accumulated 12 full-run failure records across six workspaces
and twelve heads between 2026-08-06 and 2026-08-17 before `b6246f1cf` fixed it.

### The remaining sites

`tests/test_sdd_git_contention.py` carries the identical pattern — verified on current
master:

```
5:   import multiprocessing
53:  def _acquire_epic_plan_launch_lock(anchor, acquired)      # fork target
62:  def _hold_epic_plan_launch_lock(...)                       # fork target
322: context = multiprocessing.get_context("fork")
346: context = multiprocessing.get_context("fork")
420: context = multiprocessing.get_context("fork")
492: context = multiprocessing.get_context("fork")
```

### The model fix

`b6246f1cf` replaced the fork with an in-process holder,
`_foreign_epic_launch_lock_holder` in `tests/test_plan_approval_actions.py:35`, which
takes the same `flock` on the lock path, writes the same holder-identity JSON that
production `epic_plan_launch_lock` writes, and self-checks that a second `open()` on the
path is refused with `BlockingIOError` before yielding. Read that commit and reuse the
seam rather than inventing a second one.

### Scope

Replace all four fork sites. Preserve exactly what each test is asserting — these tests
exist to prove real contention behavior, so the replacement must still hold a genuine
lock that the code under test genuinely cannot acquire. A holder that only _pretends_ to
contend would silently retire the tests' value; if the in-process seam cannot express
what a given site needs, say so in the bead note rather than weakening the assertion.

### Verification

- No `multiprocessing.get_context("fork")` remains in `tests/test_sdd_git_contention.py`
  (and the `import multiprocessing` goes too if nothing else uses it).
- The file passes serially, under `just test-contention`, and in a full parallel lane.
- `just install && just check` green.

### Worth checking while you are here

Whether any other test file still forks. If so, record it as a `PROPOSED FOLLOW-UP:`
note on the phase bead rather than widening this phase.

## Phase `saseinstall` — bead `sase-o6`

### The mechanism, verified in this environment

`/home/bryan/.local/bin/sase` is a uv tool install. Its shebang is
`/home/bryan/.local/share/uv/tools/sase/bin/python3`, editable-installed against a
**separate checkout** at `~/projects/github/sase-org/sase`, versioned independently of
any ephemeral `sase_<N>` workspace. Confirmed again during this sweep.

`just check`'s validate recipe pins `{{venv_bin}}/sase`, so it always matches the
workspace tree and is trustworthy. But an agent that types a bare
`sase init memory --check` or `sase memory init --check` — exactly what the mandatory
memory-init workflow invites — runs the _global_ build, comparing this workspace's
committed files against a different checkout's templates.

### Why this is probably the answer to a long-running puzzle

Task `sase-n0` collected five independent reproductions across six agent shells between
2026-08-09 and 2026-08-17 of "`sase validate` says green but `sase init memory --check`
says drift"; `sase-i7` was closed as superseded into it. Epic phase `sase-ns.6.6.2` then
established structurally that **no generator divergence exists in source**:
`sase validate`'s memory step shells out to
`<sys.executable> -m sase init memory --check`, so both entry points funnel through one
`plan_init_memory -> _memory_root_plans -> plan_memory_root -> memory_root_context -> render_expected_memory_files()`
chain and cannot disagree — unless they are two different _installs_ of that generator.
The reported disagreements were heading-style and memory-README-listing diffs, which is
template-version skew, exactly this shape.

### Scope — pick one, or both

- **(a) The durable fix.** Keep the global tool install synced with master, or have the
  CLI detect that its own build predates the workspace it is invoked in and say so. A
  staleness warning printed by the CLI is fully in-repo and needs no approval. A chezmoi
  apply hook or a post-merge reinstall reaches the user's machine and the `chezmoi`
  repo.
- **(b) The cheap immediate fix.** Make the memory-init workflow name the venv-pinned
  invocation explicitly wherever it appears in agent instructions.

They are not exclusive. Prefer landing the in-repo half of (a) — a staleness signal from
the CLI itself is the part that protects an agent who types the wrong thing.

### Approval boundaries — read before editing anything

- The `chezmoi` repo must be opened through `/sase_repo`, never by path. Do not modify
  it in this phase; if (a) needs a chezmoi change, leave
  `sase bead note sase-o6 "TASK NEEDS APPROVAL: ..."` describing the exact change.
- Editing `sase/memory/*.md`, `AGENTS.md`, or a generated provider shim (`CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) to carry option (b)'s wording **requires
  explicit user permission** per `CLAUDE.md`, and a plan file does not count as that
  permission. Leave a `TASK NEEDS APPROVAL` note with the proposed wording instead of
  editing.
- Reinstalling or removing the user's global `sase` is a change to their machine. Leave
  a `TASK NEEDS APPROVAL` note; do not do it.

Because most of the durable options land outside what this phase may touch unilaterally,
a `TASK NEEDS APPROVAL` note is an expected and successful outcome here — not a failure.
Land whatever in-repo part is objectively safe, note the rest.

### Verification

- Whatever lands is covered by a test: a staleness warning gets a unit test that pins
  the message and the condition that triggers it.
- `just install && just check` green.
- The bead note names precisely which of (a) and (b) landed, which was deferred to a
  `TASK NEEDS APPROVAL` note, and why.

## Landing

The land agent verifies the epic, integrates it, and closes it as usual. Two extra
obligations belong to this epic specifically:

1. **Report the approval notes.** Collect every `TASK NEEDS APPROVAL` note left by this
   epic's phases and carry them into the land agent's final response as a consolidated
   report with suggested next actions, one entry per bead.
2. **Continue the backlog-triage loop.** After the epic closes, ready task beads will
   still remain for the `sase` project. Start a pseudo monitor with the `sleep 1`
   command via `/sase_monitor`, whose `--next` action instructs the next agent to follow
   this exact prompt, with these exact steps:

   > (1) Review all of my current open (not in-progress) sase task beads for the "sase"
   > sase project. (2) Close any task beads that are no longer relevant with a good
   > reason. (3) Select the 5 task beads that would have the most impact if worked to
   > completion. (4) Use your /sase_plan skill to fix the issues / make the improvements
   > that correspond with these 5 task beads. Make sure the plan file you propose tells
   > the agent(s) to: (a) if you think any of these 5 beads need approval from the user
   > before working (be lenient here and do not ask for approval for objective
   > improvements), do not ask directly, but instead leave a `TASK NEEDS APPROVAL` note
   > on the bead; (b) mark the bead(s) you intend to work as in-progress by changing
   > their status with the `sase bead update` command; (c) leave a brief note on the
   > task bead(s) explaining the work that was done to fix the reported issue / make the
   > requested improvement or, if the agent was unable to complete the work, justifying
   > why they were unable to do so; (d) close each of the 5 task beads that it was able
   > to finish; (e) if there are more task beads associated with the "sase" project, the
   > agent should then start a pseudo monitor using the `sleep 1` command with a next
   > action that instructs the next agent to follow this exact prompt (with these exact
   > same steps); (f) if there are no more task beads to work, the agent should move on
   > to the next numbered step in this prompt. (5) Review all `TASK NEEDS APPROVAL`
   > notes left by prior agent shells and consolidate them into a single report for the
   > user with suggested next actions. (6) Terminate.

   The `--next` action must also carry a short status summary of what this epic closed,
   what it left open, and the current state of both verification gates, so the next
   sweep does not re-derive it.

## Exit criteria

- `just selection-health --fail-on-new-flake` reports no exceeding node other than the
  `sase-n4`-owned `tests/fakey/test_usage_limit_e2e.py` node.
- A serial visual run over `tests/ace/tui/visual/` shows the eleven named goldens green,
  with no other golden modified.
- `tests/test_sdd_git_contention.py` contains no `multiprocessing` fork.
- `tests/reproducible_flake_baseline.txt` is no larger than it is today, and smaller by
  the `sase-lk` entry if `supervise` succeeded.
- Each of `sase-mv`, `sase-ny`, `sase-lk`, `sase-o5`, `sase-o6` is either closed with a
  verification note, or left open with a handoff note explaining what blocked it.
- `just check-full`, run through `/sase_monitor`, is no worse than it is today. The
  pre-existing suite-cost budget failure tracked by `sase-j0` is expected and is not
  this epic's to fix.

## Out of scope

- `tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error`
  — the other node holding the flake gate red. It belongs to in-progress epic `sase-n4`.
- The suite-cost budget gate (`sase-j0`) and the process-global leak class at its root
  (`sase-j7`). Both are in-progress epics with their own land agents.
- `sase-my` (Artifacts Files PNG `AttributeError` during setup) and the three Artifacts
  Beads visual nodes recorded as a `DISCOVERED ISSUE` on epic `sase-nb`.
- The remaining 18 ready task beads. The next iteration of the triage loop takes them.
- Re-triaging `sase-dc` or `sase-o4`, both closed during the sweep that produced this
  plan.
