---
tier: tale
title:
  Unblock the bob-cli release train by fixing the four macOS-only props test failures
goal:
  bob-cli master CI and the scheduled release train pass on macOS again, so a release
  containing 37b2080e and c25d92fd ships and the stalled bob-cli-ph.3 reader-rollout
  monitor can finish on its own.
size: small
decisions:
  lock_probe:
    ask:
      How should the lock-wait tests detect that the child opened vault.lock on macOS?
    choices:
      portable:
        Keep /proc on Linux and poll lsof by inode on macOS; macOS keeps lock-wait
        coverage
      linux_only:
        Gate the three lock-wait tests to Linux; simpler, but macOS loses that coverage
    default: portable
    why: The lock-wait replan path is shared code; macOS users hit it too
    answer: portable
proposed_by: bbugyi200.apollo.fw
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-24 08:50:03
status: wip
---

# Unblock the bob-cli release train: fix four macOS-only `props` test failures

## Why this plan exists (stall diagnosis)

`bob-cli-ph.3` (the reader-rollout phase of the `bob-cli-ph` true-case drawers epic) has
finished its implementation (`fd99ebb8`). Its last acceptance gate needs a published
`vX.Y.Z` bob-cli release whose history contains both drawer-reader commits `37b2080e`
and `c25d92fd`. Its monitor `bob-cli-ph.3--mon` (monitor id `eh82bkngbp55`, 36h cap,
started 2026-11-24 12:05 UTC, so it expires around 2026-11-26 00:05 UTC) polls tags
every 5 minutes. The newest tag is still `v0.6.0`.

That release cannot happen as things stand:

- Every scheduled `release.yml` run since 2026-11-23 fails only on
  `ci / test (macos-latest)`, then skips `release` and `binaries`. Examples: run
  45257022821 at `bd67dfc8` and run 45277877777 at `588c6210`.
- Every master `ci.yml` push run fails the same way. The latest is run 45302376590 at
  `8d82c621`, the current master tip.
- The same four tests fail on every run, so they are deterministic failures, not flakes:
  - `props::absolute_note_with_relative_bob_dir_stays_confined`
  - `props::lock_wait_preserves_formatting_only_frontmatter_edit`
  - `props::lock_wait_schema_change_owns_a_previously_loose_field`
  - `props::lock_wait_semantic_edit_is_stale`
