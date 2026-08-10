---
tier: epic
title: Close the three live task beads and retire the sase-ct umbrella permanently
goal: 'The three non-snoozed task beads (sase-ct, sase-ii, sase-iq) are closed with
  their underlying issues actually resolved. sase-ii and sase-iq are verified fixed
  on master and closed. The sase-ct class is attacked at the three mechanisms that
  produce it today — a deterministic prompt-catalog convergence hang that makes the
  PNG lane red in isolation, a wait-idiom gate blind spot that lets attempt-bounded
  pause loops back into ACE tests, and the residual contention-sensitive nodes — and
  then sase-ct is retired as an umbrella: it closes with a reason that forbids future
  +1 corroboration and directs the next reporter to file a node-specific task bead
  that references sase-ct as RELATED, and /sase_new_task is changed so that instruction
  is actually reachable at the moment an agent would otherwise +1.

  '
phases:
- id: closeouts
  title: Verify and close sase-ii and sase-iq
  depends_on: []
  size: small
  description: 'closeouts: confirm on current master that the sase-ii mtime-cache
    node and both sase-iq run_pytest cost-mode nodes pass, establish that each reopening
    +1 predates the landed fix, then close both beads with evidence notes.'
- id: catalog
  title: Fix the deterministic prompt-catalog convergence hang in the PNG lane
  depends_on: []
  size: medium
  description: 'catalog: make the ACE startup prompt-catalog rebuild worker stop holding
    wait_for_visual_idle open for its full 30s deadline. Reproduced deterministically
    in isolation on clean master; fix it centrally in the visual fixtures rather than
    per file, and prove the PNG lane green.'
- id: waitgate
  title: Widen the wait-idiom gate past its receiver and name blind spots
  depends_on: []
  size: medium
  description: 'waitgate: tools/check_test_wait_helpers only recognizes bounded-wait
    loops whose receiver is literally named pilot and private helpers named _wait_until,
    so page.pause() loops and _wait_for helpers pass it. Widen both axes and migrate
    every call site the widened gate reports onto the shared waiters.'
- id: residue
  title: Fix the remaining contention-sensitive sase-ct nodes by mechanism
  depends_on:
  - waitgate
  size: medium
  description: 'residue: take the non-visual nodes still recurring on sase-ct that
    waitgate does not already fix — agent-group revival, commits-pane filters, the
    vcs_tag pair, plugins-browser updates — and fix each by mechanism, using just
    test-contention as the falsifiable before/after harness.'
- id: retire
  title: Retire the umbrella, close sase-ct, and make the no-+1 instruction reachable
  depends_on:
  - closeouts
  - catalog
  - waitgate
  - residue
  size: medium
  description: 'retire: change /sase_new_task so a retired umbrella routes the next
    reporter to a node-specific task bead with a RELATED note instead of a +1, run
    the exit criteria on the combined tree, and close sase-ct with the verbatim reason
    this plan specifies.'
proposed_by: bbugyi200.athena.xb
create_time: 2026-08-10 11:01:13
status: wip
bead_id: sase-iy
---

- **PROMPT:** [prompts/202608/retire_sase_ct_umbrella.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/retire_sase_ct_umbrella.md)
- **BEAD:** [sase-iy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-iy/README.md)

# Plan: Close the three live task beads and retire the `sase-ct` umbrella

## Why this plan exists

Three task beads are open and not snoozed: `sase-ct`, `sase-ii`, and `sase-iq`. Two of
them are already fixed on master and only need an honest closure. The third, `sase-ct`,
has been closed eight times and reopened eight times, carries 55 `+1`s, and has consumed
two full epics (`sase-h8`, and its continuation `sase-h8.10`, both still `in_progress`)
without ever staying closed.

The reason it never stays closed is structural, and naming it is the point of this plan.
`sase-ct` is an **umbrella** bead for "an ACE TUI test failed under the full parallel
run and passed in isolation." That description matches every future ACE timing failure,
forever. `/sase_new_task` correctly tells a reporter to `+1` a semantic duplicate rather
than file a new task, and a `+1` on a closed task atomically reopens it. So the bead is
reopened by the very policy that is supposed to keep the tracker clean — three times in
the last two hours alone. No amount of fixing individual nodes can terminate that loop,
which is why eight closures have not.

