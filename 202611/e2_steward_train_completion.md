---
tier: epic
title: 'E2 completion: a steward that can merge and start a train, contained core
  pins, and the first steward release'
goal: 'The release steward can actually merge a release PR and start release.yml at
  the fresh tag. Every sealed candidate proves its core tag contains sase''s core
  source pin. One real sase release goes through the steward and is proven against
  its sealed manifest. The E2 decision record is written.

  '
parent_bead: sase-1n6
phases:
- id: merge-checks
  title: The steward reads a real merge gate from the N1 inventory
  size: medium
  depends_on: []
  description: 'merge-checks: the steward''s expected release-PR checks come from
    the versioned N1 inventory instead of absent branch protection, and Master Gate
    is judged on its own jobs rather than on the PR''s check list (sase-core plus
    sase).'
- id: containment
  title: Seal and enforce the core source-pin containment
  size: medium
  depends_on: []
  description: 'containment: release.yml seals sase-core-revision.txt as the core
    source pin with a containment proof for tag v<core>, seal and evaluate refuse
    without it, and the stale rust-backend doc sentence and ratchet test names are
    fixed.'
- id: train-start
  title: The steward starts and resumes a train at a fresh tag
  size: medium
  depends_on:
  - merge-checks
  - containment
  description: 'train-start: a fresh tag with an unsealed draft dispatches release.yml
    mode=release once instead of reading as WATCHER_BROKEN. A sealed, publishable
    generation with no PyPI files resumes. A fixture walk drives one train from releasable
    to released.'
- id: exit
  title: First steward release, acceptance evidence, and the E2 decision record
  size: medium
  depends_on:
  - train-start
  description: 'exit: confirm the human cutover, watch the first real release through
    the steward, prove digests, attestation and containment against the manifest,
    record acceptance evidence, and write the E2 decision record.'
decisions:
  memory_release_record:
    ask: Add decisions:exact-release-pair and mark ci-two-speed-split and rust-core-required
      superseded in part?
    default: true
    memory:
    - decisions:exact-release-pair
    - decisions:ci-two-speed-split
    - decisions:rust-core-required
    corrects: '`ci_watch` gates release on both signals: `gating_workflows` proves
      this commit''s fast gate is green, and `heavy_max_age_hours` requires a recent
      green `Full CI` run.'
    answer: true
proposed_by: bbugyi200.athena.sase-1n6.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-03 09:51:11
status: wip
bead_id: sase-1n6.11
---

- **PROMPT:** [prompts/202611/e2_steward_train_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/e2_steward_train_completion.md)
- **BEAD:** [sase-1n6.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n6/sase-1n6.11.md)

# Plan: E2 completion — a working steward train, contained core pins, and the first steward release

This epic finishes **sase-1n6** (E2, "Exact-candidate release train and steward", plan
`plan:202610/e2_release_train_steward.md`). All ten E2 phases are closed. Its land agent
found that the epic's exit was never achieved, and that E2's own code prevents it:

- No steward release exists. PyPI's latest `sase` is 0.18.1, a legacy upload.
- `sase-1n6.10` (exit) was closed without its work.
- The steward cannot merge a release PR, and it cannot start `release.yml` at a fresh
  tag.

Read the land agent's verification note on `sase-1n6` (the newest note) before starting
any phase:

```bash
sase bead read sase-1n6 -r "<why>"
```

The parent plan remains the specification. Its shared contract, boundaries and
acceptance mapping all still apply: missing evidence is red, human-only actions stay
human, agents never create PRs, branches, tags or commits, a workflow dispatched at a
tag runs that tag's files, and single sources stay single. Read it before working:

```bash
sase artifact read plan:202610/e2_release_train_steward.md "<why>"
```

## Where things stand (measured 2026-11-03)

- **Steward.** `release-steward.yml` runs live on schedule. Merge authority
  (`RELEASE_STEWARD_MERGE_ENABLED`) is unset. The newest verdict is `core_ci_red`. Issue
  #325 (`install_broken (macos)`) is the steward's correct escalation of a cancelled
  macOS canary job, an E1 surface; leave it alone.
- **Release PR.** #322 (`chore(master): release 0.19.0`) is open. Its checks are
  `prepare`, `Conventional PR title` and `release-core-floor-smoke` (currently FAILURE
  on a stale pin). Every other CI job on it is SKIPPED.
