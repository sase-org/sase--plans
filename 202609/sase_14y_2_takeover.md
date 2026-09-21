---
tier: tale
title: Take over sase-14y.2 — land the labeled launch-context cluster
goal:
  The sase-14y.2 phase work (labeled launch-context cluster on every tab's status row)
  is ported from the stalled sase-14y.2 agent onto current master, verified with `just
  check`, screenshots refreshed right before an explicit commit, the sase-14y.2 agent
  killed before that commit, and the sase-14y.2 bead closed.
size: medium
model: opus
proposed_by: bbugyi200.apollo.1h
create_time: 2026-09-21 16:55:18
status: wip
---

# Plan: Take over the stalled `sase-14y.2` phase agent and land its work

## Context

The user asked this agent to take over the `sase-14y.2` phase agent, which has been
running for ~16h. Their instructions, verbatim in substance:

- Copy that agent's work as much as possible.
- Update the screenshots **right before committing**, then **immediately commit** (the
  user explicitly asks for a manual commit, so use `/sase_git_commit`).
- If we are ready to commit before that agent finishes, kill it
  (`sase agent kill -n sase-14y.2`) **before** committing.
- Close the `sase-14y.2` bead once the work is done.

Phase `sase-14y.2` is phase 2 of epic `sase-14y`. The epic plan is
`plan:202609/launch_context_row.md`, and its "Phase 2" section is the spec. The phase
builds `LaunchContextBar`, mounts it on the Agents, Artifacts, and Services status rows,
drops the model and project chips from `#top-bar`, rewrites the tooltips, and refreshes
tests, goldens, and docs. Phase 1 (`sase-14y.1`, `LaunchContextSource`) is closed and on
master. The epic's land agent `sase-14y.land` is WAITING on this bead and starts on its
own once the bead closes. Never close the epic bead `sase-14y`.

### State of the other agent's work at planning time

Find its workspace with `sase agent list -j`. It is the entry named `sase-14y.2`, and
its checkout is the sibling `sase_<workspace_num>` directory next to your own checkout
root. Call that path `$WS`. `$WS` has no commits of its own: its HEAD is the base the
agent started from (`42acc2979`, 45 commits behind master at planning time). All of its
work is uncommitted.

- **Code, tests, and docs are done.** The last source edit was about 14h before
  planning. Since then the agent has only been re-banking goldens file by file. It
  passed `just lint`. It never ran `just check`.
  - Tracked changes (25 files): `docs/ace.md`, `src/sase/default_config.yml`,
    `src/sase/ace/tui/{_app_layout.py,styles.tcss,actions/agent_workflow/_leader_mode.py}`
    and
    `src/sase/ace/tui/widgets/{__init__.py,__init__.pyi,_override_pill.py,agent_info_panel.py,artifacts/view.py,axe_info_panel.py,current_project_indicator.py,launch_context_source.py,llm_override_indicator.py}`,
    plus tests under `tests/ace/tui/` and `tests/test_llm_override_indicator*.py` /
    `tests/test_launch_default_indicator_pool_rotation.py`.
  - New files: `src/sase/ace/tui/widgets/launch_context_bar.py` (`LaunchContextBar`,
    `AgentInfoRow`, `AxeInfoRow`, `ArtifactsHeader`, and
    `_choose_launch_context_density`), `tests/ace/tui/test_launch_context_bar.py`,
    `tests/ace/tui/visual/test_ace_png_snapshots_launch_context_bar.py`, and five new
    goldens
    `tests/ace/tui/visual/snapshots/png/launch_context_bar_{agents,artifacts,services,override}_120x40.png`
    plus `launch_context_bar_compact_60x24.png`. The agent chose 60x24 over the spec's
    80x24 so the compact form really shows; keep it.
- **Goldens:** 312 modified PNGs plus the 5 new ones, and no deletions. 21 of the
  modified PNGs were also changed on master since `42acc2979` (for example by the `?N`
  unknown-wait clan indicator, the panel-focus change, and the Services-tab renames).
  Those 21 must be re-captured, not copied.
- **Known pre-existing visual flakes** are recorded as `PROPOSED FOLLOW-UP` notes #1–#6
  on bead `sase-14y.2`; read them with `sase bead read sase-14y.2 -r "<why>"`. They
  cover the zoom wait-bead badges, gate scroll, the agents_neighbors determinism flakes
  (related to task `sase-14w`), the waiting-file goldens, and a transient top-bar
  element near row 3 on the right. Note #6 covers the codex usage probe leaking
  `sase-codex-usage-*` tempdirs, which trips the tmp-leak guard. The agent banked those
  goldens with `SASE_TMP_LEAK_GUARD_DISABLED=1`; pixels are still determinism-verified.
- `just fix-tui-screenshots` is all-or-nothing. Any visual pytest failure or any single
  determinism-verify mismatch means no golden is written. A full run is about 22 min of
  capture plus a verify pass over only the changed nodes. The per-file sweep workaround
  is what has eaten most of the other agent's 16h. Do not repeat it.

At planning time, `git apply -3` of the agent's non-PNG diff applied cleanly to master.
A plain `git apply` fails only on `_app_layout.py`, because master now uses
`SERVICES_TAB` for `axe_classes`; the 3-way merge keeps that. No commit on master since
`42acc2979` touches the widget selectors or top-bar order this phase changes.

## Steps

### 0. Preflight

1. Run `sase agent list -j` and `sase bead show sase-14y.2`.
   - If `sase-14y.2` is no longer RUNNING and its work already landed on master
     (`git log --oneline -20` shows it, or the bead is CLOSED), do not duplicate it.
     Report that and stop.
   - Otherwise resolve `$WS` and record `BASE=$(git -C "$WS" rev-parse HEAD)`.
2. Read `sase memory read lint_and_test.md tui.md` with a reason.
3. Run `just install` if the venv looks stale.

### 1. Port the code, tests, and docs

1. Export and apply the non-PNG diff:
   ```bash
   git -C "$WS" diff -- . ':(exclude)*.png' > /tmp/sase14y2_code.patch
   git apply -3 /tmp/sase14y2_code.patch
   ```
   Resolve any conflict by keeping master's side for unrelated upstream edits and the
   agent's side for launch-context edits.
2. Copy the agent's untracked files
   (`git -C "$WS" status --porcelain --untracked-files=all | grep '^??'`), keeping their
   paths: the three `.py` files and the five `launch_context_bar_*.png` goldens.
3. Save the content signature of what you ported for step 5's re-sync: `sha256sum` of
   the patch plus the untracked non-PNG files.
4. Sanity-check the result:
   - `_app_layout.py` still imports and uses `SERVICES_TAB`.
   - `git grep -n "query_one(\"#llm-override-indicator\"\|query_one(\"#current-project-indicator\"\|query_one(LLMOverrideIndicator\|query_one(CurrentProjectIndicator" -- src tests`
     finds no single-instance lookups that the port missed. Several views now exist, so
     callers must use `query(...)` or go through `LaunchContextSource`.

### 2. Seed the agent's goldens as a head start

For every PNG the agent modified (`git -C "$WS" diff --name-only -- '*.png'`):

- If `git diff --quiet "$BASE" HEAD -- <png>` succeeds (the file is unchanged on master
  since the agent's base), copy `$WS/<png>` over yours.
- Otherwise skip it. These are the ~21 overlapping files, and the final run re-captures
  them.

The seeds are not trusted blindly: step 6's full run compares every golden with a fresh
capture and re-banks any seed that differs. Seeding just keeps that run's
determinism-verify set small, so a single flake is less likely to block the apply.

### 3. Add one full-density golden (small, deliberate addition)

None of the agent's goldens shows the labeled `default … · current +sase` form, because
at 120 columns every row is too crowded and falls back to compact. Pin the headline
labels visually:

1. Add `test_launch_context_bar_full_density_png_snapshot` to
   `tests/ace/tui/visual/test_ace_png_snapshots_launch_context_bar.py`, following that
   file's existing helpers and fixtures.
2. Use the widest-free row: Services at 160x40, or whatever width is the smallest one
   where the bar picks `full`. Before capturing, wait until `bar.density == "full"`.
3. Create its golden with a targeted run:
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_launch_context_bar.py::test_launch_context_bar_full_density_png_snapshot`.
4. View the PNG with the Read tool and confirm it shows the dim `default` / `current`
   labels, the two-tone model chip, and `·` with no dangling separator.

### 4. Verify

1. Run `just fix`.
2. Run `sase tool run check`, or `just check` if `sase tool` is unavailable. Fix any
   real failures in the ported code or tests; the agent never ran this gate.
3. Keep `just check-full` out of this: nobody explicitly asked for it.

### 5. Final re-sync, then kill the agent

This is the "ready to commit" point: only the pre-commit screenshot update remains.

1. Re-diff `$WS` against step 1's signature. The agent is still sweeping.
   - Port any new non-PNG change. If one appears, rerun step 4.
   - Re-apply step 2's seeding rule to any PNGs it banked since your first seed.
2. Run `sase agent kill -n sase-14y.2`. Confirm with `sase agent list` that it is no
   longer RUNNING. Killing it before the heavy final run also removes its competing
   visual sweeps, which the agent itself blamed for slowness and flakes.
3. Confirm that no `sase-14y.2` commit landed on master (`git log --oneline -10`). If
   one did, reconcile against it instead of duplicating.

### 6. Update the screenshots, right before committing

1. Run the full update in the foreground with a generous explicit Bash timeout (for
   example 3h):
   ```bash
   SASE_TMP_LEAK_GUARD_DISABLED=1 just fix-tui-screenshots
   ```
   The env var works around the pre-existing codex tempdir leak from note #6. Pixels are
   still determinism-verified.
2. If it fails because of a visual pytest failure or a determinism mismatch:
   1. Collect the failing node IDs from the run's manifest `errors`, `capture.log`, and
      `verify.log` (latest run under `.pytest_cache/sase-visual/runs/`).
   2. Rerun the update with only those nodes deselected (`-- --deselect <node> …`), so
      every other golden applies.
   3. Retry each deselected node solo at most twice.
   4. Cap the whole step at three update invocations after the first run. Do not start a
      file-by-file sweep.
   5. If a node still cannot bank and it is a known pre-existing flake (notes #1–#6),
      leave its seeded golden in place. Append a
      `PROPOSED FOLLOW-UP: <node> could not bank after takeover — <symptom>` note to
      `sase-14y.2` with `sase bead note`, so the land agent triages it. Do not create
      task beads.
   6. If a node that is not a known flake fails, and it fails for a reason caused by
      this change, fix the cause and rerun.
3. Inspect the run:
   - Read the retained report.
   - View every created golden with the Read tool: the five `launch_context_bar_*` files
     and the new full-density one.
   - Confirm there are no stale removals.
   - Open one representative per update group. Per the epic plan, the incidental
     updates, which come from the chips leaving the top bar and the cluster appearing on
     each status row, do not need individual review. Expand any group whose diff is not
     one of those.

### 7. Commit immediately

Use `/sase_git_commit` right away. The user explicitly asked for this manual commit.

1. Review `git status`: every change should be one of the ported files, the new files,
   the goldens, and step 3's test. Make sure no scratch files such as `/tmp` copies or
   plan scratch files are in the tree.
2. Commit with a message like
   `feat(ace): labeled launch-context cluster on every tab's status row`. In the body,
   mention the sase-14y.2 takeover and the key changes: `LaunchContextBar` on the
   Agents, Artifacts, and Services rows; chips removed from the top bar; rewritten
   tooltips; goldens refreshed.

### 8. Close the bead

```bash
sase bead close sase-14y.2 --note "Taken over from the stalled sase-14y.2 agent (killed): ported its launch-context-bar work onto master, just check green, goldens refreshed via fix-tui-screenshots, committed <sha>."
```

Mention any unbanked flaky nodes in the note. Do not close epic `sase-14y`; its land
agent does that.

## Acceptance

- On master, the Agents, Artifacts, and Services status rows each end with the
  `LaunchContextBar`. `#top-bar` no longer contains `LLMOverrideIndicator` or
  `CurrentProjectIndicator`. The tooltips and docs match the epic spec as the agent
  implemented it.
- `sase tool run check` (or `just check`) passes.
- `just fix-tui-screenshots` applied in the run immediately before the commit. Any
  golden it could not bank is a known pre-existing flake, recorded as a
  `PROPOSED FOLLOW-UP` note.
- The `sase-14y.2` agent was killed before the commit, one commit contains the work, and
  bead `sase-14y.2` is CLOSED.