So this plan does two things that previous attempts did not do together: it fixes the
mechanisms that are producing failures _right now_, and it retires the umbrella as a
tracking pattern so the next reporter files a node-specific bead instead of reopening
this one.

## What was verified while authoring this plan

All of the following was measured at `origin/master` `c8d5b3d0a` in a clean workspace
after `just install`. Later phases should re-measure rather than trust these numbers,
but they are the evidence the phase scoping rests on.

**`sase-ii` is fixed.** Commit `884951057`
(`test(ace): wait for store reload instead of racing pilot.pause()`) is on master.
`tests/ace/tui/test_tasks_pane_store.py::test_following_a_live_store_row_bypasses_the_mtime_cache`
passes both params. The `+1` that reopened the bead (`sase-il.5`, `14:29:58Z`) reports a
`just check-full` run whose tree predates that commit.

**`sase-iq` is fixed.** Commit `1417de7db` (`test: update cost mode recorder contracts`)
is on master. The node the bead names,
`test_main_cost_mode_arms_only_the_cost_recorder`, no longer exists — it was renamed to
`test_main_cost_mode_arms_cost_and_health_recorders` and its assertion inverted to match
the deliberate `HEALTH_RECORDING_MODES` behavior `354d8c19f` introduced. The companion
node `test_main_ace_page_group_isolation_uses_manifest_without_recorders` now clears the
inherited recorder env. Both pass. The `+1` that reopened the bead (`x2`, `14:27:11Z`)
landed ten seconds after the fix commit was authored, from a tree that did not contain
it.

**The PNG lane is deterministically red on clean master, in isolation.** This is the
single most valuable finding here, because it is not a flake at all:

```
just test-visual tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py
  -> 14 failed, 7 passed in 129.69s

just test-visual '...test_ace_png_snapshots_prompt_highlighting.py::test_prompt_artifact_ref_highlight_png_snapshot'
  -> 1 failed in 36.06s
  AssertionError: Timed out waiting for ACE visual render convergence after 30.00s;
    stable_frames=0/5; frame_digests=[]; pending_debouncers=[];
    pending_workers=['prompt-catalog:0']; pending_one_shot_timers=[];
    pending_animations=[]
```

`frame_digests=[]` means `wait_for_visual_idle` never sampled a single frame in 30
seconds: `_pending_visual_work` saw the `prompt-catalog:0` worker running for the entire
deadline. One node, alone, on a quiet host. That is a hang, not contention.

This is what agents have been reporting for two days as "the broad ACE visual flake
class" (`sase-il.5`: 96 failed / 480 passed; `x1`: 74 failed / 502 passed; `sase-ik.3`:
18 failed / 3 passed on this same file), and it is what makes the whole visual lane look
non-deterministic — a deterministic 30s stall per affected node, on a lane whose runs
are long enough that nobody has bisected it to one node.

The asymmetry that gives away the fix:
`tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py:33` defines
`_patch_prompt_catalog_rebuild`, which stubs out
`AceApp._schedule_prompt_catalog_rebuild` with the docstring "Disable startup
prompt-catalog work; these tests inject rows directly."
`test_ace_png_snapshots_prompt_highlighting.py` has no such patch. The workaround
exists, per file, applied inconsistently, in exactly the files that pass.

**The wait-idiom gate has a blind spot, and it is hiding the most-reported live node.**
`tests/ace/tui/widgets/test_prompt_glossary_navigation.py:25-35` defines:

```python
async def _wait_for(page, predicate, *, attempts: int = 20) -> None:
    for _ in range(attempts):
        if predicate():
            return
        await page.pause()
    assert predicate()
```

