---
tier: tale
title:
  Restore the stranded Refs Command-/ commits and close the raw-commit gaps that
  stranded them
goal:
  Both stranded commits land upstream, and an agent's raw git commit in a linked repo or
  a deferred-workspace run is absorbed and pushed or refused, never silently left local.
size: medium
decisions:
  monitor_next_wording:
    ask:
      Should the /sase_monitor skill also warn starters never to phrase a `--next`
      action as "commit <repo>"?
    default: true
    why:
      The stranded commits began with a `--next` saying "commit bob-mac-capture with
      message …"
    answer: true
proposed_by: bbugyi200.apollo.an.f0.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 14:29:50
status: wip
---

# Restore the stranded Refs Command-/ commits, then close the raw-commit gaps that stranded them

## Goal

1. Land the two commits that agent `an.f0--1` reported but never pushed:
   - bob-mac-capture:
     `feat(hotkey): open Bob Refs with Command-/ as well as Control-Shift-Command-O`
   - bob-cli: `docs(guides): document the Mac Capture Command-/ Refs alias`
2. Fix the sase host so an agent's raw `git commit` is absorbed and pushed, or loudly
   refused. It must never again be reported as success and left local-only.

Do Part 1 first. Its recovery sources are time-limited: rescue entries expire after 30
days, and GitHub can garbage-collect the dangling CI snapshot.

## Diagnosis (verified)

Timeline from session `an.f0` in project `gh_bobs-org__bob-cli`, workspace #14. Times
are UTC on 2026-10-27.

1. **17:23–17:45.** The plan-approved coder `an.f0--code` implemented the alias. It left
   the work uncommitted and handed off to a monitor. The monitor ran
   `Scripts/macos-ci-check` (run 40553524579), which was fully green on snapshot commit
   `e1afc2bb` (parent `de57e2b`). The coder's `--next` text said "commit bob-mac-capture
   with message … and commit bob-cli with message …". It did not say "submit
   `/sase_final` with commit decisions".
2. **18:03:22.** The Muse follow-up `an.f0--1` ran raw `git commit` in two places:
   - the bob-mac-capture linked clone, producing `3299706`;
   - the bob-cli workspace, producing `0ac6c3e2`.

   It never pushed. Its final context shows `repositories: []` and
   `absorbed_raw_commits: []`. The `builtin@commit` finalizer then reported **success**.
   The raw-commit absorber and backstop in
   `src/sase/llm_provider/commit_finalizer_raw_commits.py` missed both commits:
   - **Gap A, linked repos are out of scope.** `_RAW_COMMIT_REPO_KINDS` is
     `{"main", "sibling", "external"}`. A repo opened with
     `sase repo open <linked-name>` is recorded in `finalizer_baseline.json` with
     `kind: "linked"` (`OpenedRepoKind` in `src/sase/_linked_repo_markers.py`).
     Absorption and the `unpublished_local_commits` backstop therefore never scan linked
     clones. The existing tests only cover this case with `kind="sibling"`, so they
     never exercised the real record.
   - **Gap B, the run-start baseline was bound to the wrong checkout.** The `main`
     record pointed at the bob-cli _primary checkout_, not workspace #14. The scan
     therefore ran `git log HEAD --not --remotes <head>` in the wrong repo. Three paths
     produce this:
     - **B1, deferred-workspace runs.** `_capture_commit_finalizer_baseline` runs in
       bootstrap (`src/sase/axe/run_agent_runner_bootstrap.py`, after
       `enter_agent_workspace`). That happens _before_ `claim_deferred_workspace` in
       `src/sase/axe/run_agent_runner_launch.py::_prepare_workspace_and_repos`. Until
       the claim, the placeholder workspace is workspace #1, the primary checkout. See
       `_resolve_slot_workspace` in `src/sase/agent/launch_executor_workspace.py`.
       Nothing recaptures after the claim.

       This affects every run that waits on a fork, agent, or bead, including
       `an.f0--plan` and epic phases. Their baselines also carry `sibling` records for
       the user's primary checkouts of linked repos. The absorber could soft-reset the
       user's own unpushed commits there.

     - **B2, in-process follow-up runs.** The plan-approved coder (`an.f0--code`) gets a
       new artifacts dir from `create_followup_artifacts`
       (`src/sase/axe/run_agent_helpers_artifacts.py`). That dir never captures or
       inherits a baseline. Its `finalizer_baseline.json` held only `opened_repo`
       records and no `main` record at all.
     - **B3, inherited baselines.** Agent-session-attach continuations copy the parent's
       baseline verbatim (`_inherit_parent_commit_finalizer_baseline`). `an.f0--1`
       therefore inherited the plan run's stale record, which pointed at the primary
       checkout.

