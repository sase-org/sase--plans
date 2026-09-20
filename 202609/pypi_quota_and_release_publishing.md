---
tier: epic
title: Restore sase-core PyPI publishing and bound its storage growth
goal: 'The sase-core Release-plz workflow publishes complete releases to PyPI again,
  the project sits well under its 10 GB PyPI storage limit with months of headroom
  instead of days, a partial upload can no longer be mistaken for a published release,
  and a pre-flight guard fails loudly with an actionable message before PyPI can reject
  an upload mid-stream.

  '
phases:
- id: reclaim
  title: Reclaim PyPI storage below the limit
  depends_on: []
  size: medium
  description: 'reclaim: confirm no consumer pins a doomed version, build the retention
    keep/delete list, and drive the human-gated pypi-cleanup deletion until the project
    is back under 10 GB.'
- id: shrink
  title: Measure and reduce per-release wheel bytes
  depends_on: []
  size: medium
  description: 'shrink: measure link-time options against the 36 MB extension module,
    land only changes that shrink it without regressing behavior, and report the macOS
    x86_64 slice decision rather than taking it unilaterally.'
- id: heal
  title: Gate on file-set completeness and heal the partial 0.34.48 release
  depends_on:
  - reclaim
  size: medium
  description: 'heal: replace the version-existence publish gate with an expected-file-set
    check, top up the partial 0.34.48 release, and get the tagged backlog through
    0.34.66 published complete.'
- id: preflight
  title: Pre-flight PyPI quota guard and headroom reporting
  depends_on:
  - heal
  size: small
  description: 'preflight: check remaining project storage before uploading, fail
    with an actionable message instead of a mid-upload 400, and surface headroom in
    the job summary.'
- id: cadence
  title: Bound release cadence to a daily cut
  depends_on:
  - preflight
  size: medium
  description: 'cadence: stop auto-merging the release PR on every master push, cut
    releases on one daily schedule instead, and keep the six-hourly heal plus a manual
    escape hatch for urgent floor bumps.'
- id: verify
  title: End-to-end verification and downstream unblock
  depends_on:
  - heal
  - preflight
  - cadence
  - shrink
  size: small
  description: 'verify: prove a complete five-file release lands through the changed
    path, record measured headroom, and corroborate the downstream beads that were
    blocked on published core.'
proposed_by: bbugyi200.apollo.11
create_time: 2026-09-20 08:29:23
status: done
bead_id: sase-13t
---

