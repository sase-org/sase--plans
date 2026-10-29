---
tier: tale
title: Unblock sase-core's PyPI publish for 0.43.1 and 0.43.2
goal:
  The tagged-but-unpublished sase-core-rs 0.43.1, then the daily-cut 0.43.2, publish
  completely to PyPI through the existing self-healing release workflow, with no manual
  dispatch.
size: small
decisions:
  maturin_pin:
    ask:
      Should the release workflow pin maturin to 1.16.0, the version that built the
      0.43.1 artifacts?
    choices:
      float:
        Builds keep the newest maturin 1.x from pyproject; the unused MATURIN_VERSION
        line is deleted
      pin:
        Release builds stay on maturin 1.16.0 until someone bumps MATURIN_VERSION by
        hand
    default: float
    why:
      The tag-order-independent matcher removes this failure class; a pin is one more
      manual bump
    answer: float
proposed_by: bbugyi200.athena.160
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 18:05:59
status: wip
---

# Unblock sase-core's PyPI publish (0.43.1 / 0.43.2)

## Goal

sase-core's next versions reach PyPI. `v0.43.1` was tagged on 2026-10-29 but is still
not on PyPI; the newest published `sase-core-rs` is `0.43.0`. Two defects in the
sase-core release workflow block it. Every push and heal run hits them again, and the
next daily cut (`v0.43.2`, release PR #341) would hit them too. Fix both defects in the
linked `sase-core` repo. The existing self-healing publish loop then ships the release
with no manual dispatch.

Open the repo with `sase repo open sase-core -r "<why>"` and work only in the printed
path. Read its `AGENTS.md` first. All paths below are relative to that checkout.

## Evidence (gathered 2026-10-29)

- **Attempt 1 was killed by the macOS timeout.** Release-plz run `40900244522` (the push
  from merging release PR #340) timed out its `macos universal2` job with "The job has
  exceeded the maximum execution time of 45m0s", so `twine check` and `publish` were
  skipped. Phase bead `sase-1pi.1` note #6 records this as a PROPOSED FOLLOW-UP. No task
  bead was filed for it.
- **Attempt 2 was refused by the dist gate.** In push run `40920471342` (at `1762b33a`)
  every build and `twine check` passed. Then the `Guard PyPI publish` step failed at
  `pypi_release_files.py dist`:

  ```text
  - no file matches *universal2.whl
  - sase_core_rs-0.43.1-cp312-abi3-macosx_10_12_universal2.macosx_10_12_x86_64.macosx_11_0_arm64.whl matches no expected entry
  ```

  `PyO3/maturin-action` resolved maturin **v1.16.0** from
  `crates/sase_core_py/pyproject.toml` (`maturin>=1.7,<2.0`). `0.43.0` was built with
  **v1.15.0**, which named the wheel
  `...-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl`. Maturin 1.16
  sorts the compressed platform tags, so the file name no longer ends with the
  `universal2.whl` entry in `EXPECTED_DIST_SUFFIXES`. The workflow-level
  `MATURIN_VERSION: "1.7"` env var is dead: nothing reads it.

- **It repeats on every run.** Run `40957590177` (push of `f4aa8fb6`) is rebuilding
  `0.43.1` now and will fail the same way.
- **The macOS job runs close to its limit.** Over the last week, successful macOS
  universal2 builds took 26.6–43.8 min, and 6 of 16 attempts hit the 45-minute limit.
  The universal2 build compiles twice (arm64, then x86_64). Linux jobs get 60 minutes.
- **The same matcher decides PyPI completeness.** `matches()` also backs `status` (the
  `publish-plan` gate) and `pypi_retention.py` (`classify`). If only the dist check were
  loosened, a 1.16-built release on PyPI would read as `partial` forever. Every run
  would rebuild it, and retention would never count it as a complete release.
- **What happens after the fix.** Release PR #341 (`chore: release v0.43.2`) is open and
  green, and the daily cut (`41 7 * * *` UTC) merges it. sase's `sase-core-revision.txt`
  pins `1762b33a`, and only `0.43.2` contains it. The sase release steward reports
  `core_lag` with merge authority off, so it will not dispatch a cut itself.

## Changes

### 1. Match wheel platform tags in any order: `.github/scripts/pypi_release_files.py`

Rewrite `matches(filename, version, suffix)`. Keep it stdlib-only, and keep the public
names and signatures (`parse_suffixes`, `classify`, `dist_problems`, `ReleaseFiles`, the
state constants). `pypi_retention.py` and the tests import them.

- **Wheel** (`filename` ends with `.whl`): parse
  `{name}-{version}[-{build}]-{py}-{abi}-{plat}.whl`. Require `name == FILENAME_STEM`
  and an exact match on the version component. Today's `startswith(stem(version))` also
  accepts `0.43.10` files for `0.43.1`. The file matches when `suffix` ends in `.whl`
  and `tag + ".whl"` ends with `suffix` for at least one `tag` in `plat.split(".")`.
- **Anything else** (the sdist): match only `filename == stem(version) + suffix`.
- A malformed wheel name matches nothing. It does not raise.

Update the module docstring to state the rule. A wheel entry names one platform tag, not
the file's literal tail, so maturin's tag order does not matter.

### 2. Tests: `.github/scripts/test_pypi_release_files.py`

Add cases next to the existing ones, reusing `VERSION` and the fixture style:

- the maturin 1.16 macOS name
  (`macosx_10_12_universal2.macosx_10_12_x86_64.macosx_11_0_arm64.whl`) gives `classify`
  → `complete` and `dist_problems` → `[]`. The existing 1.15-ordered `MACOS` fixture
  still passes too.
- an arm64-only macOS wheel (`...-macosx_11_0_arm64.whl`) does not satisfy
  `universal2.whl`, so `classify` → `partial` and `dist_problems` reports the gap and
  the stray file.
- a version-prefix neighbour (files for `VERSION + "0"`) never matches `VERSION`, for
  either a wheel or the sdist.
- `WorkflowWiringTests`: the workflow's real `EXPECTED_DIST_SUFFIXES` accepts a complete
  set that uses the 1.16 macOS name.

### 3. Workflow: `.github/workflows/release-plz.yml`

- `macos` job: `timeout-minutes: 45` → `75`.
- `metadata-check` job: `timeout-minutes: 60` → `75`. Its comment says to keep it
  matched to the longest wheel job, so change "(60m)" in that comment to "(75m)".
- Update the comment above `EXPECTED_DIST_SUFFIXES`. It currently says each entry is
  "the tail of the file name after the version". Change that to the new rule: a wheel
  entry matches one of the wheel's compressed platform tags, and `.tar.gz` is the
  sdist's tail. Keep the block itself unchanged: one entry per line, no comments inside.
  `test_pypi_release_files.py` and `pypi_retention.read_expected_suffixes` parse it.

> [!decision] maturin_pin = float Delete the unused `MATURIN_VERSION: "1.7"` line from
> the workflow `env:`. Nothing else changes. `maturin-action` keeps resolving maturin
> from pyproject.

> [!decision] maturin_pin = pin Set `MATURIN_VERSION: "v1.16.0"` and pass
> `maturin-version: ${{ env.MATURIN_VERSION }}` to all four `PyO3/maturin-action@v1`
> steps (linux, macos, windows, sdist). Add a line to the env comment: bump it
> deliberately, and run the macOS build before a cut.

### 4. Docs: `docs/pypi-retention.md`

In "Healing a partial release", add one bullet under the `EXPECTED_DIST_SUFFIXES`
bullets. It says that a wheel entry matches a single compressed platform tag in any
order, and cites the 0.43.1 incident (maturin 1.16 reordered the universal2 tags, and
the dist gate refused a good release). Do not touch any `CHANGELOG.md`, `version` field,
or path-dependency pin; release-plz owns them.

## Verification (all in the sase-core checkout)

1. `./scripts/check.sh script-test` passes.
2. **Real artifacts.** Download the refused run's distributions into a new, empty
   scratch directory that is not inside any repo:
   `gh run download 40920471342 --repo sase-org/sase-core -D <scratch>/raw`. Copy every
   file into `<scratch>/dist`. With `EXPECTED_DIST_SUFFIXES` exported from the
   workflow's block, run
   `python3 -I .github/scripts/pypi_release_files.py dist <scratch>/dist 0.43.1`. It
   must print `verified 5 distributions`. Then run the Name/Version check from the
   workflow's `Guard PyPI publish` heredoc against those files with
   `EXPECTED_VERSION=0.43.1`. Save it as a script outside `<scratch>`, run it with
   `python3 -I`, and pass the dist path. This proves the next publish clears the whole
   guard, not just the suffix check.
3. Quota pre-flight, read-only:
   `PYPI_PROJECT_LIMIT_BYTES=10737418240 python3 .github/scripts/pypi_quota.py check <scratch>/dist --version 0.43.1`
   reports that the upload fits.
4. Live PyPI, read-only: `pypi_release_files.py status 0.43.0` prints `complete` (the
   old tag order still counts), and `status 0.43.1` prints `absent`.
5. `sase tool run check` (the guarded `just check`) passes. Give it a 10+ minute
   timeout.

Delete `<scratch>` afterwards.

## Bead bookkeeping

Phase `sase-1pi.1` ("a core containing the fix publishes") is waiting on this release.
Append one `sase bead note sase-1pi.1` that says:

- attempt 2 (run `40920471342`) was refused by the dist gate because of the maturin 1.16
  tag order, a second blocker its notes do not record yet;
- this change fixes that and also raises the macOS timeout, which resolves note #6's
  proposed follow-up;
- after landing, the next push run publishes `0.43.1` and the daily cut publishes
  `0.43.2`.

Do not close or update any `sase-1pi*` bead, and file no task bead for the timeout. This
change fixes it.

## Landing

The host commits the sase-core change. Use the subject
`fix(release): accept maturin 1.16 wheel tag order and give the macOS wheel 75 minutes`.
Do not dispatch `release-plz.yml`, merge or edit PR #341, rerun runs, or change any pin
or setting. Once the commit is on master, the release finishes by itself:

- The fix's own push run gets `needs_publish=true` from `publish-plan` (`v0.43.1`
  tagged, absent on PyPI). It builds from tag `v0.43.1` and publishes with the fixed
  workflow and scripts.
- The 07:41 UTC daily cut merges #341. Its push run then tags and publishes `0.43.2`,
  which contains sase's pinned `1762b33a`.
- If the cut merges #341 before this fix lands, the fix's push run publishes `0.43.2`
  instead, and `0.43.1` stays tagged but unpublished. That is acceptable: no consumer
  pins it. Do not backfill it.

## Out of scope

Making the macOS build faster (for example, per-arch macOS jobs), the steward's merge
authority, ratcheting sase's `sase-core-rs` pin, and PyPI quota reclaims.
