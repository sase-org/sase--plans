---
tier: epic
title:
  "Unblock E2 (sase-1n6): green exercise legs, a current steward verdict, a published
  core pin, and an executing handover gate"
goal: "sase-1n6.10 can watch and prove the first steward release. A master rehearsal of
  release.yml is green end to end, including both wheel exercise legs. Master pins a
  complete published sase-core-rs whose tag contains its source revision. A failed core
  cut that a newer green cut superseded no longer stops the steward. One human approval
  turns on merge authority, leaves the pypi environment tag-only, and relaunches
  sase-1n6.10 with corrected instructions.

  "
decisions:
  core_cut:
    ask:
      If no published core contains sase's core revision when core-pin starts, how
      should it get one published?
    choices:
      wait:
        Core-pin idles up to ~36h for the 07:41 UTC daily cut; sase-core keeps one
        release a day
      dispatch:
        Core-pin starts one release-plz cut (dry_run=false) now; PyPI gets an extra
        sase-core release
    default: wait
    why:
      The human gate paces the cutover anyway; sase-core reserves extra cuts for urgent
      fixes
    answer: wait
phases:
  - id: resolver
    title:
      release.yml finds the previous release in a throwaway venv and fails loudly on
      real errors
    depends_on: []
    size: medium
    description:
      "resolver: release.yml's previous-release resolver moves into
      tools/exercise_release, dry-runs each candidate in a throwaway uv venv instead of
      the externally managed --system interpreter, shows uv's stderr, and stops on
      errors that are not resolution failures. The SC2012 dist counts in the build step
      are replaced too."
  - id: cut-red
    title: A failed core cut stops counting as red once a newer cut succeeds
    depends_on: []
    size: small
    description:
      "cut-red: in sase-core, handle_no_train reports CoreCutRed only when the newest
      decisive completed cut is a typed failure. A later successful cut supersedes a
      stale red cut. Rust state tests and the steward state docs row are updated."
  - id: core-pin
    title: Master pins the published core that contains its source revision
    depends_on:
      - cut-red
    size: small
    description:
      "core-pin: once a complete published sase-core-rs (expected 0.42.7) contains
      sase-core-revision.txt, ratchet master's exact pin and uv.lock to it. If that tag
      also carries the cut-red fix, move the revision pin to the tag commit so the
      steward builds the fixed policy."
  - id: cutover
    title:
      Prove a green master rehearsal, then raise a handover gate that really executes
    depends_on:
      - resolver
      - core-pin
    size: medium
    description:
      "cutover: prove a green master rehearsal of release.yml and a steward verdict past
      every core check, write corrected handover notes on sase-1n6.10 and sase-1n6,
      close this phase, then raise one gate whose options flip merge authority and the
      tag-only pypi rule and relaunch sase-1n6.10 fresh."
proposed_by: bbugyi200.athena.14h
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 08:35:09
status: wip
---

- **PROMPT:**
  [prompts/202610/unblock_e2_release_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/unblock_e2_release_cutover.md)

# Plan: Unblock E2 (`sase-1n6`)

E2 (`sase-1n6`, "Exact-candidate release train and steward") has nine of ten phases
closed. The last phase, `sase-1n6.10`, must watch the first real release go through the
release steward and `release.yml`, then prove its digests and attestation. The epic's
lander, `sase-1n6.land`, waits on it. No such release can happen today. This epic clears
every blocker and then hands the cutover to one human approval.

Every phase agent should first read the diagnosis and the E2 design of record:

```bash
sase artifact read research:202610/sase_1n6_e2_blockers_20261026.md "<why>"
sase artifact read plan:202610/e2_release_train_steward.md "<why>"
sase bead read sase-1n6 sase-1n6.10 -r "<why>"
```

In the E2 plan, read "Shared contract", "Boundaries shared by every phase", "handover"
and "exit". Notes #9, #10 and #11 on `sase-1n6` are the discovered issues this epic
fixes.

## Where things stand (measured 2026-10-26, about 12:15 UTC)

- **Phase `.10`** is IN_PROGRESS. Its agent waits on monitor `9szd2935pyzr` (member
  `sase-1n6.10--mon`, label "WAITING CUTOVER"). The monitor polls every 300 s and exits
  only when three things are true: `RELEASE_STEWARD_MERGE_ENABLED` is `true`, the `pypi`
  environment has zero branch rules, and its tag rules are exactly `v*`. Its timeout is
  168 h, ending about 2026-11-01 21:52 UTC.