- **PROMPT:** [prompts/202609/pypi_quota_and_release_publishing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pypi_quota_and_release_publishing.md)
- **BEAD:** [sase-13t](https://github.com/sase-org/sase--beads/blob/main/pages/sase-13t/README.md)

# Plan: Restore sase-core PyPI publishing and bound its storage growth

All file paths below are relative to the **`sase-core`** repository. Open it with the
`/sase_repo` skill and use the path that skill prints; do not guess a checkout location.

## Root cause

The `Release-plz` workflow's `publish to PyPI` job fails on every run. PyPI rejects the
upload:

```
400 Project size too large. Limit for project 'sase-core-rs' total size is 10 GB.
```

Measured from the PyPI JSON API on 2026-09-20, `sase-core-rs` holds **272 releases /
1358 files / 10.73 GB** — 107% of the 10 GB per-project storage limit. This is not a
transient failure and not a credentials, metadata, or build problem: every job before
`publish` succeeds, `twine check` passes, and the upload itself is refused on quota.

Two independent drivers compounded to get here:

1. **Release cadence.** 272 releases across 63 active days is **4.32 releases/day**.
   `release-plz-merge` auto-merges the release PR on _every_ master push, so effectively
   every releasable conventional commit becomes a PyPI release within minutes.
2. **Per-release size growth.** A release grew from ~15 MB (0.1.2, June) to ~75 MB
   (0.34.47, September) — a 5x increase in three months. One release is five files:
   macOS universal2 28 MB, Windows 16 MB, Linux x86_64 15 MB, Linux aarch64 14 MB, sdist
   2 MB.

Together that is roughly **324 MB of quota consumed per day**.

## Why this needs fixing on two axes

Deleting old releases is necessary but is _not_ by itself a fix. Measured retention
scenarios, keeping only the newest published versions:

| Keep                 | Remaining | Headroom | Runway at 4.32 rel/day | Runway at 1 rel/day |
| -------------------- | --------- | -------- | ---------------------- | ------------------- |
| 20 (0.34.29–0.34.48) | 1.44 GB   | 8.56 GB  | ~26 days               | ~114 days           |
| 30 (0.34.19–0.34.48) | 2.13 GB   | 7.87 GB  | **~24 days**           | **~105 days**       |
| 40 (0.34.9–0.34.48)  | 2.81 GB   | 7.19 GB  | ~22 days               | ~96 days            |

Reclaiming ~8.6 GB buys only about **three and a half weeks** at the current cadence.
Bounding cadence is what turns the reclamation into a durable fix, which is why
`cadence` is in scope and not deferred.

Note also that deletion is **irreversible**: PyPI permanently burns a deleted
version/filename and will never accept a re-upload of it. Every deletion step below is
therefore human-gated, never automated.

## A second, latent defect: partial releases read as published

Version **0.34.48 is a partial release**. It holds only 3 of the expected 5 files:

- present: macOS universal2, Linux x86_64, Linux aarch64
- **missing: `win_amd64` wheel and the sdist**

Those two uploads were the ones rejected when the quota was crossed mid-upload.

The `publish-plan` job decides whether to publish by asking whether the _version
exists_:

```
GET https://pypi.org/pypi/sase-core-rs/${version}/json   ->  200 = "published", 404 = "absent"
```

A partial release answers `200`, so `needs_publish=false` and the self-healing loop the
workflow was explicitly designed around will **never** heal 0.34.48. This matters
because `sase` pins `sase-core-rs>=0.34.48,<0.35.0` — the pinned floor is exactly the
broken release, Windows installs of it cannot resolve a wheel, and there is no sdist to
fall back to. An agent already observed this independently (note on `sase-11l.11.4`:
"PyPI sase-core-rs is still 0.34.48 (wheels only, no sdist)").

The publish step already sets `skip-existing: true`, so a retry can only top up files
that are genuinely missing — the heal is safe once the gate stops lying.

## Blast radius

Versions **0.34.49 through 0.34.66** (18 releases) are tagged but unpublished. Work
already blocked on published core includes `sase-10d` (READY — cut a release and raise
the floor), `sase-12y.4`, `sase-12w.6.4.2`, `sase-11l.11.4`, and `sase-zr.7.1.1.5.4.1`.

## Constraint that shapes the design

**PyPI has no delete API.** Upload tokens are upload-only. The `pypi-cleanup` utility
deletes by scraping the PyPI web login form — CSRF token, account password, and a live
TOTP code (verified by reading `src/main/python/pypi_cleanup/__init__.py` in
`arcivanov/pypi-cleanup`). An agent cannot produce a TOTP code, and an irreversible bulk
deletion should not run unattended anyway.

So the retention story is deliberately split: **deletion stays a human-gated, occasional
operation**, and CI's job is to make it _rare_ (cadence + size) and to _fail early and
legibly_ when headroom runs low (preflight). Do not try to automate deletion in the
workflow.

---

## reclaim: Reclaim PyPI storage below the limit

Goal: `sase-core-rs` back under 10 GB with substantial headroom, so publishing can
succeed at all.

1. **Confirm nothing pins a version about to be deleted.** `sase` itself pins
   `sase-core-rs>=0.34.48,<0.35.0` (`pyproject.toml`, `uv.lock`), which is safe under
   any keep-30 policy. Before deleting, grep the other linked repos that consume the
   binding — `sase-github`, `sase-telegram`, `sase-nvim`, `sase-research-artifacts` —
   for an exact `sase-core-rs==` pin or a floor below the keep boundary. Open each with
   `/sase_repo`. If any pins a version below the boundary, raise the boundary to cover
   it rather than breaking that consumer.
2. **Recommended retention: keep the newest 30 published versions** (0.34.19 through
   0.34.48), deleting 242 older versions. That leaves 2.13 GB and frees ~8.6 GB. Keep 30
   rather than 20 so a rollback target older than the current floor still exists.
3. **Generate the delete list from live data**, never from a hand-typed list. Sort
   published versions with `packaging.version.Version`, drop the newest 30, and emit the
   remainder. Write the list to a file and have it reviewed before use.
4. **Dry-run first.** `pypi-cleanup --query-only` requires no authentication and prints
   exactly what would be deleted. The printed set must match the generated list exactly;
   if it does not, stop and re-derive it.
5. **Raise a `/sase_gate`** for the destructive run. The gate must show the full
   command, the count of versions to delete, the keep boundary, and an explicit
   statement that deletion is permanent and re-upload is impossible. The user runs it
   (or approves it running) because it needs the account password plus a live TOTP code
   — an agent cannot supply either. Do not attempt to work around this.
6. **Verify** by re-querying the PyPI JSON API: recompute total bytes and assert the
   project is under 10 GB with the expected headroom. Record the measured number.

Acceptance: total project size measured under 10 GB, keep-set intact, `0.34.48` still
present, and the measured headroom recorded for the `preflight` phase to use.

## shrink: Measure and reduce per-release wheel bytes

Goal: fewer bytes per release, without guessing and without silently dropping platform
support.

The wheel is already `--strip`ped (`--strip` in the matrix args and `strip = true` in
`[tool.maturin]`), and it contains no bundled duplicate binaries — the console scripts
(`sase_gateway`, `sase_federation_worker`, `sase_sudo_runner`) are tiny Python entry
point shims. Measured contents of the 15 MB Linux x86_64 wheel: a single
`sase_core_rs/sase_core_rs.abi3.so` at **36.3 MB uncompressed / 15.0 MB compressed**,
plus a few KB of Python. So all size work is link-time work on that one shared object.

1. **Measure before changing anything.** Build the Linux x86_64 wheel at the current
   settings and record the `.so` size as the baseline.
2. **Try, and measure individually:** `lto = "fat"` (currently `"thin"`) and
   `opt-level = "s"` / `"z"` against the current default in `[profile.release]` in the
   workspace `Cargo.toml`.
3. **Land only what is clearly worth it.** This is a performance-sensitive core library.
   Do not land an `opt-level` change that shrinks the artifact but regresses runtime
   behavior; if a size win costs measurable speed, report the tradeoff instead of taking
   it. Do **not** set `panic = "abort"` — PyO3 relies on unwinding to surface Python
   exceptions.
4. **Report, do not decide, the macOS question.** The macOS universal2 wheel (28 MB) is
   the single largest file and carries both x86_64 and arm64 slices; building arm64-only
   would save ~14 MB per release (~19% of a release) but drops Intel Mac support. That
   is a product support-matrix decision for the user, not an agent's call. Present the
   measured saving and leave the wheel as universal2 unless the user says otherwise.
5. Keep building the sdist. It is only 2 MB and it is the fallback that 0.34.48 is
   currently missing.

Acceptance: baseline and post-change `.so` sizes recorded; any landed change
demonstrably shrinks the artifact with no behavior regression; the macOS slice saving
quantified and surfaced as a question rather than applied.

## heal: Gate on file-set completeness and heal the partial 0.34.48 release

Goal: a partial upload can never again be mistaken for a published release, and the
existing backlog actually publishes.

1. **Replace the existence check with a completeness check** in the `publish-plan` job's
   `Compute publish plan` step. Instead of treating HTTP 200 on
   `/pypi/sase-core-rs/${version}/json` as "published", fetch the release's file list
   and require the full expected distribution set to be present and non-yanked:
   - `manylinux_2_28_x86_64` wheel
   - `manylinux_2_28_aarch64` wheel
   - macOS universal2 wheel
   - `win_amd64` wheel
   - sdist (`.tar.gz`)

   Treat "version exists but the set is incomplete" as `needs_publish=true`. Derive the
   expected set from one named list in the workflow so the matrix and the gate cannot
   drift apart; if the build matrix changes, that list is the single place to update.
   Note this check is scoped to the current workspace version only, so it will not start
   retroactively republishing historical releases.

2. **Confirm the heal is safe.** `Publish to PyPI` already passes `skip-existing: true`,
   so re-running an upload can only add files PyPI does not already have. Preserve that
   setting.
3. **Publish the backlog.** Once quota is reclaimed, the workspace version (0.34.66) is
   tagged-but-absent, so the normal path will build and publish it. Verify it lands with
   all five files.
4. **Heal 0.34.48 explicitly.** It is not the workspace version, so the automatic path
   will not touch it. Use the guarded manual route: `workflow_dispatch` with
   `build_wheels=true`, `publish_pypi=true`, `dry_run=false`, and
   `expected_version=0.34.48`. That rebuilds from tag `v0.34.48` and tops up the missing
   `win_amd64` wheel and sdist via `skip-existing`. Confirm 0.34.48 reports five files
   afterward.

Acceptance: the completeness gate is in place and unit-checked against a synthetic
partial-release response; 0.34.66 is published with five files; 0.34.48 reports five
files.

## preflight: Pre-flight PyPI quota guard and headroom reporting

Goal: when headroom does run low again, CI says so early and legibly instead of dying
half-way through an upload and leaving a partial release behind.

1. Add a step to the `publish` job, **before** `Publish to PyPI`, that computes current
   project size from the PyPI JSON API and compares `current + size(dist/)` against the
   10 GB limit.
2. If the upload would not fit, **fail before uploading anything** with a message that
   names the numbers and the remedy — current size, incoming size, limit, overflow, and
   a pointer to the retention runbook from `reclaim`. Failing before the first byte is
   the point: it is what prevents another partial release like 0.34.48.
3. Write remaining headroom to `$GITHUB_STEP_SUMMARY` on every publish: bytes used,
   bytes free, and approximate remaining releases at the current average release size.
   This turns a silent cliff into a visible gauge.
4. Make the limit a single named constant in the workflow, so that if PyPI later grants
   a project size limit increase, exactly one value changes.
5. Treat a PyPI API error as non-fatal for this guard (log and continue to the upload) —
   the guard exists to give a better error, and it must not itself become a new way for
   a good release to fail.

Acceptance: an over-quota condition fails the job before upload with the computed
numbers in the message; a normal publish records headroom in the step summary.

## cadence: Bound release cadence to a daily cut

Goal: turn ~4.32 releases/day into ~1/day, taking the runway from ~24 days to ~105 days
at keep-30 and making reclamation a rare event.

The chain that produces the current cadence is: master push -> `release-plz-pr` opens or
updates the release PR -> `release-plz-merge` immediately merges it -> the release
commit tags -> the matrix builds and publishes. The merge-on-every-push link is the one
to cut.

1. **Stop cutting a release on every push.** Gate `release-plz-merge` so it does not run
   for `push` events. Let the release PR accumulate commits instead — release-plz
   already keeps it updated, so nothing is lost, it is only batched.
2. **Cut once per day.** Add a dedicated daily cron alongside the existing healing cron
   and gate the merge job on that specific schedule, distinguishing them with
   `github.event.schedule`:
   - `23 */6 * * *` — existing; heals missed publishes only, does **not** merge.
   - a new daily entry — the one that merges the release PR and cuts the release.

   Guard the merge job with a condition equivalent to
   `github.event_name == 'workflow_dispatch' || github.event.schedule == '<daily cron>'`.

3. **Keep the heal path untouched.** `publish-plan` must still run on every push and
   every six-hourly cron — that is the self-healing behavior the workflow was built for
   and this phase must not weaken it.
4. **Keep an escape hatch.** `workflow_dispatch` must still cut a release on demand, so
   an urgent core fix needed for a floor ratchet (the `sase-10d` situation) is never
   stuck waiting up to 24 hours.
5. **Preserve the existing merge safety guards.** The `release-plz-merge` job's checks
   on base branch, author identity, head-branch timestamp shape, title shape, and body
   footer exist because sase-core master is unprotected. Do not relax any of them while
   changing the trigger.
6. Record the cadence decision and its reasoning in the workflow's header comment, next
   to the existing release-flow explanation, so the next reader does not "helpfully"
   restore merge-on-push.

Acceptance: a master push updates the release PR but does not publish; the daily
schedule cuts exactly one release; `workflow_dispatch` still works; the six-hourly heal
still runs.

## verify: End-to-end verification and downstream unblock

1. Confirm a full release lands through the changed path: five files present for the
   newest version, `twine check` clean, and the smoke tests green on all three
   platforms.
2. Confirm 0.34.48 now reports five files.
3. Re-measure project size and record final headroom and the projected runway at the new
   cadence. State the real number; if it came out worse than the ~105 day projection,
   say so rather than quoting the projection.
4. Confirm `Release-plz` is green by re-running `actstat` for the repo. Note that
   `actstat` is not installed on every host — it lives at `~/.cargo/bin/actstat` on
   `athena`; read the `tailnet` reference memory with `/sase_memory_read` if access
   details are needed.
5. Corroborate the downstream beads with `sase bead note` (do not close them — they are
   separately owned): `sase-10d`, `sase-12y.4`, `sase-12w.6.4.2`, `sase-11l.11.4`,
   `sase-zr.7.1.1.5.4.1`. Record which core version is now published complete so the
   floor ratchet can proceed.
6. Raising the `sase` pin floor in `pyproject.toml` is **out of scope here** — it is
   `sase-10d`'s owned work and goes through `tools/ratchet_core_window`. Leave it to
   that bead and say so in the note.

Acceptance: `Release-plz` green, both target versions complete on PyPI, measured
headroom recorded, and the blocked beads carry evidence that published core is available
again.

## Out of scope

- Requesting a PyPI project size limit increase. It is a legitimate option and worth
  doing in parallel, but it is a human request to PyPI with an open-ended turnaround, it
  cannot be relied on to fix a currently-red CI, and at 324 MB/day even a 10x increase
  is consumed in under a year without the cadence fix.
- Raising the `sase` dependency floor (owned by `sase-10d`).
- Dropping macOS Intel support — `shrink` quantifies it and surfaces it as a question.
