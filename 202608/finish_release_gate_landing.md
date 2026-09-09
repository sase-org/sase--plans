---
tier: epic
status: done
title: Complete the interrupted sase-um.9 release-gate landing
goal:
  Meet the remaining live gate criteria, ship SASE v0.17.0 through ci_watch, and publish
  a SASE-0.17-compatible bugyi-chops 0.9.0.
parent_bead: sase-um.9
phases:
  - id: chopcolor
    title: Make bugyi-chops parse gh JSON without host-only environment overrides
    depends_on: []
    size: medium
    description:
      "chopcolor: harden the GitHub command environment, verify ci_watch against source
      SASE, and stage the exact revision for live use without publishing it yet."
  - id: gatebudget
    title:
      Bring successful Master Gate runs and the trailing median inside eight minutes
    depends_on: []
    size: medium
    description:
      "gatebudget: measure warm successful eight-shard runs and apply only the next
      proven critical-path lever needed to meet the live reliability, wall-time, and
      job-minute bounds."
  - id: fullgreen
    title: Drive Full CI green on the final integrated SASE tip
    depends_on:
      - gatebudget
    size: medium
    description:
      "fullgreen: attribute the overlapping old-SHA failures, fix in-scope defects, and
      obtain a completed green Full CI run on the final post-gatebudget master tip."
  - id: ship
    title: Let ci_watch merge and publish SASE v0.17.0, then remeasure acceptance
    depends_on:
      - chopcolor
      - fullgreen
    size: medium
    description:
      "ship: install the hardened chop revision, exercise its guarded live merge of PR
      #284, publish v0.17.0, and record all seven acceptance measurements."
  - id: choppublish
    title: Ratchet and publish bugyi-chops 0.9.0 against released SASE v0.17.0
    depends_on:
      - ship
    size: medium
    description:
      "choppublish: update bugyi-chops to the released SASE 0.17 dependency window,
      prove a clean public-index install, publish 0.9.0, and verify the released wheel
      live."
proposed_by: bbugyi200.athena.sase-um.9.land
bead_id: sase-um.9.5
create_time: 2026-09-09 19:50:22
---

- **PROMPT:**
  [prompts/202608/finish_release_gate_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_release_gate_landing.md)
- **PARENT:** [202608/release_gate_completion.md](release_gate_completion.md)
- **BEAD:**
  [sase-um.9.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.9.5.md)

# Plan: Complete the interrupted release-gate landing

## 1. Why another child epic is required

The four phases of parent bead `sase-um.9` are closed, but the closures are not proof
that the epic is complete. Phase `sase-um.9.4` was auto-closed by the mid-flight commit
`fa74163b5` even though its preceding note explicitly said the release still needed a
green Master Gate, a green Full CI, a guarded merge, publication, and final
remeasurement. This plan contains only that unfinished work and integration discovered
by the land audit.

The following implementation is verified and must be preserved:

- bugyi-chops commit `c3d613d` accepts flat or per-repository `merge_method`,
  `gating_workflows`, `heavy_workflows`, and `heavy_max_age_hours`, applies
  built-in/default/repository precedence, and performs a GitHub repository-metadata
  preflight that reports `merge_method_not_allowed` before attempting a merge. Chezmoi
  commit `ec5e82fb` selects merge + Master Gate + Full CI for `sase-org/sase` and
  squash + empty allowlists for the two plugin repositories.
- SASE commit `ed74b9f7b` repairs the deterministic visual fixtures; the visual job in
  later Full CI runs is green. SASE commit `69d3d7190` raises Master Gate to eight
  shards and adds the Full-CI-fed `shard-timings-ratchet.yml` freshness path.
- Post-epic drift is integrated. The generated-memory failures reported by `sase-um.9.2`
  were repaired by later memory-init commits. Commits `45a0a8880`, `84263159f`, and
  `0235ff059` moved the Python/core workspace contract; `fa74163b5` ratcheted
  `sase-core-revision.txt` to the corresponding v0.32.14 core and replaced Models-panel
  single-pause races with `wait_for_snapshot_idle`. A focused 88-test suite covering the
  gate workflows, shard refresher, shard assignment, and Models panel passes on
  `fa74163b5`.
