---
tier: epic
title: Publish sase-core and sase, then raise plugin floors
goal: 'A complete sase-core-rs release whose tag contains the commit in sase sase-core-revision.txt
  is on PyPI, sase master Master Gate is green and Full CI is fresh-green, ci_watch
  publishes sase against that floor, plugin floors that do not resolve are raised
  and published, and sase-10d plus phase bead sase-1i5.9 are closed done. Parent epic
  sase-1i5 stays open for its land agent.

  '
phases:
- id: core-release
  title: Publish a complete sase-core-rs release that contains sase's pin
  depends_on: []
  size: medium
  description: 'core-release: make sase-core master CI green by applying the recorded
    event-store doctor expectation fix, cut the release with the documented dry_run=false
    dispatch, and prove the published tag contains the pin and is complete on PyPI.'
- id: sase-gates
  title: Make sase Master Gate and a fresh Full CI green
  depends_on: []
  size: large
  description: 'sase-gates: inventory live Master Gate and Full CI failures, fix every
    deterministic red under the fix_master decision, and leave a green Master Gate
    on the tip plus a green Full CI inside the six-hour window.'
- id: publish-sase
  title: Ratchet the release branch and let ci_watch publish sase
  depends_on:
  - core-release
  - sase-gates
  size: medium
  description: 'publish-sase: dispatch publish.yml so sync-release-metadata ratchets
    the release-please branch onto the new core, prove release-core-floor-smoke and
    probe_core_floor, and let ci_watch merge and publish sase. Do not hand-merge.'
- id: plugin-floors
  title: Raise plugin floors, prove fresh installs, and close sase-10d
  depends_on:
  - publish-sase
  size: medium
  description: 'plugin-floors: raise sase-research-artifacts to the new core floor
    where it needs it, raise sase-telegram and sase-github sase floors where they
    do not resolve, prove fresh installs, and close sase-10d and sase-1i5.9 with evidence.'
proposed_by: bbugyi200.athena.sase-1i5.9
parent_bead: sase-1i5.9
create_time: 2026-10-08 14:37:01
status: wip
bead_id: sase-1i5.9.1
---

- **PROMPT:** [prompts/202610/release_sase_core_and_plugin_floors.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/release_sase_core_and_plugin_floors.md)
- **PARENT:** [202610/close_top_ten_impact_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/close_top_ten_impact_task_beads.md)
- **BEAD:** [sase-1i5.9.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1i5/sase-1i5.9.1.md)

# Plan: Publish sase-core and sase, then raise plugin floors

This is the implementation plan for phase bead `sase-1i5.9` of epic `sase-1i5`
(`plan:202610/close_top_ten_impact_task_beads.md`). That phase is large and told its
worker to plan against the release state at execution time. The state below was
re-checked on 2026-10-08 after the earlier phases had landed. Re-verify every fact
before acting on it. Do not re-ask this epic's decisions.

Parent epic `sase-1i5` stays open. Its land agent closes it. No phase here closes an
ancestor. Phase workers do not create task beads. Record discovered follow-up as
`sase bead note sase-1i5.9 'PROPOSED FOLLOW-UP: <summary — detail>'` so the parent land
agent can triage it. Notes on an existing bead this plan names (`sase-1h8`) are allowed.
Do not edit `sase/memory/**`.

Use `/sase_monitor` (`sase monitor start`) for every wait: CI, release-plz, publish
workflows, ci_watch merge ticks, and PyPI propagation. Do not end a turn to wait, and do
not use a provider-native background wait.

## Fixed decisions

Epic `sase-1i5` already settled these. Implement only the selected branch.

- `release_red_master = fix_master`. Inventory every red Master Gate and Full CI
  failure. Fix deterministic failures that no active epic owns. When an active epic owns
  a failure, check whether that epic has already fixed it. If not, apply the minimal fix
  and note it on the owner. Do not hand-merge the sase release PR past a red gate. Do
  not raise a bypass gate.
- `plugin_floors = yes`. Where the latest published `sase-telegram` and `sase-github` do
  not resolve to the new sase, raise their `sase` floor and let their release PRs
  publish. ci_watch merges them.

## What this epic may do

Approving the parent epic already authorized these outward actions, and this plan uses
only those plus the sase-core urgent-cut path documented in `sase-core`
`docs/pypi-retention.md`:

