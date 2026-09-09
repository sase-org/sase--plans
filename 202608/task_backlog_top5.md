---
tier: epic
title: Work the top five SASE task beads
goal: "The five highest-impact task beads in the sase backlog are fixed and closed: the
  reproducible-flake gate stops being a permanent red on check-full, the config-center
  atomic-save node stops flaking under the full parallel lane, `sase bead work` can no
  longer strand an epic with no agents after a partial cleanup, chezmoi's `just check`
  is idempotent, and the Artifacts Files PNG snapshot stops erroring during setup.

  "
phases:
  - id: flake_gate_retirement
    title: Retire a fixed node's historical flake evidence
    depends_on: []
    size: large
    description:
      "flake_gate_retirement: task bead sase-nv. Let the reproducible-flake gate retire
      historical failure evidence for a node that has since been fixed, so `just
      check-full`'s last gate stops being a permanent red."
  - id: config_center_state_flake
    title: Deflake the config-center atomic-save node
    depends_on: []
    size: large
    description:
      "config_center_state_flake: task bead sase-md. Make
      test_save_atomically_replaces_existing_state deterministic under the full parallel
      lane without weakening its atomic-replace assertion."
  - id: bead_work_atomic_cleanup
    title: Make bead-work forced-reuse cleanup all-or-nothing
    depends_on: []
    size: medium
    description:
      "bead_work_atomic_cleanup: task bead sase-mt. Stop `sase bead work` from wiping
      some forced-reuse targets and then aborting, which silently leaves an epic with
      neither its old agents nor new ones."
  - id: chezmoi_check_idempotent
    title: Make chezmoi's just check idempotent
    depends_on: []
    size: small
    description:
      "chezmoi_check_idempotent: task bead sase-m8. Make the chezmoi repo's `just check`
      idempotent so a second consecutive run stops failing fmt-md-check on a pytest
      cache artifact."
  - id: artifacts_files_visual_seam
    title: Repoint the Artifacts Files PNG snapshot seam
    depends_on: []
    size: small
    description:
      "artifacts_files_visual_seam: task bead sase-my. Repoint the stale monkeypatch
      seam in the Artifacts Files PNG snapshot test so it stops erroring during setup."
proposed_by: bbugyi200.athena.sase-ns.land--1
parent_bead: sase-ns
status: done
bead_id: sase-ns.6
create_time: 2026-09-09 19:51:44
---

- **PROMPT:**
  [prompts/202608/task_backlog_top5.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_backlog_top5.md)
- **BEAD:**
  [sase-ns.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.md)

# Work the Top Five SASE Task Beads

## Goal

Five task beads from the `sase` project backlog are worked to completion and closed: the
two that keep the shared verification gates red, the correctness bug that can silently
strand an epic with no agents, and two cheap deterministic breakages whose root causes
are already identified.

These five were selected on 2026-08-16 from a triage of the whole ready backlog on
master `f8b4ebb11`. The triage also closed four beads as no longer relevant (`sase-np`,
`sase-nq`, `sase-nt`, `sase-nu`), routed two newly-found PNG drifts to the active epics
that caused them (`sase-nb`, `sase-jx`), and filed one new task (`sase-ny`).

Every phase below is independent. None depends on another, so they can all run in
parallel.

## Background

Two verification gates are currently red for every agent on this host, regardless of
what that agent changed:

- `just selection-health --fail-on-new-flake` — the last gate of `just check-full` —
  reports 13 reproducible-flake node IDs above `tests/reproducible_flake_baseline.txt`.
  Nine of the 13 are config / config-cache nodes that epic `sase-ns` already fixed in
  commit `3a22ff04f`. The gate cannot un-see their historical failures. That is phase
  `flake_gate_retirement`.
- The full parallel test lane intermittently fails
  `tests/ace/tui/test_config_center_state.py::test_save_atomically_replaces_existing_state`,
  reported independently four times in two days by four unrelated agents. That is phase
  `config_center_state_flake`.

The remaining three phases are independent defects with identified root causes.

## Phases

### `flake_gate_retirement` — sase-nv (large)

**Bead:** `sase-nv` — "flake-baseline gate keeps failing on historical records for nodes
that have since been fixed".

**Verified live** on master `f8b4ebb11` in a fresh workspace after `just install`:

```bash
just selection-health --fail-on-new-flake
# -> flake baseline gate: 13 reproducible flake(s) exceed
#    tests/reproducible_flake_baseline.txt
#    (records after 2026-08-15T17:22:27Z, at most 5 failures per run)
# -> error: recipe `selection-health` failed on line 580 with exit code 1
```