That is an attempt-bounded pause loop — precisely the idiom
`tools/check_test_wait_helpers` was built to retire, and precisely the idiom whose
removal fixed `sase-ii` (`884951057`) and the family-relaunch node (`771f7d935`). The
gate misses it on two independent axes:

- `_is_pilot_pause_await` (`tools/check_test_wait_helpers`, the
  `_inline_pilot_pause_waits` helper) requires `node.value.func.value.id == "pilot"`.
  This loop awaits `page.pause()`, so the receiver name does not match.
- `FORBIDDEN_FUNCTION = "_wait_until"`. This helper is named `_wait_for`.

`.venv/bin/python tools/check_test_wait_helpers` exits 0 today. The gate's own fixture
in `tests/test_check_test_wait_helpers_tool.py:67-72` asserts this shape _is_ rejected —
with `pilot.pause()`. So the gate is not wrong about the rule, only about the spelling.

`test_k_on_glossary_term_pushes_glossary_preview_card` — the node in this file — is the
single most frequently reported live `sase-ct` instance today, named in the `+1`s from
`sase-ik.land`, `sase-in`, `sase-ij.land`, `x0`, `wz`, `x2`, and `x3`. Epic `sase-h8.10`
chartered a `gate-gaps` phase for exactly this widening; that phase never landed, and
`sase-h8.10` is still `in_progress`.

**The remaining nodes are a different mechanism.**
`tests/test_agent_group_revival_e2e.py` already uses the shared `wait_for` helper
correctly, so `waitgate` will not fix it. Its two nodes fail under the full lane and
pass serially, and `sase-ij.land` recorded a 5.64s teardown on one of them. Note that
`wait_for` and every `AcePage.expect_*` waiter defaults to `timeout: float = 5.0`
(`src/sase/ace/testing/wait.py:73`, `src/sase/ace/testing/ace_page.py:307`), with no
scaling for worker oversubscription — under the 26-workers-on-2-CPUs shape that
`just test-contention` pins, a correct wait can still expire. Whether that fixed default
is the mechanism is for the `residue` phase to falsify, not to assume.

**Reproduction caveat, stated plainly.** A focused 4-worker run of
`test_prompt_glossary_navigation.py` plus `test_agent_group_revival_e2e.py` passed (11
passed in 12.32s) on an otherwise-busy host. Focused low-worker runs are not evidence
about this class. `just test-contention` and `just test-visual-contention` exist
precisely because of that, and their Justfile comments carry the historical baselines.
Use them.

## What is deliberately not in scope

- **`sase-gk` and `sase-gs`** are snoozed task beads. Leave them snoozed.
- **`sase-iu` and `sase-iv`** are `open` drafts, filed one minute apart by the same
  agent (`sase-il.5`), with byte-identical descriptions about a stale contract manifest.
  Both `tests/test_contract_manifest.py` nodes they name pass on current master (3
  passed in 38.95s), so both are already resolved upstream. They are duplicates of each
  other and were created after this work was scoped. Do not fold them in; the `retire`
  phase reports them to the owner instead.
- **`sase-e2`** (bead-lock contention) is tracked separately and has always been out of
  this umbrella's scope. Keep it that way.
- **The Rust `+1` reopen semantics.** `mutation.project.plus_one()` lives in
  `sase-core`, and changing whether a `+1` reopens a closed task is a core wire change
  affecting every frontend. This plan achieves the same outcome in this repo by changing
  where agents are told to look before they `+1`. If that proves insufficient, the core
  change is a follow-up, not a phase.

---

## Phase `closeouts`: Verify and close `sase-ii` and `sase-iq`

**Scope.** Close two beads whose fixes already landed. No production code changes are
expected. If verification fails, do not close — report what failed.

**Do this.**

1. `just install`, then confirm at current `origin/master`:

   ```bash
   .venv/bin/python -m pytest -q -p no:randomly \
     'tests/ace/tui/test_tasks_pane_store.py::test_following_a_live_store_row_bypasses_the_mtime_cache' \
     'tests/test_run_pytest_main.py::test_main_cost_mode_arms_cost_and_health_recorders' \
     'tests/test_run_pytest_main.py::test_main_ace_page_group_isolation_uses_manifest_without_recorders'
   ```

