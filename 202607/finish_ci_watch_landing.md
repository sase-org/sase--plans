---
tier: epic
title: Finish and land the ci_watch liveness epic
goal: Integrate the post-start SASE/core release changes, publish the audited ci_watch
  package, prove the live fixer path over a multi-day soak, and close sase-a4 with
  clean post-close validation.
phases:
- id: release-chain-integration
  title: Reconcile the SASE, Rust-core, and plugin release chain
  depends_on: []
  size: medium
  description: 'release-chain-integration: align the released SASE/core binding contract,
    recover all bead history, and verify a normal isolated plugin install.'
- id: package-publication
  title: Complete the bugyi-chops publication
  depends_on:
  - release-chain-integration
  size: small
  description: 'package-publication: repair least-privilege trusted publishing and
    verify clean published artifacts and live installation.'
- id: soak-live-fix-path
  title: Prove the live fix path over a multi-day soak
  depends_on:
  - package-publication
  size: medium
  description: 'soak-live-fix-path: audit at least 72 hours of live ticks and a genuine
    idle proposal opportunity without enabling merges.'
- id: land-sase-a4
  title: Close and validate the original epic
  depends_on:
  - release-chain-integration
  - package-publication
  - soak-live-fix-path
  size: small
  description: 'land-sase-a4: close sase-a4 normally, clean post-close Symvision findings,
    and mark the original plan done.'
parent_bead: sase-a4
create_time: 2026-07-27 14:58:14
status: wip
bead_id: sase-a4.4
---

