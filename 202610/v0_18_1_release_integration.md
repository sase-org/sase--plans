---
tier: epic
title: Integrate the remaining release blockers and publish sase 0.18.1
parent_bead: sase-1nn.8.4.3
goal:
  Repair the newly diagnosed release gates, publish a core containing the current source
  pin, and confirm sase 0.18.1 on PyPI with clean-install evidence.
phases:
  - id: integrate
    title: Repair the current release gate integration failures
    size: medium
    depends_on: []
    description:
      "integrate: preserve recovery evidence outside tracked source, remove stale
      Symvision exemptions, and repair recurring UI readiness blockers without weakening
      checks."
  - id: core-cut
    title: Start a containing core release for the current source pin
    size: small
    depends_on: []
    description:
      "core-cut: re-inventory published core and active cuts, dispatch the guarded
      release-plz cut only when needed, and record its identity without waiting."
  - id: watch
    title: Observe the committed repairs through publication
    size: small
    depends_on:
      - integrate
      - core-cut
    description:
      "watch: run a bounded, self-closing monitor that respects the existing merge
      authority, handles source-pin drift, and records the actual release outcome."
proposed_by: bbugyi200.athena.sase-1nn.8.4.3.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-24 22:21:49
status: wip
---

- **PROMPT:**
  [prompts/202610/v0_18_1_release_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_1_release_integration.md)
- **PARENT:**
  [202610/v0_18_1_release_unblock.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_1_release_unblock.md)

# Finish the sase 0.18.1 release after the landing audit

This is remaining work from **sase-1nn.8.4.3**. That epic is still open because its
watcher closed canceled, not because its already repaired Beads golden needs another
rebaseline. The release target is **0.18.1** throughout.

An epic is necessary here because the release must contain the gate repairs and pass
remote CI after the host commits them. A repair agent cannot observe its own end-of-turn
commit or the subsequent publication in the same turn. The core cut is an independent
operational prerequisite; the watch follows both. No phase closes the parent or marks
the parent's plan done. This epic's parent link resumes that landing after this child
lands.

## Starting evidence

Audit at master `5497b59c5ddac82c20cf92bd0f63394a1f5e2d82`, 2026-10-25 UTC:

- Parent phases .1, .2, .3 are closed done. .4 is closed canceled with `GATE_RED`,
  Master Gate **40149338072**, attempt 2. PR **321** is OPEN at
  `64a9a6d98f7ca34ed317647fd6e3f5307a51f640`; PyPI sase/0.18.1 returns 404.
- `e788b1718a` supplies the monitor weight completion hint and CLI snapshot, waits for
  actual Archive Options, and adds an event-driven settle before pulse Enter.
  `AcePage.pause()` without an argument is an event barrier, not a sleep. `5e79f6405f`
  changes only the stranded Beads PNG. Its fixture pins time and waits for the selected
  stranded phase. The image has the intended stranded details. Full CI **40138926091**,
  attempt 2, visual job **121024210834**, passes.
- `87827788c2` subsequently committed 99 files under `.sase-evidence/` (17,317,023
  bytes), with 38 old terminology occurrences across 15 files.
  `tests/_macro_terminology_common.py::iter_scope_files` includes these files.
  `test_macro_resources_avoid_xprompt_terms` now fails Master Gate **40149338072**, job
  **121044125086**, all three Python legs of Full CI **40138926091** attempt 2, and
  PR321 floor smoke **40140631901**, job **121011445621**. That floor job successfully
  validates all **879** bindings in core **0.42.5** before failing the unrelated
  terminology audit.
- `5497b59c5d` adds `--epic-symbol 'sase-1o5.3(project_question_round)'`, although
  `src/sase/question_gate_turn/projection.py:315` already calls the facade. Master Gate
  lint job **121044124879** rejects that unnecessary exemption. The same commit advances
  `sase-core-revision.txt` to `3156d11f33d5a5277ff806a78b92a9490f78d6be`, which is after
  core v0.42.5.
- Core 0.42.5 has all five unyanked distributions and contains the old pin
  `7ec12d2d7ad8e670abe26aa396259a512f330391`. The original cut **40131163210** failed
  later; retry **40136208288** succeeded and the core release published at about 22:54
  UTC. New push run **40149137604** succeeded, but does not prove a containing release
  has been published.
