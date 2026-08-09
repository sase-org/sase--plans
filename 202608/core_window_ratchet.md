---
tier: epic
title: Ratchet the sase-core-rs window at the release boundary instead of on feature
  PRs
goal: 'A feature agent can call a newly landed sase-core binding or behavior without
  editing pyproject.toml, uv.lock, or a version literal, and without waiting on a
  core release; the published-floor invariant is enforced once, mechanically, on the
  pending sase release instead of ~1.2 times a day by a dedicated agent.

  '
phases:
- id: derive-floor-literal
  title: Derive the telemetry smoke test's expected minimum from pyproject
  depends_on: []
  size: small
  description: 'derive-floor-literal: replace the hardcoded "0.21.3" golden assertion
    in tests/test_sase_core_rs_telemetry_smoke_tool.py with a value derived from pyproject.toml,
    so a floor change stops requiring a test edit.

    '
- id: ratchet-tool
  title: Build the window ratchet tool
  depends_on:
  - derive-floor-literal
  size: medium
  description: 'ratchet-tool: add a stdlib-only tools/ratchet_core_window that selects
    the newest fully published stable sase-core-rs from PyPI, rewrites only that requirement
    in pyproject.toml through one tested ceiling-policy function, refreshes uv.lock
    with a bounded diff guard, and supports --check/--report-only with idempotence
    and downgrade refusal.

    '
- id: release-lane
  title: Enforce the published floor on the release branch and at publish time
  depends_on: []
  size: medium
  description: 'release-lane: add a CI job that runs only on the release-please branch
    and validates the exact declared PyPI core floor against bindings, wire-schema
    probes, smokes and the contract set, and pin the floor exactly in publish.yml''s
    install-smoke so an incompatible floor mechanically blocks the PyPI upload.

    '
- id: release-ratchet
  title: Ratchet the window on the pending release branch in report-only mode
  depends_on:
  - ratchet-tool
  size: medium
  description: 'release-ratchet: extend publish.yml''s sync-lockfile job into a sync-release-metadata
    reconciler that runs the ratchet tool ahead of the lock refresh, landing it in
    report-only mode so a real release can be observed before it writes anything.

    '
- id: enable-ratchet
  title: Verify one report-only run and switch the ratchet to apply
  depends_on:
  - release-lane
  - release-ratchet
  size: small
  description: 'enable-ratchet: read the report-only output from at least one real
    master push with a pending release branch, confirm the proposed version and diff
    are exactly right, then flip the reconciler to apply and confirm the release branch
    stays green.

    '
- id: retire-conscription
  title: Stop conscripting feature agents into the floor bump
  depends_on:
  - enable-ratchet
  size: small
  description: 'retire-conscription: remove published-core-minimum-smoke from feature
    PRs and master pushes now that the invariant is enforced at the release boundary,
    and rewrite the three Justfile warnings and the docs that currently tell agents
    to bump the window by hand.

    '
- id: early-warning
  title: Add a non-fatal core-floor probe to just check
  depends_on: []
  size: medium
  description: 'early-warning: add a cached, offline-tolerant tools/probe_core_floor
    that runs the two stdlib probes against the declared floor, names the sase-core
    commit and release that provides any missing capability, and wire it into just
    check and just check-full as a warning that can never fail the run.

    '
- id: core-automerge
  title: Merge sase-core release PRs from the release-plz workflow
  depends_on: []
  size: small
  description: 'core-automerge: add a guarded job to sase-core''s release-plz.yml
    that waits for the release PR''s checks and squash-merges it, collapsing a median
    43-minute and worst-case 9h36m human wait into roughly ten minutes.'
proposed_by: bbugyi200.athena.wq
create_time: 2026-08-09 15:17:18
status: wip
bead_id: sase-ij
---

- **PROMPT:** [prompts/202608/core_window_ratchet.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/core_window_ratchet.md)
- **BEAD:** [sase-ij](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ij/README.md)

# Plan: Ratchet the sase-core-rs window at the release boundary instead of on feature PRs

## Context