3. **18:07:27.** Agent `an.f0.w0--plan` had been waiting on `an.f0` (`%w:an.f0`). It
   claimed workspace #14. Launch-time preparation then did two things:
   - It rescued the bob-mac-capture linked clone into
     `~/.sase/rescue/202610/20261027T180728Z-ws14-linked-bob-mac-capture-5c1cede8/`,
     then deleted it. The clone was re-cloned at 18:08.
   - It rebased the workspace's local bob-cli commit onto `origin/master` as `5f97b514`.
     That commit is still unpushed in that workspace.

   Neither commit is on GitHub.

Rescue-then-evict is working as designed. Once Gap A and Gap B are fixed, a turn like
`an.f0--1` absorbs and pushes its raw commits. If absorption is ineligible, the turn
fails `unpublished_local_commits` and its workspace is held, so no waiting agent can
evict it. Do not change the eviction design.

## Part 1: Restore the work

Agents never run `git commit` or `git push`. Apply each change as uncommitted work in an
opened checkout. Then give that repository a `commit` decision in `/sase_final`, and the
host commits it with SASE trailers and pushes it. The original SHAs `3299706` and
`0ac6c3e2` will not be reused. That is expected.

Do not modify the bob-cli workspace #14 checkout or its linked clones. Session
`an.f0.w0` occupies it.

### 1a. bob-mac-capture

1. Open it:
   `sase repo open gh:bobs-org/bob-mac-capture -r "Restore stranded Refs Command-/ commit"`.
   bob-mac-capture is not linked to the sase project, so it opens as an external repo.
   Read its `AGENTS.md`. Work only in the printed path, on `master` tracking
   `origin/master`, with a clean tree. Run `git fetch origin`.
2. If `git grep -q RefsAliasHotkeyRegistration origin/master -- Sources` matches, the
   alias already landed another way. Skip 1a and say so in the final report.
3. Get the rescued commit:
   `git fetch ~/.sase/rescue/202610/20261027T180728Z-ws14-linked-bob-mac-capture-5c1cede8/local-commits.bundle 'refs/*:refs/sase/rescued/20261027T180728Z/*'`.
   Check that `git rev-parse 32997066657d967615a1c90b721ba1eb3ba28f29^{tree}` prints
   `427a603b6953f060a5231c8573906fe7dd54e72e`.
   - **Fallback A.** If the bundle is gone, run
     `git fetch origin e1afc2bbacd97cb15742494604b93239eae23925`. That is the CI
     snapshot, which has the same tree and parent. Use it in place of `3299706`.
   - **Fallback B.** If both are gone, re-implement from the approved plan
     `202610/mac_refs_command_slash_hotkey.md` in the bob-cli project's plan archive.
4. Run `git cherry-pick --no-commit 3299706`, or the fallback SHA. This leaves the 14
   files as uncommitted changes:
   - 6 sources, including the new `RefsAliasHotkeyRegistration.swift`;
   - 3 tests, including the new `RefsAliasHotkeyRegistrationTests.swift`;
   - 5 docs.

   Then run `git reset` so nothing is left staged in a confusing state. The host stages
   at commit time.