- Full CI **40138926091** attempt 2 has additional UI failures: Python 3.12 job
  **121024209959** fails `test_epic_plan_modal_renders_no_raw_editor_for_capacity` with
  missing `#gate-singleton-0` during `GateBranchControls.on_mount`, and
  `test_files_ctrl_j_scrolls_page_anchor_to_top` on its 5-second live-anchor wait.
  Python 3.14 job **121024210005** fails `test_existing_typeahead_seeds_finder_query`
  with missing `#existing-finder-preview-body`. The last two already belong to
  **sase-1a7** and **sase-1mu**, which received corroboration. Do not presume one shared
  cause.
- Full CI **40151562730** was in progress at this audit. Refresh all evidence before
  editing. The later `b76183e938` free-text question fix preserves the built-in
  human-answer contract and is not a reason to restore auto answers.
- **sase-1n6.9** remains in progress: `ci_watch` still owns merges, and `publish.yml`
  still owns publication. The observe-only steward is not authorized to take over.

Read the parent and all four child notes, and audit-read
`plan:202610/v0_18_1_release_unblock.md`. Retained evidence:
`file:explicit:eca15519850917b7992759fe` (CI excerpts) and
`file:explicit:57fd79de8bb8e8659718a4cd` (full local visual check log).

The parent lander already triaged its proposals: image resize → **sase-1o6**;
selected-row PNG → **sase-1o7**; swarm panel → **sase-1o8**; collapsed clan tree →
**sase-1o9**; stranded resolve modal → **sase-1oa**. These are large flake tasks because
the causes remain unknown. Weighted-runner and maxcopy captures were recorded on
**sase-1jz**; pager history corroborated **sase-1f9**. Preserve this record and do not
duplicate it. Recovered capture warnings are infrastructure follow-ups; the strict
visual job has passed.

## Rules

1. Fetch and re-inventory before changing anything. Read your bead and the parent notes.
   Use `/sase_repo` for every other checkout and `sase artifact read` for plans and
   evidence. Do not use GitHub file-content APIs as a substitute for reading an opened
   checkout.
2. Read `lint_and_test.md`, `sase_beads.md`, and relevant reference memory through
   `/sase_memory_read`. For ACE changes read `ace.md` and `tui.md`; for Symvision read
   `symvision.md`; before changing tools read `tools.md`.
3. File-change verification is `just fix` followed by `sase tool run check` (the guarded
   `just check`). **Do not run `just check-full`.** A scoped check pass with a
   `just check-full` failure is a test-infrastructure issue, not remaining epic work.
   Remote release-policy gates still govern publication; fix diagnosed CI defects
   without changing or bypassing that policy.
4. Keep strict assertions and pixel equality. Fix semantic waits, fixture isolation, or
   actual lifecycle races. No raised timeouts, fixed sleeps, assertion retries, skips,
   or blind golden replacement. If rendering changes, use targeted screenshot
   verification and inspect the resulting goldens.
5. Shared domain changes belong in Rust; thin bindings and Textual state stay in their
   established layers. Do not revert the newer source pin or bindings to fit an older
   wheel. Do not manually alter release-plz versions/changelogs.
6. No hand commits, pushes, merges, new PRs, global installs, release.yml dispatches,
   live steward commands, or ci_watch configuration changes. `full_ci_rerun=yes`,
   `gate_rerun=yes`, and `publish_dispatch=yes` remain final.
7. Use `/sase_monitor` for long checks and operational waits. Never end a provider turn
   while a foreground command is still running. Phase workers record new
   `PROPOSED FOLLOW-UP:` notes rather than creating tasks. Resolve their own epic-symbol
   entries, close normally on success, and use `/sase_final`. An operational failure
   must be recorded as canceled, never as release success.

## Phase integrate

1. Re-read **sase-1nv**, **sase-1nv.5**, **sase-1o5**, and **sase-1o5.3**. Both active
   epics have landing-audit notes naming the concrete integration defects. Another agent
   may already have corrected either one.
2. Preserve the recovery evidence bytes before removing them from tracked source.
   Audit-read `file:explicit:0ad1354ad22ce3480a1a422c` and
   `research:202610/publication_recovery_acceptance.md`; verify coverage of all 99
   tracked `.sase-evidence/` paths. If the existing artifact is incomplete, create an
   immutable archive containing the files plus a path/size/SHA256 manifest, register it
   with `sase artifact create`, and verify its contents. Attach its ref to
   **sase-1nv.5** and this phase. Then remove only the proven archived scratch evidence
   from git. Search for live references first and update any reference that needs the
   durable artifact. Do not mutate historical evidence strings or broaden the
   terminology audit's ignore/allowlist.