SASE agents are repeatedly conscripted into raising the `sase-core-rs` version window in
`pyproject.toml:46`. The research report
`202608/core_dependency_window_ratchet/core_dependency_window_ratchet.md` in the
research sidecar measured the cost and diagnosed the cause; this plan implements its §7
recommendation. Read that report before starting any phase — it carries the
reproductions and the rejected alternatives, and this plan does not repeat them.

The short version:

- 67 commits have moved `pyproject.toml:46` since 2026-04-29; 50 of those in the 17 days
  to 2026-08-09. **21 closed beads since 2026-07-23 have a floor bump as their entire
  titled purpose — 16 of them phase beads** — plus 4 live now. That is ~1.2 dedicated
  agent launches per day, each paying a bead, a workspace claim, and a full context
  load.
- The cause is structural: a **release-time package invariant is enforced during feature
  integration**. `published-core-minimum-smoke` installs the exact declared PyPI floor
  and runs on every feature PR and master push, so the moment Python calls a new binding
  or depends on corrected core behavior, the feature agent is forced into the
  release-metadata ratchet — and then has to wait on a human merging a `sase-core`
  release PR (median ~43 min, mean ~2h20m, worst case 9h36m).
- 68 core releases published in the window against 8 `sase` releases. Moving the
  decision to the release boundary coalesces ~50 window edits into at most 8.

### The state this plan was authored against

`sase` master `be04f2874`, `sase-core` master with `459bbc6` unreleased. Reproduced now:

```
$ grep -n sase-core-rs pyproject.toml
46:    "sase-core-rs>=0.21.3,<0.22.0",
$ curl -s https://pypi.org/pypi/sase-core-rs/json | jq -r .info.version
0.21.3
$ git -C <sase-core> log --oneline --reverse -S'parse_merge_summary' -- crates/sase_core_py/src/lib.rs | head -1
459bbc6 feat(vcs-log): add parent ids and merge summaries
$ git -C <sase-core> tag --contains 459bbc6      # empty -> unreleased
```

`src/sase/core/vcs_log_facade.py:257` calls
`require_rust_binding("parse_merge_summary")`, so master right now needs a core
capability that no published wheel has. This is the live instance of the class, and it
is a useful end-to-end fixture for `early-warning`.

### Why the floor must not be computed from the probes

The report's §6 replayed the 12 most recent floor bumps against
`tools/check_sase_core_rs_bindings` + `tools/validate_sase_core_rs`: **5 of 12 (42%)
pass both probes against the pre-bump floor.** Those five moved the floor for pure
behavior changes (a git-log budget, close-metadata archival, a plan-header validation, a
per-tab icon, a snooze note) that a symbol-existence and wire-schema-version oracle
cannot see.

So this plan's authoritative policy is **track the newest published core at release
time** — correct by construction, never too low, no search needed. The probes keep their
value as a free, fast, _incomplete_ early warning and must never be promoted to
floor-setting authority. Do not "improve" `ratchet-tool` into a bisect for the lowest
satisfying version.

### What changes structurally

|                                         | Today                                      | After this epic                                         |
| --------------------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| Who decides the floor                   | whichever agent trips the gate             | `sync-release-metadata`, once per release               |
| When                                    | during feature integration                 | on the pending release-please PR                        |
| What is authoritative                   | `published-core-minimum-smoke` on every PR | the release-branch lane + the floor-exact publish smoke |
| Feature agent blocked on a core release | yes                                        | no                                                      |
| Feature agent's signal                  | a red CI job after push                    | a `just check` warning before commit                    |

### Two corrections to the report

Both were re-verified while authoring this plan; the phases below follow the corrected
version, not the report's wording.

1. **"Make it a required check" is not currently expressible on `sase`.**
   `gh api repos/sase-org/sase/branches/master/protection` returns 404 and
   `gh api repos/sase-org/sase/rulesets` returns `[]` — master is unprotected with zero
   rulesets, exactly like `sase-core`. There are no required checks to add to. So
   `release-lane` puts the _mechanical_ block where one already exists: `publish.yml`'s
   `publish` job needs `install-smoke`, so pinning the exact floor there blocks the PyPI
   upload with no repo policy change at all. The release-branch job is the early,
   visible signal, and it becomes a real gate for free if the owner ever enables a
   ruleset.

