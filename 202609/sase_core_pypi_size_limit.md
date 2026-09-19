---
tier: tale
title: Unblock sase-core-rs PyPI publishes past the 10 GiB project limit
goal:
  Restore green Release-plz for sase-org/sase-core by getting sase-core-rs under its
  PyPI project size limit, publishing the current tagged version as a complete release,
  and failing future quota exhaustion in publish-plan instead of after the wheel matrix.
size: medium
proposed_by: bbugyi200.apollo.0r
create_time: 2026-09-19 09:37:31
status: wip
---

# Unblock sase-core-rs PyPI publishes past the 10 GiB project limit

## Outcome

- `ssh athena 'actstat --color never --repo sase-org/sase-core -n 1'` reports the latest
  master commit **success**, not `Release-plz` / `publish to PyPI` failure.
- The workspace version on `sase-org/sase-core` `master` (today `0.34.63`; re-read
  `[workspace.package].version` before publishing) is a **complete** PyPI release:
  `https://pypi.org/pypi/sase-core-rs/<version>/json` returns 200 and lists every
  artifact the workflow still builds.
- `publish-plan` refuses to start the wheel matrix when remaining PyPI quota is less
  than one full release, with a message that names the 10 GiB default, the measured
  project size, and the storage-limits docs.
- Windows wheels are no longer built or published. SASE is POSIX-only (Linux x86_64,
  Linux aarch64, macOS); historical Windows files stay on PyPI unless a later retention
  pass removes them.

## Why this is a tale

One follow-up agent can land the workflow edits in `sase-org/sase-core`, file the PyPI
project-limit request, recover a few unused `0.34.x` releases to create immediate
headroom, and re-dispatch the existing guarded publish path. There is no phase split and
no Rust or Python runtime change.

## Problem

`actstat --repo sase-org/sase-core -n 5` (run on athena) shows five consecutive master
commits red on **Release-plz**, job **publish to PyPI**, step **Publish to PyPI**. The
**CI** workflow on those same commits is green.

The failed publish log for run
`https://github.com/sase-org/sase-core/actions/runs/35439501798` is not a
trusted-publishing or artifact-guard failure. Twine accepted the `0.34.63` wheels and
sdist, then `pypa/gh-action-pypi-publish` received:

```text
HTTPError: 400 Bad Request from https://upload.pypi.org/legacy/
Project size too large. Limit for project 'sase-core-rs' total size is 10 GB.
```

PyPI JSON (`https://pypi.org/pypi/sase-core-rs/json`) on 2026-09-19:

| Field                       | Value                                                     |
| --------------------------- | --------------------------------------------------------- |
| Latest published version    | `0.34.48` (incomplete: macos + two manylinux wheels only) |
| Git tag / workspace version | `0.34.63`                                                 |
| Releases / files            | 272 / 1358                                                |
| Total size                  | **9.991 GiB**                                             |
| Remaining default quota     | **~9.1 MiB**                                              |

One complete release is ~75 MiB (macos universal2 ~28 MiB, manylinux aarch64 ~14 MiB,
manylinux x86_64 ~15 MiB, win_amd64 ~16 MiB, sdist ~2 MiB). Nothing the workflow
currently uploads fits in 9.1 MiB, so every heal run will keep failing.

`0.34.0` landed 2026-09-11 and `0.34.48` landed 2026-09-18: forty-nine `0.34.x` releases
in eight days. The self-healing publish in `.github/workflows/release-plz.yml` is doing
what it was designed to do — publish every tagged workspace version — and that cadence
filled the default 10 GiB project quota.

## Constraints

Do **not** delete Linux, macOS, or sdist files for versions that published `sase` still
pins. Live examples from PyPI:

- `sase==0.5.0` → `sase-core-rs>=0.1.1,<0.2.0`
- `sase==0.12.0` → `>=0.12.1,<0.13.0`
- `sase==0.15.0` → `>=0.17.15,<0.18.0`
- `sase==0.16.0` → `>=0.19.0,<0.20.0`
- `sase==0.17.0` / `0.17.1` → `>=0.32.16,<0.33.0`

Mass-deleting `0.32.x` or earlier floors will break `pip install sase==<old>`. Yanking
does **not** free storage.

No published `sase` depends on `0.34.x`. Local sase currently declares
`sase-core-rs>=0.34.48,<0.35.0`, and `0.34.48` is already an incomplete PyPI release.
That is why `plan:202609/artifact_link_projection_core_floor.md` cannot ratchet the
published floor to `0.34.53+` (`bead_set_link_projections`). This tale unblocks that
work by publishing a complete current version; it does not perform the sase pin ratchet.

`actstat` lives on athena (`ssh athena actstat --color never`). Do not expect it on
apollo's `PATH`.

Open the repo with `sase repo open sase-core` and edit that checkout. Do not edit
`[workspace.package].version` or crate versions; release-plz owns those.

## Implementation

Work in `sase-org/sase-core` only.

### 1. Fail publish-plan when PyPI quota cannot fit the next release

In `.github/workflows/release-plz.yml`, extend the existing `publish-plan` Python that
already probes `https://pypi.org/pypi/sase-core-rs/<version>/json`.

After it knows `needs_publish` would be true, also:

1. Fetch `https://pypi.org/pypi/sase-core-rs/json`.
2. Sum every file `size` across `releases`.
3. Compare against a workflow env default `PYPI_PROJECT_LIMIT_GIB` (string `"10"`). This
   API does not return the granted quota; when PyPI raises the limit, update that env
   value in the same workflow. Do not hard-code 10 GiB in a way that cannot be bumped.
