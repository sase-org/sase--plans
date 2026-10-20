---
tier: tale
size: medium
title: Finish landing the observe-only release steward (sase-1n6.7.1)
goal:
  Fix the steward defects found at landing, turn Master Gate green again, prove two
  scheduled observe-only ticks, and close sase-1n6.7.1 and its parent phase sase-1n6.7.
proposed_by: bbugyi200.athena.sase-1n6.7.1.land
proposed_with: claude/opus
bead: sase-1n6.7.1
create_time: 2026-10-20 02:05:19
status: wip
---

- **PARENT:**
  [202610/release_steward_observe.md](https://github.com/sase-org/sase--plans/blob/main/202610/release_steward_observe.md)
- **BEAD:**
  [sase-1n6.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n6/sase-1n6.7.1.md)

# Finish landing the observe-only release steward

You are finishing the landing of child epic **sase-1n6.7.1** ("Stateless release steward
with an observe-only workflow", plan `plan:202610/release_steward_observe.md`), whose
parent is phase bead **sase-1n6.7** of the outer epic **sase-1n6**. All five phases
(sase-1n6.7.1.1 to .5) are closed and their code is on `master`:

- the Rust policy `release_steward_decide` in sase-core
  `crates/sase_core/src/release_steward/`, at pin `dd817d56` in
  `sase-core-revision.txt`;
- `tools/_release_watch_{decide,transport,facts,health,actions,sinks}.py`;
- the rewritten `tools/_release_watch.py` CLI;
- `.github/workflows/release-steward.yml`.

The land agent verified the following and recorded it on the beads:

- A live read-only `tools/release_watch --once --json` works: `core_lag`, exit 0, about
  2m40s.
- `sase bead epic-symbols` is empty for sase-1n6.7.1 and sase-1n6.7.
- The candidate-evaluator contract has not drifted.
- Follow-up triage is done; see the `LAND TRIAGE` note on sase-1n6.7.1.
- The `HANDOVER` note on sase-1n6.9 holds the handover evidence.

**Do not redo any of that.**

Four epic-caused problems remain. Finish them, then complete the closeout in **this same
turn**.

Do not wait for, or depend on, this work's own commit, push, or CI. The host commits
after your turn, and when both repos change it commits sase-core first and moves
`sase-core-revision.txt` automatically. Never create commits, branches, PRs or tags
yourself.

**Ground rules:**

- Open sase-core with `/sase_repo` (`sase repo open sase-core`), use only the printed
  path, and read its `AGENTS.md`. Never run bare `cargo`.
- Shared policy belongs in Rust: there is no Python policy fallback.
- Do not edit `release.yml`, `publish.yml`, `ci.yml`, `build-core.yml`,
  `install-smoke.yml`, memory notes, repository variables or rulesets.
- Keep `release-steward.yml` observe-only.

## 1. Master Gate is red because of this epic (sase repo)

Master Gate has failed at every commit since `112a5f4757`; check
`gh run list -w "Master Gate"`. Two of its causes are this epic's:

1. **toobig:** `tests/test_release_watch_health.py` is 1253 lines, over the 1000-line
   limit.
   - Split it by subject into two or three files, each under 850 lines if practical. For
     example, keep core-ensure, core facts and cuts in the original file, and move
     canary, issues, access-probe and snapshot-composition tests into
     `tests/test_release_watch_health_canary.py` and/or
     `tests/test_release_watch_health_snapshot.py`.
   - Move shared builders into a private helper module under `tests/`, following the
     repo's `tests/_*.py` convention.
   - Keep every test; the count must not drop.
   - Confirm with `just toobig`, which `just check` does not run.
2. **Stale contract manifest:** since `812cdd9e21`,
   `tests/test_github_actions_release_steward.py` carries
   `pytestmark = pytest.mark.contract` but `tests/contract_manifest.txt` was never
   refreshed. `test_contract_manifest_matches_marker_selection` fails, with 85 selected
   against the 84-entry budget.
   - Keep the marker: a workflow YAML guard is admitted on the same basis as every other
     `test_github_actions_*.py` guard.
   - Run `just refresh-contract-manifest`.
   - Raise `_MANIFEST_ENTRY_BUDGET` in `tests/test_contract_manifest.py` to the new
     count. Update `_MEASURED_SERIAL_COST` and append a dated "Re-curated to N" comment
     paragraph in the existing style, naming the file, why it is admitted, and its
     measured standalone cost.
   - Re-read the manifest state at the moment you edit it: sase-1n6.5.1.5.2 is
     re-curating the same file concurrently.

## 2. The Master Gate check is conflated with the PR checks (sase-core)

`checks.rs::master_gate_green` checks the **Master Gate run's jobs** against
`master.expected_required_checks`, but that set holds the **release PR's** required
checks. Once the PR set is sourced correctly (step 3), it contains
`Conventional PR title`, which no Master Gate job carries, so the gate could never be
green.

Change `master_gate_green` to stop using the PR expected set. Master Gate is green at
HEAD only when all of these hold:

- `gate_run` exists and its `sha` equals `master.sha`;
- its status is `Completed` with conclusion `Success`;
- it has at least one job;
- every job's conclusion is exactly `Success`. Skipped, neutral, cancelled, stale and
  missing are not green. `master-gate.yml` has no conditional jobs, so a green run never
  legitimately skips.

`pr_checks_green` keeps using `master.expected_required_checks` and
`expected_checks_readable`.

Update the Rust fixtures in `tests/fixtures.rs` and their tests. Today the gate jobs and
the PR checks share the names `tests` and `lint`; give them distinct, realistic names so
the tests prove the separation. Add tests for:

- a green gate run whose jobs differ from the PR set;
- a successful run that contains a skipped job (not green);
- an empty job list (not green).

## 3. Expected PR checks are sourced only from legacy branch protection (sase)

`tools/_release_watch_facts.py::_required_checks` (about lines 463-487) reads only
`branches/master/protection/required_status_checks`, and any exception counts as
"unreadable". Live on sase-org/sase:

- that endpoint returns **404 "Branch not protected"**;
- the only ruleset, `master-integrity`, has just the `deletion` and `non_fast_forward`
  rules (`gh api repos/sase-org/sase/rules/branches/master`).

So the expected set is always unreadable and the merge path can never be reached. The
child plan requires the expected set to come from "authoritative requirements **and
applicable workflow contracts**".

Replace it with a union of three sources:

1. **Legacy branch protection.** A 404 / "Branch not protected" is a _readable empty_
   result. Any other transport failure (401/403/5xx/timeout) makes the whole set
   unreadable: return `[]` and `False`.
2. **Active rulesets:** `GET repos/{repo}/rules/branches/master`. It is a list, so read
   every page. Collect `parameters.required_status_checks[].context` from each rule of
   type `required_status_checks`. A transport failure makes the set unreadable; a
   malformed shape raises `ReleaseFactsError`, as the legacy parser does today.
3. **The release-PR workflow contract.** A module constant naming the checks every
   release-please PR must pass:
   - `release-core-floor-smoke`, the `ci.yml` job that runs only when the head ref is
     `release-please--branches--master`. `ci.yml` says "Job name stays
     release-core-floor-smoke: consumers parse it";
   - `Conventional PR title`, the `pr-title.yml` job.

   Every other CI job is deliberately _skipped_ on release PRs, which you can see in the
   check rollups of #299 and #321. Do not require them, because skipped never counts as
   green.

Return the union sorted by name, each entry as `{"name", "source"}` where the source is
`branch protection`, `ruleset` or `workflow contract`. Set the master facts'
`expected_checks_source` (hard-coded today to `"branch protection"` in `_master_facts`)
to the sorted, comma-joined list of sources that contributed. The wire shape does not
change.

Add tests:

- in the facts-reader tests: 404 means readable-none, 403 means unreadable, a ruleset
  `required_status_checks` rule is parsed, a malformed ruleset raises, and the union and
  ordering are correct with the workflow-contract entries always present;
- in contract-marked `tests/test_github_actions_release_steward.py` (or the existing
  ci/pr-title workflow contract tests): `ci.yml` defines job `release-core-floor-smoke`
  gated on `release-please--branches--master`, and `pr-title.yml`'s job is named
  `Conventional PR title`. This way a rename breaks a test instead of silently disabling
  merges.

## 4. A publishable candidate whose retained assets are gone is still dispatched (sase-core)

Proposed by sase-1n6.7.1.2. In `policy.rs::handle_active_train`,
`retained_asset_missing` and `workflow_broken_at_tag` count only inside
`red_train_action`. A publishable (or not-yet-red) candidate whose draft-release dist
asset was deleted still gets a `release`/`resume` `DispatchRelease`, or waits forever.
Plan rule 5 says a missing retained asset or a workflow broken at its frozen tag **burns
an unpublished candidate**, and that public files mean preserve and escalate.

**Placement:** after the existing PublishConflict, DependencyUnpublished, veto, Released
and red-verdict branches, and before the PublishPartial, "public files await
confirmation" and initial-release branches.

**New branch:** when `eval.retained_asset_missing || eval.workflow_broken_at_tag`:

- **A matching `release.yml` run at this tag/SHA is in flight:** return `Waiting`
  ("release.yml in flight; retained assets not final"). Take no abandonment action,
  because the assets may still be uploading.
- **Zero PyPI files** (`!version_has_pypi_files(train)`): return
  `Waiting { reason: "retained distribution asset missing; burning unpublished candidate" }`
  (or "release.yml missing at frozen tag; …"). Attach `abandon_action(eval)` and the
  candidate evidence.
- **Public files exist:** return
  `PublishConflict { candidate_id, detail: "retained distribution asset missing with public files present" }`
  (or the workflow-broken wording). It escalates immediately and forbids publication
  actions. Never abandon and never dispatch.

Add Rust state tests for each case: publishable plus missing asset with zero files is
abandoned; with public files it is PublishConflict with no action; with release.yml in
flight it is suppressed. Also cover the workflow-broken variants. Then rerun the Python
real-binding round-trip tests.

## 5. Verify

- In sase-core, from its printed checkout: `just fmt`, then `sase tool run check` (about
  5+ minutes; give it a long timeout or hand it to `/sase_monitor`).
- In sase:
  - rebuild this checkout's `sase_core_rs` from the edited linked sase-core so local
    tests use the new policy (`just install-venv` builds from the linked checkout);
  - run `just fmt` and `just toobig`;
  - run `sase tool run check`. Read the `lint_and_test` memory note first; do **not**
    run `check-full`.
- Run a live read-only `.venv/bin/python tools/release_watch --once --json` and keep its
  JSON. It must still be a typed state with exit 0. With the new expected-check
  sourcing, any merge-path waiting reason should no longer say "expected required checks
  unreadable".

## 6. Prove two scheduled observe-only ticks

`release-steward.yml` landed at about 05:29 UTC on 2026-10-20 (`812cdd9e21`). As of
05:58 UTC no run existed, because GitHub's first scheduled runs of a new workflow can
lag.

**List runs:** `gh run list -w "Release steward" --event schedule -L 10`.

**If fewer than two completed scheduled runs exist, wait with `/sase_monitor`** (for
example, poll every 10 minutes with a 12-hour deadline). The monitor's follow-up
continues from this step. Never end the turn promising to resume, and never substitute
manual dispatches or fixtures for scheduled proof.

**For two completed scheduled runs, check each one:**

- record the URL;
- confirm from `gh run view <id> --log` that the tick ran `--tick --json` **without**
  `--live`;
- confirm it printed `release_watch: <state>` and a typed JSON result in which no
  disposition is `executed`;
- confirm the step summary carries the `## Release steward` report (check the run page
  if the CLI cannot show it).

**Confirm no remote writes happened:**

- `gh issue list --label release-steward --state all` shows no steward-created issue;
- no `release.yml`, `publish.yml` or sase-core `release-plz.yml` dispatch was triggered
  by those ticks.

**If the scheduled runs fail:**

- _Caused by this epic's workflow or code:_ fix it here; it is epic work.
- _Caused by credential scope_ (`WATCHER_BROKEN` from the core Actions access probe or a
  permission error): do **not** close anything. Record the evidence with
  `sase bead note sase-1n6.7.1` and `sase bead note sase-1n6.9`, state the blocker in
  the final response, and stop.

## 7. Closeout of sase-1n6.7.1 (final step, same turn)

1. Append the scheduled-run evidence to the handover: `sase bead note sase-1n6.9` with
   the two scheduled-run URLs, their typed states, and whether the core Actions access
   probe passed with `SASE_RELEASE_TOKEN`.
2. Run `sase bead epic-symbols sase-1n6.7.1`. For each entry, resolve the symbol (wire
   it up, privatize it, add a non-test pragma, or delete it) or, only if a still-open
   bead needs the exemption, re-key its Justfile line to that bead.
3. `sase bead close sase-1n6.7.1 --note "<verification>"`. The note must cover:
   - all five phases verified against source and commits;
   - the fixes from steps 1-4;
   - the `check` ToolRun IDs for both repos;
   - the live `--once` result;
   - the two scheduled-run URLs;
   - a pointer to the LAND TRIAGE note.

   If the close is refused for leftover epic-symbols, clean them up and close again.
   Never use `--force` merely to make the close succeed.

4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read sase-1n6.7.1 -r "Need the plan path for closeout"`
   (`plan:202610/release_steward_observe.md`).
6. **Parent phase sase-1n6.7.** It is a phase bead, so close only it:
   - confirm the child plan delivered the phase's scope, using
     `sase bead read sase-1n6.7 -r "Verify the phase scope before closing"`. That scope
     is: a stateless reconciler with typed states; one idempotent action per tick;
     core-ensure; bounded reruns; abandonment; 24h escalation; observe-only; and the
     phase's "Done when" (a live `--once` typed state with evidence, two scheduled ticks
     with URLs, repo checks pass);
   - run `sase bead epic-symbols sase-1n6.7` and resolve any entries;
   - then `sase bead close sase-1n6.7 --note "<verification and scheduled-run URLs>"`;
   - run `just symvision`.

   Do **not** close `sase-1n6` or any other ancestor; its own land agent owns it.