2. **Use the stdlib, not `packaging`.** The report suggests parsing with
   `packaging.version.Version`. Every other `tools/*sase_core_rs*` script is
   deliberately stdlib-only so it runs in a bare venv, and the release-metadata job
   installs neither `sase` nor its dependencies. `tools/validate_sase_core_rs_version`
   already has the right primitive: `_VERSION_RE = re.compile(r"\d+(?:\.\d+)*")` accepts
   only plain numeric releases, which _is_ the pre-release/dev/post rejection the report
   asks for. Reuse that shape. Note the hazard it protects against: PyPI's `releases`
   dict must be sorted with a version-aware key — a plain `sorted()` over those 85 keys
   returns `0.9.2` as newest.

### Owner decisions this plan commits to

Approving this plan settles the report's §11 open decisions as follows. Any of them can
be redirected in the approval feedback.

1. **Version-selection policy: newest fully published stable core at release time.**
   Accepts a narrower supported window for published `sase` installs. Justified in §6 of
   the report: `sase` and `sase-core` already release in near lockstep, and only 4 of 8
   sampled `sase` releases had a floor below the newest core available at the time.
2. **R3 variant: (a) workflow-driven merge, no repo policy change.** `sase-core` master
   is unprotected, so its release PRs are immediately mergeable, and GitHub only offers
   auto-merge on PRs that _cannot_ merge immediately — `gh pr merge --auto` there would
   degrade to an immediate merge that does not wait for CI. `core-automerge` therefore
   watches checks explicitly.
3. **`published-core-minimum-smoke` on feature PRs: removed, not advisory.** After
   `enable-ratchet`, the declared floor lags the tree _by design_ between releases, so
   an advisory job would be red on most PRs and most master pushes. Chronic red trains
   everyone to ignore the checks list. The local `just check` warning from
   `early-warning` replaces the signal at lower cost and earlier.
4. **Planning guidance and `sase/memory/` are out of scope.** See "Out of scope".
5. **`%wait(published=...)` is out of scope.** See "Out of scope".

## Phases

### Derive the telemetry smoke test's expected minimum from pyproject

`tests/test_sase_core_rs_telemetry_smoke_tool.py:35` asserts
`tool.declared_minimum_version(ROOT / "pyproject.toml") == "0.21.3"`. That literal is
the third of the three files every bump has to touch, and it adds no coverage: the very
next test in the same file, `test_declared_minimum_requires_inclusive_floor`, already
proves the parser against a `tmp_path` fixture, and a third `tmp_path` case can prove
the happy path just as well.

Rewrite the test so it derives its expectation from `pyproject.toml` rather than
restating it. Keep it a real assertion, not a tautology — assert the returned value is
exactly the inclusive minimum of the `sase-core-rs` requirement as read independently
(for example with `tomllib` plus the `>=` bound), so the test still fails if
`declared_minimum_version` returns the ceiling, the whole specifier, or `None`. Add a
`tmp_path` case that pins the happy-path parse against a fixture with a known floor if
the repo-derived assertion ends up weaker than what is there today.

Verify with `just check`. This phase touches only a test file, so the scoped lane
applies; it does not escalate.

### Build the window ratchet tool

Add `tools/ratchet_core_window`, stdlib-only, following the conventions of the existing
`tools/validate_sase_core_rs_version` (module docstring, `_error()` prefixing to stderr,
`main(argv)` returning an int, `raise SystemExit(main())`). Add
`tests/test_ratchet_core_window_tool.py` following
`tests/test_validate_sase_core_rs_version_tool.py` (loaded via `SourceFileLoader`, PyPI
access injected, never a live network call in a test). Add a `just ratchet-core-window`
recipe that forwards arguments, so the tool is runnable by hand and by a future axe
chop.

Behavior:

1. **Select the target version.** Read `https://pypi.org/pypi/sase-core-rs/json`.
   Consider only keys in `releases` that match a plain numeric release, sorted with a
   version-aware key. Skip any release whose files are all yanked. **Verify the
   distribution files, not just `info.version`** — a complete `sase-core-rs` release is
   exactly five files: four `cp312-abi3` wheels (macOS universal2,
   `manylinux_2_28_aarch64`, `manylinux_2_28_x86_64`, `win_amd64`) plus an sdist. Do not
   hardcode that list; require the candidate's set of non-yanked wheel platform tags to
   be a superset of the currently declared floor's, plus an sdist. That rejects a
   partial upload without breaking when a platform is added later. Bounded retry with
   backoff on transient PyPI failures; a network failure must be a distinguishable,
   non-destructive exit.
