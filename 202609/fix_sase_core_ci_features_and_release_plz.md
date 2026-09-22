---
tier: tale
title: Fix sase-core CI feature-drift false positives and the release-plz PR failure
goal:
  sase-core master is green again. The CI "unified dependency features" step passes on
  ubuntu and macOS with CARGO_TERM_COLOR=always, and the Release-plz "Release-plz PR"
  job succeeds instead of failing on the untagged sase_workspace_hack crate.
size: small
proposed_by: bbugyi200.athena.0po
create_time: 2026-09-22 18:44:12
status: wip
---

# Fix sase-core CI: false feature-drift failures and the broken release-plz PR job

## Context

Every sase-core `master` push since `1d129cd` ("feat(fast-loop): unify features via
workspace-hack, add drift gate, just fast, true MSRV 1.89") has been red in **both**
workflows. `actstat --repo sase-org/sase-core -n 5` shows the same two failures on every
commit:

1. **CI → `cargo fmt + clippy + test` (ubuntu-latest and macos-latest), step "unified
   dependency features"** (`./scripts/check.sh features`).
2. **Release-plz → "Release-plz PR", step "Run release-plz"**
   (`release-plz release-pr`).

The two failures have separate root causes, but the same commit introduced both. Both
were reproduced locally and both fixes were checked against throwaway copies (details
below).

All work happens in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<reason>"` and work only in the printed path. Read its
`AGENTS.md` first. Do not edit any version field, path-dependency version pin, or
`CHANGELOG.md`, because release-plz owns those.

### Root cause 1: the drift gate parses ANSI-colored `cargo tree` output

`.github/workflows/ci.yml` sets `env: CARGO_TERM_COLOR: always` for the whole workflow.
The `cmd_features` Python heredoc in `scripts/check.sh` shells out to
`cargo tree ... --prefix none -f "{p} {f}"` and `cargo metadata` through
`run_cargo(args)` without disabling color. With forced color, cargo wraps the dedupe
marker in escape codes (`\x1b[33m\x1b[2m(*)\x1b[39m\x1b[22m`). `parse_tree` removes only
the literal `(*)`, so the leftover escape codes are joined onto the last feature token
(`'util \x1b[33m\x1b[2m\x1b[39m\x1b[22m'`) or become a bogus feature of their own. The
workspace-wide tree prints many more deduplicated lines than a single-member tree, so
nearly every package reports a phantom "workspace-only" feature. The CI log is full of
entries like:

```
error: feature drift for tower v0.5.3 in -p sase_core:
  workspace-only: ['util \x1b[33m\x1b[2m\x1b[39m\x1b[22m']
```

This is a false positive. The workspace-hack is correctly generated, and the error's
advice to run `cargo hakari generate` is a red herring. Local runs pass because cargo
does not color output when stdout is a pipe.

Verified locally:

- `./scripts/check.sh features`: passes.
- `CARGO_TERM_COLOR=always ./scripts/check.sh features`: fails with the exact CI output.
- A copy of the heredoc with `["cargo", "--color", "never", *args]` in `run_cargo`, run
  with `CARGO_TERM_COLOR=always`: passes.

Consequence: CI stops at step 8, so **clippy and tests have not run in CI since
`1d129cd`**, on either OS. A full local `just check` (through `sase tool run`) on
current master passes on Linux, but the macOS leg has had no signal since then.

### Root cause 2: release-plz git_only mode looks up a tag for the new workspace-hack crate

`release-plz.toml` sets `git_only = true` and a workspace-wide
`git_tag_name = "v{{ version }}"`. In release-plz 0.3.168 and 0.3.169 (`next_ver.rs`,
`next_versions` → `collect_git_only_packages` → `process_git_only_package`), **every**
workspace member is sent through the git_only tag lookup, including `release = false`
members. The upstream HEAD source is unchanged, so no newer release-plz will fix this.
For each member, release-plz picks the latest tag matching the package's tag template,
checks out that commit in a temp worktree, and loads the package from it.

`crates/sase_workspace_hack` (added in `1d129cd`, after the `v0.34.72` release commit)
inherits `v{{ version }}`, so it matches tag `v0.34.72`. That commit has no such crate,
so release-plz aborts:

```
INFO Latest release of package sase_workspace_hack: tag `v0.34.72` (version 0.34.72)
ERROR failed to update packages
    0: failed to determine next versions
    1: cannot find package "sase_workspace_hack" in workspace ".../worktree"
```

This never clears on its own: no release PR can be opened or updated, so no new `v*` tag
is cut. The daily release cut is blocked.

Verified locally with `release-plz update` (the same `next_versions` path) in a
throwaway clone. Without the fix it fails identically. After adding a package-specific
tag template to the `sase_workspace_hack` entry, it logs
`No release tag found matching pattern ^sase_workspace_hack\-v(\d+\.\d+\.\d+)$ ... treated as initial release`
and bumps `sase_core` / `sase_core_py` 0.34.72 → 0.34.73. The changelog lists exactly
the six commits since `v0.34.72`, and the hack crate is left untouched.

This is safe because the updater (`Project::new` / `packages_to_process`) only iterates
release-enabled packages, and `sase_workspace_hack` is not in any `changelog_include`.
The only effect of the new template is that the tag lookup finds nothing. Do **not**
apply the same change to `sase_gateway`. `sase_core`'s `changelog_include` pulls in its
commits, and making it an "initial release" risks dumping its whole history into the
changelog. `sase_gateway` and `sase_xprompt_lsp` already exist at `v0.34.72`, so their
lookups succeed.

## Changes

### 1. `scripts/check.sh`: force colorless cargo output in the drift gate

In the `cmd_features` Python heredoc, change `run_cargo` so every cargo invocation (both
`cargo metadata` and every `cargo tree`) runs with color disabled:

```python
    proc = subprocess.run(
        ["cargo", "--color", "never", *args],
        ...
```

Add a short comment above it saying why. The parser reads `cargo tree` text, and a
caller's `CARGO_TERM_COLOR=always` (as CI sets) or a cargo `term.color` config would
otherwise wrap the `(*)` dedupe marker in ANSI escapes that the parser counts as
features. Keep `CARGO_TERM_COLOR: always` in `ci.yml`, because colored logs are
intended. The fix belongs in the checker so it works in any environment.

Optional hardening, only if it stays tiny: after `.replace("(*)", "")`, strip any
remaining ANSI CSI sequences (`re.sub(r"\x1b\[[0-9;]*m", "", line)`) in `parse_tree`.
`--color never` is the primary fix. Do not replace it with stripping.

### 2. `release-plz.toml`: give `sase_workspace_hack` its own tag template

In the existing `[[package]] name = "sase_workspace_hack"` entry (currently only
`release = false`), add:

```toml
git_tag_name = "sase_workspace_hack-v{{ version }}"
```

Add a comment on that entry explaining the reason: git_only mode runs a tag lookup for
every workspace member, even `release = false` ones, and loads the package from the
matched tag's tree. This crate never gets tagged, so its template must match no existing
tag, or `release-pr` fails with "cannot find package". Do not touch any other key or
package entry.

### 3. `AGENTS.md`: one-line guard against recurrence

Under "Conventions" → the release-plz bullet list, add one concise sub-bullet with the
non-obvious rule. A new workspace crate needs a `[[package]]` entry in
`release-plz.toml`. If release-plz never releases it (`release = false`), it also needs
its own `git_tag_name` (for example `"<crate>-v{{ version }}"`); otherwise git_only
`release-pr` looks the crate up at the last `v*` tag, fails, and blocks every release.
Match the file's existing tone and wrapping (about 88 columns).

## Verification

Run everything from the opened sase-core checkout:

1. `CARGO_TERM_COLOR=always ./scripts/check.sh features` passes (it fails before the
   fix), and plain `./scripts/check.sh features` still passes.
2. Confirm the gate still catches real drift. For example, temporarily comment out one
   feature-bearing line in the HAKARI section of `crates/sase_workspace_hack/Cargo.toml`
   (such as the `tower` line), run
   `CARGO_TERM_COLOR=always ./scripts/check.sh features`, and see real drift errors with
   no `\x1b` in them. **Revert the edit** and confirm `git diff` shows no change to that
   file.
3. Release-plz: if `release-plz` is installed (`release-plz --version`), clone the
   checkout into a temp dir (`git clone -q <checkout> "$tmp"` and fetch its tags),
   commit the change there, and run `release-plz update`. It must succeed and log that
   `sase_workspace_hack` is "treated as initial release". Never run `release-plz` inside
   the real checkout, because it rewrites versions and changelogs.
4. Run the full gate once: `sase tool run -- just check`. sase-core has no named `check`
   tool, so use the ad-hoc form with a timeout of 15 minutes or more. It must pass.
5. After the host commits and pushes, confirm with
   `actstat --repo sase-org/sase-core --color never` (wait with `/sase_monitor` if
   needed) that the **CI** workflow's `unified dependency features` step passes on both
   OSes and the **Release-plz** "Release-plz PR" job succeeds, opening or updating a
   `chore: release v0.34.73` PR.

   The features step has hidden clippy and tests on macOS since `1d129cd`. If the macOS
   (or Linux) clippy or test step now fails, diagnose and fix it the same way in a
   follow-up change. Do not weaken assertions, and check `sase bead list -T flake`
   before treating a failure as a flake.

## Commit

The host creates the commit. Use a Conventional Commit subject, e.g.
`fix(ci): disable cargo color in feature-drift gate and give workspace-hack its own release-plz tag`.
Do not add a `BREAKING CHANGE` footer. This touches no wire types or bindings, so no
sase `sase-core-revision.txt` pin move is needed.

## Out of scope

- sase's own Master Gate failures (`test (N)` shards and `lint` → "Check pinned core
  bindings") and sase-telegram's CI failure, which also appear in `actstat` output.
- The Node 20 deprecation warnings on `actions/checkout@v4` / `setup-python@v5`
  (warnings only).
