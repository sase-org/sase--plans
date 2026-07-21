---
tier: epic
title: Commits sub-tab query filter correctness
goal: 'Every supported query filter property on the Artifacts Commits sub-tab (repo:,
  author:, since:, until:, sidecar:, limit:, free text, and negations) behaves the
  way the query reads: date windows include the days the user named, relative windows
  stay anchored to "now", the match count never claims exactness while the hidden
  row cap is truncating results, and the cap itself is visible and actionable.

  '
phases:
- id: date-bounds
  title: Re-anchorable date bounds and end-of-day until
  depends_on: []
  size: medium
  description: '''Phase 1: Re-anchorable date bounds and end-of-day until'' section:
    rework sase/vcs_log/dates.py and the commit filter query model so day-granular
    until: values resolve to end-of-day, and relative bounds (24h/7d/...) are stored
    as re-anchorable bound specs whose epoch is resolved with an explicit now at use
    time, with text-normalized value equality.

    '
- id: collection-truth
  title: Truncation-aware collection and consistent git date windows
  depends_on:
  - date-bounds
  size: medium
  description: '''Phase 2: Truncation-aware collection and consistent git date windows''
    section: plumb truncation metadata through collect_vcs_log/VcsLogResult, resolve
    relative bounds at collection time, and widen the git-side --until window so rebased
    commits (committer date later than author date) are not silently dropped before
    the exact in-memory author-time matcher runs.

    '
