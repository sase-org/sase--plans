---
create_time: 2026-07-12 16:49:33
status: wip
prompt: 202607/prompts/finish_toolong_epic.md
tier: tale
---
# Plan: Finish Epic sase-5r — Publish `bbugyi-toolong` v0.1.0 and Close Out

## Context

Epic `sase-5r` ("Factor pylimit into toolong and migrate sase") is nearly done. A verification pass on 2026-07-12
confirmed:

- **Phase 1 (`sase-5r.1`, closed)** — verified complete: `bbugyi200/toolong` master has the full CLI
  (`cli.py`/`core.py`/`scanner.py`, `py.typed`), 15 tests, correct packaging metadata, green CI.
- **Phase 2 (`sase-5r.2`, closed)** — verified complete: CI / pr-title / publish workflows, release-please config,
  `pypi` GitHub environment, polished README. Its note about release PR #1 being left open was addressed (merged).
- **Phase 3 (`sase-5r.3`, in progress)** — release PR #1 merged; GitHub release + tag `v0.1.0` exist; the
  `release`/`build`/`install-smoke` jobs of Publish run 29207782700 are green. **The `publish` job fails with PyPI
  `invalid-publisher`**: the pending trusted publisher for `bbugyi-toolong` is not configured on pypi.org (re-confirmed
  via a fresh re-run, attempt 2, at 2026-07-12 20:46 UTC). PyPI returns 404 for `bbugyi-toolong`.
- **Phase 4 (`sase-5r.4`, in progress)** — migration commit `9b5467089` is on sase master and matches the epic plan
  exactly (Justfile `_lint-toolong`/`toolong` recipes, `bbugyi-toolong>=0.1.0,<0.2.0` dev dep, xprompt subprocess swap,
  vendored `tools/pylimit-260221` + `tools/pylimit_files-260227` deleted, tests/docs updated). `lib/bugyi-260221.sh` was
  intentionally retained because protected memory shims (`tools/CLAUDE.md` and siblings) still reference it — per the
  epic plan this is reported as a user follow-up, not changed. `just install` / `just check` cannot pass until the
  package is installable from PyPI.

Everything left is serialized behind one human prerequisite.

## Human Prerequisite (Bryan — blocks step 2)

On pypi.org, add a **pending trusted publisher** for project name `bbugyi-toolong`:

- Owner: `bbugyi200`
- Repository: `toolong`
- Workflow: `publish.yml`
- Environment: `pypi`

The failed run's OIDC claims confirm these are exactly the values GitHub presents.

## Steps

1. **Notify Bryan and wait for confirmation** that the pending trusted publisher (settings above) has been added on
   pypi.org. Do not switch to token-based auth under any circumstances (explicit epic-plan constraint).
2. **Re-run the failed publish job**: in `~/projects/github/bbugyi200/toolong/`, run `gh run rerun 29207782700 --failed`
   and watch it to green (`gh run watch`). If it fails again with a trusted-publisher error, stop and report the exact
   error to Bryan.
3. **Verify the release end-to-end** (epic plan Phase 3, step 3): in a scratch venv,
   `pip install bbugyi-toolong==0.1.0`, run `toolong --help`, and run a real scan with `--include '*.rs'` against a
   directory containing Rust files to prove the language-agnostic path works from the published artifact. Confirm
   `https://pypi.org/pypi/bbugyi-toolong/json` now resolves.
4. **Verify the sase migration** (epic plan Phase 4, step 6): in the sase workspace run `just install` (dep set
   changed), then `just check`; fix anything red. No new commit is expected — commit `9b5467089` already landed the
   migration.
5. **Close the phase beads**: `sase bead update sase-5r.3 --status closed` and
   `sase bead update sase-5r.4 --status closed`, updating each bead's notes to reflect the resolved blocker.
6. **Close the epic bead** (final work item): `sase bead close sase-5r` (fall back to
   `sase bead update sase-5r --status closed` if `close` is not a valid subcommand).
7. **Post-close dead-code sweep**: run `just pyvision` (some symbols are only whitelisted while an epic is open) and fix
   any unused code it reports.
8. **Mark the epic plan file done**: set `status: done` in the frontmatter of `<plans-dir>/202607/toolong_extraction.md`
   (resolve `<plans-dir>` via `sase sdd path plans`; currently `status: wip`).

## Non-Goals

- No changes to `lib/bugyi-260221.sh` or the protected `tools/` memory shims (requires explicit user permission;
  reported as a follow-up instead).
- No chezmoi changes; no rename of the `pylimit_split` xprompt.