- merging sase-core's release-plz PR by dispatching `release-plz.yml` with
  `dry_run=false` (that job waits for checks and squash-merges; it is the supported
  urgent cut, and the parent acceptance says to merge the green release-plz PR)
- dispatching sase `publish.yml` with `publish_existing=false`
- letting ci_watch merge and publish the sase release
- committing plugin floor changes through the normal host-owned completion path so their
  release processes publish them

## Forbidden

- Deleting or yanking any PyPI release.
- release-plz `manual-version` recovery, and any hand edit of a version field,
  path-dependency version pin, or changelog. release-plz and release-please own those.
- `gh pr merge` of the sase-core release PR by hand. `gh pr merge --auto` does not wait
  for CI. The workflow merge job does.
- Hand-merging sase `#299` (or its successor) past a red gate, including a red Master
  Gate or a stale Full CI.
- Force pushes.
- Closing parent epic `sase-1i5` or any ancestor plan bead.
- Closing `sase-10d` as `canceled` or `superseded`.

A `just check` / `sase tool run check` failure that reproduces identically on the clean
base tree does not by itself keep a phase open: record a `PROPOSED FOLLOW-UP:` on
`sase-1i5.9` (cite any task bead that already tracks it) and continue. That exception
does not apply to Master Gate or Full CI. Those two must be green before ci_watch merges
sase.

## State re-verified 2026-10-08

sase `master` tip at planning time was `af117b598e`
(`feat(tui): enforce app import budget with closure tool and ratcheted cap`).
`sase-core-revision.txt` was `cd73d9687c3813915fc6db610236be9a6fd5eab6`. Declared floor
was still `sase-core-rs>=0.35.0,<0.36.0`.

sase-core `origin/master` was `1ff436055b12`. The pin is an ancestor of that master and
of release-plz PR head `efae9b8ad4c3` (parent `1ff436055b12`). The pin is not an
ancestor of tag `v0.37.0`. `f55c63b` is already in `v0.37.0`.

PyPI `sase-core-rs` `0.37.0` is complete: macos universal2, manylinux aarch64, manylinux
x86_64, win_amd64, and the sdist; none yanked. PyPI `sase` latest is `0.17.1` (two
files). A requirements file of only `sase` must not keep resolving to `0.1.0` after this
epic; that fallback is the live breakage `sase-10d` is repairing.

sase-core CI on `1ff436055b12` (run 37820216834) and on the pin (run 37804921735) fails
only in `event_store_supports_read_queries_without_legacy_projection`. Both ubuntu and
macos assert `bead_doctor(...).contains("WARNING: issues.jsonl missing")` at
`crates/sase_core/tests/bead_read_parity.rs` (the assertion is the only such expect in
the repo). `crates/sase_core/src/bead/read.rs` emits that warning only when the legacy
path is absent and no event store is present (sase-core `7df86f4a`, sase-1h8.11).
`sase-1h8` note #3 records the fix: assert the warning is absent for event stores. Do
not restore the warning.

Release-plz PR `sase-org/sase-core#323` (`chore: release v0.37.1`, branch
`release-plz-2026-10-06T18-16-25Z`) is open, mergeable, and based on current master. Its
CI is red for the same doctor assertion. sase-core cuts one release a day. Pushes update
the release PR and do not merge it. The daily cut is cron `41 7 * * *` UTC. An urgent
cut is:

```bash
gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false
```

`dry_run` defaults to true, and a dry run skips the PR update and the merge. The
dispatch updates the release PR, waits for its checks, and squash-merges. The merge push
tags and publishes. A complete release has every suffix in `EXPECTED_DIST_SUFFIXES` in
`release-plz.yml` (manylinux x86_64, manylinux aarch64, macos universal2, win_amd64,
sdist) and none yanked. Confirm with
`.github/scripts/pypi_release_files.py status <version>` from a sase-core checkout, or
the PyPI JSON. The JSON API is CDN-cached briefly after upload; re-query before calling
a heal failed. One urgent cut costs about 75 MB of the 10 GiB project quota. That cost
is accepted for this floor. Do not pass `build_wheels` / `publish_pypi` /
`expected_version`; those are the partial-heal flags and would also merge the open
release PR.

sase release-please PR `sase-org/sase#299` (`chore(master): release 0.18.0`, head
`release-please--branches--master`) is open. At planning time its only failing check was
`release-core-floor-smoke`. That is expected until a core release containing the pin is
published and `sync-release-metadata` ratchets the branch. The published window is owned
by that job in `.github/workflows/publish.yml`, not by a hand edit on master. Dispatch
is:

```bash
gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false
```

`publish_existing=true` skips the ratchet. Do not use it for this cut.

On athena, ci_watch (interval 300s) is the only auto-merger for release-please PRs.
`merge_enabled` is true. `release_repositories` are `sase-org/sase`,
`sase-org/sase-github`, and `sase-org/sase-telegram`. sase-core is not in that list.
Merge order is `sase-github`, then `sase-telegram`, then `sase`, one merge per tick.
sase's merge method is `merge`. sase gating workflow is `Master Gate`. Heavy workflow is
`Full CI` with `heavy_max_age_hours` 6. Both signals must be green. sase-github and
sase-telegram have empty gating and heavy lists. Do not hand-merge ahead of a repo
earlier in that order when it also has an eligible release PR; wait for the ticks.

sase Master Gate is red. The last completed gate before the import-budget push, run
37820031821 on `972df1c184`, failed `lint` (`_lint-symvision`) and these tests:

- `tests/ace/tui/models/test_agent_associated_plan_cache.py::test_frontmatter_cache_reuses_parse_until_mtime_changes`
- `tests/ace/tui/models/test_agent_associated_plan_cache.py::test_title_is_normalized_cached_and_invalidated_with_file_signature`
- `tests/ace/tui/test_agent_wait_epic_follow_tui.py::test_lane_following_narrates_epic_progress_and_since`
- `tests/ace/tui/test_agent_wait_epic_follow_tui.py::test_lane_launching_reads_pending_text`
- `tests/ace/tui/test_notification_plan_gate.py::test_plan_modal_bundle_loading_stays_off_the_message_pump`
- `tests/ace/tui/widgets/test_prompt_next_word_midword.py::test_deferred_midword_defers_then_applies`
- `tests/completion/test_snapshot.py::test_checked_in_snapshot_has_no_drift`
- `tests/completion/test_snapshot.py::test_current_structural_view_matches_checked_in_snapshot`
- `tests/instructions/test_verify_cli.py::test_verify_help_documents_flags`
- `tests/main/test_bead_fast_path.py::test_fast_path_guards_mutations_but_not_reads`
- `tests/main/test_bead_fast_path.py::test_fast_path_refuses_mutation_from_plain_checkout_sidecar_record`
- `tests/main/test_bead_fast_path.py::test_fast_path_refuses_unsafe_resolved_location_before_rust`
- `tests/main/test_completion_handler.py::test_candidates_handler_prints_provider_output`
- `tests/main/test_parser_command_help.py::test_memory_help_marks_primary_command_and_init_alias`
- `tests/test_bead/test_claimed_status.py::test_default_list_includes_claimed_with_shared_glyph`
- `tests/test_finalizers_discard_guard_before_head.py::test_post_dispatch_foreign_race_on_external_is_exempt`
- `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`
- `tests/test_plan_approval_modal_title.py::test_group_submit_uses_current_branch_selection`
- `tests/test_plan_gates_execution.py::test_shared_host_executor_handles_feedback_rejection_and_races`

Two clusters already have a recorded cause. The wait-lane tests assert `since 14:32`
while CI (UTC) renders `since 10:32`, a four-hour Eastern Daylight offset.
`test_default_list_includes_claimed_with_shared_glyph` is `sase-1h8` note #2:
`AttributeError: '_ReadView' object has no attribute 'list_issue_page'` from
`src/sase/bead/cli_query.py`. Epic `sase-1h8` is still in progress (`sase-1h8.13` and
`sase-1h8.14`). The plan-gate tests expect the literal `Launch coder agent` and the UI
renders `☑️ 🚀 Launch coder`. Treat the rest as unproven until this phase re-runs them.
The in-progress Master Gate on `af117b598e` (run 37823201800) had already failed lint
and several shards when this plan was written. Full CI has no green run inside the
window: run 37785681210 on `56fcf92447` failed `full / lint`, `full / visual-test`, and
`full / test` on 3.12, 3.13, and 3.14. Full CI is `workflow_dispatch` plus cron
`17 */2 * * *`.

Open release PR `#319` (`core-pin-ratchet` to `7b3b9aa`) is older than the pin already
on master. Do not merge it.

## Global working rules