- Chezmoi commit `aec90fe2` supplies `GH_FORCE_TTY=0`, `NO_COLOR=1`, and `CLICOLOR=0` to
  this host's `ci_watch`, so the live chop can operate while bugyi-chops is repaired.
  That host-only workaround does not protect any other installation.

Measured at 2026-08-28 20:05 EDT on SASE `fa74163b5`:

- Master Gate run 33221794673 is green, but took 13.50 minutes on a core-cache miss. The
  trailing 50 runs are 33 success / 16 failure / 1 cancelled with a 9.17-minute median,
  above the required 8 minutes. The eight-shard history is not yet a durable green
  sample and successful samples remain over budget.
- Full CI run 33212832198 is red on the Models-panel snapshot race repaired by
  `fa74163b5`. Scheduled run 33216659649 is still running on older SHA `affc43a6f` and
  has failures in the 3.12 coverage leg, coverage-contexts, and contention-test. No
  completed green Full CI includes the integrated tip.
- PR #284 is OPEN / MERGEABLE / CLEAN. Tag `v0.17.0` is absent and PyPI's newest SASE is
  0.16.0.
- bugyi-chops declares version 0.9.0 but still depends on `sase>=0.16.0,<0.17.0`. A
  clean `just check` environment resolves SASE 0.16.0 and fails during collection
  because `sase.feature_flags` and `PromptDirectives.if_code` are newer contracts. PyPI
  has no bugyi-chops 0.9.0.
- bugyi-chops `run_command` inherits the ambient environment unchanged. With gh 2.98 and
  `CLICOLOR=1`, its JSON adapters fail closed unless the host config supplies the three
  color overrides above.

## 2. Phase `chopcolor`: remove the host-only JSON parsing dependency

Work in `bbugyi200/bugyi-chops`, opened through `/sase_repo`.

Make subprocess execution deterministic for every `gh` JSON read. Preserve the rest of
the caller's environment, but ensure `GH_FORCE_TTY=0`, `NO_COLOR=1`, and `CLICOLOR=0`
are present where `GitHubReader` invokes gh. Prefer scoping this to the GitHub adapter
if doing so keeps the `CommandRunner` protocol simple; a generic runner change is also
acceptable if tests prove it does not erase inherited environment variables. Add a
regression that starts from a color-forcing ambient environment and observes the exact
environment used by gh. Keep the chezmoi override until the released package is
installed; it is a safe compatibility belt during rollout.

Do not tag or publish 0.9.0 in this phase. SASE 0.17.0 does not exist on PyPI yet, so
the truthful dependency ratchet cannot produce a normal clean lock/install until phase
`ship` succeeds. Verify the ci_watch suite and the repository's lint/build gates against
the source SASE environment, commit the fix, install that exact git revision into the
live source venv, and prove a dry-run tick parses repository, PR, and workflow JSON. The
unrelated `sase_chop_tg_inbound` / `sase_chop_tg_outbound` doctor errors are tracked by
task `sase-ve`; do not weaken doctor or fold that task into this epic.

## 3. Phase `gatebudget`: satisfy the live eight-minute gate contract

Start from the eight-shard implementation already on master. Measure individual job
durations on cache-hit and cache-miss runs instead of treating early failures as fast
samples. Dispatch or observe samples at least 10 minutes apart over an hour, as the
parent plan requires, and calculate both the success majority and trailing-50 wall-time
median from run timestamps.

If warm, successful eight-shard runs are already below eight minutes and enough new
samples bring the trailing median to the target, make no speculative code change. If the
successful critical path remains over budget, take the next measured lever from the
approved parent plan: make the non-visual fast suite collect without Pillow and switch
Master Gate away from `install-visual`. The proposal in `sase-um.9.3` note #1 is folded
here conditionally rather than filed as a separate feature because the epic's timing
criterion is still unmet. If that does not remove the measured bottleneck, revisit the
`needs: core-wheel` serialization while preserving the pinned-core binding check and the
<=60 job-minute-per-commit ceiling; do not optimize unrelated Full CI cost (task
`sase-v8` owns that).

Acceptance requires a majority-green set sampled over the hour, a trailing-50 median
wall <=8.00 minutes, no cancellation-by-new-push behavior, and <=60 job-minutes per
commit. Run the focused workflow/shard tests and the repository's required `just check`
after any edit.