- **GitHub settings.** `branches/master/protection` returns 404 "Branch not protected".
  The only ruleset, `master-integrity`, has just `deletion` and `non_fast_forward`. The
  `pypi` environment still has both the `master` branch policy and the `v*` tag policy.
- **Related in-flight work. Do not duplicate it.**
  - `sase-1oe.4` (cutover) owns the green-rehearsal proof and the human gate that flips
    merge authority and removes the `pypi` master rule. Its gate was designed to
    relaunch `sase-1n6.10`, which is now closed: the E2 exit lives in this epic's `exit`
    phase.
  - `sase-1q2.9` owns clearing the blocking CI reds, the containing core publication,
    the master rehearsal, and a self-closing 48 h watch of v0.19.0. That watch cannot
    succeed until this epic's steward fixes land.
- **sase-core.** The steward workflow builds sase-core from source at
  `sase-core-revision.txt`, so a steward fix reaches the scheduled steward as soon as
  sase lands the moved pin. A sase **release** additionally needs a complete published
  `sase-core-rs` that contains that pin. The daily core cut supplies it, and so does the
  steward once merge authority is on (`cut_budget=daily`).

## Rules for every phase

1. Read the `tools`, `lint_and_test` and `docs_writing` memory notes before changing
   `tools/`, tracked files or `docs/`. Read `rust_core_backend_boundary` (always loaded)
   before splitting work between Python and Rust: steward **policy** lives in sase-core
   `crates/sase_core/src/release_steward/`; fact reading lives in sase's
   `tools/_release_watch_*.py`.
2. **sase-core changes:**
   - Open the repo with `sase repo open sase-core -r "<why>"`, work only in the printed
     path, and follow its `AGENTS.md`.
   - Verify with `sase tool run check` there.
   - A `release_steward` wire change follows the "Change a wire schema version" recipe:
     `RELEASE_STEWARD_WIRE_SCHEMA_VERSION` is copied by hand into
     `tools/_release_watch_decide.py`.
   - Move sase's `sase-core-revision.txt` past the core commit
     (`just ratchet-core-revision`, see `docs/rust_backend.md`). Both repos are commit
     obligations of the phase.
3. **sase changes:** verify with `just check` through `sase tool run check`. Never run
   `just check-full`. New contract tests join `tests/contract_manifest.txt` through
   `just refresh-contract-manifest`, within its budget rules.
4. **Docs.** `docs/contribute/release-train.md` is edited by `containment` (manifest and
   build sections) and `train-start` (the steward section, covering both steward fixes).
   No two parallel phases edit it. `merge-checks` documents itself in the inventory
   file, tool help and docstrings.
5. **Hand-offs.** Each code phase records, in a bead note, the landed contract that
   later phases depend on: names, schema versions, and the core commit and pin. Do not
   close `sase-1n6` or this epic from a phase.

## merge-checks

**Problem.**

- `tools/_release_watch_facts.py` `_required_checks()` takes the expected checks only
  from `branches/master/protection/required_status_checks`. That call 404s, so
  `expected_checks_readable` is always false.
- sase-core `release_steward/checks.rs` therefore always refuses the merge with
  "expected required checks unreadable".
