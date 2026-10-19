---
tier: tale
title: Finish the test repairs and verify the v0.18.0 public release
goal:
  The observed test failures are repaired and verified, sase 0.18.0 is published through
  the existing release automation, its distributions match its immutable release tag,
  and clean Linux and macOS installations resolve the current plugin family without
  falling back to obsolete versions.
size: medium
proposed_by: bbugyi200.athena.11n
proposed_with: codex/gpt-6-astra
create_time: 2026-10-19 05:32:53
status: wip
---

# Plan: Finish the test repairs and verify the v0.18.0 public release

## Scope and execution ownership

The user explicitly requests fixing the failing tests and releasing **v0.18.0**. This
plan completes that outcome using the repair and release machinery already running. One
follow-up coding agent owns the remaining work across monitor continuations. A medium
tale is appropriate because the multi-area repairs already have an approved epic and
workers; the new implementation identified here is a bounded test-fixture lifetime
repair. Creating another repair epic would duplicate active ownership.

Start by reconciling live state. Reuse completed repairs, preserve concurrent changes,
and do not edit files being repaired by an active worker. The existing epic `sase-1mk.9`
and its ancestors keep their own landing and bead lifecycle. This tale supplies any
uncovered repair and the additional public-release proof. It must not finish merely
because another agent is working or a watcher stopped. Use `/sase_monitor` for external
waits and long verification, with an explicit continuation that resumes the next
unfinished step of this plan.

The requested release operations are authorized. Keep `ci_watch` as the sole release-PR
merger and one release watcher as the workflow dispatcher. Do not bypass failing gates
or introduce a second concurrent publisher. If a live owner exists, observe it and
perform independent work. Transfer responsibility only after checking both its terminal
result and whether its lander started a successor.

No broad release-train rewrite, merge queue, retention change, memory edit, global
install, or ruleset change is needed for this tale. The research's E1–E6 program remains
separate. Preserve release-please's ownership of versions, changelogs, dependency
metadata, and release tags.

## Context and evidence at planning time

Read these through the audited artifact commands when resuming:

- `research:202610/pypi_release_reliability_track0_remaining_work.md`
- `research:202610/pypi_release_reliability_design_and_work_split/pypi_release_reliability_design_and_work_split.md`
- `plan:202610/ship_v0_18_0_post_repair_reds.md`
- `sase bead read sase-1mk.9 -r "Reconcile active release recovery ownership"`
- `sase bead read sase-1mk.9.3 -r "Check the remaining link-follow repair"`
- `sase bead read sase-1mk.9.6 -r "Check the active release driver"`
- `sase bead read sase-1kf -r "Read the scoped-test startup override leak"`

The report's statement that watcher `sase-1mk.8` is still driving publication is stale:
it closed **canceled**, `MASTER_GATE_RED`, at 2026-10-18 19:29:42Z. Its replacement is
phase `sase-1mk.9.6`, which waits for the nested epic's repairs. As of approximately
22:00Z, `sase-1mk.9.3` was still verifying and the replacement watcher and lander were
waiting. Re-read these facts; they will change.

| Observed blocker                 | Evidence and existing repair                                                                                                                                                                                                              |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Generated research README drift  | Full CI `39155173153` failed `init repo --check`, proposing removal of five lines. Task `sase-1mm` is already owned by phase `.9.1`, closed done with research commit `43e9066b`; `TAGS.md` and report tags were preserved.               |
| JUMP panel screenshot drift      | The same run failed `agents_jump_panel_collapsed_in_flight_mix_120x40.png`. Phase `.9.2` landed `b68b5166ad`, updating the intended footer-wrap change.                                                                                   |
| New Project owner-switch failure | All Python 3.12/3.13/3.14 legs failed `test_owner_switch_preserves_per_owner_drafts`, with missing form controls during remount. Phase `.9.4` landed `7f3c8d0965`, resetting/rearming readiness and guarding remount-time refresh.        |
| Link-follow kind correction      | Phase `.9.3` owns the pending-query inventory-miss race in beads/plans panes. Verify its eventual commit and evidence instead of reproducing its patch.                                                                                   |
| Triage/provider-drain failures   | Phase `.9.5` landed `e5c02dc8a4`, hardening triage and completion-file handling.                                                                                                                                                          |
| Scoped test-state leak           | `.9.3` reported ToolRun `26eed4f2318f79530d975401953ef079`: 2,022 passed, with `test_ace_page_restores_fast_startup_overrides_after_success` as the sole NEW failure. Existing task `sase-1kf` records earlier independent reproductions. |

