---
tier: tale
title: Truncation-aware VCS collection and consistent git date windows
goal: Commit-log collection re-anchors semantic date bounds per operation, preserves
  enough widened git candidates for exact author-time filtering, and reports when
  provider or aggregate caps may have hidden matches.
create_time: 2026-07-21 10:43:02
status: done
bead: sase-8h.2
parent: sase/repos/plans/202607/commits_filter_correctness.md
prompt: '[202607/prompts/truncation_aware_vcs_collection.md](prompts/truncation_aware_vcs_collection.md)'
---

# Plan: Truncation-aware VCS collection and consistent git date windows

## Context and scope

Phase 1 of epic `sase-8h` replaced parse-time date epochs with stable `TimeBound` values and made day-granular `until:`
bounds inclusive. The remaining collection path still resolves those values before entering `run_vcs_log`, while the git
provider applies `--since` and `--until` to committer dates even though SASE's pinned `%at` wire field, sorting,
display, and exact matcher all use author time. A rebased commit authored inside an upper-bound window can therefore be
discarded before the author-time matcher sees it. The current result model also cannot distinguish a complete timeline
from one cut by the aggregate limit or by a saturated per-repo fetch.

This tale implements only phase 2 (`sase-8h.2`). It does not change Rust aggregation or the git wire format, increase
the default visible row cap, or implement phase 3's TUI status labels, cap chips, cache policy, documentation, help
text, or visual snapshots. Collection remains on the existing commits-pane thread worker and introduces no new refresh
path or event-loop work.

## Collection-time filters and result truth

Keep `CommitFilters` as the provider-neutral, epoch-based boundary passed to VCS plugins. Add a collection-layer way to
carry stable `TimeBound` specifications and authors until `collect_vcs_log` / `run_vcs_log`; resolve both bounds once
against the operation's injectable `now`, then construct the epoch filters used for every repo. Preserve direct
epoch-filter callers where useful, but migrate the commits pane and CLI so relative bounds are anchored by the
collection operation rather than by earlier parsing or UI code. The same captured reference must drive lower and upper
bounds, fetch-cache timing, exact matching, and deterministic tests.

Extend `VcsLogResult` with defaulted truncation metadata that separately represents:

- aggregate truncation, computed from the candidate count before the global timeline cap; and
- possible provider truncation, set when any bounded repo fetch returns exactly its requested per-repo cap.

Unlimited collection must never claim either cap-based condition. Preserve the metadata when `run_vcs_log` prepends
resolution warnings and when existing filtering code uses dataclass replacement, so old constructors remain
source-compatible and phase 3 can consume one authoritative signal rather than infer completeness from displayed length.

## Safe git candidate window and exact author-time output

Define and document a fixed seven-day git upper-bound slop. The git provider should continue passing `--since` exactly,
but pass `--until` at the requested epoch plus the slop so a commit authored inside the window and committed by a later
rebase remains a candidate. When an upper bound is active, use a bounded raised candidate cap (twice the requested
limit); unlimited requests remain unlimited. Keep enough of that widened candidate set through collection for the
existing TUI matcher and the CLI renderer to discard out-of-window author timestamps before applying the user's final
row limit, preventing margin-only commits from consuming the visible budget.

Apply the exact inclusive epoch bounds consistently to every CLI output format, including JSON, and finalize
ordering/reversal and the requested limit only after that trim. The commits pane continues using
`compile_commit_matcher` as its exact author-time predicate. Document the conservative boundary in code: commits whose
committer date trails author date by more than seven days, or matching history beyond a saturated widened candidate cap,
may still be absent and must therefore leave the result marked potentially truncated.

## Verification

Expand `tests/test_vcs_log_collect.py` to cover aggregate-only truncation, per-repo saturation, combined and multi-repo
cases, exact-cap ambiguity, unlimited behavior, propagation through `run_vcs_log`, widened bounded candidate limits, and
injected-time resolution of relative bounds. Keep failure isolation, remote comparison, fetch caching, repo
inclusion/exclusion, and legacy epoch-filter behavior pinned.

Add git-provider coverage for the exact generated date arguments and a real repository whose author date is inside an
`until` window while its committer date is later. Prove the seven-day coarse fetch admits it and the exact
matcher/output keeps it while excluding margin commits. Extend CLI handler/render tests to show a named `--until` day is
included, relative values use one injected collection reference, all formats trim by author timestamp, and the final cap
is filled from matching candidates.

Run `just install` before project checks, then execute the focused collection, provider, query, CLI-handler, renderer,
and commits-pane suites. Run `just check`, fix every failure, and rerun the focused regressions after any correction at
the filter, cap, or render seams. Finally inspect the diff and working tree, close only `sase-8h.2`, and verify parent
epic `sase-8h` and sibling `sase-8h.3` remain open; do not create any beads.
