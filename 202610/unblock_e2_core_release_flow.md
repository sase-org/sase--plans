---
tier: epic
title:
  "Unblock E2 again: a green macOS core, core cuts the steward really dispatches, and a
  cutover wait that tracks master"
goal: "sase-core master CI is green on macOS again, so the daily cut can merge and
  publish a core. The release steward counts only merge-capable release-plz runs as core
  cuts, so its CORE_LAG action proposes a real cut and a heal run never hides a failed
  cut. sase-1oe.4 waits on the core pin that origin/master names now, not on a frozen
  workspace pin, so it can rehearse and raise the E2 handover gate.

  "
decisions:
  cut_budget:
    ask:
      When master's core pin lags, may the steward dispatch extra sase-core cuts beyond
      the daily 07:41 UTC cut?
    choices:
      daily: Steward cuts only with merge authority on, at most one per UTC day
      per_pin:
        One cut per lagging pin, even pre-handover; may exceed the PyPI quota cadence
    default: daily
    why: Bounds PyPI quota use, and adds no autonomous action before the human handover
    answer: daily
  core_cut:
    ask:
      Once the macOS fix lands, should macos-core dispatch one sase-core cut instead of
      waiting for the daily cut?
    choices:
      wait: Wait up to ~36h for the 07:41 UTC cut; sase-core keeps one release a day
      dispatch:
        Dispatch one cut once master CI is green; PyPI gets an extra core release
    default: wait
    why:
      Matches the earlier core_cut=wait answer; the human handover gate paces E2 anyway
    answer: wait
phases:
  - id: macos-core
    title: sase-core master CI passes on macOS, and a core containing the fix publishes
    depends_on: []
    size: medium
    description:
      "macos-core: find why wal_mode_probe_consumes_conversion_budget spends 750 ms to
      1.2 s of a 250 ms budget on macos-latest. Fix the product if it overshoots its
      wall-clock budget, or fix the oracle if the overshoot is outside the budget's
      control, without loosening the bound. Prove that master CI is green on both OSes
      and that a merge-capable cut publishes a complete core after the fix."
  - id: real-cuts
    title: Only merge-capable release-plz runs count as core cuts for the steward
    depends_on: []
    size: medium
    description:
      "real-cuts: the steward's cut facts drop push and six-hourly heal runs, which
      never merge the release PR. CORE_LAG then proposes a real cut, and a non-merging
      success no longer supersedes a failed cut. The cut_budget decision bounds how
      often the steward dispatches. Prove it with a live read-only --once."
  - id: master-wait
    title: The cutover wait follows origin/master's current pin
    depends_on: []
    size: small
    description:
      "master-wait: give tools/ratchet_core_pin a read-only --ref mode that checks the
      pins a git ref names. Write a corrected wait procedure on sase-1oe.4. If its
      running monitor polls a frozen workspace pin and can be resumed safely, restart it
      so its continuation starts now."
proposed_by: bbugyi200.athena.15m
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-28 20:07:10
status: wip
---

- **PROMPT:**
  [prompts/202610/unblock_e2_core_release_flow.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/unblock_e2_core_release_flow.md)

# Plan: Unblock E2 (`sase-1n6`) again

E2 (`sase-1n6`, "Exact-candidate release train and steward") has nine of ten phases
closed. Its last phase, `sase-1n6.10`, waits on a monitor that polls `sase-1oe.4` (the
cutover phase of the first unblock epic, `sase-1oe`) until it closes. `sase-1oe.4` waits
for a published sase-core that contains master's core source pin. That wait cannot
finish today, and the way it waits cannot win the race even after a core publishes. This
epic removes those blockers. It does **not** do `sase-1oe.4`'s cutover work (the
rehearsal, the steward-verdict and release-PR checks, the handover gate). That work
stays with `sase-1oe.4`.

Every phase agent should first read the context:

```bash
sase artifact read plan:202610/unblock_e2_release_cutover.md "<why>"
sase artifact read plan:202610/e2_release_train_steward.md "<why>"
sase bead read sase-1n6 sase-1n6.10 sase-1oe sase-1oe.4 -r "<why>"
```

In the E2 plan, read the steward's "Typed states", "Actions" and "Supporting rules". In
the `sase-1oe` plan, read "cutover". Notes #3 and #4 on `sase-1oe.4` are the latest
evidence.

## Where things stand (measured 2026-10-28, about 23:50 UTC)