- All four tests arrived in `39661003` ("feat(props): repair ownership, reversible
  writes, and CLI contract"), which landed after `v0.6.0`.
- No bead or agent owns the fix. The only record is a `PROPOSED FOLLOW-UP` note on
  `bob-cli-ph.3`. Normally `bob-cli-ph.land` routes those notes, but it waits on
  `bob-cli-ph.3`. Phases `bob-cli-ph.4` through `bob-cli-ph.11` are also queued behind
  it. Without this fix the chain is deadlocked.

All work below is in the **bob-cli** repository. Open it with your `/sase_repo` skill
(`sase repo open bob-cli -r "<reason>"`), read its `AGENTS.md`, and use only the printed
path. The host commits opened repositories like your own checkout.

## Root cause A: product bug in absolute-note confinement (1 test)

`resolve_target` in `src/native/props/cli.rs` makes `--bob-dir` absolute through
`absolute_path`, which calls `std::env::current_dir()`. On macOS, `getcwd` returns the
resolved `/private/var/folders/...` path, while the absolute NOTE argument keeps the
`/var/folders/...` spelling (`/var` is a symlink to `/private/var`). The lexical
containment check rejects the pair with `prop_traversal` ("note path escapes the
vault"):

```rust
lexical_normalize(&joined).starts_with(&lexical_normalize(vault))
```

The same bug hits any user whose vault and note spell a symlinked ancestor differently,
for example a symlinked `~/bob` or `$HOME`. The later canonical check already proves
real containment and reports `prop_symlink_escape`, so the lexical check is too strict,
not the last line of defense.

### Change

In `resolve_target`, keep the lexical check as the fast path. When it fails, do not
refuse immediately:

1. Canonicalize the deepest existing ancestor of the lexically-normalized target (in
   practice its parent directory). Accept the target if that canonical ancestor is
   inside `vault_canonical`, and continue to the existing `symlink_metadata`, canonical,
   and `strip_prefix` steps unchanged.
2. Otherwise, refuse with the existing `prop_traversal` error, message, and hint.

Keep every current error-code precedence that the existing tests pin:

- `..` traversal outside the vault and absolute paths outside it give `prop_traversal`,
  never `prop_missing`. This includes non-existent targets outside the vault.
- In-vault symlinks that point outside give `prop_symlink_escape`.
- Non-Markdown targets give `prop_not_markdown`.

`path_safety_rejects_traversal_outside_and_non_markdown` and `symlink_escape_is_refused`
must keep passing unchanged. Grep the props module for any other copy of the same
lexical-prefix confinement pattern and apply the same fix there. Leave other command
families alone unless they share this exact helper.

### Regression test, reproducible on Linux

Add a `#[cfg(unix)]` test to `tests/cli/props.rs` that builds the macOS shape with a
symlink:

- Create the vault with the existing `vault(...)` and `write_note(...)` helpers, plus an
  alias symlink beside it that points at the vault directory.
- `props show <alias>/notes/groceries.md -b <real vault>` succeeds with
  `path == "notes/groceries.md"`.
- The reverse spelling, `-b <alias>` with a real-path absolute note, succeeds too.
- A note outside the vault, reached through the alias with `..` or through a sibling
  directory, is still refused with `prop_traversal`.

Without the fix, this test must fail on Linux. Verify that once by temporarily reverting
the source change locally.

## Root cause B: Linux-only test helper (3 tests)

`wait_until_child_opens_lock` in `tests/cli/props.rs` polls `/proc/{pid}/fd`. That path
does not exist on macOS, but the helper and its three callers are `#[cfg(unix)]`, so on
macOS every call times out after 5 seconds and panics at `tests/cli/props.rs:767`
("child … never opened …/vault.lock"). The product lock path
(`ob::acquire_lock_waiting`) is not implicated.

> [!decision] lock_probe = portable Keep the `/proc/{pid}/fd` probe under
> `#[cfg(target_os = "linux")]`. On other Unix targets, poll
> `lsof -n -P -a -p <pid> -Fn`, parse its `n<path>` lines, and match each path by
> `(dev, ino)` against the lock file's metadata. Match by inode, not by string, because
> lsof reports `/private/var/...`. lsof ships with macOS and the `macos-latest` runner.
> Keep a bounded deadline; up to about 10s is fine because lsof is slower than `/proc`.
> On timeout, panic with the last lsof exit status and stderr. If lsof is missing, panic
> with an explicit message instead of silently passing.

> [!decision] lock_probe = linux_only Change the helper and the three `lock_wait_*`
> tests to `#[cfg(target_os = "linux")]`, following the `tests/cli/dataview_oom.rs`
> precedent. Add a short comment that the readiness probe reads `/proc`.

## Constraints

- **Release rules:** follow bob-cli's "Bob Cli Releases Ride A Daily Train" decision
  (`sase memory read decisions:bob-cli-releases-ride-a-daily-train` from the bob-cli
  checkout).
  - Do not edit the `Cargo.toml` version or `CHANGELOG.md`.
  - Do not dispatch `release.yml` by hand or use its `override_reason` hotfix path. The
    scheduled train publishes.
  - Use a `fix(props): …` conventional commit so release-plz cuts a patch release.
- **Bead and monitor:** do not change `bob-cli-ph.3`'s status, close any bead, or touch
  its monitor. Its continuation agent already does host verification and closure once a
  qualifying tag appears.
- **Scope:** do not weaken the other props tests or the path-safety behavior they pin.

## Verification

1. Run the focused props integration tests locally, plus bob-cli's standard pre-land
   check recipe. Inspect the `justfile` and the repo `AGENTS.md` for the right recipe.
2. Before landing, prove the macOS fix on real macOS. Run `scripts/dev/macos-ci-check`
   from the bob-cli checkout. It pushes a throwaway `ci/precommit/*` branch and waits
   for the exact-SHA macOS run. Because it is long, start it with your `/sase_monitor`
   skill. Require every macOS step green, with 0 failed in the CLI suite.
3. After landing, watch the master `ci.yml` run for the landed SHA until both
   `test (ubuntu-latest)` and `test (macos-latest)` pass, again through `/sase_monitor`
   (for example `gh run watch <id> --exit-status`).
4. Record a note on bead `bob-cli-ph.3` with the fix SHA and the green master CI run
   URL, so its pending continuation sees the blocker resolved. Use your `/sase_beads`
   skill. If cross-project bead notes are not writable from this workspace, say so in
   the final response instead.
5. **Timing check:** the `bob-cli-ph.3` monitor expires around 2026-11-26 00:05 UTC.
   Recent scheduled release runs have started between about 16:00 and 22:15 UTC. If the
   fix will not be on green master before the last scheduled run inside that window, say
   so in the final response so Bryan can re-arm `bob-cli-ph.3`. Do not re-arm it
   yourself.

## Out of scope (mention, do not implement)

The deadlock pattern itself is out of scope. A phase whose gate depends on an external
blocker records that blocker only as a `PROPOSED FOLLOW-UP` note, which its own epic's
land agent routes only after that phase closes. That is a SASE workflow concern for a
separate task, not part of this fix.