- Open any repo other than the sase workspace with `sase repo open <name> -r "<why>"`
  and work only in the printed path. Read that repo's `AGENTS.md` before editing.
  sase-core agents run `sase tool run check`, not bare `just check`. Give that check a
  timeout of at least 10 minutes.
- In sase, read `sase memory read lint_and_test.md` before finishing a phase that
  changed tracked sase files, `symvision.md` before a symvision fix, and `sase_beads.md`
  before closing a bead. Use `sase memory read`, not a direct file read.
- Land code through the host-owned completion path. Do not hand-create commits,
  branches, or PRs.
- Never weaken an assertion, raise a timeout or cap, add a symvision pragma, or skip a
  test to get green.
- Re-read `sase-core-revision.txt` at the start of `core-release` and again before
  proving the tag. The published tag must contain that commit, not the planning-time
  pin, if the file has moved.

## Phase `core-release`

Runs in parallel with `sase-gates`. Repo: sase-core.

1. Re-read `sase-1h8` note #3 and the two sites above. Confirm the event-store test
   still expects the warning and that `read.rs` still suppresses it for event stores.
2. Change only that assertion so an event store does not require
   `WARNING: issues.jsonl missing`. Leave the production warning in place for legacy
   stores. Do not add a new warning, and do not edit versions or changelogs.
3. `sase bead note sase-1h8` that phase `sase-1i5.9` / this plan applied the recorded
   expectation fix, with the commit once the host lands it.
4. Run `sase tool run check` in the sase-core checkout. Land the fix through host
   completion.
5. With `/sase_monitor`, wait until sase-core CI on that master commit is green. Fix any
   new deterministic red on that commit before cutting. A red release-plz PR check is
   the same suite; do not dispatch the cut while it is red.
6. Confirm the open release-plz PR head still has the current `sase-core-revision.txt`
   commit as an ancestor (`git merge-base --is-ancestor`). The push of the fix runs
   release-plz and updates the PR. If master moved again, wait for that update. Do not
   `manual-version`.
7. Dispatch
   `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`. Monitor
   the `Merge release PR` job and the publish jobs.
8. Acceptance for this phase:
   - The new `sase-core-rs` version's git tag contains the pin commit.
   - `pypi_release_files.py status <version>` prints `complete` (all five suffixes, none
     yanked).
   - No version or changelog file was hand-edited.

## Phase `sase-gates`

This phase is large. Its worker plans first against the live red set, then implements.
The snapshot above is a starting map, not a whitelist. Runs in parallel with
`core-release`. Repo: sase.

1. Inventory Master Gate on the current master tip and the latest Full CI. Group
   failures by root cause. Re-read `sase-1h8` before touching bead read-model or
   fast-path code.
2. Apply `fix_master`:
   - Fix deterministic failures no active epic owns.
   - For a failure an active epic owns, use the fix that epic already recorded if it is
     not on master yet, and note the owner. `sase-1h8` note #2 is the known
     `list_issue_page` case. Do not reimplement that epic's unfinished design. The
     minimal fix that makes the failing test match the landed read-model behavior is
     enough.
   - The wait-lane `since 14:32` versus `since 10:32` mismatch is a fixed-clock
     assertion that does not survive UTC CI. Fix the assertion or the clock so the text
     is timezone-stable. Do not hardcode another local hour.
   - Plan-gate copy failures must follow the real button label, or the label must follow
     the test if the test is the contract. Read the test before choosing. Do not weaken
     it.
   - Symvision failures follow `symvision.md`: privatize, wire, or delete. Re-key a
     Justfile `--epic-symbol` line only onto a bead that will still be open.
     `sase bead epic-symbols sase-1i5.9` reported no entries at planning time. Do not
     leave a symbol keyed to a phase this epic closes.
3. Land fixes through host completion. Run `sase tool run check` in sase for each landed
   batch. A base-tree-identical local failure is a `PROPOSED FOLLOW-UP:` and does not
   replace a green Master Gate.
4. Wait until Master Gate on the tip is green.
5. Dispatch Full CI on that tip
   (`gh workflow run "Full CI" --repo sase-org/sase --ref master`) and wait until that
   run is green. The scheduled lane is every two hours and the last several scheduled
   runs are red, so a dispatch is required to enter the six-hour window. If the run goes
   red only on a deterministic failure the gate did not see (visual lane or an extra
   Python), fix it and dispatch again.