5. Verification depends on whether `origin/master` has moved.
   - **Still `de57e2b7ef06e4421d6e0f07fbacffaf776f2f39`.** Confirm with
     `git add -A && git write-tree` that the tree is
     `427a603b6953f060a5231c8573906fe7dd54e72e`, then `git reset`. That exact tree is
     what macos-ci-check run 40553524579 compiled, tested, bundled, launch-smoked, and
     install-tested, all green. Cite that run as the compile and test evidence
     `AGENTS.md` requires.
   - **It moved.** Resolve any conflicts, then verify per `AGENTS.md`:
     - `just portable-check` on athena;
     - a Mac scratch `./Scripts/bundle.sh` plus `just typecheck-tests`, best effort;
     - `BOB_CLI_DIR=<bob-cli checkout> just docs-check`;
     - `Scripts/macos-ci-check` through `/sase_monitor`. Every macOS step must be green.

     The monitor's `--next` must say "submit `/sase_final` with commit decisions". It
     must never just say "commit".

6. Commit decision message:
   `feat(hotkey): open Bob Refs with Command-/ as well as Control-Shift-Command-O`.

### 1b. bob-cli docs

1. Open it: `sase repo open bob-cli -r "Restore stranded Command-/ Refs docs commit"`.
   Read its `AGENTS.md`. Use the printed path on `master`, clean, after
   `git fetch origin`.
2. If `origin/master:docs/guides/resolve-mac-capture-hotkey-conflict.md` already
   contains ``Refs alias: `Command-/` ``, skip 1b and say so.
3. Apply the patch below with `git apply`. Apply it exactly, with no edits. An identical
   patch keeps the same patch-id as the stranded commit `5f97b514` in workspace #14.
   That workspace's next `git rebase origin/master` then drops its duplicate
   automatically.

   If the patch no longer applies cleanly because the guides changed upstream, port it
   by hand. In that case, report that workspace #14's stranded `5f97b514` may conflict
   on its next rebase.

4. Run `just docs-check` in the bob-cli checkout.
5. Commit decision message:
   `docs(guides): document the Mac Capture Command-/ Refs alias`. Never touch the
   bob-cli `Cargo.toml` version or `CHANGELOG.md`.

````diff
diff --git a/docs/guides/recover-bob-refs.md b/docs/guides/recover-bob-refs.md
index 6f5b2941..c2ddff19 100644
--- a/docs/guides/recover-bob-refs.md
+++ b/docs/guides/recover-bob-refs.md
@@ -15,7 +15,7 @@ Reuse the vault and ref-job guides below for shared CLI failures.

 ## Before you start