- **Handover gate `custom-69faba44-62d3-42fd-9129-27998a15b021`** (raised by `.9`) was
  answered in the TUI. Its options only print a status, though, so GitHub never changed.
  `gh api repos/sase-org/sase/actions/variables` shows `total_count: 0`. The `pypi`
  environment still has the `master` branch rule (id 51457521) next to the `v*` tag
  rule. Step 1 (chezmoi ci_watch apply on athena) is done.
- **The `.10` next action is wrong.** Once the cutover monitor exits, `.10` plans to run
  `tools/release_watch --until-released 0.18.1 --deadline 4d`. But 0.18.1 shipped
  through the legacy `publish.yml` path, and `tools/_release_watch_facts.py` already
  counts it as a confirmed `legacy:v0.18.1` train. The watch would return at once, and
  `.10` would find no `release.yml` manifest to prove. The E2 exit section says to use
  the bare `--until-released`.
- **Steward:** every scheduled `release-steward.yml` tick since 2026-10-24 21:56 UTC,
  through run 40314997000, returns the same primary verdict. It is `core_cut_red`, from
  the failed sase-core cut 40131163210. Since then, 13+ newer cuts passed and v0.42.5
  and v0.42.6 shipped. The steward builds its policy wheel from master's
  `sase-core-revision.txt`, so a policy fix in sase-core takes effect only after that
  revision moves past it.
- **Core lag:** master pins sase-core revision `67b77d0561fb` and
  `sase-core-rs==0.42.6`. No published core contains `67b77d05`; sase-core's newest
  release is v0.42.6. sase-core PR #338 ("chore: release v0.42.7", head `73a8f71e`)
  contains `67b77d05` and is open, `MERGEABLE`, `CLEAN` and green. Only the daily
  `41 7 * * *` cut or a `workflow_dispatch` with `dry_run=false` merges it.
- **Qualification:** rehearsal 40234446645 (at `ce75abd76f`) reports
  `QUALIFY_RED(test (3.12|3.13|3.14):tests)`. The cause is language-detection tests
  (`assert None == 'swift'`, `'kotlin'`, `'typescript'`) that need the unpublished core.
- **Release PR metadata:** `publish.yml`'s `sync-release-metadata` job runs
  `tools/ratchet_core_pin` on `release-please--branches--master` every 3 hours. It has
  failed with `CORE_LAG(pin=67b77d05…, newest_complete=0.42.6)` since 2026-10-25 23:16
  UTC. Once a core containing the pin is published, it ratchets release PR #322 (0.18.2)
  by itself.
- **Exercise:** in both wheel legs, the "Resolve the previous installable PyPI release"
  step reports every version from 0.18.1 down to 0.1.0 as "no longer resolves", then
  `UPGRADE_NO_INSTALLABLE_PREVIOUS`. The sdist legs pass. `publish` needs `exercise`, so
  a real tag run would fail here.

## Root causes established during planning

