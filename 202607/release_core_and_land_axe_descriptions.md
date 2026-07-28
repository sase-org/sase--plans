---
tier: tale
title: Release sase-core v0.10.0, bump the sase pin, and land epic sase-9t
goal: 'The published sase-core-rs release and the sase dependency window both contain
  the sase-9t description-enforcement contract, so published/locked installs behave
  like dev installs, and epic sase-9t is closed with its plan file marked done.

  '
bead: sase-9t
create_time: 2026-07-26 11:38:44
status: done
---

- **PROMPT:** [202607/prompts/release_core_and_land_axe_descriptions.md](prompts/release_core_and_land_axe_descriptions.md)
- **PARENT:** [202607/axe_required_descriptions.md](https://github.com/sase-org/sase--plans/blob/main/202607/axe_required_descriptions.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9t.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9t.land/README.md)
  - [bbugyi200.athena.sase-9t.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9t.land.md#member-code)
- **COMMITS:**
  - [20c131b](https://github.com/sase-org/sase/commit/20c131b55788ae5d07ea32fcae66328c48e748ab) — build(deps): require sase-core-rs 0.10 (sase-9t)

# Plan: Release sase-core v0.10.0, bump the sase pin, and land epic sase-9t

## Context

Epic sase-9t ("Require descriptions for every AXE lumberjack and chop") is code-complete at source level across all four
repos (sase, sase-core, chezmoi, bugyi-chops) and all six phase beads are closed. Verification by the land agent
confirmed every phase's changes landed and 68 scoped AXE tests pass against a locally built core.

However, the epic plan's Phase 1 step 5 ("bump the workspace version in Cargo.toml and release so the sase repo can
depend on it") was never completed:

- sase-core master commit `8b76c42` (`feat(axe): support required config descriptions (sase-9t.1)`) is **unreleased**.
  The latest PyPI `sase-core-rs` is 0.9.2, published 2026-07-25 — before that commit existed.
- sase pins `sase-core-rs>=0.9.2,<0.10.0` in pyproject.toml and `uv.lock` resolves registry 0.9.2, so any
  published-wheel or lock-driven environment (e.g. bare `uv run` / `uv sync`) pairs sase HEAD with a core that lacks the
  feature. That pairing is demonstrably broken, not just unenforced:
  - `src/sase/core/axe_chop_facade.py:validate_axe_config` always includes `require_descriptions` in the request, and
    0.9.2's `AxeConfigValidationRequestWire` has `#[serde(deny_unknown_fields)]` → every call (e.g. `for_each` target
    validation in `src/sase/axe/_config_targets.py`) raises `ValueError: unknown field `require_descriptions``.
  - 0.9.2 does not know the lumberjack `description` key, so every shipped config (including
    `src/sase/default_config.yml`) fails composition with `unknown_key` diagnostics.
  - 0.9.2's compose/mutation wires silently ignore `require_descriptions`, so even if the above were dodged the
    enforcement would be a no-op.
- Dev workspaces only work because `just install` builds `sase_core_rs` editable from the local sase-core checkout.

sase-core releases are fully automated by release-plz (`.github/workflows/release-plz.yml`): merging conventional
commits to master maintains an open release PR; merging that PR makes the release job tag `v<version>`, create the
GitHub release, build the wheel matrix, and publish to PyPI. Publishing is self-healing: every push and a 6-hourly
scheduled run re-check "the tagged workspace version is missing from PyPI" and finish an interrupted publish. Manual
version edits are blocked by `pr-title.yml`; do not hand-edit versions in sase-core.

Current release state (as of 2026-07-26 ~15:45Z):

- Release PR **sase-org/sase-core#31** ("chore: release v0.10.0", branch `release-plz-2026-07-25T17-19-48Z`) is open but
  **stale**: last refreshed 11:53Z, before `8b76c42` landed on master (13:04Z). Its changelog lists sase-9s.2,
  sase-9q.1, sase-9m.1 (breaking), sase-9n.1, and sase-96.8.6 but not sase-9t.1.
- The on-push release-plz run for `8b76c42` (run id 30203324437) and the following scheduled run both failed with GitHub
  API rate limiting (HTTP 403, "API rate limit exceeded"). Nothing is structurally broken; the runs just need to be
  repeated once the limit resets.

The next release is v0.10.0 (master contains `feat(editor)!`, a breaking change), which falls outside sase's current
`<0.10.0` window — so the sase pin must be bumped after the release lands. sase master already runs against sase-core
master in dev (editable build), so no sase-side code adaptation is expected from the other unreleased core commits.

Repo access rule: open sase-core with the `/sase_repo` skill and use only the path it prints. GitHub PR/issue/workflow
state may be inspected with `gh`.

## Step 1 — Refresh and land the sase-core v0.10.0 release

1. Open sase-core via `/sase_repo`. Confirm `origin/master` still has `8b76c42` in `git log` and that no release tag
   contains it yet (`git tag --contains 8b76c42` is empty).
2. Get the release PR refreshed so v0.10.0's changelog includes `8b76c42`:
   - Preferred: rerun the failed release-plz run (`gh run rerun 30203324437 --repo sase-org/sase-core`) or wait for the
     6-hourly self-heal schedule (`23 */6 * * *` UTC) to run green. The GitHub API rate limit that killed the earlier
     runs is shared and hourly; if `gh` returns 403, back off and retry later rather than polling tightly.
   - Confirm PR #31's body gains a `### Added` line for `*(axe)* support required config descriptions (sase-9t.1)` and
     the version stays 0.10.0.
3. Merge PR #31 (normal merge flow for release PRs; its title already satisfies the conventional-commit check).
4. Watch the release-plz workflow run triggered by the merge push: it must tag `v0.10.0`, create the GitHub release,
   build the wheel matrix, and publish to PyPI. If the run dies partway, rerun the workflow or wait for the scheduled
   self-heal run — the publish gate is "tagged version missing from PyPI", so retries are safe.
5. Verify the publish: `curl -s https://pypi.org/pypi/sase-core-rs/json` reports `info.version == "0.10.0"` with a wheel
   set matching 0.9.2's platforms (manylinux x86_64 + aarch64, macOS universal2, Windows amd64, plus sdist).

## Step 2 — Bump the sase dependency window

In the sase repo (this workspace):

1. `pyproject.toml`: change the `sase-core-rs` specifier from `>=0.9.2,<0.10.0` to `>=0.10.0,<0.11.0`.
2. Refresh the lock so it resolves registry 0.10.0 (e.g. `uv lock`; keep all other pins as-is unless resolution forces
   them).
3. Run `just install` and confirm the `[setup] WARNING: bump the published sase-core-rs window` message does NOT appear
   (that warning fires when the local core version falls outside the pyproject window).
4. Prove the published pairing works — the exact failure mode this plan fixes: in a throwaway venv outside the project
   flow, install the published wheel
   (`uv venv /tmp/core-probe && uv pip install --python /tmp/core-probe/bin/python sase-core-rs==0.10.0`) and check that
   `sase_core_rs` accepts a validation request containing `"require_descriptions": true` and a lumberjack `description`
   key without `unknown field` / `unknown_key` errors.
5. Run `just check` and fix anything it reports.
6. Commit the pin bump + lock refresh with the `/sase_git_commit` skill, referencing this tale's bead ID in the commit
   message.

## Step 3 — Land epic sase-9t (final phase)

All six phase beads of sase-9t are already closed; only the epic remains open.

1. `sase bead close sase-9t` with a reason noting the release + pin bump completed the last integration gap.
2. AFTER closing, run `just symvision` (epic-symbol whitelist entries for sase-9t expire at close). Remove the stale
   whitelist entries and any unused code it reports, rerun `just check`, and commit those cleanups via
   `/sase_git_commit` if any files changed.
3. Open the plans sidecar via `/sase_repo` (`sase repo open plans -r ...`) and set `status: done` in the YAML
   frontmatter of `202607/axe_required_descriptions.md` (the PLAN path shown by `sase bead show sase-9t`). Commit that
   change in the plans repo per its normal workflow.

## Acceptance

- PyPI `sase-core-rs` latest is 0.10.0 and its release tag contains commit `8b76c42`.
- sase `pyproject.toml` pins `sase-core-rs>=0.10.0,<0.11.0` and `uv.lock` resolves registry 0.10.0.
- The published 0.10.0 wheel accepts `require_descriptions` and lumberjack `description` keys (step 2.4 probe).
- `just check` passes in the sase workspace.
- Bead sase-9t is CLOSED, `just symvision` is clean with no stale sase-9t entries, and the epic plan file's frontmatter
  reads `status: done`.

## Risks

- **GitHub API rate limiting** (shared 5,000/hr) already killed two release runs. Prefer widely spaced retries and the
  scheduled self-heal run over tight polling loops.
- **v0.10.0 ships other epics' core changes** (sase-9m.1 breaking editor change, sase-9n.1, sase-9q.1, sase-9s.2). This
  is expected — sase master already runs against core master in dev. If `just check` surfaces an incompatibility after
  the pin bump, it predates this plan; report it rather than papering over it.
- **Stale editable-core builds**: uv caches the editable `sase_core_rs` build keyed by version, so a workspace can
  silently run an old 0.9.2 build. After the release, the version moves to 0.10.0 and rebuilds occur naturally; if a
  test run shows `unknown_key` on `description`, rerun `just install` before debugging further.