- Worse, the policy feeds **one** list to both `master_gate_green` (the Master Gate
  jobs: `core-wheel`, `lint`, `docs`, `test (1)`…`test (8)`) and `pr_checks_green` (the
  release PR's checks). No single list can satisfy both.

**Work:**

1. **The source.** The N1 inventory (`.github/release-qualification.json`, validated by
   `tools/_release_qualification_inventory.py`) is the only list of release-blocking job
   names. Give it a merge-gate section that names the release PR's required checks: at
   least the pinned-core smoke `release-core-floor-smoke`, plus the PR checks that run
   on every release PR. Bump the inventory's schema version and update its validator,
   plus every reader that pins the version (the evaluator, metrics and the qualification
   tooling).
   - Before you commit the list, confirm each name against the real check rollup of
     release PR #322 (`gh pr view 322 --json statusCheckRollup`).
   - Keep SKIPPED and missing red. A name in the list that is SKIPPED on the PR blocks
     the merge, so list only checks that run on release PRs.
2. **The reader.**
   - The steward reads the merge-gate list from its own master checkout, the same
     revision it runs.
   - Record the list's source and the inventory digest in the master facts.
   - An unreadable or empty list still fails closed.
   - Branch protection, if a human ever configures it, may be honored as an additional
     source. Its absence is not "unreadable".
3. **The policy (sase-core).** Split the two judgments:
   - `master_gate_green` requires the Master Gate run at HEAD to be completed and
     successful, with at least one job and every job `success`. SKIPPED, cancelled and
     missing jobs are red. It is not compared against the PR's check list.
   - `pr_checks_green` requires every listed release-PR check to be present and
     `success` at the PR head.
   - Carry the PR list on the wire the policy reads (a separate field, or on the release
     PR facts), bump the wire schema version, and update the Python mirror.
4. **Tests:**
   - sase-core policy:
     - a releasable fixture whose list holds only PR checks merges;
     - a Master Gate with a skipped or failed job blocks;
     - a missing or SKIPPED listed PR check blocks;
     - an empty or unreadable list blocks.
   - sase reader: an inventory fixture produces the list with no branch-protection call
     succeeding, and a malformed inventory raises.
   - The existing `unreadable_expected_checks_cannot_merge` and empty-list tests keep
     their meaning.
5. **Live proof:** a read-only `tools/release_watch --once --json` from the landed tree
   reaches past the merge-gate checks. A `releasable` primary with the merge suppressed
   by authority off is the expected shape once core and PR facts allow it. Otherwise
   record the typed state that now stops it, which must not be "expected required checks
   unreadable".

**Done when:** both repos' checks pass, the pin moves past the core commit, the live
`--once` no longer reports the unreadable-checks wait, and a bead note records the
inventory schema version, the merge-gate names, the wire schema version and the core
commit.

## containment

**Problem.** This is parent DISCOVERED ISSUE #1, still open.

- `release.yml` line ~268 seals `--core-source-pin` from
  `grep -Eo 'sase-core-rs[^"]*' pyproject.toml`, which is the requirement string, not a
  source commit.
- `seal_manifest` accepts any string.
- `evaluate_manifest`'s `public_dependency_contract_satisfied` checks only a nonempty
  version and live files.
- So nothing proves that tag `v<core_version>` contains the source pin. The repro is
  `seal_kwargs(core_source_pin="not-a-source-commit")` in
  `tests/_release_candidate_tool_helpers.py`, which seals successfully and evaluates
  `ok=True`.

**Work:**

1. **`tools/release_candidate`:**
   - Add a subcommand that checks containment with the shared core-ensure helpers in
     `tools/_release_core.py`: the GitHub compare status `ahead` or `identical`, with
     the local-git fallback. Never reimplement it.
   - It writes a containment record (`pin`, `core_version`, the tag, the compare status,
     and the tag's peeled SHA).
   - `seal` takes that record, refuses unless the pin is a full 40-hex SHA and the
     record proves containment of exactly that pin by exactly the sealed core version,
     and seals the record. A refusal is typed so that the build log names the core lag,
     for example `CORE_LAG(pin=…, core=…)`.
   - Bump `SCHEMA_VERSION`, because the sealed key set changes. No release-mode manifest
     exists yet; only rehearsal artifacts are affected.
   - `verify` rechecks the record.
   - `evaluate`'s dependency clause requires the record and names it in its detail.
2. **`release.yml` build:**
   - Read `sase-core-revision.txt` as the pin.
   - Run the containment check against `steps.core.outputs.core_version` before sealing,
     and pass the record to `seal`.
   - A rehearse-mode `core_version` override must also contain the pin. The qualify
     job's `probe_core_version` path is separate and stays untouched.
3. **Tests:**
   - the repro above now refuses;
   - a non-containing compare status refuses;
   - a pin/version mismatch in the record refuses;
   - an unreadable compare raises;
   - `verify` catches a tampered record;
   - the workflow contract test asserts that the build reads `sase-core-revision.txt`
     and runs the check before `seal`.
   - Update the fixtures in `tests/_release_candidate_tool_helpers.py`, which still use
     a window string as `core_source_pin`.
4. **Leftovers from E2's core-pin and handover phases:**
   - `docs/contribute/rust-backend.md` "Who owns the published version window" still
     says the release lane includes "the pin-locked leg of `publish.yml`'s
     `install-smoke`", but handover deleted that job. Rewrite the sentence for the
     current lane: the release PR's pinned-core smoke, plus `release.yml`'s seal-time
     containment refusal and qualification. Keep the heading's anchor, because
     `docs/rust_backend.md` links it.
   - Rename `tests/test_ratchet_core_window_*.py` and
     `tests/_ratchet_core_window_tool_helpers.py` to the `ratchet_core_pin` name they
     test. Update imports, the contract-manifest entries (through
     `just refresh-contract-manifest`) and the path comments in
     `tests/test_contract_manifest.py`.
5. **Docs:** update `release-train.md`'s manifest schema and build description for the
   sealed containment record and the seal-time refusal.

**Done when:** `just check` passes. A local seal against live PyPI and GitHub, for the
current pin and the core-ensure pick, either seals with a containment record or refuses
with the typed lag. Record which one in a bead note.

## train-start

**Problem.**

- After the steward merges, `publish.yml` makes release-please create tag `vX.Y.Z` and
  an asset-less draft.
- `tools/_release_watch_facts.py` (around the `unsealed:` trains) turns a
  newer-than-baseline tag with no sealed manifest into a train with
  `manifest_mismatch: true`.
- sase-core `release_steward/train.rs` `select_trains` turns that into WATCHER_BROKEN
  ("tag/manifest mismatch"), so the steward never dispatches the first `release.yml`.
- `handle_active_train` dispatches `mode=release` only when `eval.publishable`, which
  needs evidence that only `release.yml` itself produces.
- Separately, a sealed, publishable generation with zero PyPI files is dispatched with
  `mode=release`, but the stash job refuses a second sealed generation without
  `rebuild: true`. That case must `resume`.

**Work:**

1. **Reader.** Emit a fresh, unsealed tag as its own fact, not as a manifest mismatch.
   The tag must be:
   - newer than the newest confirmed release;
   - peeled SHA equal to the merged release PR's head, where one exists;
   - backed by a draft release;
   - with no `candidate-manifest.json` asset;
   - with no `sase` file on PyPI for that version. Keep `manifest_mismatch` for real
     mismatches: a sealed manifest whose SHA or tag disagrees with the tag, or an
     unsealed tag that already has PyPI files.
2. **Policy (sase-core).** A fresh, unsealed tag is the active train:
   - With no `release.yml` run for that tag and SHA, propose exactly one
     `DispatchRelease { mode: "release" }`, suppressed while a matching run is in
     flight.
   - If a matching run completed without stashing a manifest, the workflow is broken at
     that tag. Burn the version through the existing abandonment path (write the
     observation, prefix the draft title), and never burn one with PyPI files. The next
     release then comes from release-please.
   - A sealed, publishable generation with no PyPI files proposes `mode: "resume"`.
   - Bump the wire schema if the facts change, and mirror it in Python.
3. **Fixture walk** (the proof that a train can finish):
   - Drive successive ticks over fact sets, each updated by the previous tick's action,
     from an open releasable PR with merge authority on, through merge, publish.yml
     metadata dispatch, tag plus draft, initial `release.yml`, sealed manifest with
     qualification and exercise evidence, publication and confirmation, to `Released`.
   - Assert at most one train action per tick, and that a repeated tick over unchanged
     facts takes none (acceptance case 10).
   - Put the walk in the sase-core policy tests and, through the Python reader, in the
     sase release_watch tests.
4. **Docs:** in `release-train.md`'s steward section, add the merge gate from
   `merge-checks` (where the list lives, and that Master Gate is judged on its own
   jobs), train start at a fresh tag, burned-at-tag, and resume.

**Done when:** both repos' checks pass, the pin moves past the core commit, a live
read-only `--once --json` from the landed tree reports a typed state, and a bead note
records the landed state names, the wire schema version, and the core commit and pin.
Then add a note on `sase-1oe.4` saying that the steward fixes have landed and that
enabling merge authority is now meaningful.

## exit

This phase replaces `sase-1n6.10`. It is the parent plan's `exit` section, updated for
the facts above. It ends with evidence and memory. It does not close `sase-1n6`: this
epic's land agent resumes that landing.

1. **Confirm the human cutover.** It is owned by `sase-1oe.4`'s gate. Check directly:
   - `gh variable list -R sase-org/sase` shows `RELEASE_STEWARD_MERGE_ENABLED=true`;
   - `gh api repos/sase-org/sase/environments/pypi/deployment-branch-policies` lists
     only the `v*` tag policy;
   - the applied `~/.config/sase/sase_athena.yml` keeps `sase-org/sase` out of
     `release_repositories` and the merge, gating and heavy maps.

   If anything is pending, hand a poll of those three checks to `/sase_monitor` with
   about a 7-day deadline. If `sase-1oe.4` is closed or canceled and no pending gate
   covers the missing steps, raise one `/sase_gate` as your last action that lists only
   the missing human steps. Remember that merge authority also lets the steward dispatch
   one core cut per UTC day.

2. **The first steward release.**
   - From a fresh clone of current `origin/master`, never a frozen workspace, hand
     `tools/release_watch --until-released --deadline 4d` (bare, with no version
     argument) to `/sase_monitor`. Read `sase-1q2.9`'s watch evidence instead of
     duplicating its monitor.
   - A burned candidate is expected; keep watching the next one.
   - If a typed blocked state persists past 24 h, triage it through the owner action
     that the state names.
   - If a new E2-caused defect blocks the train, fix it within this phase only when it
     is small, and otherwise record a `PROPOSED FOLLOW-UP:` note.

3. **Prove the release.** For that version:
   - every PyPI file's SHA-256 equals the sealed manifest's;
   - PyPI's Integrity API provenance names `refs/tags/v<version>`, a source digest equal
     to the tag SHA and the manifest SHA, and `release.yml` as the build config;
   - `master` had advanced past the release commit before the build (acceptance case 1);
   - the core file digests equal PyPI's;
   - the sealed containment record proves the tag contains the source pin;
   - no human acted beyond the pre-authorized steps.

4. **Single decider (acceptance case 10).**
   - The ci_watch run ledger shows no sase release merge after the apply.
   - During the train, a manual `release-steward` dispatch (`live=false` is enough to
     read the decision) takes no second dispatch or upload.

5. **Refusals.** List every refusal the steward reported during the soak, each with its
   typed reason and owner.

6. **Memory.** This is authorized only by this plan's accepted `memory_release_record`
   decision, which restates the parent's accepted one. Use `/sase_memory_write`:
   - add `decisions:exact-release-pair` with the claim, why, cost and reopen text from
     the parent plan's `memory_release_record` callout;
   - mark `decisions:ci-two-speed-split` and `decisions:rust-core-required`
     `metadata.status: superseded-in-part` with `superseded_by` pointing at the new
     strand and a `[[...]]` back-link that states what was retired (the ci_watch release
     gate, and the core-window cost clause, now an exact pin ratcheted at release time).
     Leave their bodies otherwise untouched.

   Then run `sase memory init`, and `just fix` to regenerate the design-decisions page.
   If the decision was declined, skip this step.

7. **Evidence.** Write one bead note on `sase-1n6` with:
   - the run URLs;
   - the manifest, attestation, containment and digest comparison;
   - post-release `tools/release_metrics` figures next to the `sase-1n6.8` baseline
     (note #4 there);
   - the parent plan's acceptance mapping.

**Done when:** a version newer than 0.18.1 is confirmed and un-drafted through the
steward, every proof above is recorded, and the authorized memory records are published
and checked per `lint_and_test`.

## Acceptance

- The steward merges a release PR using a merge gate it can read, and it starts
  `release.yml` at the fresh tag by itself.
- Every sealed candidate proves that its core tag contains the source pin.
- One real release went through the steward with no human action beyond the
  pre-authorized steps. Its PyPI digests, attestation SHA and containment record match
  the manifest.
- Every refusal named a typed reason and its owner.
- The E2 decision record exists.

## Non-goals

- Changing ci_watch code, `install-smoke.yml`, or E3-owned test steps.
- Configuring branch protection or rulesets. That is a human setting, and it would block
  the host's direct pushes to master.
- Fixing the current Full CI and Master Gate reds; `sase-1q2.9` owns them.
- Yanks, PyPI deletions and automatic repair agents.