2. Confirm `884951057` and `1417de7db` are ancestors of `origin/master`
   (`git merge-base --is-ancestor`), and read each bead's reopening `+1` to confirm it
   predates the corresponding fix. State that timeline in the close note — it is the
   whole justification for closing over a `+1` that is only minutes old.

3. Close both:

   ```bash
   sase bead close sase-ii --reason "<what you ran, what passed, and why the reopening +1 predates 884951057>"
   sase bead close sase-iq --reason "<what you ran, what passed, and why the reopening +1 predates 1417de7db>"
   ```

**Watch out for.** `sase-iq`'s bead text names a node that no longer exists. Do not
report "cannot reproduce" from a `pytest` run that errors with `not found:` — that is
the fix, not a broken repro. Name the rename
(`test_main_cost_mode_arms_only_the_cost_recorder` →
`test_main_cost_mode_arms_cost_and_health_recorders`) explicitly so the next reader does
not re-litigate it.

**Done when.** Both beads are `closed` with resolution `done` and notes a reader can
check against the tree.

---

## Phase `catalog`: Fix the prompt-catalog convergence hang

**Scope.** `wait_for_visual_idle` must stop being held open by the startup
prompt-catalog worker. This is the phase that turns `just test-visual` from
apparently-flaky into green.

**Start from the reproduction.** It is deterministic and takes 36 seconds:

```bash
just test-visual 'tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py::test_prompt_artifact_ref_highlight_png_snapshot'
```

**Diagnose before fixing.** The worker is scheduled by
`_schedule_prompt_catalog_rebuild`
(`src/sase/ace/tui/actions/_startup_prompt_catalog.py:299-305`) and runs
`_run_prompt_catalog_rebuild`, which does
`await asyncio.to_thread(build_prompt_catalog_snapshot, ...)`. Establish which of these
it is, and say which in the phase note:

- the `to_thread` call genuinely takes longer than 30s under the visual fixture (real
  filesystem scanning of project/xprompt sources that the fixture does not stub), or
- it never completes at all (blocking on something the fixture leaves unset), or
- it completes but `_prompt_catalog_rebuild_in_flight` / the pending-rebuild coalescing
  re-arms a fresh generation so `page.app.workers` never reports idle.

Instrument rather than guess: `_pending_visual_work`
(`tests/ace/tui/visual/_ace_png_snapshot_waits.py:153-201`) already reports the running
worker name, so logging the worker's start/finish and the generation counter
distinguishes all three in one run.

**Fix centrally, not per file.** The existing per-file `_patch_prompt_catalog_rebuild`
in `test_ace_png_snapshots_model_completion.py:33` is the workaround to generalize, not
the pattern to copy into thirteen more files. Preferred shapes, in order:

1. If visual snapshots should never depend on a real catalog build, make the shared
   visual fixture (`tests/ace/tui/visual/_ace_png_snapshot_helpers.py`, alongside
   `patch_startup_loaders`) disable or stub the startup rebuild for the whole lane, and
   delete the now-redundant per-file patch.
2. If some snapshots legitimately need a built catalog, make the build bounded and
   deterministic under the fixture rather than letting it run unbounded, and have
   `wait_for_visual_idle` fail fast with the worker's own diagnostic instead of a bare
   30s convergence timeout.

Either way the convergence timeout message should name _why_ the worker is still
running, so the next occurrence is a one-line diagnosis instead of a two-day
investigation.

**Exit criterion.** All of:

- `just test-visual` green on the full PNG lane. Record the counts and the duration.
- `just test-visual tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`
  green specifically (it is 14 failed / 7 passed today).
- `just test-visual-contention` run once, with the tally recorded. Its Justfile comment
  carries the prior baselines (`sase-e9.3`, 2026-08-02: 405 passed, 1 skipped in
  605.72s); compare against them and say plainly whether you matched, beat, or regressed
  it.
- `just check` green.