-Install the CLI and Bob Mac Capture. Open Bob Refs (Control-Shift-Command-O).
+Install the CLI and Bob Mac Capture. Open Bob Refs (Control-Shift-Command-O or `Command-/`).
 These commands are read-only:

 ```bash
diff --git a/docs/guides/resolve-mac-capture-hotkey-conflict.md b/docs/guides/resolve-mac-capture-hotkey-conflict.md
index 43780c40..fb02593d 100644
--- a/docs/guides/resolve-mac-capture-hotkey-conflict.md
+++ b/docs/guides/resolve-mac-capture-hotkey-conflict.md
@@ -20,6 +20,7 @@ Install Bob Mac Capture and open Settings. Default chords:
   Control-Shift-Command-R (development)
 - Capture alias: `Command-\` (on by default; Settings › Hotkey)
 - Bob Refs: Control-Shift-Command-O
+- Refs alias: `Command-/` (on by default; Settings › References)
 - Highlights-frontmost open: Command-O (configurable)

 Settings › Hotkey shows the live capture chord as **Active binding**.
@@ -29,7 +30,8 @@ Settings › Hotkey shows the live capture chord as **Active binding**.
 Open Settings › Diagnostics. A failed capture registration reports
 `Hotkey conflict` with the Carbon status. A failed Refs binding reports
 `Refs hotkey conflict: …`. A failed Command-\ alias reports
-`Capture alias hotkey conflict: …`. The same event is in Recent Activity
+`Capture alias hotkey conflict: …`. A failed Command-/ alias reports
+`Refs alias hotkey conflict: …`. The same event is in Recent Activity
 (`HH:mm:ss  message`). Confirm that status string before changing a
 binding.

@@ -49,14 +51,16 @@ need that development binding.
 Refs chords re-register as soon as you toggle **Open Bob Refs with
 Control-Shift-Command-O** or the Highlights-frontmost open key. The
 Capture alias toggle **Also open Capture with Command-\\** re-registers
+immediately. **Also open Bob Refs with Command-/** re-registers
 immediately. Each action registers independently.

 ## 3. Confirm the live chord

 Settings › Hotkey **Active binding** shows the live capture chord.
 Press it and confirm the capture panel opens. For Refs, press
-Control-Shift-Command-O and confirm Bob Refs opens. Diagnostics no
-longer shows `Hotkey conflict` or `Refs hotkey conflict`.
+Control-Shift-Command-O and `Command-/` and confirm Bob Refs opens.
+Diagnostics shows no `Hotkey conflict`, `Refs hotkey conflict`,
+or alias conflict.

 ## If it goes wrong

@@ -64,7 +68,10 @@ If Recheck Bob still reports the conflict, another process still owns
 that exact chord. Check Shortcuts, other menu-bar apps, and the
 Highlights-frontmost open key (Command-O or Control-O) separately from
 the global Refs toggle. Apps such as 1Password, whose autofill shortcut
-defaults to `Command-\`, commonly claim the Capture alias. Launch-at-login
+defaults to `Command-\`, commonly claim the Capture alias. In-app
+shortcuts such as editors' Toggle comment never block the Command-/
+alias; they just stop receiving it while the alias is on. Only another
+global hotkey owner blocks it. Launch-at-login
 approval is a different permission; use
 [Restore Mac Capture permissions](restore-mac-capture-permissions.md)
 when Settings › Launch reads `Requires approval in System Settings`.
````

## Part 2: Fix Gap A, absorb raw commits in linked repos (sase repo)

- In `src/sase/llm_provider/commit_finalizer_raw_commits.py`, add `"linked"` to
  `_RAW_COMMIT_REPO_KINDS`. Update the module docstring, which lists the in-scope kinds.
  Update the backstop docstring in
  `src/sase/finalizers/commit_validation.py::raise_if_unpublished_local_commits` too.
- Absorbed linked repos already flow through dirty discovery as `external` obligations,
  keyed by path, and render as `linked:<name>` in `absorbed_raw_commits`. Confirm both
  in a test; no other kind sets need changing.
- Tests go beside the existing ones in
  `tests/llm_provider/_commit_finalizer_raw_commits_{detection,absorb,backstop}_tests.py`.
  Build the baseline record exactly as `sase repo open` writes it: `kind="linked"`,
  `scope="opened_repo"`, `repo_id="linked:<name>"`. Preferably create it through
  `capture_opened_repo_dirty_baseline(..., kind="linked")` so future record-kind drift
  is caught. Assert three things:
  - detection reports the unattributed raw commit;
  - absorption soft-resets it to staged dirt and pins
    `refs/sase/recovery/raw-commits/<run>`;
  - the backstop raises `unpublished_local_commits` when absorption is ineligible, for
    example on a detached HEAD.
- Add a drift guard: every value in `typing.get_args(OpenedRepoKind)` must be in
  `_RAW_COMMIT_REPO_KINDS`.

## Part 3: Fix Gap B, bind the run-start baseline to the checkout the agent actually uses

Add one best-effort helper to `src/sase/llm_provider/commit_finalizer_baseline.py`, for
example
`bind_run_start_baseline_to_workspace(project_dir, artifacts_dir, *, inherited)`. Like
`capture_dirty_baseline`, it never raises; it logs and returns.

- Take the baseline lock, the same one `capture_opened_repo_dirty_baseline` uses. If a
  `run_start` record with `kind == "main"` already has the normalized `project_dir` as
  its path, do nothing. Otherwise rebuild the `run_start` records from
  `collect_baseline_repositories(project_dir)`:
  - **Not inherited:** use fresh fingerprints and the current `head`, exactly as
    `capture_dirty_baseline` does.
  - **Inherited:** set `fingerprints` to `{}` and use the current `head`. This keeps the
    lane-inheritance contract (plan `202608/lane_baseline_inheritance.md`): a parent's
    still-uncommitted work is this run's to commit, not protected foreign dirt.
- Drop every stale `run_start` record whose path is not in the fresh set. That includes
  the primary checkout and `sibling` records for the user's primary clones. Keep
  `opened_repo` records whose path lies inside `project_dir`, and drop the others. Never
  scan or absorb in a checkout this run does not own.
- No-op when `project_dir` is not a git checkout.

Call sites:

1. **B1 and B3.** In `src/sase/axe/run_agent_runner_launch.py`, call the helper once
   after `_prepare_workspace_and_repos(state, bootstrap)` returns and before the
   provider runs. Skip home mode. Pass `resolve_finalizer_project_dir()`, the same
   resolver the finalizer uses at turn end, so capture and finalize agree. Calling it
   after preparation means the recorded `head` already includes any commits that launch
   prep kept from an earlier occupant. Workspace #14's rebased `5f97b514` is an example.
   Those commits are not this run's and must not be absorbed into it.

   Plumb whether bootstrap inherited the baseline: make
   `_capture_commit_finalizer_baseline` return that fact and store it on
   `RunnerBootstrap`.

2. **B2.** Where an in-process follow-up gets its artifacts dir
   (`create_followup_artifacts` and its caller for the plan-approved coder), seed it if
   it has no `finalizer_baseline.json`: copy the previous run's file, the way
   `_inherit_parent_commit_finalizer_baseline` does. Then call the helper with
   `inherited=True` against the follow-up's workspace.

Tests:

- **B1.** Extend `tests/test_axe_run_agent_runner_deferred_workspace_flow.py` or
  `tests/test_run_agent_runner_bootstrap_baseline.py`. A baseline captured in a
  placeholder repo A, followed by a claim of workspace B, must end with exactly one
  `main` record at B, carrying B's post-prepare `head`, and no `run_start` record under
  A.
- **No-op.** A baseline whose `main` record already matches is not touched.
  Fingerprints, head, and `captured_at` stay unchanged.
- **B3.** An inherited baseline whose `main` points elsewhere becomes a B `main` record
  with `{}` fingerprints. `opened_repo` records inside B are kept, and those outside are
  dropped.
- **B2.** A plan-approved coder's follow-up artifacts dir ends up with a `main` record
  for its workspace.
- **Regression.** Reproduce the incident end to end. Start from a stale baseline, make a
  raw commit in workspace B's main repo and in a linked clone opened under B, then bind.
  `unrecorded_local_commits` must report both, and `absorb_raw_local_commits` must
  absorb both.

## Part 4: Skill wording (decision `monitor_next_wording`)

> [!decision] monitor_next_wording = no Leave the skill text alone; skip this part.

> [!decision] monitor_next_wording Make the edit below.

In `src/sase/macros/skills/sase_monitor.md`, next to the existing bullet that says
"`--reason` and `--next` text reaches the follow-up literally", add one bullet. When the
follow-up should finish by committing, write "submit `/sase_final` with a commit
decision for <repo>: <message>". Never write "commit <repo>": a follow-up on another
model may run raw `git commit`. Preview with `sase skill init --diff`. Do **not** deploy
to chezmoi from this unlanded tree; the generated-skills rule requires landing first.
Read the `generated_skills` reference memory before editing. Mention in the final report
that a post-landing `sase skill init` deploys it.

## Verification and finish

- In the sase repo, run `just check` through `sase tool run check`, which the guarded
  recipe requires. Read the `lint_and_test` reference memory first. Do not run
  `just check-full`.
- Run the Part 1 verifications listed above.
- Submit `/sase_final` with one `commit` decision per dirty repository:
  - sase:
    `fix(finalizer): absorb raw commits in linked repos and bind baselines to the claimed workspace`
  - bob-mac-capture and bob-cli: the messages from Part 1.
- In the final report, state:
  - the host-assigned SHAs that landed upstream for bob-mac-capture and bob-cli, or why
    a step was skipped;
  - that the host fixes take effect only after a human reinstalls sase (`just install`
    is human-only);
  - that session `an.f0.w0`'s pending plan was written against a bob-mac-capture tree
    without the Refs alias, so its coder should rebase onto the restored commit.

## Out of scope

- The rescue-then-evict design of launch-time preparation.
- A hook that blocks raw `git commit` in non-Claude runtimes. The host absorber is the
  designed safety net (decisions `host-owned-completion` and
  `adapters-normalize-harnesses`).
- Workspace #14's stranded local commit `5f97b514`. Its next rebase drops it once 1b
  lands an identical patch.