## 4. Phase `fullgreen`: prove the exhaustive lane on the integrated tip

Wait for run 33216659649 to finish and inspect every failed job log. Separate failures
that `fa74163b5` already fixes from fresh deterministic defects and genuine fail/pass
flakes. Phase workers do not create tasks: record any unrelated fail/pass case as a
`PROPOSED FOLLOW-UP:` note with the exact node, failed run, unchanged-tree rerun, and
existing-task match when known.

After all `gatebudget` changes land, dispatch `full.yml` on the resulting master tip.
Drive every lane green rather than accepting a green visual subset or a run on an older
SHA. Fix epic-caused or deterministic failures in scope, rerun focused nodes under the
same Python/coverage/contention lane that failed, and dispatch again after fixes land.
Use `/sase_monitor` for Full CI and `just check-full`; neither belongs in an inline
agent command. Exit only when one completed Full CI run is green on the final integrated
tip and remains inside ci_watch's six-hour heavy-lane freshness window.

## 5. Phase `ship`: exercise the guarded release path

Ensure the live source environment contains the exact bugyi-chops `chopcolor` revision
and the chezmoi per-repository mapping. Run a dry-run tick first. Then watch normal
five-minute live ticks until `sase-org/sase` reaches `eligible`; do not hand-merge PR
#284. The live `gh pr merge --merge --match-head-commit` performed by ci_watch is the
acceptance evidence for merge-strategy correctness.

Once PR #284 merges, let `publish.yml` run. Use its existing `workflow_dispatch` /
`publish_existing` path only if the three-hour schedule is the sole delay. Confirm the
`v0.17.0` tag, GitHub publication workflow, and PyPI 0.17.0 artifact.

Record all seven parent acceptance criteria numerically in the phase note: cancelled
count in the last 50 Master Gate runs with attribution; trailing-50 median wall; share
of master commits with a completed gate in 24 hours; daily non-default ci_watch reason
and at least one `eligible`; guarded release merge; PR CI queue p50; and tag/PyPI
publication. Re-check both plugin repositories' effective squash + empty-allowlist
settings and confirm no recurrence of `gating_workflow_missing` or
`heavy_lane_not_green` there.

## 6. Phase `choppublish`: publish the compatible chop after SASE exists

Return to `bbugyi200/bugyi-chops`. Change its dependency window to require the newly
published SASE 0.17 contract and exclude the next incompatible minor (normally
`sase>=0.17.0,<0.18.0`), regenerate the lock from public indexes, and create a fresh
environment. `just check` must now pass without borrowing the source SASE venv; this
specifically proves `sase.feature_flags` and `PromptDirectives.if_code` resolve from the
declared dependency.

Publish the already-declared bugyi-chops 0.9.0 through its tag-driven workflow, confirm
the PyPI artifact, install the released wheel in the live environment, and rerun the
ci_watch dry-run against all three release repositories. Keep or remove the chezmoi
color override based on whether it remains useful defense-in-depth, but the package's
own regression must prove hosts without that config are safe. Record the exact tag,
workflow run, PyPI version, installed version, and dry-run release reasons.

## 7. Follow-up dispositions to preserve for final landing

- `sase-um.9.1` note #1 reproduced after `just install`; no duplicate or causally
  responsible active epic existed. It became ready bug task `sase-ve`, linked back to
  the proposing phase.
- `sase-um.9.3` note #2 exactly duplicates ready flake `sase-qr`; the two named Master
  Gate runs were recorded as its ninth independent `+1`.
- `sase-um.9.2` note #1 is already resolved by the later generated-memory/provider-shim
  initialization commits in chezmoi and the SASE memory-template work; no task.
- `sase-um.9.3` note #1 is conditional `gatebudget` work while the <=8-minute criterion
  remains red, not a detached feature task.
- `sase-um.9.1` note #2 and `sase-um.9.4` note #2 are epic-caused blockers handled by
  `chopcolor` and `choppublish`, not follow-up tasks.

The parent `sase-um.9` land agent resumes after this child epic lands. Bead closing,
epic-symbol retirement, `just symvision`, and setting either linked plan's frontmatter
to `status: done` are deliberately not child phases.