2. **Rewrite one requirement.** Change only the `sase-core-rs` entry in
   `[project] dependencies`, in place, preserving the rest of the file byte for byte. A
   targeted line rewrite is preferable to a TOML round-trip here precisely because it
   cannot reformat anything else.
3. **Compute the ceiling from one tested policy function.** Today `0.MINOR.PATCH` →
   `<0.(MINOR+1).0`; for `>=1.0`, `MAJOR.*` → `<(MAJOR+1).0.0`. One function with its
   own unit tests, so the pre-1.0 → 1.0 transition is a one-line change and never YAML
   string arithmetic.
4. **Refresh the lock.** `uv lock --upgrade-package sase-core-rs==<V>` so unrelated pins
   stay put, then **guard the diff**. The expected `uv.lock` change is exactly seven
   lines replaced: the `version` line, the `sdist` line and the four `wheels` lines
   inside the `[[package]] name = "sase-core-rs"` stanza, plus the one
   `{ name = "sase-core-rs", specifier = "..." }` line in the metadata `requires-dist`
   block. Fail if the diff touches anything outside those two regions.
5. **Refuse to go backwards.** Never lower the floor, never select a pre-release, and
   exit cleanly with no writes when the declared floor is already the newest published
   version. Re-running on an already-ratcheted tree must produce an empty diff.
6. **Modes.** Default applies the change. `--report-only` prints the selected version
   and the exact diff it _would_ write and writes nothing. `--check` writes nothing and
   exits non-zero when a ratchet is pending. Use distinct, documented exit codes for
   "nothing to do", "ratchet pending/applied", and "could not determine" — the same
   directional-exit-code convention the plans sidecar's
   `202608/stale_core_binding_guard.md` established for this family of tools.

Do not extend this into a search for the lowest satisfying version; see "Why the floor
must not be computed from the probes" above.