6. Acceptance: Master Gate is green on the master tip that publish-sase will release,
   and a green Full CI run for that tip is inside six hours. Record the run URLs on the
   phase bead.

## Phase `publish-sase`

Starts only after `core-release` and `sase-gates` are both done.

1. Re-read the pin and the published core version. Confirm that version is `complete`
   and its tag contains the pin. If the newest complete release does not contain the
   pin, stop and return to `core-release`. Do not ratchet onto `0.37.0` while the pin is
   past that tag.
2. Dispatch `publish.yml` with `publish_existing=false`. Monitor
   `sync-release-metadata`. It must commit the ratcheted `sase-core-rs` window and
   `uv.lock` onto `release-please--branches--master`, or exit cleanly because that
   metadata already matches.
3. The release PR's `release-core-floor-smoke` must pass. From a checkout of that
   branch, `tools/probe_core_floor` must report no missing capabilities against the new
   floor. If the smoke fails because the floor still does not contain the pin, the
   ratchet picked the wrong version; fix that before merge. Do not edit the floor by
   hand on master.
4. Keep Master Gate green on the tip and Full CI inside six hours while the release PR's
   own checks run. A new red on the tip goes back to `sase-gates` behavior: fix it, do
   not bypass.
5. Let ci_watch merge the release PR (method `merge`) and let `publish.yml` publish. One
   merge per five-minute tick, after any eligible `sase-github` and `sase-telegram`
   release PRs. Do not `gh pr merge` this PR.
6. Acceptance:
   - PyPI `sase` has the new version (planning-time PR says `0.18.0`; use whatever
     release-please actually cut), with its wheel and sdist, none yanked.
   - A fresh `uv pip compile` of a requirements file whose only requirement is `sase`
     resolves to that version, not `0.1.0`.
   - A clean venv `uv pip install 'sase==<new>'` then `sase core health` succeeds. Use a
     temporary venv, not the machine's existing uv tool install.
   - Record the version, the PyPI file list, and the compile output on the phase bead.

## Phase `plugin-floors`

Starts only after `publish-sase` acceptance.

Open each plugin with `sase repo open` and follow its `AGENTS.md` release process. Land
floor commits through host completion so the plugin's own release PR publishes them.
ci_watch merges `sase-github` and `sase-telegram` (squash, github before telegram).

1. `sase-research-artifacts`. Raise its `sase-core-rs` floor and its published smoke
   pin. The minimum is the first release containing `f55c63b` (already true of
   `v0.37.0`, which is `>= 0.35.0`). Move them to the new published core when the
   artifact package needs a capability that older complete releases lack. If the
   declared floor and smoke pin already resolve and already contain `f55c63b`, leave
   them and record that evidence. Publish when a raise is required.
2. `sase-telegram` and `sase-github`. `uv pip compile` each latest published
   distribution against the new sase. Where resolution does not select the new sase,
   raise that plugin's `sase` floor and let its release PR publish. Verify each plugin
   resolves to the new sase after publish. ci_watch merges these; do not hand-merge past
   a red check the plugin's own process treats as gating.
3. Close `sase-10d` with resolution `done`:
   `sase bead close sase-10d --note "<versions, PyPI file counts, uv pip compile outputs, phase sase-1i5.9>"`.
4. Run `sase bead epic-symbols sase-1i5.9`. If any `--epic-symbol` entries remain,
   resolve each symbol or re-key the Justfile line to a still-open bead (parent epic
   `sase-1i5` or a later phase) before closing. At planning time the command reported
   none.
5. Close only phase bead `sase-1i5.9`:
   `sase bead close sase-1i5.9 --note "<what was verified>"`. Do not close `sase-1i5`.

## Landing

The land agent of this plan does not close `sase-1i5`. Before it closes this plan's own
epic bead:

- `sase bead read sase-10d sase-1i5.9 -r "Confirm the release phase closed both beads"`
  shows both `closed` with resolution `done` and notes that cite versions, PyPI file
  counts, and `uv pip compile` output.
- PyPI still serves that sase and that complete `sase-core-rs`, and a fresh
  `uv pip compile` of `sase` still resolves to the published sase.
- `sase-1i5` is still open.
- Turn any `PROPOSED FOLLOW-UP:` notes on this plan's phase beads into notes on
  `sase-1i5.9` if they are not already there, so the parent land agent sees them. Do not
  file task beads.