**Watch out for.** Do not "fix" this by regenerating goldens or by widening the PNG diff
tolerance. The failures are convergence timeouts with `frame_digests=[]` — no frame was
ever compared, so no golden is wrong. Any diff to `tests/ace/tui/visual/snapshots/png/`
in this phase is a signal you fixed the wrong thing. Do not raise the 30s
`wait_for_visual_idle` timeout either; a longer deadline turns a 2-minute red lane into
a 10-minute red lane.

---

## Phase `waitgate`: Widen the wait-idiom gate and migrate its call sites

**Scope.** `tools/check_test_wait_helpers`, plus every test it newly reports. This is
the phase `sase-h8.10` chartered as `gate-gaps` and never landed.

**Widen two axes.**

1. **Receiver.** `_is_pilot_pause_await` requires the awaited call's receiver to be a
   `Name` node with `id == "pilot"`. Every `AcePage`/`PromptPage` test spells the same
   thing `page.pause()`, and those are invisible. Match on the awaited attribute
   (`.pause()`) inside a bounded `for ... in range(...)` loop with a conditional break
   or a trailing bare `assert`, regardless of receiver name.
2. **Helper name.** `FORBIDDEN_FUNCTION = "_wait_until"` is a single literal. The
   private helper that is live today is named `_wait_for`. Reject the shape — a private
   async def whose body is a bounded pause loop over a predicate — rather than one
   spelling of the name.

Extend `tests/test_check_test_wait_helpers_tool.py` with a fixture per new axis: a
`page.pause()` loop, and a `_wait_for`-named private helper. Its existing fixtures
(lines 67-72, 88-92) show the shape to follow, including the deliberate negative case
that must keep passing (a `range(4)` loop that presses a button and collects labels is
not a wait).

**Then migrate every call site the widened gate reports.** Start with the one that
matters: `tests/ace/tui/widgets/test_prompt_glossary_navigation.py::_wait_for` →
`sase.ace.testing.wait_for`, which is already imported across the ACE suite (see
`tests/test_agent_group_revival_e2e.py:15` for the idiom). Then work the rest of the
gate's report. Known additional candidates, all to be re-derived from the gate rather
than trusted from this list: `tests/_models_panel_helpers.py:170`,
`tests/test_llm_override_indicator.py:273`,
`tests/ace/tui/test_startup_stopwatch_live_update.py:257`,
`tests/ace/tui/modals/test_commit_view_modal.py:492`.

**Exit criterion.**

- `.venv/bin/python tools/check_test_wait_helpers` exits 0 with no remaining
  suppressions or pragmas added to silence it. If a call site genuinely is not a wait,
  the gate must be taught to distinguish it, not pragma'd past.
- `tests/ace/tui/widgets/test_prompt_glossary_navigation.py` passes serially and under
  `just test-contention -- tests/ace/tui/widgets/test_prompt_glossary_navigation.py`
  with a zero-failure tally.
- `just check` green.

**Watch out for.** Migrating a wait can change what a test actually asserts — an
attempt-bounded loop that fell through to `assert predicate()` was, in effect, asserting
"eventually or now." Waiting on the real observable end state is the point; if a
migrated test then fails deterministically, you have found a real bug, and it belongs on
its own task bead via `/sase_new_task`, not silently re-loosened.

---

## Phase `residue`: Fix the remaining contention-sensitive nodes

**Scope.** The non-visual nodes still recurring on `sase-ct` after `waitgate` lands.
Derive the live list from the bead rather than from this plan — `sase bead show sase-ct`
and read the `+1`s from the last day. As of authoring, the set is:

- `tests/test_agent_group_revival_e2e.py::test_mark_save_preview_and_revive_saved_agent_group`
- `tests/test_agent_group_revival_e2e.py::test_saved_group_revive_restores_deleted_artifacts_and_tribe_real_loader`
- `tests/ace/tui/test_commits_pane_filters.py::test_commits_negative_repo_reconciles_before_collection_and_persists`
- `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name`
- `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_directory_key_spelling_also_resolves`
- `tests/ace/tui/test_plugins_browser_pane_sase_update_dev.py::test_updates_pane_sase_dev_update_shows_all_commit_groups`