Verify with `just check`, plus a manual `tools/ratchet_core_window --report-only`
against the real index confirming it currently proposes no change (the floor is already
`0.21.3`, PyPI's newest).

### Enforce the published floor on the release branch and at publish time

Two mechanisms, both landing here, because neither is useful alone.

**A. The release-branch compatibility lane (visible signal).** Add a job to
`.github/workflows/ci.yml` that is the exact inverse of the existing exclusion:

```yaml
if:
  github.event_name == 'pull_request' && github.event.pull_request.head.ref ==
  'release-please--branches--master'
```

It must not depend on `build-core` — that job is skipped on this branch on purpose, and
the whole point is to test the _published wheel_, not a source build. Install the exact
declared floor (`tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml`
already prints it) and run, at minimum:

- `tools/check_sase_core_rs_bindings`
- `tools/validate_sase_core_rs`
- the four `tools/smoke_sase_core_rs_*` scripts
- the contract-marked set (`tests/contract_manifest.txt` / `-m contract`)

Then run the full non-visual suite against that floor if the cost is acceptable. Measure
it and decide with a stated criterion rather than guessing: the release branch is
force-updated by release-please on every master push, so this lane re-runs often. The
existing `concurrency: ci-${{ github.ref }}` with `cancel-in-progress` for non-master
refs already bounds that to one live run per PR — confirm that actually applies to this
job before relying on it. If the full suite still materially delays releases, record the
measurement in the phase notes and ship the probes + smokes + contract set only; the
marginal signal the full suite adds over the master `test` job is specifically
published-wheel behavior divergence.

Reusing `.github/actions/setup-sase` is attractive because `_setup` honors
`SASE_CORE_WHEEL` and `_core-overrides-arg` then lifts the window so nothing re-resolves
the core out from under you. It currently hard-requires the `sase-core-wheel` artifact
and its `sase-core-sha.txt`, so either add an input that sources the wheel from PyPI
instead, or write the install inline. Whichever you pick, assert after install that
`importlib.metadata.version("sase-core-rs")` equals the declared floor exactly — the
failure mode this whole lane exists to prevent is silently testing a _newer_ core than
the floor.

**B. The floor-exact publish smoke (mechanical block).** `publish.yml`'s `install-smoke`
installs the built `sase` wheel and lets PyPI resolve `sase-core-rs` anywhere in the
window, so it never exercises the floor. `publish` has
`needs: [release, build, install-smoke]`, so tightening `install-smoke` is the one place
an incompatible floor can be stopped without branch protection. Add a leg (or a sibling
job that `publish` also needs) that installs the wheel with `sase-core-rs` pinned to the
exact declared floor and runs the same health check and public-command smokes. Keep the
existing free-resolution leg — the two answer different questions.

Accepted trade-off, state it in the phase notes: when the newest published core is
genuinely incompatible with the tree, this blocks the `sase` release until a core
release lands. That is the correct behavior and is precisely what `core-automerge`
shortens. It is also _live_ today — see "The state this plan was authored against" — so
expect the first release after this epic to block until `sase-core` publishes `459bbc6`.

Verify by pushing a branch that exercises the job's `if:` logic, and by confirming with
`gh run view` that the lane really ran on a release-please branch and really did not run
on an ordinary PR.

### Ratchet the window on the pending release branch in report-only mode

Extend `publish.yml`'s `sync-lockfile` job into `sync-release-metadata`. It already does
every hard part: it detects `release-please--branches--master`, checks it out with
`SASE_RELEASE_TOKEN`, commits as `github-actions[bot]`, and pushes back. Insert the
ratchet ahead of the lock refresh so one commit carries both, and land it with the
ratchet in **`--report-only`** so it prints its proposal and changes nothing.

Design constraints:

- **It is a reconciler, not a one-shot.** release-please regenerates the release branch
  on every master push. The job already re-runs on every master push and is already
  idempotent (`uv.lock already matches the pending release version.`); the window update
  must have the same shape. Never assume a previous run's commit is still on the branch.
- **Serialize.** The workflow's `concurrency` group must keep two master pushes from
  racing on the same branch. Verify the existing
  `${{ github.workflow }}-${{ github.ref }}` group with `cancel-in-progress: false` is
  sufficient here.
- **Ignore any event's notion of the version; always re-read PyPI.** Stale or
  out-of-order events must not be able to lower the floor. The tool refuses downgrades;
  the job must not route around that.
- **No pending release branch → exit 0.** The existing `check_branch` step already
  handles this. Keep it.
- **Do not open a separate dependency PR**, and do not add Dependabot or Renovate for
  this dependency. Both are capable of the text transformation and neither expresses the
  policy; both fire per core release, which at 3-5 core releases/day reproduces the
  churn in a new place. The pending release PR is the accumulator.

Also note in the phase notes what happens if release-please force-pushes between the
ratchet commit and the merge: the ratchet is an _optimization_, and `release-lane` is
the correctness guarantee. A dropped ratchet commit makes the release lane red and the
publish smoke fail; it cannot silently ship a bad floor.

Verify by running the job's script path locally against a scratch clone, and by
confirming after landing that the next master push produces a report-only comment in the
job log naming a concrete version and diff.

### Verify one report-only run and switch the ratchet to apply

Wait for at least one real master push that finds a pending release branch, then read
the report-only output. Confirm all of:

- the proposed version is the newest fully published stable `sase-core-rs`, checked
  independently against the PyPI JSON API;
- the proposed `pyproject.toml` diff is the single `sase-core-rs` line;
- the proposed `uv.lock` diff is exactly the seven expected lines;
- re-running proposes the identical change (idempotent), and proposes nothing once the
  floor already matches.

Then flip the reconciler to apply, watch the first real ratchet commit land on the
release branch, and confirm `release-lane` goes green against the newly selected floor.
If the lane goes red, do not lower the floor to make it green — that is the gate doing
its job; record which core capability is missing and stop.

If no release branch exists during this phase's window, say so in the phase notes rather
than flipping the mode unobserved.

### Stop conscripting feature agents into the floor bump

Only after `enable-ratchet` proves the invariant is enforced at the release boundary:

1. **Remove `published-core-minimum-smoke` from feature PRs and master pushes.** Its
   five steps now live in `release-lane`. Do not leave it as an advisory job — see owner
   decision 3 above.
2. **Rewrite the three warnings that actively point agents at the bump.** `Justfile:99`
   (`_setup`), `Justfile:779` (`rust-install`) and `Justfile:839` (`rust-dev-install`)
   all print `WARNING: bump the published sase-core-rs window in pyproject.toml`. That
   advice is now wrong: the checkout running ahead of the published window is the
   _normal_ dev state and requires no action, because the release-branch reconciler owns
   the window. Replace them with a message that says so. Do not remove the diagnostic
   itself — the _behind_ direction (bit 16, the hard error added by the plans sidecar's
   `202608/stale_core_binding_guard.md`) must keep failing exactly as it does today.
3. **Document it.** Extend `docs/rust_backend.md` around line 296, where the version
   window is already explained, with a short subsection: who owns the window, when it
   moves, what a contributor should do when they need an unreleased core capability
   (nothing — land the Python change; the release lane will block the release until the
   core release exists), and how to run the ratchet by hand.

Verify with `just check`, `just docs-check`, and by confirming on the next feature PR
that `published-core-minimum-smoke` no longer appears in the checks list while the
source lanes are unchanged.

### Add a non-fatal core-floor probe to just check

Add `tools/probe_core_floor` plus `tests/test_probe_core_floor_tool.py`, and wire it
into both `check` and `check-full` in the `Justfile` right after the `SASE validation`
step.

What it does: creates a scratch venv, installs the _declared floor_ wheel from PyPI,
runs `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` against it,
and emits one of three verdicts:

- `ok` — the published floor satisfies both probes.
- `stale_actionable` — a probe fails and the capability exists in a _published_ core
  release. Name it: the report verified both halves of the lookup work —
  `git log -S'<name>' -- crates/sase_core_py/src/lib.rs` finds the commit and
  `git tag --contains <sha>` finds the release.
- `blocked_unpublished` — a probe fails and the capability is on `sase-core` master but
  in no tag yet. Say so, and name the commit. **This is the message that replaces the
  forensic bead**: it is exactly the archaeology `sase-hz`'s description was
  hand-written to produce.

Emit a small JSON verdict alongside the human text, with directional exit codes matching
the `stale_core_binding_guard.md` convention, so `axe` can consume it later without a
rewrite.

Hard requirements:

- **Warn, never fail.** A `--advisory` flag forces exit 0 while still printing the
  diagnosis; `just check` uses it. Given the 42% blind rate this signal is incomplete by
  construction, and a hard gate on it would re-create the exact blocker
  `retire-conscription` removes. Non-negotiable.
- **Silent when `ok`.** `just check` output is read by every agent on every run.
- **Cached.** A published version's export set never changes, so steady state must be
  zero network calls. Cache verdicts keyed on the declared floor plus a digest of the
  collected binding-name set (`tools/check_sase_core_rs_bindings --list` needs no wheel)
  and of the two probe scripts. Mirror `tools/validate_test_environment`'s cache: JSON,
  atomic `os.replace`, `fcntl.flock` around read-modify-write. Prefer a shared cache
  under `$XDG_CACHE_HOME/sase/` over one inside `.venv` — workspaces are ephemeral and
  there are 24 of them, so a per-workspace cache would re-download once per workspace
  per core release.
- **Offline-tolerant.** No network, no PyPI, or no scratch venv means "skipped", printed
  at most once and exit 0. It must never make `just check` slower or flakier on a plane.
- **Must not touch `.venv`.** The dev venv holds the locally built extension; installing
  a published wheel into it would break every subsequent step.

Against master as it stands this must report `blocked_unpublished` naming
`parse_merge_summary` and `sase-core` commit `459bbc6` — use that as the acceptance
check, and note in the phase notes that it will report `ok` instead once that core
release publishes.

Verify with `just check` (twice — the second run must be a cache hit and near-instant),
with the network disabled, and with `just check-full`.

### Merge sase-core release PRs from the release-plz workflow

This phase works in the `sase-core` repo. Open it with
`sase repo open sase-core -r "<reason>"` and use only the path that prints.

`sase-core`'s `.github/workflows/release-plz.yml` has a `release-plz-pr` job whose
release-plz step outputs `pr` and `prs_created`. Add a guarded follow-on job (its own
job, not a step, so it gets its own timeout) with `needs: release-plz-pr` that waits for
the release PR's checks and squash-merges it:

```
gh pr checks <n> --watch --fail-fast
gh pr merge <n> --squash --delete-branch
```

Details that will bite:

- **Do not use `gh pr merge --auto`.** `sase-core` master is unprotected
  (`gh api repos/sase-org/sase-core/branches/master/protection` → 404) and
  `allow_auto_merge` is `false`, so GitHub does not offer auto-merge on an immediately
  mergeable PR and `--auto` degrades to an immediate merge that does not wait for CI.
  That is the report's own correction to its `__b` source; the whole value here is
  _waiting_.
- **Resolve the PR robustly.** `prs_created` is false when release-plz merely _updates_
  an existing release PR, which still needs merging. Look up the open PR by its head
  branch rather than relying solely on that output.
- **Guard what you merge.** Only merge a PR whose author is the release bot and whose
  head branch and title match release-plz's release-PR shape. Never merge an arbitrary
  open PR.
- **Checks are not registered instantly.** `gh pr checks --watch` on a PR whose checks
  have not appeared yet exits immediately with "no checks reported". Poll for checks to
  appear, with a bounded timeout, before watching them.
- **Timeouts.** `release-plz-pr` is `timeout-minutes: 30`; the new job needs its own
  budget covering `rust-checks` + `wheel-smoke`.
- **Token.** The release PR is opened with `SASE_RELEASE_TOKEN`, which is why its CI
  runs at all; the watch/merge steps need `GH_TOKEN` and `pull-requests: write` +
  `contents: write`.

Risk is low and worth stating in the phase notes: release-plz only opens a release PR
when releasable commits exist, the PR contains only a version bump and changelog,
`sase-core` CI already gates it, and the publish job independently verifies the tag and
every distribution's metadata and self-heals on a 6-hour schedule.

Verify by watching one real release cycle end to end and recording merge → PyPI
wall-clock in the phase notes. Expected: green CI plus ~10 minutes, against today's
median 43 minutes.

## Out of scope

- **The `axe` chop (`core_floor_ratchet`, the report's R5).** It is explicitly
  "optional, after R1-R2 prove out", and with the release-boundary ratchet in place it
  should rarely have anything to fire on. `early-warning` deliberately emits a JSON
  verdict with directional exit codes so the chop is a thin wrapper if a residual gap is
  ever observed. The chop config also lives in the chezmoi-managed global `sase.yml`,
  not in this repo.
- **Plan-authoring guidance and `sase/memory/` updates** (the report's §11.4). Once this
  epic lands, plans should stop allocating a dedicated "raise the sase-core-rs
  dependency window" phase. Editing `sase/memory/*.md`, `AGENTS.md`, or the provider
  shims requires explicit in-conversation permission from the project owner, and a plan
  file cannot supply that permission. Recorded here as a proposal for the owner, not as
  work.
- **A `%wait` predicate for external state** (the report's §11.5) — e.g.
  `%wait(published=sase-core-rs>=0.22.0)`. `release-lane` and `core-automerge` together
  make it much less urgent, and it is a separate feature in the agent runtime.
- **Merging `sase-core` into `sase`.** The deepest fix for the whole problem class is
  removing the cross-repo release coupling, and it is disproportionate: a Rust toolchain
  in every `sase` CI lane, slower installs, and loss of independent core cadence.
- **Retiring the live floor beads.** `sase-i8.8`, `sase-hz` and any sibling still open
  when this lands are the owner's triage call, not this epic's. Once `enable-ratchet` is
  done they should need no agent.
- **Bisecting for the lowest satisfying published version.** Rejected on evidence in §6
  of the report (42% blind rate). Revisit only if the probe suite grows to cover
  behavior contracts, and then only as an opt-in flag behind the newest-published
  default.