Nine of the 13 are the config nodes `3a22ff04f` fixed:

```
tests/test_config.py::test_legacy_overlay_is_discovered_but_not_a_complete_owner
tests/test_config.py::test_machine_overlays_require_matching_selector_and_keep_ordinary_overlays
tests/test_config.py::test_selected_overlay_identity_cannot_be_overridden_by_other_sources
tests/test_config_cache.py::test_clear_config_cache_forces_reload
tests/test_config_cache.py::test_clear_config_cache_resets_config_token_time_gate
tests/test_config_cache.py::test_current_config_token_refresh_is_single_flight
tests/test_config_cache.py::test_explicit_invalidation_wins_race_with_background_refresh
tests/test_config_cache.py::test_first_config_token_read_does_not_start_worker
tests/test_config_cache.py::test_yaml_content_cache_survives_config_cache_clear
```

The other four belong to live beads and must keep being reported:
`tests/ace/tui/test_top_bar_order.py::test_override_pills_keep_narrow_top_bar_in_bounds`
(sase-mp),
`tests/main/test_var_integration.py::test_var_cli_end_to_end_refreshes_index_and_round_trips_machine_outputs`,
`tests/test_bead/test_cli_golden.py::test_bead_cli_golden_contract[stats]`, and
`tests/test_query_profile.py::test_provider_query_schema_derives_fields_from_the_notes_fixture`.

**Mechanism.** `_flake_evidence_nodeids`
(`tests/_test_selection_health_correlation.py:244`) admits a node when it failed in at
least two eligible full runs with pairwise-disjoint changed-file sets and
`_has_interleaved_independent_pass` finds an independent pass _between_ those two
failures. Interleaving is evaluated only between the recorded failures, so once that
shape exists in the durable store under `~/.sase/test-selection/gh_sase-org__sase/`,
later green runs can never dislodge it. `reproducible_flake_nodeids` (line 319) then
filters only on whether the node still _collects_ — the `sase-j5` fix — which does not
help here, because these nodes still collect and now pass.

The only lever today is `# effective-after:` in `tests/reproducible_flake_baseline.txt`,
parsed at `tools/selection_health:86`. It is file-wide and all-or-nothing: bumping it
far enough to drop the nine fixed config nodes also erases the still-live evidence for
the other four. The `sase-mj` land agent declined exactly that trade when it filed
`sase-mv`.

**Scope.** Give the gate a way to retire a _single_ node's historical evidence once that
node is fixed, without touching any other node's evidence and without weakening the bar
for genuinely new flakes. The bead proposes three directions:

1. a per-node `fixed at <commit>` retirement recorded in the baseline file, so evidence
   older than that commit is ignored for that node only;
2. treat N consecutive eligible passes after a node's last recorded failure as retiring
   its evidence;
3. make `effective-after` expressible per node instead of only file-wide.