Re-check each against master first: `771f7d935` already fixed
`test_family_member_relaunch.py::test_completed_family_member_relaunch_dismisses_only_selected_child`,
and reports naming it after `2026-08-10T14:14Z` may be from trees that predate it. Drop
anything that is already fixed and say so.

**Measure before you fix.** One pass is not evidence about a class whose base rate is
under one node per run. Establish a red-rate baseline first:

```bash
just test-contention -- tests/test_agent_group_revival_e2e.py \
  tests/ace/tui/test_commits_pane_filters.py \
  tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py \
  tests/ace/tui/test_plugins_browser_pane_sase_update_dev.py
```

Record the per-node tally. Fix by mechanism. Re-run the identical invocation. Report
before/after tallies, not a single green run.

**A hypothesis worth falsifying first, because it would explain several at once.**
`wait_for` and every `AcePage.expect_*` waiter hardcode `timeout: float = 5.0` with no
awareness of worker oversubscription. `just test-contention` pins 26 workers onto 2 CPUs
— 13x — so a wait that is _correct_ can still expire on a starved event loop.
`tests/test_agent_group_revival_e2e.py` already uses the shared waiters properly and
still fails only under the full lane, which fits. If the tally confirms it, the fix is a
contention-aware timeout scale derived from the worker count the runner already knows
(`SASE_PYTEST_WORKERS`), applied once in `src/sase/ace/testing/wait.py` and
`ace_page.py`, not a hand-tuned per-test timeout. If the tally refutes it, say so and
diagnose per node — do not raise timeouts as a blanket remedy, which converts a fast
failure into a slow one.

**Note the two `test_vcs_tag_*` nodes are already in
`tests/reproducible_flake_baseline.txt`.** That file's header says entries are "debt to
remove, not suppressions to grow." If you fix them, remove them from the baseline in the
same commit. If you cannot, leave them and say why; do not add new entries.

**Exit criterion.**

- `just test-contention` on the node set above, at the default pinning and worker count,
  with a **zero-failure** tally across its repeats.
- No new entries in `tests/reproducible_flake_baseline.txt`.
- `just check-full` green.

**Watch out for.** `just test-contention` deliberately starves the host and other agents
run on this machine. Scope it to the files under investigation — a full-suite repeat is
far too slow to iterate against, and the Justfile comment says so. Also: this is the
phase where the two prior epics died. If the tally does not reach zero, do **not** let
`retire` close `sase-ct` on a hopeful reading — report the tally and hand the remainder
to `retire` as named, individually-filed task beads.

---

## Phase `retire`: Retire the umbrella and close `sase-ct`

**Scope.** Make the "do not `+1` this bead" instruction reachable, run the exit criteria
on the combined tree, close `sase-ct`, and report.

### 1. Make the instruction reachable before closing anything

The close reason alone is advisory: an agent that hits an ACE flake runs
`/sase_new_task`, whose step 4 says a semantic duplicate must be corroborated with
`sase bead +1` and that a task must **not** be created. It will find `sase-ct`, `+1` it,
and reopen it — which is exactly how the last eight closures were undone. Fixing the
skill is what makes this closure the last one.

Edit the skill source `src/sase/xprompts/skills/sase_new_task.md` (never the deployed
chezmoi `SKILL.md`) so that step 4's duplicate branch first checks whether the candidate
bead is a **retired umbrella** — a closed task whose close reason declares it retired
and forbids `+1` — and if so routes the reporter to step 7 instead: create a
node-specific task bead naming the failing node ID, and attach
`sase bead note <new-id> "RELATED: sase-ct — <how it bears on this task>"`. That
`RELATED:` convention already exists in step 7 of the skill; this reuses it rather than
inventing a mechanism.

Follow the generated-skill workflow exactly: iterate with `sase skill init --diff` or
`--dry-run`, commit the template change and land it on the canonical branch, and only
then run `sase skill init --force` from that clean merged tree. A deploy from a dirty or
unmerged tree reverts whatever another agent deployed.

