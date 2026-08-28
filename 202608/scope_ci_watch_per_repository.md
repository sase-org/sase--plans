---
tier: tale
title: Scope ci_watch release gates per repository
goal:
  Each release repository resolves safe gate and merge settings independently,
  preserving flat-form compatibility while restoring plugin releases.
size: medium
proposed_by: bbugyi200.athena.sase-um.9.1
bead: sase-um.9.1
---

- **PARENT:** [202608/release_gate_completion.md](release_gate_completion.md)
- **BEAD:**
  [sase-um.9.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.9.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-um.9.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-um.9.1.md)
- **COMMITS:**
  - [ec5e82f](https://github.com/bbugyi200/dotfiles/commit/ec5e82fb7490b21395a60bccd92cecf5c4b91379)
    — chore(config): scope ci_watch release gates by repo

# Scope `ci_watch` release gates per repository

## Goal

Make `bugyi_chop_ci_watch` resolve `merge_method`, `gating_workflows`,
`heavy_workflows`, and `heavy_max_age_hours` independently for each configured release
repository while preserving every existing flat-form configuration. Fail closed on
invalid mappings, prevent merge attempts whose strategy the repository disables, release
the compatible plugin version, and roll Athena's source configuration to restore the
plugin repositories' pre-gate behavior without changing `sase-org/sase`'s gate.

## Implementation

1. In `bugyi-chops`, introduce a small immutable per-repository release-settings value
   and parsing helpers for the four configurable fields. Each helper accepts its
   existing flat scalar/list form as the default for every release repository or a
   mapping whose keys are normalized release-repository slugs plus the reserved
   `default` key. Resolve settings during `Config.from_invocation`: an explicit
   repository entry wins, then the mapping's `default`, then the existing built-in
   (`merge`, empty workflow tuples, and six hours). Reject non-string keys, unknown
   repositories/reserved keys, malformed values, and invalid merge methods through
   `CiWatchError` before any chop work starts.

2. Replace chop-global gate lookups with the resolved settings for the repository being
   evaluated. Thread those settings through the initial release decision, merge-plan
   creation, the final default-branch/gating/heavy-lane reread, merge submission, and
   the durable outcome record so a single tick can safely plan different methods and
   gates for different repositories.

3. Extend the GitHub reader's repository metadata handling to expose/cache allowed merge
   strategies for the tick. Before submitting a live merge, compare the planned method
   with `allow_merge_commit`, `allow_squash_merge`, or `allow_rebase_merge`; fail closed
   with the distinct `merge_method_not_allowed` release reason and do not call
   `gh pr merge` when the repository forbids it. Reuse cached metadata with
   default-branch resolution so the safety check does not create unbounded API traffic.

4. Expand `tests/test_ci_watch.py` with focused parser and end-to-end tick coverage for:
   unchanged flat values across multiple repositories; different mapped methods,
   workflow allowlists, and freshness windows in one tick; repository/default/built-in
   fallback precedence; normalized unknown and malformed mapping failures; allowed merge
   metadata parsing; and a disallowed method producing `merge_method_not_allowed`
   without a merge call. Keep all existing flat-form assertions green. Update the
   README's configuration contract and examples to explain both forms and their
   precedence.

5. Bump `bugyi-chops` from `0.8.0` to `0.9.0`, run its complete `just check`, and
   inspect the resulting diff. Prepare the tag-driven release only after the host-owned
   repository change is durable; verify the matching `v0.9.0` publish workflow succeeds
   and PyPI exposes the new version before installing it into Athena's SASE environment.

6. In the linked `chezmoi` source, change `home/dot_config/sase/sase_athena.yml` to map
   all four settings explicitly: preserve `sase-org/sase` as `merge`, `Master Gate`,
   `Full CI`, and six hours; set `sase-org/sase-github` and `sase-org/sase-telegram` to
   `squash` with empty workflow allowlists (and an explicit safe freshness value or
   default). Apply the source overlay, install the released plugin, and run
   `sase chop doctor`.

## Verification and closeout

- Run focused `ci_watch` tests while iterating, then `just check` in `bugyi-chops`.
- Validate the applied Athena configuration and run a dry-run `ci_watch` tick. Confirm
  `sase-org/sase-telegram` reports neither `gating_workflow_missing` nor
  `heavy_lane_not_green`, while `sase-org/sase` retains its prior release reason.
- If PR #21 is otherwise eligible, allow the live chop to submit its squash merge and
  verify `ci_watch_state.json` records `squash_merge_submitted`; do not force a hand
  merge.
- Record any out-of-scope discovery only as a `PROPOSED FOLLOW-UP:` note on
  `sase-um.9.1`.
- Run `sase bead epic-symbols sase-um.9.1` and resolve or re-key every leftover symbol.
  Close only `sase-um.9.1` with a note naming the local suite, release/doctor checks,
  dry-run reasons, and live merge evidence actually observed. Do not close its parent or
  ancestors.