3. Remove the unnecessary `project_question_round` exemption from the Justfile.
   Re-evaluate the other question-round entries against current consumers and remove
   only entries that are now stale. Still-open .3 is a legitimate key only for symbols
   that its later code actually needs. Run `just symvision`.
4. Inventory failed jobs in the latest five completed Master Gate/Full CI runs. The
   archive and autonomy fixes are verified; preserve them. Reproduce current
   deterministic failures and recurring UI failures with focused nodes and then an
   appropriate loaded batch. Inspect the gate-control mount/teardown sequence: the
   current test exits once the screen exists while nested controls can still be
   mounting; `on_mount` immediately queries every child button. Establish whether the
   test tears down too early or the product violates its lifecycle, and fix the actual
   cause. Apply the same evidence standard separately to **sase-1a7** anchor scrolling
   and **sase-1mu** finder preview mounting when the current inventory confirms
   recurrence. One-off failures remain notes on their existing owners. Do not turn the
   five newly filed visual/image tasks into an unbounded cleanup campaign.
5. Run focused terminology, question-round, and changed UI tests, including
   `tests/test_macro_terminology.py::test_macro_resources_avoid_xprompt_terms`. Run
   `just fix` and `sase tool run check`; retain exact results and failures. Confirm no
   new core binding was introduced without a containing source pin. Close this phase
   with evidence; the host makes the commit. The watch phase owns observing that commit
   remotely.

## Phase core-cut

1. Open sase-core with `/sase_repo`, read its instructions, and fetch both repos and
   core tags. Read the current sase master source pin locally. Query PyPI for the newest
   complete core, requiring the four supported wheel platforms and sdist, all unyanked.
   Use local `git merge-base --is-ancestor` to prove whether its tag contains the pin.
2. If already satisfied, record the version, tag, pin, and file set and close. Otherwise
   inspect recent release-plz runs and the current release PR. A cut already in progress
   or a scheduled cut within 90 minutes needs no duplicate. An eligible release PR must
   be open, mergeable, green, and contain the pin. Query its identity with gh and fetch
   its head into the opened local checkout.
3. When eligible and needed, dispatch exactly once:
   `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`.
   Record the resulting run ID and expected version. Do not assume it is 0.42.6 if
   release-plz selected another version. The workflow owns the merge.
4. This phase does not wait or edit code. Close with dispatch or already-satisfied
   evidence. If the core cut cannot safely start, record failed checks and close
   canceled so the lander can plan the exact remaining blocker.

## Phase watch

1. Confirm both predecessors closed done and the integrate commit is present on master.
   Read **sase-1n6.9** and verify PR321 still targets 0.18.1. If already published,
   record it and close. If a predecessor failed, the target changed, or release
   authority changed, close canceled with evidence for the lander.
2. Reuse the approved watcher algorithm in `plan:202610/v0_18_1_release_unblock.md`,
   with these diagnosed corrections:
   - Fetch and read local sase/core checkouts opened through `/sase_repo`; replace
     repository-content and compare API reads with local git reads/ancestry.
   - Key core-lag timers and dispatch budgets by the source pin. The previous watcher
     consumed its single retry on the old pin and could not react when 5497b59c5d
     introduced a new pin. For a changed pin, recheck the containing release, active
     cuts, and eligible release PR before authorizing a fresh cut. Permit at most one
     additional guarded cut per pin and never duplicate an in-flight cut. A pin stable
     but unpublished for 4h ends CORE_LAG.
   - Key Master Gate red duration by head/run. The previous script retained the red
     timer across new heads, producing GATE_RED only minutes after the new head's final
     failed attempt. Reset on head/run changes and success, and only count terminal
     failures. Keep the global 12h deadline.
   - Persist run IDs already rerun so a restart never grants more than one failed-job
     rerun per Master Gate or Full CI run. Only final attempts count toward two
     consecutive red Full CI runs. A transient API read failure is unknown, not success
     or a failed release.
3. Keep a read-only preflight mode and validate shell syntax and state transitions with
   recorded observations. Use a durable `/tmp/sase-release-watch-<bead>/` directory
   rather than agent-managed temporary storage. Retain its script and log as artifacts
   before handoff when practical. No tracked source edits here.