- id: tui-status
  title: Truthful commits-pane status, cap visibility, and cache correctness
  depends_on:
  - date-bounds
  - collection-truth
  size: medium
  description: '''Phase 3: Truthful commits-pane status, cap visibility, and cache
    correctness'' section: make the filter bar report "N+ / capped" instead of "exact"
    when results were truncated, surface the active row cap in the pane, key the pane''s
    caches and change-detection off text-normalized filter values so unchanged queries
    and relative windows behave correctly in long sessions, and refresh docs, help
    text, and PNG goldens.

    '
create_time: 2026-07-21 10:14:38
status: wip
bead_id: sase-8h
---

# Plan: Commits sub-tab query filter correctness

## Context and diagnosis

A screenshot of the ACE Artifacts → Commits sub-tab running the query `sidecar:false since:2026-07-15` shows "40 matches
· exact" while the git history actually contains 624+ matching commits in the primary repo alone. Widening or narrowing
`since:` appears to do nothing, which reads as "the queries are broken". Investigation found four real defects in the
commit-filter pipeline (`src/sase/vcs_log/`, the git provider mixin, and the
`src/sase/ace/tui/widgets/artifacts/commits*` modules):

1. **Silent truncation is reported as "exact".** `DEFAULT_COMMIT_LOG_LIMIT = 40` caps both the per-repo git fetch and
   the aggregated timeline. The canonical query string (`to_query_string`) deliberately omits `limit:` at the default
   value, and `_apply_result` in the collection mixin unconditionally calls
   `set_status(len(displayed.commits), exact=True)`. When the cap saturates, the count is a lower bound, but nothing in
   the UI says so. This is the primary cause of the perceived breakage: every date/repo query wide enough to exceed 40
   rows collapses to "the newest 40 commits · exact".

2. **`until:` excludes the entire named day.** `parse_time_bound("2026-07-15")` resolves to midnight at the _start_ of
   the day for both `since:` (correct) and `until:` (wrong for user expectations): `until:2026-07-15` drops all of Jul
   15's commits and `until:today` matches nothing from today. `since:X until:X` can only match commits at exactly
   midnight.

3. **Relative bounds are pinned at parse time.** `CommitLogFilterValues` stores the resolved epoch, so re-parsing the
   canonical string `since:24h` one second later yields an unequal value (verified). This breaks value-equality caching
   (`_authoritative_results`, `_collection_matches`, `_live_filter_values` comparisons), bumps the generation when the
   filter bar is opened on an unchanged query, and pins the bundled default `sidecar:false since:24h` window to TUI
   startup time — in a long-lived session "24h" silently becomes "24h + uptime".

4. **Author-time display vs committer-time git filtering.** The pinned wire format uses `%at` (author time) for display,
   sorting, grouping, and the in-memory matcher, but `git log --since/--until` filter by committer date. In this
   rebase-heavy workflow 141 of 200 sampled commits have differing author/committer times (mostly seconds, occasionally
   1–12 hours). Concretely: a commit authored inside an `until:` window but rebased (re-committed) after it is dropped
   by git before the exact in-memory matcher ever sees it, even though the timeline would display it inside the window.
   For `since:` the committer-date filter is a safe superset (committer time ≥ author time in practice), so only
   `until:` loses commits.

## Design decisions

- **Author time stays canonical.** Display, sorting, and exact filtering keep using `%at`. Switching the stack to
  committer time was rejected: it would change the pinned `VCS_LOG_GIT_FORMAT` wire contract and the Rust
  parser/aggregator in the sibling Rust core repo, plus subtly change every displayed timestamp, for a skew that is
  better handled by widening the git-side prefetch window. **No Rust core / wire-format changes are needed anywhere in
  this epic**: truncation metadata lives in the Python `VcsLogResult` model, and the Rust
  `parse_git_log`/`aggregate_commit_log` APIs are untouched.
- **The default row cap stays at 40.** Rendering unbounded result sets by default would regress TUI responsiveness (the
  timeline is rebuilt from the result). Instead the cap becomes _honest and visible_: a truncated result reports "N+
  matches · capped", the pane surfaces the active `limit:40`, and `limit:all` / `limit:N` remain the explicit escape
  hatches (already offered by completion).
- **Day-granular `until:` means end-of-day.** `until:YYYY-MM-DD`, `until:today`, `until:yesterday` resolve to the last
  second of the named day (equivalently, exclusive next-midnight). Instant-precise forms (`YYYY-MM-DDTHH:MM`) and
  relative forms (`Nh/Nd/Nw`) keep exact-instant semantics. `since:` is unchanged.
- **Relative bounds re-anchor at use time; equality is text-normalized.** Parsing produces a bound spec that retains the
  kind and payload (relative amount/unit, day keyword, or absolute instant); resolution to epoch seconds takes an
  explicit `now`. Two filter values parsed from the same query text are equal.
- **Git-side date arguments are a coarse superset; the in-memory matcher is the source of truth.** The existing
  `compile_commit_matcher` already re-applies since/until/authors/text to everything displayed, so correctness only
  requires that the git prefetch never _excludes_ a commit the matcher would keep.

## Phase 1: Re-anchorable date bounds and end-of-day until

Rework the date layer in `src/sase/vcs_log/dates.py` and the query model in `src/sase/vcs_log/filter_query.py`:

- Introduce a small frozen `TimeBound` (name flexible) produced by parsing: it records the normalized source text and
  enough structure to resolve deterministically — relative (`amount`, `unit`), day keyword (`today`/`yesterday`), or
  absolute instant. It exposes resolution to epoch seconds given an explicit `now` and a boundary mode (start-of-day for
  `since:`, end-of-day for `until:` on day-granular forms). Keep `parse_time_bound` (or a successor) as the shared
  validation/parse entry point; `VcsLogDateError` and `DATE_HELP` stay, with `DATE_HELP` updated to state that
  day-granular `until` bounds are inclusive of the named day.
- `CommitLogFilterValues` carries the bound specs instead of pre-resolved epochs (`since_text`/`until_text` remain the
  canonical rendering source). Dataclass equality/hashing must be text-normalized so canonical round-trips are stable.
  `backend_filters()` and `compile_commit_matcher` accept/resolve bounds with an explicit `now` parameter (callers pass
  real time; tests inject).
- Parse-time validation of `since:` > `until:` compares both bounds resolved at one shared `now`; a same-day
  `since:X until:X` pair becomes a valid full-day window. Error spans/messages keep their current shape.
- Update the CLI handler (`src/sase/main/vcs_handler.py`) so `sase vcs log --until/--before` uses end-of-day resolution
  for day-granular values, resolved at command execution time.
- The Plans sub-tab and `sase plan search` share `parse_time_bound` via `src/sase/plan_search/filter_query.py`. Migrate
  that caller to the same helper API and give its `until:`/`--until` the same end-of-day semantics so the two adjacent
  Artifacts sub-tabs don't disagree; keep this strictly a semantics alignment (no other plan-search behavior changes).
- Mechanically update in-repo consumers of the changed fields (collection/filtering mixins, `snapshot_covers`,
  `_snapshot_breadth`, render/report paths) so behavior is preserved pending phases 2–3; they resolve bounds with real
  `now` at their existing call sites.

Tests: extend `tests/test_vcs_log_dates.py` (end-of-day resolution, day keywords, DST-safe boundary math in the
configured timezone, injected `now`), `tests/test_vcs_log_filter_query.py` (round-trip equality of relative bounds,
same-day since/until validity, canonical rendering unchanged), CLI handler coverage, and plan-search filter tests for
the aligned `until:` semantics.

## Phase 2: Truncation-aware collection and consistent git date windows

Backend collection changes in `src/sase/vcs_log/collect.py`, `src/sase/vcs_log/models.py`, and the git provider mixin
(`src/sase/vcs_provider/plugins/_git_query_ops.py`):

- **Truncation metadata.** Extend the Python `VcsLogResult` with truncation info: whether the aggregated timeline was
  cut at the limit and whether any repo's git fetch saturated its per-repo cap (meaning more matches may exist beyond
  what was fetched). Computable entirely Python-side in `collect_vcs_log`: the pre-truncation total is the sum of
  per-repo list lengths, and a repo "hit its cap" when it returned exactly the requested per-repo limit. Unlimited
  collections (`limit:all`, or the presentation-filter path that already collects uncapped via
  `_backend_collection_limit`) are never marked truncated. Default the new fields so existing constructors/tests keep
  working.
- **Resolve bounds at collection time.** `collect_vcs_log`/`run_vcs_log` resolve since/until bound specs to epochs using
  the existing injectable `now` (falling back to real time), so every scheduled TUI refresh and CLI run re-anchors
  relative windows. `CommitFilters` stays a provider-neutral epoch pair — only the resolution point moves.
- **Widened git-side until window.** When passing `--until` to git, widen the epoch by a fixed slop constant (7 days —
  comfortably above the observed ≤ 12 h author/committer drift, cheap because it only admits a boundary margin that the
  exact matcher then trims). When an `until:` bound is active, raise the per-repo git cap (e.g. double the requested
  limit, unlimited stays unlimited) so matcher-trimmed margin commits cannot starve the displayed row budget. `--since`
  continues to be passed exactly (it is already a safe superset under author-time semantics). Document the residual,
  deliberately-accepted gap: a commit rebased more than the slop after its author date, or matching commits beyond the
  widened cap, can still be missed.
- The exact author-time bounds are enforced where they already are today: `compile_commit_matcher` inside the pane's
  `_filtered_result`, and the CLI render path must apply the same exact trim so `sase vcs log --until` output matches
  the TUI (verify how the CLI currently consumes `CommitFilters`; add the trim if it renders raw provider output).

Tests: `tests/test_vcs_log_collect.py` for truncation flags across capped/uncapped/multi-repo cases and for
`now`-injected re-anchoring; `tests/test_vcs_provider_vcs_log.py` with a real git fixture containing a rebased commit
whose author date is inside an `until:` window but committer date is after it, proving the widened window plus matcher
keeps it; CLI-level assertion that the named `--until` day is included.

## Phase 3: Truthful commits-pane status, cap visibility, and cache correctness

TUI changes in `src/sase/ace/tui/widgets/artifacts/` (commits collection/filtering mixins, rendering, filter bar) riding
entirely on existing refresh paths (no new refresh code paths, per the TUI perf rules; collection stays in the existing
thread worker):

- **Status honesty.** `_apply_result` reports `exact=True` only when the authoritative result was not truncated; a
  truncated result renders as "N+ matches" with a "capped" coverage label (distinct styling from "preview").
  Live-preview labeling continues to flow through `snapshot_covers`, which must account for the new truncation metadata
  instead of re-deriving `len(commits) >= collection_limit`.
- **Cap visibility.** When (and only when) a displayed result was truncated, surface the active row cap in the pane —
  filter chips (`commit_filter_chips`) and/or the commits info line — so `limit:40` stops being invisible; the existing
  `limit:` completion (`all`, larger values) is the escape hatch.
- **Cache and change-detection correctness.** With text-normalized value equality from phase 1, verify and pin: opening
  the filter bar on an unchanged query neither bumps the generation nor triggers a spurious collection;
  `_authoritative_results` keys, `_collection_matches`, and `_reconcile_live_filter` compare stable values; scheduled
  refreshes naturally re-anchor `since:24h`-style windows (each `_collect` resolves `now`), so the bundled default query
  stays a sliding 24-hour window across long sessions. Re-filtering a cached relative-window result with a fresh `now`
  only ever narrows it (the cached window is older, hence a superset), so previews stay correct-by-trimming until
  reconciliation lands.
- **Docs, help, config.** Update `docs/ace.md` (Commits filter section and the `sase vcs log` CLI row),
  `docs/configuration.md` (`default_query` row), the filter bar `KEY_COMPLETIONS`/`VALUE_HINTS` hint strings, and — per
  the ace help-popup rule — any help-modal text documenting commits filters, to state the end-of-day `until:` semantics
  and the capped-count indicator. Check `src/sase/default_config.yml` comments for the commits default query while at
  it.
- **Visual goldens.** The status-text change will invalidate some `artifacts_commits_*` PNG snapshots; add or update a
  snapshot that pins the truncated "N+ · capped" state, and regenerate affected goldens intentionally via the
  visual-suite update flag after inspecting the diff artifacts.

Tests: `tests/ace/tui/test_commits_pane_filters.py` / `_rendering.py` / `_interactions.py` and
`tests/ace/tui/test_commits_config.py` for the status states, cap indicator, no-op filter-bar open, and relative-window
re-anchoring (injected collector/now); `tests/ace/tui/widgets/test_commit_filter_bar.py` for the new coverage label; the
PNG visual suite for the pinned states.

## Acceptance criteria

Mapped to the original complaint (`sidecar:false since:2026-07-15` looking inert):

- With more than 40 matches, the bar shows a lower-bound count with a "capped" label — never "40 matches · exact" — and
  the pane shows the active cap; `limit:all` then shows the full set with an exact count.
- `until:2026-07-15` includes commits authored anywhere on Jul 15; `until:today` shows today's commits;
  `since:X until:X` is a valid full-day window (TUI and `sase vcs log`, and the aligned Plans `until:`).
- A commit authored inside an `until:` window but rebased/re-committed after it still appears (within the documented
  7-day slop).
- Opening and dismissing the filter bar with unchanged text causes no re-collection; `since:24h` remains a sliding
  window across a multi-day TUI session.
- All existing filter properties (`repo:`, `author:`, `sidecar:`, `limit:`, free text, `-` negations) keep their current
  behavior, pinned by the existing suites.

## Risks and edge cases

- **Timezone/DST boundaries:** end-of-day math must use the configured timezone (`sase.core.time.get_timezone`) and be
  exercised across a DST transition in tests.
- **`until:`-only queries:** without `since:`, the widened git window still walks deep history under a raised cap; the
  cap raise is bounded (2× limit) precisely to keep this predictable. Residual undercount beyond the widened cap is
  accepted and documented.
- **Behavior change surface:** end-of-day `until:` changes results for the CLI (`sase vcs log`, `sase plan search`) and
  Plans tab, not just Commits — treated as a deliberate, documented semantics fix and covered by updated tests rather
  than compatibility shims.
- **Snapshot churn:** PNG goldens must only change where status/chips render; diffs are inspected before accepting.
- **Equality-semantics refactor:** moving `CommitLogFilterValues` off pre-resolved epochs touches every consumer
  (mixins, snapshot coverage, tests); phase 1 lands it mechanically with behavior preserved so phases 2–3 stay small and
  reviewable.