Full CI `39164596460`, started on `457dd728d5`, already had red lint, visual, and Python
3.13 jobs while other legs were running. It predates the later repairs above and cannot
qualify them. Master Gate `39168441419` passed on `7f3c8d0965`; the newer `e5c02dc8a4`
run was still in progress. Require fresh evidence covering the integrated fixes.

Release PR [#299](https://github.com/sase-org/sase/pull/299), head
`3c371426659ddd6bc58e7f5984c6d1214eb80dc0`, was CLEAN with a passing
`release-core-floor-smoke`. Its source-built matrix is intentionally skipped; that PR's
green rollup is not Full CI proof.

Live PyPI JSON at planning time reported `sase 0.17.1`, `sase-core-rs 0.42.0`,
`sase-github 0.2.24` requiring `sase>=0.18.0`, and `sase-telegram 0.4.29`. Core 0.42.0
has its sdist and four wheels, including universal2 macOS and both Linux architectures.
The source checkout still has the old core window; the release branch's metadata
reconciliation owns its update. Inspect that branch locally before relying on its exact
dependency range.

Commit `457dd728d5` already fixes the normal Publish build: it exports release-please's
SHA, requires it when a release was created, checks out that commit, records the built
SHA, and reads the smoke-test core floor from the same commit. Preserve this fix.
`publish_existing=true` still builds the dispatched ref, so recovery must dispatch the
immutable release tag, not a moving master.

## 1. Reconcile the current state and prepare the checkout

1. Read applicable skills and memory: `lint_and_test`, `sase_beads`, `ace`, `tui`, and
   the relevant TUI child notes before changing startup/rendering code. Open other
   repositories with `/sase_repo`; read sidecar documents with `sase artifact read`.
   Shared backend changes, if actually necessary, belong in `sase-core` and require the
   corresponding binding/pin update.
2. Inspect `git status`, fetch origin, and use a clean, current checkout without
   overwriting another agent's edits. Inspect current agents/monitors, the beads above,
   PR #299, Master Gate, Full CI, Publish, and PyPI. Record actual SHAs and run IDs in a
   compact working evidence log under the workspace.
3. Verify the research README repair is in the opened sidecar and that
   `sase init repo --check` no longer reports drift. Do not repeat the old README edit
   or change its shared template if the repair already landed.
4. Identify which newer failures are already owned by an active worker. In particular,
   wait for `.9.3` before editing its link-follow test module. Confirm the worker's host
   finalization put its changes on origin; a closed phase alone does not prove that.
   Bounded waits must carry a monitor continuation.
5. If 0.18.0 is already published, skip all publication mutations and audit it in steps
   4–5. Continue repairing any still-reproducing test failure on current master; never
   move the published tag or overwrite public files.

## 2. Fix the uncovered startup-override leak and any residual failures

The leading hypothesis for `sase-1kf` is a concrete lifetime inversion:

- `src/sase/ace/testing/_startup.py` installs `_empty_artifacts_project_choices` with an
  `AsyncExitStack` patch, but only when the current function is the original.
- `tests/ace/tui/test_link_follow_bounded_panes.py::_open_plans` applies the
  function-scoped `monkeypatch` to `_collect_artifacts_project_choices` after its caller
  has entered `AcePage`.
- Page exit restores the real function. The pytest fixture then tears down and restores
  the value it captured earlier: the temporary empty function. The next test starts with
  that leaked override, causing the identity assertion to fail.

A stdlib-only patch-stack experiment reproduced this restoration mechanism during
planning. The actual ordered project-test reproduction is still required; this
experiment does not prove there are no other leak sources.

1. Recover the failing ToolRun's selected test order. Run the suspected producer
   `test_archived_plan_reveals_through_path_context_after_scan` followed by
   `tests/test_ace_testing.py::test_ace_page_restores_fast_startup_overrides_after_success`
   **in one serial pytest process**. Also reproduce the relevant module sequence from
   the scoped lane. An isolated pass or tests on different xdist workers is insufficient
   evidence for an order-dependent bug.
2. Correct the fixture lifetime at its owner. Prefer moving stable patches before
   `AcePage` entry, as neighboring tests already do. When patching during an active page
   is intentional, use a nested `monkeypatch.context()` that unwinds before page exit.
   Audit the other callers of `_open_plans` and other setters of this function for the
   same inversion. Do not reset production globals blindly in a suite-wide autouse
   fixture or weaken the restoration assertion.
3. Add a meaningful regression that exercises the real helper/page lifecycle, checks
   function identity after both contexts unwind, and enters another page successfully.
   Cover the exceptional cleanup path where it shares the same lifetime. Preserve
   caller-supplied startup fixtures. Modify the shared harness only if a failing
   lifecycle case proves that the defect belongs there.
4. Run the ordered reproducer and the related module group serially; repeat the failing
   sequence enough to establish that the regression is deterministic and fixed. Run the
   relevant existing isolation lane if the harness changed.
5. Re-read fresh CI failures after the active repairs land. Fix genuinely remaining
   blockers with targeted regression evidence. Preserve assertions, test coverage, pixel
   equality, and release gate semantics. Do not make a release green by skipping tests,
   raising timeouts without latency evidence, or rerunning a deterministic failure until
   it happens to pass.

Keep repair changes focused. Use existing task `sase-1kf` for evidence rather than
creating a duplicate. Respect the active epic's ownership of its existing tasks. If
evidence reveals a materially larger new implementation, hand off a revised plan through
`/sase_plan`; do not call the release complete or silently defer a blocking failure.

## 3. Qualify the integrated source and drive the existing release

1. For changed sase files, run `just fix`, then the required `sase tool run check`. Use
   the checkout's virtualenv. If the wrapper escalates, join the printed ToolRun with
   `/sase_monitor`; do not cancel it and start a duplicate. A full-suite escalation is
   not authority to bypass the scoped gate. Do not run `just check-full` for this plan.
2. Only run additional targeted visual checks when visual changes or unresolved drift
   justify them. Use the existing screenshot tooling and inspect manifests and images
   before accepting any changed golden; a partial update is not a pass. Relevant paths
   are the JUMP panel visual test, New Project owner-switch tests, link-follow
   planner/seam/bounded-pane tests, triage tests, and provider-drain tests. Reuse
   settled evidence for unchanged repairs.
3. Have the host finalize repair changes through `/sase_final`. Confirm the resulting
   commits are on origin before calling their SHA the fixed candidate. Use prepared
   completion or continuation support as documented by that skill; do not create
   commits, branches, or PRs directly.
4. Let the active `.9.6` watcher and `.9.land` finish their release work. Do not launch
   a competing watcher or dispatch duplicate Full CI runs. Use read-only preflight as
   needed:

   ```sh
   tools/release_watch --repo sase-org/sase --version 0.18.0 \
     --fixed-sha <integrated-fix-sha> --await-core-version 0.42.0 --once --dry-run
   ```

5. Verify the release prerequisites together: green Master Gate at the current master
   HEAD; the newest completed Full CI green and under six hours old; green Full CI
   covering the integrated fixes; and a clean release PR with completed floor checks and
   no conflicting Publish run. Inspect the required jobs, including all three Python
   legs, visual, isolation, lint, tutorials, and performance. The intentional PR-only
   skips are not matrix evidence.
6. If the existing watcher ends unsuccessfully, read its verdict and the lander's
   continuation before acting. Own uncovered repairs and requalify changed code. If no
   live successor exists and only release waiting remains, start exactly one bounded
   `/sase_monitor` for `tools/release_watch` with the new fixed SHA, a 12-hour watcher
   deadline and a slightly longer monitor timeout. Its continuation must inspect the
   verdict and resume this plan, including on failure. Do not use `--close-bead` to
   close somebody else's work.
7. Keep `ci_watch` as merger. If it stops ticking or rejects a demonstrably eligible
   candidate, investigate its explicit reason and service state. Read its
   repository/config through `/sase_repo` before diagnosing code. A required privilege
   or service-policy change must use the applicable skill, without weakening release
   policy or manually bypassing the merger.
8. After merge, require the normal Publish run's `release`, `build`, `install-smoke`,
   `install-smoke-core-floor`, and `publish` jobs to succeed. A successful metadata-only
   Publish run with skipped build/publish is not a release. If a transient upload
   failure needs recovery and no publisher is active, inspect any existing files first
   and dispatch at most one recovery using `--ref v0.18.0 -f publish_existing=true`,
   after verifying the tag, version metadata and workflow exist there. Never recover
   from arbitrary master. Resume only with matching existing hashes; conflicting bytes
   require explicit human resolution, not overwrite or retagging.

## 4. Prove the published files match v0.18.0

Do this even if the watcher reports RELEASED or another lander closes its epic.

1. Fetch the immutable `v0.18.0` tag into the authorized checkout and record its commit,
   the release-PR merge commit, the Publish run/attempt, the built SHA, and the
   source/core qualification runs. Verify repair ancestry and distinguish any test-only
   fixes that landed after an already completed publication.
2. Fetch `https://pypi.org/pypi/sase/0.18.0/json` directly. Require the expected wheel
   and sdist, non-yanked status, version 0.18.0, and the intended core dependency.
   Download both files into an isolated scratch directory and verify every SHA-256
   digest against PyPI metadata.
3. Compare the sdist `src/` and wheel `sase/` payloads to `git archive v0.18.0`. Compare
   project metadata, runtime version, and packaged resources as well. Explain only
   genuine build-generated differences; do not broadly ignore unexpected missing or
   changed files. Assert the Publish built SHA equals the tag commit and the wheel
   metadata agrees with the tagged release metadata.
4. Build a wheel from the downloaded sdist in isolation and smoke its installed entry
   points. Use public dependencies, not an editable checkout or the source-built core
   wheel used by normal development CI.
5. Record PyPI attestation/provenance. Track 0 explicitly accepts that the normal
   workflow attestation can name the trigger commit while the payload is built from the
   tagged commit. Record both identities and the verified byte match; do not claim the
   attestation has already been fixed to run at the tag.

## 5. Verify public installation on Linux and macOS

Use isolated environments and tool directories, with no checkout on the import path and
no local package source override. Keep scratch data under the authorized workspace and
run installed commands outside the source tree, checking installed module paths. Do not
change the user's global installation or credentials.

1. On Linux/Python 3.12, resolve **unpinned** `sase`, `sase-github`, and `sase-telegram`
   against `https://pypi.org/simple/` with a refreshed/no-cache resolver. Install that
   family into a fresh environment. Separately test `sase==0.18.0` if later versions
   have appeared since approval.
2. Require 0.18.0 for the requested-release install and no silent fallback to 0.1.0. At
   planning time the expected family is core 0.42.x, github 0.2.24, telegram 0.4.29.
   Revalidate newer plugin releases if present; record the exact resolved versions and
   require the tagged sase dependency constraints.
3. Run installed `sase version -j`, `sase core health --json`, CLI help, and plugin
   discovery. Check the installed wheel's required bindings and the targeted core
   contracts/public probes used by the release floor lane. Report any expected
   missing-provider diagnostic distinctly from a broken import or core contract.
4. Dispatch the existing `install-smoke.yml` once when no equivalent current run is
   active, and require a passing documented-install result. Monitor the run through
   completion; submission alone is not evidence.
5. For macOS, read `tailnet` memory and try the configured `mac` SSH alias with a
   bounded connection timeout. Use throwaway `UV_TOOL_DIR` and `UV_TOOL_BIN_DIR` for a
   public-PyPI family install and run version/core-health/plugin checks. Record OS,
   architecture, Python, resolved versions, outputs, and exit codes.
6. If that Mac is unavailable, use GitHub-hosted macOS rather than leaving this
   acceptance criterion as a manual chore. The bounded fallback is an additional
   **workflow-dispatch-only** family smoke job in the existing install-smoke workflow,
   on `macos-latest`, using Python 3.12 and isolated public installs. Preserve the
   existing documented Linux job and its single-sourced install command. Add focused
   workflow contract coverage alongside `tests/test_render_applicability.py`, run
   required checks, let the host land it, and dispatch once. This does not require the
   broader E1 canary program.

## Completion and durable evidence

Write and register `sase_v0_18_0_public_release_verification.md` with
`sase artifact create`, linking the research, active recovery plan, and existing
release-evidence artifact if available. Include:

- repaired failure nodes, their causal explanation, commits, ordered regression results,
  and the successful required local verification;
- the exact release/tag/build SHAs, PR/merger evidence, green gate run IDs and
  freshness, Publish job results, and watcher verdict;
- the sase/core public file inventory, hashes, tag comparisons, sdist-build result, and
  the documented attestation limitation;
- Linux and macOS public-install evidence and the resolved family versions;
- any unresolved external restriction stated explicitly, never marked passed.

Respect the existing lander's ownership of `sase-1mm`, its other repair tasks, and
superseding stale release epics. Supply evidence on the relevant existing records rather
than duplicating or force-closing them. Close `sase-1kf` only after the actual ordered
leak is repaired and verified and no active owner is still working it. Use
`/sase_new_task` before recording any genuinely new, nonblocking follow-up.

Success requires repaired observed failures, qualifying release evidence, published
0.18.0 wheel and sdist matching the tag, and successful Linux and macOS family installs.
A green PR, scheduled run, canceled watcher, skipped matrix, isolated flaky-test pass,
or PyPI listing by itself is insufficient. Continue through monitor handoffs until these
conditions are met or a concrete external block requires user action. Submit the normal
final declaration with `/sase_final` only after all authorized work for that turn is
accounted for.