- **PROMPT:** [202607/prompts/finish_ci_watch_landing.md](prompts/finish_ci_watch_landing.md)
- **PARENT:** [202607/ci_watch_liveness.md](https://github.com/sase-org/sase--plans/blob/main/202607/ci_watch_liveness.md)
- **BEAD:** [sase-a4.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-a4/sase-a4.4.md)

# Finish and land the ci_watch liveness epic

## Goal

Complete the work that the `sase-a4` landing audit found unfinished, integrate changes that landed after the epic
started, and close the epic only after the shipped package and live deployment satisfy the original plan.

The audit confirmed that commits `22d98e6` and `27ae836` in `bbugyi200/bugyi-chops` implement the intended terminal-job
classification, HEAD supersession, benign `no_ci`, SHA-independent debounce/dedupe, retained ledgers, truthful `no_op`,
fail-closed guards, and tests. Commit `2301932` bumps `bugyi-chops` to 0.3.1 and tag `v0.3.1` points to it. Commit
`150f9aff` in the `chezmoi` repository correctly retunes the athena lane and keeps `merge_enabled: false`.

Do not redo those completed changes unless revalidation exposes a concrete defect.

## Remaining evidence and constraints

- `bugyi-chops` release run `30294883603` built successfully but its PyPI job failed with `invalid-publisher`. The
  trusted-publisher claims are repository `bbugyi200/bugyi-chops`, workflow `.github/workflows/publish.yml`, and
  environment `pypi`.
- A normal isolated `just install` in `bugyi-chops` still cannot resolve `sase>=0.12.0,<0.13.0`, because only SASE
  0.11.1 is currently available from the configured package index.
- SASE release PR `#243` contains the 0.12.0 release commit but remains open. Changes committed after `sase-a4` started
  added bead-history bindings in `sase-core`, and `sase-core-rs` 0.12.1 is released, while the SASE base branch still
  declares `sase-core-rs>=0.11.3,<0.12.0`. Consequently the live editable SASE tool has `sase-core-rs 0.11.4`, and
  `sase bead history` fails because `bead_history` is absent.
- The fixed lumberjack began producing live results at 2026-07-27 14:39:27 America/New_York. Five audited ticks through
  14:55:07 showed `errors=0`, `no_ci=1`, `sase-org/sase` as red instead of permanently pending, distinct decision
  ledgers, `no_op`, and correct suppression while agents were live. This is a smoke test, not the original multi-day
  soak, and it has not yet produced a live fix proposal because every mature-red tick had live agents.
- Keep `merge_enabled: false` throughout this follow-up. Do not put credentials or tokens in configuration, plans,
  prompts, evidence, logs, or command arguments.
- Use `sase repo open` before reading or modifying every sidecar, linked, different-project, or external repository.
  Follow each opened repository's instruction files. In the SASE repository, run `just install` before `just check`
  after source changes.

## Phase release-chain-integration

Dependencies: none.

Integrate the SASE and Rust-core changes that landed while `sase-a4` was running, before either SASE 0.12.0 or
`bugyi-chops` 0.3.1 is treated as publishable.

1. Re-open and revalidate current `master`/base-branch state for `sase-org/sase` and `sase-org/sase-core`, including
   SASE release PR `#243` and the published `sase-core-rs` 0.12.1 artifacts. Do not assume the versions observed by the
   landing audit are still current.
2. Update SASE's declared Rust-core compatibility floor/window and lock/test expectations so the Python
   `bead_history`/lost-note facade cannot install with a core wheel that lacks those bindings. Follow the existing
   core-floor bump convention and include a regression test that checks the required bindings against the declared
   minimum. Do not change Rust release versions manually.
3. Run `just install`, focused tests for the binding/version contract, and `just check` in SASE. Let the normal release
   process refresh PR `#243`; do not force, bypass CI, or manually merge outside the repository's release policy.
4. Confirm a compatible SASE 0.12.x and `sase-core-rs` 0.12.x pair is actually available from the configured package
   index. Then refresh the live uv tool through the supported SASE update/install path and verify
   `sase bead history sase-a4`, `sase-a4.1`, `sase-a4.2`, and `sase-a4.3` all work.
5. Inspect the recovered history, especially note revisions added or overwritten while the phases ran. If a note reveals
   an unaddressed requirement, reopen the affected phase or add narrowly scoped follow-up work; do not hide it by
   proceeding to landing.
6. From a clean `bugyi-chops` checkout, run the normal isolated `just install` and `just check` without
   `BUGYI_CHOPS_VENV_BIN`. This must pass against published dependencies.

Acceptance: the SASE/core released pair is coherent, bead history works in the live tool, all recovered `sase-a4` notes
are accounted for, the SASE checks pass, and `bugyi-chops` installs and passes its full checks without the temporary
shared-venv workaround.

## Phase package-publication

Dependencies: release-chain-integration.

Finish the package publication that phase `sase-a4.3` reported as complete even though the upload failed.

1. Verify the current PyPI and GitHub release state before changing anything. If 0.3.1 is already published and its
   artifacts match tag `v0.3.1`, record that evidence and do not publish a duplicate version.
2. Configure or coordinate the least-privilege PyPI trusted publisher matching repository `bbugyi200/bugyi-chops`,
   workflow `.github/workflows/publish.yml`, and environment `pypi`. Do not replace OIDC with a logged or
   repository-stored API token. If PyPI administration is unavailable, leave this phase open and report the exact user
   action required.
3. Once SASE's dependency is resolvable and the trusted publisher matches, rerun the failed publication workflow if
   GitHub permits a safe rerun of the existing tag. If PyPI or repository policy requires a new immutable artifact, use
   the next patch version and matching tag rather than moving `v0.3.1`.
4. Verify the published wheel and sdist metadata, hashes, `bugyi_chop_ci_watch` entry point, and dependency window.
   Install the published package into a fresh environment with published SASE/core dependencies and run a minimal
   entry-point/import smoke test.
5. Refresh the live SASE tool from the durable published/source configuration and verify the installed `bugyi-chops`
   source/version is the intended release without disturbing the compatible SASE/core pair.

Acceptance: a reproducible `bugyi-chops` release containing the `sase-a4` commits is available from PyPI, its published
metadata installs cleanly, the entry point exists, and the live tool remains dependency-consistent.

## Phase soak-live-fix-path

Dependencies: package-publication.

Complete the original multi-day soak and prove the fix half can actually fire. Do not close this phase merely because
the first few ticks classify correctly.

1. Keep the athena `ci_watch` lane running with `merge_enabled: false`, `red_debounce_ticks: 2`, the zero-agents gate,
   and the one-fix-per-tick cap. Accumulate at least 72 hours of live results beginning no earlier than the first fixed
   tick at 2026-07-27 14:39:27 America/New_York; if a later deployment changes behavior, restart the acceptance window
   from that deployment.
2. Analyze every completed tick in the acceptance window using the lumberjack log, result files, decision ledgers,
   proposed launches, notifications, and agent activity. Produce a compact durable report with counts and the exact
   window.
3. Confirm throughout the window that `sase-org/sase-nvim` is `no_ci` rather than an error, `sase-org/sase` reaches red
   or green rather than remaining structurally pending, each completed tick retains a distinct ledger, no-action ticks
   report `no_op`, merges remain zero, and no tick emits more than one fix proposal.
4. Observe at least one genuine mature-red/zero-agents opportunity where the live fix half emits a proposal, or, if no
   such opportunity occurs during the entire 72-hour window, continue the soak until one occurs. Verify the prompt is
   pinned and sanitized, the failing-job fingerprint and dedupe key remain SHA-independent, and launch-time idempotence
   prevents fixer churn as HEAD moves.
5. Correlate every emitted/suppressed proposal with job evidence and later HEAD results. Confirm no proposal was based
   only on a superseded cancelled run, green or fingerprint changes reset the streak, and live-agent ticks suppress
   proposals as `agents_busy`.
6. If any invariant fails, diagnose and fix it in the repository that owns the behavior, run that repository's full
   checks, redeploy through the durable path, and restart the 72-hour acceptance window. Do not enable merging as part
   of this plan.

Acceptance: the durable report covers at least 72 clean hours and a real live proposal opportunity, demonstrates
bounded/idempotent fixer behavior without superseded failures, and recommends either continued disabled operation or a
separate future merge-enablement decision. This phase does not itself enable merges.

## Phase land-sase-a4

Dependencies: release-chain-integration, package-publication, soak-live-fix-path.

Land the original epic only after every preceding acceptance criterion is met.

1. Re-run `sase bead show sase-a4` and each child. Confirm every child is closed with resolution `done`, every current
   and recovered note is addressed, and the original plan still points to `plans:202607/ci_watch_liveness.md`.
2. Recheck commits in SASE, `sase-core`, `bugyi-chops`, and `chezmoi` that landed since this follow-up began. Integrate
   any new direct overlap with `ci_watch`; do not broaden into unrelated cleanup.
3. Close the epic with `sase bead close sase-a4`. If closure is rejected, complete or deliberately reopen the named
   work. Use `--force --reason ... --resolution canceled|superseded` only for a genuinely canceled or superseded
   outcome, never just to make closure succeed.
4. After the close succeeds, run `just symvision` in SASE if the target exists. Remove only stale `sase-a4` epic-symbol
   whitelist entries and genuinely unused code it reports. Run `just install` and `just check` after any SASE source
   changes.
5. Open the plans sidecar through `sase repo open plans`, change only the original plan's frontmatter from `status: wip`
   to `status: done`, validate that the linked plan and bead now agree, and leave all involved worktrees clean through
   their normal commit/finalizer workflows.

Acceptance: `sase-a4` is closed normally, post-close Symvision is clean, all required repository checks pass, the
original plan is marked done, `merge_enabled` is still false, and the final report cites the release, soak, and landing
evidence.