4. Every 5 minutes observe PyPI, containing core, PR status/checks, exact-head Master
   Gate, latest Full CI and freshness, and Publish jobs. ci_watch requires exact-head
   green Master Gate, newest completed Full CI green and <=6h old, and all PR checks
   success/skipped. It dispatches Full CI when stale; do not introduce a parallel merge
   driver. Preserve the approved one-rerun policy. Once core is ready, dispatch
   `publish.yml` with `publish_existing=false` only when no Publish is in flight and
   metadata needs refresh, at most twice and at least one hour apart. After merge,
   permit one Publish dispatch if none has run since merge. Never treat a metadata-only
   Publish success as a release.
5. Use one bounded monitor, no model follow-up, with a self-closing wrapper. The normal
   watcher, signal trap, and outer timeout wrapper must all attempt idempotent bead
   closure: done only on actual PyPI 0.18.1, canceled otherwise with the terminal
   outcome/log. Retain bounded closure retries, run the epic-symbol guard before launch,
   and preserve the original outer-wrapper timeout guarantee. Wait for
   `sase monitor start` itself to exit. A host crash is recovered by
   `sase bead work <this-child-epic>`; never claim automatic recovery from an
   uncatchable kill.

## Landing and resuming the parent

The child lander must audit every phase and note, implementation commits, current
source, and subsequent master drift. A canceled watch is not publication evidence. If
repair remains, use `/sase_plan` for only that remainder; prefer a tale when it can
complete the landing without depending on its own future commit. Otherwise use the
necessary phased child. Keep a mechanically scheduled continuation.

Before declaring this child complete, verify and retain an evidence artifact with:

- PyPI sase 0.18.1 wheel/sdist filenames, SHA256 hashes, upload times, and dependency;
- GitHub v0.18.1 release/tag commit, PR321 merge commit/time/actor, qualifying Full CI
  run/SHA/attempt and Master Gate, and every Publish job conclusion;
- the exact released core floor, its five files, and tag ancestry;
- available watcher and ci_watch evidence, without inferring ci_watch authorship solely
  from the merge account;
- a fresh Python 3.12 installation of `sase==0.18.1` from PyPI with `UV_NO_CACHE=1`, in
  an isolated temporary directory outside the repo, isolated HOME supplied only to the
  child process, matching exact core dependency, and successful
  `sase core health --json`. Never alter the shell HOME or global installation.

Run `sase bead epic-symbols <this-child-epic>` and resolve or legitimately re-key every
entry, close this child normally, run `just symvision`, and mark its linked plan
`status: done` through the opened plans repo. These are lander actions, not another
phase. Never force a successful nested landing.

Then follow `parent_bead: sase-1nn.8.4.3`. Review its full audit and descendants,
re-read `plan:202610/v0_18_1_release_unblock.md`, and rerun descendant/linked-plan
readiness plus post-child drift checks. If a canceled watch blocks readiness, reopen
**sase-1nn.8.4.3.4** and close it normally with actual release evidence, preserving its
historical cancellation notes. Retire every `sase bead epic-symbols sase-1nn.8.4.3`
entry, then close:

`sase bead close sase-1nn.8.4.3 --note "<verification, follow-up dispositions, release artifact>"`

Run `just symvision` and set `status: done` in `plan:202610/v0_18_1_release_unblock.md`
using the plans checkout from `/sase_repo`. Continue only after full readiness rechecks
through the directly parented plans:

| Parent       | Linked plan                            | Canceled watch to re-evaluate if required |
| ------------ | -------------------------------------- | ----------------------------------------- |
| sase-1nn.8.4 | plan:202610/v0_18_1_final_goldens.md   | sase-1nn.8.4.2                            |
| sase-1nn.8   | plan:202610/v0_18_1_release_landing.md | sase-1nn.8.3                              |
| sase-1nn     | plan:202610/v0_18_1_release.md         | sase-1nn.7                                |

For each, read previous landing notes, all descendants/notes and linked plans, check
drift, retire epic-symbol entries, close normally with evidence, confirm Symvision, and
mark its plan done. Stop at the first incomplete or ambiguous ancestor, record its
blocker, and report it. Never force-close to bypass readiness. If an actual parent link
is instead a phase, close only that completed phase and leave its containing epic to the
waiting lander.