4. Estimate the next upload as 80 MiB while Windows is still in the matrix, or 65 MiB
   once Windows is removed (step 2). Use a named constant next to the limit env, not a
   magic number in a comment.
5. If `limit - used < estimate`, **fail the job** (do not set `needs_publish=false`). A
   skip would paint Release-plz green while the tag stays unpublished. The error must
   print used GiB, limit GiB, remaining MiB, the docs URL
   `https://docs.pypi.org/project-management/storage-limits#requesting-a-project-size-limit-increase`,
   and that yanking does not free space.

Keep the existing tag-exists / PyPI-absent logic. The quota check is an extra gate on
the publish path, not a replacement for it.

There is no unit-test harness for this inline script today. Do not invent a Python
package to host it. Re-read the script after editing; a syntax error here skips every
wheel job.

### 2. Stop publishing Windows wheels

SASE's install contract is POSIX (Linux x86_64, Linux aarch64, macOS). Drop the
`windows` job from `.github/workflows/release-plz.yml` and remove it from
`metadata-check`'s `needs`. Leave historical `win_amd64` files on PyPI; deleting 271
Windows files by hand is not this tale.

Update any workflow comments that list Windows as a published artifact. Do not change
`ci.yml` (it never built Windows wheels). Do not change `release-plz.toml`.

### 3. File the PyPI project size-limit request

File `https://github.com/pypi/support/issues/new?template=limit-request-project.yml` as
the project owner (Bryan's `gh` identity):

- Title: `Project Limit Request: sase-core-rs - 50 GiB`
- Project URL: `https://pypi.org/project/sase-core-rs/`
- Project exists: yes
- New limit: `50` GiB
- Indexes: PyPI
- About: Rust/PyO3 wheels for the SASE engine (`sase-org/sase-core`), first published
  2026-06-09. abi3 wheels for CPython 3.12+ plus three bundled binaries (`sase_gateway`,
  `sase_federation_worker`, `sase_sudo_runner`).
- Each release: ~58 MiB after this tale (macos universal2 ~28, manylinux aarch64 ~14,
  manylinux x86_64 ~15, sdist ~2). Previously ~75 MiB including Windows.
- Frequency: currently every master tag via release-plz, recently several times per day.
  State that a 10 GiB default is already exhausted (9.991 GiB / 272 versions) and that
  50 GiB is requested so publishes can resume while cadence is reviewed. Do not promise
  a cadence change in this tale.

Use the issue body's required Code of Conduct checkbox. Paste the issue URL into the PR
or stitch notes.

### 4. Recover immediate headroom, then publish the current tag

Default quota will not accept any new file until something is deleted or the limit is
raised. Limit requests are not instant.

Create a confirmation gate before any PyPI deletion. Proposed safe window:

- Delete **whole releases** `sase-core-rs` `0.34.0` through `0.34.9` only.
- That series has no published `sase` pin. It should free on the order of 0.6–0.8 GiB
  (~8–10 complete publishes).
- Do **not** delete `0.34.48` (current local floor, already incomplete).
- Do **not** delete `0.32.x` or anything earlier.

Deletion is permanent. If the reviewer rejects deletion, stop the publish attempt and
wait for the pypi/support issue; do not upload into a 9.1 MiB gap.

After headroom exists, publish the **current** workspace version (re-read `Cargo.toml`;
do not assume `0.34.63` if master moved):

- The scheduled/push path already publishes when `v<version>` exists and PyPI has no
  matching release.
- If you need a manual run, `workflow_dispatch` on `master` with `dry_run=false`,
  `build_wheels=true`, `publish_pypi=true`, and `expected_version` set to that version.
  The publish job is environment `pypi` and branch-restricted to `master`.
- `skip-existing: true` is already set; a retry will top up missing files rather than
  replace them.

Do not try to backfill every unpublished tag (`0.34.49`–`0.34.62` today). `publish-plan`
only considers the current workspace version. One complete current release is enough for
the sase floor ratchet to have a valid target.

## Verification

1. `ssh athena 'actstat --color never --repo sase-org/sase-core -n 3'` — latest master
   Release-plz is success, or (if a later unrelated commit landed) the publish job is no
   longer failing with `Project size too large`.
2. `python3` against `https://pypi.org/pypi/sase-core-rs/<version>/json` — HTTP 200, and
   the file list is the post-Windows set (macos universal2, manylinux aarch64, manylinux
   x86_64, sdist).
3. Confirm `0.32.16` is still on PyPI (published sase 0.17.1 floor).
4. Confirm `ci.yml` was not modified and still passes on the PR if one is opened.
   Workflow-only changes on `master` still run CI; do not skip `just check` /
   `./scripts/check.sh` if you touch anything a check hashes, but a YAML-only workflow
   edit does not require a Rust rebuild.

## Out of scope

- Changing release-plz's every-tag GitHub release behavior.
- Slowing PyPI publish cadence (weekly / on sase pin ratchet only). Call that out if
  pypi/support asks; do not implement it here. At several 58 MiB publishes per day, even
  50 GiB is a few months of headroom.
- Deleting historical Windows wheels or pre-`0.34` Linux/macOS/sdist files.
- Ratcheting sase's `sase-core-rs` floor
  (`plan:202609/artifact_link_projection_core_floor.md`).
- Wheel-size work (LTO is already `thin`; binaries are already `--strip`).