- **Chain of waits.** `sase-1n6.land` waits on `sase-1n6.10`. `.10`'s monitor
  (`sase-1n6.10--mon`, "WAITING CUTOVER") polls `sase-1oe.4`'s bead status, with a 7-day
  deadline. `sase-1oe.4`'s monitor `aecm017a7yjz` (`sase-1oe.4--mon-0`, "WAITING CORE")
  runs `python3 tools/ratchet_core_pin --check` every 300 s inside workspace `sase_10`.
  That checkout is frozen at master `daf54c5b2d`, with pin `3f8baa79`. The monitor's 36
  h timeout ends about 2026-10-29 11:24 UTC.
- **Blocker 1: sase-core master CI has been red on macOS since 2026-10-27 21:37 UTC.**
  All nine master pushes since core `3f8baa79` ("count the WAL probe against the busy
  budget", from `sase-1ou.2.1.2`) failed the same node, and only on `macos-latest`:
  `tool_run::store::connection::tests::wal_mode_probe_consumes_conversion_budget`
  (`connection.rs:790`, "WAL setup renewed an exhausted budget"). Elapsed times were 746
  ms to 1.23 s against a 425 ms bound, for a 250 ms budget. Ubuntu passes. Consequences:
  - The daily cut 40706057331 (2026-10-28 14:27 UTC) refused to merge core release PR
    #340 (v0.43.1). It reported
    `CORE_CUT_RED check=cargo_fmt_+_clippy_+_test_(macos-latest)` from PR
    CI 40706223791.
  - Newest core release is still v0.43.0 (2026-10-27 13:47 UTC).
  - The newest steward tick, 40761995789, reports `core_ci_red`.
  - No bead owns this failure. `sase-1oe.4` note #3 recorded it only as a
    `PROPOSED FOLLOW-UP:`. Its epic lander triages that note only after `.4` closes, and
    `.4` cannot close until the failure is fixed.
- **Blocker 2: the pin moves faster than the core publishes.** Master's
  `sase-core-revision.txt` moved 8 times between 2026-10-27 21:40 and 2026-10-28 18:12
  UTC, with commits `57d5f411e4` through `d8b91b934f`. It now names unpublished
  `65546fd2`. `ratchet_core_pin --check` exits 3:
  `CORE_LAG(pin=65546fd2…, newest_complete=0.43.0)`. sase-core releases at most once a
  day. Its `41 7 * * *` cut often starts hours late; on 2026-10-28 it ran at 14:27 UTC.
  Right after a cut publishes, master's pin is usually inside it, until the next sase
  commit pins a newer core commit.
  - `sase-1oe.4` already lost this race once (note #4). Its wait ended when 0.43.0
    published, but its continuation started hours later. By then master pinned
    `3f8baa79`, so it began another 36 h wait.
  - The current monitor polls the frozen `sase_10` pin, so it will lose the same race
    again.
- **Blocker 3: the steward's CORE_LAG action never fires.** E2's design says
  `CORE_LAG(pin)` dispatches the core cut. Scheduled steward ticks already run
  `--tick --live`, and `dispatch_core_cut` is not gated by merge authority
  (`tools/_release_watch_actions.py`). The action still always comes out
  `already_satisfied`, with reason "applicable core cut already completed" (see run
  40506052239). The cause:
  - `tools/_release_watch_health.py` `_read_cuts` passes every `release-plz.yml` run in
    as a cut.
  - `matching_core_cut` (sase-core `release_steward/checks.rs`) matches any of them by
    `source_pin`.
  - Every core master commit gets a green push run, and push runs never merge the
    release PR. Only the `41 7 * * *` schedule or a `workflow_dispatch` with
    `dry_run=false` runs "Merge release PR". The six-hourly `23 */6 * * *` heal never
    merges either.

  The same confusion lets a green push or heal run "supersede" a failed merge cut in
  `counted_typed_cut_failure`. Even after the handover, a real release would wait on
  luck: the daily cut has to publish, and no pin move can land before the release PR
  syncs and the steward merges. `sase-1oe`'s plan listed this as a non-goal, assuming
  the daily cut would publish in time. Blocker 2 shows that assumption no longer holds.

- **Related, not owned here.** `sase-1p6` (READY, reopened) is a different set of
  ToolRun store tests in `tool_run/store/tests/contention.rs`. Its fix was lost when
  `sase-1ou.land` ended with a plan handoff, and that fix also edits `connection.rs`.
  `sase-1ou`'s lander owns redoing it.
- **Steward cadence.** `release-steward.yml` is scheduled every 15 minutes, but GitHub
  starts it only every 4 to 7 hours (ticks on 2026-10-28 at 08:06, 15:07 and 20:17 UTC).
  Every steward wait below should expect that.

## Boundaries shared by every phase

1. **Agents never create PRs, branches, tags or commits** (`host-owned-completion`).
   Never merge sase-core PR #340 or sase release PR #322 by hand. Never change
   `RELEASE_STEWARD_MERGE_ENABLED` or the `pypi` environment's deployment rules.
2. **Leave other epics' work to them:**
   - Do not dispatch `release.yml`, do not raise the handover gate, and do not close
     `sase-1oe.4`, `sase-1oe`, `sase-1n6.10` or `sase-1n6`. You may add notes where a
     phase below says so, and `master-wait` may restart `sase-1oe.4`'s monitor as
     described.
   - Do not redo `sase-1p6`, which `sase-1ou`'s lander owns.
   - E2's memory decisions belong to `sase-1n6.10`. This epic edits no memory.
3. **Core cuts.** Only `macos-core` may dispatch a sase-core cut, at most once, and only
   under `core_cut = dispatch`. No phase merges a release PR.
4. **Pins.** Move `sase-core-revision.txt` only when a sase change needs a new core
   commit (see `rust_core_backend_boundary`), and only in `real-cuts`. Never edit the
   exact `sase-core-rs==` pin or `uv.lock` by hand; the release-branch sync and
   `ratchet_core_pin` own them.
5. **Phase workers create no beads.** Record follow-ups as `PROPOSED FOLLOW-UP:` notes
   on your own phase bead.
6. **Waits go to `/sase_monitor`** with an explicit deadline and a follow-up. In
   sase-core, run `sase tool run check` and read its `AGENTS.md`. In sase, read the
   `tools` reference memory before you change `tools/`. Before you finish, read
   `lint_and_test` and run the checks it requires.

## macos-core

**Repo:** sase-core (`sase repo open sase-core`). **Goal:**
`cargo fmt + clippy + test (macos-latest)` is green on master, and a merge-capable cut
publishes a complete core after the fix.

1. **Measure before you change anything.** In the failing test, the holder keeps a WAL +
   `locking_mode = EXCLUSIVE` lock, and the contender calls
   `enable_wal_mode(&contender, 250 ms)`. Find where the time goes:
   - the initial `PRAGMA journal_mode` probe;
   - each `PRAGMA journal_mode = WAL` attempt;
   - the 5 ms sleeps between attempts.

   Reproduce on a macOS host if one is reachable (read the `tailnet` reference memory
   for the `mac` host). Work in a scratch checkout of the same core commit, and run the
   single test through the repo's `just test` recipe. If no macOS host is reachable, say
   so in your note and reason from the code and the CI timings.

2. **Hypothesis to confirm or reject.** Bundled `libsqlite3-sys` 0.30.1 builds SQLite
   with `HAVE_USLEEP=1`. SQLite's default busy handler then charges its nominal sleep
   schedule (1, 2, 5, 10, 15, 20, 25, … ms) against `busy_timeout`, not elapsed wall
   time. If each short `usleep` overshoots on a macOS runner, a 250 ms budget can spend
   about 1 s, which matches CI. If that is the cause, it is a product defect, not a test
   defect: the default ToolRun budget would overshoot the same way on a developer Mac.
3. **Fix the defect where it is.**
   - If the store overshoots its wall-clock budget, make every SQLite wait that charges
     a `BusyBudget` honor the wall-clock deadline. The waits that set a busy timeout are
     at `connection.rs` lines 371, 411, 529, 557 and 694 today. One way is to give each
     attempt only a short SQLite timeout, capped by the time remaining, and retry
     against an `Instant` deadline. The WAL loop already works that way. Another way is
     a busy handler that checks the deadline. Keep the single elapsed-budget contract
     from `sase-1ou.2.1.1`.
   - If the time goes somewhere the budget cannot control, change the oracle so that it
     observes the contract directly. For example, assert that the conversion loop never
     starts with a fresh full budget. Do not just raise the 425 ms bound. The sase-core
     `AGENTS.md` rule applies: never weaken an assertion to get a green run.
4. **Coordinate with `sase-1p6`.** Its lost fix promised that "the WAL conversion
   attempts once before honoring its deadline". If that redo lands on core master before
   you do, rebase onto it and keep both contracts. If yours lands first, add a note on
   `sase-1p6` saying what changed in `enable_wal_mode`.
5. **Tests.** Add a regression test that fails on Linux whenever the store charges
   nominal time instead of wall time, if you can do so deterministically (for example,
   with an injectable sleep or clock). Run `sase tool run check` in sase-core.
6. **Live proof.** After the host lands your commit, hand a monitor the wait for that
   commit's master `ci.yml` run (about a 3 h deadline). Both OS jobs must be green.
   Then, depending on `core_cut`:

   > [!decision] core_cut = wait Monitor `release-plz.yml` until the next scheduled
   > `41 7 * * *` cut completes (about a 36 h deadline).

   > [!decision] core_cut = dispatch First confirm that no `release-plz.yml` run is
   > queued or in progress. Then run
   > `gh workflow run release-plz.yml -R sase-org/sase-core -f dry_run=false` once, and
   > monitor it (about a 3 h deadline).

   The cut must merge the release PR and publish a complete release that contains your
   commit. If it fails for another reason, record its `CORE_CUT_RED` line as a
   `PROPOSED FOLLOW-UP:` and leave this phase open. Never dispatch a second cut.

**Done when** core master CI is green on both OSes at or after your commit, and the
published core contains it. Record its version and tag commit in a note on this bead.
Also add a note on `sase-1oe.4` saying that its note #3 follow-up is fixed, with the
commit, the version and that published core's tag commit.

## real-cuts

**Repos:** sase (`tools/`), plus sase-core (`sase repo open sase-core`) if
`cut_budget = daily` needs a new wire fact. **Goal:** a core cut means a run that could
merge the release PR. The steward's CORE_LAG action then proposes a real dispatch,
bounded by `cut_budget`.

1. **Classify runs in the facts reader** (`tools/_release_watch_health.py`, `_read_cuts`
   and `_cut_wire`):
   - a `push` run is never a cut;
   - a completed `schedule` or `workflow_dispatch` run is a cut only if its "Merge
     release PR" job ran (was not skipped). Use the jobs API; `_typed_cut_evidence`
     already reads job evidence;
   - an in-flight `schedule` or `workflow_dispatch` run counts as an in-flight cut. That
     is conservative: it blocks a second dispatch.

   If you find a cheaper exact signal, use it and explain the choice. One example is a
   `run-name` in sase-core's `release-plz.yml` that names the mode. `CUT_HISTORY_LIMIT`
   then counts real cuts, so check the `since` window still holds enough history.

2. **One meaning everywhere.** With only real cuts in `core.cuts`:
   - `matching_core_cut` finds no cut for a lagging pin unless a merging run happened at
     that pin;
   - `counted_typed_cut_failure` lets only a real successful cut supersede a failed one.

   Update the `cuts` doc comment in sase-core `release_steward/facts.rs`, and the
   `core_cut_red` and `core_lag` rows in sase `docs/contribute/release-train.md`, to say
   that push and heal runs are not cuts.

3. **Bound the dispatches** (decision `cut_budget`):

   > [!decision] cut_budget = daily `dispatch_core_cut` is proposed only when merge
   > authority is on (`RELEASE_STEWARD_MERGE_ENABLED`, which is already a policy input)
   > and no steward-dispatched cut (`workflow_dispatch`, `dry_run=false`) started
   > earlier the same UTC day. Otherwise it is suppressed, with a reason naming the
   > rule. Derive the day from run timestamps, never from memory. This is policy, so it
   > belongs in the sase-core steward policy, with Rust state tests. If the wire lacks
   > the fact, add the smallest one (for example the run's start time). Land the core
   > change first, then move sase's `sase-core-revision.txt` to that commit, run
   > `tools/check_sase_core_rs_bindings`, and update the reader in the same sase change.
   > The steward builds its policy from the revision pin, so the fix is active at once.
   > The pin is unpublished until the next cut, which is expected.

   > [!decision] cut_budget = per_pin No cap: E2 as written. Keep the policy unchanged
   > and make the reader change only, with no sase-core change and no pin move. Say in
   > the docs row that the steward dispatches one cut per lagging pin, even before the
   > handover.

4. **Tests.** Add reader fixtures for:
   - a green push run at the pin, which is not a cut;
   - a green heal run, which is not a cut;
   - a daily cut whose merge job ran, which is a cut;
   - a `dry_run=true` dispatch with the merge skipped, which is not a cut;
   - an in-flight scheduled cut;
   - a failed merge cut followed by a green push run, which is still red.

   Under `daily`, also add Rust state tests for the authority gate and the UTC-day cap.
   Fix any existing steward test or fixture that encoded the old behavior.

5. **Live proof, read-only.** Run `tools/release_watch --once --json` from current
   master with your change, against real GitHub and PyPI. If the pin lags, the core-lag
   action must now be proposed or suppressed for the reason the decision names, never
   `already_satisfied` because of a push run. Never run `--tick --live`. Record the
   output.

**Done when** the tests and repo checks pass and the live `--once` evidence is recorded.
Add two notes:

- On `sase-1n6`, record this E2 defect and its fix, the way note #3 recorded a fix.
- On `sase-1oe.4`, say what the handover gate's notes must now state. Under `daily`,
  turning merge authority on also lets the steward dispatch up to one core cut a day.

## master-wait

**Repo:** sase. **Goal:** `sase-1oe.4`'s next wait ends while master's current pin is
published, and its continuation acts on master as it is then.

1. **`tools/ratchet_core_pin --ref <git-ref>`.** Read `pyproject.toml`, `uv.lock` and
   `sase-core-revision.txt` from the ref (`git show <ref>:<path>`) instead of the
   working tree. Accept `--ref` only with `--check` or `--report-only`, and refuse it
   when applying. Add `--fetch`, which fetches `origin` first; a fetch failure exits 3
   with the cause. The exit codes are unchanged. Add tests for:
   - a ref whose pin is published, which exits 0 or 2;
   - an unpublished pin, which gives `CORE_LAG` and exit 3;
   - `--ref` without `--check` or `--report-only`, which is refused;
   - a fetch failure, which exits 3.
2. **Corrected wait procedure.** Append a `CORRECTED WAIT PROCEDURE:` note to
   `sase-1oe.4`:
   - **Poll:** `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`
     every 300 s with about a 36 h deadline, and exit on 0 or 2. Polling the working
     tree waits on a frozen pin, which is why its note #4 continuation found master had
     moved.
   - **Continuation:** on exit, run the same command again at once, and dispatch the
     rehearsal in the same turn if it still exits 0 or 2. Exit 2 keeps its existing
     `core_version` override rule.
   - **If the race is lost again,** start a new poll with the same command. Never poll a
     workspace checkout.
   - **Step 5 (release-PR sync) gets the same fallback as step 4.** If the release PR's
     pinned core cannot match the rehearsal within about 48 h, raise the gate anyway,
     and name the mismatch and its owner (the steward, after the handover) in the gate
     notes, so the phase never stalls silently. The release PR is now #322 ("release
     0.19.0").
3. **Restart a frozen wait, only if that is safe.** Check `sase monitor list` for a
   running monitor of `sase-1oe.4` whose command checks the working tree. When this plan
   was written, that was `aecm017a7yjz` in `sase_10`.
   - First confirm, from `sase monitor resume -h`, its docs and its tests, that resuming
     a monitor stopped with `sase monitor stop` launches its ordinary follow-up.
   - If it does, stop the monitor, resume it, and confirm with `sase monitor show` that
     a `sase-1oe.4` continuation launched.
   - If resuming would not launch the follow-up, or a continuation agent is already
     running, leave things alone. The note covers its next wait.
   - Record what you did in a note on this bead.

**Done when** the option and its tests pass, the repo checks pass, the note is on
`sase-1oe.4`, and the monitor action (or the reason for no action) is recorded.

## After this epic

1. `sase-1oe.4`'s continuation catches master's pin inside a published core, rehearses
   `release.yml` at master, and checks the steward verdict and the release PR. It then
   writes its handover notes, closes, and raises the handover gate.
2. When `sase-1oe.4` closes, `sase-1n6.10`'s monitor exits. The human approves the gate.
   The steward clears core lag itself, within `cut_budget`, and merges the release PR in
   its daily slot. `.10` proves the first steward release with the bare
   `tools/release_watch --until-released --deadline 4d`.
3. `sase-1oe.land` triages `sase-1oe.4` note #3. This epic's `macos-core` note marks it
   fixed.

## Risks

- **The macOS root cause may not be the busy handler.** `macos-core` measures first and
  fixes the real cause; the hypothesis is only a starting point.
- **New macOS reds.** A different red on core master blocks the cut the same way.
  `macos-core` reports it instead of chasing it, and the steward shows it as
  `core_ci_red`.
- **`real-cuts` under `daily` moves the pin**, so master lags again until the next cut.
  That is expected: `master-wait`'s polling catches the window after the next cut.
- **Slow steward ticks.** GitHub runs the 15-minute schedule every 4 to 7 hours. Every
  convergence step waits for a tick, so a steward release may take a day or two after
  the handover. `.10`'s 4-day watch deadline allows that.
- **`.10`'s cutover monitor deadline** ends about 2026-11-03. If the cutover runs past
  it, `.10`'s continuation re-reads its notes and waits again. Nothing here changes
  that.

## Non-goals

- `sase-1p6`'s lost contention-test and quarantine fix, which `sase-1ou`'s lander owns.
- Slowing the pin advances (agents' pin moves, `core-pin-ratchet.yml`) or changing the
  daily core cadence.
- GitHub's throttling of scheduled workflows.
- Any part of `sase-1oe.4`'s cutover or `sase-1n6.10`'s acceptance work.
