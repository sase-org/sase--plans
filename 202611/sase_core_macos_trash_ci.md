---
tier: tale
title: Turn sase-core CI green and publish the v0.44.0 core
goal: sase-core CI is green at its master tip and on the release-plz PR head. After
  a /sase_gate-approved merge, a complete sase-core-rs release containing sase's core
  pin is on PyPI, and ci_evidence names the version that unblocks the sase phase.
size: medium
proposed_by: bbugyi200.athena.sase-1rq.1
proposed_with: claude/opus
bead: sase-1rq.1
status: done
---

- **PARENT:**
  [202611/red_ci_repair.md](https://github.com/sase-org/sase--plans/blob/main/202611/red_ci_repair.md)
- **BEAD:**
  [sase-1rq.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.1.md)

# Plan: Turn sase-core CI green and publish the v0.44.0 core (bead sase-1rq.1)

This tale implements phase `sase-1rq.1` of epic `sase-1rq` ("Repair red GitHub Actions
CI across five projects", plan `plan:202611/red_ci_repair.md`). The epic's decisions are
final and this plan follows them:

- `test_quarantine = quarantine_flakes`: a test proven flaky on an unchanged tree may be
  skipped, with a flake follow-up per test. Timeouts, job lists, and lint levels stay as
  they are.
- `release_merge = gate`: the release PR merge is proposed through `/sase_gate`. Bryan
  approves it before anything publishes.

Read the epic plan's "Contract Shared By Every Phase" section once
(`sase bead read sase-1rq.1 -r "<why>"` prints its path). It defines "green", the
landing loop, and the required `ci_evidence` variable, and this plan relies on it.

## Root Cause (Confirmed During Planning)

Only the `cargo fmt + clippy + test (macos-latest)` job in sase-core `CI` is red. That
covers master `c8adc878` (run `42294092992`) and the release-plz PR #343
`chore: release v0.44.0` (head `d8c2bfc`, run `42346248489`). One test fails:

```
agent_artifact_run_trash::tests::quarantine_moves_to_mirrored_layout_and_restore_is_byte_identical
panicked at crates/sase_core/src/agent_artifact_run_trash/tests.rs:276:5
  left: "/private/var/folders/.../T/.tmpZnQowX/trash/success/journal.jsonl"
 right: "/var/folders/.../T/.tmpZnQowX/trash/success/journal.jsonl"
```

`quarantine_agent_artifact_runs` (`agent_artifact_run_trash/implementation.rs`, about
line 218) gets its trash root from `ensure_trash_root` (`helpers.rs`), which
deliberately returns the **canonicalized** root. `result.journal_path` is therefore
canonical. The test compares it with the raw `tempfile::tempdir()` path. On macOS,
`/var` is a symlink to `/private/var`, so the two strings differ. sase-core's
`AGENTS.md` names this exact trap: "Canonicalize both sides of a path comparison, or
neither side." The implementation is correct. The test's expected side is not.

**Linux reproduction (already verified while planning):** point `TMPDIR` at a symlink.
The failure then reproduces on Linux with the same left/right shape:

```bash
mkdir -p /tmp/sase1rq_real_tmp && ln -sfn /tmp/sase1rq_real_tmp /tmp/sase1rq_link_tmp
TMPDIR=/tmp/sase1rq_link_tmp just test -p sase_core --lib agent_artifact_run_trash
# -> 12 passed; 1 failed (same test, same assertion, real vs link path)
```

**Caveat:** the test panicked at line 276, so its second half (restore, then
byte-identical payload, then batch-dir removal) has **never run** under a symlinked temp
root on macOS. The symlinked-`TMPDIR` run after the fix is what proves that half.

## Step 1: Fix The Test (sase-core repo)

Open the repo with `sase repo open sase-core -r "<why>"`, read its `AGENTS.md`, and work
only in the printed path. Its master tip was `c8adc878` at planning time. Pull or fetch
first and rebase your understanding on the current tip.

In `crates/sase_core/src/agent_artifact_run_trash/tests.rs`, in
`quarantine_moves_to_mirrored_layout_and_restore_is_byte_identical`, replace the
`journal_path` assertion:

```rust
assert_eq!(
    result.journal_path,
    fixture
        .trash
        .join("success/journal.jsonl")
        .to_string_lossy()
);
```

with a comparison against the canonical trash root. The quarantine call has created that
root by this point:

```rust
// ensure_trash_root reports paths under the canonical trash root; on macOS the
// tempdir's /var ancestor resolves to /private/var.
let canonical_trash = fs::canonicalize(&fixture.trash).unwrap();
assert_eq!(
    Path::new(&result.journal_path),
    canonical_trash.join("success/journal.jsonl")
);
```

`Path` and `fs` are already imported. This keeps the stronger claim that the reported
path is the canonical one. Do not canonicalize `result.journal_path`, and do not drop
the assertion.

Don't make the shared `Fixture` canonicalize its tempdir. Running every other test in
the module through a symlinked ancestor is useful macOS coverage, so keep it.

If the symlinked-`TMPDIR` rerun shows the restore half failing too, that is a real
implementation bug in restore path handling. Fix it in `implementation.rs` /
`helpers.rs` by canonicalizing both sides, keep the test's assertions, and use a
`fix(...)` commit subject.

## Step 2: Verify Locally

1. `TMPDIR=/tmp/sase1rq_link_tmp just test -p sase_core --lib agent_artifact_run_trash`
   must pass all 13 tests. This is the macOS-equivalent proof, including the restore
   half.
2. The same filter without `TMPDIR` must pass (Linux default).
3. `sase tool run check` in the sase-core checkout. Never run bare `just check`, and
   give it a tool timeout of at least 15 minutes. It must pass. If it fails for a reason
   that reproduces identically on the clean base tree, apply the epic contract: record a
   follow-up (Step 7) and continue.

No `sase` repo change is needed. sase's `sase-core-revision.txt` pin (`c8adc878…`) is
already an ancestor of master, and this fix adds no binding. Leave the pin alone.

## Step 3: Land And Watch CI (end of the first coding turn)

1. Submit the turn through `/sase_final` with a **commit** decision for the sase-core
   repository. Use a Conventional Commit subject. Use
   `test(agent-artifact-run-trash): compare journal path against the canonical trash root`
   for the test-only fix, or a `fix(agent-artifact-run-trash): …` subject if Step 1
   needed an implementation change. Don't edit any version, pin, or `CHANGELOG.md`;
   release-plz owns them.
2. **Before** ending the turn, start a zero-weight wait through `/sase_monitor` (`-w 0`,
   labels `WATCHING CI` → `WATCHED CI`). The fix SHA isn't known until host completion
   pushes it, so the monitored script must discover it:
   - Poll `gh api repos/sase-org/sase-core/commits/master` until the head SHA is no
     longer `c8adc878…` and its history contains the fix commit (match on the subject).
     Then `gh run watch --exit-status` the `CI` push run for that SHA.
   - Then wait for the `Release-plz` push run on that SHA to refresh the release PR.
     Poll
     `gh pr list --repo sase-org/sase-core --state open --json number,headRefOid,headRefName`
     for the `release-plz-*` PR until its head moves past `d8c2bfc` and contains the
     fix. Check with `gh api repos/sase-org/sase-core/compare/<fix>...<head>` (`status`
     is `ahead` or `identical`). Then `gh run watch --exit-status` its `CI`
     `pull_request` run for that head.
   - Print both run ids, their head SHAs, and their conclusions. Exit non-zero if either
     run is not `success`.
   - Bound the whole script, for example 3 hours. A single macOS CI run takes about 20
     to 27 minutes.
   - Give the monitor a `--next` telling the follow-up turn to inspect both results,
     then either iterate (Step 4) or proceed to the release merge (Step 5).

## Step 4: Iterate On Red (monitor follow-up turns)

- A new deterministic failure caused by this change: fix it, then rerun Steps 2 and 3.
- A failure in an unrelated test: rerun the failed job once
  (`gh run rerun <id> --failed`). The repo's `Runner starvation retry` workflow may
  already have done this. If it then passes on an unchanged tree, it is a flake under
  `quarantine_flakes`. Look for an existing `sase-core flake:` bead first
  (`sase bead list -T flake`). Record it as a follow-up (Step 7) and say "green on
  attempt N" in the evidence. Skip a test only when it is proven flaky and keeps
  blocking. Never weaken an assertion, raise a timeout, or drop a job.
- A new regression from someone else's commit: fix it if the fix is quick. Otherwise
  record it as a follow-up and cite it in the evidence.

## Step 5: Gate The Release PR Merge (`release_merge = gate`)

Once master `CI` and the release PR's `CI` are both green at a head that contains the
fix:

1. Re-resolve the PR number and head. release-plz can recreate the PR.
2. Propose the merge through `/sase_gate`, using the same merge method the repo's own
   `.github/scripts/release_plz_merge.py` uses, pinned to the head whose CI was green:

   ```bash
   gh pr merge <N> --repo sase-org/sase-core --squash --delete-branch --match-head-commit <green-head-sha>
   ```

   In the gate text, state that it publishes sase-core-rs `<version>` (expected
   `0.44.0`) to PyPI at about 75 to 105 MB of quota. Include the two green run URLs.

3. **Do not** dispatch `release-plz.yml -f dry_run=false` and do not run `gh pr merge`
   outside the gate. Either one merges and publishes without Bryan's approval.
4. The repo's daily cut (cron `41 7 * * *` UTC) may merge the PR on its own before the
   gate is approved. If the PR is already merged when a follow-up turn runs, skip the
   gate and go to Step 6.

## Step 6: Watch The Publish And Verify (gate follow-up)

After the merge, the push to master runs `Release-plz`. Its `release-plz-release` job
tags `v<version>` and creates the GitHub release. The wheel matrix then builds, and the
`publish` job uploads to PyPI.

1. Watch that run through another `/sase_monitor` wait (`-w 0`, same labels) until the
   `publish` job finishes. If the run dies to runner starvation, the
   `Runner starvation retry` workflow and the six-hourly heal re-attempt it. Don't
   dispatch a manual publish without a new gate.
2. Verify that PyPI holds a **complete** release: every distribution in the workflow's
   `EXPECTED_DIST_SUFFIXES` (manylinux x86_64, manylinux aarch64, macOS universal2,
   Windows amd64, sdist) is present and unyanked. Check with
   `curl -s https://pypi.org/pypi/sase-core-rs/<version>/json`.
3. Verify that the release contains sase's pin. Fetch tags in the sase-core checkout,
   then run `git merge-base --is-ancestor <sase-core-revision.txt SHA> v<version>`. Read
   the SHA from the sase primary repo's `sase-core-revision.txt` at its current master.
4. Confirm `CI` on the final sase-core master tip (the release commit) is green. If it
   is still running, wait for it in the same monitor.

## Step 7: Follow-Ups, Evidence, And Closing

- **Follow-ups:** don't create beads. Record each flake, unrelated regression, or other
  discovered work with
  `sase bead note sase-1rq.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`, citing
  any existing bead that already tracks it. The epic's land agent triages these.
- **Evidence (final turn only):** publish `ci_evidence` through `/sase_var`, using the
  JSON shape from the epic contract. It holds `repo`, `default_branch`, `tip_sha`, and
  `runs`. `runs` has one entry each for master `CI` at the final tip, the release PR's
  `CI` at its merged head, and the `Release-plz` publish run. Each entry has `workflow`,
  `run_id`, `url`, `head_sha`, `event`, `attempt`, `conclusion`, and `completed_at`. The
  variable also holds `fixes` (SHA plus a one-line summary), `quarantined` (normally
  empty), and `notes`. `notes` must name the published sase-core-rs version that
  unblocks the `sase` phase (`sase-1rq.2`) and the pin-ancestry check result. Repeat the
  data as a short Markdown table in the final response, with the reproducing command
  `gh run list --repo sase-org/sase-core -L 20`.
- **Fallback:** if CI can't be made green after a very large, evidence-backed effort,
  follow the epic contract's research-report fallback in the `sase--research` sidecar,
  and publish `ci_report` instead.
- **Close:** in the final turn, run `sase bead epic-symbols sase-1rq.1` and resolve any
  leftovers. Then run
  `sase bead close sase-1rq.1 --note "<what was verified: run ids, version, pin ancestry>"`.
  Never close epic `sase-1rq` or any other bead, and never set bead status by hand.

## Out Of Scope

- sase's own workflows (Full CI, CI Telemetry, Publish `CORE_LAG`, release-please PR):
  phase `sase-1rq.2` owns them. Publishing the release is this phase's whole
  contribution to unblocking them.
- Moving sase's `sase-core-revision.txt` pin, other projects' CI, and any repo variable,
  secret, branch-protection, or workflow-permission change.
