---
tier: tale
title: Negative filters for Artifacts commits and plans
goal: 'The Artifacts Commits and Plans filter bars accept exclusion terms such as
  -repo:plans, preserve exact result and completion behavior, and remain responsive
  while filtering live data and deep plan archives.

  '
create_time: 2026-07-18 15:33:58
status: done
prompt: 202607/prompts/negative_artifact_filters.md
---

# Plan: Negative filters for Artifacts commits and plans

## Context and outcome

The Commits and Plans panes already share tokenization and completion mechanics, but they parse into separate filter
models and execute against different data paths. Commits combine repository resolution and provider-side author/date
filtering with a debounced in-memory subject matcher; Plans match a prefolded snapshot and, when necessary, reconcile a
bounded deep archive off-thread. Add one consistent unary exclusion syntax without weakening the existing exact/preview,
limit, cache, or tree behavior.

The user-facing contract will be:

- A leading, unquoted `-` negates a match-bearing term. Commits support negative `repo:`, `author:`, and free-text
  terms; Plans support negative `kind:`, `status:`, `tier:`, `project:`, and free-text terms. Examples include
  `-repo:plans`, `author:Ada -author:bot`, `status:open -status:blocked`, and `-"generated rollout"`.
- Polarity applies to the whole token, so `-repo:plans,research` excludes either repository. Positive values retain OR
  semantics within a facet; all positive facets and free-text terms must match; any matching negative term vetoes the
  row. When positive and negative constraints overlap, exclusion wins.
- Date bounds remain affirmative range constraints, and `limit:` remains a result control. Negated `since:`, `until:`,
  or `limit:` tokens fail with a span-aware validation error instead of acquiring surprising complement semantics.
- Quoting the whole token keeps it literal (`"-repo:plans"` is searchable text), hyphens inside keys or values remain
  ordinary characters, matching stays case-insensitive where it is today, and repository/project aliases work for both
  inclusion and exclusion.
- Plans retain their current tree presentation: an epic may remain visible as structural context for a matching phase,
  while match counts and deep-archive decisions are based on the actual positively constrained, non-excluded records.

## Shared syntax, serialization, and completion

Teach `src/sase/filter_tokens.py` to identify an optional leading unquoted negation marker while preserving the original
raw token and source span for diagnostics. Use that shared interpretation in both filter parsers rather than duplicating
minus handling. Extend the Commits and Plans value objects with explicit exclusion collections, include those
collections in empty-state/equality/cache identity, and make their canonical serializers emit stable, round-trippable
positive and negative tokens. The same serializer output should continue to drive the header chips, so committed
negative queries are visible without a second rendering vocabulary.

Update the shared `FilterBar` completion path and the two concrete bars so typing `-re`, `-repo:pl`, or `-status:bl`
offers the same key/value candidates as the affirmative form and accepting a candidate preserves the leading minus,
including quoted and comma-separated values. Only negatable keys should be offered after a leading minus. Completion and
parsing must stay pure and in-memory on the keystroke path.

Document the two filter vocabularies, negation rules, quoting escape hatch, and representative combined queries in the
Artifacts section of `docs/ace.md`.

## Commit filtering and collection reconciliation

Extend `CommitLogFilterValues` and `compile_commit_matcher` so negative repositories use the same canonical labels and
aliases as positive repositories, negative authors veto name or email substring matches, and negative subject terms veto
case-insensitive subject matches. Keep positive author/date constraints in `backend_filters()`; exclusions must not be
mistaken for provider-supported positive filters.

Thread negative repository selections through the existing `run_vcs_log` / repository-resolution boundary. Resolve the
complete scoped catalog first, apply positive selection when present, then remove every canonical-name or alias match
for the negative set, with exclusions winning over inclusions. This makes `-repo:plans` omit matching sidecars before
provider I/O, keeps the result legend/repository metadata honest, and preserves the requested final row limit. In
all-project scope, an exclusion alias should remove every matching repository rather than become unusable merely because
the alias is shared. Retain useful non-fatal warnings for unmatched exclusions and cover interactions with positive
filters and SDD scope.

For negative authors or subject terms, which cannot be pushed through the provider interface, collect an uncapped
author/date/repository-filtered authoritative set before applying the UI row limit, just as positive subject filtering
already does. Update snapshot breadth and exact-coverage checks to distinguish constraints that narrowed backend
collection (including negative repositories) from presentation-only exclusions, so live previews never claim exactness
from an insufficient cache and broad reusable snapshots are still preferred. Reuse the current debounce, worker,
last-generation-wins, Escape restoration, and committed-query cache flows; do not add synchronous repository work to the
event loop.

## Plans matching and archive coverage

Extend `PlanFilterValues` and `compile_plan_matcher` with excluded kinds, statuses, tiers, projects, and text. A record
must satisfy the existing positive matcher and then survive every negative facet/text veto, including records with
multiple derived status or tier labels. Apply the same compiled matcher to snapshot rows and worker-fetched archive rows
so preview and deep results cannot diverge.

Update plan-filter emptiness and deep-archive reachability for exclusions. In particular, `-kind:archive` must avoid a
deep archive scan, while exclusions that still permit archive rows (for example `-kind:proposal` or a negative text
term) must retain the current debounced request, cache, cap, stale-result, and exact/preview behavior. Preserve the
existing ancestor-context behavior for matching phases and ensure displayed match counts count only records accepted by
the full positive-plus-negative predicate.

## Verification

Add focused and property-based coverage in `tests/test_vcs_log_filter_query.py` and `tests/test_plan_filter_query.py`
for mixed polarity parsing, comma lists, canonical round trips, quoting, exact error spans, aliases, case folding,
overlapping positive/negative constraints, multi-label plan records, and negative text. Add
repository-resolution/collection tests for pre-collection exclusion, shared aliases, inclusion-plus-exclusion
precedence, warnings, SDD scope, and limit preservation.

Extend the shared widget tests to exercise negative key and value completion, candidate insertion, quoted values, and
non-negatable keys. Add pane-level tests proving live `-repo:plans` filtering and reconciliation in Commits and negative
kind/status/project/text filtering in Plans, including Escape/submit persistence, cache exactness, tree counts, and deep
archive reachability/capping. Update a focused PNG snapshot only if the visible negative chip/completion contract is not
adequately represented by existing snapshots; avoid unrelated golden churn.

Run `just install` before repository checks as required for an ephemeral workspace, then run the focused parser,
resolver, widget, Commits pane, and Plans filtering tests during iteration. Finish with `just check`; if a visual golden
is intentionally changed, also run `just test-visual` and inspect the generated diff artifacts before accepting it.