1. **Exercise resolver (note #9):** `release.yml` runs
   `uv pip install --dry-run --system --python 3.12 sase==<v>`. uv refuses `--system`
   against a PEP 668 externally managed interpreter, even with `--dry-run` ("The
   interpreter at /usr is externally managed…"). The Python 3.12 on ubuntu-latest is
   Debian-packaged and externally managed. On macOS, `--python 3.12` finds either an
   externally managed Homebrew interpreter or nothing. The step also runs after "Prepare
   the isolated leg environment" moves `HOME` and the XDG dirs, so uv no longer finds
   the managed 3.12 that `uv python install` placed under the original `HOME`. Planning
   reproduced the refusal with uv 0.13.5 against Debian's `/usr/bin/python3`, and showed
   that `sase==0.18.1` (with `sase-core-rs==0.42.6`) resolves in a throwaway
   `uv venv --python 3.12`. The step captures stderr and labels every non-zero exit "no
   longer resolves", so CI never showed the real cause. The wheel legs have never gotten
   past this step, so their later steps (`uv tool install` of N-1, the fakey launch, the
   upgrade) are unproven in CI.
2. **Stale `core_cut_red` (note #11):** `handle_no_train` in sase-core
   `crates/sase_core/src/release_steward/policy.rs` returns `CoreCutRed` for the first
   completed cut with a `typed_failure` anywhere in `core.cuts`. That list holds the
   in-flight cuts plus the newest 20 completed ones, newest first, per
   `tools/_release_watch_health.py` `_read_cuts` and `CUT_HISTORY_LIMIT`. Nothing checks
   whether a newer cut succeeded, and the check runs before the core-lag and release-PR
   logic.
3. **Core lag:** commit `ce75abd76f` moved the revision pin to unpublished `67b77d05`,
   and nothing publishes a core or ratchets master's exact pin on its own schedule
   except the daily cut and the release-branch sync.
4. **The handover gate didn't execute anything,** and `.10`'s recorded next action
   targets the wrong version (see above).

## Boundaries shared by every phase

1. **The human-only GitHub settings stay human.** The repository variable and the `pypi`
   environment rule change only through the gate that `cutover` raises, and the human's
   approval runs them. No phase runs `gh variable set` or deletes a deployment policy
   directly.
2. **Agents never create PRs, branches, tags or commits** (`host-owned-completion`).
   Never merge sase-core PR #338 or sase release PR #322 by hand: the epic exit requires
   that no human acted beyond the pre-authorized steps. Live proofs are
   `workflow_dispatch` rehearsals at `master`.
3. **Leave E2's own work to E2.** Do not close, edit or reassign `sase-1n6.10` or
   `sase-1n6`. Do not apply E2's memory decisions (`decisions:exact-release-pair` and
   the supersession marks): `.10` owns those. This epic edits no memory.
4. **Only core-pin moves the core pins.** No other phase edits `sase-core-revision.txt`,
   the `sase-core-rs` requirement in `pyproject.toml`, or `uv.lock`, even after a
   sase-core change (see `cut-red`).
5. **Phase workers create no beads.** Record follow-ups as `PROPOSED FOLLOW-UP:` notes
   on your own phase bead. Record a fix to an E2 discovered issue as a note on
   `sase-1n6`, the way note #3 did for #2.
6. **Waits go to `/sase_monitor`** with an explicit deadline and a follow-up. Never end
   a turn expecting to be woken. Before you change a file under `tools/`, read the
   `tools` reference memory. Before you finish, read `lint_and_test` and run the checks
   it requires.

## resolver

**Repo:** sase. **Goal:** both wheel legs pick the newest installable release at or
below the candidate (0.18.1 for a 0.18.2 candidate). A resolver failure that is not a
real resolution failure stops the leg with uv's own stderr, instead of skipping the
whole version list.

1. Move the inline Python in `release.yml`'s "Resolve the previous installable PyPI
   release" step into `tools/exercise_release` as a subcommand (for example,
   `previous-version`). Build it on the module's existing runner abstraction so unit
   tests can fake uv and the PyPI JSON. Keep the selection rules: numeric releases only,
   newest first, at or below the candidate, and skip a release whose own dependency
   closure is gone (old cores get deleted by retention).
2. Resolve each candidate against a throwaway venv: `uv venv --python 3.12 <tmp>`, then
   `uv pip install --dry-run --python <venv> sase==<v>`. Never use `--system`. Make
   interpreter discovery work under the relocated `HOME`. You can let uv fetch a managed
   3.12, run the step before the relocation, or point `UV_PYTHON_INSTALL_DIR` at the
   original install. Explain the choice in the step.
3. Classify failures:
   - A real resolution failure prints the tail of uv's stderr, then
     `sase==<v> no longer resolves; skipping`. A real resolution failure means uv
     reports no solution, unsatisfiable requirements, or a missing distribution.
   - Any other failure stops the step with a `::error::` line that names the cause and
     carries the stderr tail. Examples are venv creation failing, no interpreter, an
     externally managed refusal, or a network or index error.
   - `UPGRADE_NO_INSTALLABLE_PREVIOUS` stays the verdict for "no candidate resolves".
4. `release.yml` calls the subcommand and still writes `previous_version` to
   `GITHUB_OUTPUT`. Keep the step `id: prev` and the later `--previous-version` wiring
   unchanged.
5. **Note #10 (SC2012):** in "Build once and check the dist set", replace
   `ls dist/*.whl | wc -l` and the matching sdist count with `nullglob` arrays. Keep the
   exactly-one-wheel, exactly-one-sdist contract and its error text.
6. **Tests:**
   - Subcommand unit tests cover: the first candidate resolves; a resolution failure is
     skipped and its stderr printed; an externally managed or interpreter error stops
     the step without skipping; nothing resolves, giving
     `UPGRADE_NO_INSTALLABLE_PREVIOUS`.
   - The `release.yml` contract tests assert that the resolver step calls the subcommand
     and that `--system` appears nowhere in the workflow.
   - actionlint on `release.yml` reports no SC2012.
7. **Local proof:**
   - Run the subcommand on this Linux host against live PyPI for candidate `0.18.2`. It
     should pick 0.18.1.
   - Show that it stops loudly when it is forced onto an externally managed system
     interpreter.
   - If practical, run the wheel leg's N-1 install and upgrade path
     (`tools/exercise_release run-leg … --previous-version 0.18.1`) in a scratch `HOME`.
     CI has never exercised those later steps. Record what you could and could not
     prove.
8. Do not dispatch `release.yml`; `cutover` runs the live proof after this lands.

**Done when** the subcommand and contract tests pass, actionlint is clean, the local
proof is recorded in a note on this phase's bead, and the repo checks pass.

## cut-red

**Repos:** sase-core (`sase repo open sase-core`; read its `AGENTS.md`), plus one docs
row in sase. **Goal:** a failed core cut stops blocking the steward once a newer cut
succeeds.

1. In `handle_no_train`, walk the completed cuts newest first:
   - skip cuts that are neither successful nor a typed failure (cancelled, skipped,
     neutral);
   - the first success means no `CoreCutRed`, and evaluation falls through to core-ci,
     core-lag and the release-PR logic;
   - the first typed failure is the counted red. The existing `diagnostic_available` →
     `broken("core_cut", …)` rule applies to that counted cut only.
   - In-flight cuts never clear a red.
2. Confirm that `core.cuts` is newest first. Check the Python reader
   (`tools/_release_watch_health.py` in sase) and the wire docs in
   `release_steward/facts.rs`. If the wire contract does not promise that order, either
   document it in the wire comment or order the cuts inside the policy. Do not change
   the wire shape. If the fix seems to need one, stop and record a
   `PROPOSED FOLLOW-UP:`.
3. **Tests** go in `crates/sase_core/src/release_steward/tests/states.rs`. Cover:
   - the 2026-10-24 shape, where a typed-red cut followed by newer successful cuts is
     not `CoreCutRed`;
   - a newest completed cut that is a typed failure is `CoreCutRed`;
   - a red cut followed only by cancelled or in-flight cuts is still `CoreCutRed`;
   - a counted red without diagnostics is broken, and a superseded red without
     diagnostics is not.
4. In sase `docs/contribute/release-train.md`, change the steward state row for
   `core_cut_red` to say that the newest completed sase-core cut failed. Update any
   sase-core doc that states the old rule.
5. **Do not** touch sase's `sase-core-revision.txt`, `pyproject.toml` or `uv.lock`. This
   is a policy-only change with no binding change. `core-pin` decides whether the
   steward picks the fix up now. The steward builds from master's revision pin, and
   moving that pin to an unpublished commit would bring core lag back.

**Done when** `cargo fmt`, `clippy` and the release-steward tests pass in sase-core, and
the sase checks pass for the docs change. Record the landed sase-core commit SHA in a
note on this phase's bead, so that `core-pin` can test whether a published tag contains
it.

## core-pin

**Repo:** sase. **Goal:** master's exact `sase-core-rs==X.Y.Z` and `uv.lock` name the
newest complete published core whose tag contains `sase-core-revision.txt`. That is
expected to be 0.42.7, containing `67b77d05`.

1. Run `python3 tools/ratchet_core_pin --check`:
   - 0: the pin already matches; skip to step 4.
   - 2: a ratchet is pending; continue with step 3.
   - 3 with `CORE_LAG`: no complete published core contains the revision yet; follow
     step 2.
2. **If no published core contains the revision yet** (decision `core_cut`):
   - Under `wait`, hand the wait to `/sase_monitor`, polling `ratchet_core_pin --check`
     until it leaves exit 3, with about a 36-hour deadline. That deadline covers at
     least one scheduled 07:41 UTC cut, even when GitHub delays its cron by hours.
   - Under `dispatch`, confirm that no `release-plz.yml` run is queued or in progress.
     Then run `gh workflow run release-plz.yml -R sase-org/sase-core -f dry_run=false`
     once and monitor the same way.
   - If a cut fails, append its `CORE_CUT_RED` line as a `PROPOSED FOLLOW-UP:` note.
     Then wait through one more scheduled cut. Never dispatch more than one cut in this
     phase, and never merge PR #338 by hand.
3. **Turn the policy fix on when it ships.** Read `cut-red`'s landed commit SHA from
   that phase's bead note.
   - If the newest complete published core's tag contains it, set
     `sase-core-revision.txt` to that tag's peeled commit. That commit also contains
     `67b77d05`. The steward then builds the fixed policy.
   - If no published core contains it yet, leave the revision at `67b77d05` and do not
     wait for another release. Note that the fix turns on with a later pin advance. The
     stale red should age out of the 20-cut window on its own, and `cutover` checks for
     it.
   - Never move the revision past a published tag.
4. Apply `python3 tools/ratchet_core_pin --allow-transitive-lock-refresh`, then
   `uv lock`. The diff should touch only `pyproject.toml`, `uv.lock`, and
   `sase-core-revision.txt` if step 3 moved it.
5. If the revision moved, run `tools/check_sase_core_rs_bindings`. Also update any sase
   steward test or fixture that encoded the old stale-red behavior.

**Done when** `ratchet_core_pin --check` exits 0 on the working tree and the repo checks
pass. Record the chosen core version, its tag commit, and whether the cut-red fix is
active in a note on this phase's bead.

## cutover

**Repo:** sase (no code change is expected). Read the `/sase_gate` and `/sase_monitor`
skills first. **Goal:** everything the first steward release needs is proven green, and
one human approval finishes the handover and restarts `.10` cleanly.

1. **Pin check on current master.** Other agents may have moved the revision pin since
   `core-pin` landed. Run `python3 tools/ratchet_core_pin --check`:
   - 0: rehearse with no override.
   - 2: a newer complete core contains the revision, but master is not ratcheted.
     Rehearse with `-f core_version=<core-ensure pick>`, which matches what
     `sync-release-metadata` writes to the release branch. Record why.
   - 3 (`CORE_LAG`): hand the wait to `/sase_monitor` until the exit leaves 3 (about 36
     hours), then take the 2 path.
2. **Rehearse.** Confirm that no `release.yml` run is queued or in progress. Dispatch
   `gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse` (plus
   the override from step 1, if any), and hand the wait to `/sase_monitor` (about a
   3-hour deadline).
3. **Accept the rehearsal** only if all of the following hold:
   - identity, build and seal are green;
   - qualification is green;
   - all four exercise legs are green;
   - both wheel legs name a resolved previous version (expected 0.18.1) and pass the N-1
     upgrade;
   - the merged exercise verdict is green;
   - the verify-only publish plan is clean.

   If anything is red for a reason other than the train working as designed, do not
   raise the gate. Append a `DISCOVERED ISSUE:` note on `sase-1n6` and a
   `PROPOSED FOLLOW-UP:` note on this bead, each with the run URL and the failing step,
   and leave this phase open.

4. **Steward verdict.** Read the newest scheduled `release-steward.yml` run's verdict
   line (`gh run view <id> --log`, step "Run the steward tick"). The primary state must
   not be `core_cut_red`, `core_lag`, `core_ci_red` or `watcher_broken`. The expected
   state is a releasable release PR whose merge is suppressed because authority is off,
   or a daily-cadence wait.
   - If one of those states persists, hand the wait to `/sase_monitor`. Poll the
     scheduled steward runs with about a 48-hour deadline.
   - If the wait expires, still raise the gate, but put the blocking verdict and its
     owner in the gate's notes. The human decides; the phase never stalls silently.
5. **Release PR.** The open release-please PR has head ref
   `release-please--branches--master` (#322, 0.18.2, when this plan was written). Its
   head must contain the `resolver` and `core-pin` commits, and its `pyproject.toml`
   must pin the same exact core as the rehearsal. If the sync has not caught up yet,
   wait for the next scheduled `publish.yml` run (it runs every 3 hours) with
   `/sase_monitor`. Do not dispatch `publish.yml`.
6. **Notes:**
   - On `sase-1n6.10`, append a handover note covering:
     - the rehearsal URL, and that the wheel exercise failure is fixed and no longer a
       baseline;
     - the pinned core and whether the cut-red fix is active;
     - the current steward verdict and the release PR;
     - that settings are now changed by an executing gate, which this phase raises;
       `.10` should confirm the settings directly rather than through the old gate
       `custom-69faba44-…`;
     - that the first-release watch must be the bare
       `tools/release_watch --until-released --deadline 4d`. `--until-released 0.18.1`
       returns at once, because 0.18.1 is a confirmed pre-train legacy release.
   - On `sase-1n6`, append a note recording that discovered issues #9, #10 and #11 are
     fixed, with their commits, and whether the #11 fix is active in the pinned
     revision.
7. **Close this phase's bead** with `sase bead close` and the evidence. Then raise the
   gate with `/sase_gate` as the very last action. Creating a gate ends the turn.

**The gate.**

- **Title and notes:**
  - The title says what is decided, for example "Hand sase release merges to the steward
    (E2 cutover)", with a 🔐 icon.
  - The notes list the green rehearsal URL, the steward's verdict, the release PR and
    its pinned core, that the chezmoi ci_watch apply (old step 1) is already done, and
    exactly what each option runs.
- **Query** `(flip AND relaunch) OR defer`, with `[flip, relaunch]` as the primary
  branch and both selected by default.
- **`flip`**:
  1. Runs `gh variable set RELEASE_STEWARD_MERGE_ENABLED --body true -R sase-org/sase`.
  2. Reads `repos/sase-org/sase/environments/pypi/deployment-branch-policies` and
     deletes every policy whose `type` is `branch` (expected: the `master` rule only).
     The policy ids are looked up at run time, never hardcoded. The `v*` tag rule stays.
  3. Re-reads all three conditions that `.10`'s monitor polls, and emits JSON with the
     before and after values. Any failed step gives a failed status, with the detail and
     nothing hidden.
- **`relaunch`**:
  1. Stops any running monitor owned by `sase-1n6.10` (`sase monitor stop sase-1n6.10`,
     which launches no follow-up; "none running" is fine). The stale next action then
     never runs.
  2. Previews `sase bead work sase-1n6 -n` and relaunches with
     `sase bead work sase-1n6 -j` only when the preview shows `.10` relaunching rather
     than PRESERVE. `-j` implies `--yes-to-all`, which a non-interactive command needs,
     because the relaunch kills and restarts the waiting lander (a destructive cleanup
     that `-y` alone still prompts for).
  3. Emits JSON naming what it stopped and launched.
  4. Must work from any working directory; prove that with the `-n` preview.
- **Order:** the two options are safe in either order. If `relaunch` runs first, the
  fresh `.10` finds the settings pending and monitors them, as E2's exit step 1 directs.
- **`defer`**: a no-op that emits JSON. Feedback is optional.
- Before raising the gate, run each command script against a stub `gh` and `sase` on
  `PATH`, so that every result matches its `result_schema`. Never run the real `flip`
  commands.

**Done when** the rehearsal and steward evidence are recorded, both notes are written,
this bead is closed, and the gate is raised.

## After this epic

1. The human approves the gate: merge authority goes on, `pypi` becomes tag-only, and
   `.10` relaunches fresh.
2. The steward merges the release PR in its daily slot. release-please tags `v0.18.2`,
   and `release.yml` (which now contains the resolver fix) builds, qualifies, exercises,
   publishes and confirms the release.
3. `.10` proves the release and applies E2's memory decisions, and `sase-1n6.land`
   closes E2.

A candidate burned on the way is expected (E2 exit step 2), and `.10` watches the next
one.

## Risks

- **Late failures in the wheel legs.** The wheel legs have never passed the resolver in
  CI, so the rehearsal may surface a later failure (N-1 `uv tool install`, the fakey
  launch, or the upgrade). `resolver`'s local proof reduces this risk, and `cutover`
  stops with the evidence instead of raising the gate.
- **New core lag.** Any sase commit that moves `sase-core-revision.txt` to an
  unpublished sase-core commit brings core lag back. `cutover` re-checks the pin before
  rehearsing, and the steward waits out lag through the daily cut.
- **The cut-red fix may not be active yet.** If v0.42.7 is cut before `cut-red` lands,
  the fix waits for a later pin advance. The stale red then has to age out of the 20-cut
  window first, which takes about a day at the observed rate of roughly nine cuts a day.

## Non-goals

- `publish.yml`'s `sync-release-metadata` job turning red on every `CORE_LAG` run. It
  fails closed by design, and it heals once the core publishes.
- Changing how core lag counts a green push-triggered cut as "applicable core cut
  already completed". With the daily core cadence, the next scheduled cut publishes.
- The optional fourth handover step (removing `publish.yml`'s trusted publisher on
  PyPI).
- E2's memory decision record and acceptance evidence, which belong to `sase-1n6.10`.