Pick one and justify the choice in the phase note. Direction 2 is the only one that
needs no hand-maintenance, which the `_flake_evidence_nodeids` docstring explicitly
argues for ("This check needs no maintenance and generalizes to whatever the next flake
turns out to be") — weigh that against the fact that directions 1 and 3 keep a human in
the loop about what was declared fixed.

**Guardrail — do not weaken the gate.** Whatever direction is chosen, after the change
the gate must still flag a node whose failures continue after the claimed fix point.
Prove that with a regression test, not by inspection.

**Exit criteria.**

- `just selection-health --fail-on-new-flake` exits 0 on master with the nine fixed
  config nodes retired.
- The four still-live node IDs above are _still reported_. If the chosen mechanism
  retires any of them, that is a bug in the mechanism, not a win.
- New regression tests in `tests/test_test_selection_health_correlation.py` and/or
  `tests/test_selection_health_tool.py` cover: (a) a fixed node's old evidence is
  retired, (b) a node that keeps failing past its claimed fix point is still flagged,
  (c) retiring one node does not retire another.
- If the retirement is expressed in `tests/reproducible_flake_baseline.txt`, its header
  comment must explain the new syntax — that file already says entries are "debt to
  remove, not suppressions to grow".

**Approval.** If the direction chosen would reduce the gate's ability to catch genuinely
new flakes — for example if N-consecutive-passes could retire a node that is merely
dormant rather than fixed — do **not** ask the user directly. Leave a note on `sase-nv`
beginning with the exact string `TASK NEEDS APPROVAL`, describing the trade-off and the
alternatives, and stop that part of the work. An implementation that preserves the
gate's strength needs no approval.

### `config_center_state_flake` — sase-md (large)

**Bead:** `sase-md` — flaky
`tests/ace/tui/test_config_center_state.py::test_save_atomically_replaces_existing_state`
under the full parallel lane. Four independent reproductions in two days (`026`, `031`,
`sase-n4.land`, `sase-n9.land`), each from an agent whose diff did not touch Config
Center persistence, and each passing immediately on a focused or serial rerun.

**Scope.** Reproduce under parallel / full-suite conditions, identify the state or
filesystem timing dependency in config-center state persistence, and make the node
deterministic **without weakening its atomic-replace assertion** — that assertion is the
only coverage that the save is actually atomic.

**Notes for whoever works this.**

- The node is already listed in `tests/reproducible_flake_baseline.txt` under
  `# sase-j7`. `sase-md`'s own audit note (`sase-mi.1`) records why it is nevertheless
  not owned by `sase-j7`: that epic's scope does not include this node, all five of its
  phases are closed, and CI still failed the node after `5601920c9`.
- A file-scoped `just test-contention` run is **not** sufficient evidence for this
  class. `sase-mv` was green at file level while the full lane was red. Reproduce under
  the whole-suite lane.
- `sase-j7.2` built a global-state leak detector. If the trigger is a poisoning test
  rather than a race inside the save path, that detector is the right instrument.
- If the fix lands, the node should also come out of
  `tests/reproducible_flake_baseline.txt` — coordinate with `flake_gate_retirement` if
  both phases want to edit that file, since they may collide there. Neither phase blocks
  the other; whichever lands second rebases.

**Exit criteria.** The node passes under a full parallel lane run, the atomic-replace
assertion is intact, and the phase note records the actual root cause — not just that
the symptom stopped.

### `bead_work_atomic_cleanup` — sase-mt (medium)

**Bead:** `sase-mt` — `sase bead work` aborts mid-cleanup after already wiping some
forced-reuse targets.

**Observed once, then self-cleared:** `sase bead work sase-m6.6.1 -Y` previewed five
cleanup targets (two KILL, three REMOVE), wiped part of the selection, then failed with
`Error: bead-work cleanup target sase-m6.6.1.6--code is no longer eligible for destructive cleanup`
and launched nothing. An immediate identical rerun succeeded and previewed only the two
KILL targets — the three REMOVE targets were already gone. The epic was left with
neither its old agents nor new ones, and nothing relaunched it.

**Mechanism** (`src/sase/bead/cli_work_cleanup_apply.py`):

```python
for target in selection.destructive_targets:
    _verify_cleanup_target_still_selected(
        target, selection=selection, bead_assignees=bead_assignees
    )
    if target.action == "RELEASE":
        _release_selected_stale_container(target)
        continue
    wipe_force_reuse_owner(target.name, allow_container_skip=False)
```

`_verify_cleanup_target_still_selected` (same file, ~line 104) re-derives the selection
and raises `ForcedReuseCleanupError` when the target is no longer among
`destructive_targets`. The guard itself is correct — it is a TOCTOU check — but it runs
_inside_ the wipe loop, so targets already processed stay wiped when a later one fails.

**Scope.** Make the destructive-cleanup phase all-or-nothing, or make the abort path
roll back or automatically re-drive the launch from the recomputed selection. The
lowest-risk shape is a pre-flight: verify **every** destructive target before wiping
**any** of them, so a stale target aborts with nothing destroyed. That does not close
the window entirely — a wipe can still fail partway — so at minimum the partial-wipe
error must name what was already removed and say that the epic now has no live agents.
The sibling message at the bottom of `revalidate_bead_work_launch_selection` already
tells the operator to "rerun", which suggests retry was always meant to be the
operator's job; make the error honest about the state it is leaving behind.

**Exit criteria.** A regression test proves that when verification fails for a later
target, no earlier target was wiped. A second test covers the message content on a
genuine partial wipe. `just check` green.

### `chezmoi_check_idempotent` — sase-m8 (small)

**Bead:** `sase-m8` — chezmoi `just check` is not idempotent.

**CROSS-REPO.** This work is in the `chezmoi` linked repo, not this one. Open it with
the `/sase_repo` skill first and use the path that skill prints as the only path for
reads and writes. Do not clone, web-fetch, or otherwise locate it another way.

**Reproduction.** From a clean chezmoi checkout, `just check` passes. Run it again
immediately with no source changes and it fails at fmt-check:
`prettier --check --prose-wrap=always --print-width=88 '**/*.md'` reports "Code style
issues found in .pytest_cache/README.md" and `fmt-md-check` exits 1.

**Root cause.** The `test-python` recipe runs
`cd home/lib/xfile && ../../../.venv/bin/pytest test`, but pytest resolves its rootdir
from the repo-root `pyproject.toml` rather than the cwd, so it writes
`.pytest_cache/README.md` at the chezmoi repo root. That directory is gitignored — so
`git status` stays clean — but prettier's `**/*.md` glob does not respect `.gitignore`
and still matches it.

**Scope.** Either (1) add `.pytest_cache` to `.prettierignore`, or pass
`--ignore-path .gitignore` to the prettier invocation in the `fmt-md` / `fmt-md-check`
recipes, or (2) pin pytest's cache-dir / rootdir so the artifact never lands at the repo
root. Prefer whichever leaves the fewest surprises for the next agent; if you take
option 1, make sure `fmt-md` (the writing recipe) and `fmt-md-check` stay consistent
with each other.

**Verification for this phase is in the chezmoi repo, not this one.** From a clean
checkout with no `.pytest_cache` present, run that repo's `just check` twice in a row
and confirm both runs pass. Then delete `.pytest_cache`, run `just check` twice again,
and confirm the same. Do **not** run this repo's `just check` for this phase — the diff
is not in this repo.

### `artifacts_files_visual_seam` — sase-my (small)

**Bead:** `sase-my` — Repair Artifacts Files PNG detail monkeypatch after pane split.

**Verified live** on master `f8b4ebb11` in a fresh workspace after `just install`, in a
serial (`-n 0`) run, so this is not xdist contention:

```bash
.venv/bin/pytest -q -n 0 -m visual tests/ace/tui/visual/
# -> 16 failed, 681 passed, 14 deselected in 1506.01s
```

One of the 16 is
`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_files.py::test_artifacts_files_populated_png_snapshot`.
It fails with
`AttributeError: <module 'sase.ace.tui.widgets.artifacts.files_options'> has no attribute 'local_now'`
**before any PNG comparison happens**, so regenerating goldens cannot fix it.

**Mechanism.** The test monkeypatches `files_options.local_now` (around
`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_files.py:159`). Commit
`c756a7c63` moved Files pane detail loading into a focused module and `files_options` no
longer imports `local_now`; `sase.core.time.local_now` is now reached from the modules
that actually render (see `artifacts/beads_data.py` and `artifacts/plans_rendering.py`
for the current import shape).

**Scope.** Repoint the stale seam at the module that owns the call today so the test
patches something real, then verify the affected PNG snapshot and the surrounding
Artifacts Files visual coverage. If the golden itself has also drifted, accept only a
confirmed-intentional change with `--sase-update-visual-snapshots` on that node — do
**not** run `just test-visual-update` broadly, because closed bead `sase-lo` records
that it rewrites every golden in the suite and silently absorbs unrelated drift.

**Exit criteria.** The node passes in `just test-visual`, and the phase note records
whether a golden was regenerated and why.

**Do not widen.** The other 15 failures in that run are already routed elsewhere: four
retry-e2e nodes to `sase-dc`, three Artifacts Beads nodes to active epic `sase-nb`, one
AXE chop-overrun node to active epic `sase-jx`, and seven to the new task `sase-ny`.
Leave them alone.

## Bead Bookkeeping — Every Phase

Each phase owns exactly one task bead. For its own bead, the phase worker MUST:

1. **Mark it in progress before starting the work:**

   ```bash
   sase bead update <task-id> --status in_progress
   ```

2. **Leave a note when the work is done**, explaining what was changed to fix the
   reported issue — or, if the work could not be completed, justifying why:

   ```bash
   sase bead note <task-id> "<what was done, or why it could not be>"
   ```

3. **Close it if it was finished:**

   ```bash
   sase bead close <task-id> --note "<what you verified>"
   ```

   Leave it open if it was not finished; the note from step 2 is then the handoff.

4. **Escalate instead of asking.** If the work genuinely needs the project owner's
   judgment before proceeding, do not ask the user directly and do not stall. Leave a
   note beginning with the exact string `TASK NEEDS APPROVAL` on that bead, describing
   what needs deciding and the options, then continue with whatever part of the work
   does not depend on that decision. Be lenient: objective bug fixes need no approval.
   Only `flake_gate_retirement` is expected to possibly need this, and only under the
   condition stated in its section.

These are task beads, not this epic's phase beads, so the usual "phase workers never
touch beads" rule does not apply to them — but it still does for _creating_ beads. A
phase worker that discovers unrelated follow-up work records it as a
`PROPOSED FOLLOW-UP:` note on its own phase bead and lets the land agent route it.

## Verification

Per this repo's two-speed rule, every phase except `chezmoi_check_idempotent` runs
`just check` before landing, and `just install` first because these workspaces are
ephemeral. Hand `just check-full` to `/sase_monitor` with a `--next` action rather than
running it inline; `just check` may run inline but belongs in a monitor too if it is
taking a long time.

`chezmoi_check_idempotent` verifies in the chezmoi repo instead, as described in its
section.

`flake_gate_retirement` additionally runs `just selection-health --fail-on-new-flake`
and must see it exit 0 with the four still-live node IDs still reported.

## Landing

After all five phases close, the land agent completes this epic's landing normally and
then does one extra thing.

More ready task beads remain for the `sase` project beyond these five — at the time of
writing roughly 20, including `sase-dc`, `sase-kh`, `sase-m0`, `sase-m1`, `sase-m3`,
`sase-mu`, `sase-n3`, `sase-n5`, `sase-n6`, `sase-nc`, `sase-nd`, `sase-ne`, `sase-nf`,
`sase-ng`, `sase-nh`, `sase-ni`, `sase-nl`, `sase-nn`, `sase-nr`, and `sase-ny`.

So the land agent MUST check the live backlog:

```bash
sase bead list --type task --status open --status ready --limit 0
```

**If any ready or open task beads remain**, start a pseudo monitor through
`/sase_monitor` running `sleep 1`, with a `--next` action instructing the next agent to
follow this exact prompt, with these exact steps:

> (1) Review all of my current open (not in-progress) sase task beads for the "sase"
> sase project. (2) Close any task beads that are no longer relevant with a good reason.
> (3) Select the 5 task beads that would have the most impact if worked to completion.
> (4) Use your /sase_plan skill to fix the issues / make the improvements that
> correspond with these 5 task beads. Make sure the plan file you propose tells the
> agent(s) to: (a) if you think any of these 5 beads need approval from the user before
> working (be lenient here and do not ask for approval for objective improvements), do
> not ask directly, but instead leave a `TASK NEEDS APPROVAL` note on the bead; (b) mark
> the bead(s) you intend to work as in-progress by changing their status with the
> `sase bead update` command; (c) leave a brief note on the task bead(s) explaining the
> work that was done to fix the reported issue / make the requested improvement or, if
> the agent was unable to complete the work, justifying why they were unable to do so;
> (d) close each of the 5 task beads that it was able to finish; (e) if there are more
> task beads associated with the "sase" project, the agent should then start a pseudo
> monitor using the `sleep 1` command with a next action that instructs the next agent
> to follow this exact prompt (with these exact same steps); (f) if there are no more
> task beads to work, the agent should move on to the next numbered step in this prompt.
> (5) Review all `TASK NEEDS APPROVAL` notes left by prior agent shells and consolidate
> them into a single report for the user with suggested next actions. (6) Terminate.

**If no ready or open task beads remain**, skip the pseudo monitor. Instead, sweep every
`TASK NEEDS APPROVAL` note left by this epic's phase workers and by prior agent shells,
and consolidate them into a single report for the user with suggested next actions.

## Prior `TASK NEEDS APPROVAL` Notes — None Outstanding

Swept before this plan was written: `sase bead search "TASK NEEDS APPROVAL"` and a
direct grep of the bead store (`events/streams/`, `issues.jsonl`, `pages/`) return
matches only inside the predecessor epic `sase-ns`'s own description and close note.
`sase-ns` recorded the same finding when it landed — no phase left one.

So there is currently **nothing outstanding for the project owner to decide**, and no
consolidated approval report is owed yet. Whoever runs the next triage round should
re-sweep rather than assume this still holds; `flake_gate_retirement` in this epic is
the phase most likely to add the first real one.

## Out Of Scope

- The other 15 visual failures in the serial `just test-visual` run. They are routed to
  `sase-dc`, `sase-nb`, `sase-jx`, and `sase-ny`.
- The remaining ~20 ready task beads. The landing handoff above passes them to the next
  triage agent.
- Bumping `# effective-after:` file-wide in `tests/reproducible_flake_baseline.txt` as a
  way to clear `sase-nv`. That is the all-or-nothing lever the bead exists to replace,
  and it would erase the four still-live node IDs along with the nine fixed ones.