Add a test that pins the new routing text, next to the existing skill-content tests, so
a later edit cannot silently drop it.

### 2. Run the exit criteria on the combined tree

All four, on the integrated tree, before touching the bead:

1. `just check-full` green.
2. `just test-visual` green.
3. `just test-contention` on the `residue` node set, zero failures in the tally.
4. `.venv/bin/python tools/check_test_wait_helpers` exits 0, and
   `just selection-health --fail-on-new-flake` passes against
   `tests/reproducible_flake_baseline.txt` with no ACE node newly added. Confirm the
   gate is judging real post-baseline records rather than passing vacuously — `sase-h8`
   was misled by exactly that — and if it is vacuous, say so plainly in the close note
   instead of counting it as met.

If a criterion cannot be run, say which and why. If a criterion fails, fix it or file it
and report; do not close `sase-ct` on a criterion you did not meet. `sase-h8.5` closed
`done` with zero commits and looked identical to a real closure from the outside.

### 3. File the true residue as node-specific beads

Anything still failing goes through `/sase_new_task` as its own bead, named for its node
ID, each carrying `RELATED: sase-ct`. That is the pattern this phase is establishing —
demonstrate it on the first real case rather than describing it.

Also report to the owner, without acting on them: `sase-iu` and `sase-iv` are
byte-identical duplicate `open` drafts whose named `tests/test_contract_manifest.py`
nodes already pass on master, so one should be closed as a duplicate of the other and
both likely closed as already-fixed. They are out of this epic's scope and are the
owner's call.

### 4. Close `sase-ct`

Use resolution `done`. The `--reason` must carry, in this order: that the tracked
instances are fixed and by which phase; that the bead is **retired as an umbrella**; and
the instruction the owner asked for, unambiguously. Use this text, adjusted only to
insert the real phase/commit references and measured numbers:

> RETIRED UMBRELLA — DO NOT `+1` THIS BEAD.
>
> `sase-ct` tracked a class ("an ACE TUI test failed under the full parallel run and
> passed in isolation") that matches every future ACE timing failure, so every reporter
> corroborated it and every `+1` reopened it: 55 `+1`s, 8 closures, 8 reopens, two epics
> (`sase-h8`, `sase-h8.10`). The bead could not stay closed because the tracking
> pattern, not the tests, was the defect.
>
> The live instances are fixed by mechanism: `<catalog>` fixed the deterministic
> `prompt-catalog:0` convergence hang that held `wait_for_visual_idle` open for its full
> 30s deadline and made the PNG lane red in isolation; `<waitgate>` widened
> `tools/check_test_wait_helpers` past its `pilot`-receiver and `_wait_until`-name blind
> spots and migrated the attempt-bounded pause loops it then reported, including
> `test_k_on_glossary_term_pushes_glossary_preview_card`; `<residue>` fixed the
> remaining contention-sensitive nodes against a measured `just test-contention`
> before/after tally of `<before>` → `<after>`.
>
> If you hit an ACE TUI test that fails under the full parallel run and passes in
> isolation: **do not `+1` this bead, and do not reopen it.** File a new task bead
> through `/sase_new_task` named for the specific failing node ID, and record
> `RELATED: sase-ct` on it with a note explaining how it bears on the new task. A
> node-specific bead can be fixed and can stay closed; this umbrella could not.
>
> Re-run the measurement with `just test-contention`, `just test-visual-contention`, and
> `tests/reproducible_flake_baseline.txt`.

### 5. Close out the plan

Set `status: done` in this plan's frontmatter. Do not close `sase-h8` or `sase-h8.10` —
those epics have their own land agents and their own remaining scope; note on each that
`sase-ct` was retired here and by what criteria, so their land agents are not left
waiting on a bead that will never reopen.

**Watch out for.** Do not close `sase-ct` before the skill change has landed. If the
skill change lands after the close, the window between them is exactly when the next
agent's `+1` reopens it for a ninth time.
